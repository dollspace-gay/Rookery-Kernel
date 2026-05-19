# Tier-5 syscall: getrlimit(2) — syscall 97

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE2(getrlimit), do_prlimit)
  - include/uapi/linux/resource.h (struct rlimit)
  - include/uapi/asm-generic/resource.h (RLIMIT_*, RLIM_INFINITY)
  - arch/x86/entry/syscalls/syscall_64.tbl (97  common  getrlimit)
-->

## Summary

`getrlimit(2)` reads the calling process's current resource limit for `resource`. It is the read-only, self-only legacy counterpart of `prlimit64(2)`. Like `setrlimit`, it carries the 32-or-64-bit `struct rlimit` quirk and is only of interest to legacy binaries and ABI conformance — glibc since 2.13 routes `getrlimit` through `prlimit64`. Two related historical syscalls also exist: `ugetrlimit(__NR_ugetrlimit=191 on x86_64-arm-only or i386 emul)` was an early 64-bit-clean variant; `prlimit64` superseded both.

Critical for: legacy shell `ulimit -a`, statically-linked binaries, kernel-conformance test suites.

## Signature

```c
int getrlimit(int resource, struct rlimit *rlim);
```

```c
struct rlimit {
    unsigned long rlim_cur;
    unsigned long rlim_max;
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `resource` | `int` | in | One of `RLIMIT_*` (0..15). |
| `rlim` | `struct rlimit *` | out | Receives current `{rlim_cur, rlim_max}`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. `*rlim` populated. |
| `-1` + `errno` | Failure; `*rlim` unmodified. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `rlim` user pointer invalid. |
| `EINVAL` | `resource` not in `[0, RLIM_NLIMITS)`. |

## ABI surface

```text
__NR_getrlimit  (x86_64)  = 97
__NR_getrlimit  (i386)    = 76      /* deprecated; ugetrlimit (191) preferred */
__NR_getrlimit  (generic) = 163     /* arm64, riscv */
__NR_ugetrlimit (i386)    = 191     /* 32-bit unsigned variant */
__NR_old_getrlimit (i386) = 76      /* pre-2.4 signed variant */

/* On 32-bit kernels, struct rlimit fields are 32-bit. */
/* On 64-bit kernels, fields are 64-bit unsigned long. */

RLIM_INFINITY width matches struct rlimit field width.
```

### Historical compat quirks (i386)

```text
sys_old_getrlimit (76):  signed-long rlimit; values > 0x7FFFFFFF clamped.
sys_ugetrlimit    (191): unsigned-long rlimit; values clamped at 0xFFFFFFFE,
                          and 0xFFFFFFFF means RLIM_INFINITY.

