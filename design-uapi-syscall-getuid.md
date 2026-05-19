---
title: "Tier-5 syscall: getuid(2) — syscall 102"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getuid(2)` returns the calling task's **real user ID** (RUID) as seen from the caller's user namespace. The kernel stores the RUID as a `kuid_t` (kernel UID — namespace-neutral) and translates it on read via `from_kuid_munged()` into the caller's user-namespace mapping. Distinct from `geteuid(2)` (effective UID, used for permission checks) and `getresuid(2)` (returns all three: real, effective, saved).

Critical for: setuid binary self-introspection (`if (getuid() != geteuid()) drop_privs()`), user-namespace container init, audit caller-identification, su/sudo bootstrap, `getlogin()` fallback, file-creation initial-owner determination via syscalls that consult cred.

### Acceptance Criteria

- [ ] AC-1: Process started by `uid=1000`: `getuid()` returns 1000.
- [ ] AC-2: Root process: `getuid()` returns 0.
- [ ] AC-3: After `setuid(1001)` from root: `getuid()` returns 1001.
- [ ] AC-4: Setuid binary executed by `uid=1000` (binary owner = root, mode 4755): `getuid() == 1000`, `geteuid() == 0`.
- [ ] AC-5: In a user namespace with `0 1000 1` uid_map, host uid 1000 inside-ns `getuid()` returns 0.
- [ ] AC-6: Task whose kuid is NOT in the active user-ns map: `getuid() == 65534` (OVERFLOWUID).
- [ ] AC-7: `getuid()` consistent with `/proc/self/status:Uid` (first field).
- [ ] AC-8: All threads of a process return the same `getuid()` (unless one thread issued `setuid` and we are in pre-NPTL-sync state — edge case).
- [ ] AC-9: `getuid()` never returns < 0 (uid_t is unsigned; trivially true).
- [ ] AC-10: Concurrent `setuid()` from sibling thread: `getuid()` observes pre- or post-state atomically.
- [ ] AC-11: After `execve()` of non-setuid binary: `getuid()` unchanged.
- [ ] AC-12: i386 syscall 24 returns clamped 16-bit; syscall 199 returns full 32-bit.

### Architecture

```rust
#[syscall(nr = 102, abi = "sysv")]
pub fn sys_getuid() -> uid_t {
    Getuid::do_getuid()
}
```

`Getuid::do_getuid() -> uid_t`:
1. let task = current();
2. /* RCU-read of real_cred */
3. let uid = rcu::read(|| {
4.     let cred = task.real_cred;          // *struct cred (RCU pointer)
5.     /* Map kuid_t → namespace-relative uid_t */
6.     let user_ns = current_user_ns();    // task.cred.user_ns
7.     UidMap::from_kuid_munged(user_ns, cred.uid)
8. });
9. /* Invariant: returns ≥ 0; OVERFLOWUID for unmapped */
10. uid

`UidMap::from_kuid_munged(ns, kuid) -> uid_t`:
1. /* Walk ns.uid_map entries; find one containing kuid.val */
2. for entry in ns.uid_map.extents() {
3.     if kuid.val ∈ [entry.lower_first, entry.lower_first + entry.count) {
4.         return entry.first + (kuid.val - entry.lower_first);
5.     }
6. }
7. /* No mapping found — return overflow */
8. return OVERFLOWUID;

`current_user_ns() -> *user_namespace`:
1. current().cred.user_ns

### Out of Scope

- `geteuid(2)` (Tier-5 separate doc — effective UID).
- `getresuid(2)` (Tier-5 separate doc — returns real, effective, saved UID).
- `setuid(2)` (Tier-5 separate doc).
- User-namespace uid_map / gid_map semantics (Tier-3 in `kernel/user_namespace.md`).
- File capabilities (Tier-3 in `security/commoncap.md`).
- Implementation code.

### signature

```c
uid_t getuid(void);
```

### parameters

(none — `SYSCALL_DEFINE0(getuid)`)

### return value

| Value | Meaning |
|---|---|
| `uid_t` | The calling task's RUID mapped into `current_user_ns()`. |

`getuid()` cannot fail. POSIX-mandated: never returns `-1`, never sets `errno`.

Return value `(uid_t)-1` (i.e. `0xFFFFFFFF` = `OVERFLOWUID`) indicates the kernel RUID is NOT mapped in the caller's user namespace (a "nobody" placeholder, `/proc/sys/kernel/overflowuid` — default 65534).

### errors

(none)

### abi surface

```text
__NR_getuid (x86_64)    = 102
__NR_getuid (i386)      = 24      /* 16-bit uid_t variant */
__NR_getuid32 (i386)    = 199     /* 32-bit uid_t — preferred */
__NR_getuid (arm64)     = 174     /* generic-syscall */
__NR_getuid (generic)   = 174

