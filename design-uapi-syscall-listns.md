---
title: "Tier-5 syscall: listns(2) — forward-looking (ENOSYS in 7.1.0-rc2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`listns(2)` is a **proposed syscall** to enumerate namespaces visible from the calling task without walking `/proc/<pid>/ns/<type>` symlinks. The motivation is twofold: (1) containers running with a strict `proc` mount mask cannot read `/proc/<pid>/ns/*` for tasks they otherwise have access to via `pidfd`; (2) auditing and tooling want a single canonical syscall to enumerate the system's namespace topology rather than scanning `/proc` and racing against task death. The current 7.1.0-rc2 baseline **does not** ship this syscall: `__NR_listns = 470` is reserved in the Rookery design and the entry trampoline returns `-ENOSYS`. Userspace MUST continue to use `/proc/<pid>/ns/*` + nsfs ioctls (`NS_GET_USERNS`, `NS_GET_PARENT`, `NS_GET_NSTYPE`) until the syscall is wired in a later phase.

This document defines the contract Rookery commits to **if and when** `listns` lands, plus the explicit `ENOSYS` behavior for now.

Critical for: namespace-aware audit (auditd, falco), pid-only container introspection (`runc`, `crun` over `pidfd`), nsenter without `/proc` access, and the planned `lsns(8)` syscall-backed mode.

### Acceptance Criteria

- [ ] AC-1: In 7.1.0-rc2, `syscall(__NR_listns, ...)` returns `-1` with `errno == ENOSYS`.
- [ ] AC-2: After implementation: `listns(pidfd_self, buf, 8, 0)` returns 8 entries on a standard task.
- [ ] AC-3: After implementation: `count = 0, buf = NULL` returns 0 (probe path).
- [ ] AC-4: After implementation: `flags = LISTNS_F_RESERVED` returns `EINVAL`.
- [ ] AC-5: After implementation: cross-userns target without `CAP_SYS_PTRACE` returns `EPERM`.
- [ ] AC-6: After implementation: `ns_id` matches `stat(/proc/.../ns/<t>).st_ino`.
- [ ] AC-7: After implementation: emission order is MNT, CGROUP, UTS, IPC, USER, PID, NET, TIME.
- [ ] AC-8: After implementation: truncation returns `count` and userspace can detect via `count == 8` heuristic.
- [ ] AC-9: After implementation: dead target (`ESRCH`) reported promptly.
- [ ] AC-10: After implementation: audit record emitted.

### Architecture

```rust
#[syscall(nr = 470, abi = "sysv")]
pub fn sys_listns(
    pidfd: i32,
    buf: UserPtr<NsInfo>,
    count: u32,
    flags: u32,
) -> isize {
    /* Forward-looking: not implemented in 7.1.0-rc2. */
    -ENOSYS as isize
}
```

When implemented:

`Listns::do_listns(pidfd, buf, count, flags) -> isize`:
1. if (flags & LISTNS_F_RESERVED) != 0 { return -EINVAL; }
2. if !buf.is_null() && count == 0 { return -EINVAL; }
3. let target = pidfd_get_task(pidfd).ok_or(ESRCH)?;
4. if !ptr::eq(&*target, current()) {
5.   if !ns_capable_for(target.cred.user_ns, CAP_SYS_PTRACE) { return -EPERM; }
6. }
7. let _g = task_lock(&target);
8. let nsp = target.nsproxy.clone();
9. let mut out = [NsInfo::default(); 8];
10. let n = Listns::snapshot_canonical(&nsp, &target.cred, &mut out);
11. let to_copy = min(n, count as usize);
12. if !buf.is_null() {
13.   buf.write_array(&out[..to_copy]).map_err(|_| EFAULT)?;
14. }
15. /* Audit */
16. audit_log(AUDIT_NAMESPACE_LIST, target.tgid, current_tgid(), to_copy);
17. to_copy as isize

