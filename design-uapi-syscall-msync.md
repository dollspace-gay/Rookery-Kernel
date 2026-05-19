---
title: "Tier-5 syscall: msync(2) — syscall 26"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`msync(2)` requests that modifications made to a memory-mapped region backed by a file be flushed to the underlying storage so that they are visible after a crash or to other processes mapping the same file. The caller designates a byte range `[addr, addr + length)` (page-aligned `addr`) and one of three semantics: `MS_ASYNC` schedules dirty-page writeback without waiting, `MS_SYNC` performs writeback and blocks until the storage stack returns, and `MS_INVALIDATE` (optionally combined with the first two) drops cached pages that are inconsistent with concurrent writes by another `MAP_SHARED` mapper, forcing subsequent accesses to refault from the backing file.

`msync(2)` is the user-space counterpart of `fsync(2)` for shared file mappings. Without it, a writer mapping a file with `MAP_SHARED` has no guarantee about when dirty PTE bits propagate to the page cache and from the page cache to disk; `MS_SYNC` closes both of those gaps for the indicated range. Critical for: durability of mmap-based persistence layers (LMDB, BoltDB, SQLite mmap mode), DAX-aware databases, log-structured stores, anything that uses `mmap` + `MAP_SHARED` for write-through I/O.

### Acceptance Criteria

- [ ] AC-1: `msync(addr, len, MS_SYNC)` on a shared file mapping flushes dirty pages and waits; `pread` from the file shows the data on return.
- [ ] AC-2: `msync(addr, len, MS_ASYNC)` returns before writeback completes; observable via slow block device.
- [ ] AC-3: `msync(addr, len, MS_SYNC | MS_ASYNC)` returns `-EINVAL`.
- [ ] AC-4: `msync(addr+1, len, MS_SYNC)` returns `-EINVAL` (unaligned).
- [ ] AC-5: `msync(addr, len, 0x10)` returns `-EINVAL` (unknown bit).
- [ ] AC-6: `msync` on a hole-punched region of a VMA returns `-ENOMEM`.
- [ ] AC-7: `msync(MS_INVALIDATE)` on an `mlock`ed range returns `-EBUSY`.
- [ ] AC-8: `msync(MS_SYNC)` on a private mapping is a no-op (returns 0, no I/O).
- [ ] AC-9: Background writeback error: subsequent `msync(MS_SYNC)` returns `-EIO` once.
- [ ] AC-10: DAX mapping `msync(MS_SYNC)`: CPU cache-line flushes issued; no block-layer I/O.

### Architecture

```rust
#[syscall(nr = 26, abi = "sysv")]
pub fn sys_msync(addr: UserAddr, length: usize, flags: i32) -> isize {
    Msync::do_msync(addr, length, flags)
}
```

`Msync::do_msync(addr, length, flags) -> isize`:
1. const VALID: i32 = MS_ASYNC | MS_SYNC | MS_INVALIDATE;
2. if (flags & !VALID) != 0 { return -EINVAL; }
3. if (flags & MS_ASYNC) != 0 && (flags & MS_SYNC) != 0 { return -EINVAL; }
4. if addr & (PAGE_SIZE - 1) != 0 { return -EINVAL; }
5. let end = addr.checked_add(page_align(length)).ok_or(EINVAL)?;
6. let mm = current().mm();
7. mm.mmap_read_lock();
8. let mut cursor = addr;
9. let mut last_err: isize = 0;
10. while cursor < end {
11.    let Some(vma) = mm.find_vma(cursor) else { last_err = -ENOMEM; break; };
12.    if vma.start() > cursor { last_err = -ENOMEM; break; }
13.    let lo = max(vma.start(), cursor);
14.    let hi = min(vma.end(), end);
15.    if (flags & MS_INVALIDATE) != 0 && vma.flags().contains(VM_LOCKED) {
16.        last_err = -EBUSY; break;
17.    }
18.    if vma.flags().contains(VM_SHARED) {
19.        if let Some(file) = vma.file() {
20.            let off = vma.file_offset() + (lo - vma.start());
21.            let len = hi - lo;
22.            let wait = (flags & MS_SYNC) != 0;
23.            let rc = filemap_fdatawrite_range(file.inode(), off, off + len - 1);
24.            if rc != 0 && last_err == 0 { last_err = rc; }
25.            if wait {
26.                let rc = filemap_fdatawait_range(file.inode(), off, off + len - 1);
27.                if rc != 0 && last_err == 0 { last_err = rc; }
28.            }
29.            let rc = file_check_and_advance_wb_err(file);
30.            if rc != 0 && last_err == 0 { last_err = rc; }
31.        }
32.    }
33.    cursor = hi;
34. }
35. mm.mmap_read_unlock();
36. last_err

