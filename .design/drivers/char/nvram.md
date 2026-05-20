# Tier-3: drivers/char/nvram.c — CMOS NVRAM chardev /dev/nvram (PC RTC scratch area, checksum + capability gates)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/char/nvram.c
  - include/uapi/linux/nvram.h
  - include/linux/nvram.h
-->

## Summary

`drivers/char/nvram.c` (~550 lines) exports `/dev/nvram` (misc minor `NVRAM_MINOR = 144`) — the user-visible interface to the "non-volatile RAM" inside the PC RTC chip (MC146818-compatible), i.e. the CMOS scratch area above byte 14 used historically by BIOS for boot configuration. The driver wraps the per-architecture `nvram_read_byte` / `nvram_write_byte` accessors (in `include/linux/nvram.h`; PC variant calls `CMOS_READ` / `CMOS_WRITE` from `<linux/mc146818rtc.h>`, holding `rtc_lock` to serialize with the RTC driver). It exposes `read` / `write` / `llseek` / `ioctl` via the misc-device `file_operations`, an `NVRAM_INIT` / `NVRAM_SETCKS` ioctl pair (uapi `_IO('p', 0x40)` / `_IO('p', 0x41)`) that zeroes or recomputes the per-platform checksum byte, and a `/proc/driver/nvram` seqfile that decodes the well-known BIOS layout (floppy types, video, etc.).

This Tier-3 captures the chardev surface, the checksum semantics, the capability gates around init / setcks, and the locking against the RTC driver.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `nvram_misc` (`miscdevice`, minor = NVRAM_MINOR) | misc-device registration | `drivers::nvram::Device` |
| `nvram_misc_fops` (llseek/read/write/ioctl/open/release) | chardev file_operations | `Device::FileOps` |
| `nvram_misc_open(inode, file)` / `_release` | per-fd init | `Device::open` / `release` |
| `nvram_misc_llseek(file, offset, origin)` | clamp seek to `pc_nvram_get_size()` | `Device::lseek` |
| `nvram_misc_read(file, buf, count, ppos)` | per-byte CMOS read | `Device::read` |
| `nvram_misc_write(file, buf, count, ppos)` | per-byte CMOS write | `Device::write` |
| `nvram_misc_ioctl(file, cmd, arg)` | `NVRAM_INIT` / `NVRAM_SETCKS` | `Device::ioctl` |
| `__nvram_read_byte(i)` / `pc_nvram_read_byte(i)` | per-arch accessor (PC variant) | `Pc::read_byte` |
| `__nvram_write_byte(c, i)` / `pc_nvram_write_byte(c, i)` | per-arch writer | `Pc::write_byte` |
| `__nvram_check_checksum()` | verify BIOS-defined CMOS checksum byte | `Pc::check_checksum` |
| `__nvram_set_checksum()` | recompute + store checksum byte | `Pc::set_checksum` |
| `pc_nvram_initialize()` | zero CMOS region then set checksum | `Pc::initialize` |
| `pc_nvram_get_size()` | NVRAM_BYTES = 114 (PC layout) | `Pc::size` |
| `nvram_mutex` | serialize chardev ops | `Device::Lock` |
| `nvram_state_lock` (`spinlock_t`) | per-byte access serialization | `Pc::IoLock` |
| `pc_nvram_proc_read(buf, seq, offset)` / `nvram_proc_read(seq, offset)` | `/proc/driver/nvram` dump | `ProcView::dump` |
| `NVRAM_INIT` (`_IO('p', 0x40)`) | zero NVRAM + checksum | `Ioctl::INIT` |
| `NVRAM_SETCKS` (`_IO('p', 0x41)`) | recompute checksum only | `Ioctl::SETCKS` |
| `NVRAM_MINOR` (144) | misc minor constant | `Device::MINOR` |

## Compatibility contract

REQ-1: Misc-device registered with `minor = NVRAM_MINOR (144)` and `name = "nvram"`; the device node is `/dev/nvram` by udev convention with default mode `0660 root:nvram` (Rookery policy tightens to `0600 root:root`).

