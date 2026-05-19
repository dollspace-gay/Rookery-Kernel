# Tier-5 syscall: sched_setattr(2) — syscall 314

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sched/core.c (SYSCALL_DEFINE3(sched_setattr))
  - kernel/sched/syscalls.c (__sched_setscheduler, sched_copy_attr)
  - kernel/sched/deadline.c (SCHED_DEADLINE admission)
  - include/uapi/linux/sched.h (struct sched_attr, SCHED_FLAG_*)
  - include/uapi/linux/sched/types.h
  - arch/x86/entry/syscalls/syscall_64.tbl (314 common sched_setattr)
-->

## Summary

`sched_setattr(2)` is the **extended** scheduling-set syscall introduced in Linux 3.14 to support SCHED_DEADLINE (EDF / Constant-Bandwidth Server) and to expose new policy flags (RESET_ON_FORK, RECLAIM, DL_OVERRUN, KEEP_POLICY, KEEP_PARAMS, UTIL_CLAMP_MIN/MAX). It supersedes `sched_setscheduler(2)` and `sched_setparam(2)` for new code: a single `struct sched_attr` carries policy, priority, nice, runtime, deadline, period, sched_flags, and util-clamp bounds. The structure is **versioned by size**: userspace MAY pass an attr smaller than the kernel's view (older binary, newer kernel) and the kernel zero-fills; if userspace passes a larger attr than the kernel knows, the kernel returns -E2BIG. Critical for: real-time scheduling, latency-sensitive control loops, EAS / util-clamp on Android-class devices, container CPU bandwidth shaping, Rookery's sandbox policy demotion path.

## Signature

```c
int sched_setattr(pid_t pid, struct sched_attr *attr, unsigned int flags);

struct sched_attr {
    __u32 size;            /* sizeof(struct sched_attr) */
    __u32 sched_policy;    /* SCHED_* */
    __u64 sched_flags;     /* SCHED_FLAG_* */

    /* SCHED_NORMAL, SCHED_BATCH */
    __s32 sched_nice;

    /* SCHED_FIFO, SCHED_RR */
    __u32 sched_priority;

    /* SCHED_DEADLINE: ns-resolution */
    __u64 sched_runtime;
    __u64 sched_deadline;
    __u64 sched_period;

    /* util-clamp (since 5.3) */
    __u32 sched_util_min;
    __u32 sched_util_max;
};
```

## Parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target task PID (caller's active pid-ns); 0 = self. |
| `attr` | `struct sched_attr *` | in (UAPI) | Policy + parameters, with size field. Must be writable to attr.size only? — no, kernel reads only; the pointer is read-only from kernel's perspective. |
| `flags` | `unsigned int` | in | Currently must be 0; reserved for future. -EINVAL on any non-zero. |

## Return value

| Value | Meaning |
|---|---|
| 0 | Success. |
| `-EINVAL` | Bad policy, bad combination, flags != 0, deadline parameters out of range, util-clamp inversion. |
| `-EPERM` | Insufficient capability for requested policy/priority. |
| `-ESRCH` | Target PID not in caller's pid-ns. |
| `-EFAULT` | attr unreadable / unaligned. |
| `-E2BIG` | attr.size > kernel's known size AND the extra bytes are non-zero. |
| `-EBUSY` | SCHED_DEADLINE bandwidth admission failure. |

## Errors

| Errno | Trigger |
|---|---|
| `EINVAL` | Unknown sched_policy; sched_flags has unknown bit; nice out of [-20,19]; priority out of [0,99] for class; runtime > deadline; deadline > period; period > 2^63; flags != 0; util-clamp min > max; SCHED_FLAG_UTIL_CLAMP_MIN/MAX > 1024. |
| `EPERM` | Non-CAP_SYS_NICE for RT/DEADLINE elevation; foreign-uid target. |
| `ESRCH` | PID not found. |
| `EFAULT` | attr unreadable. |
| `E2BIG` | attr is larger than kernel's struct sched_attr and trailing bytes are non-zero. |
| `EBUSY` | SCHED_DEADLINE admission control rejects (insufficient bandwidth on any candidate CPU). |

## ABI surface

