# HAL design (stages 89–90)

## Why a HAL at all

Before this stage all of `kernel/src/` was in fact x86 code: `kernel.c` called
`vga_clear()`, `gdt_install()`, `keyboard_wait_char()` directly, `shell.c`
pulled `inb`/`outb` to reboot via the 8042 controller, `task.c` assembled
an x86 stack frame by hand (`push ebx,esi,edi,ebp`). A port to ARM without
the split would have meant copy-pasting the whole shell/task/kheap and rewriting them
from scratch — that is, two different kernels under one banner.

The rule is simple now: **`kernel/src/` has no right to touch hardware
directly** — only through the functions declared in `kernel/hal/hal.h`. Each
architecture (`kernel/arch/x86/`, `kernel/arch/arm/`) implements this same set of
functions in its own way.

## Where the split runs

**Common (`kernel/src/`, built once, linked with any backend):**
- `kernel.c` — the entry point, calls only `hal_*` and `arch_platform_init()`/`arch_extra_demo()`
- `shell.c` — the line editor, history, commands — knows only about `hal_console_*`/`hal_input_*`
- `task.c` — the scheduler (preemptive round-robin on x86) — knows only about `hal_context_switch`/`hal_new_task_stack`/`hal_cpu_halt`
- `kheap.c` — a static arena in `.bss`, does not touch hardware at all, so it did not change by a single line
- `kprintf.c` — used to live inside `vga.c`, now common, writes through `hal_console_putc`
- `string.c` — does not touch hardware

**x86-specific (`kernel/arch/x86/`):**
GDT/IDT/PIC, VGA text-mode + hardware cursor, the PS/2 keyboard, COM1,
the PIT timer, E820/PMM/paging, ring3+syscalls (the usermode demo). All of this
stayed exactly as it was — it just moved and grew `hal_x86.c`,
a thin layer of adapters.

**ARM-specific (`kernel/arch/arm/`):**
PL011 UART (both output and input — a terminal instead of VGA+PS/2), GICv2, the ARM
generic timer (not the PIT — fundamentally different hardware), its own boot/vector
table in assembly, its own `context_switch.S`.

## Non-obvious places worth remembering

- **The "cursor" on ARM is ANSI escape sequences** (`ESC[row;colH`),
  and the position is tracked in software (`cur_row`/`cur_col` in
  `hal_arm.c`), unlike x86, where the cursor is a real hardware VGA
  register. `hal_console_put_at` (for the spinner in the corner) on ARM does a
  save/restore of the cursor (`ESC 7` / `ESC 8`), so as not to disturb the shell's cursor.
- **Special keys on ARM** are ANSI sequences (`ESC [ A/B/C/D`,
  `ESC [ 3 ~` and so on), parsed in `hal_arm.c` into the same `HAL_KEY_*` codes that
  on x86 come from PS/2 scancodes. `shell.c` knows nothing about this.
- **`hal_new_task_stack`** — previously the x86 register layout for the
  "fake" stack of a new task was hardcoded directly in the
  arch-independent `task.c`. Now it is a separate HAL function: x86 pushes
  `ebx,esi,edi,ebp`+entry (to match `context_switch.asm`), ARM — `r4-r11,lr`
  (to match `context_switch.S`). `task.c` just calls
  `hal_new_task_stack(stack_top, entry)`, without knowing the details.
- **The timer ticks differently**: x86 — the PIT through the PIC/IDT (vectors 32-47),
  ARM — the architected timer (the CP15 registers `CNTP_TVAL`/`CNTP_CTL`) through
  GICv2 (IRQ ID 30, the non-secure physical timer PPI on the QEMU `virt` machine).
  Both in the end simply increment the same `hal_timer_ticks()`.

## What is NOT ported to ARM (honestly, we don't hide it)

- Paging/MMU, the PMM, kheap on top of real physical memory (ARM currently
  uses the same static `.bss` arena as before — it works, but
  does not scale to the real memory size of the board)
- ring3/usermode/syscalls (the demo on x86 goes through `int 0x80` — on ARM it would
  be SVC + a different set of registers, a separate piece of work)
- Interrupt-driven UART RX (currently — polling the `FR` register inside
  a `wfi` loop, waking up on every timer tick — it works, but is not as
  elegant as IRQ1 on x86 for PS/2)

## Reboot/poweroff (stage 93 — done and verified on both architectures)

- x86: `kernel/arch/x86/src/acpi.c` — an honest parse of RSDP->RSDT->FADT->DSDT,
  a minimal AML parser for the `\_S5` package (SLP_TYPa/b), a write to PM1_CNT
  for poweroff, the ACPI Reset Register for reboot with a fallback to an 8042 pulse and
  a triple fault as the last resort.
- ARM: PSCI `SYSTEM_RESET`/`SYSTEM_OFF` via HVC (on QEMU `virt` this
  turned out to be the working conduit) with a fallback to SMC.
- Both paths have been really verified in a live QEMU (not just compiled):
  on x86 — via `sendkey` in the QEMU monitor (PS/2, standard input is not available
  to the x86 console directly), on ARM — simply via stdin (there input already
  goes through the UART/serial). In both cases the QEMU process either really
  restarted the boot (reboot), or exited by itself (shutdown) — and that is
  the confirmation that the ACPI/PSCI calls reached the chipset/hypervisor, and
  not just that "the code compiles without errors".

## Verified, not only in theory

x86: `make && make run-headless` — run in QEMU, the output matches byte for byte
what it was before the refactoring (the GDT/IDT/PIC/PMM/paging/ring3 demo
work as before).

ARM: `make arm && make run-arm` — built with `arm-none-eabi-gcc` (Cortex-A15),
loaded into `qemu-system-arm -M virt` directly via `-kernel`. It reaches
the interactive shell, input through stdin was really tested (`help`,
`uptime`, `echo ...`) — `uptime` shows a growing tick counter, which means
the GIC + the architected timer + the IRQ dispatcher in `vectors.S` really work, and
not just compile.
