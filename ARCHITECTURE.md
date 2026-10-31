# QlsOS architecture, v0 (Quick Light Secure OS)

Goal of v1.0.0: a **capability microkernel** that is safe, small, understandable and starts programs fast.
This page is the contract for stages B-D of `QlsOS_tasks.txt`. Where a decision belongs to a later task it is marked *(decided in Task N)* and the default written here is what that task starts from.

## Rules that never bend

1. **Everything goes through channels and handles.** No PIDs in the API, no signals, no global names. A task can touch only what it holds a handle to.
2. **The kernel does only what must run in ring 0:** memory, tasks and scheduling, interrupts, handles, channels, the ELF loader. Console, keyboard, disk, file system and shell are ordinary tasks (stage D-E).
3. **64-bit clean.** v1.0.0 stays 32-bit, but no line may assume it:
   - addresses and pointer-sized integers are `uintptr_t`, sizes and counts are `size_t`; never `uint32_t`/`int` for either, never `long`;
   - anything that crosses the user/kernel boundary (syscall arguments, messages, handle values) has an explicit width and its layout is identical on 32 and 64 bits: no pointers or `size_t` inside it;
   - 32-bit assumptions live only in `kernel/arch/x86/` and `boot/` (the list is the section *Architecture-dependent places* below, Task 8c). Paid in Task 8a: `task_t.sp` (saved stack pointer is a `uintptr_t`, opaque to `task.c`), `hal_context_switch`, `hal_new_*_stack`, `hal_set_kernel_stack`, `hal_irq_save/restore` (`hal_irq_flags_t`), file/image sizes (`vfs`, `ramfs`, `qlsfs`, `fs`, `process_spawn_elf`, `elf_*`, `exec_read_file`) and entry counts (`*_list`, `task_list`, `hal_pci_list`) are `size_t`; `kprintf`/`klog` got `%zu %zx %zd %p`. What stays fixed-width on purpose: timer ticks, slot generation, bit flags, status codes, sector numbers (`lba`) and every on-disk / ELF32 field.
4. **Every limit is an error for the one task, never a crash.** Memory frames and handle count per task already exist (`task_limits_t`).

## Task

A task is: one address space, one kernel stack, a **handle table**, limits, and a state (`READY / RUNNING / BLOCKED / STOPPED / ZOMBIE`; a free slot is `UNUSED`).

- Identity is `task_ref_t = (slot, generation)`. The generation goes up every time a slot is reused, so a stale reference is *invalid* instead of pointing at the next task (exists since Task 2). A bare slot number is kernel-internal; **userspace never sees a task id.**
- Waiting is one mechanism: `task_block(reason, arg)`. Reasons today: `SLEEP`, `CHILD`, `CHAN_RECV`, `CHAN_SEND` (Task 12); `CHILD` disappears with `wait` (Task 21). A blocked task gets no CPU and is polled by nobody: whoever satisfies the reason wakes it (`task_wake_blocked(reason, arg)`).
- A task dies by exit or by a fault; the *reason stays in the status* (e.g. page fault + address). The spawner learns it from the **result channel** (Task 17, done: section *Result channel* below), not from `wait`. There is no `wait`, no `waitpid`, no `fork` and no parent: they went in Task 21 (section *Task 21* below).
- Control (stop, resume, kill) is a handle of type `TASK` with the right `CONTROL` (Task 18, done: section *Task control* below); there is no `kill <pid>` any more in the shell.

## Handle

A handle is the only way a task refers to a kernel object. The task holds an index; the kernel holds the rest.

Since Task 9 (`kernel/include/handle.h`, `kernel/src/handle.c`):

```c
typedef uint32_t handle_t;              /* user-visible, fixed width: generation (high 16 bits) | slot (low 16) */
#define HANDLE_INVALID ((handle_t)0)    /* a real handle is never 0: generations start at 1 */

typedef struct {                        /* kernel-internal, one per table slot */
    uint8_t   type;                     /* NONE (= free slot), CHANNEL, TASK, ... */
    uint8_t   _pad;
    uint16_t  gen;                      /* bumped on close (Task 10): a closed number never silently works again */
    uint32_t  rights;                   /* bit mask: SEND RECV DUP TRANSFER CONTROL */
    struct grant* g;                    /* Task 19: its node in the derivation tree (NULL = an original nothing was made from) */
    void*     object;                   /* the kernel object (reference-counted from Task 11) */
} handle_entry_t;
```

- **The table lives in `task_t`** (`handle_table_t { entries, cap }`). It is allocated by the first `handle_install` with `cap = min(handle limit, HANDLE_TABLE_MAX = 4096)` entries (the default limit is 64 = 1.5 KiB on 32 bits), so a task without handles costs nothing. It is freed when the task is reaped.
- **API so far:** `handle_install`, `handle_lookup`, `handle_close` and `handle_dup`. All take a `task_ref_t`, never a bare slot, and run with interrupts off per call, so a task being killed or reaped in between gives `HANDLE_E_REF`, not a dangling pointer.
- **`handle_close`:** frees the slot, raises its generation (the closed number never works again, a second close is `E_BAD`), gives the handle back to the task's limit and drops one reference of the object. Handles made from it are **not** revoked: they are re-attached to the closed handle's own parent, so no `derived_from` ever points at a reused slot. Revoking is its own operation *(Task 19)*.
- **`handle_dup(h, rights)`:** needs the `DUP` right on `h` (a handle without it can never be duplicated), and `rights` must be a subset of `h`'s rights (`E_WIDEN` otherwise): rights only shrink, there is no way back up. The new handle has the same type and object, `derived_from = h`, and is charged against the limit like any other.
- **The 16-bit generation wraps** after 65535 closes of one slot and skips 0 (so a handle is never `HANDLE_INVALID`). A very old number of a slot can match again after a full wrap; that is the accepted price of a 32-bit handle.
- **Objects:** `object_retain` / `object_release` in `handle.c` are the only two places where a handle takes or drops a reference. They are empty until channels exist (Task 11); Task 15 turns the last release into the free.
- **Errors are distinct:** `E_REF` (task gone), `E_LIMIT`, `E_NOMEM`, `E_BAD` (never issued / closed / wrong generation), `E_TYPE`, `E_RIGHTS`, `E_ARG`, `E_WIDEN` (dup asked for more rights than it has). `handle_lookup` checks in the order bad handle, type, rights.
- **The limit is the usage counter** `task_limits_t.handles`: every install charges one, and a full table is an error for that task only.
- `derived_from` (a slot of the same table, Task 9) is **gone** in Task 19: it could not cross tables. The derivation tree below replaces it (`handle_install` still takes the `derived_from` argument: the new handle is then made from that one).
- Tests: `handletest` in the shell (QEMU) and `make test-handle-host` (the same logic on a 64-bit host under AddressSanitizer).

- **Rights only shrink:** `handle_dup` and sending a handle in a message (Task 13, done) give the same or fewer rights, never more.
- The derivation tree is what makes `revoke` possible (Task 19, done): revoking a handle invalidates everything derived from it, in every table.
- The last `handle_close` on an object frees the object and its memory. Done for channels in Task 11 (the endpoint counts its handles, the channel is freed when both endpoints are closed); **Task 15 (done, closes block B):** the rule holds for every kind of object and is checked by comparing the kernel heap and the free physical frames before and after. Who frees what: an endpoint when its last handle is closed and no queued message carries it; a channel when both ends are; a handle table when its task dies (a killed process closes its handles that way); the messages of a queue when its endpoint closes; a user process's frames when it is collected. Tests: `freetest` in the shell (QEMU, 21 checks: both ends in either order with messages queued, a dup keeping the object alive, 200 channels without creep, a handle in flight held by the message and released when it is received or when its receiver dies, loops refused, a 33-channel chain freed at once, a user process that ran to its end and one killed while holding the last handle: frames and heap back) and `make test-free-host` (host, AddressSanitizer: 4 tasks, 1.8 million random operations on channels and handles with loops attempted all the time, heap must be EMPTY after every task dies; without the loop check it leaks). `make test-free` runs `freetest` and every earlier self-test.
- Table size per task is bounded by `TASK_DEFAULT_MAX_HANDLES` (64).

## Channel

A channel is a **pair of endpoints**; each endpoint is a handle (`channel_create(owner, &end0, &end1)`). What is sent on one endpoint is received on the other and the other way round; a message never comes back to its sender. Since Task 11: `kernel/include/channel.h`, `kernel/src/channel.c`.

- **Message format (decided in Task 11): fixed-size slots, variable length inside the slot.** A message is 0..`CHANNEL_MSG_MAX` (64) bytes, copied into a slot of the *receiver's* queue:
  ```c
  typedef struct { uint32_t len; uint32_t nhandles; handle_carry_t carry[CHANNEL_MSG_HANDLES_MAX]; uint8_t data[CHANNEL_MSG_MAX]; } msg_t;   /* 120 bytes on x86, 136 on x86-64 */
  ```
  Why not variable-size messages in a byte ring: the kernel would allocate or fragment per message, and a sender could make it grow. Fixed slots mean the memory of a channel is known when it is made (**one channel = 2 queues x 16 slots = ~3.9 KiB since Task 13, one allocation**), a send is a `memcpy` into a free slot, and nothing is ever freed per message. 64 bytes is the *control path*: requests, replies, status. Big data will travel as a shared-memory handle (after v1.0.0); until then the file protocol (Task 31) sends several messages. `CHANNEL_MSG_MAX` and `CHANNEL_QUEUE_LEN` are two constants; nothing else depends on their value. A slot also carries up to `CHANNEL_MSG_HANDLES_MAX` = 4 handles (Task 13, below); that is where the 1.6 KiB per channel went. If memory per channel matters more than 4 handles per message, the constant is the only thing to change.
- **The queue is bounded** (`CHANNEL_QUEUE_LEN` = 16 messages per endpoint, per direction). A full queue is `CHANNEL_E_FULL`: nothing is dropped silently, nothing grows.
- **Two modes (Task 12), chosen by the call, not by a flag on the handle.** `channel_send` / `channel_recv` are **non-blocking**: a full queue is `CHANNEL_E_FULL`, an empty one `CHANNEL_E_EMPTY`, at once, from any context. `channel_send_wait` / `channel_recv_wait` are **blocking**: the task blocks (`CHAN_SEND` / `CHAN_RECV`, state `send-wait` / `recv-wait` in `ps`), leaves the round-robin, gets no CPU, and then does what the non-blocking call would have done. A blocking call never returns FULL or EMPTY; refusals that need no waiting (bad handle, missing right, `CHANNEL_E_SIZE`) come at once. **No timeout yet** (a silent but living peer keeps the caller asleep; it can be killed); a timeout needs a second argument on the block and arrives with the syscall layer.
- **Who wakes whom.** `block_arg` of a channel wait is the *endpoint* (only a name, never dereferenced). A successful send into endpoint `P` wakes the receivers blocked on `P`; a successful recv from `P` wakes the senders blocked for room in `P`; the close of an endpoint (last handle gone) wakes the receivers of its peer (queued messages first, then `CHANNEL_E_CLOSED`) and the senders waiting for room in it (`CHANNEL_E_CLOSED`). All woken tasks re-check; a wake-up is a hint, not a promise (`task_wake` on a waiter is harmless).
- **No lost wake-up.** The attempt and the `task_block` are one section with interrupts off (the pattern of `task_block`), so a send / recv / close that lands right after the attempt finds the task already blocked and wakes it.
- **A dying peer.** The endpoints of a dead task are closed when its handle table is released, and that happens in `reap_zombies`, later, in somebody's context. If the dying task was the last runnable one, nobody would ever run to do it and the waiter would sleep for ever (the machine halts). Therefore **every task death wakes all channel waiters** (`terminate_locked`), and a waiter calls `task_reap()` after every wake-up, *before* it checks again: that closes the dead peer's endpoints and the waiter sees `CHANNEL_E_CLOSED`. A spurious wake-up costs a scan of the slots; a missed one costs the machine. `chanblocktest` section 11 is this case (without the fix it hangs).
- **When blocking is impossible** (`task_block` returns -1: no other task can run, or we are inside a `preempt_disable` section) the caller halts until the next interrupt and checks again, like `task_sleep`: no busy loop, but also: do not call the blocking calls with interrupts off or preemption off.
- **Rights:** `send` needs `SEND`, `recv` needs `RECV` on the endpoint's handle; both endpoints of a new channel have `SEND RECV DUP TRANSFER`, a `dup` can make a send-only or recv-only handle.
- **Closing.** An endpoint is *closed* when its **last handle** goes (`handle_close`, or the task holding it dies: several handles to one endpoint are possible through `dup`). Then the peer's `send` returns `CHANNEL_E_CLOSED`, and its `recv` first **delivers everything already queued**, and only on an empty queue returns `CHANNEL_E_CLOSED` (a dying task's last words are not lost). When both endpoints are closed the channel's memory is freed, with its undelivered messages. A task that dies holding an endpoint closes it; this is how a client learns that a service is gone.
- **Create is all or nothing:** needs two free handles; at the limit it returns `HANDLE_E_LIMIT` and leaves no handle and no memory behind.
- **The API is kernel-level:** `data`/`buf` are kernel addresses. Syscall wrappers come with the first ring-3 user of channels (Task 16): they copy from and to user memory, check the pointers, and then call these functions.
- **Passing a handle in a message (Task 13).** `channel_send_h` / `channel_recv_h` (and `_wait_h`); the old four calls are the same with no handles. Decisions:
  - **Copy, not move.** The sender keeps its handle; to give one away for good it sends and then closes it. Why: a send can fail (full queue, closed peer, a bad second handle) and a copy has nothing to undo; the sender's table does not change at all on an error; a service can hand out a client's channel and keep serving. Rights are narrowed per handle, so "give a weaker copy" is the normal case anyway. *(Task 16 decided the same for `spawn`: copy, see below.)*
  - **Rights only shrink.** Each handle in the message is `{handle, rights}`: `rights` must be a subset of the sender's handle, else `HANDLE_E_WIDEN`; `HANDLE_RIGHTS_SAME` keeps all. The sender's handle needs `HANDLE_RIGHT_TRANSFER`, else `HANDLE_E_RIGHTS` (this is how a handle is made "stays here"; the receiver gets TRANSFER only if the sender passed it on, so a handle can end the road). Rights are fixed at send time.
  - **In flight, the message owns a reference** (`handle_carry_t`: type, rights, object). The object stays alive if the sender closes its handle or dies. The receiver gets a *new* handle, a child of the sender's handle in the derivation tree (Task 19), and the message lets go of its reference.
  - **All or nothing, both sides.** Send checks every handle first and takes the references only when the message is surely queued: a refused send (bad handle, no TRANSFER, widening, full queue, closed peer) changes nothing. Receive puts all the handles in or none: not enough room in the receiver's table or limit is `HANDLE_E_LIMIT`, the message stays queued. A recv call without room for the handles (`channel_recv`, or a too small array) returns `CHANNEL_E_HANDLES` and leaves the message queued, so a handle is never lost by a careless recv.
  - **No loops (Task 15).** Channel X *holds* Y when a message queued at X carries an endpoint of Y. A send that would make X hold itself (`CHANNEL_E_SELF`: an end of the channel the message goes on) or close a longer loop (X holds Y, Y holds X; also through three or more channels) is refused with `CHANNEL_E_LOOP`. Why: references that go round in a circle are never freed, because nobody has a handle to them to close. Without loops, counting references is exact. The check (`carry_loop` in `channel.c`) walks the holds from the carried channels with interrupts off and looks for the channel the message is queued on. A message is only ever *added* by a send, and receiving or dropping one only removes edges, so the graph of holds stays a DAG. A received handle goes into a table, a root, which cannot make a loop.
  - **Nesting limit.** The same walk visits at most `CHANNEL_NEST_MAX` = 32 channels, then the send is refused too (`CHANNEL_E_LOOP`): a chain can be 33 channels deep in all. That bounds the time spent with interrupts off and the work of freeing a whole tree. The free itself is still iterative (the list above), so the kernel stack never depends on the depth.
  - **A message nobody can receive lets go.** When an endpoint closes, what is queued at it is dropped and its handles are released. That can free more endpoints, which drop more queues: a chain as long as a user builds. It is **not a recursion** (8 KiB kernel stack): closed endpoints go on a list and one caller drains it with preemption off (so it cannot be killed halfway). The channel is freed by the end that is drained last. Host test: a chain of 20 000 channels.
  - *(The cycle leak that was open after Task 13 is closed by the two bullets above.)*
