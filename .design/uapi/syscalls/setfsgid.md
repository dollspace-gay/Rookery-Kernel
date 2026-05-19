# Tier-5 syscall: setfsgid(2) — syscall 123

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE1(setfsgid))
  - include/linux/cred.h (struct cred.fsgid)
  - kernel/cred.c (prepare_creds, commit_creds)
  - include/linux/uidgid.h (kgid_t, make_kgid)
  - security/security.c (security_task_fix_setgid LSM_SETID_FS)
  - arch/x86/entry/syscalls/syscall_64.tbl (123  common  setfsgid)
-->

## Summary

`setfsgid(2)` sets the **filesystem GID** (FSGID) of the calling process — the GID used for permission checks on filesystem operations (open, stat, chmod, mkdir, unlink — specifically the group-permission bits and group-ownership-on-create). Symmetric to `setfsuid(2)` but for group identity. Introduced alongside setfsuid for the in-kernel NFS server (`nfsd`) so server threads can impersonate a remote client's GID for permission checks without giving up other process-wide group state.

Unique semantic (same as setfsuid): the syscall **always returns the previous FSGID**, regardless of whether the requested change was permitted. No errno is set. To detect failure: call twice and compare. This API predates POSIX and is Linux-specific.

setfsgid does **NOT** affect supplementary groups (`cred.group_info`) — those are mutated only via `setgroups(2)`. setfsgid is per-thread, by design.

Critical for: nfsd, smbd, 9p server, FUSE in some configurations, container init's per-file GID impersonation, audit-trail correlation of FSGID-based file ownership.

## Signature

```c
int setfsgid(gid_t fsgid);
```

(Declared as `int`; semantically returns `gid_t` of previous fsgid.)

## Parameters

| name  | type    | constraints                                                                                       | errno-on-bad |
|-------|---------|---------------------------------------------------------------------------------------------------|--------------|
| fsgid | `gid_t` | Target FSGID; must be mappable in current_user_ns AND in `{RGID, EGID, SGID, FSGID}` for unprivileged caller, OR any value for CAP_SETGID caller. | (no errno; silent rejection) |

## Return value

| Value | Meaning |
|---|---|
| `gid_t` (always) | The **previous** FSGID, mapped into caller's user-ns. ALWAYS returned. |

`setfsgid()` **cannot signal failure via return value**.

## Errors

(no errno is set; rejection is silent)

## ABI surface

```text
__NR_setfsgid (x86_64)   = 123
__NR_setfsgid (i386)     = 139      /* 16-bit gid */
__NR_setfsgid32 (i386)   = 216      /* 32-bit gid */
__NR_setfsgid (arm64)    = 152      /* generic */
__NR_setfsgid (generic)  = 152

struct cred {
    kuid_t  uid; kuid_t euid; kuid_t suid; kuid_t fsuid;
    kgid_t  gid; kgid_t egid; kgid_t sgid;
    kgid_t  fsgid;   /* filesystem GID */
    struct group_info *group_info;  /* supplementary groups (NOT touched) */
    /* ... */
};
```

## Compatibility contract

REQ-1: Syscall number is **123** on x86_64; **152** on arm64 / generic. The 16-bit i386 variant is syscall 139; 32-bit is 216.

REQ-2: The syscall ALWAYS returns the previous (pre-call) FSGID in the caller's user-namespace mapping via `from_kgid_munged(current_user_ns(), old.fsgid)`. Returns OVERFLOWGID if the old fsgid is not mapped.

REQ-3: Permission to change: unprivileged caller may set FSGID only to a value in `{RGID, EGID, SGID, FSGID}`. CAP_SETGID caller may set to any mappable gid.

REQ-4: If permission check fails OR `make_kgid` returns invalid: NO state change; previous FSGID returned silently. NO errno set.

REQ-5: setfsgid does NOT update RGID, EGID, or SGID. Pure FSGID mutation.

REQ-6: setfsgid does NOT touch supplementary groups (`cred.group_info`).

REQ-7: setfsgid does NOT strip or modify capabilities (capabilities are UID-scoped).

REQ-8: LSM hook `security_task_fix_setgid(new, old, LSM_SETID_FS)` runs before commit_creds. LSM denial is silent (no errno).

REQ-9: All threads share signal_struct but each has its own cred. NPTL does NOT synchronize setfsgid across threads; per-thread by design.

REQ-10: After execve of a setgid binary: FSGID resets to match new EGID. After non-setgid execve: FSGID preserved.

