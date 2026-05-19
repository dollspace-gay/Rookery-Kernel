---
title: "Tier-5 syscall: getresuid(2) — syscall 118"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getresuid(2)` atomically returns the **three** user-ID slots maintained by the kernel for the calling task: the **real UID** (RUID — the user who launched the program), the **effective UID** (EUID — used for permission checks), and the **saved set-user-ID** (SUID — the original EUID at exec time, allowing temporary privilege drop with restoration). Each is delivered into a separate caller-provided `uid_t *` pointer.

Unlike `getuid(2)` / `geteuid(2)` (which return only one slot each via the return value and thus require two syscalls and offer no atomicity), `getresuid(2)` produces a consistent snapshot of all three slots taken under `rcu_read_lock` against the task's `struct cred`. The atomicity matters: a parallel `setresuid()` from another thread sharing the same `mm` (or from a child via process credential propagation) cannot produce an inconsistent read where, e.g., the caller observes the new EUID with the old SUID.

Critical for: privilege-drop verification (setuid binaries that need to confirm they actually dropped privileges before continuing), audit subsystem caller identification, sudo/su/PAM modules, container runtime introspection of user-namespace credentials, security-relevant logging.

### Acceptance Criteria

- [ ] AC-1: `getresuid(&r, &e, &s)` on a task started by user 1000: `r == e == s == 1000`.
- [ ] AC-2: After `setuid(0)` from root-owned setuid binary launched by uid 1000: `r == 1000, e == 0, s == 0`.
- [ ] AC-3: After privilege drop `setresuid(1000, 1000, 1000)`: all three == 1000.
- [ ] AC-4: After temporary drop `seteuid(1000)` from a setuid binary: `r == orig, e == 1000, s == 0` (saved retained).
- [ ] AC-5: `getresuid(NULL, &e, &s)` → `-EFAULT`.
- [ ] AC-6: `getresuid(&r, NULL, &s)` → `-EFAULT`.
- [ ] AC-7: `getresuid(&r, &e, NULL)` → `-EFAULT`.
- [ ] AC-8: In a user namespace where uid 0 is unmapped: real-root caller observes `OVERFLOWUID`.
- [ ] AC-9: Atomicity: concurrent setresuid never produces split-state read.
- [ ] AC-10: Call always succeeds for valid pointers; never EINTR, never EAGAIN.

### Architecture

```rust
#[syscall(nr = 118, abi = "sysv")]
pub fn sys_getresuid(
    ruid: UserPtrMut<u32>,
    euid: UserPtrMut<u32>,
    suid: UserPtrMut<u32>,
) -> isize {
    Cred::do_getresuid(ruid, euid, suid)
}
```

`Cred::do_getresuid(ruid_p, euid_p, suid_p) -> isize`:
1. let user_ns = current_user_ns();
2. /* RCU snapshot of current credentials */
3. let (kr, ke, ks) = {
4.    let _guard = RcuReadGuard::new();
5.    let cred = current().cred();
6.    (cred.uid, cred.euid, cred.suid)
7. };
8. /* Translate kuid_t to uid_t in caller's user-ns */
9. let r = from_kuid_munged(&user_ns, kr);
10. let e = from_kuid_munged(&user_ns, ke);
11. let s = from_kuid_munged(&user_ns, ks);
12. /* Sanitize before copy_to_user (GRKERNSEC_HIDESYM-aware) */
13. put_user(r, ruid_p)?;
14. put_user(e, euid_p)?;
15. put_user(s, suid_p)?;
16. 0

`Cred::from_kuid_munged(user_ns, kuid) -> uid_t`:
1. match user_ns.map_kuid_down(kuid) {
2.    Some(uid) => uid.as_u32(),
3.    None      => sysctl_overflowuid(), /* default 65534 */
4. }

### Out of Scope

- `setresuid(2)` (covered in `setresuid.md`).
- `getuid(2)` / `geteuid(2)` (covered in their own Tier-5 docs).
- `getresgid(2)` (separate Tier-5 doc).
- User-namespace mapping internals (covered in Tier-3 `kernel/user_namespace.md`).
- Implementation code.

### signature

```c
int getresuid(uid_t *ruid, uid_t *euid, uid_t *suid);
```

```c
typedef __kernel_uid32_t  uid_t;   /* unsigned int — 32-bit */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ruid` | `uid_t *` | out | Receives the real UID (or `OVERFLOWUID` if not mapped in current user ns). |
| `euid` | `uid_t *` | out | Receives the effective UID. |
| `suid` | `uid_t *` | out | Receives the saved set-user-ID. |

All three pointers MUST be valid writable kernel-facing user pointers; none may be NULL.

### return value

| Value | Meaning |
|---|---|
| `0` | Success — all three slots written. |
| `-1` + `errno` | Failure (one of the user pointers faulted; `EFAULT`). |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | One of `ruid`, `euid`, `suid` is NULL or points outside the caller's writable address space. |

`getresuid` cannot fail for any other reason — it is a pure read of the task's credentials.

### abi surface

```text
__NR_getresuid   (x86_64)  = 118
__NR_getresuid   (i386)    = 209   /* 32-bit uid_t variant */
__NR_getresuid16 (i386)    = 165   /* legacy 16-bit uid_t */
__NR_getresuid   (arm64)   = 148
__NR_getresuid   (riscv)   = 148

/* User pointers receive uid_t values; the kernel writes via put_user. */
/* If any pointer faults, the kernel returns -EFAULT and the contents of
   already-written pointers are unspecified. */