- Task 14 test: `chantest` in the shell (QEMU; `make test-chan` runs it plus every earlier self-test): ping-pong between two real tasks (400 round trips one at a time, 600 with 12 in flight each way), overflow (exactly 16 fit, the 17th is FULL and harms nothing, directions are separate, FIFO and content across ~10 wrap-arounds, sizes 0..MAX, a blocked sender let in by one recv), closing the peer (queued messages first, then CLOSED, stable; send/recv_wait/send_wait do not block on it; a close wakes a blocked receiver and a blocked sender; a peer that dies is a close; heap back). 41 checks.
- Tests: `chanxfertest` in the shell (QEMU, Task 13: `make test-chanxfer` runs it plus all earlier self-tests as a regression) and `make test-chanxfer-host` (host, AddressSanitizer, 80 000 checks); `chanblocktest` in the shell (QEMU, Task 12: `make test-chanblock` runs it plus the earlier self-tests as a regression) and `channeltest` in the shell (QEMU) and `make test-channel-host` (the same code on a 64-bit host under AddressSanitizer, with a model check: 300 000 random operations against a plain model of the two queues).

## Spawn from ring 3 (Task 16)

`SYS_SPAWN(name, xfer[], n)` starts a program from ring 3, on top of the ELF loader (`process_spawn_elf_h`, a thin variant of `process_spawn_elf_limited`; both go through one `spawn_elf_core`). Code: `exec_spawn` in `arch/x86/src/exec.c`, `handle_adopt` in `handle.c`, the syscalls in `syscall.c`.

- **A new task holds exactly the handles it is given, nothing else.** Its table is empty before; there is no inheritance, no "same as the parent". `xfer[]` is an array of `{handle, rights}` (8 bytes, no pointers, `channel_xfer_t`), at most `SPAWN_HANDLES_MAX` = 8.
- **DECISION: a handle given to a new task is COPIED, not moved.** The spawner keeps its handle, the new task gets a new one for the same object with the rights asked for (same or fewer, `HANDLE_RIGHTS_SAME` = all). The rule is the one of Task 13 (a handle in a message), the check is the same code (`handle_carry_prepare`): the handle needs `TRANSFER` (`HANDLE_E_RIGHTS`), `rights` must be a subset (`HANDLE_E_WIDEN`). Why: one rule instead of two; a spawn can fail late (no memory, a bad second handle) and a copy has nothing to undo, so the spawner's table is unchanged on any error; a service can start workers and keep serving over the same channel. A move is "spawn, then `handle_close`" (`spawntest` does exactly this and then sees `CLOSED` when the child ends).
- **All or nothing.** All handles are checked before the file is read, and every carry holds a reference until the end (the spawner may be killed meanwhile). If the process cannot get its handles the new task is killed before it ever ran (it cannot run earlier: preemption is off from its creation to the end of the hand-over) and the spawn fails with `SPAWN_E_NOMEM`.
- **The handle numbers reach the child on its initial stack:** `[argc][argv…][NULL][nh][h0…]`. `nh` and `h[]` are 32-bit words, always present (`nh` = 0 when none); user code finds them behind argv's NULL (`start_handles(argc, argv, &n)` in `user/usys.h`), so the layout does not depend on the pointer width. `ustack_build` reserves them, `ustack_set_handles` fills them once the task exists.
- **Not a child.** The new task has no parent (`task_create_user`): nobody `wait()`s, its slot is freed when it ends. The spawner learns how it ended from the **result channel** (Task 17, below), not from a pid; a `TASK` handle for control comes with `SYS_SPAWN` since Task 18 (section *Task control*). `SYS_SPAWN` returns 0 or an error, no pid.
- **Program name** is still a file name (VFS, then ramfs, as `exec`): the only source of programs until the file service is a task (stage D); then the name is replaced by a handle to a file / image and nothing else in the call changes. No argv through spawn: arguments travel as messages over the handles.
- **Loader type.** `SYS_SPAWN` and `exec_spawn` know nothing about ELF: the one place that does is `spawn_elf_core` (`arch/x86`). The x86 port loads ELF32. For the small IoT ports (block G: Cortex-M, RV32 / no MMU) the loader must be the smallest thing that works there, a position-independent flat image with a tiny header, relocated at load time, not a full ELF reader; it replaces `elf.c` behind the same call, so spawn, handles and the syscall do not change.
- **Errors** (eax, int32; no two sets overlap): `HANDLE_E_*` (-1…-8) for the handles, `CHANNEL_E_*` (-16…-22), `SPAWN_E_NOENT -32, IMAGE -33, NOMEM -34, LIMIT -35, ARG -36`. A bad user pointer is `SPAWN_E_ARG`.
- **New syscalls with it** (the first ring-3 users of handles; numbers in `syscall.h`, wrappers in `user/usys.h`): `HANDLE_CLOSE 20`, `CHAN_CREATE 21`, `CHAN_SEND 22`, `CHAN_RECV 23` (`CALL_WAIT` flag = blocking, no timeout yet), `SPAWN 24` (since Task 17 with an optional result pointer in `esi`, since Task 18 an optional control pointer in `edi`), `HANDLE_DUP 25`, `TASK_CTL 26` (Task 18). Sending / receiving handles inside a message from ring 3 is not there yet (a message that carries handles stays queued, `CHANNEL_E_HANDLES`).
- **Tests:** `spawntest` in the shell (QEMU: the kernel spawns `spawnchild` with 1 and 3 handles, rights narrowed, refusals change nothing, then runs the ring-3 program `spawntest`, which does the same through `SYS_SPAWN` plus 40 spawns in a row; frames and heap must be back; `make test-spawn` adds all earlier self-tests as a regression) and `make test-spawn-host` (host, 64-bit, AddressSanitizer: `handle_adopt`). `ustack` has new host checks in `make test`.
- **Size:** about +2.5 KiB of kernel code (exec 0.5, process 0.2, syscall 1.4, handle 0.4), inside the 5 KiB sub-budget of spawn + result channel + task control.

## Result channel (Task 17)

How a spawned task ended comes to its spawner **as a message on a channel**, in place of `wait()`. Code: `channel_result_open / _post / _abort` in `channel.c`, `task_result_t` and `task_result_make` in `task.h`, the hook in `reap_zombies` (`task.c`), `task_set_result_end`, `spawn_elf_core` (`process.c`), `exec_spawn` (`exec.c`), `SYS_SPAWN` (`syscall.c`).

- **Asking for it.** `SYS_SPAWN(name, xfer[], n, result*)`: a pointer in `esi` (4-byte aligned) or 0. With a pointer, a handle to the **result channel** is written there on success; with 0 nothing changes (a spawner that does not care pays nothing). In the kernel: `exec_spawn(caller, name, xfer, n, handle_t* result)`.
- **Two ends, two different parties.** The spawner gets end 0 as a handle with `CHANNEL_RESULT_RIGHTS` = `RECV | DUP | TRANSFER` and **no `SEND`**: only the kernel can put a message on it, so a result cannot be forged, not even by its owner (rights only shrink, so there is no way to get `SEND` back: `HANDLE_E_WIDEN`). `DUP`/`TRANSFER` let the spawner hand the result to a supervisor, narrowed like any handle. The kernel holds end 1 as **one plain reference in the task's slot** (`task_t.result_end`): no handle, no table entry, nothing a task can touch.
- **The message** (`task_result_t`, 24 bytes, little-endian, identical on 32 and 64 bits, `static_assert`ed on both):

  | field | meaning |
  |---|---|
  | `uint32 kind` | `TASK_RESULT_EXITED` (1): it ended by itself; `TASK_RESULT_KILLED` (2): the kernel ended it |
  | `int32 code` | exit code 0..255 for EXITED, else 0 |
  | `int32 reason` | `TASK_SIG*` for KILLED (`SIGSEGV` 11, `SIGFPE` 8, `SIGILL` 4, `SIGKILL` 9), else 0. Reasons, not signals: nothing is delivered |
  | `uint32 fault_flags` | `TASK_FAULT_*` if it died of a page fault (`PAGE`, `WRITE`, `PROT`, `KERNEL` = the kernel touched the bad address for it), else 0 |
  | `uint64 fault_addr` | the faulting address when `fault_flags` has `PAGE`, else 0. 64 bits on every port, so a 64-bit port loses nothing; ring 3 reads it as two words (`usys.h`) so the layout does not depend on how a compiler aligns a `uint64` |

  It is made from the packed end of the task by one inline function (`task_result_make`); it is the only report of a death there is.
- **Exactly one message, then closed.** After the message the kernel lets go of its end: the next `recv` returns `CHANNEL_E_CLOSED` (queued messages first, the rule of every channel), also a blocking one: it does not wait for a second message. An open channel with no message yet is `CHANNEL_E_EMPTY`, so "not yet" and "never" are different answers.
- **When it is posted: at the reap, last.** The message is posted in `reap_zombies()`, *after* the kernel stack, the address space, the handle table and the slot are given back. So "I received the result" means "it is gone": frames, heap and the task slot are already back (`resulttest` checks this immediately, without waiting). Posting at the reap, not at the moment of death, is deliberate: the post must not allocate or fail, and the dying task is still on its own stack at the moment of death. Nothing is allocated at post time: the channel exists since the spawn and the message is copied into a slot of its queue.
- **Nobody sleeps through it.** A blocking `recv` is woken by the post (the usual wake-up of a queue). The case "the dying task was the last runnable one" is the one of Task 12: every task death wakes all channel waiters and a waiter reaps after each wake-up, which now also posts. A program that polls with a non-blocking `recv` and never yields would not see the result until somebody else reaped: the syscall layer reaps once and looks again when a non-blocking `recv` finds the queue empty.
- **All or nothing.** The handle costs the spawner one handle: `HANDLE_E_LIMIT` is decided **before any work** (`exec_spawn`, before the file is read). The handle is made in the same preemption-off section as the start-up handles, after them: if it cannot be made the new task is killed before it ever ran (`SPAWN_E_NOMEM`). `*result` is written only on success; every refusal (`HANDLE_E_*`, `SPAWN_E_*`, a bad or unaligned pointer = `SPAWN_E_ARG`) leaves no handle, no task, no byte of heap or frame behind (`resulttest` compares the task table, handle count, heap and frames).
- **Nothing leaks in any order.** The spawner may close the handle early (the post finds the peer closed, drops the message, the channel is freed by the last end to let go) or die before its child (its table closes the handle; the same). The child never holds an end, so it cannot keep the channel alive. The existing rule does the rest: a channel is freed when both ends are closed and drained. `make test-result-host` runs every order (post before / after the receiver closes or dies, hand-over to a third task, abort, limit, dead spawner) 20 000 times under AddressSanitizer: the heap must be exactly back each round.
- **What it does not do (yet).** It does not stop or kill the child: control is a `TASK` handle (Task 18). It does not report on a child *the spawner did not ask about* (no result pointer). The result goes to a handle, not to "the parent": a task can give the handle to someone else; who is told is the holder's choice.
- **Errors** are the ones already there: `HANDLE_E_LIMIT` (-2) for the result handle, `SPAWN_E_ARG` for the pointer, `CHANNEL_E_CLOSED` / `_EMPTY` on the channel, `HANDLE_E_RIGHTS` for a `SEND` on it. No new numbers.
- **User side** (`user/usys.h`): `sys_spawn_r(name, xfer, n, &handle)`, `sys_result_wait(handle, &task_result_t)`, `RESULT_EXITED / RESULT_KILLED`, `FAULT_*`, `E_HLIMIT`. The old `sys_spawn` is the same call with no result pointer. Test programs: `resultchild` (ends the way it is told), `resulttest` (ring 3).
- **Tests:** `resulttest` in the shell (QEMU: the kernel spawns `resultchild` with a result pointer and reads exit codes 0 / 7 / 42 / 255, a page fault by write and by read with the address, `SIGFPE`, `SIGILL`, a bad pointer given to the kernel (`KERNEL` flag), a `task_kill` (`SIGKILL`); one message then `CLOSED`; no `SEND`, no widening; no result yet = `EMPTY`; every refusal changes nothing; a full table = `HANDLE_E_LIMIT` before any work; the handle closed early; then the ring-3 program `resulttest`, which also checks two children at once, a busy poll, 30 spawns in a row and dies before a child of its own; frames, heap and tasks must be back) and `make test-result-host` (host, 64-bit, AddressSanitizer + UBSan, ~330 000 checks). `make test-result` runs `resulttest` and every earlier self-test.
- **Size:** about +1.3 KiB of kernel code (channel 0.46, task 0.24, syscall 0.24, process 0.18, exec 0.16), inside the 5 KiB sub-budget of spawn + result channel + task control (Task 16 used 2.5, Task 17 1.3; 1.2 KiB left for Task 18). See *Kernel size budget* for how the ceiling was kept.

### The write end for the child (Task 20.1, part of goal 20: the shell waits on the result channel; `task_waitpid` was removed in Task 21)

A task spawned **with** a result pointer now also holds a handle to the channel it is reported on. Sub-tasks 20.2-20.4 build on it (the shell reads the channel); this one only puts the handle there.

