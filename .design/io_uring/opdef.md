# Tier-3: io_uring/opdef.c — Per-opcode definition table

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/opdef.c (~880 lines)
  - io_uring/opdef.h
  - include/uapi/linux/io_uring.h (IORING_OP_*)
-->

## Summary

The **io_uring opcode table** is the static dispatch fabric that maps `IORING_OP_*` enum values to (`prep`, `issue`, `cleanup`, `fail`, `sqe_copy`, `filter_populate`) function pointers and per-op flag bits (`needs_file`, `plug`, `ioprio`, `iopoll`, `buffer_select`, `hash_reg_file`, `unbound_nonreg_file`, `pollin`, `pollout`, `poll_exclusive`, `audit_skip`, `vectored`, `is_128`, `async_size`, `filter_pdu_size`). Per-table: two parallel arrays — `io_issue_defs[]` (hot-path: prep+issue+flags) and `io_cold_defs[]` (cold-path: name+cleanup+fail+sqe_copy). Per-`io_issue_sqe`: the table is read by opcode index after `mask_index(opcode, IORING_OP_LAST)`; missing opcode entries default-fill to `prep == io_eopnotsupp_prep` ⟹ `-EOPNOTSUPP`. Per-CONFIG: NET / EPOLL / FUTEX gates use `#if defined(CONFIG_*)` so opcodes degrade to `io_eopnotsupp_prep` when kconfig-disabled (no NULL pointer; preserves UAPI). Per-link_timeout: `io_no_issue` placeholder fires WARN+`-ECANCELED` because link timeouts never reach `issue()` (handled by the link machinery). Per-init: `io_uring_optable_init()` BUILD_BUG_ON-checks both arrays have exactly `IORING_OP_LAST` entries and BUG_On-checks every opcode has a non-NULL `prep`. Critical for: io_uring's pluggable opcode surface, kconfig-conditional UAPI degradation, BPF filter installation per op, audit-skip / iopoll / multishot per-op gating.

