---
title: "Tier-5 syscall: setreuid(2) — syscall 113"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setreuid(2)` sets the **real UID** (RUID) and/or **effective UID** (EUID) of the calling process. A value of `-1` for either argument leaves that UID unchanged. The **saved set-user-ID** (SUID) is updated to match the new EUID under certain conditions (specifically: if the call changes RUID, or if EUID is set to a value different from the previous RUID). Older System V did not have SUID; setreuid is the BSD-derived API that pre-dates `setresuid(2)`.

Permission model: an unprivileged caller (no `CAP_SETUID`) may only set its RUID/EUID to values among `{ current RUID, current EUID, current SUID }`. A caller with `CAP_SETUID` may set RUID/EUID to any value.

Critical for: BSD-era setuid programs, su/sudo (with sudo preferring setresuid for full control), legacy daemons (lpd, sendmail), libc semantics requiring temporary privilege drop and re-acquire, Rookery's POSIX permission-test infrastructure, container init's UID drop.

### Acceptance Criteria

- [ ] AC-1: Unprivileged caller `setreuid(uid, -1)` where uid is current EUID or current SUID: succeeds; RUID updated; SUID updated to new EUID per REQ-4.
- [ ] AC-2: Unprivileged caller `setreuid(-1, uid)` where uid is current RUID, EUID, or SUID: succeeds; EUID updated; SUID maybe updated.
- [ ] AC-3: Unprivileged caller `setreuid(arbitrary, -1)`: returns -1, errno == EPERM.
- [ ] AC-4: Privileged (CAP_SETUID) caller `setreuid(1000, 2000)`: succeeds; RUID=1000, EUID=2000, SUID=2000 (REQ-4 applies because EUID != old RUID).
- [ ] AC-5: Both args == -1: no-op; returns 0; no state change.
- [ ] AC-6: setreuid(-1, -1) is a successful no-op.
- [ ] AC-7: EUID change to 0→nonzero drops capabilities unless KEEPCAPS / SECBIT_KEEP_CAPS.
- [ ] AC-8: FSUID auto-tracks new EUID.
- [ ] AC-9: setreuid with unmappable UID in current_user_ns: returns -1, errno == EINVAL.
- [ ] AC-10: After setreuid: getuid/geteuid/getresuid reflect new values.
- [ ] AC-11: RLIMIT_NPROC exceeded for new RUID (unprivileged): returns -1, errno == EAGAIN.
- [ ] AC-12: Failure case: no cred change observable post-failure.

### Architecture

```rust
#[syscall(nr = 113, abi = "sysv")]
pub fn sys_setreuid(ruid: uid_t, euid: uid_t) -> SyscallResult<i32> {
    Setreuid::do_setreuid(ruid, euid)
}
```

`Setreuid::do_setreuid(ruid_arg, euid_arg) -> Result<i32>`:
1. let user_ns = current_user_ns();
2. let new = Cred::prepare()?;
3. /* Translate args */
4. let new_ruid = if ruid_arg == UID_NONE { new.uid } else {
5.     make_kuid(user_ns, ruid_arg).ok_or(Errno::EINVAL)?
6. };
7. let new_euid = if euid_arg == UID_NONE { new.euid } else {
8.     make_kuid(user_ns, euid_arg).ok_or(Errno::EINVAL)?
9. };
10. /* Capability check */
11. let unpriv = !ns_capable(user_ns, CAP_SETUID);
12. if unpriv {
13.     if ruid_arg != UID_NONE && new_ruid != current.uid && new_ruid != current.euid {
14.         return Err(Errno::EPERM);
15.     }
16.     if euid_arg != UID_NONE && new_euid != current.uid && new_euid != current.euid && new_euid != current.suid {
17.         return Err(Errno::EPERM);
18.     }
19. }
20. /* RLIMIT_NPROC */
21. if ruid_arg != UID_NONE && new_ruid != current.uid {
22.     if rlimit_nproc_exceeded(new_ruid) && !capable(CAP_SYS_RESOURCE) {
23.         return Err(Errno::EAGAIN);
24.     }
25. }
26. /* Apply */
27. new.uid = new_ruid;
28. new.euid = new_euid;
29. /* SUID update per REQ-4 */
30. if ruid_arg != UID_NONE || (euid_arg != UID_NONE && new_euid != current.uid) {
31.     new.suid = new_euid;
32. }
33. /* FSUID follows EUID */
34. new.fsuid = new_euid;
35. /* Capability transition */
36. cap_emulate_setxuid(&mut new, &current);
37. /* LSM */
38. security_task_fix_setuid(&new, &current, LSM_SETID_RE)?;
39. commit_creds(new)?;
40. Ok(0)

### Out of Scope

- `setuid(2)` (Tier-5 separate doc — full-RUID/EUID/SUID semantics with root privileges).
- `setresuid(2)` (Tier-5 separate doc — explicit per-field setter).
- `setregid(2)` (Tier-5 separate doc — group ID counterpart).
- `setfsuid(2)` (Tier-5 separate doc — fsuid only).
- User-namespace uid mapping (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.

### signature

```c
int setreuid(uid_t ruid, uid_t euid);
```

`-1` is the sentinel: pass `(uid_t)-1` (0xFFFFFFFF) for "no change".

### parameters

| name | type    | constraints                                                                                                          | errno-on-bad |
|------|---------|----------------------------------------------------------------------------------------------------------------------|--------------|
| ruid | `uid_t` | `(uid_t)-1` = no change; else target RUID; must be in caller's user-ns and permitted by capability checks.            | `EINVAL`/`EPERM` |
| euid | `uid_t` | `(uid_t)-1` = no change; else target EUID; subject to same checks.                                                    | `EINVAL`/`EPERM` |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` | Error; `errno` set. Original RUID/EUID/SUID untouched on failure. |

