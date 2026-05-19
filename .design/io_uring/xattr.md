# Tier-3: io_uring/xattr.c — IORING_OP_GETXATTR / SETXATTR / FGETXATTR / FSETXATTR

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/xattr.c (~197 lines)
  - io_uring/xattr.h
  - fs/xattr.c (file_getxattr, file_setxattr, filename_getxattr, filename_setxattr, setxattr_copy, import_xattr_name, struct kernel_xattr_ctx)
  - fs/internal.h (delayed_filename, delayed_getname, filename_complete_delayed, dismiss_delayed_filename, INIT_DELAYED_FILENAME)
  - include/uapi/linux/io_uring.h (IORING_OP_GETXATTR / SETXATTR / FGETXATTR / FSETXATTR)
  - include/uapi/linux/xattr.h (XATTR_CREATE, XATTR_REPLACE, XATTR_NAME_MAX, XATTR_SIZE_MAX, XATTR_LIST_MAX)
-->

## Summary

io_uring xattr implements **asynchronous getxattr/setxattr** keyed either by a path (`IORING_OP_GETXATTR` / `IORING_OP_SETXATTR`) or by a file descriptor (`IORING_OP_FGETXATTR` / `IORING_OP_FSETXATTR`). Per-shared `struct io_xattr` holds the `kernel_xattr_ctx` (key, value, size, flags) plus a `delayed_filename` for path-keyed variants. Per-prep splits into `__io_getxattr_prep` (allocates `ctx.kname`, calls `import_xattr_name` to copy the name from userspace) and `__io_setxattr_prep` (allocates `ctx.kname`, calls `setxattr_copy` to copy both name and value buffer). Path-keyed variants additionally call `delayed_getname` on sqe->addr3 (the path lives at addr3 because addr/addr2 are already used for name+value). All four ops are **always force-async** — prep sets `REQ_F_FORCE_ASYNC` and `REQ_F_NEED_CLEANUP` unconditionally; xattr handlers may sleep on filesystem I/O / locks. Cleanup frees ctx.kname (xattr name buffer), kvfree's ctx.kvalue (value buffer; may be vmalloc'd if large), and dismisses the delayed filename. Critical for: container runtimes setting capability/security xattrs in bulk, backup/restore tools, ACL/SELinux label propagation.

