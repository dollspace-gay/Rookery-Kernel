# Tier-3: io_uring/register.c — io_uring_register(2) opcode dispatch

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/register.c (~1038 lines)
  - io_uring/register.h
  - include/uapi/linux/io_uring.h (enum io_uring_register_op, IORING_REGISTER_*, IORING_REGISTER_USE_REGISTERED_RING)
-->

## Summary

`io_uring_register(2)` is the side-band syscall that mutates the per-ring (`struct io_ring_ctx`) configuration that the SQE fast path does not touch — registering long-lived resources, setting policy, querying capabilities, resizing rings, and performing synchronous (non-SQE) operations such as cross-ring message sends and synchronous cancels. The kernel-side dispatch lives in `io_uring/register.c` (~1038 lines). The signature is `int io_uring_register(unsigned fd, unsigned opcode, void *arg, unsigned nr_args)`. The high bit of `opcode` (`IORING_REGISTER_USE_REGISTERED_RING` = `1U << 31`) optionally selects a registered ring-fd index instead of the fd. A negative `fd == -1` triggers the **blind** dispatch (no ring context) used for `IORING_REGISTER_SEND_MSG_RING`, `IORING_REGISTER_QUERY`, `IORING_REGISTER_RESTRICTIONS` (task-scope), and `IORING_REGISTER_BPF_FILTER` (task-scope). All other opcodes resolve through `__io_uring_register()` under `ctx->uring_lock`, after a `submitter_task` check (single-issuer enforcement) and a `restrictions.register_op` allowance check when `IO_RING_F_REG_RESTRICTED` is in effect. The 38-way `switch` covers PROBE, EVENTFD/_ASYNC, RESTRICTIONS, IOWQ_AFF/IOWQ_MAX_WORKERS, NAPI, SYNC_CANCEL, ENABLE_RINGS, RESIZE_RINGS, CLOCK, SEND_MSG_RING, PBUF_RING, FILES/FILES2/FILES_UPDATE/FILES_UPDATE2, BUFFERS/BUFFERS2/BUFFERS_UPDATE, CLONE_BUFFERS, RING_FDS, PBUF_STATUS, FILE_ALLOC_RANGE, ZCRX_IFQ, ZCRX_CTRL, MEM_REGION, QUERY, BPF_FILTER, PERSONALITY. Critical for: io_uring lifecycle bring-up (REGISTER_ENABLE_RINGS after `IORING_SETUP_R_DISABLED`), security (RESTRICTIONS + BPF_FILTER, seccomp-like task restrictions), and online reconfiguration (RESIZE_RINGS, IOWQ_MAX_WORKERS).

