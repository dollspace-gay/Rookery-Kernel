---
title: "Tier-5 syscall: statmount(2) — syscall 457"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`statmount(2)` returns extensible, typed information about a single mount object, identified by `mnt_id` (the same id surfaced by `/proc/self/mountinfo`). It is to mounts what `statx(2)` is to files: a forward-compatible, version-bearing struct plus a `mask` of which fields to fill, replacing brittle `mountinfo` text parsing. Companion to `listmount(2)`.

Critical for: container runtimes, observability, mountns enumeration without `/proc` access (rootless / seccomp-restricted callers), and tools that need filesystem subtype / propagation / peer-group data programmatically.

### Acceptance Criteria

- [ ] AC-1: `statmount(req={mnt_id=root_id,param=STATMOUNT_FS_TYPE|STATMOUNT_MNT_POINT}, buf, sz, 0)` returns >0 and buf contains "/" + "ext4" (or matching).
- [ ] AC-2: Unknown bit in `req->param` returns `-EINVAL`.
- [ ] AC-3: `flags = 1` returns `-EINVAL`.
- [ ] AC-4: Non-existent `mnt_id` returns `-ENOENT`.
- [ ] AC-5: `bufsize` too small returns `-EOVERFLOW`.
- [ ] AC-6: Cross-ns request without caps returns `-EPERM`.
- [ ] AC-7: Output `statmount.mask` indicates only filled bits (subset of `req->param`).

### Architecture

```rust
#[syscall(nr = 457, abi = "sysv")]
pub fn sys_statmount(
    req: UserPtr<MntIdReq>, buf: UserPtr<u8>, bufsize: usize, flags: u32,
) -> isize {
    Statmount::do_statmount(req, buf, bufsize, flags)
}
```

`Statmount::do_statmount(req_ptr, buf, bufsize, flags) -> isize`:
1. if flags != 0 { return -EINVAL; }
2. let req = Statmount::copy_req_from_user(req_ptr)?;             // EFAULT, EINVAL, E2BIG
3. if req.param & !STATMOUNT__KNOWN != 0 { return -EINVAL; }
4. let ns = Mount::resolve_ns(req.mnt_ns_id, current_creds())?;   // EPERM
5. let mnt = ns.lookup_mnt_by_id(req.mnt_id).ok_or(ENOENT)?;
6. if bufsize < size_of::<Statmount>() { return -EOVERFLOW; }
7. let mut out = StatmountBuilder::new(buf, bufsize);
8. namespace_sem.read_lock();
9. Statmount::fill_mnt_basic(&mut out, &mnt, req.param);
10. Statmount::fill_sb_basic(&mut out, &mnt.sb, req.param);
11. Statmount::fill_strings(&mut out, &mnt, req.param)?;          // ENOSPC
12. namespace_sem.read_unlock();
13. out.finalize() as isize

`Statmount::copy_req_from_user(uptr) -> Result<MntIdReq>`:
1. const K: usize = size_of::<MntIdReq>();
2. let mut probe_size = 0u32;
3. uptr.copy_in_partial(&mut probe_size, 4)?;
4. let n = min(probe_size as usize, K);
5. let mut r = MntIdReq::zeroed();
6. uptr.copy_in_partial(&mut r, n)?;
7. if (probe_size as usize) > K {
8.   let tail = (probe_size as usize) - K;
9.   if !Statmount::tail_is_zero(uptr.add(K), tail)? { return Err(E2BIG); }
10. }
11. if r.spare != 0 { return Err(EINVAL); }
12. Ok(r)

### Out of Scope

- `listmount(2)` enumeration (own Tier-5 doc).
- Per-fs subtype semantics (Tier-3 per-fs docs).
- Implementation code.

### signature

```c
int statmount(const struct mnt_id_req *req,
              struct statmount *buf, size_t bufsize,
              unsigned int flags);
```

```c
struct mnt_id_req {
    __u32 size;          /* sizeof(struct mnt_id_req) */
    __u32 spare;
    __u64 mnt_id;        /* target mount id (uniq) */
    __u64 param;         /* requested STATMOUNT_* mask */
    __u64 mnt_ns_id;     /* 0 = current mntns, else namespace id */
};

#define STATMOUNT_SB_BASIC       0x00000001  /* sb_dev_major/minor, sb_magic, sb_flags */
#define STATMOUNT_MNT_BASIC      0x00000002  /* mnt_id, parent, attr, propagation, peer_group, master */
#define STATMOUNT_PROPAGATE_FROM 0x00000004
#define STATMOUNT_MNT_ROOT       0x00000008  /* string: mount root path within sb */
#define STATMOUNT_MNT_POINT      0x00000010  /* string: mountpoint within mountns */
#define STATMOUNT_FS_TYPE        0x00000020  /* string: e.g. "ext4" */
#define STATMOUNT_MNT_NS_ID      0x00000040
#define STATMOUNT_MNT_OPTS       0x00000080  /* string: comma-separated options */
#define STATMOUNT_FS_SUBTYPE     0x00000100
#define STATMOUNT_SB_SOURCE      0x00000200
#define STATMOUNT_OPT_ARRAY      0x00000400  /* nul-separated array */
#define STATMOUNT_OPT_SEC_ARRAY  0x00000800
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `req` | `const struct mnt_id_req *` | in | Caller-owned request: target mnt_id, requested mask, optional ns. |
| `buf` | `struct statmount *` | out | Caller-owned output buffer, at least `sizeof(struct statmount)`. Variable-length strings appended after fixed area. |
| `bufsize` | `size_t` | in | Size of `buf`. |
| `flags` | `unsigned int` | in | Reserved; must be 0. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes written to `buf` (fixed header + variable-string tail). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `flags != 0`; `req->size` invalid; `req->param` requests unknown bits; reserved fields non-zero. |
| `EFAULT` | `req` or `buf` pointer faults. |
| `EPERM` | Caller lacks access to the target mount namespace (`mnt_ns_id` not visible). |
| `ENOENT` | `mnt_id` does not exist in the resolved namespace. |
| `E2BIG` | `req->size` larger than kernel's known struct AND trailing bytes non-zero. |
| `ENOMEM` | Allocator failed gathering strings. |
| `EOVERFLOW` | `bufsize` too small to even fit the fixed header. |
| `ENOSPC` | `bufsize` insufficient for requested string set. |

### abi surface

```text
__NR_statmount (x86_64) = 457
__NR_statmount (arm64)  = 457
__NR_statmount (riscv)  = 457
__NR_statmount (i386)   = 457

