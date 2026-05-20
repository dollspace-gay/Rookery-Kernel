# Tier-3: drivers/char/lp.c — line-printer chardev /dev/lp* over parport (IEEE 1284)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/char/lp.c
-->

## Summary

`drivers/char/lp.c` (~1100 lines) is the classic Linux line-printer driver: it exports `/dev/lp0..lp{LP_NO-1}` (legacy LP_MAJOR=6) and `/dev/lp*` minors layered on top of `drivers/parport/` (`parport_register_driver` / `parport_register_dev_model`). Per-minor it holds an `lp_struct` containing a `pardevice *dev`, IEEE 1284 mode (`current_mode`, `best_mode`), per-printer timeout, and a small bounce buffer. Writes negotiate into the best supported IEEE 1284 mode (`IEEE1284_MODE_ECP` ▸ `_EPP` ▸ `_BYTE` ▸ `_NIBBLE` ▸ `_COMPAT`) and stream bytes through `parport_write`; reads (only meaningful in nibble/byte reverse modes) call `parport_read`. The driver supports IRQ-driven (`parport_yield_blocking`) and polled operation, exports an ioctl surface (`LPCHAR`, `LPABORT`, `LPABORTOPEN`, `LPGETSTATUS`, `LPSETIRQ`, `LPSETTIMEOUT_OLD`/`LPSETTIMEOUT_NEW` 32/64-bit variants, `LPRESET`, `LPGETSTATS`, `LPGETFLAGS`), can be a kernel `console` driver (`lpcons`) writing to the printer for debug, and uses `parport_claim_or_block` to coordinate with other parport clients (e.g. `ppdev`, `lp-scanner` hybrids).

