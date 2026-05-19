---
title: "Tier-5 syscall: setfsuid(2) — syscall 122"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setfsuid(2)` sets the **filesystem UID** (FSUID) of the calling process — the UID used for permission checks on filesystem operations (open, stat, chown, mkdir, unlink). Distinct from EUID, which historically conflated process-identity and filesystem-identity. FSUID was introduced for the Linux NFS server (`nfsd`) so an in-kernel server thread could impersonate a remote client's UID for permission checks without giving up its CAP_SETUID and other capabilities tied to EUID==0.

Unique semantic: setfsuid **never fails to return** — it always returns the **previous** FSUID, regardless of whether the requested change was permitted. To detect failure, the caller calls setfsuid twice: first with the desired UID, then with `-1` (or any sentinel) and checks if the returned previous matches the requested. This API is unusual and pre-dates POSIX standardization (not in POSIX).

Critically: setfsuid does **NOT** strip capabilities the way setuid does, even when transitioning the FSUID from 0 to nonzero. The intent is permission-check delegation, not full privilege drop.

Critical for: nfsd, smbd (Samba), 9p server, FUSE in some configurations, glibc internal helpers, container init that wants per-file impersonation without giving up CAP_NET_ADMIN/CAP_SYS_ADMIN.

### Acceptance Criteria

- [ ] AC-1: `setfsuid(current_fsuid)` returns current_fsuid; no state change.
- [ ] AC-2: Unprivileged caller `setfsuid(EUID_value)` returns previous FSUID and sets fsuid := EUID_value.
- [ ] AC-3: Unprivileged caller `setfsuid(unrelated_uid)` returns previous FSUID; FSUID unchanged silently.
- [ ] AC-4: CAP_SETUID caller `setfsuid(any_uid)` succeeds (no silent rejection).
- [ ] AC-5: After setfsuid: file-permission checks (open, stat) use new FSUID.
- [ ] AC-6: After setfsuid: RUID/EUID/SUID unchanged.
- [ ] AC-7: After setfsuid 0→nonzero: cap_effective preserved (no capability drop).
- [ ] AC-8: Detection pattern: setfsuid(target); setfsuid(-1); second return == target ⟹ change succeeded.
- [ ] AC-9: setfsuid with unmappable uid in current_user_ns: silently rejected; returns previous.
- [ ] AC-10: After execve of setuid root binary: FSUID == 0 (matches new EUID).
- [ ] AC-11: Per-thread semantics: sibling thread's FSUID unaffected by current thread's setfsuid.
- [ ] AC-12: Concurrent setfsuid: each thread observes own pre/post atomically.

### Architecture

```rust
#[syscall(nr = 122, abi = "sysv")]
pub fn sys_setfsuid(fsuid: uid_t) -> uid_t {
    Setfsuid::do_setfsuid(fsuid)
}
```

`Setfsuid::do_setfsuid(fsuid_arg) -> uid_t`:
1. let user_ns = current_user_ns();
2. let old_cred = current.cred;
3. let old_fsuid = from_kuid_munged(user_ns, old_cred.fsuid);
4. /* Try to compute new kfsuid */
5. let new_kfsuid = match make_kuid(user_ns, fsuid_arg) {
6.     Some(k) => k,
7.     None    => return old_fsuid,    /* invalid: silently return old */
8. };
9. /* Permission check */
10. let unpriv = !ns_capable(user_ns, CAP_SETUID);
11. if unpriv {
12.     if new_kfsuid != old_cred.uid
13.         && new_kfsuid != old_cred.euid
14.         && new_kfsuid != old_cred.suid
15.         && new_kfsuid != old_cred.fsuid {
16.         return old_fsuid;            /* not in set: silently return old */
17.     }
18. }
19. /* Apply via prepare/commit */
20. let new = Cred::prepare();
21. new.fsuid = new_kfsuid;
22. /* LSM hook */
23. if security_task_fix_setuid(&new, &old_cred, LSM_SETID_FS).is_err() {
24.     abort_creds(new);
25.     return old_fsuid;                /* LSM denied: silently return old */
26. }
27. commit_creds(new);
28. /* Audit */
29. audit_log_set_fsuid(old_cred.fsuid, new_kfsuid);
30. old_fsuid

### Out of Scope

- `setfsgid(2)` (Tier-5 separate doc — gid counterpart).
- `setuid(2)` / `setreuid(2)` / `setresuid(2)` (Tier-5 separate docs — full RUID/EUID/SUID).
- NFS server impersonation logic (Tier-3 in `fs/nfsd/server.md`).
- User-namespace uid mapping (Tier-3 in `kernel/user_namespace.md`).
- LSM `task_fix_setuid` hook (Tier-3 in `security/lsm.md`).
- Implementation code.

### signature

```c
int setfsuid(uid_t fsuid);
```

(Note: declared as `int` but semantically returns `uid_t` of previous fsuid.)

### parameters

| name  | type    | constraints                                                                              | errno-on-bad |
|-------|---------|------------------------------------------------------------------------------------------|--------------|
| fsuid | `uid_t` | Target FSUID; must be mappable in current_user_ns AND in `{RUID, EUID, SUID, FSUID}` for unprivileged caller, OR any value for CAP_SETUID caller. | (no errno; silent rejection) |

### return value

| Value | Meaning |
|---|---|
| `uid_t` (always) | The **previous** FSUID, mapped into caller's user-ns. ALWAYS returned, regardless of whether the change took effect. |

`setfsuid()` **cannot signal failure via return value**. To detect rejection: call once with desired uid, then call again with sentinel; compare second call's return to the first call's argument.

### errors

(no errno is set; rejection is silent)

### abi surface

```text
__NR_setfsuid (x86_64)   = 122
__NR_setfsuid (i386)     = 138      /* 16-bit uid */
__NR_setfsuid32 (i386)   = 215      /* 32-bit uid */
__NR_setfsuid (arm64)    = 151      /* generic */
__NR_setfsuid (generic)  = 151

