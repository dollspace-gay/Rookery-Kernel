---
title: "Tier-5 syscall: clock_adjtime(2) — syscall 305"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`clock_adjtime(2)` is the `clockid`-parameterized generalization of `adjtimex(2)`: it lets a privileged process apply NTP-style discipline (frequency, phase, leap, time-step) to **any settable clock** — not just `CLOCK_REALTIME`. The principal use case is **dynamic POSIX clocks** (`CLOCKFD`-encoded handles backed by PTP-hardware-clock devices `/dev/ptp*`), where NIC drivers expose hardware-locked timers that PTP daemons (`ptp4l`, `phc2sys`) discipline against grandmaster sources. For `CLOCK_REALTIME` it behaves identically to `adjtimex(2)`. Critical for: PTP grandmaster sync (sub-microsecond cross-machine clock alignment in trading, telecom, broadcast), redundant timekeeping (multiple PHCs disciplined independently), NIC-hardware-clock-to-system-clock slew (phc2sys), real-time SLA enforcement (industrial control, financial messaging).

This Tier-5 covers the userspace ABI of syscall 305. Per-PTP-clock-driver `posix_clock_ops::clock_adjtime` is owned by `drivers/ptp/ptp_clock.md` and `kernel/time/posix_clock.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `clock_adjtime(CLOCK_REALTIME, &tx)` behaves identically to `adjtimex(&tx)`.
- [ ] AC-2: `clock_adjtime(CLOCK_MONOTONIC, &tx)` ⟹ `-EINVAL`.
- [ ] AC-3: PHC clock from `/dev/ptp0` accepts `ADJ_FREQUENCY` (driver-dependent); returns `TIME_OK`.
- [ ] AC-4: Unprivileged `modes != 0` on REALTIME ⟹ `-EPERM`.
- [ ] AC-5: PHC clock without `clock_adjtime` ops ⟹ `-EOPNOTSUPP`.
- [ ] AC-6: PHC clock fd closed concurrently ⟹ `-ENODEV` (or `-EBADF` depending on race).
- [ ] AC-7: Invalid CLOCKFD encoding ⟹ `-EINVAL`.
- [ ] AC-8: `ADJ_SETOFFSET` on REALTIME wakes CANCEL_ON_SET timerfds; on PHC does not.
- [ ] AC-9: Non-init-userns root (no init-userns CAP_SYS_TIME) ⟹ `-EPERM`.
- [ ] AC-10: PHC `ADJ_FREQUENCY` outside driver-supported range ⟹ `-ERANGE` or `-EINVAL` per driver.

### Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_clock_adjtime(clk_id: i32, buf: *mut Timex) -> isize {
    let mut tx = match copy_from_user::<Timex>(buf) {
        Ok(v) => v, Err(_) => return -EFAULT,
    };
    if tx.modes != 0 && !capable(CAP_SYS_TIME) { return -EPERM; }
    /* dynamic CLOCKFD */
    if (clk_id as u32 & CLOCKFD_MASK) == CLOCKFD {
        let fd = clock_id_to_fd(clk_id);
        let file = fdget(fd).ok_or(-EBADF)?;
        let pc = PosixClock::from_file(&file).ok_or(-EINVAL)?;
        let ops = pc.ops.as_ref();
        let rc = match ops.clock_adjtime.as_ref() {
            Some(f) => f(&pc, &mut tx),
            None    => -EOPNOTSUPP,
        };
        if rc < 0 { return rc; }
        if copy_to_user(buf, &tx).is_err() { return -EFAULT; }
        return rc;
    }
    /* static clocks */
    match ClockId::try_from(clk_id) {
        Realtime | RealtimeAlarm => {
            let rc = Ntp::do_adjtimex(&mut tx);
            if rc < 0 { return rc; }
            if copy_to_user(buf, &tx).is_err() { return -EFAULT; }
            rc
        }
        Tai => {
            let rc = Ntp::do_adjtimex_tai(&mut tx);
            if rc < 0 { return rc; }
            if copy_to_user(buf, &tx).is_err() { return -EFAULT; }
            rc
        }
        _ => -EINVAL,
    }
}
```

### Out of Scope

- `adjtimex.md` / `clock_settime.md` siblings.
- `kernel/time/posix_clock.md` Tier-3: dynamic posix clock dispatch.
- `drivers/ptp/ptp_clock.md` Tier-3: PTP hardware clock driver layer.
- `kernel/time/ntp.md` Tier-3: NTP PLL/FLL discipline.
- `ptp4l` / `phc2sys` userspace algorithms.
- Implementation code.

### signature

```c
int clock_adjtime(clockid_t clk_id, struct timex *buf);
```

Rust ABI shim:

```rust
pub fn sys_clock_adjtime(clk_id: i32, buf: *mut Timex) -> isize;
```