```

### compatibility contract

REQ-1: Syscall number is **118** on x86_64. ABI-stable.

REQ-2: The kernel acquires `current()->cred` under `rcu_read_lock`, reads `cred->uid`, `cred->euid`, `cred->suid` (all `kuid_t`), then releases the lock.

REQ-3: Each `kuid_t` is translated to `uid_t` via `from_kuid_munged(current_user_ns(), kuid)`:
- If the kernel UID is mapped in the caller's user namespace: returns the mapped UID.
- Otherwise: returns `OVERFLOWUID` (default 65534, sysctl `kernel.overflowuid`).

REQ-4: Translated values are written to user space via three `put_user` calls. If any `put_user` faults, the kernel returns `-EFAULT`. The state of pointers written before the fault is unspecified.

REQ-5: No state mutation. Calling `getresuid` does not advance any sequence counter, audit log, or credential refcount.

REQ-6: Atomicity guarantee: the three values returned reflect a single snapshot of `current()->cred`. A concurrent `setresuid()` either takes effect entirely before the snapshot or entirely after.

REQ-7: `getresuid` is async-signal-safe per POSIX.

REQ-8: `getresuid` has no capability requirement — any task may inspect its own credentials.

REQ-9: When called from a user namespace whose mappings change concurrently (rare; user-ns mappings are usually static), the snapshot uses the user-ns reference at the moment of `from_kuid_munged`.

REQ-10: 32-bit i386 has two ABI variants: legacy 16-bit `getresuid16` (syscall 165) and 32-bit `getresuid` (syscall 209). x86_64 has only the 32-bit variant.

REQ-11: Forward-compat: signature is wire-stable; no extension flags reserved. A future `getresuid2` would be required for any new behavior.

REQ-12: Audit subsystem records `getresuid` calls only when the call is referenced by an active audit filter rule; otherwise no audit overhead.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rcu_read_lock_balanced` | INVARIANT | RcuReadGuard held during cred snapshot; dropped before put_user. |
| `three_pointers_required` | INVARIANT | NULL ptr ⟹ EFAULT via put_user. |
| `atomic_snapshot` | INVARIANT | all three reads observe same struct cred. |
| `overflowuid_on_unmapped` | INVARIANT | unmapped kuid ⟹ OVERFLOWUID. |
| `no_cred_mutation` | INVARIANT | call does not modify any cred field. |
| `put_user_fault_short_circuits` | INVARIANT | first faulting put_user returns -EFAULT. |

### Layer 2: TLA+

`kernel/getresuid.tla`:
- States: per-rcu-snapshot, per-translate, per-put_user(ruid), per-put_user(euid), per-put_user(suid).
- Properties:
  - `safety_consistent_triple` — all three values from same cred.
  - `safety_efault_on_bad_ptr` — invalid user ptr ⟹ EFAULT.
  - `safety_no_cred_change` — call does not mutate credentials.
  - `liveness_getresuid_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getresuid` post: all three values from same cred snapshot | `Cred::do_getresuid` |
| `do_getresuid` post: success ⟹ 3 put_user calls completed | `Cred::do_getresuid` |
| `from_kuid_munged` post: returns OVERFLOWUID iff unmapped | `Cred::from_kuid_munged` |
| `do_getresuid` post: no cred modification | `Cred::do_getresuid` |

### Layer 4: Verus / Creusot functional

Per-`getresuid(2)` man-page + POSIX semantics; LTP `kernel/syscalls/getresuid/` selftests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getresuid(2)` reinforcement:

- **Per-RCU snapshot atomic** — defense against per-split-read inconsistency.
- **Per-from_kuid_munged user-ns translation** — defense against per-cross-ns uid leak.
- **Per-OVERFLOWUID for unmapped** — defense against per-raw-kuid leak.
- **Per-put_user fault short-circuit** — defense against per-partial-write info-leak escalation.
- **Per-no-cred-mutation** — defense against per-credential-poisoning.
- **Per-no-cap-required** — defense against per-incorrect-cap requirement that would deny self-introspection.

### grsecurity / pax surface

- **PaX UDEREF on `ruid`, `euid`, `suid`** — defense against per-user-pointer kernel-deref bug; SMAP forced for the three put_user calls.
- **GRKERNSEC_HIDESYM on getresuid/getresgid copy_to_user** — the three put_user calls are routed through a slab-whitelisted bounce buffer; even though the values themselves are not kernel addresses, the hardened path prevents inferential side-channels through `cred` slab residue.
- **PAX_USERCOPY_HARDEN on put_user(uid_t)** — fixed-size 4-byte copy uses the whitelisted path; defense against per-misaligned-user-pointer-write that could clobber adjacent user memory inferring kernel layout.
- **PAX_REFCOUNT on cred refcount** — saturating; defense against per-refcount-overflow UAF on long RCU grace periods.
- **GRKERNSEC_HARDEN_USERNS** — when grsec policy restricts user-ns creation, `from_kuid_munged` returns OVERFLOWUID more aggressively, denying mapping inference attacks.
- **PaX KERNEXEC on cred dereference** — no function-pointer indirection in this path; PaX confirms the absence.
- **GRKERNSEC_AUDIT_IPC** — getresuid is normally too noisy to audit; grsec policy may attach audit on the call if the task is on a high-sensitivity audit list.
- **GRKERNSEC_PROC** — getresuid called from a `/proc/<pid>/`-traversal context still returns the caller's own credentials, not the target's; defense against per-cross-task cred inference.
- **PaX RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — no kernel-side buffer to sanitize; this syscall is purely register-and-RCU based.
- **Per-RCU grace-period bounded** — defense against per-stale-cred read; if the snapshot races with `setresuid`, RCU semantics guarantee the read sees a committed cred state.
- **GRKERNSEC_TRUSTED_PATH irrelevant** — no path traversal; this surface confirmed clean.