`Listns::snapshot_canonical(nsp, cred, out) -> usize`:
1. let mut k = 0;
2. macro emit(typ, ns) { out[k] = ns.to_info(typ); k += 1; }
3. emit(NS_TYPE_MNT,    nsp.mnt_ns);
4. emit(NS_TYPE_CGROUP, nsp.cgroup_ns);
5. emit(NS_TYPE_UTS,    nsp.uts_ns);
6. emit(NS_TYPE_IPC,    nsp.ipc_ns);
7. emit(NS_TYPE_USER,   cred.user_ns);
8. emit(NS_TYPE_PID,    nsp.pid_ns_for_children);
9. emit(NS_TYPE_NET,    nsp.net_ns);
10. emit(NS_TYPE_TIME,  nsp.time_ns);
11. k
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `enosys_in_baseline` | INVARIANT | per-7.1.0-rc2: sys_listns always returns -ENOSYS. |
| `canonical_order` | INVARIANT | per-snapshot: emission order is MNT, CGROUP, UTS, IPC, USER, PID, NET, TIME. |
| `cap_check_before_copy` | INVARIANT | per-implemented: ptrace cap check precedes copy_to_user. |
| `truncation_bounded` | INVARIANT | per-implemented: writes ≤ min(n, count) entries. |
| `task_locked_during_snapshot` | INVARIANT | per-implemented: task_lock(target) held across snapshot read. |
| `nsid_matches_nsfs_inode` | INVARIANT | per-implemented: emitted ns_id == nsfs inode of corresponding ns. |

### Layer 2: TLA+