/* Forward-compat: req->size > sizeof(req) requires zero tail (E2BIG else).
   Caller passes statmount.size = sizeof(statmount); kernel fills only fields
   selected by req->param and indicated by output statmount.mask. */
```

### compatibility contract

REQ-1: Syscall number is **457** on x86_64. ABI-stable since Linux 6.8.

REQ-2: `flags == 0`. Any other value returns `-EINVAL`.

REQ-3: `req->size` validated identically to `statx`: known size accepted as-is; larger sizes accepted iff tail is zero; smaller sizes treated as if tail were zero.

REQ-4: `req->param`: any bit outside the kernel's known `STATMOUNT_*` set returns `-EINVAL`. Only requested bits populated.

REQ-5: Output `statmount.mask` indicates which requested bits the kernel actually filled (some may be unsupported on a given fs or absent for the mount).

REQ-6: `mnt_ns_id == 0` resolves in the caller's mount namespace; non-zero must be a namespace id the caller can access (`CAP_SYS_ADMIN` in that ns's owning user_ns, or it's the caller's own).

REQ-7: Variable-length strings (`STATMOUNT_MNT_ROOT`, `STATMOUNT_MNT_POINT`, `STATMOUNT_FS_TYPE`, `STATMOUNT_MNT_OPTS`, ...) are appended after the fixed header and pointed to by `__u32` offsets in the header relative to the start of `buf`. All strings are NUL-terminated.

REQ-8: Reserved fields in `mnt_id_req` (`spare`) must be zero; non-zero ⟹ `-EINVAL`.

REQ-9: No global state mutated. Pure read.

REQ-10: Concurrency: under `namespace_sem` read-lock; consistent snapshot of the target mount.

REQ-11: Strings copied to user via `copy_to_user` with PAX_USERCOPY-style bounds; truncated strings return `-ENOSPC` (no partial fill).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_zero` | INVARIANT | flags != 0 ⟹ EINVAL. |
| `req_size_validated` | INVARIANT | req.size > sizeof(req) ∧ tail non-zero ⟹ E2BIG. |
| `param_known_bits` | INVARIANT | unknown bit in req.param ⟹ EINVAL. |
| `bufsize_min` | INVARIANT | bufsize < sizeof(Statmount) ⟹ EOVERFLOW. |
| `cross_ns_capability` | INVARIANT | mnt_ns_id != caller's ⟹ cap check. |

### Layer 2: TLA+

`fs/statmount.tla`:
- States: validate-flags, copy-req, resolve-ns, lookup-mnt, fill-fixed, fill-strings, copy-to-user, return.
- Properties:
  - `safety_no_kernel_leak` — never copy uninitialised kernel memory.
  - `safety_mask_subset` — output.mask ⊆ req.param.
  - `liveness_returns` — every statmount returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_statmount` post: ret > 0 ⟹ out.mask ⊆ req.param | `Statmount::do_statmount` |
| `copy_req_from_user` post: r.spare == 0 | `Statmount::copy_req_from_user` |
| `fill_strings` post: each string NUL-terminated | `Statmount::fill_strings` |

### Layer 4: Verus / Creusot functional

Per-`statmount(2)` man-page semantic equivalence vs `/proc/self/mountinfo` parser regression tests.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`statmount(2)` reinforcement:

- **Per-info-leak struct zero-init** — every statmount output zeroed before fill; reserved bytes always zero.
- **Per-mask subset invariant** — defense against per-uninitialised-field leak.
- **Per-cross-ns capability** — defense against per-mountns enumeration probe.
- **Per-bufsize EOVERFLOW** — defense against per-short-buffer partial fill.
- **Per-string NUL termination guaranteed** — defense against per-string-leak past terminator.

### grsecurity / pax surface

- **PaX UDEREF on `req` and `buf` copy_from_user / copy_to_user** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **GRKERNSEC_HIDESYM** — kernel addresses never embedded in statmount output (peer-group / master id are namespace-local indices, not pointers).
- **GRKERNSEC_PROC_RESTRICT** — statmount on mounts outside caller's view returns ENOENT (matches /proc/self/mountinfo redaction policy).
- **PAX_USERCOPY_HARDEN on string copy_to_user** — bounded copy uses whitelisted slab; truncation = ENOSPC.
- **GRKERNSEC_CHROOT_FINDTASK / GRKERNSEC_CHROOT_NICE** — chrooted process cannot statmount mounts outside its chroot subtree.
- **Per-mnt_ns_id strict capability** — non-init userns must own the target mountns; grsec rejects any cross-userns query without init_user_ns CAP_SYS_ADMIN.
- **Per-reserved fields strict zero** — defense against per-extension-field smuggling (req.spare must be 0).
- **PAX_REFCOUNT on mnt refcount during snapshot** — defense against per-mount-vanish race.
- **GRKERNSEC_DMESG-style rate limit on EINVAL spam** — defense against per-probe denial via repeated invalid requests.
- **Audit log on cross-userns statmount** — defense against silent mountns reconnaissance.

