# Tier-5 syscall: getgid(2) — syscall 104

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE0(getgid))
  - include/linux/cred.h (struct cred.gid)
  - include/linux/uidgid.h (kgid_t, from_kgid_munged)
  - include/linux/user_namespace.h (current_user_ns)
  - arch/x86/entry/syscalls/syscall_64.tbl (104  common  getgid)
-->

## Summary

`getgid(2)` returns the calling task's **real group ID** (RGID) as seen from the caller's user namespace. The kernel stores the RGID as a `kgid_t` (kernel GID — namespace-neutral) and translates it on read via `from_kgid_munged()` into the caller's user-namespace mapping. Distinct from `getegid(2)` (effective GID, used for permission checks) and `getresgid(2)` (returns all three: real, effective, saved). Distinct from `getgroups(2)` (supplementary group list).

Critical for: setgid-binary self-introspection (`if (getgid() != getegid()) drop_privs()`), user-namespace container init, audit caller-identification, file-creation initial-group determination via syscalls that consult cred, BSD-style group-ownership inheritance for directories with `setgid` bit.

## Signature

```c
gid_t getgid(void);
```

## Parameters

(none — `SYSCALL_DEFINE0(getgid)`)

## Return value

| Value | Meaning |
|---|---|
| `gid_t` | The calling task's RGID mapped into `current_user_ns()`. |

`getgid()` cannot fail. POSIX-mandated: never returns `-1`, never sets `errno`.

Return value `(gid_t)-1` (i.e. `0xFFFFFFFF` = `OVERFLOWGID`) indicates the kernel RGID is NOT mapped in the caller's user namespace (a "nogroup" placeholder, `/proc/sys/kernel/overflowgid` — default 65534).

## Errors

(none)

## ABI surface

```text
__NR_getgid (x86_64)    = 104
__NR_getgid (i386)      = 47      /* 16-bit gid_t variant */
__NR_getgid32 (i386)    = 200     /* 32-bit gid_t — preferred */
__NR_getgid (arm64)     = 176     /* generic-syscall */
__NR_getgid (generic)   = 176

typedef __kernel_gid32_t  gid_t;  /* unsigned int — 32-bit */

OVERFLOWGID         = 65534         /* "nogroup" — gid not mapped in current_user_ns */
DEFAULT_OVERFLOWGID = 65534

struct cred {
    ...
    kgid_t          gid;            /* RGID (kernel-namespace-neutral) */
    kgid_t          egid;           /* EGID */
    kgid_t          sgid;           /* SGID */
    kgid_t          fsgid;          /* FSGID */
    struct group_info *group_info;  /* supplementary */
    ...
};

/* kgid_t is the kernel's namespace-neutral GID; map to user-ns via:
 *   gid_t = from_kgid_munged(ns, kgid)
 * which returns OVERFLOWGID if no mapping exists.
 */
```

## Compatibility contract

REQ-1: Syscall number is **104** on x86_64; **176** on arm64 / generic. ABI-stable.

REQ-2: Returns `from_kgid_munged(current_user_ns(), current->real_cred->gid)`. The "munged" form returns `OVERFLOWGID` for unmapped GIDs (vs `from_kgid` which can return -1 / error). POSIX requires a successful return value, so the munged form is mandatory.

REQ-3: The 16-bit i386 ABI (syscall 47) returns the low 16 bits and clamps to 65535 if the actual gid exceeds 65535. Modern userspace uses syscall 200 (`getgid32`) on i386. ABI-frozen.

REQ-4: `setgid(2)`, `setregid(2)`, `setresgid(2)` mutate `cred->gid`. `getgid()` reflects the post-mutation value.

REQ-5: After `execve(2)` of a setgid binary, `cred->gid` (RGID) is UNCHANGED; `egid`/`fsgid` change. Hence `getgid() != getegid()` is the canonical setgid-binary detection.

REQ-6: `current->real_cred` is sampled (not `current->cred`) when there is a temporary credential override (rare — `override_creds()`). For ordinary syscalls these are the same `cred`.

REQ-7: All threads of a process share the same `signal_struct` but EACH THREAD HAS ITS OWN `cred` pointer (via `task_struct.real_cred`/`cred`). NPTL synchronizes cred across threads via `setgid` syscall hooking. Default behavior: per-thread.

