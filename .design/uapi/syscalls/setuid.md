# Tier-5 syscall: setuid(2) — syscall 105

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE1(setuid, uid_t))
  - kernel/cred.c (prepare_creds, commit_creds, abort_creds)
  - security/commoncap.c (cap_task_fix_setuid)
  - include/linux/cred.h (struct cred)
  - include/linux/securebits.h (SECBIT_KEEP_CAPS, SECBIT_NO_SETUID_FIXUP)
  - arch/x86/entry/syscalls/syscall_64.tbl (105  common  setuid)
-->

## Summary

`setuid(2)` sets the calling task's user IDs to `uid`. Semantics depend on privilege:

- **Privileged caller** (holds `CAP_SETUID` in caller's user-ns OR EUID==0): sets `real`, `effective`, **and** `saved` UIDs all to `uid`. (POSIX `_POSIX_SAVED_IDS` — the only path that overwrites SUID.)
- **Unprivileged caller**: `uid` must be one of `{RUID, EUID, SUID}`; only the `effective` UID is changed (and `fsuid` follows `euid`). The real/saved UIDs are unchanged. Used to drop privileges in setuid binaries.

Critical side effects:
- **File-capability strip**: when the EUID transitions to/from zero, the kernel may strip the *permitted* and *effective* capability sets via `cap_task_fix_setuid()` unless `SECBIT_KEEP_CAPS` or `SECBIT_NO_SETUID_FIXUP` is set.
- **Dumpable cleared**: when EUID changes, `mm.dumpable` may be cleared (PR_SET_DUMPABLE = 0 / SUID_DUMP_DISABLE), disabling core dumps and `/proc/<pid>/mem` access by ordinary users.
- **`no_new_privs` propagation**: NNP is preserved across setuid; if NNP is set, no privilege can be acquired.

## Signature

```c
int setuid(uid_t uid);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `uid` | `uid_t` | in | New user ID (interpreted in caller's user namespace). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. Cred updated. |
| `-1` | Failure; `errno` set. No cred changes are committed. |

## Errors

| errno | When |
|---|---|
| `EAGAIN` | `uid` does not match any of {RUID, EUID, SUID} **and** the call would change RUID, but the new RUID's per-user RLIMIT_NPROC would be exceeded. |
| `EINVAL` | `uid` is not a valid value in the caller's user namespace (no mapping in `uid_map`). |
| `EPERM` | Caller lacks `CAP_SETUID` AND `uid` is not in {RUID, EUID, SUID}. |

## ABI surface

```text
__NR_setuid (x86_64)    = 105
__NR_setuid (i386)      = 23      /* 16-bit uid_t variant */
__NR_setuid32 (i386)    = 213     /* 32-bit uid_t — preferred */
__NR_setuid (arm64)     = 146     /* generic-syscall */
__NR_setuid (generic)   = 146

typedef __kernel_uid32_t  uid_t;  /* unsigned int — 32-bit */

/* securebits flags (PR_SET_SECUREBITS) */
SECBIT_NOROOT             = 1 << 0
SECBIT_NO_SETUID_FIXUP    = 1 << 2  /* suppress cap-strip on euid 0↔nonzero */
SECBIT_KEEP_CAPS          = 1 << 4  /* preserve caps across setuid */

/* dumpable values */
SUID_DUMP_DISABLE = 0   /* set by setuid when EUID changes (default) */
SUID_DUMP_USER    = 1   /* dumpable */
SUID_DUMP_ROOT    = 2   /* root-only dumpable */
```

## Compatibility contract

REQ-1: Syscall number is **105** on x86_64; **146** on arm64 / generic. ABI-stable.

REQ-2: `uid` is interpreted in `current_user_ns()`. The kernel converts to `kuid_t` via `make_kuid(current_user_ns(), uid)`. If the result is `INVALID_UID` (not in uid_map), return `EINVAL`.

REQ-3: Privileged path (`ns_capable(current_user_ns(), CAP_SETUID)`):
- new_cred.uid = new_cred.euid = new_cred.suid = new_cred.fsuid = kuid.
- Update RLIMIT_NPROC accounting (decrement old uid's count, increment new uid's count) atomically with RCU.

REQ-4: Unprivileged path: `kuid` must equal one of `{cred.uid, cred.euid, cred.suid}`. Sets only `new_cred.euid = new_cred.fsuid = kuid`. RGID/SUID unchanged. If `kuid ∉ {ruid, euid, suid}`, return `EPERM`.

REQ-5: After cred mutation, call `cap_task_fix_setuid(new, old, LSM_SETID_ID)` (the "ID" flavor = setuid):
- If `securebits & SECBIT_NO_SETUID_FIXUP`: no cap-strip.
- Else if old.euid == 0 ∧ new.euid != 0 (root-to-nonroot transition):
  - **clear `effective` and `permitted` capability sets** unless `SECBIT_KEEP_CAPS` is set, in which case only `effective` is cleared (permitted preserved).
- Else if old.euid != 0 ∧ new.euid == 0 (nonroot-to-root transition):
  - Restore `permitted` from `old.permitted` (capabilities are not raised beyond the original bounding set).
  - Set `effective = permitted` if `old.euid != 0` (re-grant on root).
- Else if old.uid == 0 ∧ new.uid != 0 (real-uid leaves root):
  - Clear `bounding`, `inheritable` (POSIX-draft "permanent privilege drop").

REQ-6: `commit_creds(new)`:
- Atomically swap `current->real_cred` and `current->cred` pointers (RCU-COW).
- Update per-user-struct RLIMIT_NPROC count.
- Free old cred via RCU.
- Set `current.signal.flags |= SIGNAL_CGROUP_CLOSE_CONTROL_FD` (notify cgroup).

REQ-7: If EUID changes value (old.euid != new.euid), set `mm.dumpable = SUID_DUMP_DISABLE` unless the process is currently being traced (`PT_PTRACED`).

REQ-8: `setuid()` does NOT affect supplementary groups, EGID/RGID/SGID, fsgid, or pid/sid/pgid.

REQ-9: `setuid()` does NOT affect `no_new_privs`. NNP, once set, persists across setuid (and execve). NNP=1 + unprivileged caller is permitted; NNP=1 + privileged caller is also permitted (cred mutation does not raise caps beyond bounding set).

REQ-10: `setuid()` mutates per-thread cred only. NPTL synchronizes across threads via library wrapper that issues `setuid` on each thread.

REQ-11: Ambient capability set: `cap_ambient` is intersection of `cap_inheritable` and `cap_permitted`. When `cap_permitted` is cleared (root-to-nonroot fixup), ambient set is also cleared to maintain invariant.

REQ-12: `setuid()` is a CRED-MUTATING syscall: failure leaves no observable side effects. `prepare_creds()` ⟶ mutate ⟶ check ⟶ commit_creds() or abort_creds().

REQ-13: i386 16-bit (syscall 23) accepts 16-bit uid; truncation if higher bits set is rejected with EINVAL.

REQ-14: LSM hook: `security_task_fix_setuid(new, old, LSM_SETID_ID)` is invoked before commit; SELinux/AppArmor/Smack/Yama may veto.

REQ-15: RLIMIT_NPROC enforcement on the new RUID (privileged path) — if new uid's nproc count would exceed limit, return EAGAIN.

## Acceptance Criteria

- [ ] AC-1: Root-EUID caller `setuid(1000)`: RUID, EUID, SUID, FSUID all become 1000.
- [ ] AC-2: Setuid binary (EUID=0, RUID=1000, SUID=0) calls `setuid(1000)`: EUID becomes 1000; RUID stays 1000; SUID *also* becomes 1000 (because CAP_SETUID held — privileged path). Permanent drop.
- [ ] AC-3: Unprivileged caller (RUID=1000, EUID=1000, SUID=1000) calls `setuid(2000)`: returns EPERM, cred unchanged.
- [ ] AC-4: Setuid binary calls `setuid(getuid())` to drop EUID temporarily back to RUID without losing SUID? — On Linux, setuid() in the privileged path overwrites SUID too. Use `seteuid()` to drop only EUID.
- [ ] AC-5: Root-to-nonroot transition without SECBIT_KEEP_CAPS: cap_effective and cap_permitted cleared.
- [ ] AC-6: Root-to-nonroot transition with SECBIT_KEEP_CAPS: cap_effective cleared; cap_permitted preserved.
- [ ] AC-7: Root-to-nonroot transition with SECBIT_NO_SETUID_FIXUP: cap_effective and cap_permitted preserved.
- [ ] AC-8: EUID transition: mm.dumpable cleared to SUID_DUMP_DISABLE.
- [ ] AC-9: `uid` not in uid_map: returns EINVAL.
- [ ] AC-10: RLIMIT_NPROC exceeded on new uid: returns EAGAIN.
- [ ] AC-11: NNP=1 caller: setuid succeeds but caps not raised; cap_task_fix_setuid still strips per rules.
- [ ] AC-12: LSM denies via security_task_fix_setuid: returns whatever errno LSM provides (typically EACCES).
- [ ] AC-13: After successful setuid: getuid()/geteuid() reflect new values; /proc/self/status:Uid lines updated.
- [ ] AC-14: setuid in chroot/user-ns: kuid mapping respects active user_ns.

## Architecture

```rust
#[syscall(nr = 105, abi = "sysv")]
pub fn sys_setuid(uid: uid_t) -> i32 {
    Setuid::do_setuid(uid)
}
```

`Setuid::do_setuid(uid) -> Result<(), Errno>`:
1. let old = current_cred();                       // RCU read
2. let user_ns = current_user_ns();
3. let kuid = UidMap::make_kuid(user_ns, uid);
4. if !kuid.is_valid() { return Err(EINVAL); }
5. /* prepare COW cred */
6. let mut new = Cred::prepare(old)?;
7. /* Decide privileged vs unprivileged path */
8. let privileged = ns_capable(user_ns, CAP_SETUID);
9. if privileged {
10.    new.uid = kuid;
11.    new.euid = kuid;
12.    new.suid = kuid;
13.    new.fsuid = kuid;
14. } else if kuid == old.uid || kuid == old.euid || kuid == old.suid {
15.    new.euid = kuid;
16.    new.fsuid = kuid;
17. } else {
18.    Cred::abort(new);
19.    return Err(EPERM);
20. }
21. /* RLIMIT_NPROC for privileged path that changes RUID */
22. if privileged && new.uid != old.uid {
23.    if !rlimit_check_nproc(new.user, new.uid) {
24.        Cred::abort(new);
25.        return Err(EAGAIN);
26.    }
27. }
28. /* Cap-fixup (file-cap-strip / KEEPCAPS / NO_SETUID_FIXUP) */
29. Cap::task_fix_setuid(&mut new, &old, LSM_SETID_ID)?;
30. /* LSM hook */
31. Security::task_fix_setuid(&new, &old, LSM_SETID_ID)?;
32. /* Dumpable */
33. if new.euid != old.euid {
34.    current_mm().set_dumpable(SUID_DUMP_DISABLE);
35. }
36. /* Commit */
37. Cred::commit(new);
38. Ok(())

`Cap::task_fix_setuid(new, old, flag)`:
1. let sb = current_securebits();
2. if sb & SECBIT_NO_SETUID_FIXUP { return Ok(()); }
3. let root_to_nonroot = old.euid == 0 && new.euid != 0;
4. let nonroot_to_root = old.euid != 0 && new.euid == 0;
5. let real_left_root  = old.uid == 0 && new.uid != 0;
6. if root_to_nonroot {
7.     new.cap_effective.clear();
8.     if !(sb & SECBIT_KEEP_CAPS) {
9.         new.cap_permitted.clear();
10.        new.cap_ambient.clear();  // ambient ⊆ permitted invariant
11.    }
12. } else if nonroot_to_root {
13.    new.cap_effective = new.cap_permitted;  // re-grant on root
14. }
15. if real_left_root {
16.    new.cap_bounding.clear();
17.    new.cap_inheritable.clear();
18.    new.cap_ambient.clear();
19. }
20. /* Maintain ambient ⊆ inheritable ∩ permitted */
21. new.cap_ambient &= new.cap_inheritable;
22. new.cap_ambient &= new.cap_permitted;
23. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `unprivileged_uid_in_russsaved` | INVARIANT | per-setuid unprivileged: kuid ∈ {ruid, euid, suid} or EPERM. |
| `privileged_sets_all_three` | INVARIANT | per-setuid privileged: ruid=euid=suid=kuid post-commit. |
| `cred_cow_atomicity` | INVARIANT | per-setuid: failure ⟹ no cred mutation observable. |
| `ambient_subset_invariant` | INVARIANT | per-cap-fixup: cap_ambient ⊆ cap_inheritable ∩ cap_permitted. |
| `nnp_no_priv_gain` | INVARIANT | per-setuid + NNP: cap_permitted not increased. |
| `dumpable_cleared_on_euid_change` | INVARIANT | per-setuid: EUID changed ⟹ dumpable = SUID_DUMP_DISABLE. |
| `securebits_keep_caps_honored` | INVARIANT | per-cap-fixup: SECBIT_KEEP_CAPS ⟹ permitted preserved on root→nonroot. |
| `securebits_no_setuid_fixup_honored` | INVARIANT | per-cap-fixup: SECBIT_NO_SETUID_FIXUP ⟹ no cap mutation. |
| `kuid_valid_or_einval` | INVARIANT | per-setuid: make_kuid returns INVALID ⟹ EINVAL. |

### Layer 2: TLA+

`kernel/setuid.tla`:
- States: (ruid, euid, suid, fsuid, cap_e, cap_p, cap_b, cap_i, cap_a, securebits, dumpable, nnp).
- Actions: setuid_priv, setuid_unpriv, setuid_fail.
- Properties:
  - `safety_unpriv_no_escalation` — unpriv path: euid' ∈ {ruid, euid, suid}.
  - `safety_priv_all_three` — priv path: ruid'=euid'=suid'.
  - `safety_root_to_nonroot_strips_caps` — euid 0→nonzero + !SECBIT_NO_SETUID_FIXUP + !SECBIT_KEEP_CAPS ⟹ cap_p' = ∅.
  - `safety_real_left_root_clears_bounding` — ruid 0→nonzero ⟹ cap_b' = ∅, cap_i' = ∅.
  - `safety_ambient_subset` — cap_a' ⊆ cap_i' ∩ cap_p'.
  - `safety_nnp_holds` — nnp=1 ⟹ cap_p' ⊆ cap_p.
  - `safety_failure_no_mutation` — error return ⟹ state unchanged.
  - `liveness_terminates` — setuid returns in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setuid` post: success ⟹ cred reflects spec, failure ⟹ cred unchanged | `Setuid::do_setuid` |
| `prepare_creds` / `commit_creds` / `abort_creds` lifecycle | `Cred::prepare`/`commit`/`abort` |
| `cap_task_fix_setuid` post: per-securebits/per-transition spec | `Cap::task_fix_setuid` |
| `rlimit_check_nproc` post: nproc count consistent | `Rlimit::check_nproc` |

### Layer 4: Verus / Creusot functional

Per-`setuid(2)` man-page equivalence. POSIX.1-2008 `setuid` semantics (saved-set ID behavior). LTP `setuid01..setuid04` pass. Joint verification with `capabilities(7)` and `credentials(7)`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setuid(2)` reinforcement:

- **Per-cred RCU-COW** — defense against per-torn-cred update; failure commits nothing.
- **Per-CAP_SETUID strict + uid_map validation** — defense against per-unprivileged escalation and per-invalid-kuid.
- **Per-cap_task_fix_setuid + ambient-subset** — defense against per-root-cap-retention; permitted cleared by default; cap_ambient ⊆ inheritable ∩ permitted.
- **Per-dumpable cleared + NNP + RLIMIT_NPROC** — defense against per-coredump/ptrace-attach, per-escalation, per-fork-bomb.
- **Per-LSM hook + i386 16-bit clamp** — defense via SELinux/AppArmor/Yama veto; 32-bit syscall preferred.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at setuid entry** — randomizes kernel stack offset; defeats stack-spray and ret2dir attacks aimed at the cred-fixup path.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Uid/Gid/Cap* lines restricted; setuid-induced cap changes are not observable to other users, denying reconnaissance.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — `setuid()` inside a grsec chroot honors only the chroot's user-ns; cannot escalate by referencing host UIDs. uid_map is sealed at chroot entry.
- **GRKERNSEC_AUDIT_GROUP on setuid** — every setuid() call from audited groups is logged with old/new UID, old/new cap_permitted, securebits state, and NNP state. Privilege transitions are forensically traceable.
- **CAP_SETUID strict** — grsec policy can globally deny CAP_SETUID; setuid() returns EPERM for any non-{ruid,euid,suid} target. Combined with file-cap-strip, setuid binaries cannot raise EUID at all.
- **file-cap-strip + securebits SECBIT_KEEP_CAPS** — grsec strips file-caps on filesystem mount (TPE-like). SECBIT_KEEP_CAPS preserves cap_permitted across setuid root→nonroot (POSIX-draft "compat") but grsec audits its use and can deny SECBIT_KEEP_CAPS to unaudited tasks via prctl-gating.
- **no_new_privs across setuid** — when PR_SET_NO_NEW_PRIVS is set, setuid() succeeds but `cap_task_fix_setuid` never raises permitted; combined with grsec's NNP-by-default policy for unprivileged user-ns, this fully blocks setuid-based escalation.
- **Ambient / inheritable cap evolution rules** — grsec enforces: `cap_ambient ⊆ cap_inheritable ∩ cap_permitted`; `cap_inheritable` cannot be raised beyond bounding by unprivileged callers; real-uid leaving root clears bounding+inheritable+ambient (permanent drop); `cap_ambient` reset on any euid transition involving 0 (defense against ambient-cap-bleed).
- **PAX_USERCOPY on cred copy** — copy of cred fields is bounds-checked against `struct cred` slab object; defeats heap-spray that targets cred slab.
- **SECBIT_NO_SETUID_FIXUP audited** — use of SECBIT_NO_SETUID_FIXUP is logged and gated by an explicit grsec capability; default-deny.
- **GRKERNSEC_HARDEN_TTY on setuid binary** — TTY hijack mitigations active across setuid; `ioctl(TIOCSTI)` is denied if setuid raised EUID.
- **Per-dumpable enforced + ptrace gated** — grsec layered on top of `mm.dumpable = SUID_DUMP_DISABLE`: ptrace attach denied unless caller is exactly the new EUID (no SUID_DUMP_ROOT escapes).
- **Per-RLIMIT_NPROC enforced strictly** — grsec RBAC can lower NPROC for the new uid below RLIMIT; setuid returns EAGAIN earlier than upstream.

## Open Questions

- (none at this Tier-5 level; capability transition rules verified via Layer 2/3.)

## Out of Scope

- `seteuid(2)` (Tier-5 separate doc — effective-only set, no SUID overwrite).
- `setreuid(2)` (Tier-5 separate doc — real + effective with swap semantics).
- `setresuid(2)` (Tier-5 separate doc — explicit real/effective/saved triple).
- `setfsuid(2)` (Tier-5 separate doc — file-system UID).
- File capabilities and setuid-execve cred transition (Tier-3 in `security/commoncap.md` and `fs/exec.md`).
- Securebits prctl interface (Tier-5 in `prctl.md`).
- User-namespace uid_map semantics (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.
