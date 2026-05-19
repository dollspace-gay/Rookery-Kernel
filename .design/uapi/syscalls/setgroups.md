# Tier-5 syscall: setgroups(2) — syscall 116

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/groups.c (SYSCALL_DEFINE2(setgroups), set_groups, groups_from_user)
  - kernel/user_namespace.c (proc_setgroups_show, may_setgroups)
  - include/linux/cred.h (struct cred::group_info)
  - arch/x86/entry/syscalls/syscall_64.tbl (116 common setgroups)
-->

## Summary

`setgroups(2)` replaces the calling process's supplementary group list with a user-supplied array of GIDs. It requires `CAP_SETGID` in the caller's user namespace, and in a non-init user namespace it is additionally gated by `/proc/$pid/setgroups` — a one-shot write-only file that defaults to "allow" but can be set to "deny" before any `gid_map` write to prevent groups-based privilege amplification on the host (the historic `setgroups`/userns CVE-class fix).

Critical for: container init (when allowed), `su`/`login`/`sudo` cred construction, multi-tenant daemons dropping groups, user-namespace setup ceremony. Hardened distros forbid setgroups in unprivileged userns by default.

## Signature

```c
int setgroups(size_t size, const gid_t *list);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `size` | `size_t` | in | Array length; `0` ⟹ clear supplementary groups. `<= NGROUPS_MAX`. |
| `list` | `const gid_t *` | in | Caller-supplied GID array in caller's userns view. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `size > NGROUPS_MAX`. |
| `EFAULT` | `list` not readable. |
| `EPERM` | Caller lacks `CAP_SETGID` in current userns, OR current userns has `/proc/$pid/setgroups` set to "deny", OR a non-init userns without a `gid_map` configured. |
| `ENOMEM` | group_info allocation failed. |

## ABI surface

```text
__NR_setgroups  (x86_64) = 116
__NR_setgroups  (arm64)  = 159
__NR_setgroups  (riscv)  = 159
__NR_setgroups  (i386)   = 81

/* NGROUPS_MAX = 65536. */
```

## Compatibility contract

REQ-1: Syscall number is **116** on x86_64. ABI-stable.

REQ-2: Permission gates (all required):
- `ns_capable(current_user_ns(), CAP_SETGID)`. Else `-EPERM`.
- `may_setgroups()` in caller's userns. Else `-EPERM`.

REQ-3: `may_setgroups()`:
- In `init_user_ns`: always allowed (subject to CAP_SETGID).
- In a child user_ns:
  - If `/proc/$pid/setgroups` written "deny" (sticky), reject.
  - Else if `gid_map` not yet configured: reject (must map gids before setting groups).
  - Else: allow.

REQ-4: `size > NGROUPS_MAX` ⟹ `-EINVAL`.

REQ-5: `groups_from_user`:
- Allocate `group_info` with `ngroups = size`.
- `copy_from_user(kbuf, list, size * sizeof(gid_t))`.
- For each `gid` in `kbuf`: `make_kgid(current_user_ns(), gid)`; if unmapped ⟹ `-EINVAL`.
- Store kgids in `group_info->gid[]`.
- `groups_sort(group_info)` — ascending.

REQ-6: Install via `set_groups(new_cred, group_info)` + `commit_creds(new_cred)`.

REQ-7: The new cred is RCU-published; concurrent `getgroups(2)` observes either old or new atomically.

REQ-8: `size == 0`: clears supplementary groups (group_info with ngroups=0); `getgroups(0)` then returns 0.

REQ-9: 32-bit compat: 16-bit form `setgroups(2)` (`__NR_setgroups16 = 81` on i386); 32-bit form `setgroups32(2)` (= 206). Kernel implements both.

REQ-10: `/proc/$pid/setgroups`:
- Writable by anyone in the process when it has CAP_SETGID OR before gid_map is configured (privilege seeded).
- Sticky-deny: once "deny" written, cannot be flipped back to "allow" without re-creating the userns.
- Read-only after first gid_map write.

REQ-11: Interaction with `clone(CLONE_NEWUSER)`: in unprivileged userns creation, default setgroups state is "allow" but must be flipped to "deny" before writing `gid_map` for the historic safety fix; this is the well-known userns-setgroups CVE class.

REQ-12: SECCOMP: setgroups commonly allow-listed in container runtimes (with userns) but blocked outright in many seccomp profiles for unprivileged containers.

## Acceptance Criteria

- [ ] AC-1: Root in init_ns: `setgroups(3, [10,20,30])` returns 0; `getgroups` returns [10,20,30].
- [ ] AC-2: `setgroups(0, NULL)` returns 0; `getgroups(0)` returns 0.
- [ ] AC-3: `setgroups(70000, ...)`: -EINVAL.
- [ ] AC-4: Non-root: -EPERM.
- [ ] AC-5: In userns with setgroups=deny: -EPERM.
- [ ] AC-6: In userns without gid_map: -EPERM.
- [ ] AC-7: list points to unmapped fault page: -EFAULT.
- [ ] AC-8: GID not in caller's userns gid_map: -EINVAL.
- [ ] AC-9: Returned list sorted ascending.
- [ ] AC-10: Concurrent getgroups observes atomic switch (no torn read).
- [ ] AC-11: After successful setgroups, /proc/$pid/setgroups becomes read-only.

## Architecture

```rust
#[syscall(nr = 116, abi = "sysv")]
pub fn sys_setgroups(size: usize, list: UserPtr<u32>) -> isize {
    SetGroups::do_setgroups(size, list)
}
```

`SetGroups::do_setgroups(size, list) -> isize`:
1. if size > NGROUPS_MAX { return -EINVAL; }
2. /* CAP_SETGID check. */
3. if !ns_capable(current_user_ns(), CAP_SETGID) { return -EPERM; }
4. /* userns setgroups gate. */
5. if !may_setgroups() { return -EPERM; }
6. /* Allocate group_info. */
7. let mut gi = GroupInfo::alloc(size).ok_or(-ENOMEM)?;
8. /* Copy user GIDs. */
9. if size > 0 {
10.  let mut kbuf = vec![0u32; size];
11.  list.copy_in_slice(&mut kbuf).map_err(|_| EFAULT)?;
12.  for (i, gid) in kbuf.iter().enumerate() {
13.    let kgid = make_kgid(current_user_ns(), *gid);
14.    if !kgid_valid(kgid) { return -EINVAL; }
15.    gi.gid[i] = kgid;
16.  }
17.  groups_sort(&mut gi);
18. }
19. /* Install. */
20. let new = prepare_creds()?;
21. set_groups(&mut new, gi);
22. commit_creds(new);
23. return 0;

`may_setgroups() -> bool`:
1. let ns = current_user_ns();
2. if ns.is_init() { return true; }
3. /* sticky-deny */
4. if ns.setgroups_state.load(Acquire) == NS_SETGROUPS_DENY { return false; }
5. /* gid_map must be configured first */
6. if ns.gid_map.is_empty() { return false; }
7. return true;

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_bounded` | INVARIANT | size > NGROUPS_MAX ⟹ EINVAL. |
| `cap_setgid_required` | INVARIANT | missing CAP_SETGID ⟹ EPERM, no cred change. |
| `may_setgroups_required` | INVARIANT | userns deny / no gid_map ⟹ EPERM. |
| `kgid_validity` | INVARIANT | every input gid maps via gid_map; else EINVAL. |
| `sorted_after_install` | INVARIANT | group_info.gid sorted ascending post-call. |
| `cred_atomicity` | INVARIANT | new cred fully built before commit_creds. |