REQ-8: User-namespace mapping: a task in user-namespace `U` with `cred->gid.val = 1000` may have its kgid mapped into `U` as `gid_t = 0` (root-in-ns).

REQ-9: If kernel kgid not present in `U`'s `gid_map`, `getgid()` returns `OVERFLOWGID` (65534 by default). The task is effectively `nogroup` in `U`.

REQ-10: `getgid()` is async-signal-safe and re-entrant. No locks held; cred is RCU-protected.

REQ-11: No LSM hook on the getgid path. The returned GID is process-public metadata.

REQ-12: `getgid()` does NOT change credentials, capabilities, securebits, or any task state. Pure read.

REQ-13: Concurrent `setgid()` from another thread (same cred?): kernel uses RCU-COW on cred; `getgid()` sees either the pre- or post-update cred atomically.

REQ-14: Returns NEVER < 0 (gid_t is unsigned). Return value `0xFFFFFFFF` = `OVERFLOWGID` is a valid "nogroup" placeholder, not an error.

REQ-15: `getgid()` does NOT consult supplementary `group_info`. Only `cred->gid` is read.

## Acceptance Criteria

- [ ] AC-1: Process started with primary gid=1000: `getgid()` returns 1000.
- [ ] AC-2: Root-group process: `getgid()` returns 0.
- [ ] AC-3: After `setgid(1001)` from CAP_SETGID-holder: `getgid()` returns 1001.
- [ ] AC-4: Setgid binary executed by gid=1000 (binary group=root, mode 2755): `getgid() == 1000`, `getegid() == 0`.
- [ ] AC-5: In a user namespace with `0 1000 1` gid_map, host gid 1000 inside-ns `getgid()` returns 0.
- [ ] AC-6: Task whose kgid is NOT in the active user-ns map: `getgid() == 65534` (OVERFLOWGID).
- [ ] AC-7: `getgid()` consistent with `/proc/self/status:Gid` (first field).
- [ ] AC-8: All threads of a process return the same `getgid()` (post-NPTL-sync).
- [ ] AC-9: `getgid()` never returns < 0 (gid_t is unsigned; trivially true).
- [ ] AC-10: Concurrent `setgid()` from sibling thread: `getgid()` observes pre- or post-state atomically.
- [ ] AC-11: After `execve()` of non-setgid binary: `getgid()` unchanged.
- [ ] AC-12: i386 syscall 47 returns clamped 16-bit; syscall 200 returns full 32-bit.
- [ ] AC-13: Supplementary groups via `setgroups()` do NOT affect `getgid()`.

## Architecture

```rust
#[syscall(nr = 104, abi = "sysv")]
pub fn sys_getgid() -> gid_t {
    Getgid::do_getgid()
}
```

`Getgid::do_getgid() -> gid_t`:
1. let task = current();
2. /* RCU-read of real_cred */
3. let gid = rcu::read(|| {
4.     let cred = task.real_cred;          // *struct cred (RCU pointer)
5.     /* Map kgid_t → namespace-relative gid_t */
6.     let user_ns = current_user_ns();    // task.cred.user_ns
7.     GidMap::from_kgid_munged(user_ns, cred.gid)
8. });
9. /* Invariant: returns ≥ 0; OVERFLOWGID for unmapped */
10. gid

`GidMap::from_kgid_munged(ns, kgid) -> gid_t`:
1. /* Walk ns.gid_map entries; find one containing kgid.val */
2. for entry in ns.gid_map.extents() {
3.     if kgid.val ∈ [entry.lower_first, entry.lower_first + entry.count) {
4.         return entry.first + (kgid.val - entry.lower_first);
5.     }
6. }
7. /* No mapping found — return overflow */
8. return OVERFLOWGID;

`current_user_ns() -> *user_namespace`:
1. current().cred.user_ns

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `unmapped_returns_overflow` | INVARIANT | per-getgid: kgid ∉ gid_map ⟹ return OVERFLOWGID. |
| `mapped_returns_translation` | INVARIANT | per-getgid: kgid ∈ gid_map ⟹ return correct mapped value. |
| `rcu_atomic_cred` | INVARIANT | per-getgid: concurrent setgid yields consistent cred snapshot. |
| `no_state_mutation` | INVARIANT | per-getgid: read-only; no cred/securebits/cap field written. |
| `unsigned_return` | INVARIANT | per-getgid: return value is gid_t (unsigned); never < 0. |
| `real_cred_used` | INVARIANT | per-getgid: real_cred, not cred (handles override_creds correctly). |
| `supplementary_ignored` | INVARIANT | per-getgid: group_info never consulted. |

