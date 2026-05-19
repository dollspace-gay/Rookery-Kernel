# Tier-3: net/9p/trans_fd.c — 9P fd transport (TCP / Unix-socket / pipe)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/9p/00-overview.md
upstream-paths:
  - net/9p/trans_fd.c (~1095 lines)
  - include/net/9p/transport.h
  - include/net/9p/client.h
  - include/net/9p/9p.h
  - include/linux/socket.h (kernel_sendmsg / kernel_recvmsg)
  - include/linux/poll.h (poll_table, vfs_poll)
  - include/linux/workqueue.h
-->

## Summary

The **fd transport** binds a `struct p9_client` to a pair of kernel file-descriptors (one for read, one for write — usually the same socket, but can be two ends of a pipe). It exposes three `p9_trans_module` vtables: `"tcp"` (`p9_fd_create_tcp` — opens an AF_INET/AF_INET6 stream socket and connects to `addr:port`), `"unix"` (`p9_fd_create_unix` — AF_UNIX SOCK_STREAM to a filesystem path), `"fd"` (`p9_fd_create` — caller-supplied rfd/wfd, used for stdio / pipe / pre-connected sockets). Per-connection state lives in `struct p9_trans_fd { rd, wr, conn }` where `conn: struct p9_conn` carries the I/O state-machine: two `work_struct`s (`rq` / `wq`) that pump bytes via `kernel_read` / `kernel_write` (these wrap `vfs_read` / `vfs_write` with `KERNEL_DS`-style buffers), a header-decode buffer `tmp_buf[P9_HDRSZ]`, and the `req_list` / `unsent_req_list` queues. Per-poll-trigger: when probe sees `EPOLLIN`/`EPOLLOUT` on either fd, it schedules `p9_read_work` / `p9_write_work` on the system workqueue. The `p9_pollwake` waker registers via `p9_pollwait` callback installed in a poll_table; when the fd signals readability, it pushes the conn onto a global `p9_poll_pending_list` and schedules `p9_poll_workfn`, which calls `p9_poll_mux` for each pending conn to redrive the work-scheduling. Per-read state: `p9_read_work` first reads `P9_HDRSZ` bytes into `tmp_buf`, parses tag/size, looks up the matching `p9_req_t` via `p9_tag_lookup`, then reads the body straight into `rreq->rc.sdata`. Critical for: 9P-over-TCP for cross-machine mounts, 9P-over-unix-socket for VM gateways (`diod`, `9pfsd`), 9P-over-pipe for stdio-multiplexed services.

