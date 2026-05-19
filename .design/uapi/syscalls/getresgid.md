# Tier-5 syscall: getresgid(2) — syscall 120

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE3(getresgid))
  - include/linux/cred.h (struct cred.gid, .egid, .sgid)
  - include/linux/uidgid.h (kgid_t, from_kgid_munged)
  - include/linux/user_namespace.h (current_user_ns)
  - arch/x86/entry/syscalls/syscall_64.tbl (120  common  getresgid)
-->

## Summary

`getresgid(2)` atomically returns the **three** group-ID slots maintained by the kernel for the calling task: the **real GID** (RGID — the primary group at process start), the **effective GID** (EGID — used for permission checks), and the **saved set-group-ID** (SGID — the original EGID at exec time, allowing temporary group-privilege drop with restoration). Each is delivered into a separate caller-provided `gid_t *` pointer.

It is the GID-side complement to `getresuid(2)` and offers the same atomicity property: the three values returned reflect a single RCU snapshot of `current()->cred`. Without this atomicity, a parallel `setresgid()` could lead to inconsistent observations across two sequential `getgid`/`getegid` calls. The atomic read is critical for setgid binaries that temporarily lower group privileges and need to verify the drop before proceeding.

Critical for: setgid binary privilege-drop verification, audit caller-group identification, container runtime introspection of supplementary-vs-primary group state, sudo/pam group-membership checks, security-relevant logging.

## Signature

```c
int getresgid(gid_t *rgid, gid_t *egid, gid_t *sgid);
```

```c
typedef __kernel_gid32_t  gid_t;   /* unsigned int — 32-bit */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `rgid` | `gid_t *` | out | Receives the real GID (or `OVERFLOWGID` if not mapped in current user ns). |
| `egid` | `gid_t *` | out | Receives the effective GID. |
| `sgid` | `gid_t *` | out | Receives the saved set-group-ID. |

All three pointers MUST be valid writable kernel-facing user pointers; none may be NULL.

## Return value

| Value | Meaning |
|---|---|
| `0` | Success — all three slots written. |
| `-1` + `errno` | Failure (one of the user pointers faulted; `EFAULT`). |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | One of `rgid`, `egid`, `sgid` is NULL or points outside the caller's writable address space. |

`getresgid` cannot fail for any other reason.

## ABI surface

```text
__NR_getresgid   (x86_64)  = 120
__NR_getresgid   (i386)    = 211   /* 32-bit gid_t variant */
__NR_getresgid16 (i386)    = 171   /* legacy 16-bit gid_t */
__NR_getresgid   (arm64)   = 150
__NR_getresgid   (riscv)   = 150

/* User pointers receive gid_t values; the kernel writes via put_user. */
/* If any pointer faults, the kernel returns -EFAULT and the contents of
   already-written pointers are unspecified. */
```

## Compatibility contract

REQ-1: Syscall number is **120** on x86_64. ABI-stable.

REQ-2: The kernel acquires `current()->cred` under `rcu_read_lock`, reads `cred->gid`, `cred->egid`, `cred->sgid` (all `kgid_t`), then releases the lock.

REQ-3: Each `kgid_t` is translated to `gid_t` via `from_kgid_munged(current_user_ns(), kgid)`:
- If the kernel GID is mapped in the caller's user namespace: returns the mapped GID.
- Otherwise: returns `OVERFLOWGID` (default 65534, sysctl `kernel.overflowgid`).

REQ-4: Translated values are written to user space via three `put_user` calls. If any `put_user` faults, the kernel returns `-EFAULT`. The state of pointers written before the fault is unspecified.

REQ-5: No state mutation. Calling `getresgid` does not advance any sequence counter, audit log, or credential refcount.

REQ-6: Atomicity guarantee: the three values returned reflect a single snapshot of `current()->cred`. A concurrent `setresgid()` either takes effect entirely before the snapshot or entirely after.

REQ-7: `getresgid` is async-signal-safe per POSIX.

REQ-8: `getresgid` has no capability requirement — any task may inspect its own credentials.

REQ-9: Supplementary groups are NOT returned by this call. They are queried via `getgroups(2)`.

REQ-10: 32-bit i386 has two ABI variants: legacy 16-bit `getresgid16` (syscall 171) and 32-bit `getresgid` (syscall 211). x86_64 has only the 32-bit variant.

REQ-11: Forward-compat: signature is wire-stable; no extension flags reserved.

REQ-12: Audit subsystem records `getresgid` calls only when referenced by an active audit filter rule.

## Acceptance Criteria

- [ ] AC-1: `getresgid(&r, &e, &s)` on a task started with primary group 1000: `r == e == s == 1000`.
- [ ] AC-2: After `setgid(0)` from root-group setgid binary launched by gid 1000: `r == 1000, e == 0, s == 0`.
- [ ] AC-3: After privilege drop `setresgid(1000, 1000, 1000)`: all three == 1000.
- [ ] AC-4: After temporary drop `setegid(1000)` from a setgid binary: `r == orig, e == 1000, s == 0` (saved retained).
- [ ] AC-5: `getresgid(NULL, &e, &s)` → `-EFAULT`.
- [ ] AC-6: `getresgid(&r, NULL, &s)` → `-EFAULT`.
- [ ] AC-7: `getresgid(&r, &e, NULL)` → `-EFAULT`.
- [ ] AC-8: In a user namespace where gid 0 is unmapped: real-root caller observes `OVERFLOWGID`.
- [ ] AC-9: Atomicity: concurrent setresgid never produces split-state read.
- [ ] AC-10: Call always succeeds for valid pointers; never EINTR, never EAGAIN.

## Architecture

```rust
#[syscall(nr = 120, abi = "sysv")]
pub fn sys_getresgid(
    rgid: UserPtrMut<u32>,
    egid: UserPtrMut<u32>,
    sgid: UserPtrMut<u32>,
) -> isize {
    Cred::do_getresgid(rgid, egid, sgid)
}
```

`Cred::do_getresgid(rgid_p, egid_p, sgid_p) -> isize`:
1. let user_ns = current_user_ns();
2. /* RCU snapshot of current credentials */
3. let (kr, ke, ks) = {
4.    let _guard = RcuReadGuard::new();
5.    let cred = current().cred();
6.    (cred.gid, cred.egid, cred.sgid)
7. };
8. /* Translate kgid_t to gid_t in caller's user-ns */
9. let r = from_kgid_munged(&user_ns, kr);
10. let e = from_kgid_munged(&user_ns, ke);
11. let s = from_kgid_munged(&user_ns, ks);
12. /* Sanitize before copy_to_user (GRKERNSEC_HIDESYM-aware) */
13. put_user(r, rgid_p)?;
14. put_user(e, egid_p)?;
15. put_user(s, sgid_p)?;
16. 0

`Cred::from_kgid_munged(user_ns, kgid) -> gid_t`:
1. match user_ns.map_kgid_down(kgid) {
2.    Some(gid) => gid.as_u32(),
3.    None      => sysctl_overflowgid(), /* default 65534 */
4. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rcu_read_lock_balanced` | INVARIANT | RcuReadGuard held during cred snapshot; dropped before put_user. |
| `three_pointers_required` | INVARIANT | NULL ptr ⟹ EFAULT via put_user. |
| `atomic_snapshot` | INVARIANT | all three reads observe same struct cred. |
| `overflowgid_on_unmapped` | INVARIANT | unmapped kgid ⟹ OVERFLOWGID. |
| `no_cred_mutation` | INVARIANT | call does not modify any cred field. |
| `put_user_fault_short_circuits` | INVARIANT | first faulting put_user returns -EFAULT. |