### Layer 2: TLA+

`kernel/getgid.tla`:
- States: per-task cred snapshots, user-namespace gid_map.
- Properties:
  - `safety_overflow_for_unmapped` — kgid not in active gid_map ⟹ OVERFLOWGID.
  - `safety_translation_correct` — mapped gid is the unique value defined by the gid_map extent.
  - `safety_no_mutation` — getgid never mutates state.
  - `liveness_returns` — getgid terminates in O(gid_map.extents) ≤ O(5).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getgid` post: ret == from_kgid_munged(current_user_ns, current.real_cred.gid) | `Getgid::do_getgid` |
| `from_kgid_munged` post: returns OVERFLOWGID iff no mapping | `GidMap::from_kgid_munged` |
| `cred` invariant: refcount ≥ 1 during RCU read | `Cred::rcu_read` |

### Layer 4: Verus / Creusot functional

Per-`getgid(2)` man-page equivalence. LTP `getgid01..getgid03` pass. POSIX.1-2008 `getgid` semantics verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getgid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-cred-read race with concurrent setgid.
- **Per-overflow-munged for unmapped** — defense against per-namespace-leak (kgid would otherwise leak host-kernel GID).
- **Per-real_cred used (not cred)** — defense against per-override_creds() confusion bug pattern.
- **Per-readonly semantics** — defense against per-mutation-via-getgid bug pattern.
- **Per-32-bit syscall (104) preferred over 16-bit (47)** — defense against per-gid-truncation legacy bug on i386.
- **Per-supplementary-ignored** — defense against per-conflation with getgroups.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at getgid entry** — randomizes kernel stack offset on every syscall entry; defeats stack-layout fingerprinting via high-frequency `getgid()` probes from unprivileged userspace.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Gid line is restricted to owning user and CAP_SYS_ADMIN; `getgid()` of self always succeeds, but cross-process gid enumeration via `/proc/*/status` is gated by the grsec group policy.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — `getgid()` inside a grsec chroot returns the user-ns-mapped GID; the kgid is never leaked across the chroot boundary, preventing host-namespace gid discovery.
- **User-namespace gid_map enforcement** — grsec validates that gid_map entries cannot create overlapping ranges or escalate (e.g., map host gid 0 inside an unprivileged user-ns); `getgid()` reflects the enforced (post-restriction) map.
- **CAP_SETGID strict** — grsec policy can globally deny CAP_SETGID; `getgid()` continues to return the pre-existing RGID (no setgid is possible to mutate it from unprivileged context).
- **GRKERNSEC_AUDIT_GROUP** — repeated `getgid()` calls from audited users / audited groups are logged; combined with `getuid()` patterns this detects privilege-discovery reconnaissance.
- **file-cap-strip + securebits neutral** — `getgid()` does not interact with file capabilities or securebits SECBIT_KEEP_CAPS; it is a pure read of cred.gid.
- **no_new_privs neutral** — NNP affects setuid/setgid execve but not getgid.
- **Ambient / inheritable cap evolution neutral** — `getgid()` does not touch the cap-bset, inheritable set, or ambient set; only cred.gid is read.
- **OVERFLOWGID hardening** — grsec ensures `/proc/sys/kernel/overflowgid` cannot be lowered below 65534 by unprivileged users (prevents masquerading as a real gid such as 0/root or a sudoers group).
- **No LSM hook** — getgid has no security hook; grsec adds optional audit but never blocks.
- **Per-thread cred consistency** — grsec audits per-thread cred drift; if NPTL sync misfires and threads see different gids, the divergence is logged.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `getegid(2)` (Tier-5 separate doc — effective GID).
- `getresgid(2)` (Tier-5 separate doc — returns real, effective, saved GID).
- `setgid(2)` (Tier-5 separate doc).
- `getgroups(2)` / `setgroups(2)` (Tier-5 separate docs — supplementary groups).
- User-namespace gid_map / uid_map semantics (Tier-3 in `kernel/user_namespace.md`).
- File capabilities (Tier-3 in `security/commoncap.md`).
- Implementation code.