typedef __kernel_uid32_t  uid_t;  /* unsigned int — 32-bit */

OVERFLOWUID         = 65534         /* "nobody" — uid not mapped in current_user_ns */
DEFAULT_OVERFLOWUID = 65534

struct cred {
    ...
    kuid_t          uid;            /* RUID (kernel-namespace-neutral) */
    kuid_t          euid;           /* EUID */
    kuid_t          suid;           /* SUID */
    kuid_t          fsuid;          /* FSUID */
    ...
};

/* kuid_t is the kernel's namespace-neutral UID; map to user-ns via:
 *   uid_t = from_kuid_munged(ns, kuid)
 * which returns OVERFLOWUID if no mapping exists.
 */
```

### compatibility contract

REQ-1: Syscall number is **102** on x86_64; **174** on arm64 / generic. ABI-stable.

REQ-2: Returns `from_kuid_munged(current_user_ns(), current->real_cred->uid)`. The "munged" form returns `OVERFLOWUID` for unmapped UIDs (vs `from_kuid` which can return -1 / error). POSIX requires a successful return value, so the munged form is mandatory.

REQ-3: The 16-bit i386 ABI (syscall 24) returns the low 16 bits and clamps to 65535 if the actual uid exceeds 65535. Modern userspace uses syscall 199 (`getuid32`) on i386. ABI-frozen.

REQ-4: `setuid(2)`, `setreuid(2)`, `setresuid(2)` mutate `cred->uid`. `getuid()` reflects the post-mutation value.

REQ-5: After `execve(2)` of a setuid binary, `cred->uid` (RUID) is UNCHANGED; `euid`/`fsuid` change. Hence `getuid() != geteuid()` is the canonical setuid-binary detection.

REQ-6: `current->real_cred` is sampled (not `current->cred`) when there is a temporary credential override (rare — `override_creds()`). For ordinary syscalls these are the same `cred`.

REQ-7: All threads of a process share the same `signal_struct` but EACH THREAD HAS ITS OWN `cred` pointer (via `task_struct.real_cred`/`cred`). Linux historically treated cred as per-thread; POSIX-thread-implementation libraries (NPTL) synchronize cred across threads via `setuid` syscall hooking. Default behavior: per-thread.

REQ-8: User-namespace mapping: a task in user-namespace `U` with `cred->uid.val = 1000` may have its kuid mapped into `U` as `uid_t = 0` (root-in-ns). `getuid()` returns the **namespace-relative** uid_t.

REQ-9: If kernel kuid not present in `U`'s `uid_map`, `getuid()` returns `OVERFLOWUID` (65534 by default). The task is effectively `nobody` in `U`.

REQ-10: `getuid()` is async-signal-safe and re-entrant. No locks held; cred is RCU-protected.

REQ-11: No LSM hook on the getuid path. The returned UID is process-public metadata.

REQ-12: `getuid()` does NOT change credentials, capabilities, securebits, or any task state. Pure read.

REQ-13: Concurrent `setuid()` from another thread (same cred?): kernel uses RCU-COW on cred; `getuid()` sees either the pre- or post-update cred atomically.

REQ-14: Returns NEVER < 0 (uid_t is unsigned). Return value `0xFFFFFFFF` = `OVERFLOWUID` is a valid "nobody" placeholder, not an error.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `unmapped_returns_overflow` | INVARIANT | per-getuid: kuid ∉ uid_map ⟹ return OVERFLOWUID. |
| `mapped_returns_translation` | INVARIANT | per-getuid: kuid ∈ uid_map ⟹ return correct mapped value. |
| `rcu_atomic_cred` | INVARIANT | per-getuid: concurrent setuid yields consistent cred snapshot. |
| `no_state_mutation` | INVARIANT | per-getuid: read-only; no cred/securebits/cap field written. |
| `unsigned_return` | INVARIANT | per-getuid: return value is uid_t (unsigned); never < 0. |
| `real_cred_used` | INVARIANT | per-getuid: real_cred, not cred (handles override_creds correctly). |

### Layer 2: TLA+

`kernel/getuid.tla`:
- States: per-task cred snapshots, user-namespace uid_map.
- Properties:
  - `safety_overflow_for_unmapped` — kuid not in active uid_map ⟹ OVERFLOWUID.
  - `safety_translation_correct` — mapped uid is the unique value defined by the uid_map extent.
  - `safety_no_mutation` — getuid never mutates state.
  - `liveness_returns` — getuid terminates in O(uid_map.extents) ≤ O(5).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getuid` post: ret == from_kuid_munged(current_user_ns, current.real_cred.uid) | `Getuid::do_getuid` |