Syscall number: **305**.
On 32-bit time-safe variant: `sys_clock_adjtime64` (syscall 410).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `clk_id` | `clockid_t` (`i32`) | IN | target clock (`CLOCK_REALTIME`, CLOCKFD-encoded dynamic clock) |
| `buf` | `struct timex *` | IN/OUT | discipline state (same layout as adjtimex) |

### return

- **Success**: clock state code (`TIME_OK`, `TIME_INS`, `TIME_DEL`, `TIME_OOP`, `TIME_WAIT`, `TIME_ERROR`).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EPERM` | caller lacks `CAP_SYS_TIME` for writes (`modes != 0`) on the targeted clock |
| `EINVAL` | `clk_id` not adjustable; invalid `modes` bits; field range violation |
| `EFAULT` | `buf` not in user-accessible memory |
| `EOPNOTSUPP` | dynamic PHC clock backend does not implement `clock_adjtime` |
| `ENODEV` | dynamic PHC clock (`CLOCKFD`-encoded) refers to a closed/freed device |
| `EACCES` | per-userns: not init userns and not CAP_SYS_TIME-in-init-ns |

### abi surface

Settable `clk_id` values:

| Constant | Settable | Backend |
|---|---|---|
| `CLOCK_REALTIME` | yes | timekeeper NTP-state |
| `CLOCK_TAI` | yes (offset only via TAI ADJ_TAI bit) | timekeeper |
| `CLOCK_MONOTONIC` | **no** | `-EINVAL` |
| `CLOCK_BOOTTIME` | **no** | `-EINVAL` |
| `CLOCK_REALTIME_ALARM` | yes (limited) | alarmtimer backend |
| dynamic CLOCKFD (e.g., `/dev/ptp0`) | maybe | `posix_clock_ops::clock_adjtime` |

`struct timex` layout: same as `adjtimex.md`. PTP backends typically honor:

| Modes bit | Meaning for PHC |
|---|---|
| `ADJ_FREQUENCY` | adjust PHC frequency (scaled ppm) |
| `ADJ_SETOFFSET` | step PHC time |
| `ADJ_TIMECONST` | (typically ignored by PHCs) |

### compatibility decoded `clockfd`:

Linux encodes a dynamic POSIX clock id as `(((unsigned)~fd) << 3) | CLOCKFD` where `CLOCKFD == 3`. Decoded via `FD_TO_CLOCKID` macro: `clock_id_to_fd(id) = ~((unsigned)id >> 3)`.

### compatibility contract

REQ-1: Capability gate (write modes):
- `modes != 0` on `CLOCK_REALTIME`/`CLOCK_TAI`/alarm clocks: `CAP_SYS_TIME` in init userns required; else `-EPERM`.
- For dynamic PHC clocks: opening `/dev/ptp*` already required `CAP_NET_ADMIN` or device perms; `adjtime` itself also requires `CAP_SYS_TIME`.

REQ-2: `clk_id` validation:
- Recognized: `CLOCK_REALTIME`, `CLOCK_TAI`, `CLOCK_REALTIME_ALARM`, CLOCKFD-encoded dynamic clocks.
- Read-only (`CLOCK_MONOTONIC`, `_RAW`, `_COARSE`, `BOOTTIME`): `-EINVAL`.

REQ-3: `copy_from_user`:
- `buf` read into kernel-local `timex`; failure: `-EFAULT`.

REQ-4: `modes` validation: same as `adjtimex.md`.

REQ-5: Dispatch:
- For `CLOCK_REALTIME` (and TAI/REALTIME_ALARM): falls through to `do_adjtimex` (same as `sys_adjtimex`).
- For CLOCKFD-encoded: decode → `fdget(fd)` → `posix_clock` → `pc.ops.clock_adjtime(pc, tx)`.

REQ-6: PHC dispatch:
- `posix_clock_ops::clock_adjtime` must be non-NULL; else `-EOPNOTSUPP`.
- Backend may implement a subset of `modes` and return `-EOPNOTSUPP` for unsupported bits.

REQ-7: PHC field semantics:
- `ADJ_FREQUENCY`: NIC hardware register write (driver-specific).
- `ADJ_SETOFFSET`: step PHC by `time` field.
- `ADJ_OFFSET` with phase-detector mode rarely used (`ptp4l` uses its own loop filter).

REQ-8: Output:
- Kernel populates `buf` with current state (frequency, status, time, etc.).
- For PHCs, fields not relevant (e.g., `maxerror`, `esterror`) typically zero.

REQ-9: Audit:
- Successful writes audit-logged with `(uid, clk_id, modes, freq, offset)`.

REQ-10: Cancel-on-set:
- `ADJ_SETOFFSET` on `CLOCK_REALTIME`: calls `clock_was_set()` to wake CANCEL_ON_SET timerfds.
- PHC SETOFFSET does NOT wake REALTIME cancel-on-set list (different clock domain).

REQ-11: TAI integration:
- `ADJ_TAI` on `CLOCK_REALTIME` sets `tai_offset`.
- On `CLOCK_TAI`: writes interpreted as TAI-domain adjustments (offset).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_time_required_for_writes` | INVARIANT | modes != 0 ∧ ¬CAP_SYS_TIME ⟹ -EPERM |
| `clockid_validated` | INVARIANT | non-settable static clocks rejected |
| `clockfd_dispatch_correct` | INVARIANT | CLOCKFD-encoded id ⟹ PHC backend; else static dispatch |
| `udref_on_buf` | INVARIANT | both copies via access_ok |
| `eopnotsupp_on_missing_ops` | INVARIANT | PHC without clock_adjtime ⟹ -EOPNOTSUPP |