REQ-11: Audit: setfsgid is auditable; logged with old → new FSGID transition.

REQ-12: setfsgid is async-signal-unsafe (cred allocation).

REQ-13: The "no errno" API contract preserved for ABI.

REQ-14: Concurrent setfsgid from another thread in same TGID: each operates on own cred. Per-thread semantics explicit.

REQ-15: User-namespace: fsgid argument interpreted in current_user_ns().

## Acceptance Criteria

- [ ] AC-1: `setfsgid(current_fsgid)` returns current_fsgid; no state change.
- [ ] AC-2: Unprivileged caller `setfsgid(EGID_value)` returns previous FSGID and sets fsgid := EGID_value.
- [ ] AC-3: Unprivileged caller `setfsgid(unrelated_gid)` returns previous FSGID; FSGID unchanged silently.
- [ ] AC-4: CAP_SETGID caller `setfsgid(any_gid)` succeeds.
- [ ] AC-5: After setfsgid: file group-permission checks use new FSGID.
- [ ] AC-6: After setfsgid: RGID/EGID/SGID unchanged.
- [ ] AC-7: After setfsgid: supplementary groups (group_info) unchanged.
- [ ] AC-8: After setfsgid: cap_effective unchanged.
- [ ] AC-9: Detection pattern: setfsgid(target); setfsgid(-1); second return == target ⟹ change succeeded.
- [ ] AC-10: setfsgid with unmappable gid: silently rejected; returns previous.
- [ ] AC-11: After execve of setgid binary: FSGID matches new EGID.
- [ ] AC-12: Per-thread semantics: sibling thread's FSGID unaffected.
- [ ] AC-13: File creation honors FSGID for group-ownership of new inode (per BSD vs SysV semantics depending on parent dir's setgid bit).

## Architecture

```rust
#[syscall(nr = 123, abi = "sysv")]
pub fn sys_setfsgid(fsgid: gid_t) -> gid_t {
    Setfsgid::do_setfsgid(fsgid)
}
```

`Setfsgid::do_setfsgid(fsgid_arg) -> gid_t`:
1. let user_ns = current_user_ns();
2. let old_cred = current.cred;
3. let old_fsgid = from_kgid_munged(user_ns, old_cred.fsgid);
4. /* Try to compute new kfsgid */
5. let new_kfsgid = match make_kgid(user_ns, fsgid_arg) {
6.     Some(k) => k,
7.     None    => return old_fsgid,    /* invalid: silently return old */
8. };
9. /* Permission check */
10. let unpriv = !ns_capable(user_ns, CAP_SETGID);
11. if unpriv {
12.     if new_kfsgid != old_cred.gid
13.         && new_kfsgid != old_cred.egid
14.         && new_kfsgid != old_cred.sgid
15.         && new_kfsgid != old_cred.fsgid {
16.         return old_fsgid;            /* not in set: silently return old */
17.     }
18. }
19. /* Apply via prepare/commit */
20. let new = Cred::prepare();
21. new.fsgid = new_kfsgid;
22. /* LSM hook */
23. if security_task_fix_setgid(&new, &old_cred, LSM_SETID_FS).is_err() {
24.     abort_creds(new);
25.     return old_fsgid;                /* LSM denied: silently return old */
26. }
27. commit_creds(new);
28. /* Audit */
29. audit_log_set_fsgid(old_cred.fsgid, new_kfsgid);
30. old_fsgid

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_returns_old_fsgid` | INVARIANT | per-setfsgid: ret == from_kgid_munged(ns, old_cred.fsgid). |
| `no_errno_set` | INVARIANT | per-setfsgid: errno never modified. |
| `unprivileged_restricted_to_set` | INVARIANT | per-setfsgid: unprivileged ⟹ change only if target ∈ {RGID, EGID, SGID, FSGID}. |
| `caps_unchanged` | INVARIANT | per-setfsgid: cap_effective, cap_permitted unchanged. |
| `supplementary_groups_unchanged` | INVARIANT | per-setfsgid: cred.group_info pointer unchanged. |
| `uid_fields_unchanged` | INVARIANT | per-setfsgid: all kuid_t fields unchanged. |
| `silent_rejection` | INVARIANT | per-setfsgid: invalid gid or perm denial ⟹ no state change AND no error signal. |

### Layer 2: TLA+

`kernel/setfsgid.tla`:
- States: per-task (RGID, EGID, SGID, FSGID, supplementary groups).
- Properties:
  - `safety_return_old` — setfsgid always returns prior fsgid.
  - `safety_permission_check` — unprivileged calls restricted to GID set.
  - `safety_orthogonal_to_other_gids` — RGID/EGID/SGID untouched.
  - `safety_supplementary_untouched` — group_info pointer unchanged.
  - `safety_caps_untouched` — capabilities unchanged.
  - `liveness_terminates` — setfsgid completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setfsgid` post: ret == from_kgid_munged(user_ns, old.fsgid) | `Setfsgid::do_setfsgid` |
| `do_setfsgid` post (success): new.fsgid == requested; other gid fields unchanged | `Setfsgid::do_setfsgid` |
| `do_setfsgid` post (silent reject): no cred mutation | `Setfsgid::do_setfsgid` |
| `do_setfsgid` post: cap_effective unchanged; group_info unchanged | `Setfsgid::do_setfsgid` |

### Layer 4: Verus / Creusot functional

Per-`setfsgid(2)` man-page equivalence. LTP `setfsgid01..setfsgid04` pass. NFS-server group-impersonation idiom verified via integration.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setfsgid(2)` reinforcement:

- **Per-cred-prepare/commit atomicity** — defense against per-torn-cred race.
- **Per-unprivileged-target-set-restriction** — defense against per-privilege-escalation via foreign-gid FSGID set.
- **Per-CAP_SETGID gate** — defense against per-capability-bypass.
- **Per-no-cap-touch** — defense against per-spurious-capability-modification (caps are uid-scoped, not gid-scoped).
- **Per-supplementary-groups-untouched** — defense against per-implicit-group-drop.
- **Per-silent-rejection** — defense against per-permitted-set-enumeration via differential errno.
- **Per-LSM hook LSM_SETID_FS** — defense against per-LSM-bypass.
- **Per-uid-fields-orthogonal** — defense against per-cred-corruption.
- **Per-audit on cred-change** — defense against per-undetected-impersonation.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at setfsgid entry** — randomizes kernel stack offset.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status`'s Gid line FSgid field is restricted to owning user and CAP_SYS_ADMIN; setfsgid does not leak via differential observation.
- **CAP_SETGID strict** — grsec policy can globally deny CAP_SETGID; setfsgid with cross-gid arguments will then be silently rejected (the unique no-errno contract is preserved).
- **GRKERNSEC_AUDIT_GROUP on cred-change** — setfsgid is **auditable** and logged for audited users with (old_fsgid → new_fsgid, caller_uid/gid, pid). Captures group-impersonation events.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — setfsgid inside a grsec chroot constrained to gids mappable in chroot's user-ns.
- **setfsgid auditable but no cap-strip side effect** — explicitly: grsec respects the documented contract that setfsgid does NOT modify capabilities and does NOT modify supplementary groups; ALL transitions are audit-logged for post-hoc forensic analysis.
- **PAX_NOEXEC** — orthogonal but applicable.
- **no_new_privs neutral** — NNP does not gate setfsgid.
- **SECBIT_* neutral** — securebits are UID-scoped; setfsgid does not interact.
- **GRKERNSEC_PROC_GID** — special grsec admin group enforced via policy; setfsgid cannot grant membership to gids that gate /proc access for unprivileged callers.
- **Silent-rejection uniformity** — grsec ensures rejection is uniformly silent regardless of cause (perm check, LSM, chroot ns); prevents probing.
- **GRKERNSEC_TPE_GID** — Trusted Path Execution gating by GID; setfsgid into a TPE-eligible gid is audited and may be denied per policy.
- **NFS-server delegation policy** — grsec policy can require additional capabilities (CAP_SYS_ADMIN) for setfsgid → gid==0 transitions, hardening nfsd against group-privilege re-acquisition.
- **File-creation gid policy** — grsec interacts with setfsgid via BSD vs SysV file-creation semantics: when a parent dir has setgid bit, newly-created file inherits dir's gid regardless of FSGID; grsec audits any deviation.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `setfsuid(2)` (Tier-5 separate doc — uid counterpart).
- `setgid(2)` / `setregid(2)` / `setresgid(2)` (Tier-5 separate docs — full RGID/EGID/SGID).
- `setgroups(2)` (Tier-5 separate doc — supplementary groups).
- NFS server group-impersonation logic (Tier-3 in `fs/nfsd/server.md`).
- User-namespace gid mapping (Tier-3 in `kernel/user_namespace.md`).
- LSM `task_fix_setgid` hook (Tier-3 in `security/lsm.md`).
- Implementation code.