REQ-2: `nvram_misc_read` / `_write` operate one byte at a time over offsets `[0, pc_nvram_get_size()-1]`. PC size = 114 bytes (offsets 14..127 of the 128-byte CMOS, since 0..13 are the RTC live registers).

REQ-3: Per-byte access takes `rtc_lock` (in `pc_nvram_read_byte` / `_write_byte`) so that reads/writes do not race with `drivers/rtc/rtc-cmos.c` programming the alarm or time registers.

REQ-4: Read path validates the CMOS checksum at the start of the operation via `__nvram_check_checksum()`; mismatch triggers a kmesg warning but the read still proceeds (so an admin can read the corrupted region before fixing it).

REQ-5: Write path also validates checksum at start; mismatch still allows the write to proceed and the post-write checksum is automatically updated by `__nvram_set_checksum()` after the last byte.

REQ-6: `NVRAM_INIT` ioctl: zero the entire 114-byte CMOS scratch area then recompute checksum. Requires `CAP_SYS_ADMIN` (line 324).

REQ-7: `NVRAM_SETCKS` ioctl: recompute and write the checksum byte only; the existing byte contents are preserved. Requires `CAP_SYS_ADMIN` (line 336).

REQ-8: `llseek` clamps to `[0, pc_nvram_get_size()]`; out-of-range returns `-EINVAL`.

REQ-9: `/proc/driver/nvram` exposes a decoded view (floppy A/B types per `floppy_types[]`, video card type per `gfx_types[]`, plus the raw checksum status). Readable without elevated capability — but exposes a tiny amount of platform info; on hardened builds `proc_create_single_data` is gated behind CAP_SYSLOG.

REQ-10: PowerPC / Atari / SPARC variants exist in arch-specific paths and override `nvram_read_byte` / `_write_byte` via `include/linux/nvram.h`. This Tier-3 covers PC; the chardev surface is identical.

## Acceptance Criteria

- [ ] AC-1: `ls -l /dev/nvram` shows misc-major + minor 144 after `nvram.ko` loads on a PC.
- [ ] AC-2: `dd if=/dev/nvram bs=1 count=114 of=/tmp/cmos.bin` returns 114 bytes; first byte corresponds to CMOS offset 14.
- [ ] AC-3: `dd if=/tmp/cmos.bin of=/dev/nvram bs=1 count=114` succeeds and recomputes checksum (`__nvram_set_checksum` invoked) without provoking a checksum-error printk on next boot.
- [ ] AC-4: `ioctl(fd, NVRAM_INIT)` as root succeeds and zeroes the scratch area; as non-root returns `-EACCES`.
- [ ] AC-5: `ioctl(fd, NVRAM_SETCKS)` as root succeeds; without CAP_SYS_ADMIN returns `-EACCES`.
- [ ] AC-6: `cat /proc/driver/nvram` decodes floppy + gfx types and prints "Checksum status: valid" on a freshly checksummed CMOS.
- [ ] AC-7: Concurrent `read /dev/nvram` and `hwclock --read` (rtc-cmos) do not corrupt each other (rtc_lock serialization).
- [ ] AC-8: `lseek(fd, 200, SEEK_SET)` returns `-EINVAL` (beyond PC size).

## Architecture

`Device` lives in `drivers::nvram::Device`:

```
struct Device {
  misc: MiscDevice {
    minor: NVRAM_MINOR,           // 144
    name: "nvram",
    fops: &NVRAM_MISC_FOPS,
  },
  nvram_mutex: Mutex<()>,          // serialize chardev ops
  state_lock: SpinLock<()>,        // per-byte CMOS access (paired with rtc_lock)
}

struct PcAccessors {
  // PC layout constants
  size_bytes: usize,               // 114 (offsets 14..127)
  checksum_lo: u8,                 // offset of checksum byte LSB
  checksum_hi: u8,                 // offset of checksum byte MSB
}
```