This Tier-3 captures the chardev surface, the IEEE 1284 mode negotiation, and the per-port claim/release pattern that prevents parport multi-client corruption.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct lp_struct` (`lp_table[LP_NO]`) | per-minor state | `drivers::lp::Printer` |
| `lp_fops` (open/release/read/write/ioctl/compat_ioctl) | chardev file_operations | `Printer::FileOps` |
| `lp_open(inode, file)` | minor → claim port, negotiate best mode | `Printer::open` |
| `lp_release(inode, file)` | drain + release | `Printer::release` |
| `lp_write(file, buf, count, ppos)` | streaming write path with mode-aware IEEE 1284 transfer | `Printer::write` |
| `lp_read(file, buf, count, ppos)` | reverse-direction nibble/byte read | `Printer::read` |
| `lp_ioctl` / `lp_compat_ioctl` / `lp_do_ioctl` | ioctl dispatch | `Printer::ioctl` / `compat_ioctl` |
| `lp_set_timeout` / `lp_set_timeout32` / `lp_set_timeout64` | per-printer write-timeout setter | `Printer::set_timeout` |
| `lp_claim_parport_or_block(this_lp)` / `lp_release_parport(this_lp)` | wrap `parport_claim_or_block` / `parport_release` | `Printer::claim` / `release` |
| `lp_preempt(handle)` | parport "I need the port back" callback | `Printer::preempt` |
| `lp_negotiate(port, mode)` | run IEEE 1284 negotiation handshake | `Printer::negotiate` |
| `lp_check_status(minor)` | read/decode printer status (PE, ERROR, SELECT) | `Printer::check_status` |
| `lp_wait_ready(minor, nonblock)` | wait for SELECT/!ERROR with timeout | `Printer::wait_ready` |
| `lp_reset(minor)` | toggle INIT line | `Printer::reset` |
| `lp_attach(port)` / `lp_detach(port)` / `lp_register(nr, port)` | parport driver callbacks | `Subsystem::attach` / `detach` / `register` |
| `lp_console_write(co, s, count)` (`lpcons`) | optional console driver path | `Printer::console_write` |
| `parport_nr[LP_NO]` / `lp_setup(str)` | cmdline binding of minor → parport | `Subsystem::cmdline_setup` |
| `LPCHAR` (0x0601) / `LPABORT` (0x0604) / `LPGETSTATUS` (0x060b) / `LPABORTOPEN` (0x060a) | uapi ioctl vocabulary | `Printer::Ioctl::*` |

## Compatibility contract

REQ-1: At most `LP_NO` (default 8) printers exposed under minors 0..LP_NO-1; per-minor binding established at parport-attach time according to cmdline `lp=parport0,parport1,...` or `lp=none,auto`.

REQ-2: `lp_open` calls `parport_claim_or_block(this_lp->dev)`; blocks until the parport is available or returns `-EAGAIN` for non-blocking opens. On success runs an IEEE 1284 reverse-direction probe to discover `best_mode` (ECP preferred, then EPP, then COMPAT).

REQ-3: `lp_write` segments the user buffer into chunks of `LP_BUFFER_SIZE` (1 KiB), copies to a kernel bounce buffer, then calls `parport_write` in the current mode. On timeout, status is consulted: out-of-paper / printer-fault → return `-EIO`; transient → retry or sleep.

REQ-4: Mode fallback: if the negotiated `best_mode` fails mid-write (e.g. printer dropped to nibble), driver renegotiates to `IEEE1284_MODE_COMPAT` and continues.

REQ-5: `lp_read` requires that the printer support a reverse direction in IEEE 1284 (nibble or byte). Compat-mode-only printers reject reads with `-EIO` after negotiate-fail.

REQ-6: `lp_ioctl` vocabulary:
- `LPGETSTATUS`: read printer status byte (PE/ERROR/SELECT/PAPER_OUT/ONLINE).
- `LPRESET`: pulse INIT.
- `LPABORT`: abort-on-error mode (sticky per fd).
- `LPABORTOPEN`: refuse open-on-error.
- `LPCAREFUL`: enable careful mode (sticky).
- `LPCHAR`: set per-char retry count.
- `LPTIME`: set per-byte wait time.
- `LPSETTIMEOUT_OLD/_NEW`: set write timeout (32 vs 64-bit timespec variants resolved through `lp_set_timeout32`/`lp_set_timeout64`).
- `LPGETSTATS`: get per-printer transfer statistics (gated by CAP_SYS_ADMIN at line 651).
- `LPGETFLAGS`: return current flags.
- `LPSETIRQ`: change parport IRQ binding (gated by CAP_SYS_RAWIO).

REQ-7: `lp_compat_ioctl` shims 32-bit timespec ioctls onto 64-bit kernels via `lp_set_timeout32` reading two `s32` words.

REQ-8: When `CONFIG_LP_CONSOLE=y` and module param `lp=console` is set, the per-printer becomes an alternate `console` device (`lpcons`); `lp_console_write` synchronously claims, transfers, releases. Useful for early printk capture on embedded boards.

REQ-9: `lp_attach` is called by parport core when a new port is registered; it walks `parport_nr[]` and either auto-binds (if `LP_PARPORT_AUTO`) or matches a specific port number.

REQ-10: Per-printer `lp_preempt(handle)` returns `1` when the driver is mid-write to refuse preemption; otherwise returns `0` so other parport clients can take the port.

## Acceptance Criteria

- [ ] AC-1: With a parport stub (`parport_pc.ko` or `parport_serial.ko`) loaded and `lp.ko` modprobed, `/dev/lp0` appears with major 6 minor 0.
- [ ] AC-2: `echo "hello" > /dev/lp0` on a parport-attached printer prints "hello" (or, on a parport loopback, the bytes appear at the other end).
- [ ] AC-3: `ioctl(fd, LPGETSTATUS, &s)` returns the printer status byte.
- [ ] AC-4: `ioctl(fd, LPRESET)` toggles the printer-INIT line, observable on a logic analyzer.
- [ ] AC-5: Concurrent open of `/dev/lp0` and `/dev/ppdev` (same parport) serializes correctly via `parport_claim_or_block`; neither corrupts the other's transfer.
- [ ] AC-6: `LPSETIRQ` without CAP_SYS_RAWIO returns `-EPERM`.
- [ ] AC-7: `LPGETSTATS` without CAP_SYS_ADMIN returns `-EPERM`.
- [ ] AC-8: Setting `lp=parport0` then `lp=none` on cmdline disables auto-binding.
- [ ] AC-9: 32-bit-on-64-bit `LPSETTIMEOUT_OLD` (s32 timespec) through `lp_compat_ioctl` matches the 64-bit caller's result.

## Architecture

`Printer` lives in `drivers::lp::Printer`:

```
struct Printer {
  flags: u32,                      // LP_EXIST | LP_BUSY | LP_ABORT | LP_ABORTOPEN | LP_CAREFUL | LP_NO_REVERSE
  chars: u32,                      // per-char retry limit
  time: u32,                       // per-byte sleep in jiffies
  timeout: Timeout,                // sec + usec write timeout (per-printer)
  dev: Arc<ParDevice>,             // parport client handle
  port: Arc<ParPort>,
  stats: Stats,
  best_mode: IeeeMode,             // ECP / EPP / BYTE / NIBBLE / COMPAT
  current_mode: IeeeMode,
  last_error: u32,
  irq_missed: AtomicBool,
  bounce_buf: KBox<[u8; LP_BUFFER_SIZE]>,
}

