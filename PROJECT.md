# QlsOS (Quick Light Secure OS) — about the project

A capability microkernel operating system written from scratch: a bootloader, a kernel, ring-3 services and a command shell,
with no third-party libraries. It runs on two architectures — **x86 (32-bit)** and **ARM (Cortex-A15, QEMU `virt`)** —
in the QEMU emulator; the x86 bootloader and kernel are also being prepared for real PCs. Target: **v1.0.0**.

## Direction (target v1.0.0)

QlsOS is a **capability microkernel**: small, safe, easy to understand, with fast program start. The old POSIX-style plan
(exec, fork, wait, signals, pipes) was dropped; `fork`, `wait` and signals are gone from the code.

Rules of the design:
- All communication goes through **channels and handles** (capabilities). **No PIDs and no signals.**
- A task sees only the handles it was given; rights can only be narrowed (`handle_dup`, passing in a message, `spawn`), never widened;
  a handle can be revoked, and can carry a time limit.
- The system lives in userspace: a System Dispatcher task hands out channels to services; console and keyboard are ring-3 driver
  tasks; the shell is a ring-3 program. The kernel keeps only memory, tasks and scheduling, interrupts, handles, channels and the loader.
- Code is **64-bit clean** (`uintptr_t`, `size_t`, fixed-width types); v1.0.0 itself stays 32-bit x86, a 64-bit port comes later through the HAL.
- Light enough for the weakest IoT devices: no extra bloat, kernel code under 64 KiB at the v1.0.0 tag.
- Every task is closed by a test (on the host and/or in QEMU); tasks go strictly in order.

## Plan: blocks of tasks

**A** preparation (signals rolled back, task slot generations, per-task limits, clean page fault, blocking `sleep`, baseline
measurements, `ARCHITECTURE.md`, 64-bit audit, `make ksize` / `make check64`) — **done** ·
**B** handles and channels (handle table, `dup`/`close`, channels, blocking calls, handles in messages, exact freeing) — **done** ·
**C** spawn and task control (spawn from ring 3, result channel instead of `wait`, task control handle, revoke and handle time limits,
the shell waits on the result channel; then **`fork` and `wait` deleted**) — **done** ·
**D** the system in userspace (dispatcher, console service, keyboard service, ring-3 shell, `caps`, real-hardware preparation) — **in progress** ·
**E** disk and programs (ATA service, QlsFS directories, file protocol over channels, mini-libc, utilities) ·
**F** release (measurement table, docs, ISO + installer, tag v1.0.0).

After v1.0.0: more platforms through the HAL (x86-64, RISC-V RV64 and RV32, AArch64, no-MMU / Cortex-M, WebAssembly, MIPS — each port must keep
the kernel within 64 KiB), GUI (framebuffer, display server, windows as capabilities), SMP, a POSIX layer / toolchain / WASM runtime,
later orthogonal persistence, USB and networking. The platform plan is in [`ARCHITECTURE.md`](ARCHITECTURE.md), section "Platforms after v1.0.0".

## Where the project is now

Blocks A, B and C are done; block D is under way. What exists in block D:

| Goal | What | Status |
|---|---|---|
| 22 | **System Dispatcher**: the first ring-3 task; starts services, hands channels to clients ("give me service X"), supervises them | done |
| 23 | **Console service**: devices (memory regions, I/O ports) as handles; the VGA driver `consoled` in ring 3; the system's screen goes through it, with automatic fallback to the kernel driver | done |
| 24 | **Keyboard service**: interrupts as handles; the PS/2 driver `kbdd` in ring 3; the system's keyboard goes through it, with automatic fallback | done |
| 25 | **Shell in ring 3**: the kernel lends the screen and keyboard to one task; the shell `sh` (line editor, history, builtins); `run prog [args] [--grant name[:rights]]` | done |
| 26 | **`caps`**: who holds which handles and where each came from (origin tag + inspection in the kernel) | done |
| 27 | **Real hardware**: two-stage bootloader with a CRC-32 check of the kernel (27.1), a USB-bootable image and a safe writer (27.2), a kernel that survives a real PC's memory map, no COM port, ACPI above 256 MB (27.3); a checklist and a log of the first run on a real machine (27.4) | 27.1-27.3 done; 27.4: first run on a real laptop (ASUS K50IJ) done, more checks open |