```text
__NR_sched_setattr (x86_64) = 314
__NR_sched_setattr (i386)   = 351
__NR_sched_setattr (arm64)  = 274  (generic-syscall)
__NR_sched_setattr (generic) = 274

/* sched_flags bits */
SCHED_FLAG_RESET_ON_FORK    = 0x01
SCHED_FLAG_RECLAIM          = 0x02   /* deadline GRUB bandwidth reclaim */
SCHED_FLAG_DL_OVERRUN       = 0x04   /* deliver SIGXCPU on overrun */
SCHED_FLAG_KEEP_POLICY      = 0x08
SCHED_FLAG_KEEP_PARAMS      = 0x10
SCHED_FLAG_UTIL_CLAMP_MIN   = 0x20
SCHED_FLAG_UTIL_CLAMP_MAX   = 0x40

SCHED_FLAG_KEEP_ALL         = (KEEP_POLICY | KEEP_PARAMS)
SCHED_FLAG_UTIL_CLAMP       = (UTIL_CLAMP_MIN | UTIL_CLAMP_MAX)
SCHED_FLAG_ALL              = (RESET_ON_FORK | RECLAIM | DL_OVERRUN | KEEP_ALL | UTIL_CLAMP)

/* Util-clamp range */
SCHED_CAPACITY_SCALE        = 1024   /* sched_util_{min,max} ∈ [0, 1024] */

/* Deadline constraints */
DL_SCALE                    = 10     /* used internally for bandwidth */
runtime ≤ deadline ≤ period
```

## Compatibility contract

REQ-1: Syscall number is **314** on x86_64; **274** on arm64/generic.

REQ-2: pid == 0 ⟹ current.

REQ-3: flags MUST be 0; any other value ⟹ -EINVAL.

REQ-4: attr.size MUST be at least the offsetof of the first field the requested sched_policy needs. If attr.size < sizeof(struct sched_attr) on the kernel side, missing trailing fields are treated as zero. If attr.size > kernel's sizeof, the kernel reads the userspace tail and rejects with -E2BIG if any tail byte is non-zero (this is the forward-compatibility contract).

REQ-5: SCHED_DEADLINE requires (runtime, deadline, period) all > 0, runtime ≤ deadline ≤ period, all ≤ 2^63 ns, and admission control must accept the new bandwidth on the task's allowed CPUs.

REQ-6: SCHED_DEADLINE requires CAP_SYS_NICE.

REQ-7: SCHED_FIFO/RR with non-zero priority requires CAP_SYS_NICE OR (own task ∧ priority ≤ RLIMIT_RTPRIO).

REQ-8: SCHED_FLAG_KEEP_POLICY: ignore attr.sched_policy; keep current. SCHED_FLAG_KEEP_PARAMS: ignore attr.sched_priority, sched_nice, deadline triple. Combinations allow partial updates (e.g., flip util-clamp only).

REQ-9: SCHED_FLAG_RECLAIM and SCHED_FLAG_DL_OVERRUN are valid only for SCHED_DEADLINE; setting them on a non-deadline task ⟹ -EINVAL.

REQ-10: SCHED_FLAG_UTIL_CLAMP_MIN/MAX bits must be set if sched_util_min/sched_util_max are to be applied; values are in [0, 1024]; min > max ⟹ -EINVAL. Setting util-clamp requires CAP_SYS_NICE if raising above the per-task limit; lowering is free.

REQ-11: After success, `sched_getattr(pid, attr, attr.size, 0)` returns the active policy + params.

REQ-12: Permission gating identical to sched_setscheduler for RT classes; CAP_SYS_NICE alone authorizes SCHED_DEADLINE.

REQ-13: LSM `security_task_setscheduler(task)` hook fires.

REQ-14: attr is copied in via `sched_copy_attr` which performs the size-versioning and bounds checks before any state mutation.

REQ-15: SCHED_DEADLINE bandwidth is tracked per root domain; admission failure ⟹ -EBUSY (NOT -EINVAL).

REQ-16: util-clamp values affect the schedutil/EAS governor only; on non-EAS kernels they are stored but ignored.

## Acceptance Criteria

- [ ] AC-1: `sched_setattr(0, {policy=DEADLINE, runtime=1ms, deadline=10ms, period=10ms}, 0)` as CAP_SYS_NICE → 0.
- [ ] AC-2: Same call without CAP_SYS_NICE → -EPERM.
- [ ] AC-3: `sched_setattr(0, {policy=DEADLINE, runtime=20ms, deadline=10ms, period=10ms}, 0)` → -EINVAL (runtime > deadline).
- [ ] AC-4: `sched_setattr(0, attr, 1)` (flags != 0) → -EINVAL.
- [ ] AC-5: Oversized attr (size = sizeof(known)+8) with non-zero tail → -E2BIG; with zero tail → 0.
- [ ] AC-6: SCHED_FLAG_KEEP_POLICY with new sched_priority on existing RT task: priority updated, policy unchanged.
- [ ] AC-7: SCHED_DEADLINE on a CPU-saturated root domain → -EBUSY.
- [ ] AC-8: util-clamp: SCHED_FLAG_UTIL_CLAMP_MIN with min=300, max=900 → 0; min=1025 → -EINVAL.
- [ ] AC-9: SCHED_FLAG_RECLAIM on SCHED_NORMAL task → -EINVAL.
- [ ] AC-10: SCHED_FLAG_DL_OVERRUN: a deadline-overrunning task receives SIGXCPU.
- [ ] AC-11: Cross-user target without CAP_SYS_NICE → -EPERM.
- [ ] AC-12: LSM denial → -EACCES (manpage: -EPERM).