| `from_kuid_munged` post: returns OVERFLOWUID iff no mapping | `UidMap::from_kuid_munged` |
| `cred` invariant: refcount ≥ 1 during RCU read | `Cred::rcu_read` |

### Layer 4: Verus / Creusot functional

Per-`getuid(2)` man-page equivalence. LTP `getuid01..getuid03` pass. POSIX.1-2008 `getuid` semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getuid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-cred-read race with concurrent setuid.
- **Per-overflow-munged for unmapped** — defense against per-namespace-leak (kuid would otherwise leak host-kernel UID).
- **Per-real_cred used (not cred)** — defense against per-override_creds() confusion bug pattern.
- **Per-readonly semantics** — defense against per-mutation-via-getuid bug pattern.
- **Per-32-bit syscall (102) preferred over 16-bit (24)** — defense against per-uid-truncation legacy bug on i386.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at getuid entry** — randomizes kernel stack offset; defeats stack-layout fingerprinting via high-frequency `getuid()` probes.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Uid line is restricted to owning user and CAP_SYS_ADMIN; `getuid()` of self always succeeds. Sibling UID enumeration via `/proc/*/status` is gated separately.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — `getuid()` inside a grsec chroot returns the user-ns-mapped UID; the kuid is never leaked across the chroot boundary.
- **User-namespace map enforcement** — grsec validates that uid_map entries cannot create overlapping ranges or escalate (e.g., map host uid 0 inside an unprivileged user-ns); `getuid()` reflects the enforced (post-restriction) map.
- **CAP_SETUID strict** — grsec policy can globally deny CAP_SETUID; `getuid()` will continue to return the pre-existing RUID (no setuid is possible to mutate it).
- **GRKERNSEC_AUDIT_GROUP** — repeated `getuid()` calls from audited users are logged to detect uid-enumeration / privilege-discovery reconnaissance.
- **file-cap-strip + securebits neutral** — `getuid()` does not interact with file capabilities or securebits; it is a pure read of cred.uid.
- **no_new_privs neutral** — NNP affects setuid execve but not getuid.
- **KEEPCAPS neutral** — `PR_SET_KEEPCAPS` preserves capabilities across setuid but does not affect getuid.
- **OVERFLOWUID hardening** — grsec ensures `/proc/sys/kernel/overflowuid` cannot be lowered below 65534 by unprivileged users (prevents masquerading as a real uid).
- **No LSM hook** — getuid has no security hook; grsec adds optional audit but never blocks.