This Tier-3 covers `io_uring/xattr.c` (~197 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_xattr` | per-XATTR cmd payload | `IoXattr` |
| `io_xattr_cleanup()` | per-cancel/teardown cleanup | `IoUringXattr::xattr_cleanup` |
| `io_xattr_finish()` (static) | per-issue completion tail | `IoUringXattr::xattr_finish` |
| `__io_getxattr_prep()` (static) | per-shared get prep | `IoUringXattr::getxattr_prep_common` |
| `io_fgetxattr_prep()` | per-FGETXATTR prep | `IoUringXattr::fgetxattr_prep` |
| `io_getxattr_prep()` | per-GETXATTR prep | `IoUringXattr::getxattr_prep` |
| `io_fgetxattr()` | per-FGETXATTR issue | `IoUringXattr::fgetxattr` |
| `io_getxattr()` | per-GETXATTR issue | `IoUringXattr::getxattr` |
| `__io_setxattr_prep()` (static) | per-shared set prep | `IoUringXattr::setxattr_prep_common` |
| `io_setxattr_prep()` | per-SETXATTR prep | `IoUringXattr::setxattr_prep` |
| `io_fsetxattr_prep()` | per-FSETXATTR prep | `IoUringXattr::fsetxattr_prep` |
| `io_fsetxattr()` | per-FSETXATTR issue | `IoUringXattr::fsetxattr` |
| `io_setxattr()` | per-SETXATTR issue | `IoUringXattr::setxattr` |
| `struct kernel_xattr_ctx` | per-VFS xattr context | shared (`fs::xattr::KernelXattrCtx`) |
| `import_xattr_name()` | per-name copy_from_user + validate | shared (`fs::xattr`) |
| `setxattr_copy()` | per-name+value copy_from_user + validate | shared (`fs::xattr`) |
| `file_getxattr()` / `file_setxattr()` | per-fd xattr VFS | shared (`fs::xattr`) |
| `filename_getxattr()` / `filename_setxattr()` | per-path xattr VFS | shared (`fs::xattr`) |
| `delayed_getname()` | per-deferred path resolve | shared (`fs::namei`) |
| `filename_complete_delayed` (CLASS) | per-issue path materialise | shared (`fs::namei`) |
| `dismiss_delayed_filename()` | per-cleanup path drop | shared (`fs::namei`) |
| `INIT_DELAYED_FILENAME` | per-prep filename init | shared (`fs::namei`) |
| `kmalloc_obj()` | per-typed-alloc helper | shared (`mm::slab`) |
| `kvfree()` | per-mixed kmalloc/vmalloc free | shared (`mm::slab`) |
| `LOOKUP_FOLLOW` | per-path-resolve mode | shared (`fs::namei` const) |
| `AT_FDCWD` | per-cwd anchor | shared uapi const |
| `REQ_F_FIXED_FILE` / `REQ_F_NEED_CLEANUP` / `REQ_F_FORCE_ASYNC` | per-req flags | `ReqFlags` bits |
| `IO_URING_F_NONBLOCK` | per-issue flag (must not be set) | `IssueFlags::NONBLOCK` |
| `IOU_COMPLETE` | per-issue completion sentinel | `IouResult::Complete` |

## Compatibility contract

REQ-1: struct io_xattr layout (in io_kiocb cmd cache):
- file: *File (used by F* variants; path variants ignore).
- ctx: kernel_xattr_ctx { kname (*xattr_name kmalloc), value (user ptr), cvalue (user ptr; SET only), kvalue (kernel-side buffer), size, flags }.
- filename: DelayedFilename (path variants only).

REQ-2: io_xattr_cleanup(req):
- dismiss_delayed_filename(&ix.filename).
- kfree(ix.ctx.kname).
- kvfree(ix.ctx.kvalue).

REQ-3: io_xattr_finish(req, ret) [static, called by issue paths]:
- req.flags &= ~REQ_F_NEED_CLEANUP.    /* we've consumed the buffers; cancel no longer needed */
- io_xattr_cleanup(req).               /* frees kname/kvalue/filename now */
- io_req_set_res(req, ret, 0).

REQ-4: __io_getxattr_prep(req, sqe):
- INIT_DELAYED_FILENAME(&ix.filename).
- ix.ctx.kvalue = NULL.
- name = u64_to_user_ptr(READ_ONCE(sqe->addr)).
- ix.ctx.value = u64_to_user_ptr(READ_ONCE(sqe->addr2)).
- ix.ctx.size = READ_ONCE(sqe->len).
- ix.ctx.flags = READ_ONCE(sqe->xattr_flags).
- /* getxattr flags must be zero (none are defined) */
- if ix.ctx.flags ≠ 0: return -EINVAL.
- ix.ctx.kname = kmalloc_obj(*ix.ctx.kname); if !kname: return -ENOMEM.
- ret = import_xattr_name(ix.ctx.kname, name).
- if ret: kfree(ix.ctx.kname); return ret.
- req.flags |= REQ_F_NEED_CLEANUP.
- req.flags |= REQ_F_FORCE_ASYNC.
- return 0.

REQ-5: io_fgetxattr_prep(req, sqe):
- /* No path; fd comes from sqe->fd via IOSQE_FIXED_FILE or normal */
- return __io_getxattr_prep(req, sqe).

REQ-6: io_getxattr_prep(req, sqe):
- /* Path variant cannot use ring-fixed file */
- if req.flags & REQ_F_FIXED_FILE: return -EBADF.
- ret = __io_getxattr_prep(req, sqe); if ret: return ret.
- path = u64_to_user_ptr(READ_ONCE(sqe->addr3)).
- return delayed_getname(&ix.filename, path).

REQ-7: io_fgetxattr(req, issue_flags):
- WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK).
- ret = file_getxattr(req.file, &ix.ctx).
- io_xattr_finish(req, ret).
- return IOU_COMPLETE.

REQ-8: io_getxattr(req, issue_flags):
- WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK).
- name = filename_complete_delayed(&ix.filename).
- ret = filename_getxattr(AT_FDCWD, name, LOOKUP_FOLLOW, &ix.ctx).
- io_xattr_finish(req, ret).
- return IOU_COMPLETE.

REQ-9: __io_setxattr_prep(req, sqe):
- INIT_DELAYED_FILENAME(&ix.filename).
- name = u64_to_user_ptr(READ_ONCE(sqe->addr)).
- ix.ctx.cvalue = u64_to_user_ptr(READ_ONCE(sqe->addr2)).
- ix.ctx.kvalue = NULL.
- ix.ctx.size = READ_ONCE(sqe->len).
- ix.ctx.flags = READ_ONCE(sqe->xattr_flags).
- ix.ctx.kname = kmalloc_obj(*ix.ctx.kname); if !kname: return -ENOMEM.
- ret = setxattr_copy(name, &ix.ctx).   /* copies name + value buffer into kernel */
- if ret: kfree(ix.ctx.kname); return ret.
- req.flags |= REQ_F_NEED_CLEANUP.
- req.flags |= REQ_F_FORCE_ASYNC.
- return 0.

REQ-10: io_setxattr_prep(req, sqe):
- if req.flags & REQ_F_FIXED_FILE: return -EBADF.
- ret = __io_setxattr_prep(req, sqe); if ret: return ret.
- path = u64_to_user_ptr(READ_ONCE(sqe->addr3)).
- return delayed_getname(&ix.filename, path).

REQ-11: io_fsetxattr_prep(req, sqe):
- return __io_setxattr_prep(req, sqe).

REQ-12: io_fsetxattr(req, issue_flags):
- WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK).
- ret = file_setxattr(req.file, &ix.ctx).
- io_xattr_finish(req, ret).
- return IOU_COMPLETE.

REQ-13: io_setxattr(req, issue_flags):
- WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK).
- name = filename_complete_delayed(&ix.filename).
- ret = filename_setxattr(AT_FDCWD, name, LOOKUP_FOLLOW, &ix.ctx).
- io_xattr_finish(req, ret).
- return IOU_COMPLETE.

REQ-14: sqe field map:
- sqe->addr  → xattr name (user ptr to NUL-terminated string).
- sqe->addr2 → value buffer (user ptr; in for GET, out for SET).
- sqe->addr3 → path (user ptr; only for GETXATTR / SETXATTR, not F*).
- sqe->len   → value buffer size.
- sqe->xattr_flags → XATTR_CREATE | XATTR_REPLACE on SET; must be 0 on GET.

REQ-15: Force-async unconditional set at prep (all four ops):
- Implication: io-wq always runs the issue; submitter thread never blocks on xattr handlers.
- Implication: issue bodies WARN_ON_ONCE if NONBLOCK is asserted.

REQ-16: kvfree on ctx.kvalue at cleanup — required because setxattr_copy may have used vmalloc for large values (XATTR_SIZE_MAX = 64K).

REQ-17: After successful issue, REQ_F_NEED_CLEANUP is cleared via io_xattr_finish to prevent double-free; cancellation before issue still runs cleanup.

## Acceptance Criteria

- [ ] AC-1: IORING_OP_GETXATTR with valid path + name + value buffer: CQE result == bytes-read (or full size when size==0 query).
- [ ] AC-2: IORING_OP_FGETXATTR with valid fd: identical semantics to fgetxattr(2).
- [ ] AC-3: IORING_OP_SETXATTR with valid (path, name, value, XATTR_CREATE): creates xattr; -EEXIST if present.
- [ ] AC-4: IORING_OP_FSETXATTR with valid fd: identical semantics to fsetxattr(2).
- [ ] AC-5: GET variants with sqe->xattr_flags != 0: prep returns -EINVAL.
- [ ] AC-6: SET variants with XATTR_CREATE | XATTR_REPLACE (mutually exclusive): rejected at setxattr_copy / VFS — surfaces as -EINVAL in CQE.
- [ ] AC-7: Path variants with REQ_F_FIXED_FILE: prep returns -EBADF.
- [ ] AC-8: kmalloc_obj for ctx.kname OOM: prep returns -ENOMEM.
- [ ] AC-9: import_xattr_name failure (too-long, EINVAL, ENAMETOOLONG): prep frees kname and returns error.
- [ ] AC-10: setxattr_copy failure: prep frees kname and returns error (no kvalue leak — kvalue was NULL pre-call).
- [ ] AC-11: All four ops: prep sets REQ_F_FORCE_ASYNC and REQ_F_NEED_CLEANUP.
- [ ] AC-12: Successful issue: io_xattr_finish clears REQ_F_NEED_CLEANUP and frees kname / kvalue / filename.
- [ ] AC-13: Cancel before issue: io_xattr_cleanup runs once; kname / kvalue / filename freed exactly once.
- [ ] AC-14: Issue path WARN_ON_ONCE if IO_URING_F_NONBLOCK set (force-async invariant violated).
- [ ] AC-15: Large value (size > kmalloc threshold): setxattr_copy uses vmalloc; cleanup uses kvfree (no UB).

## Architecture

```
struct IoXattr {
  file: *File,
  ctx: KernelXattrCtx {
    kname: *XattrName,     // kmalloc_obj
    value: *mut u8,        // user ptr (GET dest, SET src already copied into kvalue)
    cvalue: *const u8,     // user ptr (SET src, before copy)
    kvalue: *mut u8,       // kernel buffer (SET; allocated by setxattr_copy)
    size: usize,
    flags: u32,            // XATTR_CREATE | XATTR_REPLACE; 0 for GET
  },
  filename: DelayedFilename,
}
```

`IoUringXattr::xattr_cleanup(req)`:
1. dismiss_delayed_filename(&mut ix.filename).
2. kfree(ix.ctx.kname).
3. kvfree(ix.ctx.kvalue).

`IoUringXattr::xattr_finish(req, ret)`:
1. req.flags &= !REQ_F_NEED_CLEANUP.
2. Self::xattr_cleanup(req).
3. io_req_set_res(req, ret, 0).

`IoUringXattr::getxattr_prep_common(req, sqe) -> i32`:
1. INIT_DELAYED_FILENAME(&mut ix.filename).
2. ix.ctx.kvalue = ptr::null_mut().
3. let name = sqe.addr as *const c_char.
4. ix.ctx.value = sqe.addr2 as *mut u8.
5. ix.ctx.size = sqe.len as usize.
6. ix.ctx.flags = sqe.xattr_flags.
7. if ix.ctx.flags != 0 { return -EINVAL; }
8. ix.ctx.kname = kmalloc_obj::<XattrName>(); if ix.ctx.kname.is_null() { return -ENOMEM; }
9. let ret = import_xattr_name(ix.ctx.kname, name); if ret != 0 { kfree(ix.ctx.kname); return ret; }
10. req.flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC.
11. return 0.

`IoUringXattr::fgetxattr_prep(req, sqe) -> i32`:
1. Self::getxattr_prep_common(req, sqe).

`IoUringXattr::getxattr_prep(req, sqe) -> i32`:
1. if req.flags.contains(REQ_F_FIXED_FILE) { return -EBADF; }
2. let ret = Self::getxattr_prep_common(req, sqe); if ret != 0 { return ret; }
3. let path = sqe.addr3 as *const c_char.
4. delayed_getname(&mut ix.filename, path).

`IoUringXattr::fgetxattr(req, issue_flags) -> IouResult`:
1. debug_assert!(!issue_flags.contains(IO_URING_F_NONBLOCK)).
2. let ret = file_getxattr(req.file, &mut ix.ctx).
3. Self::xattr_finish(req, ret); IouResult::Complete.

`IoUringXattr::getxattr(req, issue_flags) -> IouResult`:
1. debug_assert!(!issue_flags.contains(IO_URING_F_NONBLOCK)).
2. let name = filename_complete_delayed(&mut ix.filename).
3. let ret = filename_getxattr(AT_FDCWD, &name, LOOKUP_FOLLOW, &mut ix.ctx).
4. Self::xattr_finish(req, ret); IouResult::Complete.

`IoUringXattr::setxattr_prep_common(req, sqe) -> i32`:
1. INIT_DELAYED_FILENAME(&mut ix.filename).
2. let name = sqe.addr as *const c_char.
3. ix.ctx.cvalue = sqe.addr2 as *const u8.
4. ix.ctx.kvalue = ptr::null_mut().
5. ix.ctx.size = sqe.len as usize.
6. ix.ctx.flags = sqe.xattr_flags.
7. ix.ctx.kname = kmalloc_obj::<XattrName>(); if ix.ctx.kname.is_null() { return -ENOMEM; }
8. let ret = setxattr_copy(name, &mut ix.ctx); if ret != 0 { kfree(ix.ctx.kname); return ret; }
9. req.flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC.
10. return 0.

`IoUringXattr::setxattr_prep(req, sqe) -> i32`:
1. if req.flags.contains(REQ_F_FIXED_FILE) { return -EBADF; }
2. let ret = Self::setxattr_prep_common(req, sqe); if ret != 0 { return ret; }
3. let path = sqe.addr3 as *const c_char.
4. delayed_getname(&mut ix.filename, path).

`IoUringXattr::fsetxattr_prep(req, sqe) -> i32`:
1. Self::setxattr_prep_common(req, sqe).

`IoUringXattr::fsetxattr(req, issue_flags) -> IouResult`:
1. debug_assert!(!issue_flags.contains(IO_URING_F_NONBLOCK)).
2. let ret = file_setxattr(req.file, &mut ix.ctx).
3. Self::xattr_finish(req, ret); IouResult::Complete.

`IoUringXattr::setxattr(req, issue_flags) -> IouResult`:
1. debug_assert!(!issue_flags.contains(IO_URING_F_NONBLOCK)).
2. let name = filename_complete_delayed(&mut ix.filename).
3. let ret = filename_setxattr(AT_FDCWD, &name, LOOKUP_FOLLOW, &mut ix.ctx).
4. Self::xattr_finish(req, ret); IouResult::Complete.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `getxattr_prep_flags_zero` | INVARIANT | per-__io_getxattr_prep: ix.ctx.flags != 0 ⟹ -EINVAL. |
| `getxattr_path_rejects_fixed_file` | INVARIANT | per-io_getxattr_prep: REQ_F_FIXED_FILE ⟹ -EBADF. |
| `setxattr_path_rejects_fixed_file` | INVARIANT | per-io_setxattr_prep: REQ_F_FIXED_FILE ⟹ -EBADF. |
| `xattr_prep_sets_force_async_and_cleanup` | INVARIANT | per-all-prep: success ⟹ REQ_F_FORCE_ASYNC ∧ REQ_F_NEED_CLEANUP set. |
| `xattr_prep_kname_freed_on_import_failure` | INVARIANT | per-__io_getxattr_prep: import_xattr_name err ⟹ kfree(kname). |
| `xattr_prep_kname_freed_on_setxattr_copy_failure` | INVARIANT | per-__io_setxattr_prep: setxattr_copy err ⟹ kfree(kname). |
| `xattr_issue_never_nonblock` | INVARIANT | per-all-issue: WARN_ON_ONCE on IO_URING_F_NONBLOCK; VFS xattr never called in atomic. |
| `xattr_finish_clears_need_cleanup` | INVARIANT | per-xattr_finish: REQ_F_NEED_CLEANUP cleared before cleanup runs. |
| `xattr_cleanup_idempotent_after_finish` | INVARIANT | per-cleanup: kname / kvalue / filename freed at most once. |
| `xattr_kvalue_kvfree_safe` | INVARIANT | per-cleanup: kvalue may have been vmalloc'd; kvfree handles both. |

### Layer 2: TLA+

`io_uring/xattr.tla`:
- States: prep_idle → prep_alloc_kname → prep_import_name | prep_setxattr_copy → prep_done → queued → issued → completed | cancelled.
- Per-GETXATTR / FGETXATTR / SETXATTR / FSETXATTR traces.
- Properties:
  - `safety_kname_balanced` — per-trace: every successful kmalloc_obj matches exactly one kfree (via cleanup or prep-failure path).
  - `safety_kvalue_balanced` — per-trace: every setxattr_copy-allocated kvalue matches exactly one kvfree.
  - `safety_filename_balanced` — per-trace: every delayed_getname matches exactly one of {filename_complete_delayed, dismiss_delayed_filename}.
  - `safety_force_async_always` — per-prep: REQ_F_FORCE_ASYNC always set; inline path unreachable.
  - `safety_path_no_fixed_file` — per-{io_getxattr_prep, io_setxattr_prep}: REQ_F_FIXED_FILE ⟹ -EBADF before alloc.
  - `liveness_xattr_completes` — per-issue: every queued op terminates with CQE.
  - `liveness_cancelled_xattr_releases` — per-cancel: cleanup releases kname + kvalue + filename.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `getxattr_prep_common` post: ret == 0 ⟹ kname != NULL ∧ ctx.flags == 0 ∧ REQ_F_NEED_CLEANUP ∧ REQ_F_FORCE_ASYNC set | `IoUringXattr::getxattr_prep_common` |
| `setxattr_prep_common` post: ret == 0 ⟹ kname != NULL ∧ kvalue initialized by setxattr_copy ∧ REQ_F_NEED_CLEANUP ∧ REQ_F_FORCE_ASYNC set | `IoUringXattr::setxattr_prep_common` |
| `xattr_finish` post: REQ_F_NEED_CLEANUP cleared; kname / kvalue / filename all freed | `IoUringXattr::xattr_finish` |
| `xattr_cleanup` post: kname / kvalue / filename are NULL or freed | `IoUringXattr::xattr_cleanup` |
| `io_fgetxattr` post: CQE result == file_getxattr return | `IoUringXattr::fgetxattr` |
| `io_getxattr` post: CQE result == filename_getxattr return | `IoUringXattr::getxattr` |
| `io_fsetxattr` post: CQE result == file_setxattr return | `IoUringXattr::fsetxattr` |
| `io_setxattr` post: CQE result == filename_setxattr return | `IoUringXattr::setxattr` |

### Layer 4: Verus/Creusot functional

`Per-{F,}{GET,SET}XATTR SQE → kmalloc_obj kname → {import_xattr_name | setxattr_copy} → optional delayed_getname → REQ_F_FORCE_ASYNC enqueue → worker thread → {file_,filename_}{get,set}xattr → CQE → io_xattr_finish` semantic equivalence: per-getxattr(2) / setxattr(2) / fgetxattr(2) / fsetxattr(2) and Documentation/io_uring.7 §IORING_OP_GETXATTR / SETXATTR / FGETXATTR / FSETXATTR. Each op's result equals that of the corresponding synchronous syscall with the calling task's credentials and AT_FDCWD anchoring for path variants.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

XATTR reinforcement:

- **Per-path-variant REQ_F_FIXED_FILE rejection** — defense against per-confusion where a ring-fixed fd is passed where a path is expected.
- **Per-getxattr xattr_flags must be zero** — defense against per-future-flag silent-acceptance (no GET flags are defined; future flags must be explicit).
- **Per-prep kname alloc-then-validate ordering** — defense against per-leak: if import_xattr_name / setxattr_copy fails, kname is freed before returning.
- **Per-prep force-async unconditional** — defense against per-blocking-path on submitter thread (filesystem xattr handlers may sleep on inode locks, journal commits, network round-trips).
- **Per-issue WARN_ON_ONCE on NONBLOCK** — defense-in-depth assertion that force-async invariant held.
- **Per-CLASS-scoped filename_complete_delayed for path variants** — defense against per-leak of struct filename on early return from VFS xattr.
- **Per-xattr_finish REQ_F_NEED_CLEANUP cleared before free** — defense against per-double-free on subsequent cancel.
- **Per-cleanup kvfree for ctx.kvalue** — defense against per-UB (free) when value buffer was vmalloc'd by setxattr_copy for large xattrs.
- **Per-import_xattr_name length-cap (XATTR_NAME_MAX)** — defense against per-DOS via giant xattr names.
- **Per-setxattr_copy size-cap (XATTR_SIZE_MAX)** — defense against per-DOS via giant xattr values.
- **Per-credential inheritance through io-wq** — defense against per-privilege-escalation through ring polling; xattr permission checks run under submitter creds.
- **Per-LOOKUP_FOLLOW explicit on path variants** — defense against per-policy-drift; matches the synchronous syscall semantics.
- **Per-AT_FDCWD anchor on path variants** — defense against per-confusion with dfd-anchored xattr (no AT_FDCWD field in the prep — io_uring does not expose dfd here, matching the kernel's path-only entry points).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-check `strncpy_from_user(filename)`, `strncpy_from_user(name)`, and `copy_from_user(value)` against the slab whitelist for the per-request `struct io_xattr` and `struct xattr_ctx`; reject any copy that crosses the 256B name limit or `XATTR_SIZE_MAX` value limit.
- **PAX_KERNEXEC** — keep `io_op_def[IORING_OP_*XATTR]`, VFS `xattr_handler` tables, and per-FS `->set`/`->get` vectors in `__ro_after_init`; deny late patching by an out-of-tree FS module.
- **PAX_RANDKSTACK** — re-randomize kernel stack on `io_setxattr_prep`/`io_getxattr_prep` to defeat stack disclosure of the `struct xattr_name` buffer.
- **PAX_REFCOUNT** — saturating `path_get`/`path_put` on the resolved `struct path` and on submitter creds passed through io-wq; deny wraparound.
- **PAX_MEMORY_SANITIZE** — zero `xattr_ctx.kvalue` and the `xattr_name` page on free; never let a prior `security.selinux` or `trusted.*` value leak into a recycled SQE.
- **PAX_UDEREF** — strict user/kernel separation while parsing the `name __user *` and `value __user *`; reject any kernel pointer smuggled via `addr`/`addr2`.
- **PAX_RAP / kCFI** — type-check indirect calls to `->set`, `->get`, `->list` xattr handlers and the LSM `security_inode_setxattr` chain.
- **GRKERNSEC_HIDESYM** — hide `vfs_setxattr`, `__vfs_getxattr`, and per-FS xattr handler addresses from non-root kallsyms.
- **GRKERNSEC_DMESG** — restrict dmesg so xattr-handler error spew (which often echoes attribute names and FS internals) is not harvestable.
- **`trusted.*` requires `CAP_SYS_ADMIN`** — re-check the cap against the *submitter* creds latched at prep time, not the io-wq worker; refuse if creds were SCM_RIGHTS-laundered.
- **PAX_USERCOPY on path/name** — the path (`PATH_MAX`) and name (`XATTR_NAME_MAX`) buffers MUST be allocated from usercopy-whitelisted caches; reject if the destination slab object is not whitelisted.
- **`security.*` namespace** — enforce LSM hook even when called via io_uring; the async path MUST NOT skip `security_inode_setxattr`/`security_inode_getxattr`.
- **Rationale** — xattr operations cross every LSM and per-FS handler with attacker-controlled name+value+path triples; without PAX_USERCOPY + strict cap re-checks, io_uring becomes a creds-laundering bypass for `trusted.*`/`security.*` namespaces that the synchronous syscalls guard carefully.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VFS xattr handlers (`file_getxattr`, `file_setxattr`, `filename_getxattr`, `filename_setxattr`, `setxattr_copy`, `import_xattr_name`) — covered in `fs/xattr.md` Tier-3.
- struct kernel_xattr_ctx layout — covered in `fs/xattr.md`.
- Delayed-filename machinery (`delayed_getname`, `filename_complete_delayed`) — covered in `fs/namei.md` Tier-3.
- Per-filesystem xattr backends (ext4/xfs/btrfs/9p/cifs) — covered in respective filesystem Tier-3 docs.
- Security xattr ↔ LSM hooks (`security_inode_getxattr`, `security_inode_setxattr`) — covered in `security/lsm.md`.
- vmalloc / kmalloc backing decision for kvalue — covered in `mm/slab.md` / `mm/vmalloc.md`.
- Implementation code.