Not done yet: the end of block D (the old in-kernel drivers, the shell and the in-kernel tests still have to leave ring 0, which is what takes the
kernel down to 64 KiB), blocks E and F, ports to other platforms. The kernel is about 77 KB of code today (ceiling 77 568 in the Makefile, target 65 536 for the tag; `make ksize`).

## What it is written in

| Language | Where | Size |
|---|---|---|
| **C** (freestanding, no libc) | the kernel (shared + x86 + ARM), ring-3 programs, host tests | ~13,100 lines of kernel code + ~2,800 lines of headers; ring-3 programs ~3,300 lines; host tests and the QEMU harness ~5,600 lines |
| **x86 assembly (NASM)** | bootloader, kernel entry, interrupts, context switching, ring 3 | the files in `boot/` and `kernel/arch/x86/src/*.asm` |
| **ARM assembly (GAS, `.S`)** | startup, vector table, context switching | 3 files; x86 `.S`: `userbins.S`, `user/crt0.S` |
| **Linker script (`.ld`)** | kernel memory layout (x86 — at the 1 MB address, ARM — its own) | 2 files |
| **Make / Bash / Batch / Python** | build, launch and the QEMU test harness (`Makefile`, `build_and_run.sh`, `run.bat`, `run_wsl1.bat`, `tests/*.py`) | — |

Build tools: `nasm`, `gcc -m32`, `ld` (x86); `arm-none-eabi-gcc` (ARM); run with `qemu-system-i386` / `qemu-system-arm`.

## What it can already do

- **x86 boot:** its own two-stage bootloader: the BIOS memory map (E820, with fallbacks for old BIOSes), A20 (tested, not assumed), LBA or CHS
  reads as the BIOS allows (a failed read is retried with smaller chunks), CRC-32 check of the kernel, protected mode, relocation of the kernel
  to 1 MB. A damaged kernel is refused with a message, never started.
- **Memory:** a physical page manager (bitmap) that shrinks usable regions to whole pages and lets reserved ranges win; paging; a kernel-only window
  onto physical memory (ACPI tables above 256 MB); its own `kmalloc`/`kfree` (free-list).
- **Interrupts and hardware (x86):** GDT/IDT, PIC, a 100 Hz PIT timer, PS/2 keyboard and mouse, VGA text mode with scrolling, COM1 (optional:
  a machine without a UART is fine), CMOS RTC, PCI, ATA/IDE (PIO), ACPI (reboot / shutdown).
- **Multitasking:** a preemptive timer-driven round-robin scheduler, each task with its own kernel stack, one general blocking mechanism
  (`task_block`) behind sleep and channel waits; task identity is `(slot, generation)`.
- **Processes:** ring 3, `int 0x80` system calls, a page directory per process; a crash of a user process does not kill the kernel;
  per-task limits on memory frames and handles.
- **Handles and channels:** per-task handle table (rights only shrink, a closed number never works again), channels (a pair of endpoints,
  64-byte messages, 16 per queue, blocking and non-blocking calls), handles in messages, `revoke`, time limits, exact freeing of every object.
- **Spawn and control:** `SYS_SPAWN` from ring 3 (copy of the given handles, nothing inherited), the result channel (one message: exit code or
  the reason of death and the fault address), the task control handle (stop, resume, kill).
- **Devices as handles:** `MEMREG` and `IOPORT` handles, `SYS_DEV_MAP` / `SYS_DEV_IO`, an interrupt line as a handle (`SYS_IRQ_WAIT` / `SYS_IRQ_ACK`).
- **Userspace:** System Dispatcher, console service, keyboard service, ring-3 shell, test services and clients (`user/`).
- **Measurements:** `bench` times the boot stages and the program start and writes `baseline.txt` — the numbers v1.0.0 is compared with.
- **Build checks:** `make check64`, `make ksize`, the image limit of the bootloader, CRC-32 of the kernel.
- **ELF loader:** `exec <file> [args]` runs static ELF32 programs in ring 3 (argc/argv, 16 KB stack with a guard page, validated input); `elfinfo`.
  See `docs/elf-loader.md`.
- **File systems:** a VFS layer, a RAM disk, initrd, and its own **QlsFS** (flat, up to 64 files, name up to 23 characters) on an ATA disk.
- **Shell (kernel and ring 3):** a line editor (cursor, history, Home/End/Delete, PageUp/PageDown), colored output, `help` split into topics.
  Kernel shell commands: `help`, `clear`, `echo`, `uptime`, `bench`, `date`, `dmesg`, `mount`, `ls`, `write`, `cat`, `rm`, `diskdump`, `format`,
  `exec`, `elfinfo`, `ps`, `caps`, `bg`, `jobs`, `stop`, `cont`, `kill %N`, `frames`, `lspci`, `mouse`, `console`, `kbd`, `sh`, `lend`,
  `dispatcher`, `reboot`, `shutdown`, and the self-tests (`help tests`).
