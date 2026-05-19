# Tier-5: include/uapi/linux/io_uring.h — io_uring UAPI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/io_uring.h (~1065 lines)
  - include/uapi/linux/io_uring/zcrx.h
  - include/uapi/linux/io_uring/query.h
-->

## Summary

`io_uring` is the user-visible **asynchronous IO ABI** based on a pair of shared-memory **submission (SQ)** and **completion (CQ)** ring buffers mapped between the kernel and userspace. The ring fd is created by `io_uring_setup(2)` (returns `struct io_uring_params` with offsets), driven by `io_uring_enter(2)` (submit + reap), and configured by `io_uring_register(2)` (fixed buffers, files, eventfd, probe, NAPI, zcrx, BPF filter). Per-`struct io_uring_sqe` (64 bytes default, 128 bytes with `IORING_SETUP_SQE128`) carries the opcode + per-op union of flags + user_data cookie + addr/len/off + buf_index/file_index + personality. Per-`struct io_uring_cqe` (16 bytes default, 32 bytes with `IORING_SETUP_CQE32`) carries user_data + res (errno or count) + flags (buffer-id, MORE, NOTIF, SOCK_NONEMPTY, ...). Per-IORING_OP_* covers ~60 opcodes spanning READV/WRITEV, NET (SENDMSG/RECVMSG/CONNECT/ACCEPT/SEND_ZC), FS (OPENAT/STATX/UNLINKAT), POLL/TIMEOUT, FUTEX/WAITID, URING_CMD pass-through, plus zerocopy/multishot variants. Per-IORING_REGISTER_* covers 37+ register ops (BUFFERS, FILES, EVENTFD, PROBE, RESTRICTIONS, PBUF_RING, SYNC_CANCEL, NAPI, MEM_REGION, ZCRX_IFQ, QUERY, BPF_FILTER, ...). Per-`IORING_OFF_*` magic offsets identify the ring memory region (SQ, CQ, SQES, PBUF_RING) accepted by `mmap(2)`. Critical for: GPL-clean ABI-compat asynchronous IO surface used by liburing and any io_uring-aware userspace.

