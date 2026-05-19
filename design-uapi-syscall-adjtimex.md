---
title: "Tier-5 syscall: adjtimex(2) — syscall 159"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`adjtimex(2)` is the privileged NTP / PLL discipline interface: it allows a privileged process (typically `ntpd`, `chronyd`, or `phc2sys`) to read and adjust the kernel's NTP-state machine — frequency offset, time offset, leap-second indicator, pulse-per-second (PPS) statistics, and clock status. The single `struct timex *buf` parameter is bidirectional: input flags in `buf->modes` select which fields are being written, and on return the kernel populates the **current** state across all fields plus returns the **clock state** (`TIME_OK`, `TIME_INS`, `TIME_DEL`, `TIME_OOP`, `TIME_WAIT`, `TIME_ERROR`) as the syscall return value. Tunable parameters include slew rate (`tick`), PLL frequency (`freq`), maxerror/esterror, leap-second arming, single-step time offset (`ADJ_SETOFFSET`), and TAI offset (`ADJ_TAI`). Critical for: NTP discipline (continuous adjustment of wall clock without stepping), leap-second insertion/deletion, telecom-grade PTP grandmaster locking, real-time clock recovery after large skew.

This Tier-5 covers the userspace ABI of syscall 159. Per-`ntp_clear`, PLL loop, leap-second state machine, and `do_settimeofday64` integration are owned by `kernel/time/ntp.md` and `kernel/time/timekeeping.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: Unprivileged `modes != 0` ⟹ `-EPERM`.
- [ ] AC-2: Unprivileged `modes == 0` ⟹ success; reads current state.
- [ ] AC-3: `ADJ_FREQUENCY` with `freq = 500_000` (scaled ppm): subsequent clock_gettime drift reflects new freq.
- [ ] AC-4: `ADJ_STATUS | STA_INS`: subsequent timex read shows STA_INS set.
- [ ] AC-5: `ADJ_SETOFFSET` with 100ms: `clock_gettime(REALTIME)` jumps by 100ms; cancel-on-set timerfds fire.
- [ ] AC-6: `ADJ_OFFSET` with > MAXPHASE ⟹ `-EINVAL`.
- [ ] AC-7: `modes` with unknown bit ⟹ `-EINVAL`.
- [ ] AC-8: After leap insertion: return value transitions through `TIME_INS` → `TIME_OOP` → `TIME_WAIT` → `TIME_OK`.
- [ ] AC-9: `ADJ_TAI` updates TAI offset; readback matches.
- [ ] AC-10: `ADJ_NANO` flips reporting; subsequent reads expose `tv_nsec`-granularity fields.
- [ ] AC-11: Non-init-userns root (no init-userns CAP_SYS_TIME) ⟹ `-EPERM`.

### Architecture

Rookery surface in `kernel/time/ntp.rs`:

```rust
pub fn sys_adjtimex(buf: *mut Timex) -> isize {
    let mut tx = match copy_from_user::<Timex>(buf) {
        Ok(v) => v, Err(_) => return -EFAULT,
    };
    if tx.modes != 0 && !capable(CAP_SYS_TIME) { return -EPERM; }
    let rc = Ntp::do_adjtimex(&mut tx);
    if rc < 0 { return rc; }
    if copy_to_user(buf, &tx).is_err() { return -EFAULT; }
    rc        // clock state TIME_*
}
```

`Ntp::do_adjtimex(tx) -> isize`:
1. /* validate modes */
2. if tx.modes & !ADJ_MODE_MASK != 0 { return -EINVAL; }
3. /* read-only fast path */
4. if tx.modes == 0 {
   - Ntp::populate_readout(tx);
   - return time_state();
 }
5. /* lock + write */
6. let flags = raw_spin_lock_irqsave(&tk_core.lock);
7. write_seqcount_begin(&tk_core.seq);
8. /* validate field ranges */
9. if !Ntp::field_ranges_ok(tx) {
   - write_seqcount_end(&tk_core.seq);
   - raw_spin_unlock_irqrestore(&tk_core.lock, flags);
   - return -EINVAL;
 }
10. /* SETOFFSET — step time */
11. if tx.modes & ADJ_SETOFFSET != 0 {
    - timekeeping_inject_offset(&mut tk_core.tk, ts_from_timex(&tx));
    - update_vsyscall(&tk_core.tk);
  }
12. /* PLL / FLL discipline */
13. ntp_adjtimex(&mut tk_core.tk.ntp_state, tx);
14. write_seqcount_end(&tk_core.seq);
15. raw_spin_unlock_irqrestore(&tk_core.lock, flags);
16. /* wake cancel-on-set timerfds if SETOFFSET */
17. if tx.modes & ADJ_SETOFFSET != 0 { clock_was_set(); }
18. Ntp::populate_readout(tx);
19. audit_log_adjtimex(current.uid, tx);
20. time_state()

### Out of Scope

- `clock_settime.md` / `clock_adjtime.md` / `settimeofday.md` siblings.
- `kernel/time/ntp.md` Tier-3: PLL/FLL loop, leap-second state machine.
- `kernel/time/timekeeping.md` Tier-3: tk_core write protocol.
- glibc / musl `adjtimex` wrappers (`ntp_adjtime` POSIX surface).
- `chronyd` / `ntpd` userspace algorithms.
- Implementation code.

### signature

```c
int adjtimex(struct timex *buf);
```

Rust ABI shim:

```rust
pub fn sys_adjtimex(buf: *mut Timex) -> isize;
```

Syscall number: **159**.
On 32-bit time-safe variant: `sys_clock_adjtime64` covers the related `clock_adjtime` family.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `buf` | `struct timex *` | IN/OUT | NTP discipline state |

### return

- **Success**: clock state code (`TIME_OK`, `TIME_INS`, `TIME_DEL`, `TIME_OOP`, `TIME_WAIT`, `TIME_ERROR`); `0` means `TIME_OK`.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EPERM` | caller lacks `CAP_SYS_TIME` and is requesting writes (`modes != 0`) |
| `EINVAL` | invalid bits in `modes`; out-of-range field values (`freq`, `tick`, etc.); `ADJ_SETOFFSET` with bad nsec |
| `EFAULT` | `buf` not in readable/writable userspace |
| `EACCES` | per-userns: not init userns and not CAP_SYS_TIME-in-init-ns |

