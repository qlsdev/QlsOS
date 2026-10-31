# QlsOS on real hardware: checklist

Run this on every new machine and write the result into the table at the end. First machine: ASUS K50IJ, 2026-10-31 (details: `ARCHITECTURE.md`, section 27.4).

## 0. Before you start

- [ ] The machine has a **legacy BIOS or UEFI with CSM / "legacy boot"**. QlsOS boots through the BIOS (`int 0x13`); a pure UEFI boot is not supported.
- [ ] Back up anything you care about on the machine. QlsOS does not write to a disk it cannot see, but do not rely on that for data you cannot lose.
- [ ] A USB stick you do not need (the whole stick is overwritten).
- [ ] The machine has a built-in or PS/2-emulating keyboard (a USB-only keyboard without BIOS PS/2 emulation gives the shell no input yet).

## 1. Make the stick

1. `make usb.img` (Linux / WSL).
2. Write it:
   - Linux: `make usb-write DEV=/dev/sdX` (it refuses partitions, mounted or non-removable disks, and verifies the write by reading it back);
   - Windows: **Rufus** with `usb.img` chosen as the boot method (image mode / "DD Image"), or Win32DiskImager. WSL1 cannot reach USB devices.
3. Do **not** let Windows format the stick afterwards (it may offer to; the stick has no file system, that is normal).

## 2. Boot

1. Plug the stick in, switch the machine on and open the **boot menu** (Esc,a F12, F8 or F9, depending on the maker; ASUS laptops: Esc).
2. Choose the stick: the entry **USB-HDD** first, **USB-FDD** if the first one does not boot.
3. If the stick is not in the menu: BIOS setup (F2 / Del) -> Boot: enable USB boot / legacy boot, turn off Fast Boot / Boot Booster.

What you should see: the loader prints `kernel crc ok`, then the kernel log, the banner `QlsOS v0.1.0` and the prompt.

| You see | Meaning | Send me |
|---|---|---|
| `BAD CRC` | the kernel arrived damaged (a read went wrong); the machine halts on purpose | a photo; try another USB port / another stick |
| `DISK ERR` | a BIOS read failed or stage 2 is not whole | a photo; the machine model and the boot entry used |
| `A20 ERR` | the A20 line could not be enabled | a photo; the machine model |
| black screen, no text | the loader did not start, or the BIOS did not accept the stick | the boot entry used (USB-HDD / USB-FDD), the BIOS mode |
| the machine resets by itself | a crash before the shell | a photo of the last text, if any |

## 3. Commands, in this order

Safe on every machine (they only read, or work in RAM):

| # | Command | What to check |
|---|---|---|
| 1 | `help` | the topic list appears; the keyboard types and Backspace / arrows work |
| 2 | `frames` | prints `physical frames: N free / M total`; **16384 total** is the PMM ceiling (64 MB), not the machine's RAM |
| 3 | `lspci` | the PCI devices of the machine; write down the SATA controller (class `0106`, `prog_if 01` = AHCI) |
| 4 | `uptime`, `date` | the timer ticks (100 per second); the date comes from the RTC |
| 5 | `ps`, `caps` | the tasks and their handles are listed |
| 6 | `mouse`, then move the touchpad / mouse | the values change (PS/2) |
| 7 | `exec hello` | the program runs in ring 3 and exits with code 0 |
| 8 | `sh` | the ring-3 shell starts; inside it `help`, `echo hi`, `run hello`; `exit` returns |
| 9 | `kbd status`, `console` | both services are reported |
| 10 | `bench` | finishes and writes `baseline.txt` (`cat baseline.txt`) |
| 11 | `diskdump 0` | read-only. `no IDE disk found` = the kernel does not see a disk (AHCI mode); then `ramfs` is active and files do not survive a reboot |
| 12 | `mount` | which file system is active (`ramfs` or `qlsfs`) |
| 13 | `faulttest` | every kind of crash kills only the one task; no `FAIL` |
| 14 | `reboot` | the machine restarts (it will go to its normal system unless the stick is first in the boot order: that is expected) |
| 15 | `shutdown` | the machine powers off |

**Do not run on a machine whose disk you care about:** `format` (erases the disk if the kernel sees it) and `disktest` (writes a test sector). The kernel formats a disk
at boot only if its first 64 sectors (32 KiB) are all zero, so a disk with a system on it is never touched by the boot itself.

## 4. Result table

One row per machine. `ok` / `fail` / `n/a`, and a note where it matters.

| Machine | Date | Boot | Keyboard | Touchpad | `frames` | `lspci` | `exec hello` | `sh` | `bench` | Disk seen | `reboot` | `shutdown` | Notes |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ASUS K50IJ (Centrino 2, GM45 + ICH9M, legacy BIOS) | 2026-10-31 | ok (USB-HDD via the Esc menu) | ok | ok | 15828 / 16384 | 22 devices, SATA in AHCI mode | ok | ok | ok | no (AHCI, ramfs active) | ok | ok | `dmesg` empty; files are in RAM; after `reboot` the BIOS boots the internal system again |
|  |  |  |  |  |  |  |  |  |  |  |  |  |  |

## 5. Known limits on real machines

- The memory manager uses the first 64 MB of RAM only.
- No AHCI driver yet: a SATA disk in AHCI mode is invisible; the files of the session live in RAM. (Some BIOSes have an IDE / compatibility mode for SATA; with it the legacy driver may see the disk.)
- No USB keyboard support without PS/2 emulation in the BIOS; no input over a serial port yet.
- `dmesg` printed an empty log on the first machine (in QEMU it holds the kernel's events): not understood yet.
