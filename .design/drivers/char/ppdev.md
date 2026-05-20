# Tier-3: drivers/char/ppdev.c — userspace parallel-port chardev /dev/parportN (PP_* ioctls + IRQ-event polling)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/char/ppdev.c
  - include/uapi/linux/ppdev.h
-->

## Summary

`drivers/char/ppdev.c` (~880 lines) exports `/dev/parport0..N` — the userspace interface to raw parallel-port hardware via `drivers/parport/`. Where `drivers/char/lp.c` is a kernel-driven printer device, ppdev is the opposite: a low-level passthrough that lets userland claim a parport, read/write data/status/control lines, switch between IEEE 1284 modes (compat / nibble / byte / ECP / EPP), wait for interrupts on the ACK line, and own the port across operations. The driver registers a character device (typically major `PP_MAJOR=99`) with a per-minor `pp_struct`, allocates a per-open IDA index for `parport_register_dev_model`, and dispatches a rich PP_* ioctl vocabulary (`PPCLAIM`, `PPRELEASE`, `PPEXCL`, `PPSETMODE`, `PPNEGOT`, `PPSETPHASE`, `PPRSTATUS`, `PPRCONTROL`/`PPWCONTROL`/`PPFCONTROL`, `PPRDATA`/`PPWDATA`, `PPDATADIR`, `PPYIELD`, `PPCLRIRQ`, `PPWCTLONIRQ`, `PPSETTIME`/`PPGETTIME` 32-/64-bit variants, `PPGETMODES`, `PPGETMODE`, `PPGETPHASE`, `PPGETFLAGS`/`PPSETFLAGS` with `PP_FASTREAD`/`PP_FASTWRITE`/`PP_W91284PIC`). It is the foundation for userspace drivers of GPIB adapters, EEPROM programmers, JTAG cables, embedded boards, and historic parport scanners.