### Layer 2: TLA+

`uapi/clock_adjtime.tla`:
- Variables: `tk_ntp`, `phc_state[fd]`, `tai_offset`.
- Properties:
  - `safety_realtime_equiv_to_adjtimex` — clock_adjtime(REALTIME) ≡ adjtimex.
  - `safety_phc_isolated` — PHC SETOFFSET does not perturb REALTIME state.
  - `safety_unsettable_clocks_rejected` — MONOTONIC/BOOTTIME/_RAW ⟹ -EINVAL.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_clock_adjtime` post(0): tx populated with current state | dispatch |
| REALTIME path post: ntp_state matches adjtimex semantics | `Ntp::do_adjtimex` |
| PHC path post: backend invoked exactly once | `pc.ops.clock_adjtime` |
| audit record emitted on write | `audit_log_clock_adjtime` |

### Layer 4: Verus/Creusot functional

Per-`clock_adjtime(2)` man page, `ptp4l` (linuxptp) source, `phc2sys` source, glibc `sysdeps/unix/sysv/linux/clock_adjtime.c`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

clock_adjtime reinforcement:

- **CAP_SYS_TIME strict for writes** — init userns only.
- **PHC dispatch via posix_clock_ops** — driver-specific safety inherited; ops table validated non-NULL.
- **Field range validation before commit** — bad freq/offset rejected.
- **REALTIME SETOFFSET wakes cancel-on-set list** — wall step propagates.
- **PHC SETOFFSET isolated** — does NOT touch REALTIME timekeeper.

### grsecurity/pax-style reinforcement

- **CAP_SYS_TIME init-userns gating** — under hardened policy, CAP_SYS_TIME is honored only in the **init** user namespace for REALTIME / TAI / REALTIME_ALARM writes; container roots with CAP_SYS_TIME on a non-init userns are denied with `-EPERM`. Prevents container escape vectors disciplining the host clock.
- **PaX UDEREF on `buf` read and write** — both transfers refuse kernel-mapped addresses.
- **GRKERNSEC_CLOCK_RESOLUTION write-side rate limit** — per-uid token bucket shared with `clock_settime`, `adjtimex` (default: 1 write per 1000ms under hardened profile).
- **PAX_RANDKSTACK on every entry** — random kstack on each call; relevant because clock_adjtime crosses NTP, timekeeping, and PHC driver layers.
- **GRKERNSEC_HIDESYM on `posix_clock_ops`** — function-pointer dispatch hidden from kallsyms; defense against ROP abuse of `pc.ops.clock_adjtime` indirect call.
- **PHC ops table validated under READ_ONLY_AFTER_INIT** — driver-installed ops tables marked read-only post-registration; defense against kernel-write primitive injecting a malicious `clock_adjtime` handler.
- **Audit per dispatch** — successful writes log `(uid, clk_id, modes, freq, offset, tai)`; PHC operations log the underlying ptp index.
- **Block freq deltas > 500 ppm and SETOFFSET > 8s** — same thresholds as adjtimex; defense against catastrophic-step attacks on host clock or PTP grandmaster.
- **Refuse PHC clock_adjtime from non-owner** — only the task that opened `/dev/ptp*` (or its successors via fd inheritance) may discipline; defense against fd-table-leak abuse where a malicious task scavenges a leaked PTP fd from a privileged daemon.
- **clock_was_set notification rate-limit** — REALTIME SETOFFSET path throttled per-uid.
- **PHC-fd close race detection** — between `fdget` and ops invocation, hardened policy validates `pc.refcount > 0` and returns `-ENODEV` cleanly if the PHC was unregistered (NIC removed, driver unloaded).
- **GRKERNSEC_AUDIT_CHRDEV** — opens of `/dev/ptp*` audit-logged; combined with clock_adjtime audit, defenders can reconstruct PTP grandmaster manipulation attempts.
- **adjtimex co-rate-limit** — shared quota across clock_settime, adjtimex, clock_adjtime so an attacker cannot fan out time-mutation calls.
- **Refuse `ADJ_TAI` deltas > 100s** — same threshold as adjtimex.
- **No-cross-userns PHC access** — even if /dev/ptp* fd leaks across userns, hardened policy denies clock_adjtime unless `current.user_ns == file.f_owner.userns`.