## Architecture

```rust
#[syscall(nr = 314, abi = "sysv")]
pub fn sys_sched_setattr(pid: pid_t,
                         uattr: UserPtr<SchedAttr>,
                         flags: u32) -> SysResult<i32> {
    SchedSetattr::do_call(pid, uattr, flags)
}
```

`SchedSetattr::do_call(pid, uattr, flags) -> i32`:
1. if flags != 0 { return -EINVAL; }
2. let attr = Sched::copy_attr(uattr)?;        // -EFAULT, -E2BIG handled here
3. SchedSetattr::validate(&attr)?;             // -EINVAL on shape errors
4. let task = if pid == 0 { current() } else { find_task_by_vpid(pid).ok_or(-ESRCH)? };
5. Sched::check_setscheduler_perm(task, &attr, /*user=*/true)?;  // -EPERM
6. security_task_setscheduler(task)?;          // -EACCES
7. Sched::__sched_setscheduler(task, &attr, /*user=*/true, /*pi=*/true)

`Sched::copy_attr(uattr) -> Result<SchedAttr>`:
1. let usize: u32 = copy_from_user(&uattr.size, 4)?;       // -EFAULT
2. if usize < SCHED_ATTR_SIZE_VER0 { return Err(-EINVAL); }
3. let ksize = size_of::<SchedAttr>() as u32;
4. if usize > ksize {
       /* check the extra tail is all-zero */
       let mut buf = [0u8; 64];
       for off in (ksize..usize).step_by(buf.len() as u32) {
           let n = min(buf.len() as u32, usize - off) as usize;
           copy_from_user_slice(&mut buf[..n], uattr.as_bytes() + off)?;
           if buf[..n].iter().any(|&b| b != 0) { return Err(-E2BIG); }
       }
   }
5. /* copy known prefix */
6. let mut attr = SchedAttr::zeroed();
7. copy_from_user_slice(attr.as_bytes_mut(), uattr.as_bytes(), min(usize, ksize) as usize)?;
8. attr.size = ksize;
9. Ok(attr)

`SchedSetattr::validate(attr) -> Result<()>`:
1. let policy = attr.sched_policy & !SCHED_RESET_ON_FORK_MASK_LEGACY;
2. match policy {
       SCHED_NORMAL | SCHED_BATCH => {
           if attr.sched_priority != 0 { return Err(-EINVAL); }
           if attr.sched_nice < -20 || attr.sched_nice > 19 { return Err(-EINVAL); }
       }
       SCHED_IDLE => {
           if attr.sched_priority != 0 { return Err(-EINVAL); }
       }
       SCHED_FIFO | SCHED_RR => {
           if attr.sched_priority < 1 || attr.sched_priority > 99 { return Err(-EINVAL); }
       }
       SCHED_DEADLINE => {
           if attr.sched_priority != 0 { return Err(-EINVAL); }
           if attr.sched_runtime == 0 || attr.sched_deadline == 0 || attr.sched_period == 0 { return Err(-EINVAL); }
           if attr.sched_runtime > attr.sched_deadline { return Err(-EINVAL); }
           if attr.sched_deadline > attr.sched_period { return Err(-EINVAL); }
           if attr.sched_period >= (1u64 << 63) { return Err(-EINVAL); }
       }
       _ => return Err(-EINVAL),
   }
3. if attr.sched_flags & !SCHED_FLAG_ALL != 0 { return Err(-EINVAL); }
4. if attr.sched_flags & (SCHED_FLAG_RECLAIM|SCHED_FLAG_DL_OVERRUN) != 0 && policy != SCHED_DEADLINE { return Err(-EINVAL); }
5. if attr.sched_flags & SCHED_FLAG_UTIL_CLAMP_MIN != 0 && attr.sched_util_min > SCHED_CAPACITY_SCALE { return Err(-EINVAL); }
6. if attr.sched_flags & SCHED_FLAG_UTIL_CLAMP_MAX != 0 && attr.sched_util_max > SCHED_CAPACITY_SCALE { return Err(-EINVAL); }
7. if (attr.sched_flags & SCHED_FLAG_UTIL_CLAMP) == SCHED_FLAG_UTIL_CLAMP && attr.sched_util_min > attr.sched_util_max { return Err(-EINVAL); }
8. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `copy_attr_size_versioning` | INVARIANT | per-call: usize > ksize ⟹ tail all-zero or -E2BIG. |
| `validate_per_policy` | INVARIANT | per-policy: priority/nice/deadline-triple bounds enforced. |
| `flags_zero_only` | INVARIANT | per-call: flags arg != 0 ⟹ -EINVAL. |
| `dl_admission_atomic` | INVARIANT | per-DEADLINE: admission test atomic w.r.t. global bandwidth. |
| `util_clamp_bounds` | INVARIANT | per-call: util-clamp values ∈ [0,1024], min ≤ max. |
| `lsm_hook_invoked` | INVARIANT | per-call: security_task_setscheduler pre-commit. |
| `cap_sys_nice_for_dl` | INVARIANT | per-DEADLINE: capable(CAP_SYS_NICE) required. |

