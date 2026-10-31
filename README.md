# QlsOS (Quick Light Secure OS)

**QlsOS** (*Quick Light Secure OS*) is a small **capability microkernel** written from scratch: its own bootloader, kernel,
drivers and shell, no third-party libraries. The kernel is fully its own (Linux/WSL is only used as the build tool).
Runs on **x86 (32-bit)** and **ARM (Cortex-A15, QEMU `virt`)**; the x86 port boots in QEMU and has a bootloader prepared for real
PCs (BIOS, USB stick). Target of the first release: **v1.0.0**.

Design goals: **small** (kernel code under 64 KiB for v1.0.0), **safe** (a task touches only what it holds a handle to),
**fast to start programs**, and light enough for the weakest IoT devices.

> Project summary and status: [`PROJECT.md`](PROJECT.md) · Architecture contract: [`ARCHITECTURE.md`](ARCHITECTURE.md) · Code layout: [`docs/architecture.md`](docs/architecture.md)

## Design in short

- **Everything goes through channels and handles.** A handle is the only way a task refers to a kernel object. A task sees only the
  handles it was given; rights can only be narrowed, never widened; a handle can be revoked or given a time limit.
- **No PIDs, no signals, no `fork`/`wait`.** A program is started with `spawn`; how it ended arrives as one message on a result
  channel; a task is stopped or killed through a task handle.
- **The system lives in userspace.** A ring-3 *System Dispatcher* starts services and hands channels to clients; the console
  (VGA) and the keyboard are ring-3 driver tasks; the shell exists as a ring-3 program (`sh`) too. The in-kernel shell stays as
  the test and fallback shell. Device access is itself a capability (memory regions, I/O ports, interrupt lines as handles).
- **Code is 64-bit clean** (`uintptr_t`, `size_t`, fixed-width types, enforced by `make check64`); a 64-bit port and other platforms
  come after v1.0.0 through the HAL.
- **Every step is closed by a test** (host-side under ASan/UBSan and/or in QEMU).

## What works today

- **x86:** own two-stage bootloader (reads the BIOS memory map, uses LBA or CHS as the BIOS allows, **CRC-32 check of the kernel before
  the jump**), protected mode, paging, physical memory manager, heap, preemptive round-robin scheduler, ring-3 tasks with their own
  page directories, `int 0x80` syscalls, PS/2 keyboard and mouse, VGA text console with scrollback, ATA/IDE, RTC, PCI, ACPI reboot/shutdown.
- **Tasks:** ELF32 loader, `spawn` from ring 3 (the new task holds exactly the handles it is given), result channel, task control
  handle (stop / resume / kill), per-task limits (memory frames, handles), clean fault handling (a crash kills only that task),
  blocking `sleep` and blocking channel calls (no polling, no CPU while waiting).
- **Handles and channels:** handle table per task, channels with bounded queues (64-byte messages), handles passed in messages
  (same or fewer rights), `revoke` and handle time limits, exact freeing of every object (checked by tests), `caps` to see who holds what.
- **Userspace system:** System Dispatcher with supervised services, console service (`consoled`) and keyboard service (`kbdd`) as ring-3
  drivers with automatic fallback to the kernel driver, ring-3 shell `sh` with `run prog [args] [--grant name[:rights]]`.
- **Storage:** VFS layer, RAM disk, initrd and **QlsFS** (flat file system on the ATA disk).
- **Real hardware preparation:** USB-bootable image (`make usb.img`, safe writer `make usb-write`), a memory map that tolerates real BIOSes,
  no hangs on machines without a COM port, ACPI tables above 256 MB reachable (tested with 32 MB ... 3.5 GB of RAM in QEMU).
- **ARM:** UART shell, GICv2, generic timer, PSCI reset/off, multitasking (no paging/ring 3 yet).

Kernel size today: about 77 KB of code (`make ksize`); the target for v1.0.0 is 64 KiB, reached when the shell, the old drivers and the
in-kernel tests leave ring 0.

## Requirements

- Linux or **Windows + WSL**
- `nasm`, `gcc` (with `-m32` support), `binutils`, `make`, `qemu-system-i386`, `python3` (the QEMU test harness)
- ARM port only: `gcc-arm-none-eabi`, `qemu-system-arm`

```
sudo apt install nasm build-essential gcc-multilib python3 qemu-system-x86
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi qemu-system-arm   # ARM only
```

## Quick start

**Windows**