static LP_TABLE: [Option<Printer>; LP_NO] = ...;

struct LpFileOps {
  open:    lp_open,
  release: lp_release,
  read:    lp_read,
  write:   lp_write,
  unlocked_ioctl: lp_ioctl,
  compat_ioctl:   lp_compat_ioctl,
}
```

Open path `lp_open(inode, file)`:
1. `minor = iminor(inode)`; validate `minor < LP_NO` and `LP_F(minor) & LP_EXIST`.
2. Acquire `lp_mutex`; refuse if `LP_BUSY` and `O_EXCL`.
3. `lp_claim_parport_or_block(&lp_table[minor])` — calls `parport_claim_or_block(dev)`.
4. If status shows printer error and `LP_ABORTOPEN`, release + return `-ENOSPC` (`-EIO` if generic fault).
5. Probe IEEE 1284: try `parport_negotiate(port, IEEE1284_MODE_ECP)`; if accepted, `best_mode = ECP`; else try EPP, BYTE, NIBBLE, COMPAT.
6. Return to COMPAT (write direction).

Write path `lp_write(file, buf, count, ppos)`:
1. Per-chunk loop while `count` > 0:
   - `nbytes = min(count, LP_BUFFER_SIZE)`.
   - `copy_from_user(bounce_buf, buf, nbytes)` (PAX_USERCOPY-bounded).
   - `parport_write(port, bounce_buf, nbytes)` returns bytes accepted or negative error.
   - On per-byte timeout: `lp_check_status` → consult flags:
     - `LP_F_ABORT` → return `-EIO`.
     - Out-of-paper → `-ENOSPC`.
     - Otherwise sleep `lp_table[minor].time` jiffies and retry up to `chars` times.
   - Renegotiate to COMPAT if the printer dropped the higher mode.
   - Advance `buf += nbytes; count -= nbytes`.
2. Return total bytes written.

Read path `lp_read`:
1. Negotiate to `IEEE1284_MODE_NIBBLE` (reverse direction).
2. `parport_read(port, kbuf, count)` chunks of `LP_BUFFER_SIZE`.
3. `copy_to_user(buf, kbuf, n)`.
4. Renegotiate to COMPAT on close of the read.

Parport claim/release wraps:
- `parport_claim_or_block(dev)`: take the port; may sleep; invokes `preempt` callback on other holders.
- `parport_release(dev)`: hand the port back to the parport core; another client (or another lp minor on shared parport) may now claim.

Console mode (`CONFIG_LP_CONSOLE`):
- `lp_console_write(co, s, count)` performs a synchronous `parport_claim` + `parport_negotiate(IEEE1284_MODE_COMPAT)` + `parport_write` + `parport_release`. Used as a debug printk sink.

## Hardening

- `LP_BUFFER_SIZE` (1 KiB) is the only bounce buffer per printer; lp never directly DMAs from user memory.
- Per-minor `lp_mutex` serializes open/release; write path is naturally serialized by `parport_claim` which is exclusive while held.
- `lp_check_status` interprets the printer-status byte by bitmask; spurious bit combinations default to "error" rather than "ready".
- Per-printer timeout is per-fd at the API level but stored per-minor; the last setter wins, mitigated by mode-and-timeout being treated as a sticky printer-tuning concern.
- `lp_ioctl` per-cmd capability gating: stats read (`LPGETSTATS`) requires CAP_SYS_ADMIN; IRQ rebind (`LPSETIRQ`) requires CAP_SYS_RAWIO.
- `lp_preempt` defends the integrity of an active transfer by refusing parport preemption mid-write.
- `parport_negotiate` returns and is checked for every transition; mode is never assumed.
- Cmdline `lp=` validated; unknown tokens default to `LP_PARPORT_UNSPEC` and the minor stays unattached.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist the per-printer bounce-buf slab; bound every `copy_to_user`/`copy_from_user` against `LP_BUFFER_SIZE`. Ioctl payload sizes (`s32[2]`, `s64[2]`, `unsigned int`) explicitly enumerated.
- **PAX_KERNEXEC** — `lp_fops`, `parport_driver` ops, and the per-printer `pardevice` callback table live in `__ro_after_init` text; no runtime patching.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `lp_open`, `lp_write`, `lp_read`, `lp_ioctl`, and `lp_console_write` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-`pardevice` references; overflow trap defeats UAF on race between fd-close and parport-attach/detach.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `bounce_buf` slabs so prior print payloads (potentially containing sensitive document data) cannot bleed across opens.
- **PAX_UDEREF** — SMAP/PAN enforced on every `copy_to/from_user` and ioctl payload copy; printer-status / stats / flags are never written through bare user pointers.
- **PAX_RAP / kCFI** — `lp_fops`, `parport_driver`, `pardev_cb` (with `preempt`/`wakeup`/`irq_func`), and `console` ops are typed indirect calls; misrouted parport client traps.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `lp_struct`/`pardevice`/`parport` pointers in `/proc/sys/dev/parport*` and tracepoints; gate behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict parport-probe, IEEE-1284-negotiate-fail, and IRQ-rebind banners to CAP_SYSLOG so attackers cannot fingerprint attached parport hardware via dmesg.
- **/dev/lp* parport claim CAP_SYS_RAWIO** — opening a printer is presently CAP_DAC_OVERRIDE-only; under Rookery policy, the chardev defaults to `0600` with group=`lp`, and any cmdline-driven dynamic re-binding of minor → parport is gated by CAP_SYS_RAWIO so attackers cannot remap `lp0` onto an unintended port.
- **IEEE 1284 mode validate** — every `parport_negotiate` return is checked; the driver refuses to write in a mode the printer did not acknowledge. Unknown / reserved mode values rejected.
- **IRQ probe CAP_SYS_RAWIO** — `LPSETIRQ` and any future IRQ-line manipulation require CAP_SYS_RAWIO and are refused under `security_locked_down(LOCKDOWN_DEV_MEM)` to prevent re-routing platform interrupts via /dev/lp ioctl.
- **Bounce-buf single-writer** — per-minor bounce buffer is owned by the in-progress `lp_write`; concurrent ioctl paths are blocked by `lp_mutex`. Defense against TOCTOU on the bounce contents.
- **Stats read CAP_SYS_ADMIN** — `LPGETSTATS` returns per-printer counters that can leak workload patterns; gated by CAP_SYS_ADMIN as already done at lp.c line 651.
- **Console mode lockdown** — `CONFIG_LP_CONSOLE` console binding refused when `security_locked_down(LOCKDOWN_DEBUGFS)` (debug-time console exfiltration), and refused when integrity lockdown is active.

Rationale: parallel-port printers are vestigial but parport hardware still exists on industrial gear; `/dev/lp*` is small but it is one of the few user-facing chrdevs that can directly drive a chipset IRQ line and physical signaling. CAP_SYS_RAWIO gating on IRQ rebind, CAP_SYS_ADMIN on stats, mode-acknowledge validation on every transition, refcount saturation across parport claim/release, sanitized bounce buffers, and lockdown gating on the console path turn lp from "default-open low-stakes legacy device" into a bounded, audited chardev acceptable on hardened deployments.

## Open Questions

- Should the chardev's default mode be tightened from the historical `0660 root:lp` to `0600 root:root` on hardened deployments? Operator decision; udev rule under Rookery policy.
- Whether to remove the `lp=console` early-debug pathway entirely on production builds (it's `CONFIG_LP_CONSOLE` already, but operators sometimes ship debug kernels).

## Out of Scope

- `drivers/parport/` core (covered in `drivers/parport/00-overview.md` future Tier-3)
- `ppdev` userspace parport (covered in `ppdev.md` sibling Tier-3)
- IEEE 1284 protocol details (referenced via parport core)
- Non-Linux printing stacks (CUPS sits above this chardev)
- Implementation code
