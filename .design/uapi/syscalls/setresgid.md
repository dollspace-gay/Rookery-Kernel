# Tier-5 syscall: setresgid(2) — syscall 119

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE3(setresgid, gid_t, gid_t, gid_t))
  - kernel/cred.c (prepare_creds, commit_creds, abort_creds)
  - include/linux/cred.h (struct cred)
  - include/linux/securebits.h
  - arch/x86/entry/syscalls/syscall_64.tbl (119  common  setresgid)
-->

## Summary

`setresgid(2)` sets the calling task's **real**, **effective**, and **saved** GIDs explicitly and independently. Each of the three arguments can be `-1` ("keep current value"). This is the most precise of the setgid family — `setgid(2)`, `setregid(2)`, and `setegid(2)` can all be expressed in terms of it. POSIX.1e draft (saved-set IDs); supported on Linux since 2.1.44.

Privilege rules:
- **Privileged caller** (holds `CAP_SETGID` in caller's user-ns): may set each of {rgid, egid, sgid} to any valid kgid independently.
- **Unprivileged caller**: each non-`(gid_t)-1` argument must equal one of `{RGID, EGID, SGID}` (current values). This permits temporarily dropping EGID and restoring via SGID — the canonical setgid-binary pattern for reversible group-privilege drop.

Critical side effects:
- **No capability mutation**: unlike `setresuid()`, capabilities are NOT modified by `setresgid()`. Capabilities track UID, not GID.
- **Supplementary groups untouched**: use `setgroups(2)`.
- **Dumpable cleared** when EGID changes (same protection as setresuid for setgid binaries).
- **FSGID follows EGID** automatically (set to new EGID).
- **NNP propagation**: `no_new_privs` is preserved.

## Signature

```c
int setresgid(gid_t rgid, gid_t egid, gid_t sgid);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `rgid` | `gid_t` | in | New real GID, or `(gid_t)-1` to keep current. |
| `egid` | `gid_t` | in | New effective GID, or `(gid_t)-1` to keep current. |
| `sgid` | `gid_t` | in | New saved-set GID, or `(gid_t)-1` to keep current. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. Cred updated. |
| `-1` | Failure; `errno` set. No cred changes are committed. |

## Errors

| errno | When |
|---|---|
| `EINVAL` | One of {rgid, egid, sgid} is not `(gid_t)-1` and is not a valid value in the caller's user namespace (not in gid_map). |
| `EPERM` | Unprivileged caller AND one of the requested values is not `-1` and is not in {RGID, EGID, SGID}. |

(Note: no `EAGAIN` — RLIMIT_NPROC is per-UID, not per-GID.)

## ABI surface

```text
__NR_setresgid (x86_64)    = 119
__NR_setresgid (i386)      = 170    /* 16-bit gid_t variant */
__NR_setresgid32 (i386)    = 210    /* 32-bit gid_t — preferred */
__NR_setresgid (arm64)     = 149    /* generic-syscall */
__NR_setresgid (generic)   = 149

typedef __kernel_gid32_t  gid_t;     /* unsigned int — 32-bit */

#define KEEP_GID  ((gid_t)-1)        /* 0xFFFFFFFF — "do not change" sentinel */

/* dumpable */
SUID_DUMP_DISABLE = 0
```

## Compatibility contract

REQ-1: Syscall number is **119** on x86_64; **149** on arm64 / generic. ABI-stable.

REQ-2: For each argument `a` ∈ {rgid, egid, sgid}: if `a == (gid_t)-1`, retain current value; else convert via `make_kgid(current_user_ns(), a)`. If conversion yields `INVALID_GID`, return `EINVAL`.

REQ-3: Privileged path (`ns_capable(current_user_ns(), CAP_SETGID)`): each non-`-1` argument is accepted unconditionally.

REQ-4: Unprivileged path: for each non-`-1` argument `kgid_arg`, require `kgid_arg ∈ {cred.gid, cred.egid, cred.sgid}`. If any check fails, return `EPERM`.

REQ-5: After applying changes:
- new.gid  = (rgid == -1) ? old.gid  : kgid_arg
- new.egid = (egid == -1) ? old.egid : kegid_arg
- new.sgid = (sgid == -1) ? old.sgid : ksgid_arg
- new.fsgid = new.egid                              /* fsgid tracks egid by default */

REQ-6: NO capability mutation. cap_effective, cap_permitted, cap_bounding, cap_inheritable, cap_ambient are all PRESERVED across setresgid.

REQ-7: `commit_creds(new)`:
- Atomically swap `current->real_cred` and `current->cred` via RCU-COW.
- Free old cred via RCU.

REQ-8: If EGID changes, set `mm.dumpable = SUID_DUMP_DISABLE` (unless ptraced).

REQ-9: `setresgid()` does NOT modify EUID/RUID/SUID/FSUID, supplementary groups, pid/sid/pgid.

REQ-10: `setresgid()` does NOT affect `no_new_privs`. NNP persists.

REQ-11: `setresgid()` mutates per-thread cred only. NPTL synchronizes via library wrapper.

REQ-12: CRED-MUTATING: failure leaves no observable side effects (`prepare_creds` ⟶ mutate ⟶ check ⟶ commit or abort).

REQ-13: i386 16-bit (syscall 170) accepts 16-bit gids; truncation rejected with EINVAL. `setresgid32` (210) is preferred.

REQ-14: LSM hook: `security_task_fix_setgid(new, old, LSM_SETID_RES)` invoked before commit; SELinux/AppArmor/Smack/Yama may veto.

REQ-15: Atomicity: all three transitions are evaluated jointly; either all three are committed or none.

REQ-16: `setresgid(g, g, -1)` then later `setresgid(-1, 0, -1)` is the canonical drop-and-restore EGID pattern (paralleling setresuid).

## Acceptance Criteria

- [ ] AC-1: Privileged caller `setresgid(100, 200, 300)`: RGID=100, EGID=200, SGID=300, FSGID=200.
- [ ] AC-2: Privileged caller `setresgid(-1, 1000, -1)`: only EGID and FSGID change to 1000.
- [ ] AC-3: Unprivileged caller (RGID=1000, EGID=0, SGID=0) `setresgid(-1, 1000, -1)`: EGID=1000, FSGID=1000; RGID and SGID unchanged. Drop pattern.
- [ ] AC-4: Unprivileged caller from AC-3 then `setresgid(-1, 0, -1)`: EGID=0 (restored from SGID). Restore pattern.
- [ ] AC-5: Unprivileged caller `setresgid(-1, 9999, -1)` where 9999 ∉ {RGID, EGID, SGID}: EPERM, cred unchanged.
- [ ] AC-6: `setresgid(-1, -1, -1)`: succeeds, cred unchanged.
- [ ] AC-7: Capabilities preserved across setresgid (no mutation of cap_e, cap_p, cap_b, cap_i, cap_a).
- [ ] AC-8: Supplementary groups preserved (group_info unchanged).
- [ ] AC-9: EUID/RUID/SUID/FSUID preserved.
- [ ] AC-10: EGID transition: mm.dumpable cleared to SUID_DUMP_DISABLE.
- [ ] AC-11: Any of {rgid, egid, sgid} non-`-1` and not in gid_map: returns EINVAL.
- [ ] AC-12: LSM veto: returns LSM-provided errno.
- [ ] AC-13: After successful setresgid: getresgid() returns (new.rgid, new.egid, new.sgid); /proc/self/status:Gid line updated.
- [ ] AC-14: i386 syscall 170 returns clamped 16-bit; syscall 210 accepts full 32-bit.
- [ ] AC-15: NNP=1 caller: setresgid succeeds.
- [ ] AC-16: All-or-nothing: if any field fails validation, no field changes.

## Architecture

```rust
#[syscall(nr = 119, abi = "sysv")]
pub fn sys_setresgid(rgid: gid_t, egid: gid_t, sgid: gid_t) -> i32 {
    Setresgid::do_setresgid(rgid, egid, sgid)
}
```

`Setresgid::do_setresgid(rgid, egid, sgid) -> Result<(), Errno>`:
1. let old = current_cred();
2. let user_ns = current_user_ns();
3. /* Convert each non-(-1) to kgid_t */
4. let krgid = if rgid == KEEP_GID { old.gid } else {
5.     let k = GidMap::make_kgid(user_ns, rgid);
6.     if !k.is_valid() { return Err(EINVAL); }
7.     k
8. };
9. let kegid = if egid == KEEP_GID { old.egid } else {
10.    let k = GidMap::make_kgid(user_ns, egid);
11.    if !k.is_valid() { return Err(EINVAL); }
12.    k
13. };
14. let ksgid = if sgid == KEEP_GID { old.sgid } else {
15.    let k = GidMap::make_kgid(user_ns, sgid);
16.    if !k.is_valid() { return Err(EINVAL); }
17.    k
18. };
19. /* Privilege check */
20. let privileged = ns_capable(user_ns, CAP_SETGID);
21. if !privileged {
22.    let current_set = [old.gid, old.egid, old.sgid];
23.    if rgid != KEEP_GID && !current_set.contains(&krgid) { return Err(EPERM); }
24.    if egid != KEEP_GID && !current_set.contains(&kegid) { return Err(EPERM); }
25.    if sgid != KEEP_GID && !current_set.contains(&ksgid) { return Err(EPERM); }
26. }
27. /* Prepare COW cred */
28. let mut new = Cred::prepare(old)?;
29. new.gid = krgid;
30. new.egid = kegid;
31. new.sgid = ksgid;
32. new.fsgid = kegid;                       // fsgid follows egid
33. /* LSM hook */
34. Security::task_fix_setgid(&new, &old, LSM_SETID_RES)?;
35. /* Dumpable */
36. if new.egid != old.egid {
37.    current_mm().set_dumpable(SUID_DUMP_DISABLE);
38. }
39. /* Commit — no cap mutation */
40. Cred::commit(new);
41. Ok(())

`GidMap::make_kgid(ns, gid) -> kgid_t`:
1. for entry in ns.gid_map.extents() {
2.     if gid ∈ [entry.first, entry.first + entry.count) {
3.         return kgid_t { val: entry.lower_first + (gid - entry.first) };
4.     }
5. }
6. return INVALID_KGID;

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `minus_one_keep` | INVARIANT | per-setresgid: arg == (gid_t)-1 ⟹ corresponding field unchanged. |
| `unprivileged_in_current_set` | INVARIANT | per-setresgid unprivileged: each non-(-1) ∈ {rgid, egid, sgid} or EPERM. |
| `privileged_unrestricted` | INVARIANT | per-setresgid privileged: any valid kgid accepted. |
| `cred_cow_atomicity` | INVARIANT | per-setresgid: failure ⟹ no cred mutation observable. |
| `fsgid_tracks_egid` | INVARIANT | per-setresgid: new.fsgid == new.egid post-commit. |
| `no_cap_mutation` | INVARIANT | per-setresgid: all cap_* fields unchanged. |
| `no_supplementary_mutation` | INVARIANT | per-setresgid: group_info pointer unchanged. |
| `no_uid_mutation` | INVARIANT | per-setresgid: ruid/euid/suid/fsuid unchanged. |
| `dumpable_on_egid_change` | INVARIANT | per-setresgid: EGID changed ⟹ dumpable = SUID_DUMP_DISABLE. |
| `kgid_valid_or_einval` | INVARIANT | per-setresgid: any non-(-1) make_kgid INVALID ⟹ EINVAL. |
| `all_or_nothing` | INVARIANT | per-setresgid: commit applies all three or none. |

### Layer 2: TLA+

`kernel/setresgid.tla`:
- States: (rgid, egid, sgid, fsgid, group_info, ruid, euid, suid, fsuid, cap_e, cap_p, cap_b, cap_i, cap_a, dumpable, nnp).
- Actions: setresgid_priv, setresgid_unpriv, setresgid_keep_gid_one_field, setresgid_fail.
- Properties:
  - `safety_minus_one_keep` — arg == -1 ⟹ field unchanged.
  - `safety_unpriv_no_escalation` — unpriv: each non-(-1) ∈ {rgid, egid, sgid}.
  - `safety_priv_unrestricted` — priv: any kgid accepted.
  - `safety_fsgid_eq_egid` — post: fsgid' = egid'.
  - `safety_no_cap_change` — cap_e'=cap_e ∧ cap_p'=cap_p ∧ cap_b'=cap_b ∧ cap_i'=cap_i ∧ cap_a'=cap_a.
  - `safety_no_uid_change` — ruid'=ruid ∧ euid'=euid ∧ suid'=suid ∧ fsuid'=fsuid.
  - `safety_no_supplementary_change` — group_info'=group_info.
  - `safety_dumpable_on_egid_change` — egid' ≠ egid ⟹ dumpable'=SUID_DUMP_DISABLE.
  - `safety_failure_no_mutation` — error return ⟹ state unchanged.
  - `safety_atomicity` — three-tuple update is atomic.
  - `liveness_terminates` — setresgid returns in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setresgid` post: success ⟹ (gid', egid', sgid', fsgid') per spec; failure ⟹ cred unchanged | `Setresgid::do_setresgid` |
| `prepare_creds` / `commit_creds` / `abort_creds` lifecycle | `Cred::prepare`/`commit`/`abort` |
| `make_kgid` post: INVALID_KGID iff no mapping | `GidMap::make_kgid` |
| `no_cap_mutation` post: pre.cap_* == post.cap_* | `Setresgid::do_setresgid` |

### Layer 4: Verus / Creusot functional

Per-`setresgid(2)` man-page equivalence. POSIX.1e draft (saved-set IDs) verified. LTP `setresgid01..setresgid04` pass. Joint verification with `setresuid(2)` (parallel structure, distinct cred fields).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setresgid(2)` reinforcement:

- **Per-cred RCU-COW** — defense against per-torn-cred update; failure path commits nothing.
- **Per-(-1) keep-semantics strict** — defense against per-conflation of keep vs zero-gid.
- **Per-CAP_SETGID strict** — defense against per-unprivileged escalation.
- **Per-gid_map validation per-field** — defense against per-invalid-kgid for each non-(-1) arg.
- **Per-no-cap-mutation** — defense against per-conflation with setresuid; setresgid never touches capabilities.
- **Per-supplementary-untouched** — defense against per-setgroups-implicit; group_info requires explicit `setgroups()`.
- **Per-dumpable cleared** — defense against per-coredump / per-ptrace-attach on setgid binary after EGID change.
- **Per-NNP** — defense against per-escalation pipelined with subsequent setuid/execve.
- **Per-all-or-nothing commit** — defense against per-partial-update.
- **Per-fsgid-follows-egid** — defense against per-fsgid-stuck-at-old-value (file-group-bypass).
- **Per-no-UID-leak** — defense against per-cross-cred-field mutation bug.
- **Per-LSM hook** — defense via SELinux/AppArmor/Yama veto on LSM_SETID_RES.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at setresgid entry** — randomizes kernel stack offset; defeats stack-spray and ret2dir attacks on the cred path. Although setresgid is less commonly targeted than setresuid, randomization is uniform across the syscall surface.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Gid line (4 fields: real, effective, saved, fs) restricted; setresgid-induced changes are not observable to other users. Reconnaissance of group-state across setgid binaries is denied.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — `setresgid()` inside a grsec chroot honors only the chroot's user-ns. The gid_map is sealed at chroot entry; setresgid cannot reference host GIDs that are not mapped.
- **GRKERNSEC_AUDIT_GROUP on setgid family** — every `setresgid()` call from audited groups is logged with old/new (rgid, egid, sgid). The drop-and-restore pattern (`setresgid(-1, gid, -1)` followed by `setresgid(-1, 0, -1)`) is logged in full. Combined with setresuid audit, the full credential-mutation timeline of each setuid+setgid binary is recorded.
- **CAP_SETGID strict** — grsec policy can globally deny CAP_SETGID; setresgid() reduces to the unprivileged path for all callers. Setgid binaries cannot use the privileged path to overwrite SGID, preventing permanent group-privilege escalation.
- **file-cap-strip + securebits SECBIT_KEEP_CAPS** — grsec strips file-caps on filesystem mount. SECBIT_KEEP_CAPS is irrelevant to setresgid (no cap mutation occurs) but is preserved across setresgid; grsec audits SECBIT use generally.
- **no_new_privs across setuid/setgid** — NNP=1 is preserved across setresgid. Although setresgid does not raise caps directly, NNP+setresgid prevents subsequent setuid-execve from raising EUID. Grsec enforces NNP-by-default for unprivileged user-ns.
- **Ambient / inheritable cap evolution rules** — `setresgid()` does NOT touch capability sets at all; the invariants are preserved trivially:
  - `cap_ambient ⊆ cap_inheritable ∩ cap_permitted` (unchanged).
  - cap_bounding, cap_inheritable, cap_ambient unchanged (no RUID-leaves-root analog for GIDs).
- Grsec verifies the no-mutation property explicitly via audit-comparison of pre/post cred. Any drift is logged as a kernel-integrity violation.
- **PAX_USERCOPY on cred copy** — copy of cred fields is bounds-checked against `struct cred` slab; defeats heap-spray on cred slab.
- **GRKERNSEC_HARDEN_TTY on EGID transitions** — TTY hijack mitigations active across setresgid; raising EGID to a privileged group (e.g., `tty`, `disk`) is denied or audited.
- **Per-dumpable enforced + ptrace gated** — grsec layered on `mm.dumpable = SUID_DUMP_DISABLE`; ptrace attach denied across the drop-restore cycle for setgid binaries.
- **Per-supplementary-group-list integrity** — grsec verifies that setresgid does not alter group_info; any drift is logged. Combined with audit of `setgroups()`, supplementary-group state is fully traceable.
- **Per-CHROOT_NO_SETGID** — grsec can deny CAP_SETGID entirely inside a chroot, forcing setresgid to operate exclusively in the unprivileged path. Setgid binaries inside chroot become effectively read-only with respect to group state.
- **Per-thread cred consistency** — grsec audits per-thread (rgid, egid, sgid) drift caused by NPTL-sync edge cases; divergence is logged as a kernel-integrity event.

## Open Questions

- (none at this Tier-5 level; explicit three-tuple semantics admit no ambiguity.)

## Out of Scope

- `setgid(2)` (Tier-5 separate doc — single-arg, all-three-on-priv).
- `setegid(2)` (Tier-5 separate doc — effective only).
- `setregid(2)` (Tier-5 separate doc — real + effective with swap semantics).
- `setfsgid(2)` (Tier-5 separate doc — file-system GID).
- `getresgid(2)` (Tier-5 separate doc — returns the triple).
- `setgroups(2)` / `getgroups(2)` (Tier-5 separate docs — supplementary group list).
- File capabilities and setuid-execve cred transition (Tier-3 in `security/commoncap.md` and `fs/exec.md`).
- Securebits prctl interface (Tier-5 in `prctl.md`).
- User-namespace gid_map semantics (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.