This Tier-3 covers `net/9p/trans_fd.c` (~1095 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct p9_trans_fd` | per-transport top-level | `P9TransFd` |
| `struct p9_conn` | per-connection mux/state | `P9Conn` |
| `struct p9_poll_wait` | per-poll-wait slot | `P9PollWait` |
| `enum { Rworksched, Rpending, Wworksched, Wpending }` | per-conn wsched bits | `WschedBits` |
| `p9_tcp_trans` | per-vtable "tcp" | `P9_TCP_TRANS` |
| `p9_unix_trans` | per-vtable "unix" | `P9_UNIX_TRANS` |
| `p9_fd_trans` | per-vtable "fd" | `P9_FD_TRANS` |
| `p9_fd_create_tcp()` | per-mount TCP create | `P9TransFd::create_tcp` |
| `p9_fd_create_unix()` | per-mount Unix-socket create | `P9TransFd::create_unix` |
| `p9_fd_create()` | per-mount rfd/wfd create | `P9TransFd::create` |
| `p9_fd_open()` | per-rfd+wfd dup-and-install | `P9TransFd::open` |
| `p9_socket_open()` | per-socket-→file wrap | `P9TransFd::socket_open` |
| `p9_fd_close()` | per-unmount close | `P9TransFd::close` |
| `p9_fd_request()` | per-T-message submit | `P9TransFd::request` |
| `p9_fd_cancel()` | per-pre-send cancel | `P9TransFd::cancel` |
| `p9_fd_cancelled()` | per-post-send cancel-ack | `P9TransFd::cancelled` |
| `p9_fd_poll()` | per-vfs_poll dispatch | `P9TransFd::poll` |
| `p9_fd_read()` / `p9_fd_write()` | per-byte-pump (kernel_read/write) | `P9TransFd::read` / `write` |
| `p9_read_work()` | per-read worker | `P9Conn::read_work` |
| `p9_write_work()` | per-write worker | `P9Conn::write_work` |
| `p9_pollwait()` | per-poll-table install | `P9Conn::pollwait` |
| `p9_pollwake()` | per-wake-queue callback | `P9Conn::pollwake` |
| `p9_poll_mux()` | per-conn poll-and-schedule | `P9Conn::poll_mux` |
| `p9_poll_workfn()` | per-global poll workqueue | `P9_POLL_WORKFN` |
| `p9_conn_create()` | per-conn init | `P9Conn::create` |
| `p9_conn_destroy()` | per-conn teardown | `P9Conn::destroy` |
| `p9_conn_cancel()` | per-conn error-propagation | `P9Conn::cancel` |
| `p9_mux_poll_stop()` | per-conn poll-deregister | `P9Conn::mux_poll_stop` |
| `p9_bind_privport()` | per-TCP privport-bind | `P9TransFd::bind_privport` |
| `p9_fd_show_options()` | per-/proc/mounts options | `P9TransFd::show_options` |
| `p9_poll_pending_list` (global) | per-conn pending-poll queue | `P9_POLL_PENDING_LIST` |
| `p9_poll_lock` (global spinlock) | per-list serialization | `P9_POLL_LOCK` |
| `p9_poll_work` (global work) | per-deferred-poll worker | `P9_POLL_WORK` |

## Compatibility contract

REQ-1: struct p9_trans_fd:
- rd: *file — reader handle (fget'd at open; fput at close).
- wr: *file — writer handle (may equal rd).
- conn: p9_conn — embedded connection mux state (not pointer; freed with parent).

REQ-2: struct p9_conn:
- mux_list: list_head — historical (unused at top level; kept for ABI).
- client: *p9_client — backref, cleared at destroy.
- err: int — sticky error code; once != 0, conn is dead.
- req_lock: spinlock_t — protects req_list, unsent_req_list, statuses, err.
- req_list: list_head — REQ_STATUS_SENT (in-flight) requests.
- unsent_req_list: list_head — REQ_STATUS_UNSENT awaiting write_work.
- rreq: *p9_req_t — currently-reading request (after header parse).
- wreq: *p9_req_t — currently-writing request.
- tmp_buf[P9_HDRSZ]: scratch for header pre-decode.
- rc: p9_fcall — read-frame state (sdata, offset, capacity, size, tag).
- wpos: int — bytes already written of current wreq.
- wsize: int — total bytes to write for current wreq.
- wbuf: *char — current write pointer (wreq->tc.sdata).
- poll_pending_link: list_head — link in P9_POLL_PENDING_LIST.
- poll_wait[MAXPOLLWADDR=2]: poll-wait slots (one per fd if rd != wr).
- pt: poll_table — initialized via init_poll_funcptr(&pt, p9_pollwait).
- rq: work_struct (p9_read_work).
- wq: work_struct (p9_write_work).
- wsched: ulong bitfield {Rworksched=1, Rpending=2, Wworksched=4, Wpending=8}.

REQ-3: struct p9_poll_wait:
- conn: *p9_conn.
- wait: wait_queue_entry_t — installed via add_wait_queue.
- wait_addr: *wait_queue_head_t — owner wq (fd's poll head); NULL ⟺ unused.

REQ-4: p9_trans_module instances:
- p9_tcp_trans: name="tcp", maxsize=MAX_SOCK_BUF (1MiB), create=p9_fd_create_tcp, close=p9_fd_close, request=p9_fd_request, cancel=p9_fd_cancel, cancelled=p9_fd_cancelled, show_options=p9_fd_show_options, supports_vmalloc=true, def=false, pooled_rbuffers=false.
- p9_unix_trans: name="unix", same vtable as tcp except create=p9_fd_create_unix, supports_vmalloc=true.
- p9_fd_trans: name="fd", same vtable, create=p9_fd_create, supports_vmalloc=true.

REQ-5: p9_fd_create_tcp(client, fc):
- addr = fc->source (NULL → -EINVAL).
- opts = fc->fs_private->fd_opts (parsed earlier).
- sprintf(port_str, "%u", opts.port).
- inet_pton_with_scope(net_ns, AF_UNSPEC, addr, port_str, &stor).
- client->trans_opts.tcp.port = opts.port; .privport = opts.privport.
- __sock_create(net_ns, stor.ss_family, SOCK_STREAM, IPPROTO_TCP, &csocket, 1 /* kern */).
- if opts.privport: p9_bind_privport(csocket).
- csocket->ops->connect(csocket, (sockaddr_unsized*)&stor, sizeof(stor), 0).
- return p9_socket_open(client, csocket).

REQ-6: p9_fd_create_unix(client, fc):
- addr = fc->source (NULL/empty → -EINVAL).
- if strlen(addr) >= UNIX_PATH_MAX: return -ENAMETOOLONG.
- sun_server.sun_family = PF_UNIX; strcpy(sun_server.sun_path, addr).
- __sock_create(net_ns, PF_UNIX, SOCK_STREAM, 0, &csocket, 1).
- csocket->ops->connect(csocket, (sockaddr_unsized*)&sun_server, sizeof(sockaddr_un) - 1, 0).
- return p9_socket_open(client, csocket).

REQ-7: p9_fd_create(client, fc):
- opts = fc->fs_private->fd_opts; rfd = opts.rfd; wfd = opts.wfd.
- if rfd == ~0 ∨ wfd == ~0: return -ENOPROTOOPT.
- p9_fd_open(client, rfd, wfd).
- p9_conn_create(client).

REQ-8: p9_fd_open(client, rfd, wfd):
- ts = kzalloc(sizeof *ts).
- ts->rd = fget(rfd); if !rd ∨ !(rd.f_mode & FMODE_READ): -EIO + cleanup.
- data_race(ts->rd->f_flags |= O_NONBLOCK).  /* KCSAN-annotated intentional */
- ts->wr = fget(wfd); if !wr ∨ !(wr.f_mode & FMODE_WRITE): -EIO + cleanup.
- data_race(ts->wr->f_flags |= O_NONBLOCK).
- client->trans = ts; client->status = Connected.
- /* Note: caller (p9_fd_create) follows up with p9_conn_create */.

REQ-9: p9_socket_open(client, csocket):
- p = kzalloc(sizeof *p).
- csocket->sk->sk_allocation = GFP_NOIO.
- csocket->sk->sk_use_task_frag = false.
- file = sock_alloc_file(csocket, 0, NULL).  /* socket → file wrap */
- if IS_ERR(file): kfree(p); return PTR_ERR.
- get_file(file).  /* extra ref so rd==wr safely fput'd twice */
- p->rd = p->wr = file.
- client->trans = p; client->status = Connected.
- p->rd->f_flags |= O_NONBLOCK.
- p9_conn_create(client).

REQ-10: p9_conn_create(client):
- ts = client->trans; m = &ts->conn.
- INIT_LIST_HEAD(&m->mux_list).
- m->client = client.
- spin_lock_init(&m->req_lock).
- INIT_LIST_HEAD(&m->req_list); INIT_LIST_HEAD(&m->unsent_req_list).
- INIT_WORK(&m->rq, p9_read_work).
- INIT_WORK(&m->wq, p9_write_work).
- INIT_LIST_HEAD(&m->poll_pending_link).
- init_poll_funcptr(&m->pt, p9_pollwait).
- /* Prime via vfs_poll — installs pollwait callbacks on rd's and (if distinct) wr's wait_queue_head */
- n = p9_fd_poll(client, &m->pt, NULL).
- if n & EPOLLIN: set_bit(Rpending, &m->wsched).
- if n & EPOLLOUT: set_bit(Wpending, &m->wsched).

REQ-11: p9_fd_poll(client, pt, err):
- ts = client->trans (NULL if not Connected → EPOLLERR + *err = -EREMOTEIO).
- ret = vfs_poll(ts->rd, pt).
- if ts->rd != ts->wr: ret = (ret & ~EPOLLOUT) | (vfs_poll(ts->wr, pt) & ~EPOLLIN).
- return ret.

REQ-12: p9_pollwait(filp, wait_address, p):
- m = container_of(p, struct p9_conn, pt).
- find first poll_wait[i] with wait_addr == NULL (max MAXPOLLWADDR=2 slots).
- if none free: log "not enough wait_address slots"; return.
- pwait->conn = m; pwait->wait_addr = wait_address.
- init_waitqueue_func_entry(&pwait->wait, p9_pollwake).
- add_wait_queue(wait_address, &pwait->wait).

REQ-13: p9_pollwake(wait, mode, sync, key) [waker, called from socket-layer waker]:
- pwait = container_of(wait, struct p9_poll_wait, wait).
- m = pwait->conn.
- spin_lock_irqsave(&p9_poll_lock, flags).
- if list_empty(&m->poll_pending_link): list_add_tail(&m->poll_pending_link, &p9_poll_pending_list).
- spin_unlock_irqrestore.
- schedule_work(&p9_poll_work).
- return 1.

REQ-14: p9_poll_workfn(work) [global deferred-poll worker]:
- while !list_empty(&p9_poll_pending_list):
  - conn = list_first_entry(...).
  - list_del_init(&conn->poll_pending_link).
  - drop spin; p9_poll_mux(conn); re-acquire spin.

REQ-15: p9_poll_mux(m):
- if m->err < 0: return.
- n = p9_fd_poll(m->client, NULL, &err).
- if n & (EPOLLERR | EPOLLHUP | EPOLLNVAL): p9_conn_cancel(m, err).
- if n & EPOLLIN:
  - set_bit(Rpending, &m->wsched).
  - if !test_and_set_bit(Rworksched, &m->wsched): schedule_work(&m->rq).
- if n & EPOLLOUT:
  - set_bit(Wpending, &m->wsched).
  - if (m->wsize || !list_empty(&unsent_req_list)) ∧ !test_and_set_bit(Wworksched, ...): schedule_work(&m->wq).

REQ-16: p9_fd_request(client, req):
- ts = client->trans; m = &ts->conn.
- spin_lock(&m->req_lock).
- err = READ_ONCE(m->err).
- if err < 0: spin_unlock; return err.
- WRITE_ONCE(req->status, REQ_STATUS_UNSENT).
- list_add_tail(&req->req_list, &m->unsent_req_list).
- spin_unlock.
- p9_poll_mux(m).  /* kick if writable */
- return 0.

REQ-17: p9_read_work(work):
- m = container_of(work, struct p9_conn, rq).
- if m->err < 0: return.
- /* First-time per-frame: read header into tmp_buf */
- if !m->rc.sdata:
  - m->rc.sdata = m->tmp_buf; m->rc.offset = 0; m->rc.capacity = P9_HDRSZ.
- clear_bit(Rpending, &m->wsched).
- err = p9_fd_read(m->client, m->rc.sdata + m->rc.offset, m->rc.capacity - m->rc.offset).
- if err == -EAGAIN: goto end_clear.
- if err <= 0: goto error.
- m->rc.offset += err.
- /* Header complete? */
- if !m->rreq ∧ m->rc.offset == m->rc.capacity:
  - m->rc.size = P9_HDRSZ.
  - p9_parse_header(&m->rc, &m->rc.size, NULL, NULL, 0).
  - m->rreq = p9_tag_lookup(m->client, m->rc.tag).
  - if !rreq ∨ rreq->status != REQ_STATUS_SENT: err = -EIO; goto error.
  - if m->rc.size > rreq->rc.capacity: err = -EIO; goto error.
  - if !rreq->rc.sdata: p9_req_put; m->rreq = NULL; err = -EIO; goto error.
  - /* Switch to body buffer */.
  - m->rc.sdata = rreq->rc.sdata.
  - memcpy(m->rc.sdata, m->tmp_buf, m->rc.capacity).
  - m->rc.capacity = m->rc.size.
- /* Body complete? */
- if m->rreq ∧ m->rc.offset == m->rc.capacity:
  - m->rreq->rc.size = m->rc.offset.
  - spin_lock(&m->req_lock).
  - if rreq->status == REQ_STATUS_SENT: list_del(&rreq->req_list); p9_client_cb(client, rreq, REQ_STATUS_RCVD).
  - else if status == REQ_STATUS_FLSHD: ignore (cancelled).
  - else: spin_unlock; err = -EIO; goto error.
  - spin_unlock.
  - m->rc.sdata = NULL; offset = 0; capacity = 0; p9_req_put; m->rreq = NULL.
- end_clear:
  - clear_bit(Rworksched, &m->wsched).
  - if !list_empty(&m->req_list):
    - n = test_and_clear_bit(Rpending, &m->wsched) ? EPOLLIN : p9_fd_poll(client, NULL, NULL).
    - if (n & EPOLLIN) ∧ !test_and_set_bit(Rworksched): schedule_work(&m->rq).
- error: p9_conn_cancel(m, err); clear_bit(Rworksched, ...).

REQ-18: p9_write_work(work):
- m = container_of(work, struct p9_conn, wq).
- if m->err < 0: clear Wworksched; return.
- if !m->wsize:
  - spin_lock(&m->req_lock).
  - if list_empty(&unsent_req_list): clear Wworksched; spin_unlock; return.
  - req = list_entry(unsent_req_list.next, ...).
  - WRITE_ONCE(req->status, REQ_STATUS_SENT).
  - list_move_tail(&req->req_list, &m->req_list).
  - m->wbuf = req->tc.sdata; m->wsize = req->tc.size; m->wpos = 0.
  - p9_req_get(req); m->wreq = req.
  - spin_unlock.
- clear_bit(Wpending, &m->wsched).
- err = p9_fd_write(client, m->wbuf + m->wpos, m->wsize - m->wpos).
- if err == -EAGAIN: goto end_clear.
- if err < 0: goto error.
- if err == 0: err = -EREMOTEIO; goto error.
- m->wpos += err.
- if m->wpos == m->wsize: wpos = wsize = 0; p9_req_put(wreq); m->wreq = NULL.
- end_clear:
  - clear_bit(Wworksched, ...).
  - if wsize ∨ !list_empty(&unsent_req_list):
    - n = test_and_clear_bit(Wpending, ...) ? EPOLLOUT : p9_fd_poll(client, NULL, NULL).
    - if (n & EPOLLOUT) ∧ !test_and_set_bit(Wworksched): schedule_work(&m->wq).
- error: p9_conn_cancel(m, err); clear Wworksched.

REQ-19: p9_fd_read(client, v, len):
- ts = client->trans (NULL if Disconnected → -EREMOTEIO).
- if !(ts->rd->f_flags & O_NONBLOCK): log warning (perf-critical: blocking read).
- pos = ts->rd->f_pos.
- ret = kernel_read(ts->rd, v, len, &pos).
- if ret <= 0 ∧ ret != -ERESTARTSYS ∧ ret != -EAGAIN: client->status = Disconnected.
- return ret.

REQ-20: p9_fd_write(client, v, len):
- ts = client->trans.
- ret = kernel_write(ts->wr, v, len, &ts->wr->f_pos).
- if ret <= 0 ∧ ret != -ERESTARTSYS ∧ ret != -EAGAIN: client->status = Disconnected.

REQ-21: p9_fd_cancel(client, req):
- if req->status == REQ_STATUS_UNSENT:
  - list_del(&req->req_list); WRITE_ONCE(req->status, REQ_STATUS_FLSHD); p9_req_put.
  - return 0 (cancelled).
- else return 1 (not cancellable; already sent).

REQ-22: p9_fd_cancelled(client, req):
- if req->status != REQ_STATUS_SENT: return 0 (already moved).
- list_del(&req->req_list); WRITE_ONCE(req->status, REQ_STATUS_FLSHD).
- p9_req_put(client, req).

REQ-23: p9_conn_cancel(m, err):
- spin_lock(&m->req_lock).
- if READ_ONCE(m->err) != 0: spin_unlock; return.  /* idempotent */
- WRITE_ONCE(m->err, err); ASSERT_EXCLUSIVE_WRITER(m->err).
- splice req_list + unsent_req_list into local cancel_list; mark each REQ_STATUS_ERROR.
- spin_unlock.
- for each req in cancel_list: req->t_err = req->t_err ?: err; p9_client_cb(client, req, REQ_STATUS_ERROR).

REQ-24: p9_mux_poll_stop(m):
- for i in 0..MAXPOLLWADDR:
  - if pwait[i].wait_addr: remove_wait_queue(wait_addr, &pwait[i].wait); wait_addr = NULL.
- spin_lock_irqsave(&p9_poll_lock, flags).
- list_del_init(&m->poll_pending_link).
- spin_unlock_irqrestore.
- flush_work(&p9_poll_work).

REQ-25: p9_conn_destroy(m):
- p9_mux_poll_stop(m).
- cancel_work_sync(&m->rq).
- if m->rreq: p9_req_put(client, rreq); rreq = NULL.
- cancel_work_sync(&m->wq).
- if m->wreq: p9_req_put(client, wreq); wreq = NULL.
- p9_conn_cancel(m, -ECONNRESET).
- m->client = NULL.

REQ-26: p9_fd_close(client):
- if !client ∨ !client->trans: return.
- client->status = Disconnected.
- p9_conn_destroy(&ts->conn).
- if ts->rd: fput(ts->rd).
- if ts->wr: fput(ts->wr).  /* If rd == wr, double-fput is balanced by get_file in socket_open */
- kfree(ts).

REQ-27: p9_bind_privport(sock) [opts.privport mount-option]:
- /* Try ports from p9_ipport_resv_max (1023) down to p9_ipport_resv_min (665) until non-EADDRINUSE */.
- stor.ss_family = sock->ops->family; INADDR_ANY/in6addr_any.
- for port = max ..= min:
  - set sin_port = htons(port).
  - err = kernel_bind(sock, &stor, sizeof(stor)).
  - if err != -EADDRINUSE: break.
- return err.

REQ-28: p9_fd_show_options(m, clnt):
- if clnt->trans_mod == &p9_tcp_trans ∧ port != P9_FD_PORT: seq_printf(",port=%u").
- if clnt->trans_mod == &p9_fd_trans:
  - if rfd != ~0: seq_printf(",rfd=%u").
  - if wfd != ~0: seq_printf(",wfd=%u").

REQ-29: Module init/exit:
- p9_trans_fd_init: v9fs_register_trans(p9_tcp_trans), p9_unix_trans, p9_fd_trans.
- p9_trans_fd_exit: flush_work(&p9_poll_work); v9fs_unregister_trans(...) for all three.

REQ-30: Constants:
- MAX_SOCK_BUF = 1 << 20 (1 MiB) — maxsize for all three vtables.
- MAXPOLLWADDR = 2 — per-conn poll-wait slots (rd and possibly wr).
- p9_ipport_resv_min / _max = P9_DEF_MIN_RESVPORT / _MAX_RESVPORT — privport range.

## Acceptance Criteria

- [ ] AC-1: mount -t 9p -o trans=tcp,port=564 1.2.3.4 /mnt: connects via __sock_create + connect.
- [ ] AC-2: mount -t 9p -o trans=unix /tmp/sock /mnt: connects via AF_UNIX SOCK_STREAM.
- [ ] AC-3: mount -t 9p -o trans=fd,rfd=3,wfd=4 /mnt: dups rfd/wfd into transport.
- [ ] AC-4: rfd/wfd lacking FMODE_READ/_WRITE: -EIO.
- [ ] AC-5: Address >= UNIX_PATH_MAX: -ENAMETOOLONG.
- [ ] AC-6: opts.privport=1: socket bound to port in [665, 1023] before connect.
- [ ] AC-7: p9_fd_request enqueues to unsent_req_list and kicks p9_poll_mux ⟹ p9_write_work runs.
- [ ] AC-8: Full T-frame written via kernel_write across multiple wakeups (wpos accumulates).
- [ ] AC-9: R-frame header arrives: p9_tag_lookup matches; body read directly into rreq->rc.sdata.
- [ ] AC-10: Tag mismatch (no rreq or rreq->status != SENT): p9_conn_cancel(-EIO) cancels all reqs.
- [ ] AC-11: kernel_read returns 0 or negative non-EAGAIN: client->status = Disconnected.
- [ ] AC-12: p9_fd_cancel on UNSENT req: succeeds (returns 0); on SENT req: returns 1.
- [ ] AC-13: p9_pollwake from socket wakes p9_poll_work which calls p9_poll_mux which schedules rq/wq.
- [ ] AC-14: MAXPOLLWADDR=2 exhausted (3rd vfs_poll source): logs "not enough wait_address slots".
- [ ] AC-15: p9_fd_close: cancels both works, drains rreq/wreq, conn_cancel(-ECONNRESET), fput both fds.

## Architecture

```
struct P9TransFd {
  rd: NonNull<File>,                  // fget'd; nonblocking
  wr: NonNull<File>,                  // fget'd; nonblocking; may equal rd
  conn: P9Conn,                       // embedded, freed with parent
}

struct P9Conn {
  mux_list: ListHead,
  client: AtomicPtr<P9Client>,
  err: AtomicI32,                     // sticky; 0 = healthy
  req_lock: SpinLock<()>,
  req_list: ListHead,                 // SENT
  unsent_req_list: ListHead,          // UNSENT
  rreq: AtomicPtr<P9Req>,
  wreq: AtomicPtr<P9Req>,
  tmp_buf: [u8; P9_HDRSZ],
  rc: P9Fcall,                        // read-side frame state
  wpos: i32,
  wsize: i32,
  wbuf: AtomicPtr<u8>,
  poll_pending_link: ListHead,
  poll_wait: [P9PollWait; MAXPOLLWADDR],
  pt: PollTable,
  rq: WorkStruct,                     // p9_read_work
  wq: WorkStruct,                     // p9_write_work
  wsched: AtomicUsize,                // {Rworksched, Rpending, Wworksched, Wpending}
}

struct P9PollWait {
  conn: NonNull<P9Conn>,
  wait: WaitQueueEntry,
  wait_addr: Option<NonNull<WaitQueueHead>>,
}

static P9_POLL_PENDING_LIST: SpinLock<ListHead> = ...;
static P9_POLL_WORK: WorkStruct = WorkStruct::new(p9_poll_workfn);
```

`P9TransFd::create_tcp(client, fc) -> Result<()>`:
1. addr = fc.source.ok_or(-EINVAL)?.
2. opts = (fc.fs_private as &V9fsContext).fd_opts.
3. let port_str = format_port(opts.port).
4. inet_pton_with_scope(current.nsproxy.net_ns, AF_UNSPEC, addr, &port_str, &mut stor)?.
5. client.trans_opts.tcp.port = opts.port; client.trans_opts.tcp.privport = opts.privport.
6. let csocket = __sock_create(net_ns, stor.ss_family, SOCK_STREAM, IPPROTO_TCP, 1 /* kern */)?.
7. if opts.privport: P9TransFd::bind_privport(&csocket)?.
8. csocket.ops.connect(&csocket, &stor as &SockaddrUnsized, sizeof(stor), 0)?.
9. P9TransFd::socket_open(client, csocket).

`P9TransFd::create_unix(client, fc) -> Result<()>`:
1. addr = fc.source.ok_or(-EINVAL)?.
2. if addr.len() >= UNIX_PATH_MAX: return Err(-ENAMETOOLONG).
3. sun_server = SockaddrUn { sun_family: PF_UNIX, sun_path: copy_into(addr) }.
4. csocket = __sock_create(net_ns, PF_UNIX, SOCK_STREAM, 0, 1)?.
5. csocket.ops.connect(&csocket, &sun_server, size_of::<SockaddrUn>() - 1, 0)?.
6. P9TransFd::socket_open(client, csocket).

`P9TransFd::create(client, fc) -> Result<()>`:
1. opts = (fc.fs_private as &V9fsContext).fd_opts.
2. client.trans_opts.fd = opts.into().
3. if opts.rfd == !0 ∨ opts.wfd == !0: return Err(-ENOPROTOOPT).
4. P9TransFd::open(client, opts.rfd, opts.wfd)?.
5. P9Conn::create(client).

`P9TransFd::open(client, rfd, wfd) -> Result<()>`:
1. ts = Box::try_new_zeroed(P9TransFd { .. })?.
2. ts.rd = fget(rfd).ok_or(-EIO)?.
3. require ts.rd.f_mode & FMODE_READ else -EIO.
4. unsafe { data_race(ts.rd.f_flags |= O_NONBLOCK) }.
5. ts.wr = fget(wfd).ok_or(-EIO)?.
6. require ts.wr.f_mode & FMODE_WRITE else -EIO.
7. unsafe { data_race(ts.wr.f_flags |= O_NONBLOCK) }.
8. client.trans = Box::into_raw(ts).
9. client.status = Connected.

`P9TransFd::socket_open(client, csocket) -> Result<()>`:
1. p = Box::try_new_zeroed(P9TransFd { .. })?.
2. csocket.sk.sk_allocation = GFP_NOIO.
3. csocket.sk.sk_use_task_frag = false.
4. let file = sock_alloc_file(csocket, 0, None).map_err(|e| { kfree(p); e })?.
5. get_file(file).  /* So rd == wr can be fput twice */
6. p.rd = file; p.wr = file.
7. client.trans = Box::into_raw(p); client.status = Connected.
8. p.rd.f_flags |= O_NONBLOCK.
9. P9Conn::create(client).

`P9Conn::create(client)`:
1. let ts = client.trans as &mut P9TransFd.
2. let m = &mut ts.conn.
3. INIT_LIST_HEAD(&mut m.mux_list).
4. m.client = client.
5. spin_lock_init(&m.req_lock).
6. INIT_LIST_HEAD(&mut m.req_list).
7. INIT_LIST_HEAD(&mut m.unsent_req_list).
8. INIT_WORK(&mut m.rq, p9_read_work).
9. INIT_WORK(&mut m.wq, p9_write_work).
10. INIT_LIST_HEAD(&mut m.poll_pending_link).
11. init_poll_funcptr(&mut m.pt, p9_pollwait).
12. let n = P9TransFd::poll(client, Some(&mut m.pt), None).
13. if n & EPOLLIN: m.wsched.fetch_or(Rpending).
14. if n & EPOLLOUT: m.wsched.fetch_or(Wpending).

`P9TransFd::request(client, req) -> Result<()>`:
1. let m = &(client.trans as &P9TransFd).conn.
2. spin_lock(&m.req_lock).
3. let err = m.err.load(); if err < 0: spin_unlock; return Err(err).
4. req.status.store(REQ_STATUS_UNSENT).
5. m.unsent_req_list.add_tail(&req.req_list).
6. spin_unlock.
7. P9Conn::poll_mux(m).
8. Ok(()).

`P9Conn::read_work(work)`:
1. m = container_of(work, P9Conn, rq).
2. if m.err.load() < 0: return.
3. if m.rc.sdata.is_null(): m.rc.sdata = m.tmp_buf.as_mut_ptr(); m.rc.offset = 0; m.rc.capacity = P9_HDRSZ.
4. m.wsched.fetch_and(!Rpending).
5. let err = P9TransFd::read(m.client, m.rc.sdata.add(m.rc.offset), m.rc.capacity - m.rc.offset).
6. if err == -EAGAIN: goto end_clear.
7. if err <= 0: goto error.
8. m.rc.offset += err as usize.
9. /* Header-complete transition */
10. if m.rreq.is_null() ∧ m.rc.offset == m.rc.capacity:
    - m.rc.size = P9_HDRSZ.
    - p9_parse_header(&mut m.rc, &mut m.rc.size, None, None, 0)?.
    - m.rreq = p9_tag_lookup(m.client, m.rc.tag).
    - require m.rreq ≠ null ∧ m.rreq.status == REQ_STATUS_SENT else err=-EIO; goto error.
    - require m.rc.size ≤ m.rreq.rc.capacity else err=-EIO; goto error.
    - require m.rreq.rc.sdata ≠ null else { p9_req_put(m.client, m.rreq); m.rreq=null; err=-EIO; goto error }.
    - m.rc.sdata = m.rreq.rc.sdata.
    - memcpy(m.rc.sdata, m.tmp_buf.as_ptr(), m.rc.capacity).
    - m.rc.capacity = m.rc.size.
11. /* Body-complete transition */
12. if m.rreq ≠ null ∧ m.rc.offset == m.rc.capacity:
    - m.rreq.rc.size = m.rc.offset.
    - spin_lock(&m.req_lock).
    - match m.rreq.status {
      - REQ_STATUS_SENT => { list_del(&m.rreq.req_list); p9_client_cb(m.client, m.rreq, REQ_STATUS_RCVD); }
      - REQ_STATUS_FLSHD => { /* drop reply */ }
      - _ => { spin_unlock; err=-EIO; goto error; }
    - }.
    - spin_unlock.
    - m.rc.sdata = null; m.rc.offset = 0; m.rc.capacity = 0; p9_req_put(m.client, m.rreq); m.rreq = null.
13. end_clear:
    - m.wsched.fetch_and(!Rworksched).
    - if !m.req_list.is_empty():
      - let n = if m.wsched.test_and_clear(Rpending) { EPOLLIN } else { P9TransFd::poll(m.client, None, None) }.
      - if (n & EPOLLIN) ∧ !m.wsched.test_and_set(Rworksched): schedule_work(&m.rq).
14. error: P9Conn::cancel(m, err); m.wsched.fetch_and(!Rworksched).

`P9Conn::write_work(work)`:
1. m = container_of(work, P9Conn, wq).
2. if m.err.load() < 0: m.wsched.fetch_and(!Wworksched); return.
3. if m.wsize == 0:
   - spin_lock(&m.req_lock).
   - if m.unsent_req_list.is_empty(): m.wsched.fetch_and(!Wworksched); spin_unlock; return.
   - let req = m.unsent_req_list.first_entry::<P9Req>(req_list).
   - req.status.store(REQ_STATUS_SENT).
   - list_move_tail(&req.req_list, &m.req_list).
   - m.wbuf = req.tc.sdata; m.wsize = req.tc.size; m.wpos = 0.
   - p9_req_get(req); m.wreq = req.
   - spin_unlock.
4. m.wsched.fetch_and(!Wpending).
5. let err = P9TransFd::write(m.client, m.wbuf.add(m.wpos), m.wsize - m.wpos).
6. if err == -EAGAIN: goto end_clear.
7. if err < 0: goto error.
8. if err == 0: err = -EREMOTEIO; goto error.
9. m.wpos += err.
10. if m.wpos == m.wsize: m.wpos = 0; m.wsize = 0; p9_req_put(m.client, m.wreq); m.wreq = null.
11. end_clear: re-schedule if work remains (mirror of read_work).
12. error: P9Conn::cancel(m, err); m.wsched.fetch_and(!Wworksched).

`P9Conn::pollwake(wait, mode, sync, key) -> i32`:
1. let pwait = container_of(wait, P9PollWait, wait).
2. let m = pwait.conn.
3. spin_lock_irqsave(&P9_POLL_LOCK, flags).
4. if m.poll_pending_link.is_empty(): P9_POLL_PENDING_LIST.add_tail(&m.poll_pending_link).
5. spin_unlock_irqrestore.
6. schedule_work(&P9_POLL_WORK).
7. return 1.

`P9Conn::poll_mux(m)`:
1. if m.err.load() < 0: return.
2. let mut err = -ECONNRESET; let n = P9TransFd::poll(m.client, None, Some(&mut err)).
3. if n & (EPOLLERR | EPOLLHUP | EPOLLNVAL): P9Conn::cancel(m, err).
4. if n & EPOLLIN:
   - m.wsched.fetch_or(Rpending).
   - if !m.wsched.test_and_set(Rworksched): schedule_work(&m.rq).
5. if n & EPOLLOUT:
   - m.wsched.fetch_or(Wpending).
   - if (m.wsize > 0 ∨ !m.unsent_req_list.is_empty()) ∧ !m.wsched.test_and_set(Wworksched): schedule_work(&m.wq).

`P9Conn::cancel(m, err)`:
1. spin_lock(&m.req_lock).
2. if m.err.load() != 0: spin_unlock; return.  /* idempotent */
3. m.err.store(err).
4. let mut cancel_list = ListHead::new().
5. for req in m.req_list.drain(): cancel_list.add_tail(&req.req_list); req.status.store(REQ_STATUS_ERROR).
6. for req in m.unsent_req_list.drain(): cancel_list.add_tail(&req.req_list); req.status.store(REQ_STATUS_ERROR).
7. spin_unlock.
8. for req in cancel_list.drain():
   - if req.t_err == 0: req.t_err = err.
   - p9_client_cb(m.client, req, REQ_STATUS_ERROR).

`P9Conn::destroy(m)`:
1. P9Conn::mux_poll_stop(m).
2. cancel_work_sync(&m.rq).
3. if m.rreq != null: p9_req_put(m.client, m.rreq); m.rreq = null.
4. cancel_work_sync(&m.wq).
5. if m.wreq != null: p9_req_put(m.client, m.wreq); m.wreq = null.
6. P9Conn::cancel(m, -ECONNRESET).
7. m.client = null.

`P9TransFd::close(client)`:
1. let ts = client.trans as *mut P9TransFd; if ts.is_null(): return.
2. client.status = Disconnected.
3. P9Conn::destroy(&mut ts.conn).
4. if ts.rd != null: fput(ts.rd).
5. if ts.wr != null: fput(ts.wr).
6. Box::from_raw(ts);  /* drop */.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `req_lock_protects_lists` | INVARIANT | per-req_list/unsent_req_list mutation: req_lock held. |
| `wsched_bits_serialized` | INVARIANT | per-Rworksched/Wworksched: test_and_set + clear_bit pair; never spuriously double-scheduled. |
| `pollwait_slot_bounded` | INVARIANT | per-pollwait: at most MAXPOLLWADDR=2 slots filled. |
| `header_then_body_atomicity` | INVARIANT | per-read_work: rc.sdata switches from tmp_buf to rreq->rc.sdata exactly once per frame. |
| `err_sticky` | INVARIANT | per-conn: m.err transitions 0 → e once; subsequent writes are no-ops. |
| `tag_lookup_status_sent` | INVARIANT | per-incoming-reply: rreq->status must == REQ_STATUS_SENT else conn-cancel. |
| `req_get_put_balanced` | INVARIANT | per-write_work: p9_req_get on dequeue is matched by p9_req_put on complete (or by conn_cancel path). |
| `rd_wr_fput_balanced` | INVARIANT | per-close: every fget at open is matched by exactly one fput; socket_open's extra get_file balances rd==wr. |
| `cancel_idempotent` | INVARIANT | per-conn_cancel: m.err != 0 ⟹ no further state change. |
| `unsent_drained_on_close` | INVARIANT | per-destroy: both req_list and unsent_req_list emptied to cancel_list. |

### Layer 2: TLA+

`net/9p/trans-fd.tla`:
- Per-conn states: Init → Connected → (Reading | Writing | Idle | Erroring) → Destroyed.
- Per-req states: Created → Unsent → Sent → (Rcvd | Flshd | Error).
- Properties:
  - `safety_no_concurrent_read_work` — Rworksched bit ⟺ at most one scheduled read_work for a given conn.
  - `safety_no_concurrent_write_work` — Wworksched bit ⟺ at most one scheduled write_work.
  - `safety_tag_match_or_cancel` — incoming frame: tag matches a SENT req ∨ p9_conn_cancel(-EIO).
  - `safety_err_terminal` — m.err != 0 ⟹ all reqs eventually REQ_STATUS_ERROR.
  - `liveness_unsent_eventually_sent` — req on unsent_req_list ∧ writable fd eventually transitions to req_list.
  - `liveness_sent_eventually_rcvd_or_error` — every SENT req receives a callback (RCVD or ERROR).
  - `liveness_poll_wake_dequeues` — p9_pollwake schedules p9_poll_workfn which eventually calls p9_poll_mux on the conn.
  - `safety_destroy_after_cancel_works` — cancel_work_sync(&rq) and (&wq) both return before fput / kfree.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `P9TransFd::create_tcp` post: client.status == Connected ∨ socket released | `create_tcp` |
| `P9TransFd::create_unix` post: same | `create_unix` |
| `P9TransFd::open` post: rd has FMODE_READ; wr has FMODE_WRITE | `open` |
| `P9TransFd::request` post: req on unsent_req_list ∧ poll_mux called | `request` |
| `P9Conn::read_work` post: every byte read; rreq drained on completion | `read_work` |
| `P9Conn::write_work` post: m.wpos ≤ m.wsize; on completion m.wreq dropped | `write_work` |
| `P9Conn::pollwake` post: conn in P9_POLL_PENDING_LIST exactly once | `pollwake` |
| `P9Conn::poll_mux` post: rq/wq scheduled iff (EPOLLIN/OUT ∧ !already-scheduled) | `poll_mux` |
| `P9Conn::cancel` post: m.err set; both lists drained; ERROR callbacks fired | `cancel` |
| `P9Conn::destroy` post: poll de-registered; works cancel-synced; rreq/wreq freed | `destroy` |
| `P9TransFd::close` post: fds fput'd; ts freed | `close` |

### Layer 4: Verus/Creusot functional

`create_* → socket_open → conn_create (vfs_poll installs pollwait) → request enqueue → poll_mux schedules wq → write_work pumps kernel_write → server replies → pollwake schedules global poll-work → poll_mux schedules rq → read_work parses header + body → p9_client_cb(REQ_STATUS_RCVD)` semantic equivalence: per `Documentation/filesystems/9p.rst`, RFC-9P (Plan 9 spec), `diod`/`nfs-ganesha` reference servers.

## Hardening

(Inherits row-1 features from `net/9p/00-overview.md` § Hardening.)

trans-fd reinforcement:

- **Per-O_NONBLOCK forced on rd/wr** — defense against per-blocking-syscall holding the work-queue worker indefinitely.
- **Per-data_race-annotated f_flags OR-in** — defense against KCSAN false-positive on the intentional concurrent flag flip.
- **Per-FMODE_READ / FMODE_WRITE check** — defense against per-misconfigured-fd writing into a read-only file.
- **Per-UNIX_PATH_MAX bound** — defense against per-stack-overflow in sockaddr_un.sun_path.
- **Per-MAX_SOCK_BUF (1 MiB) maxsize** — defense against per-9P-msize abuse.
- **Per-MAXPOLLWADDR (2) hard cap on poll slots** — defense against per-overflowing-poll-table install.
- **Per-Rworksched/Wworksched single-shot bits** — defense against per-double-scheduling on the workqueue.
- **Per-err sticky atomic** — defense against per-late-arriving-IO racing a cancel.
- **Per-tag-lookup status==SENT check** — defense against per-spoof-tag UAF on cancelled or already-replied reqs.
- **Per-rc.size ≤ rreq->rc.capacity bound** — defense against per-overlong-reply heap-overflow.
- **Per-cancel_work_sync before fput** — defense against per-worker-touching-freed-file.
- **Per-privport-range scan (665..1023)** — defense against per-NFS-style port-collision on locked-down servers.
- **Per-sk_allocation = GFP_NOIO** — defense against per-reentrant-allocation deadlock while serving an mm-pressure path.
- **Per-sk_use_task_frag = false** — defense against per-task-frag misuse from the worker thread.
- **Per-conn_cancel idempotent (err != 0 ⟹ noop)** — defense against per-double-cancel storm.
- **Per-flush_work on poll_work at module-exit** — defense against per-deferred-poll-after-unload.
- **Per-net_ns scoping on __sock_create** — defense against per-namespace-violation.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — trans-fd kernel_read/kernel_write iov_iter paths use bounded payload sizes (MAX_SOCK_BUF = 1 MiB); no usercopy whitelist covers the transport buffers, which stay in kernel.
- **PAX_KERNEXEC** — trans-fd workqueue handlers (p9_read_work, p9_write_work, p9_pollwake) execute from RX text; the poll-table install path uses statically-typed callbacks.
- **PAX_RANDKSTACK** — p9_fd_create_tcp/_unix and connect helpers honor randomized stack offset; sockaddr_un.sun_path uses UNIX_PATH_MAX with no VLA.
- **PAX_REFCOUNT** — p9_conn and p9_trans_fd refcounts, file-pointer holds, and req->refcount use hardened types; cancel_work_sync ordering prevents underflow into UAF.
- **PAX_MEMORY_SANITIZE** — freed p9_conn and freed p9_req_t payloads are sanitized before kfree so file-pointer and tag-slot state cannot recur via slab recycle.
- **PAX_UDEREF** — connect() path consumes user pathnames via copy_from_user with strict length cap; data plane reads/writes operate on kernel pages only.
- **PAX_RAP / kCFI** — p9_trans_module ops (.create_tcp, .create_unix, .create_fd, .close, .request, .cancel, .show_options) are CFI-typed; the trans-mod dispatch is non-pivotable.
- **GRKERNSEC_HIDESYM** — trans-fd /proc and debugfs surfaces hide file*, socket*, p9_conn* kernel addresses from non-CAP_SYSLOG readers.
- **GRKERNSEC_DMESG** — sticky-err logs, poll-table-full warnings, and module-exit drain timeouts gate behind dmesg_restrict.
- **Per-transport PAX_REFCOUNT on p9_conn lifetime** — defense against per-cancel-vs-IO race UAF.
- **Per-kernel_read/kernel_write bounded by MAX_SOCK_BUF** — defense against per-msize-abuse heap exhaustion.
- **Per-O_NONBLOCK forced on fd at attach** — defense against per-blocking-syscall pinning workqueue worker.
- **Per-cancel_work_sync before fput** — defense against per-worker-touching-freed-file UAF.
- **Per-net_ns scoping on __sock_create** — defense against per-namespace-violation reaching a host-net socket from a container.

Rationale: trans-fd is a transport that turns a userspace-provided fd or a kernel-created socket into a 9P pipe; its threat surface spans file lifetime, workqueue race windows, and bounded I/O sizing. Grsec compounding (REFCOUNT on p9_conn, kernel_read/write bounds, RAP/kCFI on trans_module ops, MEMORY_SANITIZE on free) ensures the transport cannot be turned into either a UAF primitive against a freed file* or an unbounded-allocation DoS via msize manipulation, and HIDESYM/DMESG suppress layout leakage through log messages a privileged operator might enable.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `net/9p/client.c` core RPC/tag/fid machinery (covered in `client.md` Tier-3).
- `net/9p/protocol.c` 9P2000.L marshalling (covered in `p9.md` Tier-3).
- `net/9p/trans_virtio.c` virtio transport (covered in `trans-virtio.md` Tier-3, this batch).
- `net/9p/trans_xen.c`, `trans_rdma.c`, `trans_usbg.c` (separate Tier-3s if expanded).
- `net/socket.c` core (`socket`, `bind`, `connect` syscall machinery) — covered under `net/socket.md` Tier-3.
- VFS read/write internals (kernel_read / kernel_write) — covered under `fs/read_write.c` Tier-3.
- Workqueue subsystem (`kernel/workqueue.c`) — separate Tier-3.
- `fs/9p/*` VFS bindings — separate Tier-3.
- Implementation code.
