---
title: "Tier-5 syscall: geteuid(2) — syscall 107"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`geteuid(2)` returns the calling task's **effective user ID** (EUID) — the UID actually used for permission checks on open, exec, signal-send, mount, and almost every privileged operation. EUID differs from RUID (`getuid`) when a setuid binary is in effect, when `setuid()`/`seteuid()` has temporarily lowered privilege, or when ptrace traceability is being evaluated. Translated from `cred->euid` through the caller's user-namespace map.

Critical for: privilege gates (`if (geteuid() != 0) return EPERM`), setuid-binary `geteuid() == 0 && getuid() != 0` detection, su/sudo state machines, `access(2)` vs effective-access via `faccessat(AT_EACCESS)`, file-creation owner determination in syscalls that consult `cred->fsuid`, LSM audit caller-ID, `getpw*()` library functions when consulting EUID.

### Acceptance Criteria

- [ ] AC-1: Root process: `geteuid() == 0`.
- [ ] AC-2: Process started by `uid=1000` running a non-setuid binary: `geteuid() == 1000`.
- [ ] AC-3: Setuid-to-root binary executed by `uid=1000`: `geteuid() == 0`, `getuid() == 1000`.
- [ ] AC-4: After `seteuid(1001)` from root: `geteuid() == 1001`, `getuid() == 0`.
- [ ] AC-5: After `setuid(1001)` from root (full setuid): `geteuid() == 1001 == getuid()`.
- [ ] AC-6: In user namespace with `0 1000 1` uid_map, host-uid 1000: `geteuid() == 0`.
- [ ] AC-7: Task whose kuid_euid not mapped in active user-ns: `geteuid() == 65534`.
- [ ] AC-8: `geteuid()` consistent with `/proc/self/status:Uid` (second field).
- [ ] AC-9: `geteuid()` never returns < 0.
- [ ] AC-10: Concurrent `setuid()` race: atomic snapshot.
- [ ] AC-11: `override_creds(temp_cred); rv = geteuid(); revert_creds(temp_cred)` — `rv` reflects `temp_cred.euid` (matters for in-kernel callers; userspace observes RCU-COW cred only).
- [ ] AC-12: i386 syscall 49 returns clamped 16-bit; 201 returns full 32-bit.

### Architecture

```rust
#[syscall(nr = 107, abi = "sysv")]
pub fn sys_geteuid() -> uid_t {
    Geteuid::do_geteuid()
}
```

`Geteuid::do_geteuid() -> uid_t`:
1. let task = current();
2. /* RCU-read of cred (NOT real_cred — geteuid honors override_creds) */
3. let euid = rcu::read(|| {
4.     let cred = task.cred;               // *struct cred
5.     let user_ns = current_user_ns();    // task.cred.user_ns (same as cred above)
6.     UidMap::from_kuid_munged(user_ns, cred.euid)
7. });
8. euid

`UidMap::from_kuid_munged(ns, kuid) -> uid_t`:
1. for entry in ns.uid_map.extents() {
2.     if kuid.val ∈ [entry.lower_first, entry.lower_first + entry.count) {
3.         return entry.first + (kuid.val - entry.lower_first);
4.     }
5. }
6. return OVERFLOWUID;

### Out of Scope

