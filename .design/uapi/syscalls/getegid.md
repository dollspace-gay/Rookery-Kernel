# Tier-5 syscall: getegid(2) — syscall 108

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE0(getegid))
  - include/linux/cred.h (struct cred.egid)
  - include/linux/uidgid.h (kgid_t, from_kgid_munged)
  - include/linux/user_namespace.h (current_user_ns)
  - arch/x86/entry/syscalls/syscall_64.tbl (108  common  getegid)
-->

## Summary

`getegid(2)` returns the calling task's **effective group ID** (EGID) as seen from the caller's user namespace. The EGID is the GID the kernel uses for DAC permission checks on group-ownership decisions (file access, IPC, fcntl-locks, signals where group-matching applies). Distinct from `getgid(2)` (real GID — bookkeeping identity, untouched by setgid-execve). The EGID is what changes when a setgid binary is executed.

Critical for: file-access decision (kernel consults EGID, EUID, fsgid, fsuid, supplementary groups via `inode_permission`); setgid-binary self-introspection paired with `getgid(2)`; sudo/su privilege gating; container init group-context; IPC sysv-semaphore/msqid/shmid group access; BSD-style group inheritance bit on directories (`S_ISGID`).

## Signature

```c
gid_t getegid(void);
```

## Parameters

(none — `SYSCALL_DEFINE0(getegid)`)

## Return value

| Value | Meaning |
|---|---|
| `gid_t` | The calling task's EGID mapped into `current_user_ns()`. |

`getegid()` cannot fail. POSIX-mandated: never returns `-1`, never sets `errno`.

Return value `(gid_t)-1` (i.e. `0xFFFFFFFF` = `OVERFLOWGID`) indicates the kernel EGID is NOT mapped in the caller's user namespace (a "nogroup" placeholder, `/proc/sys/kernel/overflowgid` — default 65534).

## Errors

(none)

## ABI surface

```text
__NR_getegid (x86_64)    = 108
__NR_getegid (i386)      = 50      /* 16-bit gid_t variant */
__NR_getegid32 (i386)    = 202     /* 32-bit gid_t — preferred */
__NR_getegid (arm64)     = 177     /* generic-syscall */
__NR_getegid (generic)   = 177

typedef __kernel_gid32_t  gid_t;  /* unsigned int — 32-bit */

OVERFLOWGID         = 65534         /* "nogroup" — gid not mapped in current_user_ns */
DEFAULT_OVERFLOWGID = 65534

struct cred {
    ...
    kgid_t          gid;            /* RGID */
    kgid_t          egid;           /* EGID — this syscall returns this */
    kgid_t          sgid;           /* SGID */
    kgid_t          fsgid;          /* FSGID (used for file-system perms) */
    ...
};
```

## Compatibility contract

REQ-1: Syscall number is **108** on x86_64; **177** on arm64 / generic. ABI-stable.

REQ-2: Returns `from_kgid_munged(current_user_ns(), current->real_cred->egid)`. The "munged" form returns `OVERFLOWGID` for unmapped GIDs. POSIX requires a successful return value, so the munged form is mandatory.

REQ-3: The 16-bit i386 ABI (syscall 50) returns the low 16 bits and clamps to 65535 if the actual gid exceeds 65535. Modern userspace uses syscall 202 (`getegid32`) on i386. ABI-frozen.

REQ-4: After `execve(2)` of a setgid binary (mode 2755), the kernel sets `cred->egid = file_inode->i_gid` (mapped to current_user_ns) and `cred->sgid = egid`. `cred->gid` (RGID) is unchanged. Therefore `getgid() != getegid()` after setgid-execve.

REQ-5: `setegid(2)`, `setregid(2)`, `setresgid(2)`, `setgid(2)` (when called with CAP_SETGID) mutate `cred->egid`. Without CAP_SETGID, `setegid(g)` requires `g ∈ {RGID, EGID, SGID}` (POSIX-saved-set semantics).

REQ-6: `cred->fsgid` is normally equal to `cred->egid`; `setfsgid(2)` can desynchronize them. `getegid()` returns EGID, NOT fsgid. The kernel uses fsgid (not egid) for file-system permission checks via `inode_permission()` → `current_fsgid()`.

REQ-7: `current->real_cred` is sampled (not `current->cred`) when there is a temporary credential override via `override_creds()`. For ordinary syscalls these are the same `cred`.

