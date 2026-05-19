# Tier-5 syscall: setresuid(2) — syscall 117

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE3(setresuid, uid_t, uid_t, uid_t))
  - kernel/cred.c (prepare_creds, commit_creds, abort_creds)
  - security/commoncap.c (cap_task_fix_setuid)
  - include/linux/cred.h (struct cred)
  - include/linux/securebits.h
  - arch/x86/entry/syscalls/syscall_64.tbl (117  common  setresuid)
-->

## Summary

`setresuid(2)` sets the calling task's **real**, **effective**, and **saved** UIDs explicitly and independently. Each of the three arguments can be `-1` ("keep current value"). This is the most precise of the setuid family — `setuid(2)`, `setreuid(2)`, and `seteuid(2)` can all be expressed in terms of it. POSIX.1e draft (saved-set IDs); supported on Linux since 2.1.44.

Privilege rules:
- **Privileged caller** (holds `CAP_SETUID` in caller's user-ns): may set each of {ruid, euid, suid} to any valid kuid independently.
- **Unprivileged caller**: each non-`(uid_t)-1` argument must equal one of `{RUID, EUID, SUID}` (current values). This permits temporarily dropping EUID and restoring it later via SUID, which is the canonical use case (setuid binaries needing reversible privilege drop).

Critical side effects (same as setuid):
- **File-capability strip / restore** via `cap_task_fix_setuid()` on EUID transitions; governed by `SECBIT_KEEP_CAPS` and `SECBIT_NO_SETUID_FIXUP`.
- **Real-uid leaves root** (`old.uid==0 ∧ new.uid!=0`) clears `cap_bounding` and `cap_inheritable` (POSIX-draft permanent drop).
- **Dumpable cleared** when EUID changes.
- **FSUID follows EUID** automatically (set to new EUID unless setfsuid was used independently).
- **RLIMIT_NPROC** enforced on a new RUID.

## Signature

```c
int setresuid(uid_t ruid, uid_t euid, uid_t suid);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ruid` | `uid_t` | in | New real UID, or `(uid_t)-1` to keep current. |
| `euid` | `uid_t` | in | New effective UID, or `(uid_t)-1` to keep current. |
| `suid` | `uid_t` | in | New saved-set UID, or `(uid_t)-1` to keep current. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. Cred updated. |
| `-1` | Failure; `errno` set. No cred changes are committed. |

## Errors

| errno | When |
|---|---|
| `EAGAIN` | RUID would change and the new RUID's per-user RLIMIT_NPROC would be exceeded. |
| `EINVAL` | One of {ruid, euid, suid} is not `(uid_t)-1` and is not a valid value in the caller's user namespace. |
| `EPERM` | Unprivileged caller AND one of the requested values is not `-1` and is not in {RUID, EUID, SUID}. |

## ABI surface

```text
__NR_setresuid (x86_64)    = 117
__NR_setresuid (i386)      = 164    /* 16-bit uid_t variant */
__NR_setresuid32 (i386)    = 208    /* 32-bit uid_t — preferred */
__NR_setresuid (arm64)     = 147    /* generic-syscall */
__NR_setresuid (generic)   = 147

typedef __kernel_uid32_t  uid_t;     /* unsigned int — 32-bit */

#define KEEP_UID  ((uid_t)-1)        /* 0xFFFFFFFF — "do not change" sentinel */

/* securebits */
SECBIT_NO_SETUID_FIXUP    = 1 << 2
SECBIT_KEEP_CAPS          = 1 << 4

/* dumpable */
SUID_DUMP_DISABLE = 0
```

## Compatibility contract

REQ-1: Syscall number is **117** on x86_64; **147** on arm64 / generic. ABI-stable.

REQ-2: For each argument `a` ∈ {ruid, euid, suid}: if `a == (uid_t)-1`, retain current value; else convert via `make_kuid(current_user_ns(), a)`. If conversion yields `INVALID_UID`, return `EINVAL`.

REQ-3: Privileged path (`ns_capable(current_user_ns(), CAP_SETUID)`): each non-`-1` argument is accepted unconditionally.

REQ-4: Unprivileged path: for each non-`-1` argument `kuid_arg`, require `kuid_arg ∈ {cred.uid, cred.euid, cred.suid}`. If any check fails, return `EPERM`.

REQ-5: After applying changes:
- new.uid  = (ruid == -1) ? old.uid  : kuid_arg
- new.euid = (euid == -1) ? old.euid : keuid_arg
- new.suid = (suid == -1) ? old.suid : ksuid_arg
- new.fsuid = new.euid                              /* fsuid tracks euid by default */

REQ-6: Cap-fixup via `cap_task_fix_setuid(new, old, LSM_SETID_RES)`:
- If `securebits & SECBIT_NO_SETUID_FIXUP`: no cap mutation.
- Else if old.euid == 0 ∧ new.euid != 0 (root-to-nonroot EUID transition):
  - clear `cap_effective` and `cap_permitted` (unless `SECBIT_KEEP_CAPS`, which preserves permitted).
- Else if old.euid != 0 ∧ new.euid == 0 (nonroot-to-root EUID transition):
  - cap_effective = cap_permitted (re-grant on root).
- Else if old.uid == 0 ∧ new.uid != 0 (RUID leaves root):
  - clear cap_bounding, cap_inheritable, cap_ambient (permanent drop semantics).
- Maintain ambient ⊆ inheritable ∩ permitted invariant.

REQ-7: `commit_creds(new)`:
- Atomically swap `current->real_cred` and `current->cred` via RCU-COW.
- Update per-user RLIMIT_NPROC accounting if RUID changed.
- Free old cred via RCU.

REQ-8: If EUID changes, set `mm.dumpable = SUID_DUMP_DISABLE`.

REQ-9: `setresuid()` does NOT modify EGID/RGID/SGID/FSGID, supplementary groups, pid/sid/pgid.

REQ-10: `setresuid()` does NOT affect `no_new_privs`. NNP, once set, persists.

REQ-11: `setresuid()` mutates per-thread cred only. NPTL synchronizes via library wrapper.

REQ-12: `setresuid()` is CRED-MUTATING: failure leaves no observable side effects (`prepare_creds` ⟶ mutate ⟶ check ⟶ commit or abort).

REQ-13: i386 16-bit (syscall 164) accepts 16-bit uids; truncation rejected with EINVAL. `setresuid32` (208) is preferred.

REQ-14: LSM hook: `security_task_fix_setuid(new, old, LSM_SETID_RES)` invoked before commit; SELinux/AppArmor/Smack/Yama may veto.

REQ-15: Atomicity: all three transitions are evaluated jointly; either all three are committed or none. Partial application is impossible.

REQ-16: `setresuid(0, 0, 0)` is the canonical privilege-restore from a setuid binary that previously did `setresuid(uid, uid, -1)` to temporarily drop EUID while retaining SUID=0.

## Acceptance Criteria

- [ ] AC-1: Privileged caller `setresuid(100, 200, 300)`: RUID=100, EUID=200, SUID=300, FSUID=200.
- [ ] AC-2: Privileged caller `setresuid(-1, 1000, -1)`: only EUID and FSUID change to 1000; RUID and SUID preserved.
- [ ] AC-3: Unprivileged caller (RUID=1000, EUID=0, SUID=0) `setresuid(-1, 1000, -1)`: EUID=1000, FSUID=1000; RUID and SUID unchanged. Canonical drop-EUID pattern.
- [ ] AC-4: Unprivileged caller from AC-3 then `setresuid(-1, 0, -1)`: EUID=0 (restored from SUID). Canonical regain-EUID pattern.
- [ ] AC-5: Unprivileged caller `setresuid(-1, 9999, -1)` where 9999 ∉ {RUID, EUID, SUID}: EPERM, cred unchanged.
- [ ] AC-6: `setresuid(-1, -1, -1)`: succeeds, cred unchanged.
- [ ] AC-7: Root-to-nonroot EUID transition without SECBIT_KEEP_CAPS: cap_effective and cap_permitted cleared.
- [ ] AC-8: Root-to-nonroot EUID transition with SECBIT_KEEP_CAPS: cap_effective cleared; cap_permitted preserved.
- [ ] AC-9: Root-to-nonroot EUID transition with SECBIT_NO_SETUID_FIXUP: cap_effective and cap_permitted preserved.
- [ ] AC-10: RUID leaves root (e.g., `setresuid(1000, -1, -1)`): cap_bounding and cap_inheritable cleared; ambient cleared.
- [ ] AC-11: EUID transition: mm.dumpable cleared to SUID_DUMP_DISABLE.
- [ ] AC-12: Any of {ruid, euid, suid} non-`-1` and not in uid_map: returns EINVAL, cred unchanged.
- [ ] AC-13: RLIMIT_NPROC exceeded for new RUID: returns EAGAIN, cred unchanged.
- [ ] AC-14: LSM veto: returns LSM-provided errno, cred unchanged.
- [ ] AC-15: After successful setresuid: getresuid() returns (new.ruid, new.euid, new.suid); /proc/self/status:Uid line updated.
- [ ] AC-16: i386 syscall 164 returns clamped 16-bit; syscall 208 accepts full 32-bit.
- [ ] AC-17: NNP=1 caller: setresuid succeeds but cap_permitted is not raised.

## Architecture

```rust
#[syscall(nr = 117, abi = "sysv")]
pub fn sys_setresuid(ruid: uid_t, euid: uid_t, suid: uid_t) -> i32 {
    Setresuid::do_setresuid(ruid, euid, suid)
}
```

`Setresuid::do_setresuid(ruid, euid, suid) -> Result<(), Errno>`:
1. let old = current_cred(); let user_ns = current_user_ns();
2. /* Resolve each non-(-1) arg via make_kuid; INVALID ⟹ EINVAL */
3. let kruid = resolve_or_keep(ruid, KEEP_UID, old.uid, user_ns)?;
4. let keuid = resolve_or_keep(euid, KEEP_UID, old.euid, user_ns)?;
5. let ksuid = resolve_or_keep(suid, KEEP_UID, old.suid, user_ns)?;
6. /* Privilege check */
7. let privileged = ns_capable(user_ns, CAP_SETUID);
8. if !privileged {
9.     let cur = [old.uid, old.euid, old.suid];
10.    if ruid != KEEP_UID && !cur.contains(&kruid) { return Err(EPERM); }
11.    if euid != KEEP_UID && !cur.contains(&keuid) { return Err(EPERM); }
12.    if suid != KEEP_UID && !cur.contains(&ksuid) { return Err(EPERM); }
13. }
14. /* Prepare COW cred */
15. let mut new = Cred::prepare(old)?;
16. new.uid = kruid; new.euid = keuid; new.suid = ksuid;
17. new.fsuid = keuid;                       // fsuid follows euid
18. /* RLIMIT_NPROC if RUID changed */
19. if kruid != old.uid && !rlimit_check_nproc(new.user, kruid) {
20.    Cred::abort(new); return Err(EAGAIN);
21. }
22. /* Cap-fixup + LSM */
23. Cap::task_fix_setuid(&mut new, &old, LSM_SETID_RES)?;
24. Security::task_fix_setuid(&new, &old, LSM_SETID_RES)?;
25. /* Dumpable */
26. if new.euid != old.euid { current_mm().set_dumpable(SUID_DUMP_DISABLE); }
27. /* Commit */
28. Cred::commit(new); Ok(())

`Cap::task_fix_setuid(new, old, LSM_SETID_RES)`: same per-securebits/per-transition spec as setuid(2) — see `setuid.md` § Architecture. Summary: SECBIT_NO_SETUID_FIXUP ⟹ no-op; root→nonroot EUID clears effective and (unless SECBIT_KEEP_CAPS) permitted + ambient; nonroot→root EUID sets effective = permitted; RUID-leaves-root clears bounding + inheritable + ambient; ambient ⊆ inheritable ∩ permitted invariant restored.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `minus_one_keep` | INVARIANT | per-setresuid: arg == (uid_t)-1 ⟹ corresponding field unchanged. |
| `unprivileged_in_current_set` | INVARIANT | per-setresuid unprivileged: each non-(-1) ∈ {ruid, euid, suid} or EPERM. |
| `privileged_unrestricted` | INVARIANT | per-setresuid privileged: any valid kuid accepted. |
| `cred_cow_atomicity` | INVARIANT | per-setresuid: failure ⟹ no cred mutation observable. |
| `fsuid_tracks_euid` | INVARIANT | per-setresuid: new.fsuid == new.euid post-commit. |
| `ambient_subset` | INVARIANT | per-cap-fixup: cap_ambient ⊆ cap_inheritable ∩ cap_permitted. |
| `dumpable_on_euid_change` | INVARIANT | per-setresuid: EUID changed ⟹ dumpable = SUID_DUMP_DISABLE. |
| `securebits_keep_caps_honored` | INVARIANT | per-cap-fixup: SECBIT_KEEP_CAPS ⟹ permitted preserved. |
| `securebits_no_setuid_fixup_honored` | INVARIANT | per-cap-fixup: SECBIT_NO_SETUID_FIXUP ⟹ no cap mutation. |
| `nnp_no_priv_gain` | INVARIANT | per-setresuid + NNP: cap_permitted not increased. |
| `kuid_valid_or_einval` | INVARIANT | per-setresuid: any non-(-1) make_kuid INVALID ⟹ EINVAL. |
| `all_or_nothing` | INVARIANT | per-setresuid: commit applies all three or none. |

### Layer 2: TLA+

`kernel/setresuid.tla`:
- States: (ruid, euid, suid, fsuid, cap_e, cap_p, cap_b, cap_i, cap_a, securebits, dumpable, nnp).
- Actions: setresuid_priv, setresuid_unpriv, setresuid_keep_uid_one_field, setresuid_fail.
- Properties:
  - `safety_minus_one_keep` — arg == -1 ⟹ field unchanged.
  - `safety_unpriv_no_escalation` — unpriv: each non-(-1) ∈ {ruid, euid, suid}.
  - `safety_priv_unrestricted` — priv: any kuid accepted.
  - `safety_fsuid_eq_euid` — post: fsuid' = euid'.
  - `safety_root_to_nonroot_strips_caps` — euid 0→nonzero + !SECBIT_NO_SETUID_FIXUP + !SECBIT_KEEP_CAPS ⟹ cap_p' = ∅.
  - `safety_real_left_root` — uid 0→nonzero ⟹ cap_b' = ∅ ∧ cap_i' = ∅.
  - `safety_ambient_subset` — cap_a' ⊆ cap_i' ∩ cap_p'.
  - `safety_nnp_holds` — nnp=1 ⟹ cap_p' ⊆ cap_p.
  - `safety_failure_no_mutation` — error return ⟹ state unchanged.
  - `safety_atomicity` — three-tuple update is atomic.
  - `liveness_terminates` — setresuid returns in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setresuid` post: success ⟹ (uid', euid', suid', fsuid') per spec; failure ⟹ cred unchanged | `Setresuid::do_setresuid` |
| `prepare_creds` / `commit_creds` / `abort_creds` lifecycle | `Cred::prepare`/`commit`/`abort` |
| `cap_task_fix_setuid` post (LSM_SETID_RES flavor) | `Cap::task_fix_setuid` |
| `rlimit_check_nproc` post | `Rlimit::check_nproc` |

### Layer 4: Verus / Creusot functional

Per-`setresuid(2)` man-page equivalence. POSIX.1e draft (saved-set IDs) verified. LTP `setresuid01..setresuid04` pass. Joint verification with `setresgid(2)` and `capabilities(7)`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setresuid(2)` reinforcement:

- **Per-cred RCU-COW** — defense against per-torn-cred update; failure path commits nothing.
- **Per-(-1) keep-semantics strict** — defense against per-conflation of keep vs zero-uid.
- **Per-CAP_SETUID strict** — defense against per-unprivileged escalation.
- **Per-uid_map validation per-field** — defense against per-invalid-kuid for each non-(-1) arg.
- **Per-cap_task_fix_setuid** — defense against per-root-cap-retention on EUID drop; same rules as setuid.
- **Per-ambient-subset invariant** — defense against per-cap-leak via ambient set after fixup.
- **Per-dumpable cleared** — defense against per-coredump / per-ptrace-attach on setuid binary after EUID change.
- **Per-NNP** — defense against per-escalation via setresuid in already-restricted task.
- **Per-RLIMIT_NPROC** — defense against per-fork-bomb via new RUID.
- **Per-all-or-nothing commit** — defense against per-partial-update; either all three transitions land or none.
- **Per-fsuid-follows-euid** — defense against per-fsuid-stuck-at-old-value (file-perm-bypass).
- **Per-LSM hook** — defense via SELinux/AppArmor/Yama veto on LSM_SETID_RES.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at setresuid entry** — randomizes kernel stack offset; defeats stack-spray attacks aimed at the cred-fixup path. Setresuid is a privileged-path syscall; randomization defeats ret2dir and stack-pivot.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Uid line (4 fields: real, effective, saved, fs) restricted; setresuid-induced changes are not observable to other users. The Cap* lines are also protected, denying cap-state reconnaissance.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — `setresuid()` inside a grsec chroot honors only the chroot's user-ns. The uid_map is sealed at chroot entry; setresuid cannot reference host UIDs that are not mapped.
- **GRKERNSEC_AUDIT_GROUP on setuid family** — every `setresuid()` call from audited users is logged with old/new (ruid, euid, suid), old/new cap_permitted, securebits state, and NNP state. The drop-and-restore pattern (`setresuid(-1, uid, -1)` followed by `setresuid(-1, 0, -1)`) is logged in full.
- **CAP_SETUID strict** — grsec policy can globally deny CAP_SETUID; setresuid() reduces to the unprivileged path even for EUID=0 callers. Combined with file-cap-strip, setuid binaries cannot use the privileged path to overwrite SUID.
- **file-cap-strip + securebits SECBIT_KEEP_CAPS** — grsec strips file-caps on filesystem mount. SECBIT_KEEP_CAPS preserves cap_permitted across the EUID drop-restore cycle (the canonical setresuid use case); grsec audits SECBIT_KEEP_CAPS use and can deny it to unaudited tasks via prctl-gating.
- **no_new_privs across setuid family** — NNP=1 fully neutralizes setresuid as an escalation vector: `cap_task_fix_setuid` never raises permitted under NNP. Grsec enforces NNP-by-default for unprivileged user-ns and gates the unprivileged-user-ns creation behind a separate capability.
- **Ambient / inheritable cap evolution rules** — grsec enforces all four invariants:
  1. `cap_ambient ⊆ cap_inheritable ∩ cap_permitted`.
  2. EUID 0→nonzero clears ambient (defense against ambient-cap-bleed across temporary EUID drop).
  3. RUID 0→nonzero clears bounding+inheritable+ambient (permanent drop).
  4. EUID restore (nonzero→0) does NOT re-raise ambient (it can only be raised via PR_CAP_AMBIENT_RAISE).
- **PAX_USERCOPY on cred copy** — copy of cred fields is bounds-checked against `struct cred` slab; defeats heap-spray on cred slab.
- **SECBIT_NO_SETUID_FIXUP audited** — use of SECBIT_NO_SETUID_FIXUP is logged and gated by an explicit grsec capability; default-deny. The "preserve all caps across setresuid" mode is treated as a privileged operation in itself.
- **GRKERNSEC_HARDEN_TTY on EUID restoration** — TTY hijack mitigations re-armed when EUID transitions to 0; TIOCSTI denied if the restoration came from an unaudited setuid binary.
- **Per-dumpable enforced + ptrace gated** — grsec layered on `mm.dumpable = SUID_DUMP_DISABLE`; ptrace attach denied across the drop-restore cycle.
- **Per-RLIMIT_NPROC enforced strictly** — grsec RBAC can lower NPROC for the new RUID; setresuid returns EAGAIN earlier than upstream.
- **Per-CHROOT_NO_SETUID** — grsec can deny CAP_SETUID entirely inside a chroot, forcing setresuid to operate exclusively in the unprivileged path.
- **Per-thread cred consistency** — grsec audits per-thread (ruid, euid, suid) drift caused by NPTL-sync edge cases; divergence is logged as a kernel-integrity event.

## Open Questions

- (none at this Tier-5 level; the explicit three-tuple semantics admit no ambiguity.)

## Out of Scope

- `setuid(2)` (Tier-5 separate doc — single-arg, all-three-on-priv).
- `seteuid(2)` (Tier-5 separate doc — effective only).
- `setreuid(2)` (Tier-5 separate doc — real + effective with swap semantics).
- `setfsuid(2)` (Tier-5 separate doc — file-system UID).
- `getresuid(2)` (Tier-5 separate doc — returns the triple).
- File capabilities and setuid-execve cred transition (Tier-3 in `security/commoncap.md` and `fs/exec.md`).
- Securebits prctl interface (Tier-5 in `prctl.md`).
- User-namespace uid_map semantics (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.