### Layer 2: TLA+

`kernel/setgroups.tla`:
- States: per-task cred, per-userns gid_map, per-userns setgroups_state.
- Properties:
  - `safety_cap_setgid_gate` — no cred mutation without CAP_SETGID.
  - `safety_userns_deny_sticky` — deny once set, never re-allowed.
  - `safety_gid_map_first` — gid_map must precede setgroups in unprivileged userns.
  - `safety_kgid_valid` — every kgid in group_info has mapping.
  - `liveness_call_returns` — bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setgroups` post: success ⟹ current.cred.group_info reflects input | `SetGroups::do_setgroups` |
| `do_setgroups` post: EPERM iff !CAP_SETGID ∨ !may_setgroups | `SetGroups::do_setgroups` |
| `do_setgroups` post: every gid via make_kgid valid | `SetGroups::do_setgroups` |
| `may_setgroups` post: returns false for deny or unmapped userns | `may_setgroups` |

### Layer 4: Verus/Creusot functional

Per-`setgroups(2)` man page + Documentation/admin-guide/namespaces/user_namespaces.7 setgroups gate semantic equivalence.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setgroups(2)` reinforcement:

- **Per-CAP_SETGID gate** — defense against per-unprivileged group amplification.
- **Per-may_setgroups (deny + gid_map) gate** — defense against per-userns setgroups-CVE class.
- **Per-make_kgid validity check** — defense against per-unmapped-gid host injection.
- **Per-groups_sort post-install** — defense against per-unsorted oracle.
- **Per-cred atomic commit via prepare_creds + commit_creds** — defense against per-partial-cred state.
- **Per-NGROUPS_MAX cap** — defense against per-ngroups-DoS.

## Grsecurity / PaX surface

- **PaX UDEREF on copy_from_user(list)** — defense against per-list kernel-deref bug; SMAP forced.
- **setgroups CAP_SETGID + uid_map setgroups gate** — grsec enforces the canonical fix: in any non-init user_ns the setgroups state file MUST be deny-flipped before writing `gid_map`, otherwise the userns rejects setgroups regardless of state. This closes the CVE-2014-8989-class group-membership amplification: an unprivileged user inside a userns cannot retain a host gid via supplementary groups.
- **PAX_USERCOPY_HARDEN on group list copy** — bounded `size * sizeof(u32)`; whitelisted slab.
- **PAX_REFCOUNT on group_info / cred refcounts** — defense against per-refcount-overflow UAF.
- **GRKERNSEC_AUDIT_GROUPS** — every setgroups call logged with uid + new gid list; defense against silent group-elevation in compromise scenario.
- **GRKERNSEC_CHROOT_CAPS interaction** — in a chroot, even root cannot use setgroups to add gids outside chroot's allowed range; grsec_chroot_setgroups_allowed sysctl gates.
- **Per-cred snapshot under RCU** — defense against per-cred-swap torn install.
- **GRKERNSEC_NO_RBAC_GROUPS** — when RBAC policy fixes the supplementary group set, setgroups is denied with EPERM; RBAC group set wins.
- **Per-userns setgroups sticky-deny enforced kernel-wide** — defense against per-state-flip race; the deny write uses a CAS to NS_SETGROUPS_DENY and refuses any subsequent ALLOW write.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `getgroups(2)` reader (covered in `getgroups.md`).
- User-namespace gid_map setup (covered in `userns.md`).
- POSIX initgroups(3) library wrapper.
- Implementation code.