REQ-8: Per-thread cred: each thread has its own `task_struct.real_cred`. NPTL synchronizes via setgid syscall hooking.

REQ-9: User-namespace mapping via `gid_map`: `getegid()` returns the namespace-relative gid_t, or OVERFLOWGID if not mapped.

REQ-10: `getegid()` is async-signal-safe and re-entrant. No locks held; cred is RCU-protected.

REQ-11: No LSM hook on the getegid path. EGID is process-public metadata.

REQ-12: `getegid()` does NOT change credentials, capabilities, securebits, or any task state. Pure read.

REQ-13: Concurrent `setegid()` / `setgid()` from another thread: RCU-COW on cred; `getegid()` sees either the pre- or post-update cred atomically.

REQ-14: Returns NEVER < 0 (gid_t is unsigned). `0xFFFFFFFF` = OVERFLOWGID is a valid "nogroup" placeholder, not an error.

REQ-15: Supplementary groups (`group_info`) NOT consulted; only `cred->egid`.

## Acceptance Criteria

- [ ] AC-1: Process started with primary gid=1000 (no setgid): `getegid()` returns 1000.
- [ ] AC-2: Root-egid process: `getegid()` returns 0.
- [ ] AC-3: After `setegid(1001)` from CAP_SETGID-holder: `getegid()` returns 1001.
- [ ] AC-4: Setgid binary executed by gid=1000 (binary group=root, mode 2755): `getgid() == 1000`, `getegid() == 0`.
- [ ] AC-5: After `setresgid(-1, 2000, -1)`: `getegid() == 2000`, `getgid()` unchanged.
- [ ] AC-6: Unprivileged `setegid(other_gid)` where other_gid ∉ {RGID, EGID, SGID}: EPERM; `getegid()` unchanged.
- [ ] AC-7: In a user namespace with `0 1000 1` gid_map: host egid 1000 inside-ns `getegid()` returns 0.
- [ ] AC-8: Task whose kegid is NOT in the active user-ns map: `getegid() == 65534` (OVERFLOWGID).
- [ ] AC-9: `getegid()` consistent with `/proc/self/status:Gid` (second field).
- [ ] AC-10: `setfsgid()` does NOT change `getegid()` return value.
- [ ] AC-11: Concurrent `setegid()` from sibling thread: `getegid()` observes pre- or post-state atomically.
- [ ] AC-12: i386 syscall 50 returns clamped 16-bit; syscall 202 returns full 32-bit.

## Architecture

```rust
#[syscall(nr = 108, abi = "sysv")]
pub fn sys_getegid() -> gid_t {
    Getegid::do_getegid()
}
```

`Getegid::do_getegid() -> gid_t`:
1. let task = current();
2. /* RCU-read of real_cred */
3. let gid = rcu::read(|| {
4.     let cred = task.real_cred;
5.     let user_ns = current_user_ns();
6.     GidMap::from_kgid_munged(user_ns, cred.egid)
7. });
8. /* Invariant: returns ≥ 0; OVERFLOWGID for unmapped */
9. gid

`GidMap::from_kgid_munged(ns, kgid) -> gid_t`:
1. for entry in ns.gid_map.extents() {
2.     if kgid.val ∈ [entry.lower_first, entry.lower_first + entry.count) {
3.         return entry.first + (kgid.val - entry.lower_first);
4.     }
5. }
6. return OVERFLOWGID;

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `unmapped_returns_overflow` | INVARIANT | per-getegid: kegid ∉ gid_map ⟹ return OVERFLOWGID. |
| `mapped_returns_translation` | INVARIANT | per-getegid: kegid ∈ gid_map ⟹ return correct mapped value. |
| `rcu_atomic_cred` | INVARIANT | per-getegid: concurrent setegid yields consistent cred snapshot. |
| `no_state_mutation` | INVARIANT | per-getegid: read-only; no cred/securebits/cap field written. |
| `unsigned_return` | INVARIANT | per-getegid: return value is gid_t (unsigned); never < 0. |
| `real_cred_used` | INVARIANT | per-getegid: real_cred, not cred. |
| `egid_field_read` | INVARIANT | per-getegid: cred.egid read, NOT cred.gid / cred.fsgid / cred.sgid. |

### Layer 2: TLA+

