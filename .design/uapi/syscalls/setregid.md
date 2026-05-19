# Tier-5 syscall: setregid(2) — syscall 114

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE2(setregid))
  - include/linux/cred.h (struct cred.gid, .egid, .sgid)
  - kernel/cred.c (prepare_creds, commit_creds)
  - include/linux/uidgid.h (kgid_t, make_kgid)
  - security/commoncap.c (cap_capable for CAP_SETGID)
  - arch/x86/entry/syscalls/syscall_64.tbl (114  common  setregid)
-->

## Summary

`setregid(2)` sets the **real GID** (RGID) and/or **effective GID** (EGID) of the calling process. Symmetric to `setreuid(2)` but for group IDs. A value of `(gid_t)-1` for either argument leaves that GID unchanged. The **saved set-group-ID** (SGID) is updated to match the new EGID under the same conditions as setreuid's SUID rule (if RGID is changed, OR if EGID is set to a value different from the previous RGID).

Permission model: an unprivileged caller (no `CAP_SETGID`) may only set RGID/EGID to values among `{ current RGID, current EGID, current SGID }`. A caller with `CAP_SETGID` may set to any value.

Critical for: BSD-era setgid programs, group-permission daemons (sendmail's `mail` group, lpd, mlocate), tmp-directory creators (`/tmp` with sticky-gid), Rookery's POSIX permission tests, container init's GID drop, supplementary-group bookkeeping (setregid does NOT touch supplementary groups — `setgroups` does).

## Signature

```c
int setregid(gid_t rgid, gid_t egid);
```

`-1` is the sentinel: pass `(gid_t)-1` (0xFFFFFFFF) for "no change".

## Parameters

| name | type    | constraints                                                                                          | errno-on-bad |
|------|---------|------------------------------------------------------------------------------------------------------|--------------|
| rgid | `gid_t` | `(gid_t)-1` = no change; else target RGID; must be in caller's user-ns and permitted by checks.       | `EINVAL`/`EPERM` |
| egid | `gid_t` | `(gid_t)-1` = no change; else target EGID; subject to same checks.                                    | `EINVAL`/`EPERM` |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` | Error; `errno` set. Original RGID/EGID/SGID untouched on failure. |

## Errors

| errno    | condition                                                                       |
|----------|---------------------------------------------------------------------------------|
| `EINVAL` | One or both GIDs are invalid (not mappable in current_user_ns).                  |
| `EPERM`  | Unprivileged caller AND requested GID is not in `{RGID, EGID, SGID}`; or other capability check failure. |

(Note: setregid does NOT return EAGAIN — RLIMIT_NPROC is RUID-scoped, not RGID.)

## ABI surface

```text
__NR_setregid (x86_64)   = 114
__NR_setregid (i386)     = 71       /* 16-bit gid */
__NR_setregid32 (i386)   = 204      /* 32-bit gid */
__NR_setregid (arm64)    = 143      /* generic */
__NR_setregid (generic)  = 143

#define GID_NONE    ((gid_t)-1)      /* "no change" sentinel */

struct cred {
    kuid_t  uid; kuid_t euid; kuid_t suid; kuid_t fsuid;
    kgid_t  gid;     /* RGID */
    kgid_t  egid;    /* EGID */
    kgid_t  sgid;    /* SGID */
    kgid_t  fsgid;   /* FSGID */
    /* ... */
};
```

## Compatibility contract

REQ-1: Syscall number is **114** on x86_64; **143** on arm64 / generic. The 16-bit i386 variant is syscall 71; 32-bit is 204.

REQ-2: For each of rgid, egid: if argument == `(gid_t)-1`, that value is unchanged; otherwise, converted via `make_kgid(current_user_ns(), arg)`. Unmappable → EINVAL.

REQ-3: Permission check: unprivileged caller (no `CAP_SETGID`) may set RGID only to current RGID or current EGID; may set EGID only to current RGID, current EGID, or current SGID. Privileged caller may set to any value.

REQ-4: SGID update rule (POSIX): if RGID is changed, OR if EGID is set to a value different from the previous RGID, then SGID := new EGID. Otherwise SGID unchanged.

REQ-5: FSGID always follows EGID: any change to EGID updates FSGID to the same value, unless the FSGID was explicitly set previously.

REQ-6: Capability transitions: setregid does NOT itself drop capabilities (capabilities are uid-scoped, not gid-scoped). However, setregid does NOT regrant any dropped capabilities.

REQ-7: Supplementary groups (cred.group_info) are NOT touched by setregid. Use `setgroups(2)` to modify the supplementary group list.

REQ-8: All threads share signal_struct but each has its own `cred`. glibc NPTL synchronizes via `tgkill`+`SIGCANCEL` replay.

REQ-9: After successful setregid: `commit_creds(new)` swaps the task's cred atomically; LSM hook `task_fix_setgid` is invoked.

REQ-10: User-namespace: rgid/egid arguments are interpreted in `current_user_ns()`; produced kgid_t may not be visible in parent ns.

REQ-11: Audit: credential change is logged.

REQ-12: setregid is async-signal-unsafe.

REQ-13: After execve of a non-setgid binary: gid cred preserved. Setgid binary execve resets EGID per the binary's group.

## Acceptance Criteria

- [ ] AC-1: Unprivileged caller `setregid(gid, -1)` where gid is current EGID or current SGID: succeeds; RGID updated; SGID per REQ-4.
- [ ] AC-2: Unprivileged caller `setregid(-1, gid)` where gid is current RGID, EGID, or SGID: succeeds; EGID updated; SGID maybe updated.
- [ ] AC-3: Unprivileged caller `setregid(arbitrary, -1)`: returns -1, errno == EPERM.
- [ ] AC-4: Privileged (CAP_SETGID) caller `setregid(100, 200)`: succeeds; RGID=100, EGID=200, SGID=200.
- [ ] AC-5: Both args == -1: no-op; returns 0; no state change.
- [ ] AC-6: setregid(-1, -1) is a successful no-op.
- [ ] AC-7: setregid with unmappable GID in current_user_ns: returns -1, errno == EINVAL.
- [ ] AC-8: After setregid: getgid/getegid/getresgid reflect new values.
- [ ] AC-9: setregid does NOT change supplementary groups (verify via getgroups).
- [ ] AC-10: setregid does NOT change uid/euid/suid/fsuid.
- [ ] AC-11: FSGID auto-tracks new EGID.
- [ ] AC-12: Failure case: no cred change observable.

## Architecture

```rust
#[syscall(nr = 114, abi = "sysv")]
pub fn sys_setregid(rgid: gid_t, egid: gid_t) -> SyscallResult<i32> {
    Setregid::do_setregid(rgid, egid)
}
```

`Setregid::do_setregid(rgid_arg, egid_arg) -> Result<i32>`:
1. let user_ns = current_user_ns();
2. let new = Cred::prepare()?;
3. /* Translate args */
4. let new_rgid = if rgid_arg == GID_NONE { new.gid } else {
5.     make_kgid(user_ns, rgid_arg).ok_or(Errno::EINVAL)?
6. };
7. let new_egid = if egid_arg == GID_NONE { new.egid } else {
8.     make_kgid(user_ns, egid_arg).ok_or(Errno::EINVAL)?
9. };
10. /* Capability check */
11. let unpriv = !ns_capable(user_ns, CAP_SETGID);
12. if unpriv {
13.     if rgid_arg != GID_NONE && new_rgid != current.gid && new_rgid != current.egid {
14.         return Err(Errno::EPERM);
15.     }
16.     if egid_arg != GID_NONE && new_egid != current.gid && new_egid != current.egid && new_egid != current.sgid {
17.         return Err(Errno::EPERM);
18.     }
19. }
20. /* Apply */
21. new.gid = new_rgid;
22. new.egid = new_egid;
23. /* SGID update per REQ-4 */
24. if rgid_arg != GID_NONE || (egid_arg != GID_NONE && new_egid != current.gid) {
25.     new.sgid = new_egid;
26. }
27. /* FSGID follows EGID */
28. new.fsgid = new_egid;
29. /* LSM */
30. security_task_fix_setgid(&new, &current, LSM_SETID_RE)?;
31. commit_creds(new)?;
32. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `none_sentinel_no_change` | INVARIANT | per-setregid: arg == -1 ⟹ that field unchanged. |
| `unprivileged_restricted_to_set` | INVARIANT | per-setregid: unprivileged ⟹ target ∈ {RGID, EGID, SGID}. |
| `sgid_update_rule` | INVARIANT | per-setregid: SGID updated per REQ-4 conditions. |
| `fsgid_tracks_egid` | INVARIANT | per-setregid: FSGID := new EGID. |
| `supplementary_unchanged` | INVARIANT | per-setregid: cred.group_info unchanged. |
| `uid_fields_unchanged` | INVARIANT | per-setregid: uid/euid/suid/fsuid untouched. |
| `failure_no_mutation` | INVARIANT | per-setregid: any error path leaves cred unchanged. |

### Layer 2: TLA+

`kernel/setregid.tla`:
- States: per-task (RGID, EGID, SGID, FSGID).
- Properties:
  - `safety_perm_check_correct` — unprivileged caller's target validated.
  - `safety_sgid_rule` — SGID updated per REQ-4 trigger conditions.
  - `safety_atomic_commit` — all-or-nothing cred update.
  - `safety_uid_orthogonal` — UID fields never touched.
  - `liveness_terminates` — setregid completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setregid` pre: args validated | `Setregid::do_setregid` |
| `do_setregid` post (success): cred matches new values + SGID rule | `Setregid::do_setregid` |
| `do_setregid` post (failure): cred unchanged | `Setregid::do_setregid` |
| `do_setregid` post: uid/euid/suid/fsuid invariant | `Setregid::do_setregid` |

### Layer 4: Verus / Creusot functional

Per-`setregid(2)` man-page equivalence. LTP `setregid01..setregid04` pass. POSIX.1-2008 setregid semantics verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setregid(2)` reinforcement:

- **Per-cred-prepare/commit atomicity** — defense against per-torn-cred race.
- **Per-unprivileged-target-set-restriction** — defense against per-privilege-escalation via foreign-gid set.
- **Per-SGID-update-rule** — defense against per-privilege-regain.
- **Per-CAP_SETGID gate** — defense against per-capability-bypass.
- **Per-LSM hook task_fix_setgid** — defense against per-LSM-bypass.
- **Per-FSGID-track-EGID** — defense against per-fsgid stale.
- **Per-supplementary-groups-untouched** — defense against per-implicit-group-drop confusion (must use explicit setgroups).
- **Per-uid-orthogonal** — defense against per-cred-corruption (UID fields never modified).

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at setregid entry** — randomizes kernel stack offset; defeats fingerprinting around cred transitions.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status`'s Gid line is restricted; setregid does not leak via differential observation.
- **CAP_SETGID strict** — grsec enforces strict CAP_SETGID: if policy denies CAP_SETGID for a role, setregid with cross-gid arguments fails with EPERM regardless of saved-set membership.
- **GRKERNSEC_AUDIT_GROUP on cred-change** — every successful setregid logs (old_gid, old_egid, old_sgid) → (new_gid, new_egid, new_sgid) with caller pid/tgid/uid trail; auditable for group-pivot detection.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, setregid constrained to gids mappable in chroot's user-ns.
- **Saved-set strict re-elevation** — grsec hardens SGID rule: SGID updates are one-way under policy mode (no re-elevation without CAP_SETGID).
- **Supplementary-group policy** — grsec may enforce a fixed supplementary-group set per role; setregid does not bypass this (setgroups is the gated path).
- **GRKERNSEC_PROC_GID** — special "grsec admin" group enforced via policy; setregid cannot grant membership to this gid for unprivileged callers.
- **no_new_privs strict honored** — under PR_SET_NO_NEW_PRIVS, gid-elevation paths via setgid-binary execve are blocked; setregid's saved-set is not a workaround.
- **SECBIT_NO_SETUID_FIXUP / SECBIT_KEEP_CAPS** — these securebits are UID-scoped; setregid does not interact with them (capabilities are not gid-dependent).
- **Uniform EPERM** — grsec returns uniform EPERM regardless of "gid invalid" vs "gid not in saved-set"; prevents probing.
- **GRKERNSEC_TPE_GID** — Trusted Path Execution may be gated by GID; setregid into/out of TPE-eligible gids is logged and may be denied per policy.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `setgid(2)` (Tier-5 separate doc — full-RGID/EGID/SGID via root privileges).
- `setresgid(2)` (Tier-5 separate doc — explicit per-field).
- `setreuid(2)` (Tier-5 separate doc — uid counterpart).
- `setfsgid(2)` (Tier-5 separate doc — fsgid only).
- `setgroups(2)` (Tier-5 separate doc — supplementary groups).
- User-namespace gid mapping (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.