Read path `nvram_misc_read(file, buf, count, ppos)`:
1. Acquire `nvram_mutex`.
2. Validate `*ppos < size` and clamp `count` to remaining.
3. `__nvram_check_checksum()` — under `nvram_state_lock`, walk bytes summing per-byte values and compare against checksum byte. Mismatch → `printk_once(KERN_WARNING ...)`; read continues.
4. For each requested offset `i`: `kbuf[k++] = pc_nvram_read_byte(i)` which takes `rtc_lock` and emits `inb(RTC_CMD)` + `inb(RTC_DATA)` per the MC146818 protocol.
5. `copy_to_user(buf, kbuf, count)`.
6. Release `nvram_mutex`.

Write path `nvram_misc_write(file, buf, count, ppos)`:
1. Acquire `nvram_mutex`.
2. `copy_from_user(kbuf, buf, count)`.
3. `__nvram_check_checksum()`; mismatch → kmesg warning.
4. For each requested offset `i`: `pc_nvram_write_byte(kbuf[k++], i)`.
5. `__nvram_set_checksum()` — recompute and write checksum byte.
6. Release `nvram_mutex`.

Ioctl path `nvram_misc_ioctl(file, cmd, arg)`:
- `NVRAM_INIT`:
  - `if (!capable(CAP_SYS_ADMIN)) return -EACCES`.
  - Acquire `nvram_mutex`.
  - `pc_nvram_initialize()`: zero offsets 0..size-1, then `__nvram_set_checksum`.
  - Release.
- `NVRAM_SETCKS`:
  - `if (!capable(CAP_SYS_ADMIN)) return -EACCES`.
  - Acquire `nvram_mutex`.
  - `__nvram_set_checksum`.
  - Release.

`__nvram_check_checksum` (PC variant):
1. `sum = 0` for offset `i` in 0..size-1 (excluding checksum byte itself): `sum += pc_nvram_read_byte(i)`.
2. `stored = (pc_nvram_read_byte(checksum_hi) << 8) | pc_nvram_read_byte(checksum_lo)`.
3. Return `(sum & 0xffff) == stored`.

`__nvram_set_checksum`: same sum computation, then write `stored = sum & 0xffff` back into the two checksum bytes.

`/proc/driver/nvram` `nvram_proc_read(seq, offset)`:
1. Snapshot the 114-byte region under `nvram_mutex` + `state_lock`.
2. Call `pc_nvram_proc_read(buf, seq, offset)` which:
   - Prints "Checksum status: %s" (valid / invalid).
   - Decodes well-known BIOS offsets: floppy types from `floppy_types[]`, video card from `gfx_types[]`, hard-drive types, equipment byte.
3. Output goes to the seq_file.

Locking discipline:
- `nvram_mutex` is the outer per-device mutex; held across multi-byte read/write/ioctl.
- `nvram_state_lock` (spinlock) wraps the checksum walk so individual byte sequences are atomic against IRQ-time RTC accessors.
- `rtc_lock` (in `<linux/mc146818rtc.h>`) is taken by `pc_nvram_read_byte` / `_write_byte` per byte; this serializes against the rtc-cmos driver's alarm/time-set paths.

## Hardening