`kernel/getegid.tla`:
- States: per-task cred snapshots, user-namespace gid_map, setgid-execve transition.
- Properties:
  - `safety_overflow_for_unmapped` — kegid not in gid_map ⟹ OVERFLOWGID.
  - `safety_setgid_execve_changes_egid_not_gid` — execve(setgid binary) sets egid=file_gid, gid unchanged.
  - `safety_no_mutation` — getegid never mutates state.
  - `safety_egid_distinct_from_fsgid` — setfsgid does not affect getegid.
  - `liveness_returns` — getegid terminates in O(gid_map.extents) ≤ O(5).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getegid` post: ret == from_kgid_munged(current_user_ns, current.real_cred.egid) | `Getegid::do_getegid` |
| `from_kgid_munged` post: returns OVERFLOWGID iff no mapping | `GidMap::from_kgid_munged` |
| `cred` invariant: refcount ≥ 1 during RCU read | `Cred::rcu_read` |

### Layer 4: Verus / Creusot functional

Per-`getegid(2)` man-page equivalence. LTP `getegid01..getegid02` pass. POSIX.1-2008 `getegid` semantics verified. Setgid-execve invariants verified jointly with `execve(2)` Tier-5 doc.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getegid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-cred-read race with concurrent setegid.
- **Per-overflow-munged for unmapped** — defense against per-namespace-leak.
- **Per-real_cred used (not cred)** — defense against per-override_creds() confusion bug pattern.
- **Per-readonly semantics** — defense against per-mutation-via-getegid bug pattern.
- **Per-egid distinct from fsgid** — defense against per-conflation with file-system gid.
- **Per-32-bit syscall (108) preferred over 16-bit (50)** — defense against per-gid-truncation legacy bug on i386.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at getegid entry** — randomizes kernel stack offset on syscall entry; defeats stack-layout fingerprinting via high-frequency `getegid()` probes.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Gid line (second field = EGID) is restricted to owning user and CAP_SYS_ADMIN; `getegid()` of self always succeeds. Cross-process EGID enumeration via `/proc/*/status` is gated by the grsec group policy, preventing setgid-binary discovery reconnaissance.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — `getegid()` inside a grsec chroot returns the user-ns-mapped EGID; the kegid is never leaked across the chroot boundary.
- **GRKERNSEC_AUDIT_GROUP** — repeated `getegid()` calls from audited groups are logged; the divergence `getgid() != getegid()` (setgid-binary indicator) is flagged for security review.
- **CAP_SETGID strict** — grsec policy can globally deny CAP_SETGID; `getegid()` continues to return the pre-existing EGID. Combined with file-cap-strip, setgid-binary execve cannot raise EGID beyond the policy.
- **file-cap-strip + securebits SECBIT_KEEP_CAPS** — when grsec strips file-caps on filesystem-mount and SECBIT_NO_SETUID_FIXUP is set, setgid-execve does NOT change EGID; `getegid()` returns the pre-execve value. Defense in depth against fs-tampering with setgid binaries.
- **no_new_privs across setuid/setgid** — when `PR_SET_NO_NEW_PRIVS` is set, setgid-execve is neutralized: EGID is not raised, `getegid()` is unchanged. Grsec enforces NNP-by-default for unprivileged user-ns.
- **Ambient / inheritable cap evolution neutral** — `getegid()` does not touch cap-bset, inheritable, or ambient sets; only cred.egid is read.
- **OVERFLOWGID hardening** — grsec ensures `/proc/sys/kernel/overflowgid` cannot be lowered below 65534; prevents masquerading as gid=0 or a sudoers gid.
- **User-namespace gid_map enforcement** — grsec validates gid_map cannot create overlapping or escalating ranges; `getegid()` reflects the enforced map.
- **No LSM hook** — getegid has no security hook; grsec adds optional audit but never blocks.
- **Per-thread cred consistency** — grsec audits per-thread egid drift caused by NPTL-sync edge cases.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `getgid(2)` (Tier-5 separate doc — real GID).
- `getresgid(2)` (Tier-5 separate doc — returns real, effective, saved GID).
- `setegid(2)` / `setgid(2)` / `setresgid(2)` (Tier-5 separate docs).
- `setfsgid(2)` (Tier-5 separate doc — file-system gid).
- User-namespace gid_map semantics (Tier-3 in `kernel/user_namespace.md`).
- File capabilities and setgid-execve cred transition (Tier-3 in `security/commoncap.md` and `fs/exec.md`).
- Implementation code.