In modern Rookery, sys_getrlimit on x86_64 is the same as sys_ugetrlimit on i386
(both unsigned).
```

## Compatibility contract

REQ-1: Syscall number is **97** on x86_64; **163** on generic-syscall archs. ABI-stable.

REQ-2: Target is always the calling process (`current().group_leader()`). No `pid` argument.

REQ-3: `resource` MUST be in `[0, RLIM_NLIMITS) = [0, 16)`. Else `-EINVAL`.

REQ-4: Reads from `task.rlim[resource]` under the per-task seqlock read side; observation is atomic (no torn read).

REQ-5: Writes `*rlim = current_limit`. On 32-bit ABI, any kernel-internal value > `0xFFFFFFFE` is clamped to `0xFFFFFFFF` (RLIM_INFINITY_32). On 64-bit ABI, no clamping; raw `u64` returned.

REQ-6: Implementation delegates to `do_prlimit(0, resource, NULL, &out)`. Identical semantics.

REQ-7: getrlimit requires NO capability. Anyone can read their own limits.

REQ-8: `rlim_cur <= rlim_max` invariant maintained by `setrlimit`/`prlimit64`; `getrlimit` does NOT re-enforce — it just reports.

REQ-9: Concurrent `setrlimit`/`prlimit64` on caller's task: getrlimit observes either pre or post snapshot.

REQ-10: Coredump-disabled tasks (`PR_SET_DUMPABLE = 0`): getrlimit is still permitted (not restricted by `/proc` hide rules; it's a self-only read).

REQ-11: getrlimit does NOT log via audit (read-only operation).

REQ-12: Old `sys_old_getrlimit` (i386 only) signed-arithmetic compat path: kernel converts a signed-overflow value to `RLIM_INFINITY_32`. Modern kernels still expose this for static binaries.

## Acceptance Criteria

- [ ] AC-1: `getrlimit(RLIMIT_NOFILE, &rl)` returns 0 and populates `rl.rlim_cur` / `rl.rlim_max`.
- [ ] AC-2: `getrlimit(16, &rl)` returns `-EINVAL`.
- [ ] AC-3: `getrlimit(RLIMIT_NOFILE, NULL)` returns `-EFAULT`.
- [ ] AC-4: After `setrlimit(RLIMIT_NOFILE, &{1024, 1024})`, `getrlimit` reports `{1024, 1024}`.
- [ ] AC-5: After `setrlimit(RLIMIT_NOFILE, &{RLIM_INFINITY, RLIM_INFINITY})`, `getrlimit` reports `RLIM_INFINITY`.
- [ ] AC-6: 32-bit caller, kernel internal value = `0x0FFFFFFFFF`: getrlimit returns `0xFFFFFFFF` (clamped).
- [ ] AC-7: 64-bit caller, kernel internal value = `0x0FFFFFFFFF`: getrlimit returns `0x0FFFFFFFFF` (no clamp).
- [ ] AC-8: Concurrent setrlimit + getrlimit: getrlimit sees atomic snapshot.
- [ ] AC-9: `getrlimit` does not modify any kernel state (read-only).
- [ ] AC-10: Limits returned consistent with `prlimit64(0, resource, NULL, &old)`.
- [ ] AC-11: getrlimit does NOT emit audit record (silent read).
- [ ] AC-12: getrlimit from chroot'd task returns the (possibly grsec-stripped) limits as currently in effect.

## Architecture

```rust
#[syscall(nr = 97, abi = "sysv")]
pub fn sys_getrlimit(resource: i32, rlim: UserPtr<Rlimit>) -> isize {
    Getrlimit::do_getrlimit(resource, rlim)
}
```

`Getrlimit::do_getrlimit(resource, rlim_ptr) -> isize`:
1. if rlim_ptr.is_null() { return Err(EFAULT); }
2. if resource as u32 >= RLIM_NLIMITS { return Err(EINVAL); }
3. /* Snapshot under seqlock read */
4. let lim64: Rlimit64 = {
5.   let guard = current().group_leader().lock_rlimits_read();
6.   guard.rlim[resource as usize].clone()
7. };
8. /* Truncate to ABI width */
9. let lim_abi = Rlimit {
10.   rlim_cur: clamp_to_abi(lim64.rlim_cur),
11.   rlim_max: clamp_to_abi(lim64.rlim_max),
12. };
13. rlim_ptr.copy_out(&lim_abi)?;
14. Ok(0)

`clamp_to_abi(v: u64) -> usize`:
1. if ABI_WORD_BITS == 32 {
2.   if v >= 0xFFFFFFFF { return 0xFFFFFFFF; }     // clamp to 32-bit RLIM_INFINITY
3.   return v as u32 as usize;
4. }
5. v as usize     // 64-bit: identity

(Delegation alternative: forward to `Prlimit::do_prlimit(0, resource, UserPtr::null(), rlim_64_ptr)`; functionally identical.)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `resource_range` | INVARIANT | resource ∈ [0, RLIM_NLIMITS). |
| `read_only_no_mutation` | INVARIANT | post: no task field mutated. |
| `atomic_snapshot` | INVARIANT | (cur, max) drawn from single seqlock observation. |
| `abi_clamp_correct` | INVARIANT | 32-bit ABI: any v ≥ 0xFFFFFFFF clamps to 0xFFFFFFFF. |
| `delegates_to_prlimit` | INVARIANT | getrlimit ≡ prlimit64(0, resource, NULL, &out). |

### Layer 2: TLA+

`kernel/getrlimit.tla`:
- States: task.rlim[0..15].
- Properties:
  - `safety_read_only` — invariant: no state change.
  - `safety_self_only` — invariant: target is always caller.
  - `safety_atomic_snapshot` — invariant.
  - `safety_abi_clamp` — invariant for 32-bit width.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getrlimit` post: *rlim_ptr == current.rlim[resource] (modulo ABI clamp) | `Getrlimit::do_getrlimit` |
| `clamp_to_abi` post: 32-bit clamp ↔ 64-bit identity per ABI_WORD_BITS | `clamp_to_abi` |

### Layer 4: Verus / Creusot functional

Per-`getrlimit(2)` man-page semantic equivalence. LTP `getrlimit01..getrlimit04` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getrlimit(2)` reinforcement:

- **Per-resource range check** — defense against per-out-of-bounds rlim_array read.
- **Per-self-only target** — defense against per-cross-process limit reconnaissance via this path.
- **Per-atomic snapshot** — defense against per-torn-read race.
- **Per-ABI-width clamp** — defense against per-32-bit-overflow misinterpretation by legacy callers.
- **Per-read-only semantics** — defense against per-mutation-via-getrlimit bug pattern.

## Grsecurity / PaX surface

- **PaX UDEREF on `rlim` copy_to_user** — defense against per-rlim-buffer kernel-deref bug.
- **PAX_RANDKSTACK at getrlimit entry** — randomizes kernel stack offset.
- **GRKERNSEC_CHROOT_CAPS** — getrlimit returns the chroot-time-stripped values when running inside grsec chroot (consistent with the post-strip task state).
- **No audit emission required** — getrlimit is a read-only self-query; grsec audit policy does NOT log getrlimit by default (low signal).
- **Per-grsec /proc consistency** — `/proc/self/limits` reads via the same path; grsec ensures both /proc and getrlimit return identical, post-policy values.
- **PaX KERNEXEC** — the read path is in KERNEXEC-mapped text; no R/W gadget to corrupt the read result.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `prlimit64(2)` (covered in `prlimit64.md` — getrlimit is a thin self-read wrapper).
- `setrlimit(2)` (covered in `setrlimit.md`).
- 32-bit i386 `sys_old_getrlimit` and `sys_ugetrlimit` (Tier-3 in `kernel/compat.md`).
- Per-resource enforcement sites (Tier-3 in respective subsystem docs).
- Implementation code.