This Tier-3 covers `io_uring/opdef.c` (~880 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_issue_def` | hot per-op record (prep/issue/flags) | `IoIssueDef` |
| `struct io_cold_def` | cold per-op record (name/cleanup/fail/sqe_copy) | `IoColdDef` |
| `io_issue_defs[]` | hot-path opcode table (IORING_OP_LAST entries) | `IO_ISSUE_DEFS` |
| `io_cold_defs[]` | cold-path opcode table (IORING_OP_LAST entries) | `IO_COLD_DEFS` |
| `io_no_issue()` | placeholder issue fn (WARN + -ECANCELED) | `IoOpdef::no_issue` |
| `io_eopnotsupp_prep()` | placeholder prep fn (-EOPNOTSUPP) | `IoOpdef::eopnotsupp_prep` |
| `io_uring_get_opcode()` | per-opcode name lookup | `IoOpdef::get_opcode_name` |
| `io_uring_op_supported()` | per-opcode UAPI-supported query | `IoOpdef::is_supported` |
| `io_uring_optable_init()` | per-boot integrity check | `IoOpdef::optable_init` |
| `IORING_OP_LAST` | sentinel — array bound | UAPI constant |

## Compatibility contract

REQ-1: struct io_issue_def (per-op hot record):
- needs_file : 1 — req.file must be assigned by prep.
- plug : 1 — should plug block IO.
- ioprio : 1 — supports IORING_RECVSEND_FIXED_BUF / ioprio passthrough.
- iopoll : 1 — supports iopoll completion polling.
- buffer_select : 1 — supports IORING_OP_PROVIDE_BUFFERS group selection.
- hash_reg_file : 1 — workqueue hash by reg-file dev/ino.
- unbound_nonreg_file : 1 — submit to unbound wq if file is non-regular.
- pollin : 1 — supports POLLIN arm-on-EAGAIN.
- pollout : 1 — supports POLLOUT arm-on-EAGAIN.
- poll_exclusive : 1 — only one waiter on file at a time.
- audit_skip : 1 — bypass LSM audit on issue.
- vectored : 1 — handler distinguishes vector vs scalar.
- is_128 : 1 — opcode consumes 128-byte SQE in mixed-size ring.
- async_size : u16 — bytes of async slab to allocate.
- filter_pdu_size : u16 — BPF filter PDU size.
- issue : `fn(*IoKiocb, u32) -> i32`.
- prep : `fn(*IoKiocb, *const IoUringSqe) -> i32`.
- filter_populate : `fn(*IoUringBpfCtx, *IoKiocb)`.

REQ-2: struct io_cold_def (per-op cold record):
- name : `*const c_char` — opcode display name.
- sqe_copy : `fn(*IoKiocb)` — optional SQE deep-copy hook (URING_CMD).
- cleanup : `fn(*IoKiocb)` — release per-op buffers (cmd-cleanup path).
- fail : `fn(*IoKiocb)` — per-op fail-path hook (RW / sendrecv).

REQ-3: Two parallel arrays:
- io_issue_defs[IORING_OP_LAST] — hot fields only.
- io_cold_defs[IORING_OP_LAST] — cold fields only.
- Indexed in parallel by opcode; cache-line discipline: hot in one cache, cold in another.

REQ-4: Default (unfilled) opcode slots:
- prep == io_eopnotsupp_prep (autofilled by C designated-initializer zero behavior + the `__maybe_unused` static prepfunc).
- Implementations may explicitly assign it under #else branches when CONFIG_NET / CONFIG_EPOLL / CONFIG_FUTEX disable a build.

REQ-5: io_no_issue(req, flags):
- WARN_ON_ONCE(1).
- return -ECANCELED.
- Used for IORING_OP_LINK_TIMEOUT (handled by link machinery before issue is reached).

REQ-6: io_eopnotsupp_prep(kiocb, sqe):
- return -EOPNOTSUPP.

REQ-7: io_uring_get_opcode(opcode):
- if opcode < IORING_OP_LAST: return io_cold_defs[opcode].name.
- else: return "INVALID".

REQ-8: io_uring_op_supported(opcode):
- if opcode < IORING_OP_LAST ∧ io_issue_defs[opcode].prep != io_eopnotsupp_prep: return true.
- return false.

REQ-9: io_uring_optable_init() `__init`:
- BUILD_BUG_ON(ARRAY_SIZE(io_cold_defs) != IORING_OP_LAST).
- BUILD_BUG_ON(ARRAY_SIZE(io_issue_defs) != IORING_OP_LAST).
- for i in 0 .. ARRAY_SIZE(io_issue_defs):
  - BUG_ON(!io_issue_defs[i].prep).
  - if io_issue_defs[i].prep != io_eopnotsupp_prep:
    - BUG_ON(!io_issue_defs[i].issue).
  - WARN_ON_ONCE(!io_cold_defs[i].name).

REQ-10: Per-opcode profile (selected, illustrative — full table covers IORING_OP_NOP through IORING_OP_URING_CMD128):

| Opcode | needs_file | unbound_nonreg | pollin/out | plug | iopoll | buffer_select | audit_skip | async_size | prep | issue | cleanup | fail | special |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| NOP | – | – | – | – | y | – | y | – | io_nop_prep | io_nop | – | – | – |
| READV | y | y | pollin | y | y | y | y | io_async_rw | io_prep_readv | io_read | io_readv_writev_cleanup | io_rw_fail | vectored |
| WRITEV | y (hash_reg) | y | pollout | y | y | – | y | io_async_rw | io_prep_writev | io_write | io_readv_writev_cleanup | io_rw_fail | vectored |
| FSYNC | y | – | – | – | – | – | y | – | io_fsync_prep | io_fsync | – | – | – |
| READ_FIXED | y | y | pollin | y | y | – | y | io_async_rw | io_prep_read_fixed | io_read_fixed | io_readv_writev_cleanup | io_rw_fail | – |
| WRITE_FIXED | y (hash_reg) | y | pollout | y | y | – | y | io_async_rw | io_prep_write_fixed | io_write_fixed | io_readv_writev_cleanup | io_rw_fail | – |
| POLL_ADD | y | y | – | – | – | – | y | – | io_poll_add_prep | io_poll_add | – | – | – |
| POLL_REMOVE | – | – | – | – | – | – | y | – | io_poll_remove_prep | io_poll_remove | – | – | – |
| SYNC_FILE_RANGE | y | – | – | – | – | – | y | – | io_sfr_prep | io_sync_file_range | – | – | – |
| SENDMSG | y | y | pollout | – | – | – | – | io_async_msghdr | io_sendmsg_prep | io_sendmsg | io_sendmsg_recvmsg_cleanup | io_sendrecv_fail | CONFIG_NET |
| RECVMSG | y | y | pollin | – | – | y | – | io_async_msghdr | io_recvmsg_prep | io_recvmsg | io_sendmsg_recvmsg_cleanup | io_sendrecv_fail | CONFIG_NET |
| TIMEOUT | – | – | – | – | – | – | y | io_timeout_data | io_timeout_prep | io_timeout | – | – | – |
| TIMEOUT_REMOVE | – | – | – | – | – | – | y | – | io_timeout_remove_prep | io_timeout_remove | – | – | – |
| ACCEPT | y | y | pollin (excl) | – | – | – | – | – | io_accept_prep | io_accept | – | – | CONFIG_NET |
| ASYNC_CANCEL | – | – | – | – | – | – | y | – | io_async_cancel_prep | io_async_cancel | – | – | – |
| LINK_TIMEOUT | – | – | – | – | – | – | y | io_timeout_data | io_link_timeout_prep | io_no_issue | – | – | placeholder issue |
| CONNECT | y | y | pollout | – | – | – | – | io_async_msghdr | io_connect_prep | io_connect | – | – | CONFIG_NET |
| FALLOCATE | y (hash_reg) | – | – | – | – | – | – | – | io_fallocate_prep | io_fallocate | – | – | – |
| OPENAT | – | – | – | – | – | – | – | – | io_openat_prep | io_openat | io_open_cleanup | – | filter_pdu_size+filter_populate |
| CLOSE | – | – | – | – | – | – | – | – | io_close_prep | io_close | – | – | – |
| FILES_UPDATE | – | – | – | – | y | – | y | – | io_files_update_prep | io_files_update | – | – | – |
| STATX | – | – | – | – | – | – | y | – | io_statx_prep | io_statx | io_statx_cleanup | – | – |
| READ | y | y | pollin | y | y | y | y | io_async_rw | io_prep_read | io_read | io_readv_writev_cleanup | io_rw_fail | – |
| WRITE | y (hash_reg) | y | pollout | y | y | – | y | io_async_rw | io_prep_write | io_write | io_readv_writev_cleanup | io_rw_fail | – |
| FADVISE | y | – | – | – | – | – | y | – | io_fadvise_prep | io_fadvise | – | – | – |
| MADVISE | – | – | – | – | – | – | y | – | io_madvise_prep | io_madvise | – | – | – |
| SEND | y | y | pollout | – | – | y | y | io_async_msghdr | io_sendmsg_prep | io_send | io_sendmsg_recvmsg_cleanup | io_sendrecv_fail | CONFIG_NET |
| RECV | y | y | pollin | – | – | y | y | io_async_msghdr | io_recvmsg_prep | io_recv | io_sendmsg_recvmsg_cleanup | io_sendrecv_fail | CONFIG_NET |
| OPENAT2 | – | – | – | – | – | – | – | – | io_openat2_prep | io_openat2 | io_open_cleanup | – | filter_pdu_size+filter_populate |
| EPOLL_CTL | – | y | – | – | – | – | y | – | io_epoll_ctl_prep | io_epoll_ctl | – | – | CONFIG_EPOLL |
| SPLICE | y (hash_reg) | y | – | – | – | – | y | – | io_splice_prep | io_splice | io_splice_cleanup | – | – |
| PROVIDE_BUFFERS | – | – | – | – | y | – | y | – | io_provide_buffers_prep | io_manage_buffers_legacy | – | – | – |
| REMOVE_BUFFERS | – | – | – | – | y | – | y | – | io_remove_buffers_prep | io_manage_buffers_legacy | – | – | – |
| TEE | y (hash_reg) | y | – | – | – | – | y | – | io_tee_prep | io_tee | io_splice_cleanup | – | – |
| SHUTDOWN | y | – | – | – | – | – | – | – | io_shutdown_prep | io_shutdown | – | – | CONFIG_NET |
| RENAMEAT | – | – | – | – | – | – | – | – | io_renameat_prep | io_renameat | io_renameat_cleanup | – | – |
| UNLINKAT | – | – | – | – | – | – | – | – | io_unlinkat_prep | io_unlinkat | io_unlinkat_cleanup | – | – |
| MKDIRAT | – | – | – | – | – | – | – | – | io_mkdirat_prep | io_mkdirat | io_mkdirat_cleanup | – | – |
| SYMLINKAT | – | – | – | – | – | – | – | – | io_symlinkat_prep | io_symlinkat | io_link_cleanup | – | – |
| LINKAT | – | – | – | – | – | – | – | – | io_linkat_prep | io_linkat | io_link_cleanup | – | – |
| MSG_RING | y | – | – | – | y | – | – | – | io_msg_ring_prep | io_msg_ring | io_msg_ring_cleanup | – | – |
| FSETXATTR | y | – | – | – | – | – | – | – | io_fsetxattr_prep | io_fsetxattr | io_xattr_cleanup | – | – |
| SETXATTR | – | – | – | – | – | – | – | – | io_setxattr_prep | io_setxattr | io_xattr_cleanup | – | – |
| FGETXATTR | y | – | – | – | – | – | – | – | io_fgetxattr_prep | io_fgetxattr | io_xattr_cleanup | – | – |
| GETXATTR | – | – | – | – | – | – | – | – | io_getxattr_prep | io_getxattr | io_xattr_cleanup | – | – |
| SOCKET | – | – | – | – | – | – | y | – | io_socket_prep | io_socket | – | – | CONFIG_NET, filter_pdu_size+filter_populate |
| URING_CMD | y | – | – | y | y | y | – | io_async_cmd | io_uring_cmd_prep | io_uring_cmd | io_uring_cmd_cleanup | – | sqe_copy |
| SEND_ZC | y | y | pollout | – | – | – | y | io_async_msghdr | io_send_zc_prep | io_sendmsg_zc | io_send_zc_cleanup | io_sendrecv_fail | CONFIG_NET |
| SENDMSG_ZC | y | y | pollout | – | – | – | – | io_async_msghdr | io_send_zc_prep | io_sendmsg_zc | io_send_zc_cleanup | io_sendrecv_fail | CONFIG_NET |
| READ_MULTISHOT | y | y | pollin | – | – | y | y | io_async_rw | io_read_mshot_prep | io_read_mshot | io_readv_writev_cleanup | – | multishot |
| WAITID | – | – | – | – | – | – | – | io_waitid_async | io_waitid_prep | io_waitid | – | – | – |
| FUTEX_WAIT | – | – | – | – | – | – | – | – | io_futex_prep | io_futex_wait | – | – | CONFIG_FUTEX |
| FUTEX_WAKE | – | – | – | – | – | – | – | – | io_futex_prep | io_futex_wake | – | – | CONFIG_FUTEX |
| FUTEX_WAITV | – | – | – | – | – | – | – | – | io_futexv_prep | io_futexv_wait | – | – | CONFIG_FUTEX |
| FIXED_FD_INSTALL | y | – | – | – | – | – | – | – | io_install_fixed_fd_prep | io_install_fixed_fd | – | – | – |
| FTRUNCATE | y (hash_reg) | – | – | – | – | – | – | – | io_ftruncate_prep | io_ftruncate | – | – | – |
| BIND | y | – | – | – | – | – | – | io_async_msghdr | io_bind_prep | io_bind | – | – | CONFIG_NET |
| LISTEN | y | – | – | – | – | – | – | io_async_msghdr | io_listen_prep | io_listen | – | – | CONFIG_NET |
| RECV_ZC | y | y | pollin | – | – | – | – | – | io_recvzc_prep | io_recvzc | – | – | CONFIG_NET |
| EPOLL_WAIT | y | – | pollin | – | – | – | y | – | io_epoll_wait_prep | io_epoll_wait | – | – | CONFIG_EPOLL |
| READV_FIXED | y | y | pollin | y | y | – | y | io_async_rw | io_prep_readv_fixed | io_read | io_readv_writev_cleanup | io_rw_fail | vectored |
| WRITEV_FIXED | y (hash_reg) | y | pollout | y | y | – | y | io_async_rw | io_prep_writev_fixed | io_write | io_readv_writev_cleanup | io_rw_fail | vectored |
| PIPE | – | – | – | – | – | – | – | – | io_pipe_prep | io_pipe | – | – | – |
| NOP128 | – | – | – | – | y | – | y | – | io_nop_prep | io_nop | – | – | is_128 |
| URING_CMD128 | y | – | – | y | y | y | – | io_async_cmd | io_uring_cmd_prep | io_uring_cmd | io_uring_cmd_cleanup | – | is_128, sqe_copy |

REQ-11: CONFIG_NET / CONFIG_EPOLL / CONFIG_FUTEX gating:
- Per-op: `#if defined(CONFIG_X)` populates real prep/issue; `#else` sets `prep = io_eopnotsupp_prep` and omits `issue` so the BUG_ON in `optable_init` for non-eopnotsupp prep is skipped.
- UAPI invariant: opcode number remains stable across CONFIG_*; behavior degrades gracefully to -EOPNOTSUPP.

REQ-12: filter_pdu_size + filter_populate (BPF filter hook):
- Per-op: opens / sockets pre-populate an `io_uring_bpf_ctx` PDU so io_uring BPF filters can decide accept/reject before the operation runs.
- IORING_OP_OPENAT, IORING_OP_OPENAT2 use `sizeof_field(struct io_uring_bpf_ctx, open)` and `io_openat_bpf_populate`.
- IORING_OP_SOCKET uses `sizeof_field(struct io_uring_bpf_ctx, socket)` and `io_socket_bpf_populate`.

REQ-13: is_128 (mixed-size SQE flag):
- IORING_OP_NOP128, IORING_OP_URING_CMD128.
- Required when caller registered the ring with IORING_SETUP_SQE128 mixed.

REQ-14: audit_skip:
- Skip LSM/audit step on submission of trusted opcodes (NOP, fast-path RW, etc.).
- Default = 0 (audit runs).

REQ-15: iopoll:
- Op completes via `io_iopoll_check()` polling rather than IRQ-driven completion.
- Required for IORING_SETUP_IOPOLL rings.

REQ-16: poll_exclusive:
- ACCEPT: only one ACCEPT may arm POLLIN at a time on a given listener (avoids thundering-herd accept-wakeups).

REQ-17: vectored:
- READV / WRITEV / READV_FIXED / WRITEV_FIXED: handler distinguishes iovec from scalar buffer.

REQ-18: async_size:
- Bytes of allocator-fast-cache async-context the prep step needs (e.g., `sizeof(io_async_rw)`, `sizeof(io_async_msghdr)`, `sizeof(io_timeout_data)`, `sizeof(io_async_cmd)`, `sizeof(io_waitid_async)`).
- 0 = no async slab needed.

## Acceptance Criteria

- [ ] AC-1: io_uring_optable_init: panics at boot if `ARRAY_SIZE(io_issue_defs) != IORING_OP_LAST`.
- [ ] AC-2: io_uring_optable_init: panics at boot if any op has prep == NULL.
- [ ] AC-3: io_uring_optable_init: panics at boot if op has non-eopnotsupp prep but issue == NULL.
- [ ] AC-4: io_uring_optable_init: WARN if any op has cold.name == NULL.
- [ ] AC-5: Opcode >= IORING_OP_LAST: io_uring_get_opcode returns "INVALID".
- [ ] AC-6: io_uring_op_supported(NOP) returns true; io_uring_op_supported(invalid) returns false.
- [ ] AC-7: !CONFIG_NET: io_uring_op_supported(SENDMSG / RECVMSG / SEND / RECV / ACCEPT / CONNECT / SHUTDOWN / SOCKET / SEND_ZC / SENDMSG_ZC / BIND / LISTEN / RECV_ZC) returns false.
- [ ] AC-8: !CONFIG_EPOLL: io_uring_op_supported(EPOLL_CTL / EPOLL_WAIT) returns false.
- [ ] AC-9: !CONFIG_FUTEX: io_uring_op_supported(FUTEX_WAIT / FUTEX_WAKE / FUTEX_WAITV) returns false.
- [ ] AC-10: LINK_TIMEOUT submit + dispatched to issue: io_no_issue WARN_ON_ONCE; returns -ECANCELED.
- [ ] AC-11: Any opcode with needs_file=1: prep returns -EBADF if SQE.fd absent or invalid.
- [ ] AC-12: READV / WRITEV / READV_FIXED / WRITEV_FIXED: vectored == 1.
- [ ] AC-13: NOP128 / URING_CMD128: is_128 == 1.
- [ ] AC-14: URING_CMD / URING_CMD128: cold.sqe_copy == io_uring_cmd_sqe_copy.
- [ ] AC-15: ACCEPT: poll_exclusive == 1; only first ACCEPT waiter receives POLLIN edge.

## Architecture

```
struct IoIssueDef {
  needs_file: bool,           // 1-bit packed
  plug: bool,
  ioprio: bool,
  iopoll: bool,
  buffer_select: bool,
  hash_reg_file: bool,
  unbound_nonreg_file: bool,
  pollin: bool,
  pollout: bool,
  poll_exclusive: bool,
  audit_skip: bool,
  vectored: bool,
  is_128: bool,
  async_size: u16,
  filter_pdu_size: u16,
  issue: fn(req: *IoKiocb, issue_flags: u32) -> i32,
  prep: fn(req: *IoKiocb, sqe: *const IoUringSqe) -> i32,
  filter_populate: Option<fn(ctx: *IoUringBpfCtx, req: *IoKiocb)>,
}

struct IoColdDef {
  name: &'static str,
  sqe_copy: Option<fn(req: *IoKiocb)>,
  cleanup: Option<fn(req: *IoKiocb)>,
  fail: Option<fn(req: *IoKiocb)>,
}

const IO_ISSUE_DEFS: [IoIssueDef; IORING_OP_LAST] = /* per-opcode initializers */;
const IO_COLD_DEFS:  [IoColdDef;  IORING_OP_LAST] = /* per-opcode initializers */;
```

`IoOpdef::no_issue(req, flags) -> i32`:
1. WARN_ON_ONCE(true).
2. return -ECANCELED.

`IoOpdef::eopnotsupp_prep(kiocb, sqe) -> i32`:
1. return -EOPNOTSUPP.

`IoOpdef::get_opcode_name(opcode: u8) -> &'static str`:
1. if opcode < IORING_OP_LAST: return IO_COLD_DEFS[opcode as usize].name.
2. return "INVALID".

`IoOpdef::is_supported(opcode: u8) -> bool`:
1. if opcode >= IORING_OP_LAST: return false.
2. return IO_ISSUE_DEFS[opcode as usize].prep != IoOpdef::eopnotsupp_prep.

`IoOpdef::optable_init()` `__init`:
1. const_assert!(IO_ISSUE_DEFS.len() == IORING_OP_LAST as usize).  /* BUILD_BUG_ON */
2. const_assert!(IO_COLD_DEFS.len()  == IORING_OP_LAST as usize).  /* BUILD_BUG_ON */
3. for i in 0 .. IO_ISSUE_DEFS.len():
   - if IO_ISSUE_DEFS[i].prep.is_none(): panic("prep == null at opcode {i}").  /* BUG_ON */
   - if IO_ISSUE_DEFS[i].prep != IoOpdef::eopnotsupp_prep ∧ IO_ISSUE_DEFS[i].issue.is_none():
     - panic("issue == null at opcode {i} (prep is real)").  /* BUG_ON */
   - if IO_COLD_DEFS[i].name.is_empty(): warn_once("name == null at opcode {i}").  /* WARN_ON_ONCE */

Per `io_issue_sqe(req)` dispatch path (consumer):
1. def = &IO_ISSUE_DEFS[req.opcode as usize].
2. if def.needs_file ∧ req.file.is_none(): return -EBADF.
3. ret = def.issue(req, issue_flags).
4. /* iopoll / pollin / pollout / buffer_select / async_size honored upstream */

Per `io_clean_op(req)` (cleanup path):
1. def = &IO_COLD_DEFS[req.opcode as usize].
2. if def.cleanup.is_some(): def.cleanup(req).

Per `io_req_defer_failed(req)`:
1. def = &IO_COLD_DEFS[req.opcode as usize].
2. if def.fail.is_some(): def.fail(req).

Per opcode profile (rust per-op initializer pattern, abridged):

```
IORING_OP_READV => IoIssueDef {
  needs_file: true, unbound_nonreg_file: true, pollin: true,
  buffer_select: true, plug: true, audit_skip: true, ioprio: true,
  iopoll: true, vectored: true,
  async_size: size_of::<IoAsyncRw>() as u16,
  prep: io_prep_readv, issue: io_read, .. Default::default()
},
IORING_OP_LINK_TIMEOUT => IoIssueDef {
  audit_skip: true,
  async_size: size_of::<IoTimeoutData>() as u16,
  prep: io_link_timeout_prep, issue: IoOpdef::no_issue,
  .. Default::default()
},
IORING_OP_OPENAT => IoIssueDef {
  filter_pdu_size: size_of_field!(IoUringBpfCtx, open) as u16,
  prep: io_openat_prep, issue: io_openat,
  filter_populate: Some(io_openat_bpf_populate),
  .. Default::default()
},
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `issue_defs_length_eq_op_last` | INVARIANT | per-build: IO_ISSUE_DEFS.len() == IORING_OP_LAST. |
| `cold_defs_length_eq_op_last` | INVARIANT | per-build: IO_COLD_DEFS.len() == IORING_OP_LAST. |
| `prep_never_null` | INVARIANT | per-op: prep != NULL after optable_init. |
| `real_prep_implies_real_issue` | INVARIANT | per-op: prep != eopnotsupp_prep ⟹ issue != NULL. |
| `name_set_for_supported` | INVARIANT | per-op: is_supported ⟹ cold.name != NULL. |
| `is_128_only_on_designated_ops` | INVARIANT | per-op: is_128 set only on NOP128, URING_CMD128. |
| `vectored_only_on_vector_ops` | INVARIANT | per-op: vectored set only on READV, WRITEV, READV_FIXED, WRITEV_FIXED. |
| `poll_exclusive_only_on_accept` | INVARIANT | per-op: poll_exclusive set only on ACCEPT. |
| `hash_reg_implies_needs_file` | INVARIANT | per-op: hash_reg_file ⟹ needs_file. |
| `filter_populate_paired_with_pdu_size` | INVARIANT | per-op: filter_populate set ⟺ filter_pdu_size > 0. |
| `eopnotsupp_returns_correct_errno` | INVARIANT | per-call: eopnotsupp_prep returns -EOPNOTSUPP. |
| `no_issue_returns_canceled` | INVARIANT | per-call: no_issue returns -ECANCELED + WARN_ON_ONCE. |

### Layer 2: TLA+

`io_uring/opdef.tla`:
- Per-boot: optable_init enforces per-op invariants over the table.
- Per-submit: io_issue_sqe(opcode) → table lookup → prep → issue → cleanup chain.
- Properties:
  - `safety_no_oob_lookup` — per-submit: opcode < IORING_OP_LAST always before table access.
  - `safety_prep_always_called_before_issue` — per-submit: issue not invoked without successful prep.
  - `safety_cleanup_called_on_terminal` — per-submit: cleanup invoked iff req has REQ_F_NEED_CLEANUP.
  - `safety_unsupported_returns_eopnotsupp` — per-submit: !is_supported(opcode) ⟹ -EOPNOTSUPP.
  - `safety_link_timeout_never_issued` — per-submit: LINK_TIMEOUT never reaches issue() (handled by link path).
  - `liveness_supported_op_reaches_issue` — per-submit: is_supported(opcode) ∧ valid SQE ⟹ issue invoked.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `optable_init` post: every entry has prep; either prep == eopnotsupp or issue != NULL | `IoOpdef::optable_init` |
| `get_opcode_name` post: opcode < IORING_OP_LAST ⟹ matches `IO_COLD_DEFS[opcode].name` | `IoOpdef::get_opcode_name` |
| `is_supported` post: returns prep != eopnotsupp_prep for in-range opcodes | `IoOpdef::is_supported` |
| `no_issue` post: returns -ECANCELED | `IoOpdef::no_issue` |
| `eopnotsupp_prep` post: returns -EOPNOTSUPP | `IoOpdef::eopnotsupp_prep` |
| Table: `needs_file` ⇒ caller resolves req.file before issue | submit path |
| Table: `async_size` matches the structure allocated by prep | each prep |

### Layer 4: Verus/Creusot functional

`Per-io_issue_sqe → opcode bound-check → table-row lookup → flag-driven setup (needs_file / async_size / plug / iopoll) → prep → (poll/iopoll/issue) → cleanup / fail` semantic equivalence: per-Documentation/io_uring.rst + per-io_uring_enter(2) + per-io_uring_setup(2) manual pages.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

Opdef-table reinforcement:

- **Per-build-time array-size check (IORING_OP_LAST)** — defense against per-table-drift miscount.
- **Per-boot BUG_ON prep == NULL** — defense against per-NULL-deref on submission.
- **Per-boot BUG_ON real-prep + NULL-issue** — defense against per-half-implemented opcode.
- **Per-boot WARN cold.name == NULL** — defense against per-display-NULL in fdinfo.
- **Per-opcode bound-check before table lookup** — defense against per-OOB read.
- **Per-CONFIG-gated UAPI degradation to -EOPNOTSUPP** — defense against per-stub-NULL-deref when kconfig disables.
- **Per-LINK_TIMEOUT io_no_issue placeholder** — defense against per-link-bypass executing the op.
- **Per-needs_file ⇒ file resolved before issue** — defense against per-issue-NULL-file.
- **Per-hash_reg_file workqueue partitioning** — defense against per-regular-file write-amplification through unbound wq.
- **Per-poll_exclusive on ACCEPT** — defense against per-thundering-herd accept wake-storm.
- **Per-audit_skip explicit opt-in** — defense against per-bypass-by-default audit elision.
- **Per-filter_populate paired with filter_pdu_size** — defense against per-uninitialized BPF PDU.
- **Per-is_128 only on 128-byte-SQE opcodes** — defense against per-SQE-size confusion.
- **Per-async_size pre-allocated by prep** — defense against per-issue-time async-OOM mid-op.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- io_uring/io_uring.c submit / issue / completion loop (covered in `io_uring-core.md` Tier-3)
- io_uring/rw.c / net.c / poll.c per-op implementations (covered in their own Tier-3 docs)
- io_uring/cmd.c URING_CMD payload handling (covered in `io_uring-core.md` Tier-3)
- io_uring/fdinfo.c — name printing consumer (covered separately)
- io_uring BPF filter framework `io_uring_bpf_ctx` lifecycle (covered separately if expanded)
- Implementation code