This Tier-3 captures the chardev surface, the per-fd claim semantics, the IRQ-event polling state machine, and the IEEE 1284 mode/phase model.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pp_struct` (per-fd state) | claimed flag, current mode/phase, irq accounting, pardevice handle | `drivers::ppdev::PpFd` |
| `pp_fops` (open/release/read/write/ioctl/compat_ioctl/poll) | chardev file_operations | `PpFd::FileOps` |
| `pp_open(inode, file)` | per-fd alloc, IDA index | `PpFd::open` |
| `pp_release(inode, file)` | release port if held, free index | `PpFd::release` |
| `pp_read(file, buf, count, ppos)` / `pp_write` | data-line burst transfer in current mode | `PpFd::read` / `write` |
| `pp_ioctl(file, cmd, arg)` / `pp_do_ioctl` | PP_* ioctl dispatch | `PpFd::ioctl` |
| `pp_irq(private)` | per-pardevice IRQ callback; bumps `irqc`, queues a SIGIO | `PpFd::irq` |
| `pp_enable_irq(pp)` | request the pardevice IRQ line | `PpFd::enable_irq` |
| `register_device(minor, pp)` | call `parport_register_dev_model` with `ppdev_cb` (preempt/wakeup/irq) | `PpFd::register_device` |
| `init_phase(mode)` | initial IEEE 1284 phase per mode | `PpFd::init_phase` |
| `pp_set_timeout(pdev, tv_sec, tv_usec)` | per-pardevice access timeout | `PpFd::set_timeout` |
| `PP_BUFFER_SIZE` (1024) | per-call bounce-buffer cap | `PpFd::BUFFER_SIZE` |
| `PP_CLAIMED` / `PP_EXCL` / `PP_W91284PIC` / `PP_FASTREAD` / `PP_FASTWRITE` (`pp->flags`) | per-fd state bits | `PpFd::Flags` |
| `PPCLAIM` / `PPRELEASE` / `PPEXCL` / `PPYIELD` | claim/release control | `Ioctl::Claim` family |
| `PPSETMODE` / `PPNEGOT` / `PPSETPHASE` / `PPGETMODE[S]` / `PPGETPHASE` | mode/phase control | `Ioctl::Mode` family |
| `PPRSTATUS` / `PPRCONTROL` / `PPWCONTROL` / `PPFCONTROL` / `PPRDATA` / `PPWDATA` / `PPDATADIR` | line manipulation | `Ioctl::Lines` family |
| `PPCLRIRQ` / `PPWCTLONIRQ` | IRQ event consume / on-IRQ control-line write | `Ioctl::Irq` family |
| `PPSETTIME32` / `PPGETTIME32` / `PPSETTIME64` / `PPGETTIME64` | timeout I/O (32/64-bit timespec) | `Ioctl::Timeout` family |
| `PPGETFLAGS` / `PPSETFLAGS` (with `PP_FLAGMASK`) | fast-IO flag bits | `Ioctl::Flags` |
| `ppdev_cb` (`preempt`/`wakeup`/`irq_func`/`flags`) | parport client callback table | `PpFd::ParportCb` |

## Compatibility contract

REQ-1: `/dev/parport0..N` per parport core; minor maps 1:1 to `parport_find_number(minor)`. Open of an unattached minor returns `-ENXIO`.

REQ-2: Each `open()` allocates a new `pp_struct` and an IDA index; the parport is not claimed yet. Caller must issue `PPCLAIM` to take ownership before any data/control/status I/O.

REQ-3: `PPCLAIM` blocks (or returns `-EAGAIN` on `O_NONBLOCK`) until `parport_claim_or_block(pp->pdev)` succeeds; success sets `pp->flags |= PP_CLAIMED` and enables the `pp_irq` IRQ callback.

REQ-4: `PPRELEASE` calls `parport_release(pp->pdev)` and clears `PP_CLAIMED`. Closing the fd implicitly releases.

REQ-5: `PPEXCL` (before `PPCLAIM`) sets `PARPORT_FLAG_EXCL`; the subsequent `PPCLAIM` is exclusive and refuses to let any other parport client preempt.

REQ-6: `PPSETMODE` (`int` arg) validates against `IEEE1284_MODE_*` and stores `pp->state.mode`; `PPNEGOT` runs `parport_negotiate(port, mode)` and updates `pp->state.phase` via `init_phase(mode)`.

REQ-7: `PPRSTATUS` / `PPRCONTROL` / `PPRDATA` read the corresponding parport register (1 byte); `PPWCONTROL` / `PPWDATA` write 1 byte; `PPFCONTROL` (`struct ppdev_frob_struct`) performs a masked control-line update.

REQ-8: `PPDATADIR` (`int` arg, 0=output, 1=input) calls `parport_data_forward` / `_reverse` to flip the data-line driver; only meaningful in compat/byte modes.

REQ-9: `pp_read` / `pp_write` use `PP_BUFFER_SIZE`-chunked bounce; the transfer function is mode-dependent (`parport_write` for fwd modes; `parport_read` for reverse modes). `PP_FASTREAD`/`PP_FASTWRITE` enable `PARPORT_EPP_FAST` for EPP mode.

REQ-10: `PPCLRIRQ` reads and clears `pp->irqc` (the count of IRQ events delivered since last clear); returns the consumed count via `int __user *arg`.

REQ-11: `PPWCTLONIRQ` arms a "write this control-line byte on next IRQ" — the per-pardevice IRQ callback writes the supplied byte to the control register from IRQ context, supporting tight latency-bound USB-trigger-style scenarios.

REQ-12: `PPSETTIME`/`PPGETTIME` 32/64-bit variants set/get the per-pardevice access timeout (`pardevice->timeout`); 32-bit variants pack two `s32` words, 64-bit variants two `s64` words. Compat ioctl wired via `pp_ioctl`'s explicit switching on the magic.

REQ-13: `PPYIELD` calls `parport_yield_blocking(pp->pdev)` to let other parport clients run briefly; returns when the port is re-claimed.

REQ-14: `poll()` reports `POLLIN | POLLRDNORM` whenever `pp->irqc > 0` (an IRQ has been observed); `SIGIO` is delivered when ip's `fasync_queue` is populated and a new IRQ arrives.

## Acceptance Criteria

- [ ] AC-1: With `parport_pc.ko` + `ppdev.ko` loaded on a board with a real parport, `/dev/parport0` exists with major `PP_MAJOR`.
- [ ] AC-2: `open("/dev/parport0", O_RDWR) + ioctl(PPCLAIM)` succeeds as root; without the parport being free, blocks (or returns `-EAGAIN` with O_NONBLOCK).
- [ ] AC-3: `ioctl(PPSETMODE, IEEE1284_MODE_COMPAT)` then `write(fd, "hi", 2)` sends 2 bytes to the data lines.
- [ ] AC-4: `ioctl(PPRSTATUS, &s)` returns the current status byte (BUSY/ACK/SELECT/PE/ERROR bits).
- [ ] AC-5: An IRQ fires on the parport ACK line → `poll(POLLIN)` returns, `PPCLRIRQ` returns count >= 1.
- [ ] AC-6: `PPEXCL` followed by `PPCLAIM` prevents `lp.ko` from auto-attaching to the same parport while ppdev holds it.
- [ ] AC-7: 32-bit-on-64-bit `PPSETTIME32` through `pp_compat_ioctl` matches 64-bit caller's `PPSETTIME64` result.
- [ ] AC-8: Closing the fd while claimed releases the parport (verified by another client successfully claiming).
- [ ] AC-9: Read/write while `(PP_CLAIMED == 0)` returns `-EINVAL` ("claim the port first").

## Architecture

`PpFd` lives in `drivers::ppdev::PpFd`:

```
struct PpFd {
  pdev: Option<Arc<ParDevice>>,    // None until PPCLAIM
  port: Option<Arc<ParPort>>,
  index: i32,                      // IDA index for unique pardevice name
  flags: AtomicU32,                // PP_CLAIMED | PP_EXCL | PP_FASTREAD | PP_FASTWRITE | PP_W91284PIC
  state: PpState {
    mode: IeeeMode,                // current IEEE 1284 mode
    phase: IeeePhase,
  },
  saved_state: PpState,            // before parport_yield
  irqc: AtomicU32,                 // unread IRQ events
  irq_wait: WaitQueue,
  fasync: Option<FasyncQueue>,
  default_inactivity: long,
  pending_byte: Option<u8>,        // PPWCTLONIRQ target byte
  pending_mask: Option<u8>,
  timeout: Timeout,                // per-fd timespec
}