### Out of Scope

- mm/filemap.c writeback engine (covered in Tier-3 `mm/filemap.md`).
- DAX-specific flush paths (covered in Tier-3 `fs/dax.md`).
- Per-filesystem `->fsync` implementation (covered in fs Tier-3 docs).
- Implementation code.

### signature

```c
int msync(void *addr, size_t length, int flags);
```

```c
#define MS_ASYNC      1   /* Sync memory asynchronously. */
#define MS_INVALIDATE 2   /* Invalidate the caches. */
#define MS_SYNC       4   /* Synchronous memory sync. */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `addr` | `void *` | in | Page-aligned start of the range; must point inside an existing VMA. |
| `length` | `size_t` | in | Byte length of the range; rounded up to whole pages. |
| `flags` | `int` | in | Bitmask of `MS_ASYNC`, `MS_SYNC`, `MS_INVALIDATE`. `MS_ASYNC` and `MS_SYNC` are mutually exclusive. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBUSY` | `MS_INVALIDATE` and some page in the range is locked via `mlock(2)`. |
| `EINVAL` | `addr` not page-aligned; `length` overflow; `flags` has bits other than the three defined; both `MS_ASYNC` and `MS_SYNC` set. |
| `ENOMEM` | Range partially or wholly outside any VMA. |
| `EIO` | Underlying storage I/O error during synchronous writeback. |

### abi surface

```text
__NR_msync  (x86_64)  = 26
__NR_msync  (arm64)   = 227
__NR_msync  (riscv)   = 227
__NR_msync  (i386)    = 144

/* All three flags are bit-positions in a single int; MS_ASYNC=1, MS_INVALIDATE=2, MS_SYNC=4. */
/* Page size is the architecture page size (`getpagesize(3)`). */
```

### compatibility contract

REQ-1: Syscall number is **26** on x86_64. ABI-stable.

REQ-2: `flags` validation:
- `(flags & ~(MS_ASYNC | MS_INVALIDATE | MS_SYNC)) != 0` returns `-EINVAL`.
- `(flags & MS_ASYNC) && (flags & MS_SYNC)` returns `-EINVAL`.

REQ-3: `addr` must be page-aligned. `addr & (PAGE_SIZE - 1) != 0` returns `-EINVAL`.

REQ-4: `length` is rounded up to a whole-page boundary using `PAGE_ALIGN(length)`. Arithmetic overflow during alignment returns `-EINVAL`.

REQ-5: The kernel walks the VMAs intersecting `[addr, addr + length)` under `mmap_read_lock`. Any sub-range not covered by a VMA terminates the walk and the syscall returns `-ENOMEM`, but writeback already initiated on earlier VMAs is not undone.

REQ-6: Per-VMA semantics:
- Anonymous VMA: writeback is a no-op; only invalidate semantics apply (none — anonymous pages have no backing inode).
- File-backed VMA with `VM_SHARED`: `filemap_fdatawrite_range(inode, start, end - 1)` is issued; with `MS_SYNC` the call additionally waits via `filemap_fdatawait_range`.
- Private VMA (`!VM_SHARED`): writeback is skipped; dirty pages stay private to the process.

REQ-7: `MS_INVALIDATE` semantics: rejects the call with `-EBUSY` if any page in the range is `VM_LOCKED`. Otherwise it asks the file system to drop clean cached pages so that subsequent reads see the most recent on-disk content; this is the path used by NFS clients to discover writes from other clients.

REQ-8: Writeback error reporting: after issuing writeback, `msync` consults the per-file `errseq_t` via `file_check_and_advance_wb_err()`, which returns `-EIO` if a writeback error occurred since the last consultation by this file descriptor.

REQ-9: Forward compatibility: no extension bits are reserved in `flags`; the kernel refuses unknown bits today. New semantics would need a new syscall (e.g. `msync_range2`) rather than overloading the existing flag word.