### Layer 2: TLA+

`kernel/getresgid.tla`:
- States: per-rcu-snapshot, per-translate, per-put_user(rgid), per-put_user(egid), per-put_user(sgid).
- Properties:
  - `safety_consistent_triple` — all three values from same cred.
  - `safety_efault_on_bad_ptr` — invalid user ptr ⟹ EFAULT.
  - `safety_no_cred_change` — call does not mutate credentials.
  - `liveness_getresgid_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getresgid` post: all three values from same cred snapshot | `Cred::do_getresgid` |
| `do_getresgid` post: success ⟹ 3 put_user calls completed | `Cred::do_getresgid` |
| `from_kgid_munged` post: returns OVERFLOWGID iff unmapped | `Cred::from_kgid_munged` |
| `do_getresgid` post: no cred modification | `Cred::do_getresgid` |

### Layer 4: Verus / Creusot functional

Per-`getresgid(2)` man-page + POSIX semantics; LTP `kernel/syscalls/getresgid/` selftests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getresgid(2)` reinforcement:

- **Per-RCU snapshot atomic** — defense against per-split-read inconsistency.
- **Per-from_kgid_munged user-ns translation** — defense against per-cross-ns gid leak.
- **Per-OVERFLOWGID for unmapped** — defense against per-raw-kgid leak.
- **Per-put_user fault short-circuit** — defense against per-partial-write info-leak escalation.
- **Per-no-cred-mutation** — defense against per-credential-poisoning.
- **Per-no-cap-required** — defense against per-incorrect-cap requirement that would deny self-introspection.

## Grsecurity / PaX surface

- **PaX UDEREF on `rgid`, `egid`, `sgid`** — defense against per-user-pointer kernel-deref bug; SMAP forced for the three put_user calls.
- **GRKERNSEC_HIDESYM on getresuid/getresgid copy_to_user** — the three put_user calls are routed through a slab-whitelisted bounce buffer; defense against inferential side-channels through `cred` slab residue.
- **PAX_USERCOPY_HARDEN on put_user(gid_t)** — fixed-size 4-byte copy uses the whitelisted path; defense against per-misaligned-user-pointer-write.
- **PAX_REFCOUNT on cred refcount** — saturating; defense against per-refcount-overflow UAF on long RCU grace periods.
- **GRKERNSEC_HARDEN_USERNS** — when grsec policy restricts user-ns creation, `from_kgid_munged` returns OVERFLOWGID more aggressively.
- **PaX KERNEXEC on cred dereference** — no function-pointer indirection in this path; PaX confirms the absence.
- **GRKERNSEC_AUDIT_IPC** — getresgid is normally too noisy to audit; grsec policy may attach audit on the call if the task is on a high-sensitivity audit list.
- **GRKERNSEC_PROC** — getresgid called from a `/proc/<pid>/`-traversal context still returns the caller's own credentials, not the target's; defense against per-cross-task cred inference.
- **PaX RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — no kernel-side buffer to sanitize; this syscall is purely register-and-RCU based.
- **Per-RCU grace-period bounded** — defense against per-stale-cred read.
- **GRKERNSEC_TRUSTED_PATH irrelevant** — no path traversal; this surface confirmed clean.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `setresgid(2)` (covered in `setresgid.md`).
- `getgid(2)` / `getegid(2)` (covered in their own Tier-5 docs).
- `getresuid(2)` (separate Tier-5 doc).
- `getgroups(2)` (separate Tier-5 doc).
- User-namespace mapping internals (covered in Tier-3 `kernel/user_namespace.md`).
- Implementation code.