### errors

| errno    | condition                                                                                                                |
|----------|--------------------------------------------------------------------------------------------------------------------------|
| `EAGAIN` | RLIMIT_NPROC would be exceeded for the new RUID (CHANGED — same as setuid). |
| `EINVAL` | One or both UIDs are invalid (not mappable in current_user_ns).                                                          |
| `EPERM`  | Unprivileged caller AND requested UID is not in `{RUID, EUID, SUID}`; or other capability check failure.                  |

### abi surface

```text
__NR_setreuid (x86_64)   = 113
__NR_setreuid (i386)     = 70       /* 16-bit uid */
__NR_setreuid32 (i386)   = 203      /* 32-bit uid */
__NR_setreuid (arm64)    = 145      /* generic */
__NR_setreuid (generic)  = 145

#define UID_NONE    ((uid_t)-1)      /* "no change" sentinel */

struct cred {
    kuid_t  uid;     /* RUID */
    kuid_t  euid;    /* EUID */
    kuid_t  suid;    /* SUID */
    kuid_t  fsuid;   /* FSUID */
    /* gid counterparts ... */
    kernel_cap_t cap_permitted;
    kernel_cap_t cap_effective;
    /* ... */
};
```

### compatibility contract

REQ-1: Syscall number is **113** on x86_64; **145** on arm64 / generic. The 16-bit i386 variant is syscall 70; 32-bit is 203.

REQ-2: For each of ruid, euid: if argument == `(uid_t)-1`, that value is unchanged; otherwise, the argument is converted to a kuid_t via `make_kuid(current_user_ns(), arg)`. Invalid (unmappable) → EINVAL.

REQ-3: Permission check: unprivileged caller may set RUID only to current RUID or current EUID; may set EUID only to current RUID, current EUID, or current SUID. Privileged caller (`CAP_SETUID` in current_user_ns) may set to any value.

REQ-4: SUID update rule (POSIX-compliant Linux semantics): if RUID is changed, OR if EUID is set to a value different from the previous RUID, then SUID := new EUID. Otherwise SUID unchanged. This is more conservative than setuid() (which always updates SUID when caller is root).

REQ-5: FSUID always follows EUID: any change to EUID updates FSUID to the same value, unless the FSUID was explicitly set previously (which setreuid does not differentiate from initial state).

REQ-6: Capability transitions: per file-cap / securebits rules, transitioning from euid==0 to euid!=0 drops capabilities (cap_effective cleared) unless `SECBIT_KEEP_CAPS` is set or `KEEPCAPS` was acquired via `prctl(PR_SET_KEEPCAPS)`.

REQ-7: All threads of the calling process share `signal_struct` but each has its own `cred`. POSIX requires per-process semantics; glibc NPTL implements this by signaling siblings (`tgkill`+`SIGCANCEL`) and replaying the syscall. Kernel `setreuid` operates per-thread.

REQ-8: After successful setreuid: `commit_creds(new)` is called, swapping the task's cred pointer atomically; LSM hook `cred_prepare` (in `prepare_creds`) and `task_fix_setuid` is invoked.

REQ-9: User-namespace: ruid/euid arguments are interpreted in `current_user_ns()`; the kuid_t produced may or may not be visible in parent ns.

REQ-10: RLIMIT_NPROC: if changing RUID would cause the new RUID's process count to exceed RLIMIT_NPROC (rlimit on new RUID), the call fails with EAGAIN. CAP_SYS_RESOURCE / CAP_SYS_ADMIN bypass this check.

REQ-11: Audit: the credential change is logged via `__audit_log_capset` / `audit_log_user_acct`.

REQ-12: setreuid is async-signal-unsafe (allocates new cred, takes refcounts, runs LSM hooks).