This Tier-5 covers `include/uapi/linux/io_uring.h` (~1065 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_uring_sqe` | per-submission entry (64B / 128B) | `IoUringSqe` |
| `struct io_uring_cqe` | per-completion entry (16B / 32B) | `IoUringCqe` |
| `struct io_uring_attr_pi` | per-PI attribute payload | `IoUringAttrPi` |
| `struct io_uring_params` | per-setup descriptor | `IoUringParams` |
| `struct io_sqring_offsets` | per-mmap SQ offsets | `IoSqringOffsets` |
| `struct io_cqring_offsets` | per-mmap CQ offsets | `IoCqringOffsets` |
| `struct io_uring_files_update` | per-deprecated files update | `IoUringFilesUpdate` |
| `struct io_uring_rsrc_register` | per-buffer/file register (v2) | `IoUringRsrcRegister` |
| `struct io_uring_rsrc_update` | per-rsrc-update (v1) | `IoUringRsrcUpdate` |
| `struct io_uring_rsrc_update2` | per-rsrc-update (tagged v2) | `IoUringRsrcUpdate2` |
| `struct io_uring_probe` / `_probe_op` | per-feature probe reply | `IoUringProbe` / `Op` |
| `struct io_uring_restriction` / `_task_restriction` | per-allowed-op gate | `IoUringRestriction` |
| `struct io_uring_clock_register` | per-clockid select | `IoUringClockRegister` |
| `struct io_uring_clone_buffers` | per-cross-ring buffer-clone | `IoUringCloneBuffers` |
| `struct io_uring_buf` / `_buf_ring` / `_buf_reg` / `_buf_status` | per-provided-buffers ring | `IoUringBuf*` |
| `struct io_uring_napi` | per-NAPI busy-poll setup | `IoUringNapi` |
| `struct io_uring_reg_wait` | per-fixed wait region | `IoUringRegWait` |
| `struct io_uring_getevents_arg` | per-EXT_ARG | `IoUringGeteventsArg` |
| `struct io_uring_sync_cancel_reg` | per-sync-cancel arg | `IoUringSyncCancelReg` |
| `struct io_uring_file_index_range` | per-FILE_ALLOC_RANGE arg | `IoUringFileIndexRange` |
| `struct io_uring_recvmsg_out` | per-RECVMSG cqe-out | `IoUringRecvmsgOut` |
| `struct io_uring_region_desc` / `mem_region_reg` | per-MEM_REGION arg | `IoUringMemRegion*` |
| `struct io_timespec` | per-io_uring timespec | `IoTimespec` |
| `enum io_uring_op` | IORING_OP_* opcodes | `IoUringOp` |
| `enum io_uring_register_op` | IORING_REGISTER_* opcodes | `IoUringRegisterOp` |
| `enum io_uring_msg_ring_flags` | MSG_RING command types | `IoUringMsgRingFlags` |
| `enum io_uring_socket_op` | SOCKET_URING_OP_* | `IoUringSocketOp` |
| `enum io_uring_napi_op` / `_tracking_strategy` | NAPI sub-ops | `IoUringNapi*` |
| `enum io_uring_register_pbuf_ring_flags` | IOU_PBUF_RING_* | `IoUringPbufRingFlags` |
| `enum io_uring_register_restriction_op` | IORING_RESTRICTION_* | `IoUringRestrictionOp` |
| `enum io_wq_type` | IO_WQ_BOUND / UNBOUND | `IoWqType` |
| `IORING_OFF_*` mmap offsets | per-region magic offset | shared constants |

## ABI surface (constants + structs)

### sqe->flags (IOSQE_*)

- `IOSQE_FIXED_FILE` (bit 0) — `sqe->fd` is an index into the fixed-file table.
- `IOSQE_IO_DRAIN` (bit 1) — issue after all in-flight IO completes.
- `IOSQE_IO_LINK` (bit 2) — link next SQE; failure breaks chain.
- `IOSQE_IO_HARDLINK` (bit 3) — link next SQE; never broken by failure.
- `IOSQE_ASYNC` (bit 4) — always offload to io-wq.
- `IOSQE_BUFFER_SELECT` (bit 5) — select buffer from `sqe->buf_group`.
- `IOSQE_CQE_SKIP_SUCCESS` (bit 6) — suppress CQE if request returned success.

### IORING_SETUP_* (io_uring_params.flags)

`IOPOLL` (1<<0), `SQPOLL` (1<<1), `SQ_AFF` (1<<2), `CQSIZE` (1<<3), `CLAMP` (1<<4), `ATTACH_WQ` (1<<5), `R_DISABLED` (1<<6), `SUBMIT_ALL` (1<<7), `COOP_TASKRUN` (1<<8), `TASKRUN_FLAG` (1<<9), `SQE128` (1<<10), `CQE32` (1<<11), `SINGLE_ISSUER` (1<<12), `DEFER_TASKRUN` (1<<13), `NO_MMAP` (1<<14), `REGISTERED_FD_ONLY` (1<<15), `NO_SQARRAY` (1<<16), `HYBRID_IOPOLL` (1<<17), `CQE_MIXED` (1<<18), `SQE_MIXED` (1<<19), `SQ_REWIND` (1<<20).

### IORING_FEAT_* (io_uring_params.features, output)

`SINGLE_MMAP` (1<<0), `NODROP` (1<<1), `SUBMIT_STABLE` (1<<2), `RW_CUR_POS` (1<<3), `CUR_PERSONALITY` (1<<4), `FAST_POLL` (1<<5), `POLL_32BITS` (1<<6), `SQPOLL_NONFIXED` (1<<7), `EXT_ARG` (1<<8), `NATIVE_WORKERS` (1<<9), `RSRC_TAGS` (1<<10), `CQE_SKIP` (1<<11), `LINKED_FILE` (1<<12), `REG_REG_RING` (1<<13), `RECVSEND_BUNDLE` (1<<14), `MIN_TIMEOUT` (1<<15), `RW_ATTR` (1<<16), `NO_IOWAIT` (1<<17).

### IORING_OP_* (enum io_uring_op)

`NOP` (0), `READV`, `WRITEV`, `FSYNC`, `READ_FIXED`, `WRITE_FIXED`, `POLL_ADD`, `POLL_REMOVE`, `SYNC_FILE_RANGE`, `SENDMSG`, `RECVMSG`, `TIMEOUT`, `TIMEOUT_REMOVE`, `ACCEPT`, `ASYNC_CANCEL`, `LINK_TIMEOUT`, `CONNECT`, `FALLOCATE`, `OPENAT`, `CLOSE`, `FILES_UPDATE`, `STATX`, `READ`, `WRITE`, `FADVISE`, `MADVISE`, `SEND`, `RECV`, `OPENAT2`, `EPOLL_CTL`, `SPLICE`, `PROVIDE_BUFFERS`, `REMOVE_BUFFERS`, `TEE`, `SHUTDOWN`, `RENAMEAT`, `UNLINKAT`, `MKDIRAT`, `SYMLINKAT`, `LINKAT`, `MSG_RING`, `FSETXATTR`, `SETXATTR`, `FGETXATTR`, `GETXATTR`, `SOCKET`, `URING_CMD`, `SEND_ZC`, `SENDMSG_ZC`, `READ_MULTISHOT`, `WAITID`, `FUTEX_WAIT`, `FUTEX_WAKE`, `FUTEX_WAITV`, `FIXED_FD_INSTALL`, `FTRUNCATE`, `BIND`, `LISTEN`, `RECV_ZC`, `EPOLL_WAIT`, `READV_FIXED`, `WRITEV_FIXED`, `PIPE`, `NOP128`, `URING_CMD128`. Sentinel `IORING_OP_LAST`.

### IORING_REGISTER_* (enum io_uring_register_op)

`REGISTER_BUFFERS` (0), `UNREGISTER_BUFFERS` (1), `REGISTER_FILES` (2), `UNREGISTER_FILES` (3), `REGISTER_EVENTFD` (4), `UNREGISTER_EVENTFD` (5), `REGISTER_FILES_UPDATE` (6), `REGISTER_EVENTFD_ASYNC` (7), `REGISTER_PROBE` (8), `REGISTER_PERSONALITY` (9), `UNREGISTER_PERSONALITY` (10), `REGISTER_RESTRICTIONS` (11), `REGISTER_ENABLE_RINGS` (12), `REGISTER_FILES2` (13), `REGISTER_FILES_UPDATE2` (14), `REGISTER_BUFFERS2` (15), `REGISTER_BUFFERS_UPDATE` (16), `REGISTER_IOWQ_AFF` (17), `UNREGISTER_IOWQ_AFF` (18), `REGISTER_IOWQ_MAX_WORKERS` (19), `REGISTER_RING_FDS` (20), `UNREGISTER_RING_FDS` (21), `REGISTER_PBUF_RING` (22), `UNREGISTER_PBUF_RING` (23), `REGISTER_SYNC_CANCEL` (24), `REGISTER_FILE_ALLOC_RANGE` (25), `REGISTER_PBUF_STATUS` (26), `REGISTER_NAPI` (27), `UNREGISTER_NAPI` (28), `REGISTER_CLOCK` (29), `REGISTER_CLONE_BUFFERS` (30), `REGISTER_SEND_MSG_RING` (31), `REGISTER_ZCRX_IFQ` (32), `REGISTER_RESIZE_RINGS` (33), `REGISTER_MEM_REGION` (34), `REGISTER_QUERY` (35), `REGISTER_ZCRX_CTRL` (36), `REGISTER_BPF_FILTER` (37). Sentinel `REGISTER_LAST`. High bit `REGISTER_USE_REGISTERED_RING` (1U << 31) selects ring fd indirection.

### IORING_ASYNC_CANCEL_* (sqe->cancel_flags / sync_cancel arg)

`ALL` (1<<0), `FD` (1<<1), `ANY` (1<<2), `FD_FIXED` (1<<3), `USERDATA` (1<<4), `OP` (1<<5).

### IORING_TIMEOUT_* (sqe->timeout_flags)

`ABS` (1<<0), `UPDATE` (1<<1), `BOOTTIME` (1<<2), `REALTIME` (1<<3), `LINK_TIMEOUT_UPDATE` (1<<4), `ETIME_SUCCESS` (1<<5), `MULTISHOT` (1<<6), `IMMEDIATE_ARG` (1<<7). Masks: `CLOCK_MASK` = `BOOTTIME | REALTIME`; `UPDATE_MASK` = `UPDATE | LINK_TIMEOUT_UPDATE`.

### IORING_RECVSEND_* / IORING_SEND_* (sqe->ioprio for send/recv variants)

`RECVSEND_POLL_FIRST` (1<<0), `RECV_MULTISHOT` (1<<1), `RECVSEND_FIXED_BUF` (1<<2), `SEND_ZC_REPORT_USAGE` (1<<3), `RECVSEND_BUNDLE` (1<<4), `SEND_VECTORIZED` (1<<5). `IORING_NOTIF_USAGE_ZC_COPIED` (1U<<31) as cqe.res flag.

### IORING_POLL_* (sqe->len for POLL_ADD)

`ADD_MULTI` (1<<0), `UPDATE_EVENTS` (1<<1), `UPDATE_USER_DATA` (1<<2), `ADD_LEVEL` (1<<3).

### IORING_URING_CMD_*

`FIXED` (1<<0), `MULTISHOT` (1<<1), mask `FIXED | MULTISHOT`. Top 8 bits reserved (kernel-only).

### IORING_FSYNC_DATASYNC

bit 0 of `sqe->fsync_flags`.

### IORING_ACCEPT_* (sqe->ioprio)

`MULTISHOT` (1<<0), `DONTWAIT` (1<<1), `POLL_FIRST` (1<<2).

### IORING_MSG_RING (sqe->addr selects sub-op)

`IORING_MSG_DATA` (0), `IORING_MSG_SEND_FD` (1). Flags (sqe->msg_ring_flags): `CQE_SKIP` (1<<0), `FLAGS_PASS` (1<<1).

### IORING_FIXED_FD_INSTALL flags

`IORING_FIXED_FD_NO_CLOEXEC` (1<<0).

### IORING_NOP_* (sqe->nop_flags)

`INJECT_RESULT` (1<<0), `FILE` (1<<1), `FIXED_FILE` (1<<2), `FIXED_BUFFER` (1<<3), `TW` (1<<4), `CQE32` (1<<5).

### IORING_RW_ATTR_* (sqe->attr_type_mask)

`FLAG_PI` (1U<<0).

### IORING_CQE_F_* (cqe->flags)

`BUFFER` (1<<0; upper 16 bits = buffer ID), `MORE` (1<<1), `SOCK_NONEMPTY` (1<<2), `NOTIF` (1<<3), `BUF_MORE` (1<<4), `SKIP` (1<<5), `F_32` (1<<15). Shift: `IORING_CQE_BUFFER_SHIFT` = 16.

### IORING_OFF_* mmap offsets

`SQ_RING` (`0`), `CQ_RING` (`0x8000000`), `SQES` (`0x10000000`), `PBUF_RING` (`0x80000000`), `PBUF_SHIFT` (16), `MMAP_MASK` (`0xf8000000`).

### IORING_SQ_* (sq_ring->flags)

`NEED_WAKEUP` (1<<0), `CQ_OVERFLOW` (1<<1), `TASKRUN` (1<<2).

### IORING_CQ_* (cq_ring->flags)

`EVENTFD_DISABLED` (1<<0).

### IORING_ENTER_* (io_uring_enter flags)

`GETEVENTS` (1<<0), `SQ_WAKEUP` (1<<1), `SQ_WAIT` (1<<2), `EXT_ARG` (1<<3), `REGISTERED_RING` (1<<4), `ABS_TIMER` (1<<5), `EXT_ARG_REG` (1<<6), `NO_IOWAIT` (1<<7).

### IORING_RSRC_* / IORING_REGISTER_FILES_SKIP

`IORING_RSRC_REGISTER_SPARSE` (1<<0). `IORING_REGISTER_FILES_SKIP` (-2) marks no-update.

### IOU_PBUF_RING_* (io_uring_buf_reg.flags)

`MMAP` (1), `INC` (2).

### IORING_REG_WAIT_TS

(1<<0) — io_uring_reg_wait.flags marker.

### IORING_REGISTER_SRC_REGISTERED / DST_REPLACE (clone_buffers)

`SRC_REGISTERED` (1<<0), `DST_REPLACE` (1<<1).

### IORING_MEM_REGION_TYPE_USER

(1) — region backed by user_addr. `IORING_MEM_REGION_REG_WAIT_ARG` (1) — region exposed as registered wait arg.

### IORING_FILE_INDEX_ALLOC

`(~0U)` — sentinel that requests kernel-side allocation.

### IORING_RESTRICTION_*

`REGISTER_OP` (0), `SQE_OP` (1), `SQE_FLAGS_ALLOWED` (2), `SQE_FLAGS_REQUIRED` (3). Sentinel `RESTRICTION_LAST`.

### NAPI sub-enums

`io_uring_napi_op`: `REGISTER_OP` (0), `STATIC_ADD_ID` (1), `STATIC_DEL_ID` (2). `io_uring_napi_tracking_strategy`: `DYNAMIC` (0), `STATIC` (1), `INACTIVE` (255).

### Socket URING_CMD sub-ops (sqe->cmd_op when file is a socket)

`SOCKET_URING_OP_SIOCINQ` (0), `SOCKET_URING_OP_SIOCOUTQ`, `GETSOCKOPT`, `SETSOCKOPT`, `TX_TIMESTAMP`, `GETSOCKNAME`. TX timestamp shifts: `IORING_TIMESTAMP_HW_SHIFT` (16), `IORING_TIMESTAMP_TYPE_SHIFT` (17). cqe flag `IORING_CQE_F_TSTAMP_HW` = (1 << 16).

### IO_URING_OP_SUPPORTED

(1<<0) — bit set in `io_uring_probe_op.flags` if opcode is supported.

### io_wq_type

`IO_WQ_BOUND` (0), `IO_WQ_UNBOUND` (1).

### SPLICE_F_FD_IN_FIXED

(1U<<31) — sqe->splice_flags extension marking fixed splice-fd-in.

## Compatibility contract

REQ-1: `struct io_uring_sqe` is exactly **64 bytes** when `IORING_SETUP_SQE128` is not set. When set, each entry is **128 bytes** (second half is `cmd[80]` arbitrary command data). The leading 64-byte layout is binary-stable and matches the upstream union exactly: `opcode:u8, flags:u8, ioprio:u16, fd:s32, {off|addr2|{cmd_op:u32,__pad1:u32}}:u64, {addr|splice_off_in|{level:u32,optname:u32}}:u64, len:u32, <op_flags_union>:u32, user_data:u64, {buf_index|buf_group}:packed u16, personality:u16, {splice_fd_in|file_index|zcrx_ifq_idx|optlen|{addr_len:u16,__pad3:u16}|{write_stream:u8,__pad4:u8[3]}}:u32, {{addr3:u64,__pad2:u64}|{attr_ptr:u64,attr_type_mask:u64}|optval:u64|cmd[0]:u8}:u128`.

REQ-2: `struct io_uring_cqe` is exactly **16 bytes** when `IORING_SETUP_CQE32` is not set. When set, each entry is **32 bytes** (extra `big_cqe[2]` u64). Mixed mode (`IORING_SETUP_CQE_MIXED`) tags 32-byte entries with `IORING_CQE_F_32`. Mixed mode (`IORING_SETUP_SQE_MIXED`) tags 128-byte sqes via a 128b opcode.

REQ-3: `struct io_uring_params` is binary-stable: `sq_entries:u32, cq_entries:u32, flags:u32, sq_thread_cpu:u32, sq_thread_idle:u32, features:u32, wq_fd:u32, resv[3]:u32, sq_off:io_sqring_offsets, cq_off:io_cqring_offsets`. Caller sets flags+entries; kernel returns features+offsets. `resv[3]` must be zero on input.

REQ-4: `struct io_sqring_offsets` / `struct io_cqring_offsets`: per-field offsets into the SQ/CQ ring memory: `head:u32, tail:u32, ring_mask:u32, ring_entries:u32, flags:u32, dropped|overflow:u32, array|cqes:u32, resv1:u32, user_addr:u64`. `user_addr` is the userspace pointer when `IORING_SETUP_NO_MMAP` is used.

REQ-5: `IORING_OFF_*` magic offsets identify the ring memory in `mmap(2)`. With `IORING_FEAT_SINGLE_MMAP` the SQ-ring and CQ-ring share one mapping; without it, two mappings are required. `IORING_OFF_PBUF_RING | (bgid << IORING_OFF_PBUF_SHIFT)` selects a provided-buffer ring by group id. `IORING_OFF_MMAP_MASK` covers the high bits.

REQ-6: `struct io_uring_buf_ring` and `struct io_uring_buf`: the ring is an array of `io_uring_buf{addr:u64, len:u32, bid:u16, resv:u16}` whose first slot is aliased with `{resv1:u64, resv2:u32, resv3:u16, tail:u16}`. Tail is updated by userspace; kernel reads bufs. Ring may be `MMAP`-allocated by kernel (`IOU_PBUF_RING_MMAP`) or pre-allocated by user (`ring_addr`). `IOU_PBUF_RING_INC` enables incremental consumption.

REQ-7: `struct io_uring_buf_reg`: `ring_addr:u64, ring_entries:u32, bgid:u16, flags:u16, min_left:u32, resv[5]:u32`. `ring_entries` must be a power of two ≤ 32768.

REQ-8: `struct io_uring_buf_status`: `buf_group:u32` (input), `head:u32` (output), `resv[8]:u32`.

REQ-9: `struct io_uring_probe`: `last_op:u8, ops_len:u8, resv:u16, resv2[3]:u32, ops[]:io_uring_probe_op` where each op = `op:u8, resv:u8, flags:u16, resv2:u32`. Flag `IO_URING_OP_SUPPORTED` (1<<0) marks supported opcodes.

REQ-10: `struct io_uring_files_update` (deprecated) = `offset:u32, resv:u32, fds:__aligned_u64`. Superseded by `io_uring_rsrc_update`/`_update2`.

REQ-11: `struct io_uring_rsrc_register`: `nr:u32, flags:u32, resv2:u64, data:__aligned_u64, tags:__aligned_u64`. `flags=IORING_RSRC_REGISTER_SPARSE` registers fully sparse files with no fd array.

REQ-12: `struct io_uring_rsrc_update2`: `offset:u32, resv:u32, data:__aligned_u64, tags:__aligned_u64, nr:u32, resv2:u32`. Tagged variant tracks per-resource cookies returned in CQE on release.

REQ-13: `struct io_uring_napi`: `busy_poll_to:u32, prefer_busy_poll:u8, opcode:u8, pad[2]:u8, op_param:u32, resv:u32`. `opcode` is an `io_uring_napi_op`; `op_param` is the strategy for REGISTER_OP or a napi id for STATIC_ADD/DEL.

REQ-14: `struct io_uring_restriction`: `opcode:u16, {register_op|sqe_op|sqe_flags}:u8, resv:u8, resv2[3]:u32`. Restrictions are applied once before `IORING_REGISTER_ENABLE_RINGS`.

REQ-15: `struct io_uring_sync_cancel_reg`: `addr:u64, fd:s32, flags:u32, timeout:__kernel_timespec, opcode:u8, pad[7]:u8, pad2[3]:u64`.

REQ-16: `struct io_uring_reg_wait`: `ts:__kernel_timespec, min_wait_usec:u32, flags:u32, sigmask:u64, sigmask_sz:u32, pad[3]:u32, pad2[2]:u64`. Indexed by ext-arg-reg.

REQ-17: `struct io_uring_getevents_arg`: `sigmask:u64, sigmask_sz:u32, min_wait_usec:u32, ts:u64`. Passed when `IORING_ENTER_EXT_ARG` set.

REQ-18: `struct io_uring_recvmsg_out`: `namelen:u32, controllen:u32, payloadlen:u32, flags:u32`. Prepended to CQE buffer payload for OP_RECVMSG.

REQ-19: `struct io_uring_mem_region_reg` / `struct io_uring_region_desc`: regions allow registering user-provided memory for fixed wait args or future uses.

REQ-20: All structures use `__aligned_u64` for userspace pointer slots and explicit `resv*` padding. Padding fields MUST be written as zero by userspace; kernel rejects nonzero reserved bits to keep room for ABI growth.

## Acceptance Criteria

- [ ] AC-1: `sizeof(IoUringSqe) == 64` and `sizeof(IoUringSqe128) == 128`.
- [ ] AC-2: `sizeof(IoUringCqe) == 16` and `sizeof(IoUringCqe32) == 32`.
- [ ] AC-3: All `IOSQE_*`, `IORING_SETUP_*`, `IORING_FEAT_*`, `IORING_ENTER_*`, `IORING_CQE_F_*`, `IORING_SQ_*`, `IORING_CQ_*` bit values match the C header byte-for-byte.
- [ ] AC-4: `IORING_OP_LAST` count matches the upstream enumeration (currently `URING_CMD128` + 1).
- [ ] AC-5: `IORING_REGISTER_LAST` and `IORING_REGISTER_USE_REGISTERED_RING` (1<<31) match upstream.
- [ ] AC-6: `IORING_OFF_SQ_RING`/`CQ_RING`/`SQES`/`PBUF_RING`/`PBUF_SHIFT`/`MMAP_MASK` match upstream constants.
- [ ] AC-7: `IORING_FILE_INDEX_ALLOC == ~0u32`; `IORING_REGISTER_FILES_SKIP == -2`.
- [ ] AC-8: `struct io_uring_params` layout: offset of `sq_off` == 32, `cq_off` == 32 + sizeof(io_sqring_offsets) = 64. Total size == 32 + sizeof(sq_off) + sizeof(cq_off) = 120.
- [ ] AC-9: `struct io_sqring_offsets`/`io_cqring_offsets` are 40 bytes each.
- [ ] AC-10: `io_uring_buf_ring` first 16 bytes alias `io_uring_buf` exactly; `tail` field at offset 14.
- [ ] AC-11: `io_uring_probe_op` is 8 bytes; `io_uring_probe` header is 16 bytes (last_op + ops_len + resv + resv2[3]).
- [ ] AC-12: `io_uring_napi` is 16 bytes; `io_uring_buf_status` is 40 bytes.
- [ ] AC-13: `io_uring_sync_cancel_reg` is 64 bytes (8 + 4 + 4 + 16 + 1 + 7 + 24).
- [ ] AC-14: `io_uring_reg_wait` total 64 bytes.
- [ ] AC-15: Reserved fields rejected when nonzero on input syscall paths.
- [ ] AC-16: `IORING_SETUP_SQE128` + `IORING_SETUP_CQE32` may be set together.
- [ ] AC-17: `IORING_SETUP_SQ_REWIND` requires `IORING_SETUP_NO_SQARRAY` and is incompatible with `IORING_SETUP_SQPOLL`.
- [ ] AC-18: liburing test suite passes against the Rookery ABI with no source modifications.

## Architecture

Rust struct layout uses `#[repr(C)]` and unions modeled with `MaybeUninit<[u8; N]>` plus accessor helpers, since Rust does not support arbitrary anonymous C unions natively.

```
#[repr(C)]
pub struct IoUringSqe {
    pub opcode: u8,
    pub flags: u8,
    pub ioprio: u16,
    pub fd: i32,
    pub off_addr2_cmd: u64,                 // union { off | addr2 | {cmd_op:u32, __pad1:u32} }
    pub addr_splice_off_in_levelopt: u64,   // union { addr | splice_off_in | {level:u32, optname:u32} }
    pub len: u32,
    pub op_flags: u32,                      // union of all per-op flag fields
    pub user_data: u64,
    // packed 16-bit pair
    pub buf_index_or_group: u16,            // union { buf_index | buf_group }
    pub personality: u16,
    pub aux: u32,                           // union { splice_fd_in | file_index | zcrx_ifq_idx | optlen | {addr_len,__pad3} | {write_stream,__pad4[3]} }
    pub trailer: [u64; 2],                  // union { {addr3,__pad2[1]} | {attr_ptr,attr_type_mask} | optval | cmd[0..16] }
}
```

```
#[repr(C)]
pub struct IoUringSqe128 {
    pub head: IoUringSqe,
    pub cmd_tail: [u8; 64],                 // extends cmd[] to 80 bytes total
}
```

```
#[repr(C)]
pub struct IoUringCqe {
    pub user_data: u64,
    pub res: i32,
    pub flags: u32,
}

#[repr(C)]
pub struct IoUringCqe32 {
    pub head: IoUringCqe,
    pub big_cqe: [u64; 2],
}
```

```
#[repr(C)]
pub struct IoSqringOffsets {
    pub head: u32, pub tail: u32, pub ring_mask: u32, pub ring_entries: u32,
    pub flags: u32, pub dropped: u32, pub array: u32, pub resv1: u32,
    pub user_addr: u64,
}

#[repr(C)]
pub struct IoCqringOffsets {
    pub head: u32, pub tail: u32, pub ring_mask: u32, pub ring_entries: u32,
    pub overflow: u32, pub cqes: u32, pub flags: u32, pub resv1: u32,
    pub user_addr: u64,
}

#[repr(C)]
pub struct IoUringParams {
    pub sq_entries: u32,
    pub cq_entries: u32,
    pub flags: u32,
    pub sq_thread_cpu: u32,
    pub sq_thread_idle: u32,
    pub features: u32,
    pub wq_fd: u32,
    pub resv: [u32; 3],
    pub sq_off: IoSqringOffsets,
    pub cq_off: IoCqringOffsets,
}
```

```
#[repr(C)]
pub struct IoUringProbeOp { pub op: u8, pub resv: u8, pub flags: u16, pub resv2: u32 }

#[repr(C)]
pub struct IoUringProbeHdr {
    pub last_op: u8, pub ops_len: u8, pub resv: u16, pub resv2: [u32; 3],
    // followed by ops_len * IoUringProbeOp
}
```

```
#[repr(C)]
pub struct IoUringRsrcRegister {
    pub nr: u32, pub flags: u32, pub resv2: u64,
    pub data: u64,  /* __aligned_u64 */
    pub tags: u64,
}

#[repr(C)]
pub struct IoUringRsrcUpdate {
    pub offset: u32, pub resv: u32, pub data: u64,
}

#[repr(C)]
pub struct IoUringRsrcUpdate2 {
    pub offset: u32, pub resv: u32,
    pub data: u64, pub tags: u64,
    pub nr: u32, pub resv2: u32,
}
```

```
#[repr(C)]
pub struct IoUringBuf { pub addr: u64, pub len: u32, pub bid: u16, pub resv: u16 }

#[repr(C)]
pub struct IoUringBufRingHdr {
    pub resv1: u64,
    pub resv2: u32,
    pub resv3: u16,
    pub tail: u16,
    // bufs follow as flexible array
}

#[repr(C)]
pub struct IoUringBufReg {
    pub ring_addr: u64,
    pub ring_entries: u32,
    pub bgid: u16,
    pub flags: u16,
    pub min_left: u32,
    pub resv: [u32; 5],
}

#[repr(C)]
pub struct IoUringBufStatus { pub buf_group: u32, pub head: u32, pub resv: [u32; 8] }
```

```
#[repr(C)]
pub struct IoUringNapi {
    pub busy_poll_to: u32,
    pub prefer_busy_poll: u8,
    pub opcode: u8,                         // io_uring_napi_op
    pub pad: [u8; 2],
    pub op_param: u32,
    pub resv: u32,
}
```

```
#[repr(C)]
pub struct IoUringRegWait {
    pub ts: KernelTimespec,
    pub min_wait_usec: u32,
    pub flags: u32,
    pub sigmask: u64,
    pub sigmask_sz: u32,
    pub pad: [u32; 3],
    pub pad2: [u64; 2],
}

#[repr(C)]
pub struct IoUringGeteventsArg {
    pub sigmask: u64,
    pub sigmask_sz: u32,
    pub min_wait_usec: u32,
    pub ts: u64,
}
```

```
#[repr(C)]
pub struct IoUringSyncCancelReg {
    pub addr: u64,
    pub fd: i32,
    pub flags: u32,
    pub timeout: KernelTimespec,
    pub opcode: u8,
    pub pad: [u8; 7],
    pub pad2: [u64; 3],
}

#[repr(C)]
pub struct IoUringFileIndexRange { pub off: u32, pub len: u32, pub resv: u64 }
#[repr(C)]
pub struct IoUringRecvmsgOut { pub namelen: u32, pub controllen: u32, pub payloadlen: u32, pub flags: u32 }
#[repr(C)]
pub struct IoUringRegionDesc { pub user_addr: u64, pub size: u64, pub flags: u32, pub id: u32, pub mmap_offset: u64, pub __resv: [u64; 4] }
#[repr(C)]
pub struct IoUringMemRegionReg { pub region_uptr: u64, pub flags: u64, pub __resv: [u64; 2] }
#[repr(C)]
pub struct IoUringClockRegister { pub clockid: u32, pub __resv: [u32; 3] }
#[repr(C)]
pub struct IoUringCloneBuffers { pub src_fd: u32, pub flags: u32, pub src_off: u32, pub dst_off: u32, pub nr: u32, pub pad: [u32; 3] }
#[repr(C)]
pub struct IoTimespec { pub tv_sec: u64, pub tv_nsec: u64 }
#[repr(C)]
pub struct IoUringFilesUpdate { pub offset: u32, pub resv: u32, pub fds: u64 /* __aligned_u64 */ }
#[repr(C)]
pub struct IoUringAttrPi { pub flags: u16, pub app_tag: u16, pub len: u32, pub addr: u64, pub seed: u64, pub rsvd: u64 }
#[repr(C)]
pub struct IoUringRestriction { pub opcode: u16, pub sub: u8, pub resv: u8, pub resv2: [u32; 3] }
```

Per-syscall surface (signatures only; semantics belong to Tier-3 docs):

- `io_uring_setup(entries: u32, params: *mut IoUringParams) -> i32`
- `io_uring_enter(fd: i32, to_submit: u32, min_complete: u32, flags: u32, arg: *const c_void, argsz: usize) -> i32`
- `io_uring_register(fd: i32, opcode: u32, arg: *const c_void, nr_args: u32) -> i32`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sqe_size_16_or_128` | INVARIANT | `sizeof(IoUringSqe) == 64` and `sizeof(IoUringSqe128) == 128`. |
| `cqe_size_16_or_32` | INVARIANT | `sizeof(IoUringCqe) == 16` and `sizeof(IoUringCqe32) == 32`. |
| `params_size_120` | INVARIANT | `sizeof(IoUringParams) == 120`; `sq_off` at 32; `cq_off` at 72. |
| `sqring_offsets_40` | INVARIANT | `sizeof(IoSqringOffsets) == sizeof(IoCqringOffsets) == 40`. |
| `bufring_head_aliases_buf` | INVARIANT | The first 16 bytes of `IoUringBufRingHdr` exactly alias `IoUringBuf`; `tail` at offset 14. |
| `setup_sq_rewind_excludes_sqpoll` | INVARIANT | `SQ_REWIND` ⟹ `NO_SQARRAY` ∧ ¬`SQPOLL`. |
| `register_use_registered_ring_bit` | INVARIANT | bit 31 of register opcode argument is reserved for ring-fd indirection. |
| `file_index_alloc_sentinel` | INVARIANT | `IORING_FILE_INDEX_ALLOC == u32::MAX`. |
| `register_files_skip_sentinel` | INVARIANT | `IORING_REGISTER_FILES_SKIP == -2_i32`. |
| `reserved_zero_on_input` | INVARIANT | All `resv*`, `__pad*`, `__resv*` bytes are zero on syscall input. |

### Layer 2: TLA+

`uapi/io_uring.tla`:
- Models the SQ/CQ ring transitions: `Submit`, `Complete`, `EventfdNotify`, `WakeupSQPoll`, `Drain`, `LinkSucceed`, `LinkBreak`, `Cancel`, `Timeout`.
- Properties:
  - `safety_sq_head_monotonic` — SQ head never decreases.
  - `safety_cq_tail_monotonic` — CQ tail never decreases.
  - `safety_cqe_skip_only_on_success` — `IOSQE_CQE_SKIP_SUCCESS` suppresses CQE only when res >= 0.
  - `safety_drain_orders_after_inflight` — `IOSQE_IO_DRAIN` issues only after all earlier SQEs complete.
  - `safety_io_link_chain_breaks_on_err` — `IO_LINK` breaks on error; `IO_HARDLINK` does not.
  - `safety_setup_flag_compat` — `SQ_REWIND ⟹ NO_SQARRAY ∧ ¬SQPOLL`; `IOPOLL` excludes `SQPOLL` on non-blockdev; `CQE_MIXED ⟹ CQE32 not required`.
  - `liveness_submit_completes` — every accepted SQE eventually produces a CQE (or is skipped per CQE_SKIP_SUCCESS).
  - `safety_buf_ring_tail_owned_by_user` — kernel never writes to `buf_ring->tail`; only userspace advances it.
  - `safety_mmap_offset_disjoint` — SQ/CQ/SQES/PBUF mmap offsets do not overlap.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoUringSqe::new_zero()` post: every field is zero | `IoUringSqe` |
| `IoUringParams::validate()` post: reserved fields zero; flags ∈ known set | `IoUringParams` |
| `IoUringRsrcRegister` post: `flags & RSRC_REGISTER_SPARSE` ⟹ `data == 0` | `IoUringRsrcRegister` |
| `IoUringBufReg::validate()` post: `ring_entries` is power-of-two, ≤ 32768 | `IoUringBufReg` |
| `IoUringProbe::for_each_op()` post: returned ops length == `ops_len` | `IoUringProbe` |
| `IoUringNapi::validate()` post: `opcode ∈ {REGISTER_OP, STATIC_ADD_ID, STATIC_DEL_ID}` | `IoUringNapi` |
| `IoUringSyncCancelReg::validate()` post: flags subset of `IORING_ASYNC_CANCEL_*` mask | `IoUringSyncCancelReg` |

### Layer 4: Verus/Creusot functional

`Per-binary-equivalence with liburing 7.x headers`:
- Compile-time `static_assert` (Rust `const _: () = assert!(...)`) parity on every struct size and field offset.
- Cross-check by parsing the Linux headers via `bindgen` and diffing against the Rookery Rust definitions in CI; fail PR if any byte offset differs from the baseline at commit `27a26ccfd528da725a999ea1e3102503c61eb655`.
- Round-trip property: a randomly populated `IoUringSqe` serialized via `core::ptr::copy_nonoverlapping` to a `[u8; 64]` buffer and deserialized back round-trips bit-for-bit.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening once that overview exists.)