### abi surface

`struct timex` (UAPI, 200+ bytes):

```c
struct timex {
    unsigned int  modes;       /* ADJ_* bitmask */
    long          offset;      /* time offset (usec or nsec) */
    long          freq;        /* frequency offset (scaled ppm) */
    long          maxerror;
    long          esterror;
    int           status;      /* STA_* */
    long          constant;    /* PLL time constant */
    long          precision;
    long          tolerance;
    struct timeval time;        /* current time */
    long          tick;         /* usec between ticks */
    long          ppsfreq;      /* PPS frequency, scaled */
    long          jitter;
    int           shift;
    long          stabil;
    long          jitcnt;
    long          calcnt;
    long          errcnt;
    long          stbcnt;
    int           tai;          /* TAI-UTC offset on return */
    int           pad[11];
};
```

Important `modes` bits:

| Bit | Field written | Semantics |
|---|---|---|
| `ADJ_OFFSET` | `offset` | clock offset (nsec if STA_NANO, else usec) |
| `ADJ_FREQUENCY` | `freq` | PLL frequency offset |
| `ADJ_MAXERROR` | `maxerror` | maximum error estimate |
| `ADJ_ESTERROR` | `esterror` | estimated error |
| `ADJ_STATUS` | `status` | NTP status bits (STA_PLL, STA_PPSFREQ, STA_PPSTIME, STA_INS, STA_DEL, etc.) |
| `ADJ_TIMECONST` | `constant` | PLL time constant |
| `ADJ_TICK` | `tick` | µs per system tick |
| `ADJ_TAI` | `constant` (reused) | set TAI−UTC offset |
| `ADJ_SETOFFSET` | `time` | step current time by `time` (ts; signed) |
| `ADJ_NANO` | (no field) | switch reporting to nanoseconds |
| `ADJ_MICRO` | (no field) | switch reporting to microseconds |
| `ADJ_OFFSET_SS_READ` (special) | — | read-only single-shot offset |