- **ARM port:** PL011 UART, GICv2, the ARM generic timer, PSCI (reset/off), an interactive shell and multitasking. The console is right in the terminal (UART).

## What is not there yet

- ARM has no paging/PMM, ring 3 or system calls (stubs).
- QlsFS: no directories, and deleting a file does not free space on the disk.
- No USB keyboard without PS/2 emulation, no serial-console input for headless boards, no networking, graphics mode or sound.
- Timeouts on channel waits, shared memory for big data (messages are 64 bytes; the file protocol will send several).

## The architecture in one picture

```
boot/boot.asm (x86 only)  ->  kernel/arch/<arch>/  (entry + drivers)
                                        |
                                        v
                    kernel/src/kernel.c : kernel_main()   (common code)
                    tasks, handles, channels, VFS, QlsFS, shell
                                        |
                          hal_* interface (kernel/hal/hal.h)
                                        |
                         x86 (kernel/arch/x86)  |  ARM (kernel/arch/arm)

ring 3:  dispatcher  ->  services (consoled, kbdd, ...)  <->  clients (sh, programs)      all talk over channels and handles
```

The main rule: the common code in `kernel/src/` does not touch hardware directly, only through the functions from `kernel/hal/hal.h`.
See `ARCHITECTURE.md`, `docs/architecture.md` and `docs/hal-design.md` for details.

## Folder structure

```
boot/                 the x86 bootloader
kernel/src/           common code (scheduler, handles, channels, heap, file systems, shell)
kernel/include/       common headers
kernel/hal/hal.h      the HAL interface
kernel/arch/x86/      everything x86-specific
kernel/arch/arm/      everything ARM-specific
user/                 ring-3 programs (ELF): dispatcher, services, shell, tests + crt0/linker script
tests/                host-side tests and the QEMU harness (`make test`, `make test-*-host`, `make test-*`)
tools/                crc32, usb-write.sh
ARCHITECTURE.md       the architecture contract of the capability microkernel (read this first)
docs/                 architecture.md (how the code is split), hal-design.md, elf-loader.md, hardware-checklist.md (running on a real PC), history.md (old README)
```

## How to build and run

| Method | File | When |
|---|---|---|
| WSL2 + a QEMU window from WSL | `run.bat` (`run.bat arm` — ARM) | WSL2 works (needs "Virtual Machine Platform") |
| WSL1 for building + QEMU for Windows | `run_wsl1.bat` | WSL2 is unavailable; needs `qemu-system-i386.exe` for Windows |
| A USB stick or a real PC | `make usb.img`, `make usb-write DEV=/dev/sdX` | legacy/CSM boot; see `ARCHITECTURE.md`, goal 27 |

Manually in WSL: `make`, `make run`, `make run-headless`, `make arm`, `make run-arm`.

## Run logs

The `logs/` folder next to the scripts: `last-build.log` (the full output), `last-errors.log` (errors only), `last-qemu.log` (QEMU's internal log),
`last-arm-serial.log` (kernel output to the UART, ARM only), `run-bat.log` (the actions of the `.bat` itself). The 20 most recent runs are kept.

## Development history

Stages so far: HAL + ARM + ACPI/PSCI → ATA/IDE, RTC, PCI, a terminal with colors and scrolling → QlsFS v0 → mouse, VFS → open/read/write/close, RAM disk,
initrd → preemptive multitasking → separate process address spaces → rename FlowOS → QlsOS (the names QuaOS and QuasarOS turned out to be taken) →
ELF loader, `fork`, `waitpid`, signals (the `unix-13` state, the last POSIX-style snapshot) → the capability microkernel plan: block A, block B (handles
and channels), block C (spawn, result channel, task control, revoke; `fork`/`wait` deleted), block D (dispatcher, console and keyboard services,
ring-3 shell, `caps`, the bootloader for real hardware).

**31 October 2026, 14:47 — the first stable boot of QlsOS on real hardware** (ASUS K50IJ laptop, from a USB stick).

The source is hosted on GitHub: https://github.com/qlsdev/QlsOS .
Working snapshots between plan tasks are kept as zip files. More about the early stages and troubleshooting: `docs/history.md`.