REQ-13: After execve of a non-setuid binary: cred is preserved including any setreuid changes. setuid binary execve resets EUID per the binary's owner.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `none_sentinel_no_change` | INVARIANT | per-setreuid: arg == -1 ⟹ that field unchanged. |
| `unprivileged_restricted_to_set` | INVARIANT | per-setreuid: unprivileged ⟹ target ∈ {RUID, EUID, SUID}. |
| `suid_update_rule` | INVARIANT | per-setreuid: SUID updated per REQ-4 conditions. |
| `fsuid_tracks_euid` | INVARIANT | per-setreuid: FSUID := new EUID. |
| `cap_drop_on_euid_zero_transition` | INVARIANT | per-setreuid: euid 0→nonzero ⟹ cap_effective cleared (unless KEEPCAPS). |
| `failure_no_mutation` | INVARIANT | per-setreuid: any error path leaves cred unchanged. |

### Layer 2: TLA+

`kernel/setreuid.tla`:
- States: per-task (RUID, EUID, SUID, FSUID, caps).
- Properties:
  - `safety_perm_check_correct` — unprivileged caller's target validated against set.
  - `safety_suid_rule` — SUID updated iff REQ-4 trigger conditions hold.
  - `safety_atomic_commit` — all-or-nothing cred update.
  - `liveness_terminates` — setreuid completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setreuid` pre: args validated | `Setreuid::do_setreuid` |
| `do_setreuid` post (success): cred matches new values + SUID rule | `Setreuid::do_setreuid` |
| `do_setreuid` post (failure): cred unchanged | `Setreuid::do_setreuid` |

### Layer 4: Verus / Creusot functional

Per-`setreuid(2)` man-page equivalence. LTP `setreuid01..setreuid07` pass. POSIX.1-2008 setreuid semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setreuid(2)` reinforcement:

- **Per-cred-prepare/commit atomicity** — defense against per-torn-cred race.
- **Per-unprivileged-target-set-restriction** — defense against per-privilege-escalation via foreign-uid set.
- **Per-SUID-update-rule** — defense against per-privilege-regain via subsequent setuid (SUID must be tracked correctly so re-elevation is intentional).
- **Per-CAP_SETUID gate** — defense against per-capability-bypass.
- **Per-RLIMIT_NPROC enforcement** — defense against per-fork-bomb via uid swap.
- **Per-LSM hook task_fix_setuid** — defense against per-LSM-bypass.
- **Per-FSUID-track-EUID** — defense against per-fsuid stale (would otherwise allow filesystem-access privilege regain).

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at setreuid entry** — randomizes kernel stack offset; defeats fingerprinting probes around credential transitions (high-value attacker target).
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status`'s Uid line (real/eff/saved/fs) is restricted to owning user and CAP_SYS_ADMIN; setreuid does not leak old vs new values.
- **CAP_SETUID strict** — grsec enforces strict CAP_SETUID: if policy denies CAP_SETUID for a role, setreuid with cross-uid arguments fails with EPERM regardless of saved-set membership.
- **GRKERNSEC_AUDIT_GROUP on cred-change** — every successful setreuid is logged with (old_uid, old_euid, old_suid) → (new_uid, new_euid, new_suid), uid trail, and caller pid/tgid; auditable trail for credential-pivot detection.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — setreuid inside a grsec chroot is constrained: target uids must be mappable within chroot's user-ns; host-ns uids never leak in.
- **PAX_NOEXEC / KERNEXEC** — orthogonal but applicable: setreuid does not allocate executable memory; cred mutation goes through audited path.
- **Saved-set strict re-elevation** — grsec hardens the SUID rule: if SUID is updated to EUID per REQ-4, the new SUID is permanently lower in policy mode (no re-elevation possible without CAP_SETUID), unless the role explicitly allows saved-set re-elevation.
- **GRKERNSEC_NO_SIMULT_CONNECT** — orthogonal.
- **no_new_privs strict honored** — under PR_SET_NO_NEW_PRIVS, setreuid that would grant elevated EUID via setuid-binary execve later is preempted (NNP transitively prevents the cred from being used to escalate).
- **SECBIT_NO_SETUID_FIXUP** — grsec honors securebits: if SECBIT_NO_SETUID_FIXUP is set, capability transitions on EUID change are suppressed; setreuid respects this.
- **KEEPCAPS gating** — grsec validates KEEPCAPS use; capability retention across setreuid is permitted only when SECBIT_KEEP_CAPS or PR_SET_KEEPCAPS is explicitly enabled by an audited path.
- **Uniform EPERM** — grsec ensures EPERM is returned uniformly for both "uid invalid" (in policy terms) and "uid not in saved-set"; prevents probing for permitted UIDs via differential errno.

