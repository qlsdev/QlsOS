# ELF loader

Runs static 32-bit x86 ELF executables as ring3 processes.

```
exec <file> [args...]    run a program (active filesystem first, then ramfs); the shell waits for its result
elfinfo <file>           show header/segments and whether the file is loadable
```
Built-in programs (embedded in the kernel, copied to ramfs at boot): `hello` (self-test of
.text/.rodata/.data/.bss/stack/argv), `crash <null|kernel|unmapped|stack|div|ud|cli|spin>`
(each must kill only that process). More test programs live in `user/` (dispatcher, services, drivers, the ring-3 shell `sh`).

## Pieces

| File | Role |
|---|---|
| `kernel/arch/x86/src/elf.c`, `include/elf.h` | validation + mapping of PT_LOAD segments (`elf_check/validate/load`) |
| `kernel/arch/x86/src/ustack.c` | initial user stack: `argc, argv[], NULL, strings` |
| `kernel/arch/x86/src/process.c` | `process_spawn_elf_h()`: directory + segments + 4-page stack (+ guard page) + task + the handles it is given |
| `kernel/arch/x86/src/exec.c` | file lookup (active VFS, then ramfs), built-in program table, `exec_spawn` (`SYS_SPAWN`) |
| `kernel/src/task.c` | tasks, the result message built at death (`task_result_make`) |
| `user/` | `crt0.S`, `link.ld`, `usys.h` (syscall wrappers), `hello.c`, `crash.c` |
| `tests/` | host-side tests of the real loader code (`make test`) |

## What the loader guarantees
* Only ET_EXEC / EM_386 / ELF32 / little endian. PIE, shared objects, PT_INTERP/PT_DYNAMIC are rejected with a specific error.
* The whole file is validated BEFORE any memory is allocated (64-bit arithmetic for every offset/size).
* Every segment must lie in `[0x40000000, 0x7FFFC000)`; the entry point must be inside an executable segment.
* Max 16 program headers, max 2048 pages (8 MB) per image.
* .bss is zero (frames are zeroed), segments sharing a page are supported (page becomes writable if any is).
* Failure leaves nothing behind (frames and page tables are freed).

## Process ABI
* Image at `0x40000000` (see `user/link.ld`), stack top `0x80000000`, 16 KB, unmapped guard page below.
* Entry: `esp -> argc, argv[0..], NULL, strings` (crt0 turns it into `main(argc, argv)`; return = exit).
* Syscalls (`int 0x80`, eax): 1 write(ebx=str), 2 exit, 3 sleep(ecx=ticks), 4 yield, 5 write_serial(ebx=str), then the capability calls:
  20 handle_close, 21 chan_create, 22 chan_send, 23 chan_recv, 24 spawn, 25 handle_dup, 26 task_ctl, 27 handle_ctl, 28/29 chan_send_h / chan_recv_h
  (handles in messages), 30 chan_wait, 31 dev_map, 32 dev_io, 33/34 irq_wait / irq_ack. The numbers and arguments are in `kernel/arch/x86/include/syscall.h`,
  the wrappers in `user/usys.h`.
* A spawned task holds exactly the handles it is given (they are listed on its initial stack behind argv's NULL: `nh, h0, h1, ...`); a spawn with
  a result channel also puts the child's write end at the fixed handle 3.
* Any fault in ring3 (page fault, #GP, #UD, #DE) kills only that process.

## Adding a program
1. `user/foo.c` (include `usys.h`, define `int main(int, char**)`).
2. Add `foo` to `USER_PROGS` in the Makefile.
3. Add a `.incbin` block to `kernel/arch/x86/src/userbins.S` and a row to `builtin[]` in `exec.c`.

## Tests
`make test` builds the user ELFs and runs `tests/elf_host_test` under ASan/UBSan: real binaries,
crafted layouts, ~30 malformed-image cases, out-of-memory/map-failure leak checks at every
failure point, `ustack_build`, and 200k mutation-fuzz iterations. It does NOT boot the kernel.

## Limits / next steps
* The bootloader reads the kernel (up to 512 KiB) in checked chunks and verifies its CRC-32; the Makefile fails the build above the limit.
* Built-in programs only live in the kernel image; there is no way yet to copy a binary into QlsFS (a `cp` command or a host tool would fix that).
  Programs start by file name; once the file service is a task the name is replaced by a handle to an image and nothing else in the call changes.
* No heap (brk), environment variables or Ctrl+C; arguments reach a spawned task as messages over the handles it holds. The exit code exists (`SYS_EXIT`)
  and arrives on the result channel together with the reason of death.
* For the small IoT ports planned after v1.0.0 (no MMU, Cortex-M) the loader must be the smallest thing that works there (a position-independent flat
  image with a tiny header); it replaces `elf.c` behind the same call.