`kernel/listns.tla`:
- States: per-task nsproxy, per-task creds, snapshot output.
- Properties:
  - `safety_enosys_baseline` — in 7.1.0-rc2, every call returns ENOSYS.
  - `safety_cap_gate_cross_userns` — cross-userns ⟹ cap check.
  - `safety_canonical_order` — output[i].ns_type matches canonical[i].
  - `safety_no_widen_caps` — listns success never grants any new capability.
  - `liveness_terminates` — O(1) over 8 fixed types.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_listns` post (baseline): returns -ENOSYS unconditionally | `sys_listns` (baseline) |
| `do_listns` post (implemented): success ⟹ to_copy ≤ count ∧ to_copy ≤ 8 | `Listns::do_listns` |
| `snapshot_canonical` post: out[..8] in canonical order | `Listns::snapshot_canonical` |
| `to_info` post: ns_id == nsfs inode of ns | `Namespace::to_info` |

### Layer 4: Verus / Creusot functional

- Baseline: probe call equals `errno == ENOSYS`.
- Implemented: equivalence to enumerating `/proc/<pid>/ns/*` symlinks, matching inode order — TBD pending implementation.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`listns(2)` reinforcement (forward-looking):

- **Per-CAP_SYS_PTRACE cross-userns gate** — defense against per-info-leak of containers' ns topology.
- **Per-pidfd target binding** — defense against per-PID-race; pidfd is stable.
- **Per-task_lock snapshot atomicity** — defense against per-mid-unshare split observations.
- **Per-canonical order** — defense against per-implementation-defined-order parsing bugs.
- **Per-`flags` reserved bit strict-zero** — defense against per-extension-field smuggling.
- **Per-no-cap-widening** — defense against per-cap-escalation via novel syscall.
- **Per-`ENOSYS` baseline semantics** — defense against per-EINVAL-confusion in libc probes.

## Grsecurity / PaX surface

- **Namespace-listing CAP gating (CAP_SYS_ADMIN in init_user_ns)** — grsec elevates the cross-task case from `CAP_SYS_PTRACE` to `CAP_SYS_ADMIN`-in-init_userns. In a container, `listns` on any task other than self returns `EPERM`. Defeats container reconnaissance.
- **GRKERNSEC_PROC namespace info-leak parity** — `listns` MUST NOT expose more information than `/proc/<pid>/ns/*` already does under the same `GRKERNSEC_PROC_USERGROUP` restrictions. If `/proc` access is denied by group policy, `listns` for that target returns `EPERM`.
- **GRKERNSEC_CHROOT_NSENTER blocks listns from chroot** — inside a grsec chroot, `listns` of any non-self target returns `EPERM` even with `CAP_SYS_ADMIN`. Removes a chroot-aware ns-discovery primitive.
- **PaX UDEREF on `buf` copy_to_user** — SMAP-guarded write; kernel-range `buf` rejected.
- **PAX_USERCOPY_HARDEN on the 8-entry write** — buffer copy uses whitelisted slab; fixed-size 8 * 32 byte payload bounded.
- **GRKERNSEC_AUDIT_NAMESPACE** — every successful listns audited with caller / target / cap bundle. Aligns with audit's `AUDIT_NAMESPACE_LIST` record.
- **PaX KERNEXEC neutral** — read-only path; no W^X concerns.
- **Per-reserved-flag strict-zero** — extension-field smuggling rejected (matches existing pidfd / setns hardening).
- **PaX KSPP no_new_privs neutral** — NNP unaffected; no cred change.
- **Anti-fingerprint** — canonical emission order means no implementation-version oracle from ordering differences.
- **Per-`-ENOSYS` is the only baseline outcome** — under grsec on 7.1.0-rc2, slot 470 cannot be hijacked to a wrong syscall handler; the table entry points to a sealed `sys_ni_syscall` thunk.

## Open Questions

- Q1: Should `pidfd == -1` mean "caller's namespaces" with no ptrace gate? Leaning yes for ergonomics, but punted to implementation phase.
- Q2: Add `LISTNS_F_REPORT_TOTAL` for total-count-on-truncation in v1, or v2? Defer.
- Q3: Should the syscall expose `ns_id` as `__kernel_dev_t`-style for nsfs filesystems with non-inode identifiers? Punted; current nsfs is single-fs.
- Q4: When implemented, should it gain its own LSM hook? Probably yes — `security_task_namespace_list`.

## Out of Scope

- `setns(2)` (Tier-5 separate doc — entering an existing ns).
- `unshare(2)` (Tier-5 separate doc — creating new ns).
- `pidfd_open(2)` (Tier-5 separate doc — target-task fd).
- nsfs internals and `NS_GET_*` ioctls (Tier-3 in `fs/nsfs.md`).
- `/proc/<pid>/ns/*` symlink resolution (Tier-3 in `fs/proc/ns.md`).
- Implementation code.

### signature

```c
int listns(int pidfd, struct ns_info *buf, unsigned int count,
           unsigned int flags);

struct ns_info {
    __u32  ns_type;     /* CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC | ... */
    __u32  flags;       /* future extensions; currently 0 */
    __u64  ns_id;       /* matches stat(/proc/.../ns/<t>).st_ino */
    __u64  user_ns_id;  /* owning user namespace id */
    __u64  parent_id;   /* parent ns id (PID/USER) or 0 */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pidfd` | `int` | in | Target task (via `pidfd_open`); `AT_FDCWD` reserved (currently rejected). |
| `buf` | `struct ns_info *` | out | User buffer to receive up to `count` namespace descriptors. |
| `count` | `unsigned int` | in | Number of entries `buf` can hold. |
| `flags` | `unsigned int` | in | Bitmask: `LISTNS_ALL_TYPES` (= 0 default), reserved otherwise. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of `ns_info` entries actually written; never exceeds `count`. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | **Always, in 7.1.0-rc2.** The syscall is reserved but not implemented. |
| `EBADF` | `pidfd` is not a valid pidfd. |
| `EINVAL` | `flags` includes unknown bits, or `count == 0` with non-NULL `buf`. |
| `EFAULT` | `buf` points outside the caller's address space. |
| `ESRCH` | The target task has exited and its namespaces are dropped. |
| `EPERM` | Caller lacks `CAP_SYS_PTRACE` over the target's userns AND target is not the calling task. |

### abi surface

```text
__NR_listns (x86_64)   = 470   /* reserved; ENOSYS in 7.1.0-rc2 */
__NR_listns (arm64)    = 470   /* reserved; ENOSYS in 7.1.0-rc2 */
__NR_listns (generic)  = 470

