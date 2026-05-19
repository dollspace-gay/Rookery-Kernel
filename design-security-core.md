---
title: "Tier-3: security/security.c — LSM core (hook dispatch + composite blobs)"
tags: ["tier-3", "security", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The **Linux Security Module (LSM) framework core** is the per-callsite dispatch layer that translates `security_<op>()` calls embedded in the kernel (VFS, IPC, networking, ptrace, capabilities, binder, io_uring, BPF, ...) into per-active-LSM callbacks (SELinux, AppArmor, Smack, TOMOYO, Yama, Landlock, IPE, IMA/EVM, BPF-LSM, ...). Per-hook a fixed-size **static call table** (`struct lsm_static_calls_table`) holds up to `MAX_LSM_COUNT` slots; each enabled LSM patches one slot per hook via `security_add_hooks()` and `__static_call_update()` plus `static_branch_enable()` so disabled slots are zero-cost branch-elided. Per-object a **composite blob** (`cred->security`, `file->f_security`, `inode->i_security`, `task->security`, `sb->s_security`, `sock->sk_security`, ...) is co-allocated across all LSMs; per-LSM offset is computed at `lsm_prepare()` time from each LSM's `struct lsm_blob_sizes` and stored back into the LSM's own `lbs_*` fields (turning the request-size into the assigned offset). LSM hooks are dispatched via `call_int_hook(HOOK, ...)` (short-circuit on first non-default return) or `call_void_hook(HOOK, ...)` (all-LSMs invoked). Stacking allows multiple LSMs to co-exist subject to ordering (capabilities first, integrity last) and exclusivity (`LSM_FLAG_EXCLUSIVE` only one major LSM that touches `cred`/`task` blobs). Critical for: MAC enforcement, capabilities, integrity, auditing, namespace confinement.

This Tier-3 covers `security/security.c` (~5718 lines) plus the closely-coupled init pieces in `security/lsm_init.c`.

### Acceptance Criteria

- [ ] AC-1: A LSM declared via DEFINE_LSM with a non-NULL `init` is invoked from `security_init()` after `lsm_prepare()`.
- [ ] AC-2: A LSM declared via DEFINE_EARLY_LSM is invoked from `early_security_init()` before `security_init()`.
- [ ] AC-3: security_add_hooks(hooks, count, lsmid) patches `count` hook slots; each LSM's hooks[i].lsmid == lsmid post-call.
- [ ] AC-4: If a hook is registered MAX_LSM_COUNT+1 times, the (MAX_LSM_COUNT+1)th call panics with "exhausted LSM callback slots".
- [ ] AC-5: After security_init(), `static_calls_table` is __ro_after_init (writes after init faults).
- [ ] AC-6: After lsm_prepare(lsm), each LSM's `blobs->lbs_*` field equals the offset of that LSM's region within the composite blob (not its original size request).
- [ ] AC-7: blob_sizes.lbs_inode is at least sizeof(struct rcu_head) if any LSM requests an inode blob.
- [ ] AC-8: lsm_cred_alloc / lsm_task_alloc / lsm_inode_alloc / lsm_file_alloc / lsm_ipc_alloc / lsm_superblock_alloc / lsm_msg_msg_alloc / lsm_bdev_alloc / lsm_key_alloc / lsm_bpf_*_alloc each kzalloc a single blob of the framework-computed size; per-LSM views the blob at its offset.
- [ ] AC-9: call_int_hook short-circuits on the first non-LSM_RET_DEFAULT return (deny semantics).
- [ ] AC-10: call_void_hook invokes every active LSM's callback.
- [ ] AC-11: A disabled LSM's hook slot has its static_branch_false key disabled — the dispatch branch is elided.
- [ ] AC-12: security_binder_set_context_mgr / _transaction / _transfer_binder / _transfer_file return 0 when no LSM is loaded (LSM_RET_DEFAULT == 0).
- [ ] AC-13: security_capable() returns the LSM verdict (0 grant; -EPERM/-EACCES deny).
- [ ] AC-14: lsm_fill_user_ctx with NULL uctx returns 0 and sets *uctx_len to the required size; with too-small *uctx_len returns -E2BIG; success copies the lsm_ctx to user.
- [ ] AC-15: security_initramfs_populated() fires the per-LSM initramfs_populated callback exactly once after rootfs unpack.

### Architecture

```
struct LsmId { name: &'static str, id: u64 }

struct LsmBlobSizes {
  lbs_cred:        u32,  lbs_file:        u32,  lbs_backing_file: u32,
  lbs_ib:          u32,  lbs_inode:       u32,  lbs_sock:         u32,
  lbs_superblock:  u32,  lbs_ipc:         u32,  lbs_key:          u32,
  lbs_msg_msg:     u32,  lbs_perf_event:  u32,  lbs_task:         u32,
  lbs_xattr_count: u32,  lbs_tun_dev:     u32,  lbs_bdev:         u32,
  lbs_bpf_map:     u32,  lbs_bpf_prog:    u32,  lbs_bpf_token:    u32,
}

enum LsmOrder { First = -1, Mutable = 0, Last = 1 }

struct LsmInfo {
  id:              &'static LsmId,
  order:           LsmOrder,
  flags:           u32,                    // LSM_FLAG_LEGACY_MAJOR | LSM_FLAG_EXCLUSIVE
  blobs:           Option<&'static mut LsmBlobSizes>,
  enabled:         Option<&'static mut bool>,
  init:            fn() -> KResult<()>,
  initcall_pure:   Option<fn() -> KResult<()>>,
  initcall_early:  Option<fn() -> KResult<()>>,
  initcall_core:   Option<fn() -> KResult<()>>,
  initcall_subsys: Option<fn() -> KResult<()>>,
  initcall_fs:     Option<fn() -> KResult<()>>,
  initcall_device: Option<fn() -> KResult<()>>,
  initcall_late:   Option<fn() -> KResult<()>>,
}

union SecurityListOptions {                  // ABI-compatible
  // one fn-ptr per LSM hook, generated from lsm_hook_defs.h
  lsm_func_addr: *mut (),
  // ... binder_set_context_mgr: fn(&Cred) -> KResult<()>, ...
}

struct SecurityHookList {
  scalls: *mut LsmStaticCall,                // base of static_calls_table.<HOOK>
  hook:   SecurityListOptions,
  lsmid:  *const LsmId,
}

struct LsmStaticCall {
  key:        *mut StaticCallKey,
  trampoline: *mut (),                       // STATIC_CALL_TRAMP or NULL
  hl:         Option<*mut SecurityHookList>,
  active:     *mut StaticKeyFalse,
}

struct LsmStaticCallsTable {
  // generated: lsm_static_call <HOOK>[MAX_LSM_COUNT] for every hook in lsm_hook_defs.h
}
```

`Lsm::early_init() -> KResult<()>`:
1. for lsm in .early_lsm_info.init:
   - Lsm::enabled_set(lsm, true).
   - Lsm::order_append(lsm, "early").
   - Lsm::prepare(lsm).
   - Lsm::init_single(lsm).
   - LSM_COUNT_EARLY.fetch_add(1).
2. Ok(()).

`Lsm::init() -> KResult<()>`:
1. /* Order resolution */
2. if let Some(cl) = lsm_order_cmdline:
   - lsm_order_legacy = None.
   - Lsm::order_parse(cl, "cmdline").
3. else: Lsm::order_parse(lsm_order_builtin, "builtin").
4. /* Phase 1: compute composite blob layout */
5. for lsm in order: Lsm::prepare(lsm).
6. /* Phase 2: provision SLAB caches for kmem_cache-backed blobs */
7. if blob_sizes.lbs_file > 0: lsm_file_cache = kmem_cache_create("lsm_file_cache", lbs_file, 0, SLAB_PANIC, None).
8. if blob_sizes.lbs_backing_file > 0: lsm_backing_file_cache = kmem_cache_create(...).
9. if blob_sizes.lbs_inode > 0: lsm_inode_cache = kmem_cache_create(...).
10. /* Phase 3: per-LSM init() — typically calls Lsm::add_hooks() */
11. for lsm in order: Lsm::init_single(lsm).
12. Ok(()).

`Lsm::prepare(lsm)`:
1. let blobs = lsm.blobs.as_mut()?; (none ⇒ return).
2. Lsm::blob_size_update(&mut blobs.lbs_cred, &mut blob_sizes.lbs_cred).
3. Lsm::blob_size_update(&mut blobs.lbs_file, &mut blob_sizes.lbs_file).
4. Lsm::blob_size_update(&mut blobs.lbs_backing_file, &mut blob_sizes.lbs_backing_file).
5. Lsm::blob_size_update(&mut blobs.lbs_ib, &mut blob_sizes.lbs_ib).
6. /* Inode blob carries rcu_head */
7. if blobs.lbs_inode > 0 ∧ blob_sizes.lbs_inode == 0: blob_sizes.lbs_inode = size_of::<RcuHead>().
8. Lsm::blob_size_update(&mut blobs.lbs_inode, &mut blob_sizes.lbs_inode).
9. /* same for lbs_ipc, lbs_key, lbs_msg_msg, lbs_perf_event, lbs_sock, lbs_superblock, lbs_task, lbs_tun_dev, lbs_xattr_count, lbs_bdev, lbs_bpf_map, lbs_bpf_prog, lbs_bpf_token */

`Lsm::blob_size_update(sz_req, sz_cur)`:
1. if *sz_req == 0: return.
2. let offset = align_up(*sz_cur, size_of::<*const ()>()).
3. *sz_cur = offset + *sz_req.
4. *sz_req = offset.   /* in-place: request → assigned offset */

`Lsm::add_hooks(hooks: &[SecurityHookList], lsmid: &'static LsmId)` (__init):
1. for h in hooks:
   - h.lsmid = lsmid.
   - if Lsm::static_call_init(h).is_err(): panic("exhausted LSM callback slots with LSM {}", lsmid.name).

`Lsm::static_call_init(hl: &mut SecurityHookList) -> KResult<()>` (__init):
1. let mut scall = hl.scalls.
2. for _ in 0..MAX_LSM_COUNT:
   - if scall.hl.is_none():
     - __static_call_update(scall.key, scall.trampoline, hl.hook.lsm_func_addr).
     - scall.hl = Some(hl).
     - static_branch_enable(scall.active).
     - return Ok(()).
   - scall = scall.add(1).
3. Err(ENOSPC).

`Lsm::call_int_hook!(HOOK, args...)` (macro, all MAX_LSM_COUNT slots unrolled):
1. let mut rc = LSM_RET_DEFAULT(HOOK);
2. for NUM in 0..MAX_LSM_COUNT:
   - if static_branch_unlikely(ACTIVE_KEY(HOOK, NUM)):
     - rc = static_call!(LSM_STATIC_CALL(HOOK, NUM))(args).
     - if rc != LSM_RET_DEFAULT(HOOK): goto OUT.
3. OUT: rc.

`Lsm::call_void_hook!(HOOK, args...)`:
1. for NUM in 0..MAX_LSM_COUNT:
   - if static_branch_unlikely(ACTIVE_KEY(HOOK, NUM)):
     - static_call!(LSM_STATIC_CALL(HOOK, NUM))(args).

`Lsm::cred_alloc(cred: &mut Cred, gfp: Gfp) -> KResult<()>`:
1. Lsm::blob_alloc(&mut cred.security, blob_sizes.lbs_cred, gfp).

`Lsm::task_alloc(task: &mut TaskStruct) -> KResult<()>`:
1. Lsm::blob_alloc(&mut task.security, blob_sizes.lbs_task, Gfp::KERNEL).

`Lsm::inode_alloc(inode: &mut Inode, gfp: Gfp) -> KResult<()>`:
1. if !lsm_inode_cache: inode.i_security = None; return Ok(()).
2. inode.i_security = kmem_cache_zalloc(lsm_inode_cache, gfp).
3. if inode.i_security.is_none(): Err(ENOMEM) else Ok(()).

`Lsm::file_alloc(file: &mut File) -> KResult<()>`:
1. if !lsm_file_cache: file.f_security = None; return Ok(()).
2. file.f_security = kmem_cache_zalloc(lsm_file_cache, Gfp::KERNEL).
3. if file.f_security.is_none(): Err(ENOMEM) else Ok(()).

`Lsm::fill_user_ctx(uctx, uctx_len, val, val_len, id, flags) -> KResult<()>`:
1. let nctx_len = align_up(size_of::<LsmCtx>() + val_len, size_of::<*const ()>()).
2. if nctx_len > *uctx_len: rc = -E2BIG; goto out.
3. if uctx.is_null(): goto out (success).
4. nctx = kzalloc(nctx_len, Gfp::KERNEL); if !nctx: rc = -ENOMEM; goto out.
5. nctx.id = id; nctx.flags = flags; nctx.len = nctx_len; nctx.ctx_len = val_len; memcpy(nctx.ctx, val, val_len).
6. if copy_to_user(uctx, nctx, nctx_len).is_err(): rc = -EFAULT.
7. out: kfree(nctx); *uctx_len = nctx_len; rc.

`Lsm::<hook>(args...)` per-callsite wrapper (e.g. `Lsm::binder_set_context_mgr(mgr)`):
1. return call_int_hook!(binder_set_context_mgr, mgr).

### Out of Scope

- `security/lsm_audit.c` common audit helpers (covered in `lsm-audit.md` Tier-3).
- `security/selinux/` (covered under `security/selinux/00-overview.md` and Tier-3 docs there).
- `security/apparmor/` (covered under `security/apparmor/00-overview.md`).
- `security/smack/`, `security/tomoyo/`, `security/yama/`, `security/landlock/`, `security/loadpin/`, `security/lockdown/`, `security/ipe/`, `security/safesetid/` (covered separately if expanded).
- `security/integrity/` (covered under `security/integrity/00-overview.md`).
- `security/keys/` (covered under `security/keys/00-overview.md`).
- `security/commoncap.c` POSIX capability LSM (covered separately if expanded).
- `kernel/cred.c` cred lifecycle (callsites only; not the LSM core).
- `kernel/audit*.c` audit subsystem (callsites only).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct lsm_id` | per-LSM identifier (name + LSM_ID_*) | `LsmId` |
| `struct lsm_info` | per-LSM registration record (.lsm_info.init / .early_lsm_info.init) | `LsmInfo` |
| `struct lsm_blob_sizes` | per-object blob size request / per-LSM offset | `LsmBlobSizes` |
| `struct security_hook_list` | per-LSM-per-hook callback entry | `SecurityHookList` |
| `struct lsm_static_call` | per-hook static-call slot (key, trampoline, hl, active) | `LsmStaticCall` |
| `struct lsm_static_calls_table` | fixed table of MAX_LSM_COUNT slots per hook | `LsmStaticCallsTable` |
| `union security_list_options` | per-hook typed function pointer | `SecurityListOptions` |
| `security_add_hooks()` | per-LSM register hooks | `Lsm::add_hooks` |
| `lsm_static_call_init()` | per-hook patch first free slot | `Lsm::static_call_init` |
| `early_security_init()` | per-early-LSM phase init | `Lsm::early_init` |
| `security_init()` | per-LSM-framework init | `Lsm::init` |
| `lsm_prepare()` | per-LSM compute blob offsets | `Lsm::prepare` |
| `lsm_init_single()` | per-LSM call .init | `Lsm::init_single` |
| `lsm_blob_size_update()` | per-blob accumulate size + assign offset | `Lsm::blob_size_update` |
| `lsm_blob_alloc()` / `lsm_cred_alloc()` / `lsm_task_alloc()` / `lsm_inode_alloc()` / `lsm_file_alloc()` / `lsm_ipc_alloc()` / `lsm_superblock_alloc()` / `lsm_msg_msg_alloc()` / `lsm_bdev_alloc()` / `lsm_key_alloc()` / `lsm_backing_file_alloc()` / `lsm_bpf_map_alloc()` / `lsm_bpf_prog_alloc()` / `lsm_bpf_token_alloc()` | per-object composite blob alloc | `Lsm::blob_alloc_*` |
| `call_int_hook(HOOK, ...)` | per-hook short-circuit dispatch | `Lsm::call_int_hook!` |
| `call_void_hook(HOOK, ...)` | per-hook all-LSM dispatch | `Lsm::call_void_hook!` |
| `lsm_for_each_hook(scall, NAME)` | per-hook active iterator | `Lsm::for_each_hook` |
| `LSM_RET_DEFAULT(NAME)` | per-hook default-return constant | `Lsm::ret_default!` |
| `security_binder_set_context_mgr()` / `_transaction()` / `_transfer_binder()` / `_transfer_file()` | binder callsites | `Lsm::binder_*` |
| `security_ptrace_access_check()` / `_traceme()` | ptrace callsites | `Lsm::ptrace_*` |
| `security_capget()` / `_capset()` / `_capable()` | capability callsites | `Lsm::cap_*` |
| `security_vm_enough_memory_mm()` | mm overcommit hook | `Lsm::vm_enough_memory_mm` |
| `security_bprm_*` (creds_for_exec, creds_from_file, check, committing_creds, committed_creds) | per-execve callsites | `Lsm::bprm_*` |
| `security_sb_*` (alloc, free, delete, eat_lsm_opts, mnt_opts_compat, remount, kern_mount, show_options, statfs, mount, umount, pivotroot, set_mnt_opts, clone_mnt_opts, move_mount) | per-superblock | `Lsm::sb_*` |
| `security_inode_*` (alloc, free, init_security, init_security_anon, permission, setattr, getattr, getxattr, setxattr, removexattr, listxattr, link, unlink, mkdir, rmdir, mknod, symlink, rename, readlink, follow_link, getsecurity, setsecurity, listsecurity, copy_up, file_setattr, file_getattr, post_*) | per-inode | `Lsm::inode_*` |
| `security_path_*` (mknod, mkdir, rmdir, unlink, symlink, link, rename, truncate, chmod, chown, chroot, notify) | per-path (CONFIG_SECURITY_PATH) | `Lsm::path_*` |
| `security_file_*` (alloc, free, permission, ioctl, mprotect, mmap_*, fcntl, lock, send_sigiotask, receive, open, post_open, truncate, set_fowner) | per-file | `Lsm::file_*` |
| `security_task_*` (alloc, free, fix_setuid, fix_setgid, fix_setgroups, setpgid, getpgid, getsid, getsecid_subj, getsecid_obj, setnice, setioprio, getioprio, prlimit, setrlimit, setscheduler, getscheduler, movememory, kill, prctl, to_inode) | per-task | `Lsm::task_*` |
| `security_cred_*` (alloc_blank, free, prepare_creds, transfer_creds, getsecid, getlsmprop) | per-cred composite blob | `Lsm::cred_*` |
| `security_kernel_*` (act_as, create_files_as, module_request, load_data, read_file, post_read_file, module_from_file) | per-kernel-load | `Lsm::kernel_*` |
| `security_ipc_*` (permission, getsecid) / `security_msg_msg_*` / `security_msg_queue_*` / `security_shm_*` / `security_sem_*` | per-SysV-IPC | `Lsm::ipc_*` |
| `security_socket_*` (create, post_create, socketpair, bind, connect, listen, accept, sendmsg, recvmsg, getsockname, getpeername, setsockopt, getsockopt, shutdown, sock_rcv_skb, getpeersec_stream, getpeersec_dgram) / `security_sk_*` (alloc, free, clone_security, classify_flow, getsecid, get_peer_sec) | per-net | `Lsm::socket_*` / `Lsm::sk_*` |
| `security_key_*` (alloc, free, permission, getsecurity, post_create_or_update) | per-key | `Lsm::key_*` |
| `security_audit_*` / `security_setprocattr()` / `security_getprocattr()` / `security_secctx_to_secid()` / `security_secid_to_secctx()` / `security_release_secctx()` | per-audit/procattr | `Lsm::audit_*` |
| `security_bpf_*` (map, map_alloc, map_free, prog, prog_alloc, prog_free, token_alloc, token_free, token_capable, token_cmd) | per-BPF | `Lsm::bpf_*` |
| `security_perf_event_*` / `security_uring_*` | per-perf / per-io_uring | `Lsm::perf_*` / `Lsm::uring_*` |
| `security_locked_down()` | per-lockdown | `Lsm::locked_down` |
| `security_initramfs_populated()` | per-rootfs unpack notification | `Lsm::initramfs_populated` |
| `lsm_fill_user_ctx()` | per-userspace ctx copy-out | `Lsm::fill_user_ctx` |
| `lockdown_reasons[]` | per-LOCKDOWN_*_MAX descriptions | shared |
| `lsm_debug` / `lsm_active_cnt` / `lsm_idlist[MAX_LSM_COUNT]` / `blob_sizes` / `lsm_file_cache` / `lsm_backing_file_cache` / `lsm_inode_cache` | per-LSM globals | shared |
| `static_calls_table` | per-hook MAX_LSM_COUNT static-call table | shared |

### compatibility contract

REQ-1: struct lsm_id (uapi/linux/lsm.h):
- name: const char * (e.g. "selinux", "apparmor", "smack", "tomoyo", "yama", "landlock", "lockdown", "loadpin", "safesetid", "ipe", "bpf", "ima", "evm", "capability").
- id: u64 LSM_ID_* assigned by maintainers; stable UAPI value referenced by lsm_ctx.

REQ-2: struct lsm_blob_sizes:
- lbs_cred, lbs_file, lbs_backing_file, lbs_ib, lbs_inode, lbs_sock, lbs_superblock, lbs_ipc, lbs_key, lbs_msg_msg, lbs_perf_event, lbs_task, lbs_xattr_count, lbs_tun_dev, lbs_bdev, lbs_bpf_map, lbs_bpf_prog, lbs_bpf_token.
- /* Per-LSM declares this struct __ro_after_init with its requested sizes */
- /* After lsm_prepare() the LSM's own lbs_* fields are rewritten to the offset of that LSM's region within the composite blob (input size → output offset semantics) */
- /* The framework's global `blob_sizes` accumulates total sizes used to allocate composite blobs */

REQ-3: struct security_hook_list:
- scalls: pointer into static_calls_table.<HOOK>[0..MAX_LSM_COUNT].
- hook: union security_list_options { RET (*HOOK_NAME)(args); ... void *lsm_func_addr; }.
- lsmid: const struct lsm_id *.
- /* Initialized via LSM_HOOK_INIT(NAME, HOOK) macro */

REQ-4: struct lsm_static_call:
- key: struct static_call_key * (per-CONFIG_HAVE_STATIC_CALL: real key; else NULL trampoline).
- trampoline: void * (STATIC_CALL_TRAMP) when arch supports static calls.
- hl: struct security_hook_list * (populated when slot is taken; NULL slot is free).
- active: struct static_key_false * (jump-label gate; disabled by default, enabled when hl is set).

REQ-5: struct lsm_static_calls_table:
- Per-hook array `struct lsm_static_call NAME[MAX_LSM_COUNT];` generated from `<linux/lsm_hook_defs.h>` via X-macros.
- `__ro_after_init __aligned(sizeof(u64))` — frozen after `security_init()`; aligned to avoid early-init unaligned-access faults.
- `__packed __randomize_layout`.

REQ-6: struct lsm_info:
- id: const struct lsm_id *.
- order: enum lsm_order { LSM_ORDER_FIRST = -1 (capability only), LSM_ORDER_MUTABLE = 0, LSM_ORDER_LAST = 1 (integrity only) }.
- flags: LSM_FLAG_LEGACY_MAJOR | LSM_FLAG_EXCLUSIVE.
- blobs: struct lsm_blob_sizes * (optional; NULL if LSM uses no blobs).
- enabled: int * (controlled by CONFIG_LSM, optional).
- init: int (*init)(void) — LSM-specific init (typically calls security_add_hooks()).
- initcall_pure / _early / _core / _subsys / _fs / _device / _late: optional per-phase initcall fan-out.
- /* Placed in .lsm_info.init via DEFINE_LSM(lsm) or .early_lsm_info.init via DEFINE_EARLY_LSM(lsm) */

REQ-7: security_add_hooks(hooks, count, lsmid):
- /* __init only */
- for i in 0..count:
  - hooks[i].lsmid = lsmid.
  - if lsm_static_call_init(&hooks[i]) != 0: panic("exhausted LSM callback slots with LSM %s", lsmid->name).

REQ-8: lsm_static_call_init(hl):
- scall = hl->scalls (= &static_calls_table.NAME[0]).
- for i in 0..MAX_LSM_COUNT:
  - if scall->hl == NULL:
    - __static_call_update(scall->key, scall->trampoline, hl->hook.lsm_func_addr).
    - scall->hl = hl.
    - static_branch_enable(scall->active).
    - return 0.
  - scall++.
- return -ENOSPC.

REQ-9: early_security_init():
- /* Phase 1: built-in early LSMs (DEFINE_EARLY_LSM, e.g. capability, lockdown, integrity, landlock) */
- /* lsm_pr_dbg() not usable yet (lsm_debug unset) */
- for each lsm in .early_lsm_info.init:
  - lsm_enabled_set(lsm, true).
  - lsm_order_append(lsm, "early").
  - lsm_prepare(lsm).
  - lsm_init_single(lsm).
  - lsm_count_early++.
- return 0.

REQ-10: security_init():
- /* Phase 2: ordinary LSMs */
- /* Emit debug listing if lsm_debug */
- if lsm_order_cmdline: parse cmdline (overrides legacy).
- else: parse builtin (CONFIG_LSM).
- for each lsm in order: lsm_prepare(lsm).
- /* Compute lsm_file/lsm_inode/lsm_backing_file caches */
- if blob_sizes.lbs_file: lsm_file_cache = kmem_cache_create("lsm_file_cache", lbs_file, 0, SLAB_PANIC, NULL).
- if blob_sizes.lbs_backing_file: lsm_backing_file_cache = kmem_cache_create(...).
- if blob_sizes.lbs_inode: lsm_inode_cache = kmem_cache_create(...).
- for each lsm in order: lsm_init_single(lsm).
- return 0.

REQ-11: lsm_prepare(lsm):
- blobs = lsm->blobs.
- if !blobs: return.
- /* Per-blob accumulate sizes; each LSM's lbs_* is rewritten in-place to its assigned offset */
- lsm_blob_size_update(&blobs->lbs_cred, &blob_sizes.lbs_cred).
- lsm_blob_size_update(&blobs->lbs_file, &blob_sizes.lbs_file).
- lsm_blob_size_update(&blobs->lbs_backing_file, &blob_sizes.lbs_backing_file).
- lsm_blob_size_update(&blobs->lbs_ib, &blob_sizes.lbs_ib).
- /* Inode blob carries an rcu_head; allocate one if any LSM requests an inode blob */
- if blobs->lbs_inode ∧ blob_sizes.lbs_inode == 0: blob_sizes.lbs_inode = sizeof(struct rcu_head).
- lsm_blob_size_update(&blobs->lbs_inode, &blob_sizes.lbs_inode).
- lsm_blob_size_update for: lbs_ipc, lbs_key, lbs_msg_msg, lbs_perf_event, lbs_sock, lbs_superblock, lbs_task, lbs_tun_dev, lbs_xattr_count, lbs_bdev, lbs_bpf_map, lbs_bpf_prog, lbs_bpf_token.

REQ-12: lsm_blob_size_update(sz_req, sz_cur):
- /* sz_req is the LSM's requested size (input); on return it holds the assigned offset */
- if *sz_req == 0: return.
- offset = ALIGN(*sz_cur, sizeof(void *)).
- *sz_cur = offset + *sz_req.
- *sz_req = offset.

REQ-13: lsm_blob_alloc(dest, size, gfp):
- if size == 0: *dest = NULL; return 0.
- *dest = kzalloc(size, gfp).
- if !*dest: return -ENOMEM.
- return 0.

REQ-14: Per-object composite-blob allocators (single-allocation, all-LSMs):
- lsm_cred_alloc(cred, gfp): lsm_blob_alloc(&cred->security, blob_sizes.lbs_cred, gfp).
- lsm_task_alloc(task): lsm_blob_alloc(&task->security, blob_sizes.lbs_task, GFP_KERNEL).
- lsm_inode_alloc(inode, gfp): kmem_cache_zalloc(lsm_inode_cache, gfp) → inode->i_security.
- lsm_file_alloc(file): kmem_cache_zalloc(lsm_file_cache, GFP_KERNEL) → file->f_security.
- lsm_backing_file_alloc(backing_file): kmem_cache_zalloc(lsm_backing_file_cache, GFP_KERNEL).
- lsm_superblock_alloc(sb): lsm_blob_alloc(&sb->s_security, lbs_superblock, GFP_KERNEL).
- lsm_ipc_alloc(kip): lsm_blob_alloc(&kip->security, lbs_ipc, GFP_KERNEL).
- lsm_msg_msg_alloc(mp): lsm_blob_alloc(&mp->security, lbs_msg_msg, GFP_KERNEL).
- lsm_bdev_alloc(bdev): lsm_blob_alloc(&bdev->bd_security, lbs_bdev, GFP_KERNEL).
- lsm_key_alloc(key) [CONFIG_KEYS]: lsm_blob_alloc(&key->security, lbs_key, GFP_KERNEL).
- lsm_bpf_map_alloc(map) / lsm_bpf_prog_alloc(prog) / lsm_bpf_token_alloc(token) [CONFIG_BPF_SYSCALL].

REQ-15: Per-hook dispatch macros (X-macro generated):
- LSM_LOOP_UNROLL(M, ...): UNROLL(MAX_LSM_COUNT, M, ...).
- __CALL_STATIC_VOID(NUM, HOOK, ...): if static_branch_unlikely(active_##HOOK##_##NUM): static_call(LSM_STATIC_CALL(HOOK, NUM))(...).
- call_void_hook(HOOK, ...): LSM_LOOP_UNROLL(__CALL_STATIC_VOID, HOOK, ...) — all-LSMs invoked.
- __CALL_STATIC_INT(NUM, R, HOOK, LABEL, ...): if active: R = static_call(...)(...); if R != LSM_RET_DEFAULT(HOOK): goto LABEL — short-circuit on first non-default (typically non-zero / deny).
- call_int_hook(HOOK, ...): RC = LSM_RET_DEFAULT(HOOK); unroll over MAX_LSM_COUNT slots; return RC.
- lsm_for_each_hook(scall, NAME): iterate static_calls_table.NAME guarded by static_key_enabled(&scall->active->key).

REQ-16: Per-callsite security_*() wrappers — uniform shape:
- int security_<op>(args...) { return call_int_hook(<op>, args...); }
- void security_<op>(args...) { call_void_hook(<op>, args...); }
- /* Per-EXPORT_SYMBOL / EXPORT_SYMBOL_GPL on a subset (e.g. security_free_mnt_opts, security_inode_init_security, security_path_mknod, security_path_mkdir, security_path_unlink, security_path_rename, security_inode_create, security_inode_mkdir, security_inode_listsecurity, security_inode_copy_up, security_dentry_init_security, security_dentry_create_files_as, security_sb_eat_lsm_opts, security_sb_mnt_opts_compat, security_sb_remount, security_sb_set_mnt_opts, security_sb_clone_mnt_opts, security_kernel_read_file, security_kernel_post_read_file, security_cred_getsecid, security_cred_getlsmprop, …) — out-of-tree modules legally referencing exported hooks. */

REQ-17: lockdown_reasons[] (LOCKDOWN_NONE .. LOCKDOWN_CONFIDENTIALITY_MAX):
- LOCKDOWN_NONE = "none".
- LOCKDOWN_MODULE_SIGNATURE = "unsigned module loading".
- LOCKDOWN_DEV_MEM = "/dev/mem,kmem,port".
- LOCKDOWN_EFI_TEST, _KEXEC, _HIBERNATION, _PCI_ACCESS, _IOPORT, _MSR, _ACPI_TABLES, _DEVICE_TREE, _PCMCIA_CIS, _TIOCSSERIAL, _MODULE_PARAMETERS, _MMIOTRACE, _DEBUGFS, _XMON_WR, _BPF_WRITE_USER, _DBG_WRITE_KERNEL, _RTAS_ERROR_INJECTION, _XEN_USER_ACTIONS, _INTEGRITY_MAX = "integrity".
- LOCKDOWN_KCORE, _KPROBES, _BPF_READ_KERNEL, _DBG_READ_KERNEL, _PERF, _TRACEFS, _XMON_RW, _XFRM_SECRET, _CONFIDENTIALITY_MAX = "confidentiality".

REQ-18: lsm_fill_user_ctx(uctx, uctx_len, val, val_len, id, flags):
- /* Compute required size for struct lsm_ctx + val_len, aligned to sizeof(void *) */
- nctx_len = ALIGN(struct_size(nctx, ctx, val_len), sizeof(void *)).
- if nctx_len > *uctx_len: rc = -E2BIG; goto out.
- if !uctx: goto out (success, *uctx_len holds required size).
- nctx = kzalloc(nctx_len, GFP_KERNEL); if !nctx: rc = -ENOMEM; goto out.
- nctx->id = id; ->flags = flags; ->len = nctx_len; ->ctx_len = val_len; memcpy(nctx->ctx, val, val_len).
- if copy_to_user(uctx, nctx, nctx_len): rc = -EFAULT.
- kfree(nctx); *uctx_len = nctx_len; return rc.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `static_call_slot_monotonic_fill` | INVARIANT | per-static_call_init: only fills `hl == None` slot; never overwrites. |
| `static_calls_table_ro_after_init` | INVARIANT | post-security_init: writes to static_calls_table fault. |
| `blob_offsets_within_total` | INVARIANT | per-prepare: every LSM's assigned lbs_* offset + its requested size ≤ blob_sizes.lbs_*. |
| `blob_offsets_align_ptr` | INVARIANT | per-prepare: offsets are aligned to sizeof(void *). |
| `inode_blob_carries_rcu_head` | INVARIANT | per-prepare: lbs_inode > 0 ⇒ first sizeof(rcu_head) bytes reserved. |
| `call_int_hook_short_circuits` | INVARIANT | per-call_int_hook: first slot returning ≠ LSM_RET_DEFAULT terminates the loop. |
| `disabled_slot_branch_elided` | INVARIANT | per-call: !static_key_enabled(active) ⇒ static_call is not invoked. |
| `add_hooks_panics_on_exhaust` | LIVENESS | per-add_hooks: MAX_LSM_COUNT exceeded ⇒ panic (not silent overflow). |
| `lsmid_set_post_add_hooks` | INVARIANT | per-add_hooks: ∀i. hooks[i].lsmid == lsmid. |
| `fill_user_ctx_no_overflow` | INVARIANT | per-fill_user_ctx: nctx_len ≤ *uctx_len ∨ -E2BIG. |

### Layer 2: TLA+

`security/lsm-core.tla`:
- Per-LSM-register + per-prepare + per-init_single + per-hook-dispatch + per-blob-alloc.
- Properties:
  - `safety_no_double_slot_assignment` — per-static_call_init: a slot is taken at most once.
  - `safety_blob_offsets_disjoint` — per-prepare: no two LSMs receive overlapping offsets in the same blob.
  - `safety_early_before_normal` — early_security_init completes before security_init begins.
  - `safety_capability_first` — LSM_ORDER_FIRST occupies slot 0 of every hook it registers.
  - `safety_integrity_last` — LSM_ORDER_LAST occupies the highest-index occupied slot.
  - `liveness_add_hooks_terminates` — per-call: returns success or panics within MAX_LSM_COUNT steps.
  - `liveness_call_int_hook_terminates` — bounded by MAX_LSM_COUNT.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lsm::add_hooks` post: ∀i. hooks[i].lsmid == lsmid ∧ slot patched | `Lsm::add_hooks` |
| `Lsm::static_call_init` post: hl assigned to a previously-free slot ∨ ENOSPC | `Lsm::static_call_init` |
| `Lsm::prepare` post: composite-blob layout monotone-nondecreasing | `Lsm::prepare` |
| `Lsm::blob_size_update` post: *sz_req = old(*sz_cur) aligned; *sz_cur = new offset + size | `Lsm::blob_size_update` |
| `Lsm::cred_alloc` post: cred.security ∈ {None, ptr to kzalloc'd lbs_cred bytes} | `Lsm::cred_alloc` |
| `Lsm::inode_alloc` post: inode.i_security ∈ {None, ptr from lsm_inode_cache} | `Lsm::inode_alloc` |
| `Lsm::file_alloc` post: file.f_security ∈ {None, ptr from lsm_file_cache} | `Lsm::file_alloc` |
| `Lsm::call_int_hook` post: rc = LSM_RET_DEFAULT ∨ rc = first non-default hook return | dispatch |
| `Lsm::call_void_hook` post: every enabled slot's hook invoked | dispatch |
| `Lsm::fill_user_ctx` post: rc ∈ {0, -E2BIG, -ENOMEM, -EFAULT} ∧ *uctx_len = required size | `Lsm::fill_user_ctx` |

### Layer 4: Verus/Creusot functional

`Per-bootstrap: early_security_init → security_init (order-parse → prepare → kmem-caches → init_single) → security_add_hooks(per-LSM, per-hook) → static_call patched` semantic equivalence with the upstream C path. `Per-callsite: kernel callsite invokes Lsm::<hook>(args) ≡ call_int_hook(<hook>, args) ≡ first non-default of the enabled LSMs' callbacks in registered order` — proven against `Documentation/security/lsm.rst` and the kernel-internal `include/linux/security.h` callsite list.

### hardening

(Inherits row-1 features from `security/00-overview.md` § Hardening.)

LSM-core reinforcement:

- **Per-static_calls_table __ro_after_init** — defense against per-runtime hook-table tampering (post-init write faults).
- **Per-MAX_LSM_COUNT exhaustion panics** — defense against per-silent-hook-loss; an over-quota LSM aborts boot rather than going un-enforced.
- **Per-hook static-branch-false default** — defense against per-stale-call-into-unloaded-LSM (cold-path branch not taken when slot empty).
- **Per-static_call_init writes only empty slots** — defense against per-hook-displacement (no overwrite of a registered LSM's slot).
- **Per-LSM_ORDER_FIRST = capability** — defense against per-MAC-bypass (capability check evaluated first).
- **Per-LSM_ORDER_LAST = integrity (IMA/EVM)** — defense against per-measurement-skew (integrity sees final cred/inode state).
- **Per-LSM_FLAG_EXCLUSIVE for major-LSM** — defense against per-blob-collision on cred/task fields managed by a single major LSM.
- **Per-blob composite alloc aligned to sizeof(void *)** — defense against per-misaligned-access on architectures with strict alignment (early-init path runs before fixup).
- **Per-inode-blob includes rcu_head** — defense against per-blob-freed-while-readers-walk (free deferred via RCU).
- **Per-file/-inode/-backing-file kmem_cache SLAB_PANIC** — defense against per-cache-create-fail at boot (panic visible rather than NULL-deref later).
- **Per-call_int_hook short-circuit deny-wins** — defense against per-permissive-stacking (any LSM denial terminates dispatch).
- **Per-cred_free / task_free / sb_free kfree-and-NULL** — defense against per-UAF on stale blob pointer.
- **Per-fill_user_ctx -E2BIG with size-out** — defense against per-truncated-blob-leak; userspace re-sizes and retries.
- **Per-lockdown_reasons[] indexed by enum** — defense against per-out-of-range reason audit-string leak.
- **Per-lsm_id->name maintained const** — defense against per-spoofed-LSM-name in audit records.

