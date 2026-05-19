# Tier-5 syscall: setrlimit(2) — syscall 160

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE2(setrlimit), do_prlimit)
  - include/uapi/linux/resource.h (struct rlimit)
  - include/uapi/asm-generic/resource.h (RLIMIT_*, RLIM_INFINITY)
  - arch/x86/entry/syscalls/syscall_64.tbl (160  common  setrlimit)
-->

## Summary

`setrlimit(2)` is the legacy, self-only, 32-or-64-bit variant of `prlimit64(2)`. It modifies the calling process's resource limit for the given `resource`. Glibc since 2.13 redirects `setrlimit` through `prlimit64`; the syscall remains for ABI compatibility with statically-linked binaries, musl variants, and direct-syscall callers. On 32-bit archs `struct rlimit` carries 32-bit `rlim_cur` and `rlim_max` (with `RLIM_INFINITY = (unsigned long)-1`); on 64-bit archs the fields are 64-bit `unsigned long`. Cross-architecture ABI quirks make `prlimit64` strongly preferred for new code.

Critical for: legacy shell `ulimit` (statically-linked busybox, older glibc), portable test suites that bypass libc, ABI conformance.

## Signature

```c
int setrlimit(int resource, const struct rlimit *rlim);
```

```c
struct rlimit {
    unsigned long rlim_cur;     /* soft limit (== rlim_t == unsigned long) */
    unsigned long rlim_max;     /* hard limit */
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `resource` | `int` | in | One of `RLIMIT_*` (0..15). |
| `rlim`     | `const struct rlimit *` | in | New `{rlim_cur, rlim_max}`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. Calling process's limit updated atomically. |
| `-1` + `errno` | Failure; no state mutation. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `rlim` user pointer invalid. |
| `EINVAL` | `resource` not in `[0, RLIM_NLIMITS)`; `rlim->rlim_cur > rlim->rlim_max`; on 32-bit ABI, value would overflow if clamped via incorrect `RLIM_INFINITY`. |
| `EPERM`  | Raising hard limit without `CAP_SYS_RESOURCE` in own user-namespace. |

## ABI surface

```text
__NR_setrlimit (x86_64)  = 160
__NR_setrlimit (i386)    = 75
__NR_setrlimit (generic) = 164    /* arm64, riscv */

/* On 32-bit architectures, struct rlimit fields are 32-bit. */
/* On x86_64 / arm64 / riscv64 / ppc64 / s390x, fields are 64-bit. */

RLIM_INFINITY (per ABI width)
  32-bit: 0xFFFFFFFF
  64-bit: 0xFFFFFFFFFFFFFFFF

RLIMIT_* enum identical to prlimit64 (0..15).
RLIM_NLIMITS = 16.

Compat path
  On i386 / s390 / ppc compat: setrlimit clamps any 64-bit kernel value above 32-bit max to RLIM_INFINITY_32 when returning via getrlimit; setrlimit accepts the 32-bit RLIM_INFINITY and promotes internally to 64-bit RLIM_INFINITY.