- `__nvram_check_checksum` runs on every read and write, catching firmware-side or RAM-rot corruption.
- Out-of-range offsets refused at `llseek` and at per-byte access; the driver cannot accidentally touch the RTC live registers (offsets 0..13).
- `NVRAM_INIT` is a destructive operation (zeroes BIOS config); gated on CAP_SYS_ADMIN.
- `NVRAM_SETCKS` can be used to "validate" an attacker-prepared CMOS without rewriting bytes; gated on CAP_SYS_ADMIN to prevent stealth tampering by a non-admin process that somehow has write to /dev/nvram via udev misconfig.
- Per-byte access pairs with `rtc_lock` so a buggy CMOS write cannot corrupt the RTC clock.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist the small per-op kernel buffer used in `nvram_misc_read` / `_write`; bound `copy_to_user`/`copy_from_user` against `pc_nvram_get_size()` (114 bytes) at all times.
- **PAX_KERNEXEC** — `nvram_misc_fops`, `nvram_misc` miscdevice descriptor, `floppy_types[]`, and `gfx_types[]` live in `__ro_after_init` text/data; no runtime patching.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `nvram_misc_read`, `_write`, `_ioctl`, and `__nvram_check_checksum`/`_set_checksum` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `nvram_misc.this_device` and on the per-open file refcount; overflow trap defeats UAF on race between fd-close and module unload.
- **PAX_MEMORY_SANITIZE** — zero-on-free for any heap buffer used to snapshot CMOS in `nvram_proc_read` so prior BIOS configuration bytes (potentially encoding boot policy) do not bleed across reads. Also enforce secure-erase semantics in `pc_nvram_initialize`: explicit byte-by-byte zero, no compiler short-circuit.
- **PAX_UDEREF** — SMAP/PAN enforced on every `copy_to/from_user`; ioctl `arg` not used as a pointer by `NVRAM_INIT`/`NVRAM_SETCKS` but the gate is present for forward compat.
- **PAX_RAP / kCFI** — `nvram_misc_fops` and the misc-class indirect-call table are typed; misrouted fops substitution traps.
- **GRKERNSEC_HIDESYM** — suppress `%p` of the per-arch `nvram_priv` pointers (where present on non-PC arches) and the rtc_lock pointer in trace paths; gate behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict checksum-mismatch warning, `NVRAM_INIT` audit banner, and per-arch probe banners to CAP_SYSLOG so attackers cannot probe boot configuration via dmesg.
- **/dev/nvram CAP_SYS_ADMIN** — chardev mode default tightened from upstream `0660 root:nvram` to `0600 root:root`; even `nvram` group members cannot open without explicit policy. `NVRAM_INIT` and `NVRAM_SETCKS` already enforce CAP_SYS_ADMIN at lines 324 + 336.
- **CMOS index/data port gated** — `CMOS_READ`/`CMOS_WRITE` use I/O ports `0x70`/`0x71`; under STRICT_DEVMEM these are off-limits to /dev/port. The only way to reach them is through this driver, which enforces the chardev + ioctl gating above.
- **Checksum validation** — every read/write triggers `__nvram_check_checksum`; mismatch produces a rate-limited audit record so an attacker overwriting CMOS via DMA/SMM cannot silently subvert boot config.
- **Secure-erase MEMORY_SANITIZE** — `pc_nvram_initialize` writes deliberate zeros (no implementation-defined memset shortcut) and re-reads each byte to confirm; mismatch returns `-EIO` and audit-logs a hardware fault.
- **Proc view gating** — `/proc/driver/nvram` made CAP_SYSLOG-gated under Rookery policy so the decoded BIOS layout (floppy/video/HDD types) cannot be enumerated by unprivileged processes.
- **Per-op mutex held across checksum** — `nvram_mutex` covers both the byte ops and the checksum recompute; defense against TOCTOU between write and `__nvram_set_checksum`.

Rationale: `/dev/nvram` is a tiny chardev but it touches platform-firmware-managed state (boot order, virtualization toggles encoded by some BIOSes, etc.) and an unrestricted write can survive reboot — a uniquely persistent attack vector against integrity. CAP_SYS_ADMIN gating on `NVRAM_INIT`/`NVRAM_SETCKS`, default `0600` device mode, checksum validation on every read/write with audit, MEMORY_SANITIZE secure-erase semantics during init, and lockdown-style gating of the I/O-port path turn /dev/nvram from "BIOS scratchpad anyone in group nvram can touch" into a CAP_SYS_ADMIN-only, audited, integrity-checked surface.

## Open Questions

- Whether per-arch variants (PPC, Atari, SPARC) should share the CAP_SYS_ADMIN gate uniformly — they currently do via the shared `nvram_misc_ioctl` dispatcher.
- Should `NVRAM_INIT` be additionally gated by `security_locked_down(LOCKDOWN_INTEGRITY_MAX)` since it destroys boot config? Current policy: yes, but not yet wired in upstream.

## Out of Scope

- `drivers/rtc/rtc-cmos.c` (covered in `drivers/rtc/rtc-cmos.md` future Tier-3)
- PPC / Atari / SPARC arch-specific NVRAM (different physical address spaces, separate Tier-3 candidates)
- BIOS layout interpretation beyond floppy/video decoding
- Implementation code
