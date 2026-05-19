---
title: "Tier-5 syscall: listmount(2) — syscall 458"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`listmount(2)` enumerates child mount ids underneath a parent mount, returning a vector of `u64` mount ids in `mnt_ids`. It is the iterator companion to `statmount(2)`: a caller does `listmount` to discover ids, then `statmount` on each. Together they replace `/proc/self/mountinfo` text parsing with a typed, paginated, namespace-scoped enumeration.

Critical for: container runtimes that want a complete mountns walk without touching `/proc`, observability of large mount trees, and rootless mount-table snapshotting.

### Acceptance Criteria

- [ ] AC-1: `listmount(req={mnt_id=root,param=0}, ids, 16, 0)` returns count of immediate root children.
- [ ] AC-2: Calling again with `param = last_id` returns next page; eventually returns < nr_mnt_ids signalling end.
- [ ] AC-3: `LISTMOUNT_REVERSE` returns ids in descending order.
- [ ] AC-4: Unknown flag bit returns `-EINVAL`.
- [ ] AC-5: Non-existent parent `mnt_id` returns `-ENOENT`.
- [ ] AC-6: Cross-ns request without caps returns `-EPERM`.
- [ ] AC-7: `nr_mnt_ids = 0` returns 0 (no fault).

### Architecture

```rust
#[syscall(nr = 458, abi = "sysv")]
pub fn sys_listmount(
    req: UserPtr<MntIdReq>, mnt_ids: UserPtr<u64>, nr: usize, flags: u32,
) -> isize {
    Listmount::do_listmount(req, mnt_ids, nr, flags)
}
```

`Listmount::do_listmount(req_ptr, ids_ptr, nr, flags) -> isize`:
1. if flags & !LISTMOUNT_REVERSE != 0 { return -EINVAL; }
2. let req = Listmount::copy_req_from_user(req_ptr)?;             // EFAULT, EINVAL, E2BIG
3. let ns = Mount::resolve_ns(req.mnt_ns_id, current_creds())?;   // EPERM
4. let parent = ns.lookup_mnt_by_id(req.mnt_id).ok_or(ENOENT)?;
5. if nr == 0 { return 0; }
6. let cursor = req.param;
7. let reverse = flags & LISTMOUNT_REVERSE != 0;
8. let mut out = KernelBuf::<u64>::with_capacity(nr);
9. namespace_sem.read_lock();
10. Listmount::collect_children(&parent, cursor, reverse, &mut out, nr);
11. namespace_sem.read_unlock();
12. unsafe { ids_ptr.copy_out(&out)?; }                            // EFAULT
13. out.len() as isize

`Listmount::collect_children(parent, cursor, reverse, out, cap)`:
1. let mut iter = parent.children.iter_sorted_by_id(reverse);
2. for child in iter.skip_while(|c| if reverse { c.id >= cursor } else { c.id <= cursor }) {
3.   if out.len() == cap { break; }
4.   out.push(child.id);
5. }

### Out of Scope