#define LISTNS_ALL_TYPES   0x00000000  /* (default) emit all 8 ns types */
#define LISTNS_F_RESERVED  0xFFFFFFFE  /* any bit ⟹ EINVAL */

/* ns_type values match include/linux/nsproxy.h: */
#define NS_TYPE_MNT     CLONE_NEWNS       /* 0x00020000 */
#define NS_TYPE_CGROUP  CLONE_NEWCGROUP   /* 0x02000000 */
#define NS_TYPE_UTS     CLONE_NEWUTS      /* 0x04000000 */
#define NS_TYPE_IPC     CLONE_NEWIPC      /* 0x08000000 */
#define NS_TYPE_USER    CLONE_NEWUSER     /* 0x10000000 */
#define NS_TYPE_PID     CLONE_NEWPID      /* 0x20000000 */
#define NS_TYPE_NET     CLONE_NEWNET      /* 0x40000000 */
#define NS_TYPE_TIME    CLONE_NEWTIME     /* 0x00000080 */
```

### compatibility contract

REQ-1: In 7.1.0-rc2, `listns` is wired into the syscall table at slot 470 but the implementation body returns `-ENOSYS`. Userspace probing the syscall MUST see `ENOSYS`, NOT `EINVAL` or `EFAULT`.

REQ-2: `__NR_listns = 470` slot is reserved cross-arch (x86_64, arm64, riscv, generic) to avoid divergent numbering when the implementation lands. Reservation is committed to the ABI registry.

REQ-3: When implemented (target: phase-E or later), the syscall MUST:
  - Validate `flags & LISTNS_F_RESERVED == 0`, else `EINVAL`.
  - Validate `count > 0` if `buf != NULL`, else `EINVAL`.
  - Resolve `pidfd` to a task via `pidfd_get_task`; failure: `ESRCH`.
  - Cross-userns access requires `CAP_SYS_PTRACE` in the target's user namespace if the caller is not the target itself.

REQ-4: Enumeration order is **canonical**: MNT, CGROUP, UTS, IPC, USER, PID, NET, TIME. Implementations MUST emit in this order so userspace can binary-compare.

REQ-5: `ns_id` MUST equal `stat(/proc/<tgid>/ns/<type>).st_ino` for the corresponding namespace — i.e. it is the nsfs inode number that the kernel already exposes. This guarantees interop with `setns(2)` and `ioctl(nsfd, NS_GET_*)`.

REQ-6: `user_ns_id` is the owning user namespace's id; for `NS_TYPE_USER` entries it equals `ns_id`. `parent_id` is the parent for `PID` and `USER` namespaces; 0 for other types (which are not hierarchical).

REQ-7: Truncation: if the target has more namespaces than `count` permits, the call writes `count` entries and returns `count`. Userspace re-invokes with a larger buffer to detect saturation. (Future flag `LISTNS_F_REPORT_TOTAL` may instead write `count` entries but return the total count.)

REQ-8: Atomicity: the snapshot is consistent across a single `task_lock(target)` window; concurrent `setns` / `unshare` does not split the result.

REQ-9: No LSM hook is added beyond the existing `security_capable` for `CAP_SYS_PTRACE`. Future may add `security_namespace_enumerate(target)`.

REQ-10: Audit emits `AUDIT_NAMESPACE_LIST` with `target_tgid`, `caller_tgid`, `caller_caps`, `count_returned`.

REQ-11: `pidfd == -1` is reserved; future versions MAY accept it as "enumerate caller's namespaces" without ptrace gate.

REQ-12: The syscall MUST NOT widen any existing capability surface; it only consolidates an enumeration that `/proc` already permits.

REQ-13: Backward compat: kernels without `listns` MUST return `-ENOSYS` from slot 470 (do not return `-EINVAL`); libc probes rely on this.

REQ-14: Forward compat: `struct ns_info` is sized 32 bytes; future versions may extend via a separate syscall or a `flags` opt-in for a larger struct. No silent struct-size widening.

REQ-15: When wired, this syscall replaces neither `setns(2)` nor `pidfd_open(2)` — it is purely an enumeration helper.