- `getuid(2)` (Tier-5 separate doc — real UID).
- `getresuid(2)` (Tier-5 separate doc — real, effective, saved triple).
- `seteuid(2)` / `setuid(2)` / `setresuid(2)` (Tier-5 separate docs).
- `setfsuid(2)` and fsuid/FSUID semantics (Tier-5 separate doc).
- File capabilities and securebits (Tier-3 in `security/commoncap.md`).
- User-namespace uid_map (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.

### signature

```c
uid_t geteuid(void);
```

### parameters

(none — `SYSCALL_DEFINE0(geteuid)`)

### return value

| Value | Meaning |
|---|---|
| `uid_t` | The calling task's EUID mapped into `current_user_ns()`. |

`geteuid()` cannot fail. POSIX-mandated: never returns `-1`, never sets `errno`.

Return value `(uid_t)-1` (i.e. `OVERFLOWUID` = 65534 by default) indicates the kernel EUID is NOT mapped in the caller's user namespace.

### errors

(none)

### abi surface

```text
__NR_geteuid (x86_64)    = 107
__NR_geteuid (i386)      = 49      /* 16-bit uid_t variant */
__NR_geteuid32 (i386)    = 201     /* 32-bit uid_t — preferred */
__NR_geteuid (arm64)     = 175     /* generic-syscall */
__NR_geteuid (generic)   = 175

typedef __kernel_uid32_t  uid_t;  /* unsigned int — 32-bit */

OVERFLOWUID = 65534

struct cred {
    ...
    kuid_t          uid;            /* RUID */
    kuid_t          euid;           /* EUID — this syscall returns this field */
    kuid_t          suid;           /* SUID */
    kuid_t          fsuid;          /* FSUID — used by VFS for ownership decisions */
    ...
};
```

### compatibility contract

REQ-1: Syscall number is **107** on x86_64; **175** on arm64 / generic. ABI-stable since the earliest Linux.

REQ-2: Returns `from_kuid_munged(current_user_ns(), current->cred->euid)`. Uses `cred` (not `real_cred`) so that `override_creds()` regions report the override-EUID.

REQ-3: The 16-bit i386 ABI (syscall 49) clamps to 65535. Modern i386 userspace uses 201 (`geteuid32`).

REQ-4: After `execve(2)` of a setuid binary: `euid` is set to the binary's owner-uid (file `st_uid`). RUID is unchanged. `geteuid()` then returns the binary's owner UID.

REQ-5: `setuid(2)`: when called by privileged process, sets `uid`, `euid`, `suid`, `fsuid` all to the argument. `geteuid()` reflects post-call. When called by unprivileged process, sets only `euid` (and `fsuid`) — the canonical "lower privilege" form for setuid binaries.

REQ-6: `seteuid(2)`: sets `euid` and `fsuid` only (no change to RUID or SUID). `geteuid()` returns the new value.

REQ-7: `setreuid(2)` / `setresuid(2)`: explicit-arg form. `geteuid()` returns whatever `euid` they leave in cred.

REQ-8: `geteuid()` is what Linux uses for almost all permission checks. The exception is filesystem operations, which use `fsuid` (typically tracks `euid` automatically except when `setfsuid()` decouples them).

REQ-9: `geteuid()` is async-signal-safe and re-entrant. No locks; cred is RCU-protected.

REQ-10: No LSM hook on geteuid path. Public metadata.

REQ-11: Concurrent `setuid/seteuid/setresuid` from another thread: kernel uses RCU-COW; `geteuid()` sees either pre- or post-update cred atomically. Never tears.

REQ-12: User-namespace mapping: returned value is namespace-relative. Unmapped kuid → `OVERFLOWUID`.

REQ-13: For a task transitioning through privileged operations, the sequence `getuid(), geteuid()` is a common pattern; both syscalls operate on the SAME cred snapshot only if userspace samples them under a quiescent thread (RCU does NOT guarantee cross-syscall snapshot stability — each syscall samples independently).

REQ-14: Returns NEVER < 0 (uid_t is unsigned).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `unmapped_returns_overflow` | INVARIANT | per-geteuid: kuid ∉ uid_map ⟹ OVERFLOWUID. |
| `mapped_returns_translation` | INVARIANT | per-geteuid: kuid ∈ uid_map ⟹ correct mapped value. |
| `cred_used_not_real_cred` | INVARIANT | per-geteuid: cred (not real_cred); override_creds honored. |
| `rcu_atomic_cred` | INVARIANT | per-geteuid: concurrent setuid yields consistent snapshot. |
| `no_state_mutation` | INVARIANT | per-geteuid: read-only. |
| `unsigned_return` | INVARIANT | per-geteuid: never < 0. |

### Layer 2: TLA+

`kernel/geteuid.tla`:
- States: per-task cred (with override stack), user-ns uid_map.
- Properties:
  - `safety_overflow_for_unmapped` — unmapped EUID ⟹ OVERFLOWUID.
  - `safety_override_creds_honored` — within override block, geteuid sees override.euid.
  - `safety_atomic_snapshot` — concurrent setuid snapshot is consistent.
  - `liveness_returns` — terminates O(uid_map.extents).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_geteuid` post: ret == from_kuid_munged(current_user_ns, current.cred.euid) | `Geteuid::do_geteuid` |
| `from_kuid_munged` post: OVERFLOWUID iff no map | `UidMap::from_kuid_munged` |
| Cred refcount ≥ 1 during RCU read | `Cred::rcu_read` |

### Layer 4: Verus / Creusot functional

Per-`geteuid(2)` man-page equivalence. LTP `geteuid01..geteuid02` pass. POSIX.1-2008 `geteuid` semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`geteuid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-cred-read race with concurrent setuid/seteuid.
- **Per-overflow-munged for unmapped** — defense against per-namespace UID leak across user-ns boundary.
- **Per-cred (not real_cred) used** — defense against per-override_creds() confusion and audit-logging mismatch.
- **Per-readonly semantics** — defense against per-mutation-via-geteuid bug pattern.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at geteuid entry** — randomizes kernel stack offset; defeats fingerprinting via high-frequency `geteuid()` probes (a common technique in privilege-discovery shellcode).
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Uid (effective) is restricted to owner and CAP_SYS_ADMIN; `geteuid()` of self always succeeds.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, `geteuid()` returns the user-ns-mapped EUID; kuid never leaks across the chroot boundary.
- **GRKERNSEC_AUDIT_GROUP on setuid** — grsec audits `setuid/seteuid/setresuid` transitions for audited groups; subsequent `geteuid()` reflecting the post-set value is the verification point.
- **CAP_SETUID strict** — grsec policy can globally deny CAP_SETUID; if denied, `geteuid()` never changes from initial value (no setuid possible).
- **file-cap-strip + securebits** — file capabilities are stripped on certain setuid transitions per kernel rules (SECBIT_KEEP_CAPS controls); `geteuid()` reflects the post-transition EUID without leaking pre-transition state.
- **no_new_privs across setuid** — when NNP is set, setuid execve does NOT elevate EUID even from a setuid binary; `geteuid()` will reflect the (unelevated) value of the calling process post-exec. NNP is honored on the geteuid read path implicitly (no special handling, but the cred.euid would not have been raised).
- **KEEPCAPS preservation behavior** — `PR_SET_KEEPCAPS` does not affect EUID; only capabilities are preserved across setuid. `geteuid()` returns whatever setuid set.
- **User-namespace map enforcement** — grsec prevents privileged uid_map establishment by unprivileged user-ns; `geteuid()` reflects only legitimate maps.
- **OVERFLOWUID hardening** — `/proc/sys/kernel/overflowuid` is sealed against unprivileged downgrades; `geteuid()` returning OVERFLOWUID is unambiguous "nobody".
- **No LSM hook** — grsec adds optional audit but never blocks `geteuid()`.