- **What the child gets.** The other end of the same channel (end 1), as a handle with `CHANNEL_RESULT_WRITE_RIGHTS` = `SEND` and nothing else: it can write to its parent, it cannot read the channel, `dup` it or pass it on (`HANDLE_E_RIGHTS`, tested from ring 3). The kernel keeps its own reference to end 1 as before, so the final `task_result_t` is still posted at the reap.
- **The fixed number.** Slot `HANDLE_FIXED_SLOT_RESULT` = 3 of the child's table, so the handle is `HANDLE_FIXED_RESULT` = `0x00010003` (`RESULT_HANDLE` in `usys.h`). Why not the bare 3: a handle is `generation << 16 | slot`, generation 0 is `HANDLE_INVALID`'s and never issued, and the slots of a table that was just made all start at generation 1, so (1, 3) is the same in every child. It is **not** in the start-up list (`start_handles`), and the listed handles never take slot 3: they fill 0, 1, 2, 4, 5 ... (the new `handle_install_at` takes the slot first, `handle_adopt` goes around it).
- **Only with a result pointer.** Without one there is no channel and no handle 3 (`sys_chan_send(RESULT_HANDLE, ...)` is `E_BAD`): a spawner that does not care pays nothing, as in Task 17. The handle lives in the child's own table: it costs the spawner nothing, the child one slot (its table must have at least 4: any limit of 4 or more, the default is 64).
- **Not in the derivation tree.** It is a root (no grant node, Task 19): a spawn with a result channel does not use the 128-node pool. So revoking the spawner's handle does not cut the child's write end; the child's end goes when the child's table dies.
- **What changes in the Task 17 promise.** "A result cannot be forged" held because nobody but the kernel could send. Now the child can send, so the guarantee is the weaker one that matters: **the kernel's message is always the last one** and it is followed by `CHANNEL_E_CLOSED`. If the child filled the 16-message queue, `channel_result_post` drops the oldest message to make room (only a message without handles, and ring 3 cannot send handles), so a child cannot keep its result from arriving. What the parent reads before the last message is the child's own words.
- **Order at spawn** (`spawn_elf_core`, one preemption-off section): result channel (the spawner's read end, the kernel's reference, the child's write end), then the listed handles, then the control handle. Any failure closes what was made for the spawner and kills the child unseen: `*result` / `*control` are written only on success.
- **Errors.** One new number, `HANDLE_E_BUSY` (-15), from `handle_install_at` (that slot is taken); the spawn code never shows it to ring 3.
- **Tests.** Host: `make test-result-host` (the number, BUSY / LIMIT / ARG / REF, rights, both orders of death and post, a child that fills the queue, the heap empty each time). QEMU: `resulttest` (new part "Task 20.1": a message written to handle 3 arrives before the kernel's, the probe from inside the child, eight listed handles around slot 3, no handle 3 without a result pointer, the flooding child).

### The death of a task with a result channel (Task 20.2, part of goal 20)

How a task dies is now recorded **once, in the form its reader wants**, in `terminate_locked()` (every way of dying ends there: `task_exit_code`, `task_die`, `task_die_fault`, `task_kill`, `TASK_CTL_KILL`).

- **With a result channel** (spawned with a result pointer, so it holds the write end as handle 3): the kernel packs the exit code / the reason and the page fault into a `task_result_t` *at the moment of death*, while everything is known. Nothing else is recorded, nobody is woken, and the slot goes `ZOMBIE` -> `UNUSED` at the reap.
- **Without one:** (Task 21) nothing at all is recorded; the slot is free at the reap. Until Task 21 such a task left an exit status for `task_waitpid`; that path is gone.
- **When the message is sent.** Packed at death, but posted at the **reap**, after the kernel stack, the frames and the handle table are given back. The order is deliberate: whoever reads the result may rely on "it is gone" (frames and slot back), which `resulttest` checks. Posting inside `task_die`, on the dying task's own stack, would break that promise and could run before the memory is free.
- **Through whose handle.** The kernel posts through **its own reference** to the channel's end 1, not through the handle 3 in the child's table. So a child that closes handle 3 (or whose table is already destroyed in the reap) cannot lose its result; the spawner sees an open channel with no message (`EMPTY`) until the task is reaped, then the message, then `CLOSED`. Closing handle 3 early does not signal "dead".
- **Memory.** (Task 21) The legacy half is gone: `task_t` holds only `result` (24 bytes) and no `exit_status`, `fault` or `parent`.
- **Tests.** Host: `make test-task-host` (part 7b on the real `task.c`: a result task is killed while task 0 sleeps: task 0 stays asleep, the message is packed at death, is not there before the reap and arrives after it, the slot is `UNUSED`; a task without a result channel leaves no record; a 64-bit fault address and an exit code survive packing; a reused slot has no leftover result end; the spawner gone before the death leaks nothing under ASan). QEMU: `resulttest` (new parts "20.2": the child closes its own handle 3 and the result still comes; a result task and a `sleeper` with its own result channel side by side) plus every earlier test, which now run through the new packing (faults with the address and the `KERNEL` flag, `SIGFPE`, `SIGILL`, `task_kill`).

### The test that closes goal 20 (Task 20.4: `spawn_chan_test`)

One in-kernel self-test (`selftest_x86.c`, shell command `spawn_chan_test`) for what the shell's `exec` has relied on since 20.3: **the code that arrives on the result channel is the right one for every way a program can end, and nothing of the spawn is left behind.** No kernel code and no user program were added: it uses `resultchild`, `crash` and `nop`, which are already in the image.

- **Part 1, the way `exec` starts programs** (`exec_spawn_args` with argv, a result pointer, no handles): `nop` (EXITED 0), `crash usage` (EXITED 2), and `crash div / ud / cli / null / unmapped / kernel / peek / stack` (KILLED with `SIGFPE` / `SIGILL` / `SIGSEGV`, the fault record of a page fault with its address and the `WRITE` / `KERNEL` bits; `#GP` and the others leave no record). The expected values are the ones `faulttest` already checks through `task_waitpid`, so the two paths must agree. The `PROT` bit is not part of what is promised and is not compared.
- **Part 2, every exit code** 0..255 through `resultchild`: kind, code, reason 0, no fault record; after each one the task table, frames, heap and my handle count are as before.
- **Part 3, six ways to end a spawn**, 8 rounds each: the result handle closed first (nobody left to read); the plain way; the message left **unread** in the queue and then the handle closed; killed through the control handle (`SIGKILL`); the control handle closed **first** (closing it does not end the task); the command channel end the spawner kept, which must see its peer closed as soon as the result is there (the child's own table is gone by then). After every round: tasks, frames, heap, my handles and the derivation pool (`rv_pool`) equal to the start.
- **Part 4, refused spawns** (bad handle, no such program, too many handles), 30 rounds: the error is the expected one, `*result` is not written, nothing is left.
- **On every spawn:** exactly ONE message of `sizeof(task_result_t)`, then `CHANNEL_E_CLOSED` (never a second message); the handle closes and is dead (`handle_lookup` fails); and the handle of the round before stays dead although its slot is in use again (the generation goes up on close).
- **Size.** Kernel: +131 bytes, the help line only (`make ksize` 62 986 -> 63 117, ceiling 69 632, target 65 536: both ok). The test is in `selftest_x86.o`, which `ksize` does not count: +2.5 KiB, and the image 152 400 -> 155 024 bytes (the bootloader's limit is 196 608). Its section headings are compiled in only with `TEST_MSGS=1`, as the rule of 20.3 says.
- **Run:** in the shell `spawn_chan_test` (about 300 spawns, a few seconds in QEMU); `make test-spawnchan` runs it and then every earlier self-test as the regression that closes goal 20 (`tests/qemu_cmd.py` learned the `_` key for the name). `task_waitpid` itself stays: the old self-tests (`faulttest`, `blocktest`, `sleeptest`...) still use it for tasks without a result channel.

## Task control (Task 18)

Stop, resume and kill go **through a handle**, not a pid. Code: `task_control` and the `STOPPED` state in `task.c`, the object in `handle.c` (`handle_task_open`, `handle_task_control`), the spawn side in `spawn_elf_core` (`process.c`) / `exec_spawn_ex` (`exec.c`), `SYS_TASK_CTL` and `edi` of `SYS_SPAWN` (`syscall.c`), `usys.h`, the shell jobs in `shell_x86.c`.

- **Getting a handle.** `SYS_SPAWN(name, xfer[], n, result*, control*)`: `edi` is a 4-byte-aligned pointer or 0 (callers that do not use it must pass 0: `sys_call4` in `usys.h` does). With a pointer, a `TASK` handle for the new task is written there, rights `CONTROL | DUP | TRANSFER`. In the kernel: `exec_spawn_ex(caller, name, xfer, n, result, control)`; `exec_spawn` stays as the call without it. A task has a handle to another task only if it was given one (spawn, `dup`, passing on): that is the whole authority to touch it.
- **The object** is a small record `{task_ref_t ref, refs}`: slot **and generation**, so a handle names one task for all time. When the task ends, or its slot goes to another task, the handle is not dangling: it answers `TASK_E_GONE` (-9) for ever and cannot touch the newcomer. Counted like every object (a handle, a message in flight), freed with the last reference. It holds no other object: no cycle.
- **Operations** (`SYS_TASK_CTL(handle, op)`, right `CONTROL`): `TASK_CTL_STOP 1`, `RESUME 2`, `KILL 3`. Errors in this order: `HANDLE_E_BAD` / `_TYPE` / `_RIGHTS` (the handle), then `HANDLE_E_ARG` (unknown op), `TASK_E_GONE` (ended), `TASK_E_SELF` (-11: the caller itself or task 0; a task has `exit`), `TASK_E_STATE` (-10: stop of a stopped task, resume of one that is not). Kill ends the task as the kernel's kill does (result `KILLED`, reason `SIGKILL`), also a stopped one.
- **What "stopped" is.** One state, `TASK_STOPPED`, and one flag, `task_t.stopped`. A READY task becomes `STOPPED` (out of the round-robin and of `task_count`). A task that is BLOCKED (sleeping, waiting on a channel, ...) **stays BLOCKED** with the flag set: its wait goes on, `ps` shows "stopped", and `unblock_locked()` (the one place where a task becomes runnable) turns it into `STOPPED` instead of READY when what it waits for happens. So nothing can run a stopped task: not a message, not the end of a sleep, not a closing peer. Resume clears the flag (STOPPED -> READY; a task still blocked just loses the hold). A running task is never flagged: stop of the caller is refused.
- **Orphans.** When the **last** `TASK` handle goes (every handle closed, every message that carried it received or dropped, its holders dead) and the task is still stopped, it is **resumed** (not killed): nobody could ever resume it, and "stop everything you can reach, then die" must not freeze other tasks for ever. A handle is the right to control, not ownership: closing it never ends a task.
- **Rights only shrink**, as for every handle: a dup with `CONTROL` only, with rights 0 (can be held and passed on, never used: `HANDLE_E_RIGHTS`), `HANDLE_E_WIDEN` for a right it does not have; a hand-over at spawn needs `TRANSFER`. This is the supervisor pattern: give a helper the victim's handle with `CONTROL` only.
- **All or nothing** as for the result channel: the spawner pays one handle per pointer, `HANDLE_E_LIMIT` is decided before any work, the control handle is made in the same preemption-off section first and closed again if the result channel then fails; `*control` is written only on success (a bad / unaligned pointer or the same place as `result` is `SPAWN_E_ARG`).
- **The shell.** `kill <pid>` is gone. `bg <prog>` spawns with both handles and keeps them in a 4-entry table; `jobs` reads the result channels; `stop %N`, `cont %N`, `kill %N` go through the task handle (`%N` is the index in that table, not a task id; `kill 3` prints the usage). `ps` still lists tasks and shows `stopped`. Task 20.3: `exec <file> [args]` starts the program with a result channel and sleeps in one blocking `channel_recv_wait` until the death packet arrives (no `task_waitpid`, no polling). The Esc key no longer kills a foreground program (a blocked shell cannot poll the keyboard); use `bg` + `kill %N`. `task_kill(int)` stays only as kernel-internal (old self-tests) and goes with `fork/wait` (Task 21). Built-in `spin` is a busy loop for `bg spin`.
- **Tests:** `make test-task-host` (host, 64-bit, AddressSanitizer: the **real `task.c`** over a fake HAL, fixed scenarios and 200 000 random steps against a model; invariants: `task_count` = number of READY/RUNNING, one RUNNING, STOPPED has the flag and no block reason, the round-robin never picks a stopped or blocked task), `make test-taskctl-host` (the object: rights, error order, stale / reused slot, dup, message in flight, last handle resumes, no leak in any order, random load), and in QEMU `ctltest` (`make test-ctl`: the kernel spawns `spin` and `resultchild` with a control handle: a stopped task's `cpu_ticks` stand still, a task stopped while it waits in `recv` or sleeps is not run by the message / the end of the sleep, rights and errors, ended = `TASK_E_GONE`, a stale handle misses the newcomer, the last handle resumes, the handle limit; then the ring-3 program `ctltest` does it through `SYS_TASK_CTL`, supervisor children included, and ends while a child is still stopped; tasks, frames, heap, handles must be back; every earlier test is run as a regression).
- **Size:** about +1.4 KiB of kernel code (task 0.43, handle 0.44, syscall 0.29, process 0.14, exec 0.10), 0.2 KiB over the 1.2 KiB that was left of the 5 KiB sub-budget (spawn + result + control = 5.2 KiB); plus about 0.8 KiB for the shell jobs and help text, which leave ring 0 with the shell. The live states are one run of the enum (`task_is_live`) and `unblock_locked` is not inlined, which saved 0.3 KiB.

## Revoke and the life of a handle (Task 19)

Code: the node pool and `handle_revoke / handle_dup_ttl / handle_expire_in / handle_remaining / handle_sweep` in `handle.c`, `channel_end_wake` in `channel.c`, `SYS_HANDLE_CTL` and `esi` of `SYS_HANDLE_DUP` in `syscall.c`, `usys.h`.

- **Who is made from whom.** A handle made from another is its *child*: by `dup`, by being carried in a message (the receiver's handle is a child of the sender's), by `SYS_SPAWN` (the new task's handles are children of the spawner's). Originals (`channel_create`, result and task handles) are roots. The tree **crosses tables**: this is what `derived_from` of Task 9 could not do.
- **`handle_revoke(h)`** kills every handle derived from `h`, in any table, to any depth. Each is closed as if its holder had called `handle_close`: slot and handle count come back, the object loses the reference, a peer reads `CHANNEL_E_CLOSED` if that was the last handle, a task asleep in a blocking call through it is woken (`channel_end_wake`) and gets `E_BAD`. `h` and its parents stay valid, and `h` can be derived from again afterwards: revoke takes back what was given, it is not a ban. It needs no right: whoever holds a handle may take back what was made from it. It returns how many handles of running tasks it closed.
- **Closing is not revoking.** Handles made from a closed handle live on, and a later revoke of *its* parent still reaches them (the closed handle's node stays as long as anything derived from it does).
- **A handle in a message** is dead too once revoked (or out of time): the receiver gets `HANDLE_INVALID` in its place, no slot is used, `nhandles` keeps its position meaning. Known price: until the message is received or dropped, the dead carry still holds its reference to the object (nobody can use it; a peer's `CLOSED` comes when the message goes). There is no registry of all channels to purge them eagerly, and adding one costs 2 pointers per channel.
- **A time limit.** `handle_dup_ttl(h, rights, ttl)`, `handle_expire_in(h, ttl)` give a handle a deadline in timer ticks. After it every use answers `HANDLE_E_EXPIRED` (-12) for good: lookup, dup, passing on, spawn. A child never outlives its parent (the life of a handle only shrinks, like its rights); `handle_expire_in` only shortens. **Nothing runs when the time is up**: the deadline is compared where the handle is used, so an idle device spends no power on it (a tickless kernel keeps this for free). An expired handle keeps its slot until it is closed (`handle_close` works on it) or the owner calls `handle_sweep` (closes all expired, returns how many). A task asleep in `recv` on an expiring handle is not woken by the deadline (there is no timeout yet, a later task); it finds `E_EXPIRED` when something wakes it.
- **How, in one paragraph.** One *node* per derived handle / carry, plus one for a handle something was made from, in a **static pool** of `HANDLE_GRANTS_MAX = 128` nodes (20 bytes on x86 = 2.5 KiB `.bss`; `-DHANDLE_GRANTS_MAX=32` for a tiny board): no heap, so nodes are taken and given back with interrupts off, and the memory of the feature is known at build time. A node is `{parent, epoch, parent's epoch when made, deadline, refs, depth}`. `revoke(h)` adds 1 to the epoch of `h`'s node: every child's remembered parent epoch is now stale, so every child, grandchild ... is dead; **one store, whatever the size of the tree**. A handle is alive when no node on the way up is stale or past its deadline (at most `HANDLE_GRANT_DEPTH_MAX = 8` nodes: a deeper derivation is `HANDLE_E_DEPTH` -14). Then the tables are swept once (preemption off, each entry freed in its own short section, objects released with interrupts on). An original nothing was made from has **no node and costs nothing**; lookup of it is as before.
- **Limits are errors for the one who asks.** Pool empty: `HANDLE_E_GRANTS` (-13) from dup, from a send with a handle, from spawn; the refusal changes nothing in the tables or the queue (`handle_grants_check(n)` asks for the whole call up front, 2 nodes per carry). The pool is shared; a per-task share is a later task if needed.
- **Layout:** `handle_entry_t` is as big as before (the node pointer took the place of `derived_from`); `handle_carry_t` grew by one pointer (+4 bytes x 128 carries = +0.5 KiB per channel on x86).
- **A race that revoke created, and its fix.** Another task can now free a handle's record while its owner is between two calls. Everything that reads a record does so inside the section that looked it up; `handle_task_control` did not, and now does.
- **Syscalls:** `SYS_HANDLE_CTL (27)`: `HCTL_REVOKE`, `HCTL_EXPIRE`, `HCTL_LEFT`, `HCTL_SWEEP`; `SYS_HANDLE_DUP` takes the time limit in `esi` (callers that do not use it pass 0: `usys.h` does). Errors: `E_EXPIRED -12`, `E_GRANTS -13`, `E_DEPTH -14`.
- **Tests:** `make test-revoke-host` (host, 64-bit, AddressSanitizer: fixed scenarios, then 64 000 random steps against an independent model that keeps every handle ever made and checks every live handle and the pool count after each step; mutation-checked: no epoch bump and a `>` instead of `>=` on the deadline both fail it) and in QEMU `revoketest` (real tasks, the real timer: revoke in one table, across tables through messages and a second hop, a handle in flight, "as if closed", time limits, depth, dying tasks; then the ring-3 program `revoketest` with `revokechild`, which includes a child asleep in `recv` being woken by a revoke; tasks, frames, heap and pool must be back).
- **Size:** `handle.c` grew by 1.2 KiB, `syscall.c` 0.3, `exec.c` 0.2 (the code of the feature, about 1.7 KiB at `-O2`). To keep the ceiling, `handle.c`, `channel.c` and `syscall.c` are built with `-Os` (control-path code; `channel.c` got 0.5 KiB smaller by itself, `syscall.c` 0.6). Net from the compiled C objects: about +0.7 KiB, i.e. roughly 69.4 KB against the 69 632 ceiling: **`make ksize` is the number to trust.**

## Handles in messages from ring 3 (Task 22.1)

The kernel could pass handles in messages since Task 13 (`channel_send_h` / `channel_recv_h`), but ring 3 could not use it: a message that carried handles stayed queued for `SYS_CHAN_RECV` (`CHANNEL_E_HANDLES`). The System Dispatcher (goal 22) has to hand channels to services and clients, so ring 3 gets the same calls. Code: `sys_chan_call_h` in `arch/x86/src/syscall.c`, wrappers `sys_chan_send_h` / `sys_chan_recv_h` in `user/usys.h`.

- **Two new calls, the old ones keep their ABI.** `SYS_CHAN_SEND_H` (28) and `SYS_CHAN_RECV_H` (29). `SYS_CHAN_SEND` / `SYS_CHAN_RECV` are unchanged, so every old program works as before.
- **Send:** `ebx` = handle, `ecx` = data, `edx` = len, `esi` = `flags | (nxfer << 8)`, `edi` = `channel_xfer_t[nxfer]`. The count lives in the flags word because the calls have only five argument registers (handle, data, len, flags, array and count are six values). Plain 32-bit words only, no pointer inside a structure: the same ABI on a 64-bit port.
- **Receive:** `edi` = `uint32_t out[1 + 4]`, 4-byte aligned: `out[0]` = how many handles came, `out[1..]` = their new numbers. `out[0]` is written on `E_HLIMIT` (so the caller knows how many slots to free) and is 0 on other errors.
- **Rules are the kernel's rules, not new ones:** a COPY (the sender keeps its handle), needs `TRANSFER`, the same rights or fewer, all or nothing on both sides, no loops, at most 4 handles. A full table is `E_HLIMIT` and the message STAYS queued.
- **Checked before any work:** `nxfer > 4` is `CHANNEL_E_HANDLES`, `len > 64` is `CHANNEL_E_SIZE`, a bad or unaligned pointer is `SPAWN_E_ARG`.
- **Tests:** QEMU `chanxfer_user` (`make test-chanxfer-user`): ring-3 `xfertest` with the child `xferchild` (copy not move, rights, widening, no TRANSFER, 4 handles in one message and 5 refused, refusals leave nothing queued, `E_SELF`, `E_LOOP`, ring 3 receives a handle, full table keeps the message queued, a send to a program that ended), then the kernel checks tasks, frames, heap and derivation nodes are back.

## System Dispatcher (Task 22, done: 22.1 - 22.4)

The first ring-3 task of the system (`user/dispatcher.c`). It is started with TWO handles and nothing else: it sees only what it was given. `[0]` is the control channel to the shell, `[1]` the **desk**, the clients' door (below). It starts the services of its static table (`echosvc`, `countsvc`) with `SYS_SPAWN`; each service gets its OWN channel to the dispatcher as its only handle, and the dispatcher keeps three handles per service (the channel, the result channel, the task handle), a table name -> handles. Until the shell is a ring-3 program (stage D) the kernel shell starts it: `dispatcher start`. The shell is its control client: it talks over the control channel, one message is up to 64 bytes, byte 0 is the operation (the full list is in the header comment of `user/dispatcher.c`).

- **Control operations (22.2):** `L` list (one `S` message per service: alive or ended, then `E`), `P i bytes` ping (the bytes go to service i, its answer comes back as `A`, or `X` with a code: 1 no such service, 2 it has ended, 3 send failed, 4 no answer in 3 s), `Q` stop (kill the services, answer `K`, exit 0). First message from it: `R n` (n services up) or `F i err` (the spawn of service i failed: it stops what did start and exits with 1).
- **Three ways to end, all clean:** `Q`; the shell closes its end of the control channel (the same as `Q` without the answer); the dispatcher is killed (its death closes its channels, and the services exit by themselves when their recv reports CLOSED, so nothing is left behind).
- **Test services:** `echosvc` answers every message with the same bytes, `countsvc` with 4 bytes: how many messages it has read **on that connection** (little-endian; every client has a count of its own).

### Asking for a service (Task 22.3)

A client says "give me service X" and the dispatcher makes it a channel to the service. After that the two talk directly and the dispatcher is out of the way.

```
client                         dispatcher                         service
  | make reply channel (r0,r1)     |                                  |
  |-- 'G' name + r1 (SEND) ------->| (desk)                           |
  |                                | find name; make a pair (s,c)     |
  |                                |-- 'C' + s (SEND|RECV) ---------->| (its dispatcher channel)
  |<-- 'O', i + c (SEND|RECV) -----| (reply channel)                  |
  |<================== c <---------- direct ----------> s ============>|
  |   unknown name: 'X', 0xFF, 1   (the error code is the answer)     |
```

- **The desk.** One channel, made by the shell. The dispatcher holds one end with RECV only; the shell keeps the other and gives each client a copy of it with **SEND only** (a spawn copies a handle, and copies of one end share its queue). So all clients come in through ONE queue, one wait covers all of them, and a client can neither read what another wrote nor say `Q` (that is the control channel, which only the shell has).
- **The request** is `'G'` + the service name (the rest of the message; the whole name must match) and carries ONE handle: the client's reply end (SEND). The answer comes on the reply channel: `'O', i` + the client's end of the new pair, or `'X', i, code` (1 no such service, `i` = 0xFF; 2 the service has ended; 3 could not connect: the service's queue is full or no channel could be made; 5 not a request). An unknown name is an ANSWER with an error code, never silence.
- **Rights narrowed on both ends: `SEND | RECV`, no `DUP`, no `TRANSFER`.** Neither the client nor the service can copy its end or hand it to anybody else, so the pair stays what the dispatcher made it. (The pair is made with every right in the dispatcher's table and the carry names the two.)
- **The dispatcher keeps nothing per client.** Every handle of a request (the reply end, both ends of the pair) is closed before the next message is read, so a flood of clients costs it no table space; the connection lives in the two end points.
- **A client that dies in the middle of a request costs nothing.** Before the answer: the answer finds the reply channel closed, the dispatcher closes both ends it just made, the service sees its end closed. After the connection: the service reads CLOSED on that end and frees the place.
- **The service side (`user/svc.h`, shared by both test services).** One loop, one blocking call: the wait is over the dispatcher channel and one channel per client (at most `SVC_MAXC` = 6). A message WITH a handle on the dispatcher channel is a new client; one without is the shell's ping. Sends never wait (a client that does not read must not stop the service). The table full: the new end is closed at once, which the client sees as CLOSED = "busy".
- **Why a new call: `SYS_CHAN_WAIT` (30), `channel_wait_any`.** A service has several channels and `recv` waits for one. The ways without a new call were polling (a timer wake-up every tick for as long as a client is connected, bad for a battery) or serving one client at a time. `sys_chan_wait(handles, n)` blocks until a recv on one of up to 8 handles would not block (a message is queued, or the peer is closed) and returns the INDEX of the first such handle; it receives nothing. The task sleeps as `TASK_BLOCK_CHAN_RECV` with key 0, and `task_wake_blocked` lets key 0 match every wake of that reason, so no channel code changed: the cost is one more compare in the wake loop and about 0.4 KiB of kernel (`channel.c` +191 B, `syscall.c` +165 B, `task.c` +12 B). A wake for a channel that is not in the list costs one look and the task sleeps again. A bad handle or one without RECV counts as ready (the recv says why), so a single broken entry cannot freeze the wait. No timeout, as with `recv`.
- **Tests:** host `make test-channel-host` (`channel_wait_any`: blocks with the right reason and key, spurious wake-ups, index order, closed peer, bad handle, bad arguments) and QEMU `dispatcher_test` run 4 (`make test-dispatcher`): `svcclient` with echo and count, an unknown / empty / prefix / too long name (the answer is code 1), three clients at once (two on count, each sees its own 1..20), clients that die on purpose after the request and after their first message, 20 clients one after another (more than the 6 places: the dead ones leave nothing), the shell's ping after all that, then stop with tasks, frames, heap, derivation nodes and handles back. By hand: `dispatcher start`, `dispatcher client echo 3`, `dispatcher client count 5`, `dispatcher client nope`, `dispatcher client echo 3 die1`, `dispatcher client count 3 die2`, `dispatcher list`, `dispatcher stop`.
- **Not tested:** the "busy" path (a 7th client at once): it needs seven clients alive together, more tasks than the test table has room for. It is a one-line branch (`close` of the new end).

### Supervision (Task 22.4)

A service that falls is restarted by the dispatcher, not by the kernel and not by the shell.

- **One wait over everything.** The dispatcher waits (`sys_chan_wait`, no CPU, no timer) on the shell's control channel, on the **result channel of every running service**, and on the desk, in that order: the shell first, the supervision before the clients, so a flood of clients cannot delay a restart. At most 1 + `NSVC` + 1 handles, far below the 8 the call takes.
- **What it does with a result (`reap`).** A service that ends, by exit, a fault or a kill, posts its result (Task 17). The dispatcher reads it, records how it ended (the LAST end only: kind, and the exit code or the death reason), closes the service's three handles, writes one line on the console (`dispatcher: service echo ended (killed, reason 9): restarted 1/3`) and starts the service again with a NEW channel. The same code runs when a control request or a client request touches a service whose result is waiting (`alive` calls `reap`), so no answer is built from stale state.
- **The limit: `MAX_RESTARTS` = 3, in all, for the life of the dispatcher.** The fourth death is final: the service stays in the table as `ended` (`'L'` says so, a ping is refused with code 2, a client that asks for it is answered `'X', i, 2`), the others go on. A failed restart (no memory, no slot) counts as an attempt and leaves the service down. There is no time window, so a service that falls once a day is given up after three days; that is the next thing to decide (a window, or an `up for a minute = forgiven` rule) when a real service exists.
- **The clients of the service that fell see CLOSED.** The service held the other end of each of their channels; its death closes them, and `recv` on the client's end reads CLOSED. Nothing in the dispatcher is involved, and nothing is kept for them. A client that asks AFTER the restart is connected to the new instance. A client does not have to ask again for the same service by name if the dispatcher is restarted too: that is up to the client.
- **Two more control operations** (the shell only): `'T'` status (per service: running, restarts, last kind, last code, clients served; then `'E'`) and `'D', i` down (kill service i; the supervision restarts it). `D` exists so the shell, and the test, can make a service fall without holding its task handle: only the dispatcher has it.
- **Closing the dispatcher still leaves nothing behind.** Killed, `Q`, or the control channel closed: the services, including the restarted ones, read CLOSED on their channel to it and exit (their loop in `svc.h`), or are killed by `Q`/control-closed (`stop_all`).
- **Tests:** QEMU `dispatcher_super` (`make test-dispatcher` runs it after `dispatcher_test`): echo killed and restarted (status, the shell's ping, a new client on the new instance, count untouched); a `svcclient ... watch` connected when its service is killed reads CLOSED and ends with 0; the third restart works and the fourth death is final (list, ping, a client's answer, a second `D` refused with code 2, count still serves); `Q` kills only what runs; then the dispatcher is killed with restarted services behind it and tasks, frames, heap, derivation nodes and handles are back. By hand: `dispatcher start`, `dispatcher status`, `dispatcher kill 0`, `dispatcher status` (restarts 1/3, last end killed), `dispatcher client echo 3`, `dispatcher kill 0` three more times, `dispatcher status` (ENDED), `dispatcher client echo` (refused, code 2), `dispatcher stop`.
- **Not done, on purpose:** the dispatcher itself is not supervised (nothing above it exists until stage D); a hung service that still reads its channel is not noticed (there is no heartbeat); restart policy is per table row, not configurable.

## Devices as handles (Task 23.1)

A driver that runs in ring 3 needs what the CPU keeps from it: the memory a device lives in and the device's I/O ports. Both are capabilities, like channels and tasks (`kernel/include/device.h`, `kernel/src/device.c`).

- **Two handle types.** `HANDLE_TYPE_MEMREG`: a range of physical memory that is NOT RAM; `HANDLE_TYPE_IOPORT`: a range of I/O ports. The record behind a handle (`dev_obj_t`: first address or port, length) is counted by the handles that hold it and freed with the last one; the device itself is never touched by that.
- **No new right bits.** The rights say which way the data goes: `RECV` = read (a MEMREG mapping is readable, a port is IN), `SEND` = write (the mapping is also writable, a port is OUT). `DUP` and `TRANSFER` as everywhere: the console service will get neither, so it cannot hand the screen on.
- **Only the kernel makes them** (`device_grant_mem`, `device_grant_io`); there is no call that makes one out of nothing. The first holder (for now the shell, later the dispatcher) gives copies on with narrower rights through `SYS_SPAWN` or a message. The HAL decides what may be handed out: on x86 never a range that overlaps RAM (any `MMAP_TYPE_USABLE` entry, so never the kernel or a frame of the allocator), nothing above 4 GiB, nothing that is not page aligned; and never the ports the kernel keeps for itself (PIC, PIT, keyboard controller 0x60-0x64, COM1, PCI config 0xCF8-0xCFF). Everything else is `DEVICE_E_DENIED` (-43).
- **`SYS_DEV_MAP` (31)** puts the range of the handle into the address space of the calling task, in a window of the user area (`0x70000000`, 1024 pages = one page table) and returns the address and length as two 64-bit values written as four 32-bit words (so the call is the same on a 64-bit port). The same range with the same rights again is the SAME address; a read-only handle for the same frames gets another address. The pages are uncached and carry a software bit, `PAGE_DEVICE`: `paging_destroy_directory` does not free their frames and `paging_count_frames` does not count them. Without that bit the VGA frame would go into the frame allocator when the task dies. The mapping costs the task nothing but the page table (one frame, counted).
- **`SYS_DEV_IO` (32)** reads or writes ONE byte of a port, and only inside the range of the handle (`DEVICE_E_RANGE` -40). One syscall per access, not a TSS permission bitmap: a driver touches ports rarely (the VGA cursor: two writes per update) and a syscall is a check the kernel can make for every single access. Wider accesses (16/32 bit) come with the first driver that needs them.
- **Tests:** QEMU `dev_test` (`make test-dev`, then every earlier test): the page tables (one frame, not two; the same address again; read-only another address; destroying the directory does NOT free the device frame), what may be handed out (RAM, unaligned, empty, wrapping, above 4 GiB, keyboard, PIC, timer, COM1 also inside a wider range, PCI, bad port ranges), the ring-3 `devtest` (the mapping, the rights, the refusals, the ports) leaves a marker in the VGA memory that the kernel finds (in the SECOND page of the text buffer, 0xB9000, which the screen does not show: a visible cell is overwritten or scrolled by the kernel's own output during the test), `devfault ro` (a write to a read-only mapping) and `devfault none` (a write where nothing is mapped) must both be KILLED by SIGSEGV; then tasks, frames, heap, derivation nodes and handles are back.
- **Not done, on purpose:** interrupts for a ring-3 driver (the keyboard service will need them: a later task), DMA, unmapping (a mapping lives until the task dies), and a per-device claim so two tasks cannot both own the cursor ports (the dispatcher will hold the only handles).

## The console service: the driver (Task 23.2)

`consoled` (`user/consoled.c`, protocol in `user/console.h`) is all of `vga.c` in ring 3: put a character, new line, backspace, colours, clear, scroll, the history and the view into it, the hardware cursor.

- **Starts with three handles:** `[0]` MEMREG (the text buffer, SEND|RECV, no DUP/TRANSFER), `[1]` IOPORT 0x3D4..0x3D5 (SEND only: it never reads a port), `[2]` a channel to its client. It touches neither screen nor cursor before the first message, sleeps in ONE blocking recv (no CPU idle), exits 0 when the peer closes, 7 / 9 / 8 for missing handles / unmappable or too small buffer / broken channel.
- **Protocol** (message 1..64 bytes, byte 0 = opcode): `W text` (0..63 bytes: `\n \r \b` handled, the rest is a character), `C` clear screen + history, `A fg bg` (fg&15, bg&7), `M row col` (cut to the last cell), `G` state (the only one with an answer: 20 bytes, see `console.h`), `V lo hi` signed 16-bit lines back in the history (0 = live). A message that is none of these is *bad*: nothing happens, the `bad` counter in the state goes up.
- **Decisions for small devices.** Only `G` is answered, and the answer is dropped when the client's queue is full: the driver never waits for a client. The hardware cursor is moved once per message, not once per character (4 syscalls instead of up to 252). History is `CON_HISTORY` = 100 lines (16 KB of `.bss`; the kernel console had 300 = 48 KB); one number, `-DCON_HISTORY=`, and `G` reports it so a client does not have to know. Driver 1.7 KB of code.
- **Changed on the way from `vga.c`.** A write while the history is shown first returns to the live screen (the old code wrote into the history window). `V` towards live when already live does nothing (the old code put back an old copy of the screen). Out-of-range values are cut exactly as before.
- **Tests.** `consoled_test` in QEMU (`make test-consoled`): a ring-3 client (`conclient`) plays 17 steps and after each the kernel reads the video page (0xB9000, not shown on screen) and the cursor registers and compares them with the hand-written expectation in `kernel/arch/x86/include/conexpect.h`: text, new line, backspace, colours, cursor and its clamping, a 200-character line, 26 lines in a row (2 scrolls), the history view (back, forward, live, a write while back), bad messages (9, nothing changes), a history that wraps round, clear while the view is back. Then consoled is killed under a flood (result KILLED/SIGKILL, the client reads CLOSED), the kernel talks to an idle consoled and kills it, and tasks, frames (the VGA frame stays out of the allocator), heap, nodes and handles are back. `make test-console-host` runs the SAME programs and checker on the host with a simulated kernel (ASan/UBSan, history 10/30/100/300, damaged screens must be caught).
- **Not done, on purpose (23.3+):** several clients on one driver (the dispatcher will give each a channel), taking over the screen from the kernel console, keyboard input.

## The console service in the dispatcher (Task 23.3)

`consoled` is the THIRD service of the System Dispatcher's table (`console`, program `consoled`). A client asks for it by name exactly as for `echo` and `count` (22.3: `'G' console` on the desk with a reply channel), gets its OWN channel and talks the protocol of `console.h` on it.

- **The dispatcher holds the devices.** It is started with FOUR handles: control, desk, `[2]` the screen's MEMREG and `[3]` the cursor ports' IOPORT (both with TRANSFER, no DUP). Every start of `console`, the first and each restart, gives the new instance copies with narrower rights (screen SEND|RECV, ports SEND) in front of its channel: `[0]` screen, `[1]` ports, `[2]` channel to the dispatcher. Without the two handles the dispatcher still runs but `console` does not start (`F`). The clients never see a device.
- **Many clients, one message at a time.** consoled waits (one `sys_chan_wait`, no CPU idle) on the dispatcher's channel and up to `CON_MAXC` = 6 client channels. A message WITH a handle on the dispatcher's channel is a new client (a full table closes the end at once: the client reads CLOSED); WITHOUT one it is an ordinary protocol message, so the dispatcher's ping `P 2 G` returns the state. A message is done completely before the next is looked at, so what two clients write never mixes inside a message (a client that wants a line whole sends it as ONE `W`). The waiting list turns round after every served client: a client with a queue of messages cannot starve one with a single message. A client that leaves, or sends a handle, loses its place. The cursor and the colour are ONE for all clients, like a terminal's.
- **A restart keeps the screen AND the cursor.** The video memory outlives the driver; so does the state that matters: cursor, colour, scroll counter and "history shown" are kept in the 24 spare bytes behind the 4000 of the screen, in two blocks with a check word each (the newer valid one wins, a write goes to the other block, so a kill in the middle of a save leaves the older one: at worst one message behind, never garbage). A new instance reads them, does not touch the hardware cursor to take over, and goes on where the dead one stopped. The history (RAM) is lost. A buffer without the spare bytes works as before, from (0,0).
- **Supervision as in 22.4.** `console` killed (`dispatcher kill 2`) is read, noted and started again with a new channel (3 restarts in all); its clients read CLOSED; a client that asks again is connected to the new instance (until the dispatcher has seen the end it is told 2 "ended" or 3 "could not connect": it waits and asks again). `user/conwrite.c` is such a client: it writes numbered lines, asks for the state after each (the answer proves the line was done), and after CLOSED asks again and writes the SAME line again, so a line may be on the screen twice but is never lost.
- **Tests.** QEMU `console_test` (`make test-console`; then `consoled_test` and every earlier test; `dispatcher_test` / `dispatcher_super` now expect THREE services): two clients write at the same time, every line whole and once; two more are writing when the console is killed, it is restarted, they reconnect and go on: the first lines are untouched, every new line is there, the cursor continued (no scroll, on the row after the last line). The kernel reads the hidden page (0xB9000) and prints nothing while the clients run. Host: `make test-console-host` now also runs three clients on one driver, the fairness case, the full table, a client that sends a handle, and a driver started again in the same video memory (newest block damaged -> the older one, both damaged -> cold start, a buffer without spare bytes).
- **By hand:** `dispatcher start`, `dispatcher list` (3 services), `dispatcher write A 5`, `dispatcher write B 5`, `dispatcher screen` (the console writes the second text page, which is not on the screen: `screen` prints its rows), `dispatcher ping 2 G` (the state), `dispatcher kill 2`, `dispatcher status` (restarts 1/3), `dispatcher write C 5`, `dispatcher screen` (A and B still there, C below them), `dispatcher stop`.
- **The image outgrew the bootloader (202 992 > 196 608 bytes).** Tasks 21-23 had brought it from 149.5 KB back to the 192 KiB wall (the in-kernel self-tests are about 90 KB of it, the embedded test programs 46 KB). `boot.asm` now reads a FOURTH piece (LBA 385 = cylinder 10, head 1, sector 8, into 0x40000), cap 256 KiB; the real cure is still to get the self-tests out of ring 0.
- **No more RWX segment.** `kernel/arch/x86/link.ld` now has two loadable segments, code + constants (R X) and data + bss (R W, on its own page), instead of one that was readable, writable AND executable (the linker warned about it, and the first page tables with NX would refuse it). The flat image grows by the padding between them, under 4 KB.
- **Not done, on purpose:** the kernel console (`vga.c`) still exists and writes the first page; taking the visible screen over from it, and the keyboard, are the next steps of goal 23. Per-client colour and cursor.

## Interrupts as handles (Task 24.1)

A driver in ring 3 cannot take an interrupt, so it gets a capability for ONE line: `HANDLE_TYPE_IRQ`, made by the kernel only (`device_grant_irq`, the same family as `device_grant_mem` / `_io`), handed on like any handle (spawn, message, dup with fewer rights). It is the mechanism every later driver needs (the keyboard service in 24.2-24.3, then mouse, serial, disk), so it is generic and knows nothing about any device.

- **What the kernel does on an interrupt:** the HAL entry calls `device_irq_fire(ctx)` with interrupts off: the line is MASKED (a device that keeps asking cannot flood the machine), the object counts it (`dev_obj_t.len`, saturating at 2^31 - 1) and wakes the tasks asleep in `SYS_IRQ_WAIT` (`TASK_BLOCK_IRQ`). It never touches the device.
- **`SYS_IRQ_WAIT` (33, needs RECV):** blocks with no CPU and no polling until the count is not 0, returns it (>= 1) and sets it to 0; the check and the block are one section with interrupts off, so an interrupt cannot slip between. **`SYS_IRQ_ACK` (34, needs SEND):** the driver has served the device: the line is unmasked. The two rights keep their meaning ("which way the information goes").
- **One owner per line.** A second grant is `DEVICE_E_DENIED` until the last handle to the first is gone (close, revoke of the derived ones, the holder's death: the handles of a dead task are closed when it is reaped). Then the line goes back to the kernel: masked, no handler. A message in flight holds the line too.
- **Which lines (x86, `hal_irqline_claim`):** 3, 5, 6, 9, 10, 11. Never 0 (timer), 1 (keyboard: the kernel's until 24.2), 2 (cascade), 4 (COM1, the debug port), 7 and 15 (can be spurious), 8 (RTC), 12 (mouse), 13, 14 (disk).
- **HAL:** `hal_irqline_claim / _release / _mask / _unmask` (`hal.h`), so ARM (GIC) and later ports implement the same four. `pic_mask_irq` is new. Revoke wakes a sleeper (`object_wake`): it looks again and gets `HANDLE_E_BAD`.
- **Tests:** host `make test-irq-host` (78 checks: lines, one owner, count, saturation, rights, expiry, revoke while asleep, last handle, a handle in a message, the holder's death, no leak, with a fake interrupt controller). QEMU `irq_test` (`make test-irq`, then every earlier test): the kernel raises line 5 ITSELF with `int $0x25` (the entry runs as for a real interrupt; a masked line can still be raised, which is how three pile up), reads the mask from the PIC, and the ring-3 `irqtest` sleeps in wait: woken by one interrupt, three counted, masked while it serves, its ack unmasks, revoked while asleep (E_BAD, also for the handles made from it), killed while asleep, the only holder killed (the line comes back by itself); tasks, frames, heap, nodes and handles are back.
- **Size:** kernel +920 bytes (device 490, hal 205, syscall 126, handle 62, pic 37). The ceiling was raised by 1 KiB (69 632 -> 70 656) FOR NOW: it goes back when the shell and `bench` leave ring 0 in stage D, and `keyboard.c` (~0.8 KiB) leaves when the shell input is the service (24.3 / stage D).
- **Not done, on purpose:** the keyboard line (24.2 releases it from the kernel's `keyboard.c`), level-triggered lines with a shared owner, timeouts on `SYS_IRQ_WAIT` (a driver that wants one waits on a channel with a timer task), the ARM port of the four HAL calls.

## The system's console is the service (Task 23.4)

From the moment the shell starts, every `kprintf`, every `SYS_WRITE` and every key echo of the ring-0 shell goes to `consoled` on the REAL screen (0xB8000); `vga.c` is the emergency path. `hal_console_late_init()` (called by `kernel_main` just before `shell_run`) starts `consoled` straight from the kernel with three handles (the screen, the cursor ports, a channel) and keeps the kernel's end of the channel in task 0's table (`kernel/arch/x86/src/consvc.c`). Early boot, a panic, a service that does not start or dies: the kernel writes the screen itself. No dispatcher is needed for this (the dispatcher keeps its own `console` instance on the hidden page for its tests; at stage D the system console moves under it).

- **Decision: a mirror in the kernel, not a shared state page.** The ring-0 shell reads the cursor and the scroll counter right after it writes (prompt, redraw of the line, "did someone else print?"). A page the driver fills is always behind the shell; "wait until the driver caught up" is a request to the service. The mirror (row, column, colour, scroll counter, history length, view) is the same few lines of cursor arithmetic as `vga.c`/`consoled`, needs no page and no mapping, and is exact at once. It is right while the kernel is the only writer of this screen. The same state is what the emergency path needs: when the service goes, it is handed to `vga.c` and the output goes on where it was. `conswitch_test` compares it with the driver's own `G` answer.
- **Output path.** `hal_console_*` -> mirror + a 1 KiB ring of records (text gathered into `W` records of up to 63 bytes; colour, move, clear, view, put-at are their own small records) -> `channel_send` into the driver's 16-slot queue. The ring is drained at the end of every `kprintf` (`hal_console_flush`), before the shell sleeps for a key and by the timer tick. A task that may wait (interrupts on, no `preempt_disable`) yields to the driver while the channel is full, for at most one second; then the service counts as stalled and output is dropped (counted) until it moves again. Interrupts, faults and preempt-off sections never wait. New protocol operation `P row col ch` (put one cell, cursor stays) for the line editor's `put_at`, also in the last cell without scrolling.
- **Death.** A send that finds the channel closed, or the timer tick finding the driver's result, makes the kernel take the screen back (`falls`++); one line `[!!] console service ended` is printed once. What was not delivered yet (the ring, the channel's queue) is lost, a scroll in progress can leave a torn screen; the cursor goes on from the mirror. `console on` starts a new driver (it is not restarted by itself: a console that keeps falling must not loop). A panic calls `hal_console_emergency()` first: interrupts off, direct, no task is touched.
- **For small devices.** `vga.c` lost the 300-line scrollback (48 KB `.bss`, live-screen backup, PageUp code, about 0.6 KB of code): the history lives in `consoled` (100 lines, one number). PageUp/PageDown work only while the service runs. Boot starts ONE task for the screen, not the dispatcher with echo and count. Cost: one 1 KiB ring and 3 handles of task 0; kernel code +1.0 KiB (`make ksize` 69448 of 69632; the diagnostics are in `consvc_dbg.c`, not counted, like the self-tests).
- **Tests.** `make test-conswitch`: `conswitch_test` (the driver STOPPED = a printed line is not on the screen, resumed = it is; mirror = driver after text, a 70-line flood, put-at incl. the last cell, the history view and a write while it is shown; a ring-3 program through `SYS_WRITE`; the driver KILLED in the middle of 100 lines -> the kernel takes the screen back, a line is on the screen at once, the scroll counter does not go back, 30 more lines scroll by themselves; `console on`; nothing left behind), then all of `test-dispatcher`'s regression, then `conswitch_test` again (messages sent grew by at least 50: the regression went through the service). By hand: `console` (status), `console kill`, `console on`, `console off`.
- **Not done (on purpose):** keyboard input, moving the shell to ring 3 (stage D), putting the system console under the dispatcher, restarting the console automatically.

## Task 21: fork and wait are gone (2026-10-22)

QlsOS is not a UNIX and now has nothing left of its process model: a task is made by `spawn` with a list of handles, ends with a message on a channel, and nobody is its parent. Done in six steps, each closed with a QEMU run:

| Step | What went |
|---|---|
| 21.1 | `SYS_FORK` (6) and `SYS_WAITPID` (8) leave the syscall table, `syscall.h` and `user/usys.h` (`sys_fork`, `sys_waitpid`, `sys_wait`, `WNOHANG`). An unknown number now answers `-1` (it used to hand its own number back, which looked like a pid). `crash gone` checks it from ring3. |
| 21.2 | `process_fork`, `task_fork`, `paging_clone_directory`, and everything that only they used: `hal_new_fork_stack` (x86, ARM, the host stub), `fork_return` in `isr.asm`, the `fork_frame` branch of `create_task`. |
| 21.3 | `forktest` and `waittest` deleted with their entries in the `Makefile`, `userbins.S`, `exec.c` and `help`. `blocktest` rewritten: one task sleeps (`SLEEP`), another waits on a channel (`CHAN_RECV`); spurious wake-ups, refused wake-ups, kill, heap back. |
| 21.4a | All self-tests and the bench moved from `exec_file` + `task_waitpid` to `exec_start` (no parent; optional result channel) + `exec_poll` / `exec_wait` (the result message). |
| 21.4b | `task_waitpid(_ex)`, `TASK_BLOCK_CHILD`, `wake_parent`, `TASK_WNOHANG`, `TASK_NO_PARENT`, the `EXITED` state, `exit_status` / `fault` / `parent` in `task_t`, `task_info_t.parent` (and the parent column of `ps`), `task_create_user_child`, `process_spawn_elf(_limited)`, `exec_file(_limited)`, the POSIX status macros in `usys.h`. The host test was rewritten for the new model. |

- **The rules now.** A task has no parent. A finished task gives back its slot at the reap, always. How it ended is told only by the result channel, if it has one; without one nobody is told. Blocking has three reasons: `SLEEP`, `CHAN_RECV`, `CHAN_SEND`.
- **For the kernel's own code** (self-tests, bench): `exec_start(name, argc, argv, limits, &result)`; `result == NULL` means fire and forget (watch `task_ref_valid`).
- **Why.** A parent that must collect its children is a global rule hidden in every process (zombies, re-parenting, a waiting parent that must be woken). A result channel is an ordinary handle: it can be passed on, closed, waited on with everything else, and a task that does not want to know does not pay for it.
- **Tests.** Host: all nine `make test-*-host`, `make check64`. QEMU: `blocktest`, `crash gone`, and every earlier self-test on the new start-up path.

## Syscall surface (target ≤ 16 calls)

`handle_close, handle_dup, chan_create, chan_send, chan_recv, task_spawn, task_control, task_sleep, task_yield, task_exit, debug_write` plus a few for memory limits. `fork`, `waitpid`, `kill` and `signal` are gone (`kill` in Task 1, `fork` and `waitpid` in Task 21). Each call takes only integers of fixed width and `uintptr_t` pointers.

## Architecture-dependent places (Task 8c)

**Shared code** is `kernel/src/`, `kernel/include/` and `kernel/hal/`. It never includes a header from `kernel/arch/`, has no `#if defined(__i386__)`-style branches, and reaches the CPU only through `kernel/hal/hal.h`. Everything that knows the CPU lives in `kernel/arch/<arch>/` (x86 today, ARM as a stub; the other targets are in *Platforms after v1.0.0*) or in `boot/`. `make check64` enforces both rules (see the last section). The five places a port to x86-64 has to rewrite, and nothing else:

| Place | Files today (x86) | What shared code sees | What the 64-bit port changes |
|---|---|---|---|
| **Paging** | `arch/x86/src/paging.c`, `pmm.c`, `mmap.c` (+ headers `paging.h`, `pmm.h`, `mmap.h`); page-fault entry in `isr.asm`/`idt.c`; user layout (`USER_BASE`, `USER_LIMIT`, `ELF_STACK_BOTTOM`) in `process.h` | `hal_switch_address_space(void*)`, `hal_destroy_address_space`, `hal_address_space_frames`; the task creation calls take the address space as an opaque `void* page_dir`; frame counts are `size_t` | 4-level tables instead of 2-level 4 KiB pages, 64-bit physical addresses (no PAE trick), a new user layout, the canonical-address check in the fault record |
| **Context switch** | `arch/x86/src/context_switch.asm`, `hal_new_task_stack` / `hal_irq_save` / `hal_irq_restore` / `hal_set_kernel_stack` in `hal_x86.c`, TSS in `gdt.c` | `hal_context_switch(uintptr_t*, uintptr_t)` and the others above; the saved SP is an opaque `uintptr_t` | callee-saved set `rbx rbp r12-r15` instead of `ebx esi edi ebp`, 8-byte stack slots, `RFLAGS`, `TSS.rsp0`. `hal_new_fork_stack` disappears with `fork` (Task 21) |
| **Syscall entry** | `arch/x86/src/isr.asm` (vector `int 0x80`, register frame), `syscall.c` / `syscall.h` (reads the frame: the numbers and arguments are in registers), `usermode.asm` (`hal_enter_usermode`), `userprog.asm`, `gdt.c` selectors | the dispatcher gets already-decoded integers of fixed width; `hal_enter_usermode(uintptr_t entry, uintptr_t stack)` | `syscall`/`sysret` (or `int 0x80` first), a 64-bit frame, other argument registers. The ABI of the syscalls themselves (rule 3 above) does not change |
| **ELF loader** | `arch/x86/src/elf.c`, `process.c`, `ustack.c`, `exec.c` (+ `elf.h`, `process.h`, `ustack.h`, `exec.h`); the user programs in `userbins.S` | `process_spawn_elf_h` / `exec_start` / `exec_spawn_args` as a *task* spawn; `elf_*` and `exec_read_file` sizes are `size_t` | the loader reads **ELF32 (EM_386)**; the 64-bit port adds an ELF64 / EM_X86_64 loader beside it. Shared code must not learn the ELF class |
| **Bootloader** | `boot/boot.asm` (BIOS real mode, `int 0x15/E820`, `int 0x13` disk read, switch to protected mode), `arch/x86/src/entry.asm`, `gdt_flush.asm`, `arch/x86/link.ld` | `kernel_main()` and the memory map only | the switch to long mode (own page tables before the jump) or a Multiboot2/UEFI entry; the image limit (512 KiB since Task 27.1; before it 256 KiB since Task 23.3, four fixed `int 0x13` reads) is a property of this bootloader, not of the kernel; Task 27.1: two stages, kernel at LBA 3, CRC-32 checked before the jump, see "The bootloader on real hardware" |

Smaller arch-dependent things that are *not* in the five places, all behind the HAL or in the arch directory:

- **Devices** (VGA console, keyboard, mouse, serial, PIT timer, PIC, ACPI, RTC, PCI, ATA): `arch/x86/src/*.c`, used only through the `hal_console_*`, `hal_input_*`, `hal_timer_*`, `hal_pci_list`, `hal_disk_*`, `hal_mouse_*`, `hal_poweroff` functions. In stage D-E they leave ring 0 and become service tasks.
- **Shell commands that need process internals** (`exec`, `elfinfo`, `kill`, `frames`, `ptest`, `proctest`, `slottest`, `limtest`, `faulttest`, `sleeptest`, `blocktest`, `handletest`, `channeltest`, `chanblocktest`, `chanxfertest`, `spawntest`, `resulttest`): `arch/x86/src/shell_x86.c` (exec, elfinfo, kill, frames, proctest) and `arch/x86/src/selftest_x86.c` (all the `*test` commands, Task 13; `chantest`, Task 14). The shell finds them through `hal_shell_commands()`; ARM returns an empty table.
- **The spawn loop of `bench`**: `bench_spawn_run()` is declared in `bench.h`, defined in `arch/x86/src/bench_x86.c`; ARM sets `spawn_supported = 0`.
- **Fixed-width on purpose** and *not* a 64-bit problem: every on-disk / ELF32 field, the BIOS-era sector numbers (`lba`), timer ticks, slot generations.

**How to add something arch-dependent:** put the code in `kernel/arch/x86/`, declare the *smallest possible* function in `hal.h` (opaque pointers and `uintptr_t`/`size_t`, no register names), implement it for every port (an empty stub is fine). Never add `#if defined(__i386__)` to shared code: under `-m64` the compiler skips such a branch, so the code inside it is never checked.

## Kernel size budget

What is counted: text + rodata + data of all kernel objects, **without** the user programs embedded in the image (`userbins.o`) and without `.bss` (heap arena and page tables are RAM, not image). Measured by `make ksize`.

| | bytes | |
|---|---|---|
| Today (after Task 7) | 72 999 (71.3 KiB) | everything is still in ring 0 |
| **Ceiling until stage D ends** | **76 544 (74.8 KiB)**, tests not counted (was 98 304 with tests, until Task 12; 69 632 until Task 24.1; raised by 24.1-24.3, 25.1, 25.2, 26.1 and 26.2, see the Makefile) | stages B-C add code before D removes any; `make ksize` fails above it |
| **Target for the v1.0.0 tag** | **65 536 (64 KiB)** | the check that closes stage D |
| Hard limit of the bootloader | 262 144 (256 KiB since Task 23.3; 196 608 from Task 16; 131 072 before) | whole image, already enforced by the Makefile |

Why 64 KiB is realistic. Roughly 37 KiB leaves ring 0 in stage D-E: shell (~26), VGA, keyboard, mouse, ATA, QlsFS, VFS/ramfs, `bench`. What stays is about 36 KiB (memory, tasks, interrupts, ELF loader, HAL). New code gets a sub-budget: handles ≤ 4 KiB, channels ≤ 6 KiB, spawn + result channel + task control ≤ 5 KiB. That is 36 + 15 = ~51 KiB, with ~13 KiB of margin. Measured so far (`-m32 -O2`): handles 1.7 KiB, channels 1.0 KiB.

**Task 11 (ksize 99 728 > 98 304).** Cause: the in-kernel test commands (`channeltest` ~10.7 KiB, `handletest` ~6 KiB and the others) are mostly long message strings, and `-O2` on i386 pads every long string literal to 4 bytes. Fix: `shell_x86.c` and `shell.c` are built with `-Os` (`X86_OPT_SIZE` in the Makefile); scheduler, paging, syscalls, handles and channels stay on `-O2`. Result: the kernel measures about 86 800 bytes (estimate, the host has no nasm; check with `make ksize`), ~11 KiB below the ceiling. The ceiling itself is unchanged. The test commands leave ring 0 together with the shell in stage D, so this is a temporary cost, not a reason to raise the budget.

**Task 12 (ksize 96 745, ceiling 98 304, margin 1.5 KiB).** The real code grew by about 0.9 KiB (channel.c +0.4, task.c +0.3, shell +0.1); the other ~9 KiB are the `chanblocktest` command itself (messages again). The ceiling is unchanged. **The margin is almost gone: the tests of Tasks 13-15 will not fit under 98 304 without a decision**, to be made at Task 13, not by accident (options: shorten the test messages; build the in-kernel tests as an optional `KERNEL_TESTS` part counted separately; raise the NOW ceiling with a reason).

**Task 13 (the tests are no longer counted).** `make ksize` showed 104 496 bytes against 98 304. Measured: the real code of the task grew by 2.0 KiB (channel.c 1.46 -> 2.87 KiB, handle.c 1.69 -> 2.24 KiB; both inside their sub-budgets of 6 and 4 KiB), but `shell_x86.c` was 40 KiB, nearly all of it the in-kernel self-tests. Tests are not kernel code and leave ring 0 with the shell in stage D, so they were moved to their own object, `kernel/arch/x86/src/selftest_x86.c` (the shell sees one table: `hal_shell_commands()` merges the two), and `make ksize` counts the kernel without it and prints the self-tests apart. Result: **kernel 65 627 bytes (64.1 KiB)**, 91 bytes from the v1.0.0 target of 65 536, with `shell.c` (10 KiB) and `bench.c` (6 KiB) still inside, both of which leave or shrink later. The ceiling is now 69 632 (68 KiB), tighter than the 98 304 before. Self-tests: 38.2 KiB; the image as a whole must still fit the bootloader's 128 KB (131 072), it is 116 KB now, so each new test costs room there.

**Task 20.3 (the image hit the bootloader's 192 KiB: 196 948 > 196 608; goal: under 150 KiB).** The image was 8 bytes under the limit before the task, so any growth broke the build. Measured where the bytes were: the in-kernel self-tests 81.7 KB (26 KB of it the text of 558 checks, printed only on a failure) and the test programs 45 KB (with symbol tables, `-O2`), against ~56 KB of real kernel code. What was done, in order of size: (1) the text of a check is not compiled in (`slot_check` / `check` drop the literal; a failure prints its line number; `make clean && make TEST_MSGS=1` brings back every text and PASS line), about -21 KB in the kernel and -10 KB in the test programs; (2) the user programs are built `-Os -ffunction-sections --gc-sections -s`, about -6.5 KB; (3) `-Os` for code that is not on the path of a task switch or a spawn: spawn glue (exec, process, ustack), log, console, file systems, keyboard / mouse / ACPI, one-time set-up (`idt_install` is 1.6 KiB), about -7 KB. Kept on `-O2`: scheduler, paging, pmm, kheap, string, the ELF loader, the ATA read. Result: about 149.5 KB (146 KiB), i.e. -47 KB. Rule from now on: a new check gets its text only through the macro; a new kernel file that is not hot goes on the `-Os` list in the Makefile.

**Task 20.4 (`spawn_chan_test`).** Kernel +131 bytes (the help line; `make ksize` 63 117 against the ceiling 69 632). The test itself is +2.5 KiB in `selftest_x86.o` (not counted by `ksize`) and the image grew 152 400 -> 155 024 bytes; its headings are compiled in only with `TEST_MSGS=1`.

**Task 17 (ksize 70 186 > 69 632).** Task 16 left the kernel at 68 774 bytes, 858 under the ceiling; the result channel added 1 412 (real code 1 279: channel 464, task 240, syscall 240, process 176, exec 159; plus 133 of help text in `shell.c`), so `make ksize` went **over by 554**. Fix, without touching the ceiling: `bench.c` is built with `-Os` like the shell (`X86_OPT_SIZE` in the Makefile). It is 6 227 bytes at `-O2` and 2 477 at `-Os`: nearly all of it is report text (the 4-byte string padding of `-O2` again), it runs once per `bench`, it leaves ring 0 with the shell in stage D, and what it *measures* (the spawn, the boot) is other code that stays at `-O2`; `make test-bench` passes. Result: **kernel 66 436 bytes (64.9 KiB)**: 3 196 under the ceiling and **900 bytes above the v1.0.0 target of 65 536**, 2 338 *smaller* than before the task. The ceiling is unchanged. The 900 bytes are the shell and bench text that leave in stage D.

**Task 18 (task control).** Real kernel code +1 412 bytes (task 432, handle 443, syscall 292, process 144, exec 101), shell jobs +597 and help text +222: **+2 231 in all, so the kernel is about 68.7 KB, ~900 under the ceiling** (computed from the compiled C objects with the Makefile's flags; `nasm` was not available where the task was written, so `make ksize` is the number to trust). A first version was +3 045: the live states were four comparisons in four places (now one run of the enum), `unblock_locked` was inlined into every caller, and the shell kept a name per job.

A block of tasks is not closed if `make ksize` is over its ceiling. If a task needs more, the budget is changed here on purpose, with the reason, not by accident.

## Platforms after v1.0.0 (block G)

Goal: one kernel that can be taken to many CPUs: **x86-64, RISC-V (RV32 / RV64), ARM (Cortex-A / Cortex-M), MIPS and WebAssembly.** v1.0.0 itself stays 32-bit x86; nothing below is started before the tag. What *is* done before it is the preparation (G0), because it is cheap now and expensive later.

### Three kinds of platform

| Kind | Targets | What is different |
|---|---|---|
| **MMU + privilege levels** | x86-64, RISC-V RV64 / RV32 (with MMU), ARM Cortex-A (AArch64), MIPS | the normal port: boot, context switch, page tables, interrupts, timer, syscall entry. The five places of *Architecture-dependent places* are all that changes |
| **No MMU** | ARM Cortex-M, RISC-V without MMU | tasks share one address space; protection is an MPU with a few regions (or none). Programs are position-independent and relocated at load. The *kernel* learns a second protection model, not only `arch/` |
| **Hosted** | WebAssembly | no CPU, no interrupts, no privilege levels. The kernel is a module in a host (browser, WASI runtime); a task is its own Wasm instance (own linear memory, so isolation comes for free); scheduling is cooperative (blocking = yield to the host); handles and channels are the natural fit. A separate project that shares the shared code, not an `arch/` folder |

### Order (each step proves something the previous one could not)

| Step | Port | Why in this place |
|---|---|---|
| G0 | preparation, inside v1.0.0 | see the next list |
| G1 | **x86-64** (long mode, QEMU `qemu-system-x86_64`) | closest to what works; the five places are already listed above; proves the 64-bit-clean rule on real code |
| G2 | **RISC-V RV64** (QEMU `virt`, S-mode on OpenSBI, Sv39, `ecall`) | the cleanest ISA and the first *non-x86 / non-ARM* port: proves the HAL is really portable |
| G3 | **ARM AArch64** (Cortex-A, QEMU `virt`, `svc`) | the existing ARM32 port is a stub (no paging, no ring3); this one is a full port |
| G4 | **RISC-V RV32** (Sv32) | first 32-bit port after x86: proves the code is clean on 32 *and* 64 bits |
| G5 | **no-MMU mode + ARM Cortex-M** (MPU, QEMU `mps2-*` boards, `svc`) | needs the kernel change below; the smallest targets, where 64 KiB matters most |
| G6 | **WebAssembly** (hosted, cooperative) | separate branch, after the model for "an address space that is not paging" exists |
| G7 | **MIPS** (little-endian `mipsel`, QEMU `malta`) | last: least value, and its software-refilled TLB is a different kind of work |

### G0: what is decided and done before the v1.0.0 tag

1. **Memory is an interface, not "x86 pages".** `hal_switch_address_space(void*)`, `hal_destroy_address_space` and `hal_address_space_frames` already take an opaque `void*`. G0 writes the contract down: an *address space* may be page tables (MMU), a set of MPU regions, or nothing (NONE); shared code never looks inside. The kernel asks the HAL which kind it is (`hal_protection_kind()`), so a no-MMU port later means no change to the shared scheduler, handles or channels.
2. **Little-endian only** on every port (MIPS = `mipsel`). Message layout, handle values and syscall arguments are fixed-width and have no pointers (rule 3), so userspace sources compile unchanged on every port.
3. **Clean on 32 and 64 bits**, not only 64 (RV32 and Cortex-M are 32-bit). `uintptr_t` / `size_t` everywhere stays the rule; the 32-bit side is covered by the x86 port itself.
4. **`make ksize` and `make check64` per port.** Both take the architecture as a parameter, so each port has its own number, and the shared code is compiled for every port that exists.
5. **No port before the HAL settles.** The HAL is stable after stage D (drivers and shell are userspace services). A second port started earlier would be rewritten.

### The size rule for every port: 64 KiB

**Shared core + the arch part of that one port, text + rodata + data, without `.bss` and without embedded user programs, must be ≤ 65 536 bytes** (see *Kernel size budget*). The x86 figures in that section already include the x86 arch part, so a new port is expected to land in the same range. If a port does not fit, the cause is looked for in the port (`-Os` for non-hot files, a feature moved to a userspace service), **not by raising the budget for everybody**. For WebAssembly the same number applies to the code section of the kernel `.wasm` module.

### A port is done when

- it boots in QEMU (or in the Wasm host) and runs `bench`;
- the same self-tests as on x86 pass (`faulttest`, `sleeptest`, `blocktest`, `slottest`, `limtest`, `handletest`, `channeltest`, and the ones added later);
- `make ksize ARCH=<arch>` is ≤ 64 KiB;
- `make check64` still passes **without a change in shared code**. If the port needs one, the HAL was wrong: fix the HAL and re-run every earlier port.

## At the end of every block

`make test-bench` (QEMU boots, kernel works), `make ksize` (budget), and from Task 8b `make check64` (every `kernel/src/*.c` and every shared header compiled as x86-64 with `-Wall -Werror`, so no pointer/integer cast survives; only `kernel/include` and `kernel/hal` are on its include path; it also fails on an `#include` of `arch/` or an `#if __i386__`-style branch in shared code). `make check64-selftest` proves the check still catches a `(uint32_t)ptr` cast and an architecture `#if`.

**Task 16 (image 140 232 bytes > 131 072).** The "116 KB" of Task 13 was out of date: Tasks 14-15 had already brought the image to ~129 KB, and Task 16 added ~11 KB (kernel +2.5, `selftest_x86.o` +3.5, two user programs +5), so the build stopped at the bootloader limit. The ELF files are not the problem (`ld -s` saves only 2 KB); the self-tests are 54 KB of the image. Decision: `boot.asm` reads a THIRD piece (LBA 257 = cylinder 7, head 0, sector 6, into 0x30000), limit 192 KiB, `KERNEL_MAX_BYTES` 196 608. The kernel proper (`make ksize`) is unchanged in budget. When the self-tests leave ring 0 (stage D) the image shrinks again.

## Keyboard service (goal 24)

Subtask 24.2: `user/kbdd.c` is the PS/2 keyboard as a ring-3 driver, not yet connected to the system. It starts with three handles: IRQ1,
the single port 0x60 (0x61..0x64 stay the kernel's: speaker, 8042 command port, mouse), a channel to the client. All the layout (tables,
Shift, Caps, E0, arrows, PageUp/PageDown) lives there; the key codes are the HAL_KEY_* numbers. Protocol: `user/kbd.h` (text messages of
1..64 bytes, 'G' -> state: modifiers, lost, delivered). The driver never waits for the client: a full queue drops the character and counts it.
The kernel side: `keyboard_release()` gives the keyboard away (IRQ1 masked, no handler, clean state; only then the HAL grants line 1 and
port 0x60), `keyboard_reclaim()` takes it back and is called by the HAL when the last handle to line 1 goes (a dead driver never leaves the
machine without a keyboard). `device_io` re-asks the HAL on every access, so a port handle does not outlive the permission.
Known limit: one task cannot sleep on its IRQ and its channel at once, so 'G' and a closed client are noticed at the next key event.
Tests: `make test-kbd-host` (same scancode streams through kbdd.c and the kernel's keyboard.c), `make test-kbd` (QEMU, `kbdd_test`; qemu_cmd.py
presses the keys the kernel asks for with a "SENDKEYS ..." serial line).

Subtask 24.3: the system's keyboard is the service. `kbdsvc.c` (kernel side, modelled on consvc.c) starts kbdd before the shell
(`hal_input_late_init`); `hal_input_getchar` takes a message without waiting, `hal_input_wait_char` sleeps in ONE blocking
`channel_recv_wait` (when nothing else can run the scheduler refuses the block and the CPU halts until an interrupt: no polling).
The way back is automatic: if kbdd does not start, does not reach its first wait in 1 s, or ends for any reason, its handles close, the
HAL's `keyboard_reclaim` runs, and the reader (it sees the closed channel) takes what the driver had queued and then reads keyboard.c.
The harness types every test through the keyboard, so a bug in kbdd cannot kill a run. Shell: `kbd [status|on|off|kill]`.
Tests: `make test-kbdswitch` (kbdswitch_test: keys through the service, the driver killed in the middle of a line, `kbd on`, nothing left;
then the whole regression list, all of it typed through kbdd). kbdd_test takes the line from the service and gives it back.

## The shell in ring 3 (goal 25)

Subtask 25.1: the kernel LENDS the screen and the keyboard to one ring-3 task and takes them back when it ends (`lend.c`, `lend.h`;
`consvc_lent/unlend`, `kbdsvc_lent/unlend`). What is lent is the two channels the kernel already speaks to `consoled` and `kbdd` over:
the task gets them as handles [0] (console, `user/console.h`) and [1] (keyboard, `user/kbd.h`), SEND | RECV, no DUP, no TRANSFER, plus the
result channel at the usual slot 3. A task that has them needs no new driver and no new system call: it speaks the protocol the kernel spoke.
The kernel LETS GO: after the spawn it closes its own two ends (a move, so nobody else can steal a key the task waits for). While the task
has them the kernel is silent (its output is counted and dropped, nothing waits, keyboard.c stays off).
The way back is automatic: whatever ends the task (exit, kill, a fault) closes its table; `consoled` reads CLOSED after everything the task
sent and ends by itself; `kbdd` is killed by the kernel (it sleeps in its interrupt wait). Then the keyboard is keyboard.c's again (the line
goes back with the driver's last handle) and the screen is vga.c's with the cursor READ FROM THE CRTC REGISTERS: the kernel's mirror knows
nothing of what the task wrote, the controller does. What the task wrote stays on the screen and the kernel's output continues right after it.
Both services are off afterwards, as after `console off` / `kbd off`; `console on` / `kbd on` bring them back. The hardware cursor is read
only after the driver ended (it sets the index and the data register in two writes: a read in between would corrupt it). Known limit: a task
that dies with the history (PageUp) on the screen leaves the history there, the live screen was in the driver's RAM.
Shell: `lend [prog]` (default `lendterm`: reads keys, writes them back, `quit` ends it, `crash` faults it). Test: `make test-shlend`
(`shlend_test`: refusals, the keys through the task, a normal end, a kill in the middle of a line at the bottom of a scrolled screen,
nothing left). Size: kernel about +1.1 KiB; the ceiling went 72 704 -> 73 984 (temporary, like 24.x). Next: 25.2 the ring-3 shell `sh`
(line editor, built-ins), 25.3 `run prog [args] [--grant name[:rights]]`.

Subtask 25.2: `sh`, the shell as a ring-3 task (`user/sh.c`), started by the kernel's `sh` command (`lend_run`: lend, wait, and the services
start again by themselves, so the kernel's own shell goes on as before). It reads the keys with ONE blocking `chan_recv` on its keyboard
channel (asleep until a key comes, no polling) and only writes to the screen ('W', 'P', 'M', 'C', 'V'); it asks the driver once, 'G', where the
cursor is, and from then on keeps its own mirror of it (it is the only writer while it has the screen). Line editor: insert, Backspace,
Delete, Left / Right, Home / End, Up / Down = history (8 lines; the line being typed is a draft that Down brings back), PageUp / PageDown
= the driver's screen history (any other key goes back to the live screen first); a line is up to 120 characters and may wrap and scroll.
Built-ins: `help`, `echo`, `clear`, `exit`. A line that grows is written through the cursor (the driver wraps and scrolls, the mirror
counts the scrolls and the line's start row moves with them); a change in the middle repaints the tail with 'P' and puts the cursor back.
Tests: `make test-sh` = the host half (`tests/sh_host_test.c`: the real `sh.c` against the real `consoled.c`, a model of the editor,
thousands of random keys at every row of the screen, ASan / UBSan) and `sh_test` in QEMU. Size: kernel about +0.35 KiB, ceiling 73 984 ->
74 496 (temporary). Next: 25.3 `run prog [args] [--grant name[:rights]]`.

Subtask 25.3: `run prog [args] [--grant name[:rights]]...` in `sh`. The line is read and checked completely first (`user/runcmd.h`, no side
effects): an unknown name, a right the shell has not (the shell's `con` and `kbd` are SEND | RECV | TRANSFER, no DUP: `d` is refused), an
unknown letter, an empty spec, the same handle twice: the reason is told and NOTHING starts. Rights letters s r d t; none given = all the shell
has on that handle; the program gets a COPY (rights only shrink, the kernel checks it again: E_WIDEN / E_RIGHTS). Then `SYS_SPAWN` with a result
pointer and ONE blocking `chan_recv` loop on the result channel until it reads CLOSED (the kernel's task_result_t is the last message); the shell
tells "prog: exit code N" or "prog: killed: page fault (write, not mapped) at 0x...", "divide error (SIGFPE)", "SIGKILL" ...; the result handle is
closed after every run. Afterwards the shell asks the driver for the cursor again ('G'): a program may have written through a granted `con`.
Kernel changes: the name given to `SYS_SPAWN` is a COMMAND LINE ("prog w1 w2", up to 16 words, 23-character name, 127 bytes; argv[0] = the name;
argc is now always >= 1 from ring 3; bad lines are E_ARG before any work) and `lend` gives the shell TRANSFER on its two handles. Test program
`grantprobe` (exit code = the rights it found: 1 send, 2 recv, 4 dup, 8 transfer, handle 1 in the high half).
Known limits: the shell does not read keys while a program runs (a stuck program is killed from outside; Ctrl-C is a later subtask); while the
screen is lent a program's `SYS_WRITE` text is dropped (it has to write through a granted `con`).
Tests: `make test-run` (host: `run_host_test`, `sh_host_test`; QEMU: `run_test`, then sh_test, spawntest, resulttest, freetest).


## Who holds what, and where it came from (goal 26)

Subtask 26.1: the kernel can now LOOK at a handle table; nothing changes in how handles work. Two additions in `handle.c`:
- **Origin tag**: the free byte of `handle_entry_t` (was `_pad`) is now `origin`, set once when the entry is filled: `NEW` (made for the
  task by the kernel: `channel_create`, a device, a task handle, the spawner's end of a result channel), `DUP` (`handle_dup`/`_ttl`),
  `SENT` (received in a message), `SPAWN` (`handle_adopt`), `RESULT` (`handle_install_at`: the task's own result end). Free slot = 0. The
  kernel's own logic never reads it. Cost: no new byte in an entry; measured +510 bytes of code in `handle.o` at `-Os` (a first estimate of ~0.3 KiB was too low), so `make ksize` went 74 362 -> 74 872 and the ceiling was raised 74 496 -> 75 008 on purpose (temporary: `caps` is a shell command and leaves ring 0 with the shell in stage D).
- **`handle_inspect_next(owner, &cursor, &info)`**: one open handle per call, in slot order: number, type, rights, origin, state (`OK` /
  `REVOKED` / `EXPIRED`), ticks left, the object address (to tell two handles of one object apart), and the PARENT: `ROOT` (an original),
  `OPEN` (made from a handle that is still open: the holder's task id and the handle number there) or `GONE` (made from a handle that was
  closed, or whose task died). The parent is found, not stored: the node of a handle is held by exactly one table entry, so the holder of
  `node->parent` is the entry with that node (one pass over the tables while interrupts are off, per handle asked; a diagnostic, not a hot path).
Ring 0 only, on purpose: no syscall gives a task the table of another task (that would be ambient authority). Test: `make test-caps-host`
(handle.c + channel.c on the host, ASan/UBSan): every way a handle comes to be, parent open / gone / dead task, time left and expiry, revoke,
the walk and its cursor, then 20 000 random steps against an independent model. Next: 26.2 (below).

Subtask 26.2: `caps [N]` in the kernel shell (`kernel/src/caps.c`, shared code: x86 and ARM). `caps` lists every task that holds a handle, `caps N`
task N (also one with none); a bad argument prints `usage: caps [task id]`. One line per handle, built in a buffer by `caps_format_line` so that tests can read it:

    task 3 (ready): handles 3
      3:0 chan srdt- new @00123456
      3:2 chan s-d-- dup of 3:0 @00123456
    task 5 (sleeping): handles 2
      5:0 chan sr--- spawn from 3:4 @00123abc
      5:3 chan s---- result @00124000

Reading a line: `<task>:<slot>`, the type (chan task test mem io irq), the rights as the letters s r d t c (`-` = not held), the origin (new, dup, sent,
spawn, result), then `of` (after dup) or `from` and the parent as `<task>:<slot>`, or `(closed)` when the handle it was made from is closed or its task is
dead; `ttl N` (ticks left), `EXPIRED` or `REVOKED`; `@` and the object's address: two handles with the same one are the same thing (the two ends of a
channel differ). The numbers are slots, not full handle numbers: a parent is found by its node, so it is always the current holder. Cost: `caps.o` 1 239 bytes
at `-Os`, shell +141: the ceiling went 75 008 -> 76 544 (temporary: `caps` and the look in handle.c leave with the shell in stage D). Tests: `make test-caps` =
`make test-caps-host` (every shape of a line, a line cut to every size, 20 000 random lines with guard bytes, the real `caps` command on the fake tasks with the
exact text, bad arguments, parent closed, holder dead, time, nothing changed by looking) and `caps_test` in QEMU (kernel tasks that make, dup, send, close and
let a handle run out of time; a real spawn: the program's own table is exactly its result end at slot 3 and the handle it was given, "spawn from" the shell's; the
shell's result and task handles are "new"; nothing left), then the earlier handle and spawn tests as a regression. Not in this task: `caps` in ring 3 (see 26.1).

## The bootloader on real hardware (goal 27)

### 27.1: it reads what the BIOS really has, and refuses a kernel that arrived damaged

**Why.** QEMU's floppy is the kind of disk the old `boot.asm` was written for: drive 0, 18 sectors x 2 heads, four fixed reads that never fail. A real
machine, an old board or a cheap USB stick that emulates a disk, does none of that. The failure that matters most is quiet: the BIOS answers "ok" and
hands over the wrong sectors (one missed, one read twice, a geometry the stick invents). The loader then copies a piece of something else to 0x40000, jumps
into it and the machine resets with nothing on the screen. Rule for the whole goal: **a read is trusted only after it was checked.**

**What the old loader did wrong on hardware (found by reading it, not by a crash).**
- `boot_drive` was declared and read for every `int 0x13`, **never written**: always drive 0. A stick that boots as drive 0x80 was read as a floppy.
- The disk was cut by a table of four fixed CHS addresses (18 x 2): right only for a 1.44 MB floppy.
- The wait for the keyboard controller (`wait_8042`) had no timeout: a machine without an 8042 reads 0xFF from port 0x64 and waits for ever.
- Every message went to the serial port only; a real machine has none. A 16-bit loop waited on the UART with no limit.
- Interrupts were off for the whole BIOS time (a floppy BIOS waits for IRQ6); CS was assumed 0 (some BIOSes enter at 07C0:0000).

**What it does now.**

| Step | How | Fails as |
|---|---|---|
| drive | `mov [boot_drive], dl` first thing; every call uses it | (was: always 0) |
| how to read | `int 0x13 AH=41h` extensions -> a plain LBA (`AH=42h`, no geometry at all); else CHS cut by the geometry the BIOS reports (`AH=08h`), default 18 x 2 when it says nothing or says 0 sectors | - |
| one read | at most 32 sectors, never across a 64 KB DMA window (nor across a track in CHS); a failed read: controller reset, the chunk size HALVED (32, 16, ... 1), again; 6 failures in a row | `DISK ERR` |
| stage 2 whole? | a magic dword at its start and at its very last 4 bytes: a missed or shifted sector shows at one of the two ends | `DISK ERR` |
| A20 | tested, not assumed: already on / fast gate 0x92 (bit 0 never set: it is "reset") / keyboard controller, each with a timeout, each followed by a re-test | `A20 ERR` |
| the kernel | **CRC-32 over exactly `KERNEL_BYTES`** (zlib's, bit by bit, real mode, 64 KB windows) against `KERNEL_CRC`, both computed by `tools/crc32` at build time and handed to nasm with `-D` | `BAD CRC` |
| output | BIOS teletype (screen) **and** serial; the serial wait is bounded (2048 reads of the line status) | - |

All three errors stop the machine with `hlt` in a loop (no reboot loop, no jump into garbage) and print on the screen and on the serial port.

**Layout.** A boot sector is 512 bytes and the first version of all this was 743: two stages. Sector 0 = stage 1 (probe, the generic reader
`read_sectors`, stage-2 check), sectors 1-2 = stage 2 (`STAGE2_SECTORS`, loaded to 0x7E00: memory map, A20, kernel read, CRC, the switch to protected mode,
the GDT), the kernel behind them at **LBA 3** (it was LBA 1). `boot.bin` is the whole 3-sector prefix and the Makefile puts the kernel at
`seek = size(boot.bin) / 512`. Stage 1 is 401 of 440 bytes (the last 70 bytes of the sector are the disk signature and the partition table, 27.2), stage 2 uses 404 of 1020.
**The kernel limit is 512 KiB** (`KERNEL_MAX_BYTES` 524 288; read to 0x10000..0x8FFFF, then copied to 1 MiB): the old four-piece limit of 256 KiB, 4 KiB away
from the kernel, is gone, and the way the limit is reached is now a constant, not a fifth `int 0x13` call.

**For IoT (the rule of the goal).** Nothing in the loader depends on a floppy or on a PC disk: it asks the BIOS what the disk is, reads LBA when it can, and
treats "the media lied" as an ordinary case. The check is a property of the image (size + CRC-32 produced at build time by a 40-line C tool), not of x86:
the ARM / flat-image path (G, `Task 16` loader note) can verify the same two numbers over the same file. The tool is 64-bit clean (`make test-crc32` compiles
it with `-m64 -Wall -Wextra -Werror`). The kernel proper is unchanged: `make ksize` 76 389, `make check64` ok.

**Tests.** `make test-crc32` (host: the known vector "123456789" -> 0xCBF43926, a split run equals a whole run, -m64 clean) and `make test-boot` (QEMU,
`tests/boot_test.py`): good images as a floppy (drive 0x00), a hard disk (0x80, LBA) and a USB stick (usb-storage behind the BIOS), each must print `kernel crc ok`
and the kernel's serial log (`handing off to shell`; the banner goes to the screen only, the first version of the test waited for it and failed on good images); damaged copies (a sector zeroed, **one sector missed so that everything behind it is shifted**, one bit flipped) must print `BAD CRC` on
the serial port AND on the screen (read from video memory through the monitor), never start the kernel and never reboot; stage 2 zeroed / shifted must print
`DISK ERR`. The same loader was also run (not as a test of this task, as a check while writing it) in a CPU emulator against a fake BIOS with the rules of a real
one: floppy 18 x 2, USB-FDD 36 x 2, hard disk CHS 63 x 255 and 32 x 64 without extensions, LBA with 127 / 8 / 1 sector per call, reads that fail 3 times, a BIOS that
lies in 4 ways, A20 by each of the three methods or none, a floating 8042, no UART, entry at 07C0:0000, kernels of 1 byte ... 262 144 bytes: all behave as above.

**Not in 27.1 (the next subtasks).** The partition table / `usb.img` and a safe `make usb-write` (27.2, below); (27.3 is done, below); a checklist and a log of the first run on a real machine (27.4).

### 27.2: a disk image a BIOS accepts, and a safe way to write it

**Why.** Many BIOSes, booting a USB stick in "hard disk" mode, want more than 0x55AA: a partition table with an **active** entry (some also reject an
all-zero table). The 1.44 MB `disk.img` was built for QEMU's floppy and is the wrong size and shape for a stick.

**What.** (1) The boot sector now ends with a disk signature (bytes 440-443) and ONE partition entry (446-461): active, type 0xDA ("non-filesystem data", so no OS
offers to mount or format it), LBA 1 .. end of the image, CHS "beyond CHS" (FE FF FF). Stage 1 never reads it. The entry's size is `IMAGE_SECTORS`, passed by the
Makefile: boot sectors + kernel sectors + 8 spare, rounded up to 1 MiB. (2) `make usb.img` writes boot.bin and the kernel into an image of exactly that size (the size
is read back from the entry in boot.bin: one place for the number). `disk.img` (floppy, QEMU) is unchanged apart from carrying the same table. (3) `make usb-write
DEV=/dev/sdX` runs `tools/usb-write.sh` (POSIX sh, only dd/cmp/stat): it refuses a partition instead of a disk, anything mounted or swap, a non-removable disk
(`ALLOW_FIXED=1` for USB hard disks), a disk over `MAX_GB` (64), a disk smaller than the image, not being root; shows vendor/model/size, makes you type the device name
(`YES=1` for scripts), writes with `conv=fsync`, then reads the image back with `iflag=direct` (no page cache) and compares. On Windows: Rufus in "DD Image" mode or
Win32DiskImager (WSL1 cannot reach USB devices). Boot the stick in legacy/CSM mode, the "USB-HDD" or "USB-FDD" entry.

**Tests.** `make test-usbimg` (host): usb.img has the signature, one active entry covering LBA 1..end, three empty entries, stage 2 whole, the kernel byte for byte at its LBA with
the CRC the loader carries, zeros behind it; `usb-write.sh` refuses no device / missing image / a regular file / a non-block path / no confirmation / an image that is not whole
sectors, and writes + verifies a file target. `make test-boot` got three cases: usb.img as a hard disk, as a USB stick, and damaged as a USB stick. **Fixed in 27.1's test:** it
waited for the banner `QlsOS v0.1.0`, which the kernel writes to the screen only; good images booted fine and the test said FAIL. It now waits for `handing off to shell`.

### 27.3: the kernel on a machine that is not QEMU's default

Read for what a real PC does differently; every item below was a way the kernel could hang or hurt, found in the source (nothing crashed yet).

| Found | Why it matters on hardware | Now |
|---|---|---|
| `serial_write` waited for the transmitter with no limit | a machine with no UART at COM1 (laptop, small board): the FIRST message waits for ever, a black screen | `serial_init` tests for a UART (modem-control loopback, works on 8250 too); no UART = `serial_write` returns at once; with one, the wait is bounded (100 000 status reads) |
| `pmm_init` freed `base/4096 .. end/4096`, rounding a usable region OUTWARD | real BIOS regions often start/end inside a page: a page that is half reserved (ACPI tables, the EBDA) was handed out | `mmap_fill_frames`: usable shrinks INWARD to whole pages |
| usable and reserved entries that overlap were treated in map order | a later reserved entry did not take back a page already freed | reserved is applied AFTER all usable ones and rounds outward: reserved wins |
| `acpi_ext` bit 0 ("ignore this entry") was not looked at, and the loader never set the field | a BIOS that answers with 20-byte entries left whatever was in memory there | the kernel ignores entries with the bit clear; the loader presets the field to 1 before every E820 call |
| no E820 = count 0 = not one free page | old boards and some embedded BIOSes have no E820 | loader builds the map from INT 12h + INT 15h E801 (or CX/DX form) or AH=88h; the kernel clamps a count over 512 as corrupt |
| the "blank disk" test read ONE sector (LBA 0) and a blank disk is formatted WITHOUT asking | on a PC with a disk, sector 0 all zero + data further in = the user's disk erased at boot | blank = the first 64 sectors (32 KiB) all zero; one unreadable sector = not blank |
| **ACPI tables were read straight at their physical address; the kernel maps only the first 256 MB** (found by `make test-hw` with 1 GB: `PANIC: Page fault at 0x3FFE1C52` in the ACPI parse) | firmware puts the tables at the TOP of RAM: ~127 MB with QEMU's 128 MB (inside the map, so it always worked there), ~1 GB with 1 GB, ~3.5 GB with 4 GB. **Every real machine with 256 MB or more panicked at boot** | `paging_map_phys`: a kernel-only window (directory entry 64, 0x10000000) onto physical memory, two slots (1 MB for the RSDT, 3 MB for any other table); a range inside the identity map is returned unchanged, so a small machine behaves exactly as before. `acpi_init` maps each table by physical address, copies what it needs (PM1 ports, SMI command, the S5 values, the I/O reset register) and never touches a table again: shutdown and reset work in any address space |
| `ata_disk_present` looked for status 0x00 only | a floating bus reads 0xFF: the code spun through two 100 000-read waits | 0xFF = no controller, answered at once |

Not changed because it was already safe: the PS/2 mouse handshake (every wait is bounded, "Mouse not detected" is not fatal), the keyboard (interrupt driven, nothing waits), the ATA waits (bounded).
**Not solved, said plainly:** a machine whose only keyboard is USB with no PS/2 emulation in the BIOS has no input for the shell (the shell reads the PS/2 keyboard
and nothing else); a serial console with input would be the way for a headless IoT board, and that is its own subtask, not a patch. PMM still manages the first 64 MB.

**Size.** Kernel 76 389 -> 77 232 bytes (+843: +437 for the first six rows, +406 for the ACPI window); `KERNEL_BUDGET_NOW` raised 76 544 -> 77 568 in two steps in the Makefile with the reason written next to it (not "temporary": no later stage removes these).

**Tests.** `make test-mmap` (host, the kernel's own `mmap.c` compiled into the test, 64-bit and, where a 32-bit libc exists, 32-bit): page-unaligned regions, reserved inside usable, entry order, ignore bit, zero length, a length that overflows 64 bits, a region over the managed limit, no entries, the map the loader builds without E820.
`make test-hw` (QEMU): RAM 32 / 64 / 1024 / 3500 MB (the 1024 MB case is the one that found the ACPI panic); no UART at all (`-nodefaults`) and a UART with no listener (`-serial none`), proved by the kernel still formatting an all-zero disk (a side effect that happens after dozens of serial writes); a disk with data behind an empty sector 0 stays untouched; an all-zero disk is formatted as before.
The loader's new memory-map code was run in the CPU emulator with a fake BIOS: normal E820, 20-byte entries, no E820 with E801, E801 in CX/DX, AH=88h only, nothing but INT 12h.

### 27.4: the first run on a real machine (ASUS K50IJ, 2026-10-31, 14:47)

**Machine.** ASUS K50IJ laptop (Intel Centrino 2: GM45 chipset + ICH9M), old BIOS (no UEFI), Windows 7 on its disk. **Image:** `usb.img`, written to a 2 GB USB stick
with Rufus (image mode); booted from the BIOS boot menu (Esc), the stick as a USB disk.

| Check | Result |
|---|---|
| bootloader: stage 2, memory map, A20, kernel read, CRC-32 | passed (no `BAD CRC` / `DISK ERR` / `A20 ERR`), the kernel started |
| shell, built-in keyboard (PS/2) | works |
| touchpad (`mouse`) | works: the values change when the finger moves (PS/2) |
| `frames` | 15828 free / 16384 total: the PMM manages only the first 64 MB, as documented (the machine has more RAM) |
| `lspci` | 22 devices: GM45 host bridge and graphics, 3 UHCI + 1 EHCI USB controllers twice (`1a.x`, `1d.x`), HD Audio, PCIe root ports, LPC bridge, **SATA in AHCI mode** (class 0106, prog_if 01), Atheros Wi-Fi (168c:002b), Atheros Ethernet (1969:1026) |
| `exec hello`, `sh` (the ring-3 shell) | work |
| files: `ls`, `write`, `cat`, the tests' files | work, but **in RAM** (`diskdump 0`: "no IDE disk found"; a file written before `reboot` was gone after it): the active file system is `ramfs`, not persistent |
| `bench` | works, writes `baseline.txt` |
| `kbd status`, `console` | work |
| `reboot`, `shutdown` | work; after either the BIOS boots Windows 7 again because the stick is not first in the boot order (expected: the stick is chosen by hand in the boot menu) |

**Found.**
- **The kernel does not see the internal disk.** The SATA controller is in AHCI mode (class 0106, prog_if 01) and the kernel has only the legacy IDE/PIO driver:
  `diskdump 0` answers "no IDE disk found". By the boot code (`kernel.c`) that means `ramfs` is the active file system ("No disk detected -- ramfs ... not persistent"),
  so the files of this run lived in RAM: the initrd files plus what was written in the session, and a file written before `reboot` (`t1`) was gone afterwards.
  The consequence is good for the owner of the machine: with no disk visible the kernel cannot touch the Windows disk (Windows 7 booted again after every run).
  An AHCI driver is needed for disks on this kind of machine; it belongs to block E (the disk service). To be sure which file system was active: `mount`, and the
  boot line about the disk (PageUp scrolls back to it).
- `dmesg` printed an empty log on this machine. In QEMU it holds the kernel's events. Cause not known yet: to be looked at as its own small task.
- The memory ceiling of the PMM (64 MB) is the next limit that real machines hit; see "Limits to know about" in the README.

**Not checked yet.** The real RAM size as the loader reports it; the ring-3 `run prog --grant`; a USB keyboard; the output of `mount` and the boot line about the disk (to confirm `ramfs`).