struct cred {
    kuid_t  uid; kuid_t euid; kuid_t suid;
    kuid_t  fsuid;   /* filesystem UID */
    /* ... */
};
```

### compatibility contract

REQ-1: Syscall number is **122** on x86_64; **151** on arm64 / generic. The 16-bit i386 variant is syscall 138; 32-bit is 215.

REQ-2: The syscall ALWAYS returns the previous (pre-call) FSUID in the caller's user-namespace mapping via `from_kuid_munged(current_user_ns(), old.fsuid)`. Returns OVERFLOWUID if the old fsuid is not mapped.

REQ-3: Permission to change: unprivileged caller may set FSUID only to a value in `{RUID, EUID, SUID, FSUID}`. CAP_SETUID caller may set to any mappable uid.

REQ-4: If permission check fails OR `make_kuid` returns invalid: NO state change occurs; the previous FSUID is still returned. NO errno is set.

REQ-5: setfsuid does NOT update RUID, EUID, or SUID. Pure FSUID mutation.

REQ-6: setfsuid does NOT strip capabilities. Even transitioning FSUID 0 → nonzero leaves cap_effective intact. This is intentional (the NFS-server use case).

REQ-7: LSM hook `security_task_fix_setuid(new, old, LSM_SETID_FS)` runs before commit_creds. If the LSM denies: change suppressed, previous returned silently (LSM denial is not directly visible to caller via errno — same silent-rejection contract).

REQ-8: All threads share signal_struct but each has its own cred. NPTL does NOT synchronize setfsuid across threads (unlike setuid); setfsuid is per-thread by design.

REQ-9: After execve of a setuid binary: FSUID resets to match new EUID (per setuid-binary cred replacement). After non-setuid execve: FSUID preserved.

REQ-10: Audit: setfsuid is auditable; logged with old → new FSUID transition.

REQ-11: setfsuid is async-signal-unsafe (cred allocation).

REQ-12: The "no errno" API contract is preserved for ABI: glibc wrapper does not set errno; userspace must use the return-value-comparison pattern.

REQ-13: Concurrent setfsuid from another thread in same TGID: each operates on its own cred independently. Per-thread semantics are explicit.

REQ-14: User-namespace: fsuid argument interpreted in current_user_ns().

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_returns_old_fsuid` | INVARIANT | per-setfsuid: ret == from_kuid_munged(ns, old_cred.fsuid). |
| `no_errno_set` | INVARIANT | per-setfsuid: errno never modified. |
| `unprivileged_restricted_to_set` | INVARIANT | per-setfsuid: unprivileged ⟹ change only if target ∈ {RUID, EUID, SUID, FSUID}. |
| `cap_preserved_on_zero_transition` | INVARIANT | per-setfsuid: FSUID 0→nonzero ⟹ caps unchanged. |
| `other_creds_invariant` | INVARIANT | per-setfsuid: RUID, EUID, SUID, all gid fields unchanged. |
| `silent_rejection` | INVARIANT | per-setfsuid: invalid uid or perm denial ⟹ no state change AND no error signal. |

