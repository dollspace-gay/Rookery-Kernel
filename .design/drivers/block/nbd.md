# Tier-3: drivers/block/nbd.c — Network Block Device (TCP/UNIX socket, netlink genl, multi-connection)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/block/00-overview.md
upstream-paths:
  - drivers/block/nbd.c
  - include/uapi/linux/nbd.h
  - include/uapi/linux/nbd-netlink.h
-->

## Summary

The Network Block Device (NBD) driver exposes a remote block device over TCP (or UNIX domain) sockets — userspace runs `nbd-client` (or a netlink-aware tool such as `nbdctl`) which connects to a remote `nbd-server`, opens `/dev/nbdN`, passes the socket FD into the kernel, and the kernel then translates block layer requests into `struct nbd_request`/`struct nbd_reply` packets on the wire per the NBD protocol (see `Documentation/admin-guide/blockdev/nbd.rst` and `https://github.com/NetworkBlockDevice/nbd/blob/master/doc/proto.md`). Multi-connection support (NBD_FLAG_CAN_MULTI_CONN) lets the kernel use multiple sockets in parallel for higher throughput; per-socket failure triggers reconnect-and-requeue via `nbd_dead_link_work` and the `dead_conn_timeout`. The driver has both a legacy ioctl interface (`NBD_SET_SOCK`, `NBD_SET_BLKSIZE`, `NBD_SET_SIZE`, `NBD_DO_IT`, `NBD_DISCONNECT`, `NBD_SET_TIMEOUT`, `NBD_SET_FLAGS`) and a modern netlink-genl interface (`nbd_genl_connect`, `nbd_genl_disconnect`, `nbd_genl_reconfigure`, `nbd_genl_status`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nbd_device` | per-nbd state (tag set, index, refcounts, gendisk, recv workq, flags, pid, backend) | `drivers::block::nbd::Nbd` |
| `struct nbd_config` | per-connection config (flags, socks[], num_connections, blksize_bits, bytesize, dead_conn_timeout) | `drivers::block::nbd::Config` |
| `struct nbd_sock` | per-socket wrapper (sock, tx_lock, pending, sent, dead, cookie, work) | `drivers::block::nbd::Sock` |
| `struct nbd_cmd` | per-request driver state (in `request::pdu`) | `drivers::block::nbd::Cmd` |
| `nbd_fops` (`block_device_operations`) | `nbd_open`/`nbd_release`/`nbd_ioctl`/`nbd_compat_ioctl` | `drivers::block::nbd::FOPS` |
| `nbd_mq_ops` (`blk_mq_ops`) | `nbd_queue_rq` + `nbd_complete_rq` + `nbd_timeout` + `nbd_init_request` | `drivers::block::nbd::MQ_OPS` |
| `nbd_genl_family` (`genl_family`) | netlink family `"nbd"`, NBD_GENL_VERSION 1, ops {CONNECT,DISCONNECT,RECONFIGURE,STATUS} | `drivers::block::nbd::GENL` |
| `nbd_attr_policy` + `nbd_sock_policy` | nla policy for genl attributes | `drivers::block::nbd::Policy` |
| `NBD_SET_SOCK` (_IO(0xab,0)) | pass socket FD into kernel | `Nbd::set_sock` (legacy) |
| `NBD_SET_BLKSIZE` (_IO(0xab,1)) / `NBD_SET_SIZE` (_IO(0xab,2)) / `NBD_SET_SIZE_BLOCKS` (_IO(0xab,7)) | size config | `Nbd::set_blksize` / `_set_size` |
| `NBD_DO_IT` (_IO(0xab,3)) | blocking recv loop entry (legacy nbd-client model) | `Nbd::do_it` |
| `NBD_CLEAR_SOCK` (_IO(0xab,4)) / `NBD_CLEAR_QUE` (_IO(0xab,5)) | tear down | `Nbd::clear_sock` / `_clear_que` |
| `NBD_DISCONNECT` (_IO(0xab,8)) | send NBD_CMD_DISC | `Nbd::disconnect` |
| `NBD_SET_TIMEOUT` (_IO(0xab,9)) / `NBD_SET_FLAGS` (_IO(0xab,10)) | per-request timeout / server flags | `Nbd::set_timeout` / `_set_flags` |
| `nbd_genl_connect` / `_disconnect` / `_reconfigure` / `_status` | netlink genl handlers | `Genl::connect` / `_disconnect` / `_reconfigure` / `_status` |
| `nbd_send_cmd` / `nbd_handle_reply` / `recv_work` | TX path / RX path / per-socket recv worker | `Nbd::send_cmd` / `_handle_reply` / `_recv_work` |
| `nbd_xmit_timeout` | per-request timeout callback (reconnect-and-requeue) | `Nbd::timeout` |
| `nbd_disconnect_and_put` / `nbd_dead_link_work` | dead-link teardown / requeue | `Nbd::disconnect_and_put` / `_dead_link_work` |
| `nbds_max` (module param) | preallocated device count | `Nbd::nbds_max` |

## Compatibility contract

REQ-1: `NBD_MAJOR` (43) registered via `register_blkdev(NBD_MAJOR, "nbd")`; per-device minors are simply the IDR index shifted by `part_shift = fls(max_part)`.

REQ-2: Wire protocol per `include/uapi/linux/nbd.h`: every request frame is `struct nbd_request {__be32 magic; __be32 type; __be64 cookie; __be64 from; __be32 len}` (28 bytes packed) followed by an optional payload (writes); replies are `struct nbd_reply {__be32 magic; __be32 error; __be64 cookie}` (16 bytes) followed by an optional payload (reads). Magics are `NBD_REQUEST_MAGIC = 0x25609513` and `NBD_REPLY_MAGIC = 0x67446698`.

REQ-3: Command set per uapi: `NBD_CMD_READ`, `NBD_CMD_WRITE`, `NBD_CMD_DISC`, `NBD_CMD_FLUSH`, `NBD_CMD_TRIM`, `NBD_CMD_WRITE_ZEROES`; upper 16 bits of `type` are command flags (`NBD_CMD_FLAG_FUA`, `NBD_CMD_FLAG_NO_HOLE`).

REQ-4: Server flags negotiated through userspace and pushed via `NBD_SET_FLAGS`/`NBD_ATTR_SERVER_FLAGS`: `NBD_FLAG_HAS_FLAGS`, `NBD_FLAG_READ_ONLY`, `NBD_FLAG_SEND_FLUSH`, `NBD_FLAG_SEND_FUA`, `NBD_FLAG_ROTATIONAL`, `NBD_FLAG_SEND_TRIM`, `NBD_FLAG_SEND_WRITE_ZEROES`, `NBD_FLAG_CAN_MULTI_CONN`.

REQ-5: Legacy ioctl ABI (numbers `_IO(0xab, n)`, n ∈ {0..10}) preserved; netlink genl family `"nbd"` with policy from `nbd_attr_policy` (`NBD_ATTR_INDEX`, `NBD_ATTR_SIZE_BYTES`, `NBD_ATTR_BLOCK_SIZE_BYTES`, `NBD_ATTR_TIMEOUT`, `NBD_ATTR_SERVER_FLAGS`, `NBD_ATTR_CLIENT_FLAGS`, `NBD_ATTR_SOCKETS`, `NBD_ATTR_DEAD_CONN_TIMEOUT`, `NBD_ATTR_DEVICE_LIST`, `NBD_ATTR_BACKEND_IDENTIFIER`) — both paths fully supported.

REQ-6: Multi-connection (NBD_FLAG_CAN_MULTI_CONN): up to `num_connections` `nbd_sock`s nested in `config->socks[]`; `nbd_queue_rq` selects a socket via `nbd_handle_cmd` round-robin with fallback on dead links.

REQ-7: Reconnect: when a socket transitions to `nbd_sock::dead = true`, in-flight requests on that socket are re-queued via `nbd_requeue_cmd` and a `link_dead_args` work item is enqueued; userspace may attach a fresh socket via `NBD_ATTR_SOCKETS` in `nbd_genl_reconfigure`.

REQ-8: Per-device blk-mq tag set: `nr_hw_queues = config->num_connections`, `queue_depth = NBD_DEF_BLKSIZE_BITS-related default`, `cmd_size = sizeof(struct nbd_cmd)`, `flags = BLK_MQ_F_BLOCKING`.

REQ-9: Per-request cookie monotonically generated; replies looked up via blk-mq tag (`blk_mq_tag_to_rq`) cross-checked against `nbd_cmd::cmd_cookie` to defeat reply-mismatch and double-completion.

REQ-10: Sysfs: `/sys/block/nbdN/pid` (read-only — pid of attached nbd-client task), `/sys/block/nbdN/backend` (read-only — backend identifier from `NBD_ATTR_BACKEND_IDENTIFIER`).

REQ-11: debugfs (`CONFIG_DEBUG_FS`): per-nbd `flags`, `socks`, `pid`, `tasks` files under `/sys/kernel/debug/nbd/`.

REQ-12: NBD_CFLAG_DESTROY_ON_DISCONNECT and NBD_CFLAG_DISCONNECT_ON_CLOSE bits in `NBD_ATTR_CLIENT_FLAGS` control whether the device is deleted on disconnect, and whether close-by-last-opener triggers disconnect.

## Acceptance Criteria

- [ ] AC-1: `nbd-client <server> <port> /dev/nbd0` connects and exposes a usable block device.
- [ ] AC-2: `mkfs.ext4 /dev/nbd0 && mount /dev/nbd0 /mnt && fsck.ext4 /dev/nbd0` cycle works.
- [ ] AC-3: Multi-connection: `nbd-client -C 4 <server> /dev/nbd0` yields `nr_hw_queues == 4` in `/sys/block/nbd0/queue/nr_hw_queues`.
- [ ] AC-4: `NBD_DISCONNECT` ioctl or `nbd-client -d /dev/nbd0` cleanly tears down.
- [ ] AC-5: Netlink path via `nbdctl` (genl `connect`, `disconnect`, `status`) works against the same server.
- [ ] AC-6: Reconnect: drop one of multiple sockets (firewall + RST); in-flight requests requeue on surviving sockets without -EIO to userspace.
- [ ] AC-7: `blktests` block/nbd subset passes.

## Architecture

`nbd_device` keeps a list of sockets in `nbd_config::socks[]`; each `nbd_sock` carries its own `tx_lock` so multiple `queue_rq` CPUs can serialise sends on a chosen socket. Per request arriving via `nbd_queue_rq`: `nbd_handle_cmd` picks a live socket (round-robin from `current->index` rotated by retries), serialises on `tx_lock`, formats `struct nbd_request`, then `sock_xmit` writes both the header and any write payload via `kernel_sendmsg` with `MSG_NOSIGNAL`. After successful send the request is left in flight in the blk-mq tag set, awaiting a matching reply.

A per-socket `recv_thread_args` work item is queued onto `nbd->recv_workq` at connect time; the worker (`recv_work`) loops `nbd_handle_reply` → `sock_xmit(MSG_WAITALL)` to read `struct nbd_reply` → `blk_mq_tag_to_rq(tag = cookie & TAG_MASK)` → `nbd_cmd::cmd_cookie` match check → if WRITE/TRIM/FLUSH/WRITE_ZEROES: complete with `BLK_STS_OK`; if READ: read payload and complete; on protocol error: mark socket dead and trigger reconnect.

Lock hierarchy: `nbd_index_mutex` (global IDR) → `nbd->config_lock` (per-device config mutation) → `nsock->tx_lock` (per-socket TX serialisation) → `cmd->lock` (per-command state mutation: NBD_CMD_INFLIGHT, NBD_CMD_REQUEUED, NBD_CMD_PARTIAL_SEND bits). Refcounts: `nbd_device::refs` (lifetime), `nbd_device::config_refs` (per-config — when 0, config is freed); both are `refcount_t` so overflow traps.

Teardown: `nbd_genl_disconnect` or `NBD_DISCONNECT` ioctl sends `NBD_CMD_DISC`; the recv workers exit when sockets close; `nbd_clear_que` drains in-flight commands; `del_gendisk` and `blk_mq_free_tag_set` complete on the `remove_work` workqueue when refcounts drop to zero.

## Hardening

- Every `nbd_reply` validates `magic == NBD_REPLY_MAGIC` and `cookie` against the issuing `nbd_cmd::cmd_cookie` before completing; mismatches mark the socket dead.
- `nbd_request::len` upper-bounded by `BLK_DEF_MAX_SECTORS` and `request_queue::limits.max_hw_sectors`.
- `sock_xmit` uses `MSG_WAITALL | MSG_NOSIGNAL` on RX and `MSG_NOSIGNAL` on TX; partial sends tracked via `nbd_sock::sent` and resumed (NBD_CMD_PARTIAL_SEND).
- Socket FDs passed via `NBD_SET_SOCK`/`NBD_ATTR_SOCKETS` validated via `sockfd_lookup` + family check (TCP/UNIX) + state check (SS_CONNECTED).
- `dead_conn_timeout` per-config cap on how long a dead link will keep in-flight requests pending before EIO; default 0 (no timeout, requeue forever).
- `NBD_DO_IT` legacy ioctl is the recv-loop entry point — it blocks the calling task until disconnect; documented and bounded by `signal_pending(current)` checks.
- `nbd_clear_que` walks all in-flight tags and completes them with `BLK_STS_IOERR` before tag set free.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist `struct nbd_request`/`nbd_reply` (kernel-internal, never copy_to_user'd), nla attrs (validated via `nbd_attr_policy`), and the per-cmd scratch state; refuse copy_from_user paths that touch any other slab.
- **PAX_KERNEXEC** — `nbd_fops`, `nbd_mq_ops`, `nbd_genl_family`, and `nbd_connect_genl_ops` placed in `__ro_after_init` kernel text.
- **PAX_RANDKSTACK** — randomize kernel stack across `nbd_ioctl`, `nbd_genl_connect/_disconnect/_reconfigure/_status`, `nbd_queue_rq`, `recv_work`, and `nbd_handle_reply` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `nbd_device::refs` and `nbd_device::config_refs`; overflow trap defeats concurrent connect/disconnect race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `nbd_sock`, `nbd_config`, `recv_thread_args`, `link_dead_args`, and `nbd_cmd` slabs so server identifiers, backend strings, and cookies do not bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every `nbd_ioctl`, genl `doit`, and sysfs store entry; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `nbd_fops` (`.open`, `.release`, `.ioctl`, `.compat_ioctl`), `nbd_mq_ops` (`.queue_rq`, `.complete`, `.timeout`, `.init_request`), and every `genl_small_ops::doit` (`nbd_genl_connect`, `_disconnect`, `_reconfigure`, `_status`) marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-nbd `nbd_device` pointer disclosure behind CAP_SYSLOG; suppress `%p` for backend strings in debugfs.
- **GRKERNSEC_DMESG** — restrict dead-link, protocol-magic-mismatch, and reconnect banners to CAP_SYSLOG so attackers cannot probe link state via dmesg.
- **CAP_SYS_ADMIN on every NBD_CMD entry** — every legacy ioctl (`NBD_SET_SOCK`, `NBD_SET_SIZE`, `NBD_DO_IT`, `NBD_DISCONNECT`, etc.) and every genl op (`NBD_CMD_CONNECT`, `_DISCONNECT`, `_RECONFIGURE`, `_STATUS`) gated by CAP_SYS_ADMIN in the device's user namespace; refuse unprivileged callers.
- **Socket-passed FD lifecycle** — `sockfd_lookup` validates family, type (SOCK_STREAM), and state (SS_CONNECTED); socket's net-namespace and credential check against caller's; on reconfigure, old socket reference released only after new socket is bound.
- **SIZE_OVERFLOW on server-supplied lengths** — `nbd_reply` errors and `nbd_request::len` checked against `request_queue::limits.max_hw_sectors`; refuse server-controlled `len` outside [0, INT_MAX] AND outside per-request expected payload.
- **Reply-cookie validation** — `nbd_handle_reply` verifies `blk_mq_tag_to_rq(tag)` is non-NULL AND `nbd_cmd::cmd_cookie` matches the upper 32 bits of the cookie; otherwise mark the socket dead.
- **Per-socket TX serialisation** — `nsock->tx_lock` mutex prevents interleaved frame writes; refuse `partial send` past `NBD_CMD_PARTIAL_SEND` retry budget.
- **dead_conn_timeout enforcement** — bounded reconnect window prevents permanent userspace IO hang; defense against malicious server hanging on accept.
- **Magic-field hardening** — `nbd_request::magic` always set to `NBD_REQUEST_MAGIC`; replies with mismatched magic mark socket dead immediately.

Rationale: NBD is the rare "kernel block device backed entirely by user-supplied socket frames" — every reply is attacker-controlled if the server is hostile. A relaxed magic check, an unvalidated `len` field, a missed cookie comparison, a writable `nbd_mq_ops`, or a leaked socket reference turns a network connection into kernel block-layer corruption (or arbitrary read of kernel data via crafted READ replies). RAP/kCFI on every vtable, CAP_SYS_ADMIN on every config entry, refcount-overflow trap on config_refs, SIZE_OVERFLOW on `len`, magic/cookie validation on every reply, and bounded reconnect windows turn NBD from "trust the wire" into a structural enforcement boundary.

## Open Questions

- [ ] Q1: Should we deprecate the legacy `NBD_DO_IT` ioctl entirely and force migration to netlink genl, given the recv-loop blocking semantics complicate signal handling and core-dump capture?
- [ ] Q2: How aggressively should the kernel rate-limit reconnect attempts when `dead_conn_timeout == 0` (the historical default), to prevent a flapping link from DoSing the workqueue?
- [ ] Q3: Should `NBD_FLAG_CAN_MULTI_CONN` require the same socket family/protocol across all sockets (mixed TCP+UNIX is currently permitted)?

## Verification

- `nbd-client <host> <port> /dev/nbd0` then `cat /sys/block/nbd0/{pid,backend}`; verify pid points at running nbd-client.
- `lsblk` shows `/dev/nbd0` with correct major:minor and size from server.
- `blktests` block/nbd subset passes.
- `dmesg | grep -i nbd` after disconnect-on-RST and reconnect.
- `nbdctl status` lists all active devices with backend identifiers and live socket counts.
- ftrace `block:block_rq_issue` while `dd if=/dev/nbd0 of=/dev/null bs=1M count=1024`.