REQ-10: `msync` is asynchronous-signal-safe on best-effort basis only; it is not in the POSIX list of async-signal-safe functions. Implementations may take page-table locks.

REQ-11: HugeTLB mappings are flushed at the hugepage granularity; the range is rounded out to the surrounding huge pages.

REQ-12: DAX (Direct Access) mappings short-circuit: writeback is replaced by a cache-line flush of the persistent-memory range, since the mapping bypasses the page cache.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_validation_total` | INVARIANT | every reserved bit in `flags` rejected. |
| `mutually_exclusive` | INVARIANT | `MS_ASYNC | MS_SYNC` always rejected. |
| `align_check_pre_walk` | INVARIANT | unaligned addr returns before VMA walk. |
| `mmap_read_lock_held_during_walk` | INVARIANT | mm.lock balanced. |
| `enomem_on_hole` | INVARIANT | VMA gap inside range ⟹ ENOMEM. |
| `wb_err_once_per_fd` | INVARIANT | errseq_t advanced exactly once per file. |

### Layer 2: TLA+

`mm/msync.tla`:
- States: per-flag-validate, per-vma-walk, per-write, per-wait, per-errseq-check.
- Properties:
  - `safety_no_writeback_on_private` — private VMA never triggers fdatawrite.
  - `safety_einval_unknown_flag` — unknown flag bit always rejected.
  - `safety_ebusy_invalidate_locked` — MS_INVALIDATE + VM_LOCKED ⟹ EBUSY.
  - `liveness_msync_terminates` — every msync call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_msync` post: returns `-EINVAL` for invalid flags | `Msync::do_msync` |
| `do_msync` post: returns `-ENOMEM` if range gap | `Msync::do_msync` |
| `do_msync` post: MS_SYNC implies fdatawait called | `Msync::do_msync` |
| `do_msync` post: errseq_t observed at most once per fd | `Msync::do_msync` |

### Layer 4: Verus / Creusot functional

Per-`msync(2)` man-page and POSIX `msync` semantics; LTP `lib/testcases/kernel/syscalls/msync/` selftests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`msync(2)` reinforcement:

- **Per-page-alignment EINVAL** — defense against per-range-confusion bugs.
- **Per-VMA shared-only writeback** — defense against per-private-leak.
- **Per-fd errseq_t single-report** — defense against per-EIO-silent-loss.
- **Per-MS_INVALIDATE mlock check** — defense against per-pin-bypass.
- **Per-arithmetic overflow on PAGE_ALIGN** — defense against per-integer-overflow into kernel.
- **Per-MS_SYNC bounded wait** — defense against per-DoS via stalled storage.

### grsecurity / pax surface

- **PaX UDEREF on `addr` validation** — defense against per-user-addr kernel-deref bug; SMAP forced when the syscall reads VMA metadata using `addr`.
- **GRKERNSEC_HARDEN_MMAP_SEMANTICS** — strict flag mask, no future-extension flag silently accepted; non-zero reserved bits always EINVAL.
- **PAX_USERCOPY_HARDEN on errseq propagation** — `file_check_and_advance_wb_err` return passes through whitelisted slab path; no inline-stack copy_to_user.
- **GRKERNSEC_NO_SIMULT_CONNECT-style writeback bound** — MS_SYNC waits are capped against per-process-blocked-on-storage DoS budget.
- **PaX KERNEXEC on writeback callbacks** — `address_space_operations.writepages` indirect call goes through the function-pointer trampoline.
- **GRKERNSEC_TRUSTED_PATH on backing inodes** — msync on a backing file whose inode is on a non-trusted mount may be denied when the policy forbids cross-mount writeback.
- **PAX_REFCOUNT on file refcount during fdatawait** — defense against per-refcount-underflow UAF if VMA torn down mid-call.
- **Per-MS_INVALIDATE audit-trail** — invalidations on shared mappings emit `audit_log_syscall` records when GRKERNSEC_AUDIT_MOUNT is on, since invalidate can mask remote writer activity.
- **GRKERNSEC_HIDESYM on writeback error stacks** — kernel-mode EIO panics from msync paths sanitize kernel addresses in the panic message.
- **Per-DAX direct-flush gated by CAP_SYS_RAWIO** — msync on a DAX mapping verifies caller still holds the cap; defense against per-priv-drop bypass.