io_uring UAPI reinforcement:

- **Per-`#[repr(C)]` exact layout** — defense against per-ABI drift between compilers.
- **Per-`__aligned_u64` u64-alignment of userspace pointer slots** — defense against per-arch misalignment trap.
- **Per-reserved-bytes-must-be-zero on input** — defense against per-flag-squat attempt by old userspace.
- **Per-`copy_from_user` for sqe contents** — defense against per-TOCTOU read of shared memory; sqe is *always* copied into a kernel-only `io_kiocb` snapshot before validation.
- **Per-mmap offset masking with `IORING_OFF_MMAP_MASK`** — defense against per-confused mmap region.
- **Per-`IORING_SETUP_SINGLE_ISSUER` enforcement** — defense against per-task-misuse races on submission.
- **Per-`IORING_SETUP_R_DISABLED` + `IORING_REGISTER_ENABLE_RINGS`** — defense against per-pre-restriction submission window.
- **Per-restriction set sealed before enable** — defense against per-bypass after enable.
- **Per-personality credential cookie** — defense against per-credential-leak across rings.
- **Per-`IOSQE_CQE_SKIP_SUCCESS` rejected if reserved bits set** — defense against per-flag-collision in future ABI additions.
- **Per-`IORING_OP_LAST` bound** — defense against per-out-of-range opcode dispatch.
- **Per-bgid scoped per ring** — defense against per-cross-ring buffer-id confusion.
- **Per-`buf_ring->tail` userspace-owned** — defense against per-kernel-write into user-managed cursor.
- **Per-`fixed_files` and `fixed_buffers` slot tagging** — defense against per-stale-resource use after replace.
- **Per-`IORING_SETUP_NO_MMAP` user_addr validated** — defense against per-bogus-user-pointer ring backing.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** — every read of `struct io_uring_sqe` from the SQE ring and every copy of `io_uring_params`/`io_uring_register` argument structs goes through hardened `copy_from_user`/`copy_to_user` paths that refuse kernel-pointer aliases and SMAP-violating accesses; the SQE shared-memory mapping is treated as untrusted.
- **PaX USERCOPY whitelist** — `io_uring_buf_ring` and pinned fixed-buffer/fixed-file metadata live in dedicated slabs whose `usercopy` whitelists allow only the exact ring-entry struct bytes to be copied in/out.
- **GRKERNSEC_HIDESYM** — `cqe->res` and probe replies never embed kernel pointers or KASLR-relative addresses; opcode and feature numbering is the only side-channel into kernel layout, and feature gating is read-only.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/io_uring` (if exposed) is restricted to the owning process under grsec; `IORING_REGISTER_QUERY` returns scrubbed information (no pointers, no PID leaks across user namespaces).
- **PAX_RANDKSTACK** — `io_uring_setup`, `io_uring_enter`, and `io_uring_register` are syscall entry points and benefit from per-syscall kernel-stack randomization; SQE batch processing must not anchor pointers into the syscall stack across batches.
- **PAX_REFCOUNT** — every refcount derived from userland-provided counts (`nr` in `io_uring_rsrc_register`, `ring_entries` in `io_uring_buf_reg`, `ops_len` in `io_uring_probe`, `count` in `io_uring_clone_buffers`) goes through `refcount_t`/`atomic_t` types with PAX-style overflow trapping.
- **GRKERNSEC_NO_SIMULT_CONNECT** — applies to `IORING_OP_CONNECT` and `IORING_OP_BIND` paths: per-task rate limiting and connect-while-listening prohibitions mirror the network stack hardening, since io_uring acts as a multiplexer.
- **PAX_KERNEXEC W^X** — `IORING_REGISTER_BPF_FILTER` JIT'd filters live in non-writable, executable-only pages once finalized; the JIT image is built in a writable scratch region and then sealed (`set_memory_ro` + `set_memory_x`) before being made reachable.
- **GRKERNSEC_BPF_HARDEN** — `IORING_REGISTER_BPF_FILTER` is gated on the same capability set as standalone `bpf(2)` (`CAP_BPF`), not the io_uring opener's credentials alone; programs are subject to the same verifier and signature policies.
- **CAP_SYS_NICE / CAP_SYS_RESOURCE** — `IORING_REGISTER_IOWQ_MAX_WORKERS` and `IORING_REGISTER_IOWQ_AFF` honor `CAP_SYS_NICE`/`CAP_SYS_RESOURCE` rather than letting any ring opener exceed system-wide worker limits.
- **GRKERNSEC_DMESG-style logging** — overflow on SQ submission, restriction violation, and probe rejection paths use rate-limited, sanitized logging so that `dmesg` does not leak pointer values or PIDs across user namespaces.
- **PaX kernel-pointer scrubbing for `cqe->user_data`** — `user_data` is a userspace cookie; the kernel never reuses an internal pointer as `user_data`, eliminating a class of pointer-leak via completion replay.
- **PAX_USERCOPY size-cap on register args** — `nr_args` for every `IORING_REGISTER_*` opcode is capped by a small constant (`IORING_MAX_REG_BUFFERS`, `IORING_MAX_RING_FDS`, `IORING_MAX_RESTRICTIONS`) verified before any allocation.
- **GRKERNSEC_KMEM-style** — `IORING_SETUP_NO_MMAP` user_addr regions are pinned with `get_user_pages` and unpinned on teardown; no raw user pointer is dereferenced post-fork.
- **PaX SMAP/SMEP** — every io_uring code path that touches user memory does so with SMAP toggled only inside short `copy_from_user`/`copy_to_user` windows; SMEP forbids the SQ shared mapping from being executed.

## Open Questions

- Whether to expose `IORING_REGISTER_BPF_FILTER` at all in the GPL-clean Rookery build, given the BPF subsystem is itself a major boundary.
- Treatment of `IORING_REGISTER_ZCRX_IFQ` and `IORING_REGISTER_ZCRX_CTRL`: these reach into `linux/io_uring/zcrx.h` which is a separate UAPI surface; covered in a follow-up Tier-5 once the zcrx subsystem design lands.
- Whether `IORING_SETUP_SQE_MIXED` should require a corresponding capability for unprivileged users (it can be abused to confuse pre-mixed-mode userspace).
- Whether to publish `IORING_REGISTER_QUERY` reply shapes in this header or factor into `linux/io_uring/query.h` only.

## Out of Scope

- `io_uring_setup(2)` / `io_uring_enter(2)` / `io_uring_register(2)` syscall implementation (Tier-3 in `io_uring/setup.md`, `io_uring/enter.md`, `io_uring/register.md`).
- Per-opcode handler bodies in `io_uring/rw.md`, `io_uring/net.md`, `io_uring/cancel.md`, etc.
- BPF verifier interactions for `IORING_REGISTER_BPF_FILTER` (covered in `bpf/verifier.md`).
- ZCRX subsystem (covered in `net/zcrx.md` / its own Tier-5 once headerized).
- Implementation code.