- `statmount(2)` per-mount detail (own Tier-5 doc).
- Recursive enumeration policy (caller's responsibility).
- Implementation code.

### signature

```c
ssize_t listmount(const struct mnt_id_req *req,
                  u64 *mnt_ids, size_t nr_mnt_ids,
                  unsigned int flags);
```

```c
/* req fields: mnt_id selects parent; param is the "last mnt_id seen"
   cursor for pagination (0 = start). mnt_ns_id selects namespace. */

#define LISTMOUNT_REVERSE  0x00000001  /* return ids in reverse order */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `req` | `const struct mnt_id_req *` | in | Caller-owned: `mnt_id` = parent mount; `param` = cursor (last id from previous call, 0 to start); `mnt_ns_id` = namespace. |
| `mnt_ids` | `u64 *` | out | Output array of child mount ids. |
| `nr_mnt_ids` | `size_t` | in | Capacity of `mnt_ids` in elements. |
| `flags` | `unsigned int` | in | `LISTMOUNT_REVERSE` or 0. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of mount ids written (may be 0 = no more children, or up to `nr_mnt_ids`). To detect end-of-iteration, call again with `req->param` set to the last returned id. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | Unknown bit in `flags`; reserved fields non-zero; `req->size` invalid. |
| `EFAULT` | `req` or `mnt_ids` pointer faults. |
| `EPERM` | Caller cannot access the target mount namespace. |
| `ENOENT` | Parent `mnt_id` does not exist. |
| `E2BIG` | `req->size` larger than kernel-known with non-zero tail. |
| `ENOMEM` | Allocator failed during enumeration. |

### abi surface

```text
__NR_listmount (x86_64) = 458
__NR_listmount (arm64)  = 458
__NR_listmount (riscv)  = 458
__NR_listmount (i386)   = 458

/* Pagination contract: caller treats return == nr_mnt_ids as "more
   may exist" (re-call with param = last id); return < nr_mnt_ids
   ⟹ enumeration complete. */
```

### compatibility contract

REQ-1: Syscall number is **458** on x86_64. ABI-stable since Linux 6.8.

REQ-2: `flags` validation: only `LISTMOUNT_REVERSE` accepted; any other bit returns `-EINVAL`.

REQ-3: `req->size` validated as in `statmount`: known size accepted; larger sizes with zero tail accepted; larger sizes with non-zero tail returns `-E2BIG`.

REQ-4: `req->mnt_id` selects the parent mount whose children are enumerated. The parent itself is **not** included in output.

REQ-5: `req->param` is the cursor: starting from 0, the kernel returns up to `nr_mnt_ids` children of mount id strictly greater than `param` (or strictly less, with `LISTMOUNT_REVERSE`). Caller passes back the last returned id to continue.

REQ-6: `req->mnt_ns_id == 0` → caller's mountns. Non-zero requires the namespace to be accessible (`CAP_SYS_ADMIN` in that ns's owning user_ns, or owned by caller).

REQ-7: Only direct children are returned (not grand-children); recursion is the caller's job.

REQ-8: Order: ascending mnt_id (kernel-internal monotonic id) by default; descending with `LISTMOUNT_REVERSE`.

REQ-9: Consistency: enumeration takes `namespace_sem` read-lock per page; mounts created/destroyed between pages may be skipped or repeated. Callers tolerant of mount churn.

REQ-10: `nr_mnt_ids == 0` returns 0 (caller can probe existence of parent only by checking ENOENT vs 0; passing 0 capacity is legal).

REQ-11: Reserved `mnt_id_req.spare` must be zero.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_mask_validated` | INVARIANT | flags & ~LISTMOUNT_REVERSE ⟹ EINVAL. |
| `req_size_validated` | INVARIANT | tail non-zero ⟹ E2BIG. |
| `cap_for_cross_ns` | INVARIANT | mnt_ns_id != caller's ⟹ cap check. |
| `output_bounded` | INVARIANT | written ids ≤ nr_mnt_ids. |
| `cursor_progress` | INVARIANT | each returned id > cursor (or < with REVERSE). |

### Layer 2: TLA+

`fs/listmount.tla`:
- States: flags-validate, req-copy, resolve-ns, lookup-parent, enumerate, copy-out.
- Properties:
  - `safety_no_parent_in_output` — parent's own id never appears.
  - `safety_strict_cursor_progress` — guarantees forward progress.
  - `liveness_returns` — every listmount returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_listmount` post: ret ≤ nr_mnt_ids | `Listmount::do_listmount` |
| `collect_children` post: all written ids strictly past cursor | `Listmount::collect_children` |
| `copy_req_from_user` post: spare == 0 | `Listmount::copy_req_from_user` |

### Layer 4: Verus / Creusot functional

Per-`listmount(2)` man-page semantic equivalence with `/proc/self/mountinfo` enumeration: union of paged calls reproduces the mountinfo set (modulo concurrent churn).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`listmount(2)` reinforcement:

- **Per-info-leak buffer zero-init** — never copy uninitialised kernel memory.
- **Per-flag whitelist** — defense against per-extension-flag smuggling.
- **Per-cross-ns capability** — defense against per-mountns enumeration probe.
- **Per-output cap clamp** — defense against per-array-overflow.
- **Per-cursor strict progress** — defense against per-iterator infinite-loop.

### grsecurity / pax surface

- **PaX UDEREF on `req` copy_from_user and `mnt_ids` copy_to_user** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **GRKERNSEC_PROC_RESTRICT** — listmount on mounts outside caller's view returns ENOENT (matches /proc/self/mountinfo redaction).
- **GRKERNSEC_CHROOT_FINDTASK / GRKERNSEC_CHROOT_NICE** — chrooted process cannot listmount above its chroot subtree.
- **PAX_USERCOPY_HARDEN on `mnt_ids` copy_to_user** — bounded copy uses whitelisted slab.
- **Per-init_user_ns required for cross-userns enumeration** — non-init userns cannot list host-mountns mounts even with CAP_SYS_ADMIN.
- **GRKERNSEC_HIDESYM** — mount ids are namespace-local counters, never kernel pointers.
- **PAX_REFCOUNT on mnt refcount during enumeration** — defense against per-mount-vanish race.
- **GRKERNSEC_DMESG-style rate limit on EPERM/ENOENT** — defense against per-probe denial via id sweeps.
- **Audit log on cross-userns listmount** — defense against silent mountns reconnaissance.
- **Per-reserved field strict zero** — defense against per-extension-field smuggling.