Return-value clock states:

| Code | Meaning |
|---|---|
| `TIME_OK` (0) | clock synchronized |
| `TIME_INS` (1) | leap second insertion pending |
| `TIME_DEL` (2) | leap second deletion pending |
| `TIME_OOP` (3) | leap second in progress |
| `TIME_WAIT` (4) | leap second occurred; awaiting clear |
| `TIME_ERROR` (5) | clock not synchronized |

### compatibility contract

REQ-1: Capability gate:
- If `modes != 0`: `CAP_SYS_TIME` required in init userns; else `-EPERM`.
- Read-only mode (`modes == 0`): no capability required.

REQ-2: `copy_from_user`:
- `buf` read into kernel-local `struct timex`; failure: `-EFAULT`.

REQ-3: `modes` validation:
- Unknown bits ⟹ `-EINVAL`.

REQ-4: Field validation:
- `tick`: must be within `[MIN_TICK, MAX_TICK]` (system-tick HZ-dependent).
- `freq`: within `[-MAXFREQ, MAXFREQ]` (scaled ppm).
- `offset`: if `ADJ_NANO` reporting, `|offset| < MAXPHASE * NSEC_PER_USEC`; else `< MAXPHASE`.

REQ-5: `ADJ_SETOFFSET`:
- `time.tv_usec` (or `tv_nsec` if STA_NANO) MUST be in `[0, 999_999_999]` or `[0, 999_999]`.
- Calls `timekeeping_inject_offset(ts)` under `tk_core.lock`.
- Updates VDSO data; calls `clock_was_set()` to wake CANCEL_ON_SET timerfds.

REQ-6: NTP discipline update:
- `ntp_update_frequency()` applied if `ADJ_FREQUENCY` set.
- PLL loop updated for `ADJ_OFFSET`/`ADJ_TIMECONST`.

REQ-7: Status bits:
- `STA_PLL`, `STA_PPSFREQ`, `STA_PPSTIME`, `STA_FLL`, `STA_INS`, `STA_DEL`, `STA_UNSYNC`, `STA_FREQHOLD`, `STA_PPSJITTER`, `STA_PPSWANDER`, `STA_PPSERROR`, `STA_CLOCKERR`, `STA_NANO`, `STA_MODE`, `STA_CLK` — bitmask.
- Read-only bits (e.g., `STA_PPSSIGNAL`) cannot be set.

REQ-8: Leap-second arming:
- `STA_INS` set ⟹ leap insertion scheduled at next end-of-day UTC.
- `STA_DEL` set ⟹ leap deletion scheduled.
- Cleared automatically after leap event.

REQ-9: TAI offset:
- `ADJ_TAI` writes the TAI−UTC offset; readable via `buf.tai`.

REQ-10: Output population:
- Kernel populates ALL fields with current state (regardless of `modes`).
- `copy_to_user(buf, &k_timex)` after kernel update.

REQ-11: Return value:
- Clock state (`time_state`) returned; not always 0 on success.
- `-1` on error.

REQ-12: Audit:
- `modes != 0` calls audit-logged (uid + modes + key field deltas).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_time_required_for_writes` | INVARIANT | modes != 0 ∧ ¬CAP_SYS_TIME ⟹ -EPERM |
| `udref_on_buf` | INVARIANT | both copies via access_ok |
| `modes_whitelisted` | INVARIANT | unknown modes bit ⟹ -EINVAL |
| `field_ranges_validated` | INVARIANT | tick, freq, offset, time bounds |
| `tk_core_lock_during_write` | INVARIANT | adjtimex write under tk_core.lock + seqcount |

### Layer 2: TLA+

`uapi/adjtimex.tla`:
- Variables: `ntp_state`, `tai_offset`, `pll_freq`, `time_state`.
- Properties:
  - `safety_cap_gated_writes` — only privileged callers mutate state.
  - `safety_leap_state_machine` — INS/DEL/OOP/WAIT transitions monotonic.
  - `safety_setoffset_invokes_clock_was_set` — wall step ⟹ cancel-on-set wake.
  - `liveness_leap_event_completes` — TIME_INS eventually → TIME_OK.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_adjtimex` post(0): ntp_state reflects requested modes | `ntp_adjtimex` |