### Layer 2: TLA+

`kernel/sched_setattr.tla`:
- States: per-task policy/params + per-root-domain DL bandwidth.
- Properties:
  - `safety_dl_admission_sound` — per-call: success ⟹ total DL utilization ≤ bandwidth cap.
  - `safety_validation_pre_commit` — per-call: failure leaves task untouched.
  - `safety_size_versioning` — per-call: tail must be zero or -E2BIG.
  - `liveness_terminates` — per-call: O(n_cpus) for admission, otherwise O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: flags != 0 ⟹ -EINVAL | `SchedSetattr::do_call` |
| `copy_attr` post: tail-bytes all zero on success | `Sched::copy_attr` |
| `validate` post: ret Ok ⟹ shape invariants hold | `SchedSetattr::validate` |
| `__sched_setscheduler` post: task state updated under rq lock | `Sched::__sched_setscheduler` |

### Layer 4: Verus / Creusot functional

Per-`sched_setattr(2)` man-page equivalence. LTP `sched_setattr01..04` pass. SCHED_DEADLINE bandwidth math matches the GRUB admission proof.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sched_setattr(2)` reinforcement:

- **Per-size-versioning bounded loop** — defense against per-O(n)-userspace-controlled iteration (n is capped at attr.size which is u32 capped at PAGE_SIZE by sanity check).
- **Per-validation-before-commit** — defense against per-partial-update on validation failure.
- **Per-DL-admission control** — defense against per-RT-DoS via bandwidth oversubscription.
- **Per-flag-policy compatibility check** — defense against per-RECLAIM/DL_OVERRUN on non-deadline tasks.
- **Per-util-clamp bounds** — defense against per-OOB clamp value confusing schedutil.
- **Per-LSM hook pre-commit** — defense against per-policy bypass.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `attr`** — every byte of the attr buffer is fetched via copy_from_user; UDEREF blocks per-kernel-pointer smuggling for the entire size-versioned tail.
- **CAP_SYS_NICE strict for SCHED_FIFO/RR/DEADLINE** — grsec demands CAP_SYS_NICE for ANY real-time elevation (including raising util-clamp above the SoC's prior ceiling), ignoring RLIMIT_RTPRIO when GRKERNSEC_HARDEN is on.
- **GRKERNSEC_RESLOG on rlimit violations** — RLIMIT_RTPRIO ceiling hits, deadline admission failures, and util-clamp denials are logged with task name/uid/policy/attempted params.
- **Sched_deadline attack-surface reduction** — under sandbox flags (no_new_privs, seccomp), SCHED_DEADLINE is denied wholesale with -EPERM; in non-sandboxed paths, deadline admission is paired with a per-uid RT bandwidth quota (separate from cpu.rt_runtime_us cgroup knob).
- **GRKERNSEC_CHROOT_NICE** — inside a grsec chroot, RT/DEADLINE policy changes are denied regardless of capability; only NORMAL/BATCH/IDLE permitted.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset per call.
- **No_new_privs gate** — when current has PR_SET_NO_NEW_PRIVS, any policy/flag that elevates priority (FIFO/RR/DEADLINE, util-clamp raise, RESET_ON_FORK clear) is denied with -EPERM.
- **Util-clamp clamping** — grsec adds an additional per-uid util-clamp ceiling (separate from per-task), preventing low-priv users from spiking SoC-wide power policy.
- **Sandbox SCHED_DEADLINE attack-surface reduction** — even with CAP_SYS_NICE, deadline parameters are clamped: runtime ≥ 100us, deadline ≤ 1s, period ≤ 1s under sandbox; outside these bounds returns -EINVAL.
- **KEEPCAPS aware** — capability checks honor effective set; SECBIT_KEEP_CAPS does not let demoted-then-rooted tasks bypass.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `sched_getattr(2)` (Tier-5 separate doc).
- `sched_setscheduler(2)` (Tier-5 separate doc — legacy API).
- SCHED_DEADLINE EDF / CBS internals (Tier-3 in `kernel/sched-deadline.md`).
- util-clamp / EAS internals (Tier-3 in `kernel/sched-util-clamp.md`).
- RT bandwidth controller (Tier-3 in `kernel/sched-rt-bandwidth.md`).
- Implementation code.