```

## Compatibility contract

REQ-1: Syscall number is **160** on x86_64; **164** on generic-syscall archs. ABI-stable.

REQ-2: Target is always the calling process (`current().group_leader()`). There is no `pid` argument.

REQ-3: `resource` MUST be in `[0, RLIM_NLIMITS)`. Else `-EINVAL`.

REQ-4: `rlim->rlim_cur <= rlim->rlim_max`. Else `-EINVAL`.

REQ-5: Raising hard limit requires `CAP_SYS_RESOURCE` in the calling task's user-namespace. Else `-EPERM`.

REQ-6: Lowering hard limit is permitted unconditionally.

REQ-7: Per-`RLIMIT_NOFILE`: hard limit ≤ `sysctl_nr_open` (else `-EPERM` even with cap).

REQ-8: 32-bit compat path: kernel-internal `rlim64_t` is always 64-bit. The compat shim:
  - Reads 32-bit user struct.
  - Promotes `RLIM_INFINITY_32` (0xFFFFFFFF) to `RLIM_INFINITY_64` (0xFFFFFFFFFFFFFFFF).
  - Calls `do_prlimit(current_pid, resource, &new64, NULL)`.

REQ-9: On 32-bit ABI, if kernel-internal value > 0xFFFFFFFE (i.e. effectively RLIM_INFINITY in 64-bit), `getrlimit` later reports `0xFFFFFFFF`; setrlimit must therefore round-trip semantically (idempotent if userland writes back what it read).

REQ-10: Atomicity: read-modify of `task.rlim[resource]` under per-task `seqlock` write side. Concurrent readers (getrlimit / prlimit64) see either pre or post.

REQ-11: Implementation strictly delegates to `do_prlimit(0, resource, &new, NULL)`. Therefore semantics are identical to `prlimit64(0, resource, &new, NULL)`.

REQ-12: AUDIT_RLIMIT record emitted on success.

REQ-13: Limits inherited across `fork`; preserved across `execve`.

REQ-14: setrlimit does NOT modify the bounding-set-style ceilings (`sysctl_nr_open`, `kernel.threads-max`); it stays within them.

## Acceptance Criteria

- [ ] AC-1: `setrlimit(RLIMIT_NOFILE, &{1024, 1024})` succeeds; subsequent `getrlimit` reads back same.
- [ ] AC-2: `setrlimit(RLIMIT_NOFILE, &{4096, 8192})` with current hard = 4096 returns `-EPERM` (no cap).
- [ ] AC-3: With `CAP_SYS_RESOURCE`, `setrlimit(RLIMIT_NOFILE, &{4096, 8192})` succeeds.
- [ ] AC-4: `setrlimit(RLIMIT_NOFILE, &{cur > max, max})` returns `-EINVAL`.
- [ ] AC-5: `setrlimit(16, &any)` (out of range) returns `-EINVAL`.
- [ ] AC-6: `setrlimit(RLIMIT_NOFILE, NULL)` returns `-EFAULT`.
- [ ] AC-7: `setrlimit(RLIMIT_NOFILE, &{N, sysctl_nr_open + 1})` with cap returns `-EPERM`.
- [ ] AC-8: 32-bit caller: setrlimit accepts `0xFFFFFFFF` and round-trips through getrlimit as `0xFFFFFFFF`.
- [ ] AC-9: `setrlimit(RLIMIT_CORE, &{0, 0})` then segfault yields no coredump.
- [ ] AC-10: `setrlimit(RLIMIT_NPROC, &{0, 0})` then `fork` returns `-EAGAIN`.
- [ ] AC-11: `setrlimit(RLIMIT_AS, &{1 GiB, 1 GiB})` then `mmap(2 GiB)` returns `MAP_FAILED` with `ENOMEM`.
- [ ] AC-12: AUDIT_RLIMIT emitted on success.
- [ ] AC-13: Limits inherited across `fork`.
- [ ] AC-14: Limits preserved across `execve`.
- [ ] AC-15: Failed setrlimit (any reason): limit unchanged.

## Architecture

```rust
#[syscall(nr = 160, abi = "sysv")]
pub fn sys_setrlimit(resource: i32, rlim: UserPtr<Rlimit>) -> isize {
    Setrlimit::do_setrlimit(resource, rlim)
}
```

`struct Rlimit { rlim_cur: usize, rlim_max: usize }`  // usize = ABI-word

`Setrlimit::do_setrlimit(resource, rlim_ptr) -> isize`:
1. if rlim_ptr.is_null() { return Err(EFAULT); }
2. let user: Rlimit = rlim_ptr.copy_in()?;
3. /* Promote 32-bit RLIM_INFINITY to 64-bit on compat path */
4. let new64 = Rlimit64 {
5.   rlim_cur: promote_infinity(user.rlim_cur),
6.   rlim_max: promote_infinity(user.rlim_max),
7. };
8. /* Delegate to prlimit64 with pid=0 (self), no old_limit needed */
9. Prlimit::do_prlimit(0, resource, UserPtr::new_inline(&new64), UserPtr::null())
```