### Layer 2: TLA+

`kernel/setfsuid.tla`:
- States: per-task (RUID, EUID, SUID, FSUID, caps).
- Properties:
  - `safety_return_old` — setfsuid always returns prior fsuid (mapped or OVERFLOWUID).
  - `safety_permission_check` — unprivileged calls restricted to UID set.
  - `safety_cap_preservation` — caps not stripped on FSUID transition.
  - `safety_orthogonal_to_other_uids` — RUID/EUID/SUID untouched.
  - `liveness_terminates` — setfsuid completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setfsuid` post: ret == from_kuid_munged(user_ns, old.fsuid) | `Setfsuid::do_setfsuid` |
| `do_setfsuid` post (success): new.fsuid == requested; other fields unchanged | `Setfsuid::do_setfsuid` |
| `do_setfsuid` post (silent reject): no cred mutation | `Setfsuid::do_setfsuid` |
| `do_setfsuid` post: cap_effective == old.cap_effective | `Setfsuid::do_setfsuid` |

### Layer 4: Verus / Creusot functional

Per-`setfsuid(2)` man-page equivalence. LTP `setfsuid01..setfsuid04` pass. NFS-server impersonation idiom verified via integration.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setfsuid(2)` reinforcement:

- **Per-cred-prepare/commit atomicity** — defense against per-torn-cred race.
- **Per-unprivileged-target-set-restriction** — defense against per-privilege-escalation via foreign-uid FSUID set.
- **Per-CAP_SETUID gate** — defense against per-capability-bypass.
- **Per-cap-preservation explicit** — defense against per-NFS-server-loss-of-caps regression (cap drop would break nfsd).
- **Per-silent-rejection** — defense against per-permitted-set-enumeration via differential errno (no errno emitted, attacker cannot probe).
- **Per-LSM hook LSM_SETID_FS** — defense against per-LSM-bypass; LSM can audit/deny silently.
- **Per-RUID/EUID/SUID-orthogonal** — defense against per-cred-corruption.
- **Per-audit on cred-change** — defense against per-undetected-impersonation.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at setfsuid entry** — randomizes kernel stack offset.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status`'s Uid line FSuid field is restricted to owning user and CAP_SYS_ADMIN; setfsuid does not leak old/new via differential observation.
- **CAP_SETUID strict** — grsec policy can globally deny CAP_SETUID; setfsuid with cross-uid arguments will then be silently rejected (the unique no-errno contract is preserved).
- **GRKERNSEC_AUDIT_GROUP on cred-change** — setfsuid is **auditable** and logged for audited users with (old_fsuid → new_fsuid, caller_uid, pid). The auditable trail captures impersonation events even though the syscall silently returns no error.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — setfsuid inside a grsec chroot is constrained to uids mappable in chroot's user-ns.
- **setfsuid auditable but no cap-strip side effect** — explicitly: grsec respects the documented contract that setfsuid does NOT strip capabilities (NFS-server use case requires this), but ALL transitions are audit-logged so post-hoc forensic analysis can detect anomalous FSUID pivots.
- **PAX_NOEXEC** — orthogonal but applicable.
- **no_new_privs neutral** — NNP does not interact with setfsuid (which does not grant new privileges per se; it changes filesystem-check identity).
- **SECBIT_NO_SETUID_FIXUP** — securebits interact: setfsuid does not normally trigger cap-fixup; the bit is honored as a no-op.
- **GRKERNSEC_PROC_USER / GRKERNSEC_PROC_GROUP** — visibility of /proc/<pid> entries determined by FSUID/FSGID at access time; setfsuid changes can alter what /proc/* the caller can see.
- **Silent-rejection uniformity** — grsec ensures rejection (whether due to perm check, LSM, or chroot ns) is uniformly silent (no errno, return prior fsuid); prevents probing.
- **NFS-server delegation policy** — grsec policy can require CAP_SYS_ADMIN in addition to CAP_SETUID for setfsuid → uid==0 transitions, hardening nfsd against privilege re-acquisition exploits.