| `do_adjtimex` post: tx populated with current readout | `Ntp::populate_readout` |
| `SETOFFSET` post: timekeeper xtime stepped; vDSO updated | `timekeeping_inject_offset` |
| audit record emitted on write | `audit_log_adjtimex` |

### Layer 4: Verus/Creusot functional

Per-`adjtimex(2)` man page, `Documentation/core-api/timekeeping.rst`, `chronyd` source (sys_linux.c), `ntpd` `ntp_loopfilter.c`, LTP `testcases/kernel/syscalls/adjtimex/*`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

adjtimex reinforcement:

- **CAP_SYS_TIME strict for writes** — read-only mode unprivileged; any write requires capability in init userns.
- **Field-range validation before commit** — bad freq/tick/offset rejected without touching kernel state.
- **tk_core.lock + seqcount writer for all updates** — userspace vDSO readers never observe torn state.
- **Cancel-on-set wake on SETOFFSET** — wall step wakes all CANCEL_ON_SET timerfds.
- **Audit log** — every write logged.

### grsecurity/pax-style reinforcement

- **CAP_SYS_TIME init-userns gating** — under hardened policy, CAP_SYS_TIME is honored only in the **init** user namespace; container roots with CAP_SYS_TIME on a non-init userns are denied with `-EPERM`. Prevents container escape vectors that obtained CAP_SYS_TIME in a userns from disciplining the host clock or arming leap seconds.
- **PaX UDEREF on `buf` read and write** — both transfers refuse kernel-mapped addresses; defense against type-confused mutation of the timekeeper through a forged `timex` pointer.
- **GRKERNSEC_CLOCK_RESOLUTION write-side rate limit** — per-uid token bucket on `adjtimex` writes (shared with `clock_settime` and `clock_adjtime`); default: 1 write per 1000ms under hardened profile.
- **PAX_RANDKSTACK on every entry** — random kstack on each call; relevant because adjtimex paths cross NTP, PLL, leap-state, and timekeeping subsystems — all attractive ROP targets.
- **GRKERNSEC_HIDESYM on `tk_core` and `ntp_state`** — both globals hidden from kallsyms under `kptr_restrict ≥ 2`.
- **Refuse `STA_FREQHOLD` toggling under fast cadence** — hardened policy logs and rate-limits status-bit flapping to prevent NTP-loop-poisoning attacks where an attacker rapidly toggles `STA_FREQHOLD` to lock the kernel into a bad frequency offset.
- **Block `freq` deltas > 500 ppm without explicit confirmation** — large frequency changes are atypical; hardened policy rejects |freq − freq_old| > 500 ppm with `-EPERM` plus audit; legitimate operators iterate via small ADJ_FREQUENCY steps.
- **Block `ADJ_SETOFFSET` > 8s** — large single-step time changes are atypical (leap-second handling does ±1s only); hardened policy rejects |delta| > 8s on SETOFFSET with `-EPERM` plus audit.
- **GRKERNSEC_AUDIT_GRPADD-style audit** — successful adjtimex logs `(syscall, uid, gid, pid, modes, freq, offset, tick, status, tai)`; defenders reconstruct any clock-poisoning attack post-incident.
- **`clock_was_set` notification rate-limit** — SETOFFSET-triggered notifications throttled per-uid to prevent storms destabilizing timerfd consumers.
- **Refuse `ADJ_TAI` deltas > 100s** — TAI offset has bounded real-world variance (~37s currently); hardened policy rejects large jumps as suspicious.
- **Leap-second arming logged distinctly** — STA_INS/STA_DEL arming produces a high-severity audit record so defenders are aware of any pending leap event.
- **`ntp_state` integrity canary** — hardened build embeds a per-tk_core canary; mid-call corruption (e.g., via SMM injection or kernel-write primitive) is detected and the call returns `-EFAULT` instead of committing.
- **adjtimex co-rate-limit with clock_settime / clock_adjtime** — shared per-uid quota across all three privileged time-mutating syscalls.