`promote_infinity(v: usize) -> u64`:
1. if ABI_WORD_BITS == 32 && v == 0xFFFFFFFF { return RLIM_INFINITY_64; }
2. v as u64

(The bulk of the logic — range check, soft ≤ hard, cap-check on hard-raise, sysctl_nr_open ceiling, atomic commit, audit — lives in `Prlimit::do_prlimit`. See `prlimit64.md`.)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `resource_range` | INVARIANT | resource ∈ [0, RLIM_NLIMITS). |
| `soft_le_hard` | INVARIANT | post: rlim_cur ≤ rlim_max. |
| `hard_raise_requires_cap` | INVARIANT | new.rlim_max > old.rlim_max ⟹ CAP_SYS_RESOURCE. |
| `nofile_cap_bounded` | INVARIANT | RLIMIT_NOFILE.rlim_max ≤ sysctl_nr_open. |
| `delegates_to_prlimit` | INVARIANT | setrlimit ≡ prlimit64(0, resource, &new, NULL). |
| `compat_promote` | INVARIANT | 32-bit RLIM_INFINITY → 64-bit RLIM_INFINITY. |

### Layer 2: TLA+

`kernel/setrlimit.tla`:
- States: task.rlim[0..15].
- Properties:
  - `safety_self_only` — invariant: target is always caller.
  - `safety_equiv_prlimit64` — semantic equivalence to `prlimit64(0, ...)`.
  - `safety_soft_le_hard` — invariant for every resource.
  - `safety_hard_monotone_without_cap` — invariant.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setrlimit` post: result == do_prlimit(0, resource, &new, NULL) | `Setrlimit::do_setrlimit` |
| `promote_infinity` post: 32-bit RLIM_INFINITY ⟺ 64-bit RLIM_INFINITY | `promote_infinity` |

### Layer 4: Verus / Creusot functional

Per-`setrlimit(2)` man-page semantic equivalence. LTP `setrlimit01..setrlimit04` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setrlimit(2)` reinforcement:

- **Per-self-only target** — defense against per-cross-process limit injection (cannot specify pid).
- **Per-resource range check** — defense against per-out-of-bounds rlim_array write.
- **Per-soft ≤ hard invariant** — defense against per-malformed-limit confusion.
- **Per-CAP_SYS_RESOURCE gate on hard-raise** — defense against per-self-elevation.
- **Per-sysctl_nr_open ceiling on NOFILE** — defense against per-fd-table-exhaustion DoS.
- **Per-32-bit RLIM_INFINITY promotion** — defense against per-32-bit-clamp confusion.
- **Per-atomic commit** — defense against per-torn-state read.
- **Per-AUDIT_RLIMIT** — defense against per-limit-drop log elision.

## Grsecurity / PaX surface

- **PaX UDEREF on `rlim` copy_from_user** — defense against per-rlim-buffer kernel-deref bug.
- **PAX_RANDKSTACK at setrlimit entry** — randomizes kernel stack offset.
- **GRKERNSEC_CHROOT_CAPS** — `CAP_SYS_RESOURCE` stripped on chroot entry; chroot'd tasks cannot raise hard limits.
- **GRKERNSEC_RESLOG** — every successful set logged.
- **Per-grsec RLIMIT_NPROC strict** — grsec enforces RLIMIT_NPROC per real-uid more strictly than upstream (includes RT and stopped tasks).
- **Per-grsec RLIMIT_NOFILE pin** — grsec policy can pin a global ceiling lower than `sysctl_nr_open`.
- **GRKERNSEC_BRUTE** — repeated `-EPERM` on hard-raise triggers per-uid rate-limit.
- **PaX MPROTECT** — RLIMIT_STACK lowering composed with MPROTECT lock keeps stack non-executable.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `prlimit64(2)` (covered in `prlimit64.md` — setrlimit is a thin self-only wrapper).
- `getrlimit(2)` (Tier-5 separate doc).
- Per-resource enforcement sites (Tier-3 in respective subsystem docs).
- 32-bit ABI compat shim (Tier-3 in `kernel/compat.md`).
- Implementation code.
