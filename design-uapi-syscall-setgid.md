---
title: "Tier-5 syscall: setgid(2) — syscall 106"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setgid(2)` sets the calling task's group IDs to `gid`. Semantics depend on privilege:

- **Privileged caller** (holds `CAP_SETGID` in caller's user-ns OR EUID==0): sets `real`, `effective`, **and** `saved` GIDs all to `gid`. (POSIX `_POSIX_SAVED_IDS` — the only path that overwrites SGID.)
- **Unprivileged caller**: `gid` must be one of `{RGID, EGID, SGID}`; only the `effective` GID is changed (and `fsgid` follows `egid`). The real/saved GIDs are unchanged. Used to drop group privileges in setgid binaries.

Critical side effects:
- **Dumpable cleared**: when EGID changes, `mm.dumpable` may be cleared (SUID_DUMP_DISABLE), disabling core dumps and `/proc/<pid>/mem` access.
- **No file-cap strip on setgid**: unlike `setuid`, the kernel does NOT clear capabilities on EGID transitions. Capabilities track UID, not GID.
- **Supplementary groups untouched**: `setgid()` does NOT modify `group_info`; use `setgroups(2)` for that.
- **`no_new_privs` propagation**: NNP is preserved across setgid.

### Acceptance Criteria

- [ ] AC-1: CAP_SETGID-holding caller `setgid(1000)`: RGID, EGID, SGID, FSGID all become 1000.
- [ ] AC-2: Setgid binary (EGID=0, RGID=1000, SGID=0) with CAP_SETGID calls `setgid(1000)`: RGID=EGID=SGID=1000 (permanent drop).
- [ ] AC-3: Unprivileged caller (RGID=1000, EGID=1000, SGID=1000) calls `setgid(2000)`: returns EPERM, cred unchanged.
- [ ] AC-4: Unprivileged caller calls `setgid(SGID)`: only EGID changes to SGID; RGID and SGID unchanged.
- [ ] AC-5: `setgid()` does NOT modify cap_effective / cap_permitted / cap_bounding / cap_inheritable / cap_ambient.
- [ ] AC-6: `setgid()` does NOT modify supplementary groups (`group_info`).
- [ ] AC-7: EGID transition: mm.dumpable cleared to SUID_DUMP_DISABLE.
- [ ] AC-8: `gid` not in gid_map: returns EINVAL.
- [ ] AC-9: NNP=1 caller: setgid succeeds; no capability raise possible (irrelevant — setgid never raises caps).
- [ ] AC-10: LSM denies via security_task_fix_setgid: returns LSM-provided errno.
- [ ] AC-11: After successful setgid: getgid()/getegid() reflect new values; /proc/self/status:Gid lines updated.
- [ ] AC-12: setgid in chroot/user-ns: kgid mapping respects active user_ns.
- [ ] AC-13: setgid does NOT affect EUID/RUID/SUID/FSUID.
- [ ] AC-14: i386 syscall 46 returns clamped 16-bit; syscall 214 accepts full 32-bit.

### Architecture

```rust
#[syscall(nr = 106, abi = "sysv")]
pub fn sys_setgid(gid: gid_t) -> i32 {
    Setgid::do_setgid(gid)
}
```

`Setgid::do_setgid(gid) -> Result<(), Errno>`:
1. let old = current_cred();                       // RCU read
2. let user_ns = current_user_ns();
3. let kgid = GidMap::make_kgid(user_ns, gid);
4. if !kgid.is_valid() { return Err(EINVAL); }
5. /* prepare COW cred */
6. let mut new = Cred::prepare(old)?;
7. /* Decide privileged vs unprivileged path */
8. let privileged = ns_capable(user_ns, CAP_SETGID);
9. if privileged {
10.    new.gid = kgid;
11.    new.egid = kgid;
12.    new.sgid = kgid;
13.    new.fsgid = kgid;
14. } else if kgid == old.gid || kgid == old.egid || kgid == old.sgid {
15.    new.egid = kgid;
16.    new.fsgid = kgid;
17. } else {
18.    Cred::abort(new);
19.    return Err(EPERM);
20. }
21. /* LSM hook */
22. Security::task_fix_setgid(&new, &old, LSM_SETID_ID)?;
23. /* Dumpable */
24. if new.egid != old.egid {
25.    current_mm().set_dumpable(SUID_DUMP_DISABLE);
26. }
27. /* Commit — no capability mutation for setgid */
28. Cred::commit(new);
29. Ok(())

`GidMap::make_kgid(ns, gid) -> kgid_t`:
1. for entry in ns.gid_map.extents() {
2.     if gid ∈ [entry.first, entry.first + entry.count) {
3.         return kgid_t { val: entry.lower_first + (gid - entry.first) };
4.     }
5. }
6. return INVALID_KGID;

### Out of Scope

- `setegid(2)` (Tier-5 separate doc — effective-only set, no SGID overwrite).
- `setregid(2)` (Tier-5 separate doc — real + effective with swap semantics).
- `setresgid(2)` (Tier-5 separate doc — explicit real/effective/saved triple).
- `setfsgid(2)` (Tier-5 separate doc — file-system GID).
- `setgroups(2)` / `getgroups(2)` (Tier-5 separate docs — supplementary group list).
- File capabilities and setuid-execve cred transition (Tier-3 in `security/commoncap.md` and `fs/exec.md`).
- Securebits prctl interface (Tier-5 in `prctl.md`).
- User-namespace gid_map semantics (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.

### signature

```c
int setgid(gid_t gid);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `gid` | `gid_t` | in | New group ID (interpreted in caller's user namespace). |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. Cred updated. |
| `-1` | Failure; `errno` set. No cred changes are committed. |

### errors

| errno | When |
|---|---|
| `EINVAL` | `gid` is not a valid value in the caller's user namespace (no mapping in `gid_map`). |
| `EPERM` | Caller lacks `CAP_SETGID` AND `gid` is not in {RGID, EGID, SGID}. |

(Note: `setgid()` does NOT return `EAGAIN` for RLIMIT_NPROC — that limit is per-user (UID), not per-group.)

### abi surface

```text
__NR_setgid (x86_64)    = 106
__NR_setgid (i386)      = 46      /* 16-bit gid_t variant */
__NR_setgid32 (i386)    = 214     /* 32-bit gid_t — preferred */
__NR_setgid (arm64)     = 144     /* generic-syscall */
__NR_setgid (generic)   = 144

typedef __kernel_gid32_t  gid_t;  /* unsigned int — 32-bit */

/* securebits — note these affect setuid, NOT setgid; included for context */
SECBIT_NO_SETUID_FIXUP    = 1 << 2  /* irrelevant to setgid */
SECBIT_KEEP_CAPS          = 1 << 4  /* irrelevant to setgid */

/* dumpable cleared on EGID change */
SUID_DUMP_DISABLE = 0
```

### compatibility contract

REQ-1: Syscall number is **106** on x86_64; **144** on arm64 / generic. ABI-stable.

REQ-2: `gid` is interpreted in `current_user_ns()`. The kernel converts to `kgid_t` via `make_kgid(current_user_ns(), gid)`. If the result is `INVALID_GID` (not in gid_map), return `EINVAL`.

REQ-3: Privileged path (`ns_capable(current_user_ns(), CAP_SETGID)`):
- new_cred.gid = new_cred.egid = new_cred.sgid = new_cred.fsgid = kgid.

REQ-4: Unprivileged path: `kgid` must equal one of `{cred.gid, cred.egid, cred.sgid}`. Sets only `new_cred.egid = new_cred.fsgid = kgid`. RGID/SGID unchanged. If `kgid ∉ {rgid, egid, sgid}`, return `EPERM`.

REQ-5: `setgid()` does NOT modify any capability fields (cap_effective, cap_permitted, cap_bounding, cap_inheritable, cap_ambient). Capabilities are UID-anchored, not GID-anchored.

REQ-6: `setgid()` does NOT modify supplementary group list (`cred->group_info`). Use `setgroups(2)`.

REQ-7: `commit_creds(new)`:
- Atomically swap `current->real_cred` and `current->cred` pointers (RCU-COW).
- Free old cred via RCU.

REQ-8: If EGID changes value (old.egid != new.egid), set `mm.dumpable = SUID_DUMP_DISABLE` unless the process is currently being traced. (Same dumpable semantics as setuid; protects setgid binaries from /proc/<pid>/mem read.)

REQ-9: `setgid()` does NOT affect EUID/RUID/SUID/FSUID, fsuid, pid/sid/pgid, or supplementary groups.

REQ-10: `setgid()` does NOT affect `no_new_privs`. NNP, once set, persists across setgid.

REQ-11: `setgid()` mutates per-thread cred only. NPTL synchronizes across threads via library wrapper that issues `setgid` on each thread.

REQ-12: `setgid()` is a CRED-MUTATING syscall: failure leaves no observable side effects. `prepare_creds()` ⟶ mutate ⟶ check ⟶ commit_creds() or abort_creds().

REQ-13: i386 16-bit (syscall 46) accepts 16-bit gid; truncation if higher bits set is rejected with EINVAL.

REQ-14: LSM hook: `security_task_fix_setgid(new, old, LSM_SETID_ID)` is invoked before commit; SELinux/AppArmor/Smack/Yama may veto.

REQ-15: User-namespace gid_map enforcement: `setgid(0)` in a user-ns with map `0 1000 1` maps kgid to host gid 1000; from outside the ns, the task's kgid is 1000.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `unprivileged_gid_in_rgssaved` | INVARIANT | per-setgid unprivileged: kgid ∈ {rgid, egid, sgid} or EPERM. |
| `privileged_sets_all_three` | INVARIANT | per-setgid privileged: rgid=egid=sgid=kgid post-commit. |
| `cred_cow_atomicity` | INVARIANT | per-setgid: failure ⟹ no cred mutation observable. |
| `no_cap_mutation` | INVARIANT | per-setgid: cap_e, cap_p, cap_b, cap_i, cap_a unchanged. |
| `no_supplementary_mutation` | INVARIANT | per-setgid: group_info pointer unchanged. |
| `no_uid_mutation` | INVARIANT | per-setgid: ruid/euid/suid/fsuid unchanged. |
| `dumpable_cleared_on_egid_change` | INVARIANT | per-setgid: EGID changed ⟹ dumpable = SUID_DUMP_DISABLE. |
| `kgid_valid_or_einval` | INVARIANT | per-setgid: make_kgid returns INVALID ⟹ EINVAL. |

### Layer 2: TLA+

`kernel/setgid.tla`:
- States: (rgid, egid, sgid, fsgid, group_info, ruid, euid, suid, cap_e, cap_p, cap_b, cap_i, cap_a, dumpable, nnp).
- Actions: setgid_priv, setgid_unpriv, setgid_fail.
- Properties:
  - `safety_unpriv_no_escalation` — unpriv path: egid' ∈ {rgid, egid, sgid}.
  - `safety_priv_all_three` — priv path: rgid'=egid'=sgid'.
  - `safety_no_cap_change` — cap_e' = cap_e ∧ cap_p' = cap_p ∧ cap_b' = cap_b ∧ cap_i' = cap_i ∧ cap_a' = cap_a.
  - `safety_no_uid_change` — ruid'=ruid ∧ euid'=euid ∧ suid'=suid ∧ fsuid'=fsuid.
  - `safety_no_supplementary_change` — group_info'=group_info.
  - `safety_dumpable_on_egid_change` — egid' ≠ egid ⟹ dumpable'=SUID_DUMP_DISABLE.
  - `safety_failure_no_mutation` — error return ⟹ state unchanged.
  - `liveness_terminates` — setgid returns in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setgid` post: success ⟹ cred reflects spec, failure ⟹ cred unchanged | `Setgid::do_setgid` |
| `prepare_creds` / `commit_creds` / `abort_creds` lifecycle | `Cred::prepare`/`commit`/`abort` |
| `make_kgid` post: INVALID_KGID iff no mapping | `GidMap::make_kgid` |

### Layer 4: Verus / Creusot functional

Per-`setgid(2)` man-page equivalence. POSIX.1-2008 `setgid` semantics (saved-set ID behavior). LTP `setgid01..setgid03` pass. Joint verification with `credentials(7)`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setgid(2)` reinforcement:

- **Per-cred RCU-COW** — defense against per-torn-cred update; failure path commits nothing.
- **Per-CAP_SETGID strict** — defense against per-unprivileged escalation; EPERM if !ns_capable.
- **Per-gid_map validation** — defense against per-invalid-kgid; EINVAL if not mapped in current_user_ns.
- **Per-no-cap-mutation** — defense against per-conflation with setuid; capabilities track UID only.
- **Per-supplementary-untouched** — defense against per-setgroups-implicit; supplementary list requires explicit `setgroups()`.
- **Per-dumpable cleared on EGID change** — defense against per-coredump / per-ptrace-attach on setgid binary.
- **Per-LSM hook** — defense via SELinux/AppArmor/Yama veto.
- **Per-i386 16-bit clamped** — defense against per-gid-truncation legacy bug; 32-bit syscall preferred.
- **Per-no-UID-leak** — defense against per-cross-cred-field mutation bug.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at setgid entry** — randomizes kernel stack offset; defeats stack-spray and ret2dir attacks aimed at the cred path.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Gid lines (Real, Effective, Saved, FS) restricted; setgid-induced changes are not observable to other users.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — `setgid()` inside a grsec chroot honors only the chroot's user-ns; cannot escalate by referencing host GIDs. gid_map is sealed at chroot entry.
- **GRKERNSEC_AUDIT_GROUP on setgid** — every `setgid()` call from audited groups is logged with old/new GID and the privileged/unprivileged path taken. Combined with audit of setuid, full credential-mutation history is recorded.
- **CAP_SETGID strict** — grsec policy can globally deny CAP_SETGID; setgid() returns EPERM for any non-{rgid,egid,sgid} target. Setgid binaries cannot raise EGID at all under deny-CAP_SETGID policy.
- **file-cap-strip + securebits SECBIT_KEEP_CAPS** — grsec strips file-caps on filesystem mount. SECBIT_KEEP_CAPS is irrelevant to setgid (setgid doesn't strip caps) but is preserved across setgid-execve; grsec audits SECBIT use generally.
- **no_new_privs across setuid/setgid** — NNP is preserved across setgid. Although setgid does not raise caps directly, NNP+setgid prevents subsequent setuid-execve from raising EUID. Grsec enforces NNP-by-default for unprivileged user-ns.
- **Ambient / inheritable cap evolution rules** — `setgid()` does NOT touch capability sets at all; the ambient ⊆ inheritable ∩ permitted invariant is preserved trivially. Grsec verifies the no-mutation property explicitly via audit-comparison of pre/post cred.
- **PAX_USERCOPY on cred copy** — copy of cred fields is bounds-checked against `struct cred` slab object; defeats heap-spray targeting cred slab.
- **GRKERNSEC_HARDEN_TTY on setgid binary** — TTY hijack mitigations (TIOCSTI denial) active across setgid; setgid raising EGID to a privileged group (e.g., `tty`) cannot be exploited to push input.
- **Per-dumpable enforced + ptrace gated** — grsec layered on top of `mm.dumpable = SUID_DUMP_DISABLE`: ptrace attach denied after EGID change to a privileged group.
- **Per-supplementary-group-list integrity** — grsec verifies that setgid does not alter group_info; any drift is logged as a kernel-integrity violation.
- **Per-thread cred consistency** — grsec audits per-thread egid drift caused by NPTL-sync edge cases.
- **CHROOT_NO_SETGID** — grsec can deny CAP_SETGID entirely inside a chroot, forcing setgid binaries to fail-closed within constrained environments.