struct ParportCb {
  preempt: pp_preempt,
  wakeup:  pp_wakeup,
  irq_func: pp_irq,
  flags:   ParportFlags,           // PARPORT_FLAG_EXCL when PP_EXCL set
}
```

Open path `pp_open(inode, file)`:
1. Allocate `PpFd` + IDA index for naming.
2. Snapshot `minor = iminor(inode)`.
3. Defer `parport_register_dev_model` until `PPCLAIM` so that opens of unattached parports do not pin the parport device.
4. Return.

Claim path (`PPCLAIM`):
1. If `pp->pdev` is None: `register_device(minor, pp)`:
   - `port = parport_find_number(minor)`; refuse if NULL.
   - Build `ppdev_cb` with `flags = (pp->flags & PP_EXCL) ? PARPORT_FLAG_EXCL : 0`.
   - `pdev = parport_register_dev_model(port, "ppN", &ppdev_cb, index)`.
2. `parport_claim_or_block(pp->pdev)` → blocks or returns `-EAGAIN`.
3. Set `pp->flags |= PP_CLAIMED`; `pp_enable_irq(pp)`.
4. Set initial mode (`IEEE1284_MODE_COMPAT`) and phase.

Per-line ioctl pattern (e.g. `PPRDATA`):
1. Require `(pp->flags & PP_CLAIMED)`; else return `-EINVAL`.
2. `val = parport_read_data(pp->pdev->port)`.
3. `put_user(val, (unsigned char __user *)arg)`.

IRQ event path:
1. Parport core calls `pp_irq(private = pp)` from per-port IRQ handler.
2. Increment `pp->irqc` atomically.
3. If `pending_byte`+`pending_mask` set (`PPWCTLONIRQ`): `parport_frob_control(port, mask, val)`; clear pending.
4. Wake `irq_wait`; emit SIGIO via `kill_fasync(fasync, SIGIO, POLL_IN)`.

Mode + phase model:
- IEEE 1284 modes: COMPAT (forward, host→peripheral), NIBBLE / BYTE (reverse), EPP (bidirectional, address+data phase), ECP (negotiable bidirectional with FIFO).
- Each mode has a default initial phase set by `init_phase(mode)`; userland may override via `PPSETPHASE`.
- Mode transition: `PPNEGOT` issues the IEEE 1284 negotiation handshake; failure (no peripheral acknowledgement) returns `-EIO` and leaves mode unchanged.

Read/write path (`pp_read` / `pp_write`):
1. Refuse if `!PP_CLAIMED`.
2. Chunk into `PP_BUFFER_SIZE` (1024) per iteration; `kmalloc(min(count, 1024), GFP_KERNEL)`.
3. `copy_from_user` (for write) into `kbuffer`.
4. Per chunk: `parport_write(port, kbuffer, n)` or `parport_read(port, kbuffer, n)` per mode. EPP fast variants set `PARPORT_EPP_FAST` flag if `PP_FASTREAD`/`PP_FASTWRITE` are in `pp->flags`.
5. For W91284PIC variant (per-vendor IEEE 1284 quirk), `parport_ieee1284_interrupt` path enabled before read.
6. Accumulate bytes, advance, loop.

## Hardening

- Per-fd `PP_BUFFER_SIZE` cap (1 KiB) limits per-syscall memory footprint.
- IRQ event count `pp->irqc` is bounded only by `AtomicU32`; under sustained IRQ storm it saturates rather than overflowing — but to defeat IRQ-storm DoS, the IRQ handler is rate-limited via a per-pardevice deferred-work counter (Rookery hardening).
- Per-fd state is owned exclusively by the open file; no global mutable state shared across opens of the same parport (parport core enforces the single-active-client model via claim/release).
- `parport_register_dev_model` does not auto-bind the port; explicit `PPCLAIM` is required, defeating the "open and forget" capture of a parport that some legacy chardevs allowed.
- Mode/phase fields validated against `IEEE1284_MODE_*` and `IEEE1284_PH_*` enums at every set; out-of-range refused with `-EINVAL`.
- The `PPYIELD` / `PPWAIT` ops are voluntary; the kernel cannot be made to keep a userspace process blocked indefinitely on the port — `parport_yield_blocking` returns periodically.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist the per-call `kmalloc(min(count, PP_BUFFER_SIZE))` bounce buffer; bound every `copy_to_user`/`copy_from_user` against `PP_BUFFER_SIZE` and per-ioctl payload sizes. Ioctl argument types enumerated (`int`, `unsigned char`, `struct ppdev_frob_struct`, `s32[2]`, `s64[2]`).
- **PAX_KERNEXEC** — `pp_fops`, `ppdev_cb`, and the per-mode dispatch tables live in `__ro_after_init` text; no runtime patching of parport callbacks.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `pp_open`, `pp_read`, `pp_write`, `pp_ioctl`, and `pp_irq` (IRQ entry) frames.
- **PAX_REFCOUNT** — saturating `refcount_t` on `pardevice` references and on the IDA index allocator; overflow trap defeats UAF on race between fd-close and parport-detach.
- **PAX_MEMORY_SANITIZE** — zero-on-free for the per-call kbuffer so prior transfer payloads (potentially containing JTAG / EEPROM / GPIB sensitive data) cannot bleed across opens or across kmalloc reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every PP_* ioctl payload copy; `struct ppdev_frob_struct`, `s32[2]`, `s64[2]` reads/writes use canonical helpers.
- **PAX_RAP / kCFI** — `pp_fops`, `ppdev_cb` (preempt/wakeup/irq), and `parport_driver` indirect calls are typed; misrouted parport callback substitution traps.
- **GRKERNSEC_HIDESYM** — suppress `%p` of `pp_struct`, `pardevice`, and `parport` pointers in `/proc/sys/dev/parport*` and tracepoints; gate behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict "claim the port first" warnings, IEEE 1284 negotiation failures, and IRQ-storm banners to CAP_SYSLOG so attackers cannot fingerprint attached parport peripherals via dmesg.
- **/dev/parportN PP_CLAIM CAP_SYS_RAWIO** — `PPCLAIM` (and `PPEXCL` claim) gated by CAP_SYS_RAWIO under Rookery policy; chardev default mode tightened to `0600 root:root` (upstream udev typically `0660 root:lp`).
- **PP_* ioctl PAX_USERCOPY bounds** — every ioctl payload type registered with PAX_USERCOPY explicit length; no ioctl-driven copy_to/from_user larger than its `struct`/scalar declared size.
- **IRQ-event polling rate-limit** — `pp_irq` increments `irqc` but defers wake/SIGIO via a per-pardevice token-bucket (default ~1000 events/sec); excess events accumulate without flooding poll/wake paths. Defense against malicious peripheral generating an interrupt storm.
- **Mode validation** — `PPSETMODE`/`PPNEGOT`/`PPSETPHASE` validate against the enumerated IEEE 1284 modes/phases; reserved bit patterns refused with `-EINVAL` and a rate-limited audit record.
- **PPWCTLONIRQ control gating** — the on-IRQ control-line write feature is gated by CAP_SYS_RAWIO and restricted to CAP_SYS_RAWIO opens; defense against IRQ-context line manipulation used as a covert side channel between fds.
- **Per-fd IRQ ownership** — `pp_irq` callback is bound to the specific `pp` private; per-pardevice exclusivity guaranteed by parport core. Defense against cross-fd IRQ leak.

Rationale: ppdev is the "give userspace direct access to a chipset bus" chardev — a legitimate need for industrial / embedded scenarios but a sharp tool. The hardware can drive arbitrary external devices (EEPROM programmers, JTAG cables, debug pods) and the IRQ line is wired into the platform interrupt controller. CAP_SYS_RAWIO on PPCLAIM, lockdown-eligible mode validation, IRQ-storm rate-limiting, PAX_USERCOPY bounds on every PP_* ioctl, refcount saturation across parport teardown, and kCFI on the parport callback vtables turn /dev/parport from "raw chipset bus to root processes" into a CAP_SYS_RAWIO-gated, audited, rate-limited interface acceptable on hardened deployments while preserving its legitimate industrial use case.

## Open Questions

- Should `PPWCTLONIRQ` be available at all under integrity lockdown, or refused outright? IRQ-context control-line manipulation is intrinsically tight to the platform interrupt path.
- Whether to ship a sysctl `kernel.ppdev_irq_rate_limit` for operators to tune the IRQ-storm token bucket.

## Out of Scope

- `drivers/parport/` core (covered in `drivers/parport/00-overview.md` future Tier-3)
- `lp` printer chardev (covered in `lp.md` sibling Tier-3)
- IEEE 1284 protocol specification details
- Per-platform parport chip drivers (`parport_pc.c`, `parport_serial.c`)
- Implementation code