This Tier-3 covers `io_uring/register.c` (~1038 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE4(io_uring_register, ...)` | per-syscall entry | `Register::sys_register` |
| `__io_uring_register()` | per-opcode dispatch (ring-bound) | `Register::dispatch` |
| `io_uring_register_blind()` | per-opcode dispatch (fd == -1) | `Register::dispatch_blind` |
| `io_uring_register_send_msg_ring()` | per-blind MSG_RING send | `Register::send_msg_ring_blind` |
| `io_uring_ctx_get_file()` | per-fd resolution (incl. registered-ring fd) | `Register::ctx_get_file` |
| `io_probe()` | per-IORING_REGISTER_PROBE | `Register::probe` |
| `io_register_personality()` / `io_unregister_personality()` | per-IORING_(UN)REGISTER_PERSONALITY | `Register::register_personality` / `_unregister` |
| `io_parse_restrictions()` | per-IORING_REGISTER_RESTRICTIONS parser | `Register::parse_restrictions` |
| `io_register_restrictions()` | per-ring restrictions reg | `Register::register_restrictions` |
| `io_register_restrictions_task()` | per-task (blind) restrictions reg | `Register::register_restrictions_task` |
| `io_register_bpf_filter_task()` | per-task (blind) BPF filter reg | `Register::register_bpf_filter_task` |
| `io_register_enable_rings()` | per-IORING_REGISTER_ENABLE_RINGS | `Register::enable_rings` |
| `io_register_iowq_aff()` / `__io_register_iowq_aff()` / `io_unregister_iowq_aff()` | per-IORING_(UN)REGISTER_IOWQ_AFF | `Register::iowq_aff_*` |
| `io_register_iowq_max_workers()` | per-IORING_REGISTER_IOWQ_MAX_WORKERS | `Register::iowq_max_workers` |
| `io_register_clock()` | per-IORING_REGISTER_CLOCK | `Register::clock` |
| `io_register_resize_rings()` | per-IORING_REGISTER_RESIZE_RINGS | `Register::resize_rings` |
| `io_register_free_rings()` | per-resize cleanup | `Register::free_rings_state` |
| `io_register_mem_region()` | per-IORING_REGISTER_MEM_REGION | `Register::mem_region` |
| `io_sqe_buffers_register()` / `_unregister()` | per-IORING_REGISTER_BUFFERS / UNREGISTER | `Rsrc::buffers_register` / `_unregister` |
| `io_sqe_files_register()` / `_unregister()` | per-IORING_REGISTER_FILES / UNREGISTER | `Rsrc::files_register` / `_unregister` |
| `io_register_files_update()` | per-IORING_REGISTER_FILES_UPDATE (legacy) | `Rsrc::files_update_legacy` |
| `io_register_rsrc()` / `io_register_rsrc_update()` | per-IORING_REGISTER_FILES2/BUFFERS2 + UPDATE2/BUFFERS_UPDATE (tagged) | `Rsrc::register` / `_update` |
| `io_register_clone_buffers()` | per-IORING_REGISTER_CLONE_BUFFERS | `Rsrc::clone_buffers` |
| `io_register_file_alloc_range()` | per-IORING_REGISTER_FILE_ALLOC_RANGE | `Rsrc::file_alloc_range` |
| `io_eventfd_register()` / `io_eventfd_unregister()` | per-IORING_(UN)REGISTER_EVENTFD[_ASYNC] | `Eventfd::register` / `_unregister` |
| `io_ringfd_register()` / `io_ringfd_unregister()` | per-IORING_(UN)REGISTER_RING_FDS | `Tctx::ringfd_register` / `_unregister` |
| `io_register_pbuf_ring()` / `io_unregister_pbuf_ring()` / `io_register_pbuf_status()` | per-IORING_REGISTER_PBUF_RING / UNREG / STATUS | `Kbuf::*` (see `kbuf.md`) |
| `io_sync_cancel()` | per-IORING_REGISTER_SYNC_CANCEL | `Cancel::sync_cancel` |
| `io_register_napi()` / `io_unregister_napi()` | per-IORING_(UN)REGISTER_NAPI | `Napi::register` / `_unregister` |
| `io_uring_sync_msg_ring()` | per-blind MSG_RING delivery | `MsgRing::sync_send` |
| `io_register_zcrx()` / `io_zcrx_ctrl()` | per-IORING_REGISTER_ZCRX_IFQ / ZCRX_CTRL | `Zcrx::register` / `_ctrl` |
| `io_query()` | per-IORING_REGISTER_QUERY | `Query::query` |
| `io_register_bpf_filter()` | per-IORING_REGISTER_BPF_FILTER (ring) | `BpfFilter::register` |
| `struct io_ring_ctx_rings` | per-resize transient (rings + sq_sqes + regions) | `RingsState` |
| `struct io_uring_clock_register` | per-CLOCK UAPI arg | `ClockReg` |
| `struct io_uring_mem_region_reg` | per-MEM_REGION UAPI arg | `MemRegionReg` |
| `struct io_uring_restriction` | per-RESTRICTIONS array item UAPI | `RestrictionItem` |
| `struct io_uring_task_restriction` | per-task RESTRICTIONS header UAPI | `TaskRestriction` |
| `IORING_REGISTER_LAST` | per-opcode count bound | constant |
| `IORING_REGISTER_USE_REGISTERED_RING` | per-opcode high-bit selector | flag |
| `IO_RING_F_REG_RESTRICTED` / `_OP_RESTRICTED` | per-ctx int_flags | flags |
| `IORING_MAX_RESTRICTIONS` | per-cap on `nr_args` | constant |
| `RESIZE_FLAGS` / `COPY_FLAGS` | per-resize flag policy | masks |

## Compatibility contract

REQ-1: SYSCALL_DEFINE4 sys_register(fd, opcode, arg, nr_args):
- use_registered_ring = `!!(opcode & IORING_REGISTER_USE_REGISTERED_RING)`.
- opcode &= ~IORING_REGISTER_USE_REGISTERED_RING.
- if opcode ≥ IORING_REGISTER_LAST → -EINVAL.
- if fd == -1 → tail-call io_uring_register_blind(opcode, arg, nr_args).
- file = `io_uring_ctx_get_file(fd, use_registered_ring)`; IS_ERR → return PTR_ERR.
- ctx = file->private_data.
- mutex_lock(&ctx->uring_lock).
- ret = __io_uring_register(ctx, opcode, arg, nr_args).
- trace_io_uring_register(ctx, opcode, files.nr, buf_table.nr, ret).
- mutex_unlock(&ctx->uring_lock).
- if !use_registered_ring → fput(file).
- return ret.

REQ-2: io_uring_register_blind(opcode, arg, nr_args):
- Allowed opcodes only:
  - IORING_REGISTER_SEND_MSG_RING → io_uring_register_send_msg_ring (parse SQE, call io_uring_sync_msg_ring).
  - IORING_REGISTER_QUERY → io_query(arg, nr_args).
  - IORING_REGISTER_RESTRICTIONS → io_register_restrictions_task(arg, nr_args).
  - IORING_REGISTER_BPF_FILTER → io_register_bpf_filter_task(arg, nr_args).
- Any other → -EINVAL.

REQ-3: __io_uring_register(ctx, opcode, arg, nr_args):
- WARN if `percpu_ref_is_dying(&ctx->refs)` → -ENXIO.
- if `ctx->submitter_task && ctx->submitter_task != current` → -EEXIST (single-issuer registered to another task).
- if `(ctx->int_flags & IO_RING_F_REG_RESTRICTED) && !(ctx->flags & IORING_SETUP_R_DISABLED)`:
  - opcode = `array_index_nospec(opcode, IORING_REGISTER_LAST)`.
  - if `!test_bit(opcode, ctx->restrictions.register_op)` → -EACCES.
- 38-way switch — see REQ-4..REQ-25.

REQ-4: IORING_REGISTER_BUFFERS / _UNREGISTER_BUFFERS (opcode 0, 1):
- BUFFERS: !arg → -EFAULT; else io_sqe_buffers_register(ctx, arg, nr_args, NULL).
- UNREGISTER_BUFFERS: arg ∨ nr_args → -EINVAL; else io_sqe_buffers_unregister(ctx).

REQ-5: IORING_REGISTER_FILES / _UNREGISTER_FILES (opcode 2, 3):
- FILES: !arg → -EFAULT; else io_sqe_files_register(ctx, arg, nr_args, NULL).
- UNREGISTER_FILES: arg ∨ nr_args → -EINVAL; else io_sqe_files_unregister(ctx).

REQ-6: IORING_REGISTER_EVENTFD / _ASYNC / _UNREGISTER_EVENTFD (opcode 4, 7, 5):
- EVENTFD: nr_args != 1 → -EINVAL; else io_eventfd_register(ctx, arg, /*async=*/0).
- EVENTFD_ASYNC: nr_args != 1 → -EINVAL; else io_eventfd_register(ctx, arg, 1).
- UNREGISTER_EVENTFD: arg ∨ nr_args → -EINVAL; else io_eventfd_unregister(ctx).

REQ-7: IORING_REGISTER_FILES_UPDATE (opcode 6):
- io_register_files_update(ctx, arg, nr_args). (Legacy untagged.)

REQ-8: IORING_REGISTER_PROBE (opcode 8):
- !arg ∨ nr_args > 256 → -EINVAL.
- io_probe(ctx, arg, nr_args):
  - clamp nr_args to IORING_OP_LAST.
  - size = struct_size(p, ops, nr_args).
  - p = memdup_user(arg, size).
  - reject any non-zero input via memchr_inv(p, 0, size) → -EINVAL (must zero-fill).
  - p->last_op = IORING_OP_LAST - 1.
  - for i in 0..nr_args: p->ops[i].op = i; if `io_uring_op_supported(i)` → flag |= IO_URING_OP_SUPPORTED.
  - p->ops_len = i.
  - copy_to_user(arg, p, size).

REQ-9: IORING_REGISTER_PERSONALITY / UN (opcode 9, 10):
- PERSONALITY: arg ∨ nr_args → -EINVAL; else io_register_personality(ctx):
  - get_current_cred(); xa_alloc_cyclic into ctx->personalities with limit USHRT_MAX → returns id ≥ 0.
- UNREGISTER_PERSONALITY: arg → -EINVAL; else io_unregister_personality(ctx, /*id=*/nr_args):
  - xa_erase; put_cred.

REQ-10: IORING_REGISTER_ENABLE_RINGS (opcode 12):
- arg ∨ nr_args → -EINVAL.
- io_register_enable_rings(ctx):
  - if !(ctx->flags & IORING_SETUP_R_DISABLED) → -EBADFD.
  - if `IORING_SETUP_SINGLE_ISSUER`:
    - ctx->submitter_task = `get_task_struct(current)`.
    - if poll_wq has sleepers → io_activate_pollwq(ctx).
  - `smp_store_release(&ctx->flags, ctx->flags & ~IORING_SETUP_R_DISABLED)`.
  - if sq_data has sleepers → wake_up(&sq_data->wait).

REQ-11: IORING_REGISTER_RESTRICTIONS (opcode 11):
- io_register_restrictions(ctx, arg, nr_args):
  - if !(ctx->flags & IORING_SETUP_R_DISABLED) → -EBADFD.
  - if any restrictions already registered → -EBUSY.
  - parse_restrictions(arg, nr_args, &ctx->restrictions):
    - reject if !arg ∨ nr_args > IORING_MAX_RESTRICTIONS.
    - memdup_user.
    - per item:
      - IORING_RESTRICTION_REGISTER_OP: register_op < IORING_REGISTER_LAST → set_bit; reg_registered = true.
      - IORING_RESTRICTION_SQE_OP: sqe_op < IORING_OP_LAST → set_bit; op_registered = true.
      - IORING_RESTRICTION_SQE_FLAGS_ALLOWED: sqe_flags_allowed = sqe_flags.
      - IORING_RESTRICTION_SQE_FLAGS_REQUIRED: sqe_flags_required = sqe_flags.
      - default → -EINVAL.
    - nr_args == 0 → still mark op_registered + reg_registered (lock-down).
  - On parse error: zero out restrictions but retain `bpf_filters` and `bpf_filters_cow`.
  - if op_registered → ctx->int_flags |= IO_RING_F_OP_RESTRICTED.
  - if reg_registered → ctx->int_flags |= IO_RING_F_REG_RESTRICTED.

REQ-12: IORING_REGISTER_FILES2 / FILES_UPDATE2 / BUFFERS2 / BUFFERS_UPDATE (opcode 13..16):
- Tagged variants:
  - FILES2: io_register_rsrc(ctx, arg, nr_args, IORING_RSRC_FILE).
  - FILES_UPDATE2: io_register_rsrc_update(ctx, arg, nr_args, IORING_RSRC_FILE).
  - BUFFERS2: io_register_rsrc(ctx, arg, nr_args, IORING_RSRC_BUFFER).
  - BUFFERS_UPDATE: io_register_rsrc_update(ctx, arg, nr_args, IORING_RSRC_BUFFER).

REQ-13: IORING_REGISTER_IOWQ_AFF / _UN (opcode 17, 18):
- IOWQ_AFF: !arg ∨ !nr_args → -EINVAL; else io_register_iowq_aff(ctx, arg, nr_args):
  - alloc_cpumask_var(new_mask).
  - clamp len to cpumask_size().
  - if compat_syscall → compat_get_bitmap from user; else copy_from_user.
  - On copy fail → -EFAULT.
  - __io_register_iowq_aff(ctx, new_mask):
    - if !(SQPOLL) → io_wq_cpu_affinity(current->io_uring, new_mask).
    - else: drop uring_lock; io_sqpoll_wq_cpu_affinity(ctx, new_mask); re-acquire.
  - free_cpumask_var.
- UNREGISTER_IOWQ_AFF: arg ∨ nr_args → -EINVAL; else __io_register_iowq_aff(ctx, NULL).

REQ-14: IORING_REGISTER_IOWQ_MAX_WORKERS (opcode 19):
- !arg ∨ nr_args != 2 → -EINVAL.
- io_register_iowq_max_workers(ctx, arg) (__must_hold(uring_lock)):
  - copy_from_user(new_count[2]).
  - any new_count[i] > INT_MAX → -EINVAL.
  - if SQPOLL ∧ ctx->sq_data:
    - refcount_inc(sqd->refs); drop uring_lock; mutex_lock(sqd->lock); re-acquire uring_lock (sqd->lock → uring_lock order).
    - tsk = sqpoll_task_locked(sqd); if tsk → tctx = tsk->io_uring.
  - else: tctx = current->io_uring.
  - BUILD_BUG_ON sizeof check vs ctx->iowq_limits.
  - For each i: if new_count[i] != 0 → ctx->iowq_limits[i] = new_count[i].
  - ctx->int_flags |= IO_RING_F_IOWQ_LIMITS_SET.
  - if tctx ∧ tctx->io_wq → io_wq_max_workers(tctx->io_wq, new_count) → err goto.
  - else: zero new_count.
  - if SQPOLL → release sqd lock + uring_lock, put_sq_data, re-acquire uring_lock.
  - copy_to_user(arg, new_count) (returns OLD values).
  - if sqd → return 0 (only SQPOLL task creates requests).
  - else: walk ctx->tctx_list under tctx_lock; for each node: copy ctx->iowq_limits into new_count and io_wq_max_workers(node->task->io_uring->io_wq, new_count).

REQ-15: IORING_REGISTER_RING_FDS / _UN (opcode 20, 21):
- io_ringfd_register / io_ringfd_unregister(ctx, arg, nr_args).

REQ-16: IORING_REGISTER_PBUF_RING / _UN / _PBUF_STATUS (opcode 22, 23, 26):
- PBUF_RING: !arg ∨ nr_args != 1 → -EINVAL; else io_register_pbuf_ring(ctx, arg).
- UNREGISTER_PBUF_RING: !arg ∨ nr_args != 1 → -EINVAL; else io_unregister_pbuf_ring(ctx, arg).
- PBUF_STATUS: !arg ∨ nr_args != 1 → -EINVAL; else io_register_pbuf_status(ctx, arg).

REQ-17: IORING_REGISTER_SYNC_CANCEL (opcode 24):
- !arg ∨ nr_args != 1 → -EINVAL; else io_sync_cancel(ctx, arg).

REQ-18: IORING_REGISTER_FILE_ALLOC_RANGE (opcode 25):
- !arg ∨ nr_args → -EINVAL; else io_register_file_alloc_range(ctx, arg).

REQ-19: IORING_REGISTER_NAPI / _UN (opcode 27, 28):
- NAPI: !arg ∨ nr_args != 1 → -EINVAL; else io_register_napi(ctx, arg).
- UNREGISTER_NAPI: nr_args != 1 → -EINVAL; else io_unregister_napi(ctx, arg). (Note: arg may be NULL for default unregister.)

REQ-20: IORING_REGISTER_CLOCK (opcode 29):
- !arg ∨ nr_args → -EINVAL; else io_register_clock(ctx, arg):
  - copy_from_user(reg).
  - memchr_inv(reg.__resv, 0, …) → -EINVAL.
  - switch reg.clockid:
    - CLOCK_MONOTONIC → ctx->clock_offset = 0.
    - CLOCK_BOOTTIME → ctx->clock_offset = TK_OFFS_BOOT.
    - default → -EINVAL.
  - ctx->clockid = reg.clockid.

REQ-21: IORING_REGISTER_CLONE_BUFFERS (opcode 30):
- !arg ∨ nr_args != 1 → -EINVAL; else io_register_clone_buffers(ctx, arg).

REQ-22: IORING_REGISTER_ZCRX_IFQ / _CTRL (opcode 32, 36):
- ZCRX_IFQ: !arg ∨ nr_args != 1 → -EINVAL; else io_register_zcrx(ctx, arg).
- ZCRX_CTRL: io_zcrx_ctrl(ctx, arg, nr_args). (No fast pre-checks; helper validates.)

REQ-23: IORING_REGISTER_RESIZE_RINGS (opcode 33):
- !arg ∨ nr_args != 1 → -EINVAL.
- io_register_resize_rings(ctx, arg):
  - Require IORING_SETUP_DEFER_TASKRUN → else -EINVAL.
  - copy_from_user(p).
  - p->flags must subset RESIZE_FLAGS = `IORING_SETUP_CQSIZE | IORING_SETUP_CLAMP`.
  - p->flags |= (ctx->flags & COPY_FLAGS) where `COPY_FLAGS = IORING_SETUP_NO_SQARRAY | _SQE128 | _CQE32 | _NO_MMAP | _CQE_MIXED | _SQE_MIXED`.
  - io_prepare_config(&config) — recompute sizes.
  - Allocate n.ring_region (rd.size = PAGE_ALIGN(rl->rings_size); USER if NO_MMAP). copy params back to user.
  - Allocate n.sq_region (rd.size = PAGE_ALIGN(rl->sq_size); USER if NO_MMAP).
  - If SQPOLL → drop uring_lock; park sq_data; re-acquire.
  - mutex_lock(&ctx->mmap_lock); spin_lock(&ctx->completion_lock).
  - o.rings = ctx->rings; ctx->rings = NULL; o.sq_sqes = ctx->sq_sqes; ctx->sq_sqes = NULL.
  - Copy SQ: if (tail - old_head) > p->sq_entries → -EOVERFLOW + restore old; else memcpy each entry through src_mask/dst_mask (handles SQE128).
  - Copy CQ: if (tail - old_head) > p->cq_entries → -EOVERFLOW + restore old; else memcpy each cqe (handles CQE32). Invalidate cqe_cached / cqe_sentinel.
  - WRITE_ONCE various o→n fields (sq_dropped, sq_flags, cq_flags, cq_overflow).
  - If !NO_SQARRAY → ctx->sq_array = (u32*)((char*)n.rings + rl->sq_array_offset).
  - ctx->sq_entries / cq_entries = p->*.
  - atomic_or IORING_SQ_TASKRUN | IORING_SQ_NEED_WAKEUP into n.rings->sq_flags.
  - ctx->rings = n.rings; rcu_assign_pointer(ctx->rings_rcu, n.rings).
  - ctx->sq_sqes = n.sq_sqes; swap_old fields ring_region + sq_region into o.
  - Unlock completion_lock + mmap_lock; synchronize_rcu_expedited (concurrent io_ctx_mark_taskrun() readers); io_register_free_rings(ctx, &o).
  - If SQPOLL → io_sq_thread_unpark.

REQ-24: IORING_REGISTER_MEM_REGION (opcode 34):
- !arg ∨ nr_args != 1 → -EINVAL.
- io_register_mem_region(ctx, arg):
  - If io_region_is_set(&ctx->param_region) → -EBUSY.
  - copy_from_user(reg) (io_uring_mem_region_reg) and then copy_from_user(rd) from reg.region_uptr.
  - memchr_inv(&reg.__resv, 0, …) → -EINVAL.
  - reg.flags ⊆ {IORING_MEM_REGION_REG_WAIT_ARG} → else -EINVAL.
  - if WAIT_ARG ∧ !R_DISABLED → -EINVAL.
  - io_create_region(ctx, &region, &rd, IORING_MAP_OFF_PARAM_REGION).
  - copy_to_user(rd_uptr, &rd).
  - if WAIT_ARG → ctx->cq_wait_arg = io_region_get_ptr(&region); ctx->cq_wait_size = rd.size.
  - io_region_publish(ctx, &region, &ctx->param_region).

REQ-25: IORING_REGISTER_QUERY / _BPF_FILTER (opcode 35, 37):
- QUERY: io_query(arg, nr_args).
- BPF_FILTER: nr_args != 1 → -EINVAL; else io_register_bpf_filter(&ctx->restrictions, arg); on success WRITE_ONCE(ctx->bpf_filters, ctx->restrictions.bpf_filters->filters).

REQ-26: Blind send_msg_ring:
- io_uring_register_send_msg_ring(arg, nr_args):
  - !arg ∨ nr_args != 1 → -EINVAL.
  - copy_from_user(sqe).
  - sqe.flags != 0 → -EINVAL.
  - sqe.opcode != IORING_OP_MSG_RING → -EINVAL.
  - io_uring_sync_msg_ring(&sqe).

REQ-27: Blind register_restrictions_task / register_bpf_filter_task:
- task_no_new_privs OR CAP_SYS_ADMIN in current_user_ns required (seccomp-like).
- restrictions_task: if current->io_uring_restrict already set → -EPERM; nr_args != 1 → -EINVAL; copy in `io_uring_task_restriction`; flags/resv must be zero; allocate `io_restriction`; io_parse_restrictions; on success → current->io_uring_restrict = res.
- bpf_filter_task: nr_args != 1 → -EINVAL; reuse or allocate `io_restriction`; io_register_bpf_filter(res, arg); on success → current->io_uring_restrict = res.

REQ-28: io_register_personality():
- xa_alloc_cyclic(&ctx->personalities, &id, current_cred, XA_LIMIT(0, USHRT_MAX), &ctx->pers_next, GFP_KERNEL).
- ret < 0 → put_cred + propagate ret.
- else return id (≥ 0).

REQ-29: io_unregister_personality(ctx, id):
- creds = xa_erase(&ctx->personalities, id); if NULL → -EINVAL; else put_cred + 0.

REQ-30: RESIZE_RINGS constants:
- `RESIZE_FLAGS = IORING_SETUP_CQSIZE | IORING_SETUP_CLAMP`.
- `COPY_FLAGS = IORING_SETUP_NO_SQARRAY | IORING_SETUP_SQE128 | IORING_SETUP_CQE32 | IORING_SETUP_NO_MMAP | IORING_SETUP_CQE_MIXED | IORING_SETUP_SQE_MIXED`.

REQ-31: Locking discipline:
- ring-bound dispatch: uring_lock held across __io_uring_register.
- iowq_max_workers temporarily drops uring_lock if SQPOLL (to take sqd->lock first, then uring_lock).
- iowq_aff (SQPOLL path) likewise drops uring_lock around io_sqpoll_wq_cpu_affinity.
- resize_rings:
  - drops uring_lock around SQ-thread park (if SQPOLL).
  - holds mmap_lock + completion_lock over the actual `ctx->rings` pointer swap.
  - synchronize_rcu_expedited before freeing old rings.
- mem_region: uring_lock held; param_region published via `io_region_publish` (RCU-safe).

REQ-32: Restriction enforcement:
- IO_RING_F_REG_RESTRICTED + !R_DISABLED: each opcode checked against `ctx->restrictions.register_op` bitmap via array_index_nospec — Spectre v1 mitigation in case opcode came from attacker-controlled register.
- IO_RING_F_OP_RESTRICTED: enforced in SQE submission path (out of scope here).
- Per-task `current->io_uring_restrict`: enforced by callers of register-task variants in `io_uring/io_uring.c` and SQE prep — this file only installs it.

REQ-33: Trace point:
- `trace_io_uring_register(ctx, opcode, file_table.data.nr, buf_table.nr, ret)` emitted after dispatch under uring_lock.

REQ-34: USE_REGISTERED_RING:
- High-bit flag selects looking up `fd` as an index into the per-task registered-ring table instead of the FD table; resolved by `io_uring_ctx_get_file(fd, true)`.
- fput is skipped on return when this flag was used (the registered ring keeps its own ref).

## Acceptance Criteria

- [ ] AC-1: `opcode >= IORING_REGISTER_LAST` (after stripping high bit) → -EINVAL.
- [ ] AC-2: `fd == -1` with disallowed opcode → -EINVAL via blind dispatcher.
- [ ] AC-3: `fd == -1` with IORING_REGISTER_SEND_MSG_RING + valid OP_MSG_RING SQE → success.
- [ ] AC-4: Ring with IO_RING_F_REG_RESTRICTED active + bit clear in register_op bitmap → -EACCES.
- [ ] AC-5: Calling thread differs from `ctx->submitter_task` → -EEXIST.
- [ ] AC-6: IORING_REGISTER_RESTRICTIONS without IORING_SETUP_R_DISABLED → -EBADFD.
- [ ] AC-7: Double registration of restrictions → -EBUSY.
- [ ] AC-8: IORING_REGISTER_ENABLE_RINGS on already-enabled ring → -EBADFD.
- [ ] AC-9: IORING_REGISTER_PROBE with non-zero input buffer → -EINVAL.
- [ ] AC-10: IORING_REGISTER_CLOCK with `clockid` not in {MONOTONIC, BOOTTIME} → -EINVAL.
- [ ] AC-11: IORING_REGISTER_IOWQ_MAX_WORKERS with `new_count[i] > INT_MAX` → -EINVAL.
- [ ] AC-12: IORING_REGISTER_IOWQ_MAX_WORKERS returns OLD limits in arg.
- [ ] AC-13: IORING_REGISTER_RESIZE_RINGS without IORING_SETUP_DEFER_TASKRUN → -EINVAL.
- [ ] AC-14: RESIZE_RINGS shrink that would lose unconsumed SQ/CQ entries → -EOVERFLOW + state intact.
- [ ] AC-15: IORING_REGISTER_MEM_REGION twice → -EBUSY.
- [ ] AC-16: IORING_REGISTER_MEM_REGION with WAIT_ARG on enabled ring → -EINVAL.
- [ ] AC-17: IORING_REGISTER_PERSONALITY returns id ≥ 0 ≤ USHRT_MAX; subsequent UNREGISTER releases cred.
- [ ] AC-18: Blind RESTRICTIONS path: caller without task_no_new_privs and without CAP_SYS_ADMIN → -EACCES.
- [ ] AC-19: Blind RESTRICTIONS path: second registration on same task → -EPERM.
- [ ] AC-20: USE_REGISTERED_RING high bit selects index lookup; ordinary fput skipped on return.

## Architecture

```
struct RingsState {
  rings: *mut IoRings,
  sq_sqes: *mut IoUringSqe,
  sq_region: MappedRegion,
  ring_region: MappedRegion,
}
```

`Register::sys_register(fd, opcode_raw, arg, nr_args) -> isize`:
1. use_registered = (opcode_raw & IORING_REGISTER_USE_REGISTERED_RING) != 0.
2. opcode = opcode_raw & !IORING_REGISTER_USE_REGISTERED_RING.
3. if opcode ≥ IORING_REGISTER_LAST → return -EINVAL.
4. if fd == -1 → return Register::dispatch_blind(opcode, arg, nr_args).
5. file = Register::ctx_get_file(fd, use_registered)?.
6. ctx = file.private_data.
7. mutex_lock(&ctx.uring_lock).
8. ret = Register::dispatch(ctx, opcode, arg, nr_args).
9. trace_register(ctx, opcode, ctx.file_table.data.nr, ctx.buf_table.nr, ret).
10. mutex_unlock(&ctx.uring_lock).
11. if !use_registered → fput(file).
12. return ret.

`Register::dispatch_blind(opcode, arg, nr_args) -> i32`:
- match opcode:
  - IORING_REGISTER_SEND_MSG_RING → Register::send_msg_ring_blind(arg, nr_args).
  - IORING_REGISTER_QUERY → Query::query(arg, nr_args).
  - IORING_REGISTER_RESTRICTIONS → Register::register_restrictions_task(arg, nr_args).
  - IORING_REGISTER_BPF_FILTER → Register::register_bpf_filter_task(arg, nr_args).
  - _ → -EINVAL.

`Register::dispatch(ctx, opcode, arg, nr_args) -> i32`:
1. WARN if percpu_ref_is_dying(ctx.refs) → -ENXIO.
2. if ctx.submitter_task.is_some() ∧ ctx.submitter_task != current → -EEXIST.
3. if (ctx.int_flags & IO_RING_F_REG_RESTRICTED) ∧ !(ctx.flags & IORING_SETUP_R_DISABLED):
   - opcode = array_index_nospec(opcode, IORING_REGISTER_LAST).
   - if !test_bit(opcode, ctx.restrictions.register_op) → -EACCES.
4. match opcode:
   - 0  BUFFERS         → Rsrc::buffers_register(ctx, arg, nr_args, None).
   - 1  UNREGISTER_BUFFERS → Rsrc::buffers_unregister(ctx).
   - 2  FILES           → Rsrc::files_register(ctx, arg, nr_args, None).
   - 3  UNREGISTER_FILES → Rsrc::files_unregister(ctx).
   - 4  EVENTFD         → Eventfd::register(ctx, arg, async=false).
   - 5  UNREGISTER_EVENTFD → Eventfd::unregister(ctx).
   - 6  FILES_UPDATE    → Rsrc::files_update_legacy(ctx, arg, nr_args).
   - 7  EVENTFD_ASYNC   → Eventfd::register(ctx, arg, async=true).
   - 8  PROBE           → Register::probe(ctx, arg, nr_args).
   - 9  PERSONALITY     → Register::register_personality(ctx).
   - 10 UNREGISTER_PERSONALITY → Register::unregister_personality(ctx, nr_args).
   - 11 RESTRICTIONS    → Register::register_restrictions(ctx, arg, nr_args).
   - 12 ENABLE_RINGS    → Register::enable_rings(ctx).
   - 13 FILES2          → Rsrc::register(ctx, arg, nr_args, FILE).
   - 14 FILES_UPDATE2   → Rsrc::register_update(ctx, arg, nr_args, FILE).
   - 15 BUFFERS2        → Rsrc::register(ctx, arg, nr_args, BUFFER).
   - 16 BUFFERS_UPDATE  → Rsrc::register_update(ctx, arg, nr_args, BUFFER).
   - 17 IOWQ_AFF        → Register::iowq_aff(ctx, arg, nr_args).
   - 18 UNREGISTER_IOWQ_AFF → Register::iowq_aff_unregister(ctx).
   - 19 IOWQ_MAX_WORKERS → Register::iowq_max_workers(ctx, arg).
   - 20 RING_FDS        → Tctx::ringfd_register(ctx, arg, nr_args).
   - 21 UNREGISTER_RING_FDS → Tctx::ringfd_unregister(ctx, arg, nr_args).
   - 22 PBUF_RING       → Kbuf::register_ring(ctx, arg).
   - 23 UNREGISTER_PBUF_RING → Kbuf::unregister_ring(ctx, arg).
   - 24 SYNC_CANCEL     → Cancel::sync_cancel(ctx, arg).
   - 25 FILE_ALLOC_RANGE → Rsrc::file_alloc_range(ctx, arg).
   - 26 PBUF_STATUS     → Kbuf::register_status(ctx, arg).
   - 27 NAPI            → Napi::register(ctx, arg).
   - 28 UNREGISTER_NAPI → Napi::unregister(ctx, arg).
   - 29 CLOCK           → Register::clock(ctx, arg).
   - 30 CLONE_BUFFERS   → Rsrc::clone_buffers(ctx, arg).
   - 31 SEND_MSG_RING   → ABSENT here — see dispatch_blind.
   - 32 ZCRX_IFQ        → Zcrx::register(ctx, arg).
   - 33 RESIZE_RINGS    → Register::resize_rings(ctx, arg).
   - 34 MEM_REGION      → Register::mem_region(ctx, arg).
   - 35 QUERY           → Query::query(arg, nr_args).
   - 36 ZCRX_CTRL       → Zcrx::ctrl(ctx, arg, nr_args).
   - 37 BPF_FILTER      → BpfFilter::register(&ctx.restrictions, arg) + WRITE_ONCE ctx.bpf_filters.
   - _                  → -EINVAL.

`Register::enable_rings(ctx) -> i32`:
1. if !(ctx.flags & IORING_SETUP_R_DISABLED) → -EBADFD.
2. if ctx.flags & IORING_SETUP_SINGLE_ISSUER:
   - ctx.submitter_task = get_task_struct(current).
   - if wq_has_sleeper(&ctx.poll_wq) → io_activate_pollwq(ctx).
3. smp_store_release(&ctx.flags, ctx.flags & !IORING_SETUP_R_DISABLED).
4. if ctx.sq_data ∧ wq_has_sleeper(&ctx.sq_data.wait) → wake_up(&ctx.sq_data.wait).
5. return 0.

`Register::resize_rings(ctx, arg) -> i32`:
1. if !(ctx.flags & IORING_SETUP_DEFER_TASKRUN) → -EINVAL.
2. config.p = copy_from_user(arg)?.
3. if config.p.flags & !RESIZE_FLAGS → -EINVAL.
4. config.p.flags |= ctx.flags & COPY_FLAGS.
5. io_prepare_config(&config)?.
6. Allocate n.ring_region (rl.rings_size) ; if NO_MMAP set user_addr from p.cq_off.user_addr.
7. n.rings = io_region_get_ptr(&n.ring_region).
8. WRITE_ONCE n.rings.{sq_ring_mask, cq_ring_mask, sq_ring_entries, cq_ring_entries}.
9. copy_to_user(arg, &config.p).
10. Allocate n.sq_region (rl.sq_size).
11. n.sq_sqes = io_region_get_ptr(&n.sq_region).
12. if ctx.sq_data → drop uring_lock; io_sq_thread_park(ctx.sq_data); reacquire.
13. mutex_lock(&ctx.mmap_lock); spin_lock(&ctx.completion_lock).
14. Snapshot o ← ctx.{rings, sq_sqes}; clear ctx pointers.
15. Copy SQ entries (handle SQE128 double-width); overflow → restore, set ret = -EOVERFLOW, goto out.
16. Copy CQ entries (handle CQE32 double-width); overflow → restore, set ret = -EOVERFLOW, goto out.
17. WRITE_ONCE n.rings.{sq_dropped, cq_overflow, cq_flags}; atomic copy sq_flags.
18. If !NO_SQARRAY → ctx.sq_array = (n.rings + rl.sq_array_offset).
19. ctx.{sq_entries, cq_entries} = p.{sq_entries, cq_entries}.
20. atomic_or IORING_SQ_TASKRUN | IORING_SQ_NEED_WAKEUP into n.rings.sq_flags.
21. ctx.rings = n.rings; rcu_assign_pointer(ctx.rings_rcu, n.rings).
22. ctx.sq_sqes = n.sq_sqes.
23. swap_old(ctx, o, n, ring_region); swap_old(ctx, o, n, sq_region).
24. to_free = &o; ret = 0.
25. out: spin_unlock(completion_lock); mutex_unlock(mmap_lock).
26. if to_free == &o → synchronize_rcu_expedited().
27. io_register_free_rings(ctx, to_free).
28. if ctx.sq_data → io_sq_thread_unpark.
29. return ret.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `opcode_bounds_checked` | INVARIANT | per-sys_register: opcode-after-strip < IORING_REGISTER_LAST before dispatch. |
| `submitter_task_singleton` | INVARIANT | per-dispatch: ctx.submitter_task ∈ {None, current} else -EEXIST. |
| `restricted_op_index_nospec` | INVARIANT | per-restricted: opcode passed through array_index_nospec. |
| `enable_rings_one_shot` | INVARIANT | per-enable_rings: requires IORING_SETUP_R_DISABLED then clears it. |
| `restrictions_once` | INVARIANT | per-register_restrictions: returns -EBUSY if op_registered ∨ reg_registered. |
| `personality_id_in_ushrt` | INVARIANT | per-register_personality: returned id ≤ USHRT_MAX. |
| `iowq_max_workers_int_max` | INVARIANT | per-iowq_max_workers: rejects new_count > INT_MAX. |
| `resize_rings_overflow_restores` | INVARIANT | per-resize_rings: on overflow ctx.rings/sq_sqes restored to o.*. |
| `mem_region_busy_idempotent` | INVARIANT | per-mem_region: if param_region already set → -EBUSY. |
| `blind_priv_check` | INVARIANT | per-register_restrictions_task / bpf_filter_task: requires task_no_new_privs ∨ CAP_SYS_ADMIN. |

### Layer 2: TLA+

`io_uring/register.tla`:
- Variables: ctx.flags, ctx.int_flags, ctx.submitter_task, ctx.restrictions (op_registered, reg_registered, register_op bitmap), ctx.rings, ctx.sq_sqes, current.io_uring_restrict.
- Actions: SysRegister(fd, opcode, arg, nr_args), EnableRings, RegisterRestrictions, ResizeRings, RegisterPersonality, UnregisterPersonality, RegisterMemRegion, BlindRegisterTaskRestrictions.
- Properties:
  - `safety_no_dispatch_when_dying` — per-dispatch: percpu_ref_is_dying ⟹ -ENXIO + ctx unchanged.
  - `safety_restrictions_immutable_after_enable` — per-RESTRICTIONS: blocked once R_DISABLED cleared.
  - `safety_resize_rings_atomic` — per-resize: from external observer ctx.rings transitions atomically (under completion_lock + mmap_lock).
  - `safety_resize_rings_no_loss` — per-resize: SQ/CQ entries old_head..tail preserved in new rings.
  - `safety_personality_cred_balanced` — per-register/unregister_personality: get_current_cred / put_cred ref-counted.
  - `safety_blind_only_allowed_opcodes` — per-blind: only SEND_MSG_RING, QUERY, RESTRICTIONS, BPF_FILTER.
  - `safety_task_restriction_one_shot` — per-blind RESTRICTIONS: rejected if current.io_uring_restrict already set.
  - `liveness_per_register_returns` — per-call: dispatch eventually returns (no infinite hold of uring_lock).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Register::dispatch` post: ctx.uring_lock held on entry and exit (annotation) | `Register::dispatch` |
| `Register::probe` post: ops[i].op == i ∀ i ∧ ops_len == nr_args | `Register::probe` |
| `Register::register_personality` post: id ∈ [0, USHRT_MAX] ∨ < 0 | `Register::register_personality` |
| `Register::enable_rings` post: ctx.flags & IORING_SETUP_R_DISABLED == 0 ∨ -EBADFD | `Register::enable_rings` |
| `Register::register_restrictions` post: ctx.int_flags reflects op_registered / reg_registered bits | `Register::register_restrictions` |
| `Register::iowq_max_workers` post: ctx.iowq_limits[i] == new_count[i] for non-zero entries | `Register::iowq_max_workers` |
| `Register::resize_rings` post: ctx.rings == n.rings ∧ ctx.sq_sqes == n.sq_sqes (success); else ctx.* unchanged | `Register::resize_rings` |
| `Register::clock` post: ctx.clockid ∈ {MONOTONIC, BOOTTIME} | `Register::clock` |
| `Register::mem_region` post: param_region set; cq_wait_arg/size set iff WAIT_ARG | `Register::mem_region` |

### Layer 4: Verus/Creusot functional

`Per-syscall(io_uring_register, fd, opcode, arg, nr_args) → switch-dispatch` is observationally equivalent to upstream `io_uring/register.c` per `Documentation/io_uring/io_uring.rst` and `man io_uring_register(2)`. Resize ring trace `Old(SQE=64, CQE=128, head..tail entries) → REGISTER_RESIZE_RINGS(new=128/256) → New ring contains all in-flight entries in [head, tail)` equates pre/post atomically per the completion_lock+mmap_lock invariant.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

io_uring_register reinforcement:

- **Per-submitter_task single-issuer check** — defense against per-cross-task hijack of registered resources.
- **Per-percpu_ref_is_dying early return** — defense against per-register-after-tear-down UAF.
- **Per-array_index_nospec on restricted opcode** — defense against per-Spectre-v1 leak of restriction bitmap.
- **Per-IORING_REGISTER_USE_REGISTERED_RING fput skip** — defense against per-double-fput on registered-ring path.
- **Per-IORING_SETUP_R_DISABLED gated RESTRICTIONS + ENABLE_RINGS** — defense against per-late-policy install.
- **Per-task_no_new_privs / CAP_SYS_ADMIN on blind RESTRICTIONS + BPF_FILTER** — defense against per-unprivileged-policy escalation.
- **Per-resize_rings DEFER_TASKRUN only** — defense against per-SQPOLL-or-IOPOLL concurrent-mutation races.
- **Per-resize_rings completion_lock + mmap_lock + synchronize_rcu_expedited** — defense against per-mid-resize CQE drop / mmap tear / task_run races.
- **Per-resize_rings entry-loss detection (-EOVERFLOW restores old)** — defense against per-shrink data-loss.
- **Per-memchr_inv on RESTRICTIONS / PROBE / CLOCK / MEM_REGION reserved fields** — defense against per-future-ABI flag collision and uninitialized-memory leak through clarifying input.
- **Per-MEM_REGION param_region one-shot** — defense against per-double-publish of cq_wait_arg.
- **Per-IOWQ_MAX_WORKERS INT_MAX cap + uring_lock⇄sqd_lock ordering** — defense against per-lock-inversion deadlock with SQPOLL.
- **Per-PERSONALITY cred get/put balance via xa_erase** — defense against per-stale-creds use.
- **Per-blind-dispatch allowlist** — defense against per-leakage of ring-scoped opcodes into fd==-1 path.
- **Per-WRITE_ONCE on ctx.bpf_filters after BPF_FILTER install** — defense against per-tearing-load by submission path.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `io_uring_register` copies opcode-specific argument structs (file FDs table, buffer iovecs, eventfd, restrictions, PERSONALITY creds) from user to a stack/heap object; whitelist sizes per-opcode and reject short/long copies before dereferencing.
- **PAX_USERCOPY on IORING_REGISTER_FILES table** — the `__s32 *fds` array and `io_uring_rsrc_register` payload land in a heap object that must be USERCOPY-allowlisted; per-entry width is exact (`sizeof(__s32)` for files, `sizeof(struct io_uring_rsrc_update)` for update), no slack.
- **PAX_KERNEXEC** — register handlers do not patch text; ensure no `set_memory_rw` on register-table dispatch arrays; the per-opcode function table is `__ro_after_init`.
- **PAX_RANDKSTACK** — REGISTER syscall enters with full stack randomization; per-opcode prep must not cache stack pointers across schedule (no aliasing of `&argv` after `cond_resched`).
- **PAX_REFCOUNT** — `ctx->refs`, personality `xa_node` refs, restriction-set refcount, BPF filter program refs all routed through hardened atomic; saturation traps before wrap.
- **PAX_MEMORY_SANITIZE** — on `IORING_UNREGISTER_*`, slab-poison the table memory (creds, restrictions, bpf-filter handle) before `kfree`; do not leave cred pointers reachable through freed slab.
- **PAX_UDEREF** — every register-arg dereference uses the SMAP-enforced accessors; no raw `__get_user` shortcuts on register fast paths.
- **PAX_RAP / kCFI** — per-opcode dispatch is a function-pointer call; emit kCFI prologue/epilogue, type-stamp the register-op signature, refuse mismatched indirect calls.
- **GRKERNSEC_HIDESYM** — error paths must not leak kernel-pointer values (e.g., `xa_node` address) into the dmesg/audit record for register failures.
- **GRKERNSEC_DMESG** — register-time failures emit at most one ratelimited line per ctx; redact PIDs/UIDs of remote ringfd issuers if `grsec_resource_logging` is off.
- **Per-op CAP gating** — RESTRICTIONS/PERSONALITY/MAP_BUFFERS each require a CAP check against submitter creds at register-time (no deferral to issue-time); UNREGISTER mirrors the CAP check to refuse stealth-disarm of a privileged restriction.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `io_uring/rsrc.c` resource management internals (covered in `rsrc.md` Tier-3)
- `io_uring/eventfd.c` eventfd glue (covered in `io_uring-core.md`)
- `io_uring/napi.c` busy-poll integration (covered in `net.md`)
- `io_uring/msg_ring.c` MSG_RING semantics (covered in `msg-ring.md`)
- `io_uring/cancel.c` sync_cancel inner (covered in `cancel.md`)
- `io_uring/zcrx.c` zero-copy receive interface (covered separately if expanded)
- `io_uring/query.c` query opcode (covered separately if expanded)
- `io_uring/bpf_filter.c` BPF filtering interface (covered separately if expanded)
- `io_uring/kbuf.c` PBUF_RING / PBUF_STATUS bodies (covered in `kbuf.md` Tier-3)
- Implementation code