| Situation | Run |
|---|---|
| WSL2 works (WSLg shows the QEMU window) | `run.bat` (ARM: `run.bat arm`) |
| Only WSL1 (no "Virtual Machine Platform") | `run_wsl1.bat` — builds in WSL1, runs QEMU for Windows |

Logs go to the `logs/` folder next to the scripts (`last-build.log`, `last-errors.log`, `last-qemu.log`, ...).

**Linux / inside WSL**

```
make                # build disk.img (+ disk_data.img on first run)
make run            # QEMU with a window
make run-headless   # QEMU without a window, serial log in the console
make arm            # build the ARM kernel
make run-arm        # run it in qemu-system-arm -M virt (UART in the terminal)
make usb.img        # image for a USB stick (BIOS/CSM boot); make usb-write DEV=/dev/sdX writes it safely
make clean
```

**Checks**

```
make check64             # shared code must be 64-bit clean
make ksize               # kernel size against the budget
make test                # ELF loader and user stack on the host (ASan/UBSan, no QEMU)
make test-handle-host  test-channel-host  test-task-host  test-revoke-host  test-caps-host  ...   # the kernel logic on the host
make test-spawn  test-result  test-ctl  test-dispatcher  test-sh  test-run  test-caps  ...        # in QEMU (each one also runs the earlier self-tests)
make test-boot  test-usbimg  test-hw  test-mmap  test-crc32                                       # bootloader, images, odd machines
```

The full list of targets is in the `Makefile`; what each test checks is written in [`ARCHITECTURE.md`](ARCHITECTURE.md).

## First steps in the shell

```
help                  list of help topics        help all   everything
ps                    tasks (state, CPU ticks)   caps [N]   who holds which handles and where each came from
exec hello            run a program (ELF) in ring 3; the shell waits for its result
bg spin               run in the background      jobs  stop %N  cont %N  kill %N   (through the task handle)
sh                    the shell in ring 3 (it has the screen and the keyboard until `exit`)
                      inside it:  run hello   run prog [args] --grant name:rights
dispatcher start      start the System Dispatcher and its services      (list | ping <i> <text> | kill <i> | ...)
console  kbd          the screen / the keyboard are the services'  (on | off | kill; kbd also status)
faulttest  sleeptest  blocktest  handletest  channeltest  spawntest  resulttest  ctltest  ...   self-tests (help tests)
bench                 boot/spawn baseline, written to baseline.txt
write note hello      create a file on QlsFS     cat note       print it
```

`exec` programs available out of the box: `hello`, `crash`, `sleeper`, `nop`, `spin`. Full command list: `help all`.

## Repository layout

```
boot/               x86 bootloader (two stages)
kernel/src/         arch-independent code (shell, scheduler, handles, channels, heap, VFS, QlsFS, ...)
kernel/include/     arch-independent headers
kernel/hal/hal.h    the HAL contract between common code and the architectures
kernel/arch/x86/    everything x86-specific (drivers, paging, ring 3, syscalls, ELF loader, in-kernel self-tests)
kernel/arch/arm/    everything ARM-specific
user/               ring-3 programs (C, static ELF): dispatcher, services, drivers, shell, tests + crt0 + linker script + usys.h
tests/              host-side tests (ASan/UBSan) and the QEMU test harness
tools/              crc32 (image check), usb-write.sh (safe image writer)
ARCHITECTURE.md     the architecture contract of the capability microkernel
docs/               architecture.md, hal-design.md, elf-loader.md, hardware-checklist.md, history.md
```

## Limits to know about

- The bootloader loads a kernel of at most **512 KiB** (`make` fails loudly above that) and refuses a kernel whose CRC-32 does not match.
- QlsFS is flat: up to 64 files, names up to 23 characters; `rm` does not reclaim disk space.
- ARM has no paging, PMM, ring 3 or syscalls yet.
- The x86 shell reads the PS/2 keyboard: a PC whose only keyboard is USB without BIOS PS/2 emulation has no input yet.
- The physical memory manager manages the first 64 MB.
- If QEMU "stops working" after several launches, a previous QEMU is probably still holding
  `disk.img` — see the troubleshooting section in [`docs/history.md`](docs/history.md)
  (`wsl --shutdown` from Windows fixes it).

## License

[MPL-2.0](https://github.com/qlsdev/QlsOS#MPL-2.0-1-ov-file) (Mozilla Public License 2.0).
