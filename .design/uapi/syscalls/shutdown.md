# Tier-5 syscall: shutdown(2) — shut down part of a full-duplex connection (syscall 48)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_shutdown, sys_shutdown)
  - net/ipv4/tcp.c (tcp_shutdown, tcp_close)
  - net/ipv4/af_inet.c (inet_shutdown)
  - net/unix/af_unix.c (unix_shutdown)
  - include/uapi/sys/socket.h (SHUT_RD=0, SHUT_WR=1, SHUT_RDWR=2)
-->

## Summary

`shutdown(2)` shuts down all or part of a full-duplex socket connection. Unlike `close(2)` (which drops the fd and may close the connection if it's the last reference), `shutdown(2)` operates at the socket layer: it disables further sends (`SHUT_WR`), receives (`SHUT_RD`), or both (`SHUT_RDWR`), while leaving the fd open. For TCP this initiates the local-side FIN handshake; for UNIX it transitions the peer's read/write state without an on-wire signal. It is syscall number **48** on x86-64.

The path is `sys_shutdown(fd, how)` → `__sys_shutdown()` → `sock.ops.shutdown(sock, how)` → per-AF: e.g. `inet_shutdown` → `tcp_shutdown` for TCP, `unix_shutdown` for UNIX, etc.

Critical for: graceful TCP close ("half-close" pattern — finish writing, signal EOF, then drain remaining reads); RPC frameworks that want to signal request-end while keeping the response-stream open; UNIX-domain control sockets that want to atomically wake all peers on a multiplexed listener.

This Tier-5 covers syscall **48** `shutdown(int sockfd, int how)`.

## Signature

```c
int shutdown(int sockfd, int how);
```

```rust
pub fn sys_shutdown(sockfd: i32, how: i32) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_shutdown` → `__sys_shutdown(fd, how)`.

## Parameters

- **`sockfd`** — fd from `socket(2)` / `accept(2)`.
- **`how`** — one of:
  - `SHUT_RD` (0): disable further receives; pending data in receive queue may be discarded; future reads return 0 (EOF) for stream / `-ENOTCONN` for dgram.
  - `SHUT_WR` (1): disable further sends; TCP transmits FIN; future writes return `-EPIPE` (+ SIGPIPE unless masked).
  - `SHUT_RDWR` (2): both directions disabled.

## Return

0 on success; `-1`/`-errno` on error.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EINVAL` — how not in {0, 1, 2}.
- `ENOTCONN` — TCP/UNIX-stream not connected (TCP_CLOSE, TCP_LISTEN, TCP_SYN_SENT in some variants; UNIX-stream without peer).
- `ENOBUFS` — kmem alloc fail to queue FIN skb (TCP).
- `EOPNOTSUPP` — family does not implement shutdown (e.g. AF_PACKET has stub).
- `EACCES` / `EPERM` — LSM denial.

## ABI surface

- `how` is taken `int`, internally `how + 1` is used as a bitmask:
  - SHUT_RD + 1 == 1 → bit 0 → RECEIVE_SHUTDOWN.
  - SHUT_WR + 1 == 2 → bit 1 → SEND_SHUTDOWN.
  - SHUT_RDWR + 1 == 3 → bits 0|1 → both.
- `sk_shutdown` field in `struct sock` holds the bitmask (0..3).
- shutdown is idempotent: calling `SHUT_WR` twice is not an error.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: how validation:
- if how < 0 ∨ how > 2: return `-EINVAL`.

REQ-3: LSM:
- err = security_socket_shutdown(sock, how); if err: return err.

REQ-4: Per-AF dispatch:
- err = sock.ops.shutdown(sock, how).

REQ-5: AF_INET TCP semantics (tcp_shutdown):
- if sk.sk_state ∈ {TCP_LISTEN, TCP_SYN_SENT}: return -ENOTCONN.
- if how == SHUT_RD: sk.sk_shutdown |= RCV_SHUTDOWN; wake readers; do NOT drop receive queue (Linux: discards on close, not on shutdown).
- if how == SHUT_WR: sk.sk_shutdown |= SEND_SHUTDOWN; tcp_send_fin(sk); wake writers (so they observe EPIPE).
- if how == SHUT_RDWR: sk.sk_shutdown |= RCV_SHUTDOWN|SEND_SHUTDOWN; tcp_send_fin(sk); discard recv queue.
- transitions: TCP_ESTABLISHED → TCP_FIN_WAIT1 on first SHUT_WR.

REQ-6: AF_UNIX semantics (unix_shutdown):
- unix_state_lock(sk).
- sk.sk_shutdown |= (how + 1).
- let peer = unix_peer(sk).
- if peer:
  - peer_mask = 0;
  - if how != SHUT_RD: peer_mask |= RCV_SHUTDOWN; (we sent → peer received)
  - if how != SHUT_WR: peer_mask |= SEND_SHUTDOWN; (we won't read → peer write fails)
  - unix_state_lock(peer); peer.sk_shutdown |= peer_mask; unix_state_unlock(peer).
  - sock_wake_async(peer.sk_socket, SOCK_WAKE_WAITD, POLL_HUP).
- unix_state_unlock(sk).
- wake all waiters on sk.

REQ-7: AF_INET6 semantics (inet6_shutdown):
- mostly delegates to inet_shutdown / tcp_shutdown.

REQ-8: AF_PACKET:
- returns `-EOPNOTSUPP`; shutdown has no on-wire meaning.

REQ-9: Audit:
- audit_log_socketcall(SYS_SHUTDOWN, fd, NULL, 0, ret, how).

## Acceptance Criteria

- [ ] AC-1: shutdown(TCP, SHUT_WR) on connected socket: FIN sent; peer recv returns 0 after queue drain.
- [ ] AC-2: shutdown(TCP, SHUT_WR) followed by write: -EPIPE + SIGPIPE (unless MSG_NOSIGNAL).
- [ ] AC-3: shutdown(TCP, SHUT_RD): pending read data may be returned to local reader; subsequent reads return 0.
- [ ] AC-4: shutdown(TCP, SHUT_RDWR): both directions closed; queues drained; peer sees FIN.
- [ ] AC-5: shutdown(TCP, how=3): -EINVAL.
- [ ] AC-6: shutdown(TCP-LISTEN socket): -ENOTCONN.
- [ ] AC-7: shutdown(TCP-SYN_SENT): -ENOTCONN.
- [ ] AC-8: shutdown(UDP, SHUT_WR) on unconnected UDP: send returns -EPIPE thereafter.
- [ ] AC-9: shutdown(UNIX-stream, SHUT_WR): peer's recv returns 0 (EOF) after queued data drained; peer's send to us returns -EPIPE.
- [ ] AC-10: shutdown(closed fd): -EBADF.
- [ ] AC-11: shutdown on pipe: -ENOTSOCK.
- [ ] AC-12: shutdown(SHUT_WR) idempotent: second call returns 0; no second FIN.
- [ ] AC-13: shutdown(UNIX-stream, SHUT_RDWR) wakes blocked recv on peer with POLLHUP.

## Architecture

```
SysSocket::sys_shutdown(fd, how) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. if !(0..=2).contains(&how) { return Err(EINVAL); }
  3. Lsm::socket_shutdown(&sock, how)?;
  4. sock.ops.shutdown(&sock, how)?;
  5. Audit::log_socketcall(SYS_SHUTDOWN, fd, &(), 0, 0, how);
  6. Ok(0)
```

`tcp_shutdown(sk, how)` (AF_INET):
1. let how_mask = (how + 1) as u8; // bitmask of RCV_SHUTDOWN | SEND_SHUTDOWN.
2. lock_sock(sk).
3. if sk.sk_state ∈ {TCP_LISTEN, TCP_SYN_SENT}: release_sock(sk); return Err(ENOTCONN).
4. sk.sk_shutdown |= how_mask.
5. if how_mask & SEND_SHUTDOWN:
   - if sk.sk_state == TCP_ESTABLISHED: sk.sk_state = TCP_FIN_WAIT1.
   - elif sk.sk_state == TCP_CLOSE_WAIT: sk.sk_state = TCP_LAST_ACK.
   - tcp_send_fin(sk).
6. if how_mask & RCV_SHUTDOWN:
   - sk_wake_async(sk, SOCK_WAKE_WAITD, POLL_IN).
7. release_sock(sk).
8. return Ok(()).

`unix_shutdown(sock, how)` (AF_UNIX):
1. let how_mask = (how + 1) as u8.
2. unix_state_lock(sk).
3. sk.sk_shutdown |= how_mask.
4. let peer = unix_peer_get(sk).
5. unix_state_unlock(sk).
6. wake_up_interruptible_all(&sk.sk_wq.wait).
7. if let Some(peer) = peer:
   - let peer_mask = 0
     | (if how != SHUT_RD { RCV_SHUTDOWN } else { 0 })
     | (if how != SHUT_WR { SEND_SHUTDOWN } else { 0 });
   - unix_state_lock(peer).
   - peer.sk.sk_shutdown |= peer_mask.
   - unix_state_unlock(peer).
   - wake_up_interruptible_all(&peer.sk_wq.wait).
   - sock_wake_async(peer.sk_socket, SOCK_WAKE_WAITD, POLL_HUP).
   - sock_put(peer).
8. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `shutdown_how_validated` | INVARIANT | per-sys_shutdown: how ∈ {0,1,2} else EINVAL. |
| `shutdown_idempotent` | INVARIANT | per-tcp_shutdown: repeated SHUT_WR does not emit second FIN. |
| `shutdown_state_progression` | INVARIANT | per-tcp_shutdown: TCP_ESTABLISHED + SHUT_WR ⟹ TCP_FIN_WAIT1; TCP_CLOSE_WAIT + SHUT_WR ⟹ TCP_LAST_ACK. |
| `shutdown_listen_rejected` | INVARIANT | per-tcp_shutdown: sk_state ∈ {TCP_LISTEN, TCP_SYN_SENT} ⟹ ENOTCONN. |
| `shutdown_peer_wake` | INVARIANT | per-unix_shutdown: peer existence ⟹ peer waitqueue woken before return. |
| `shutdown_peer_refcount_balanced` | INVARIANT | per-unix_shutdown: every unix_peer_get balanced by sock_put. |

### Layer 2: TLA+

`uapi/syscalls/shutdown.tla`:
- States: validate, lsm, dispatch, lock, set-flags, send-fin (TCP), wake-peer (UNIX), release-lock, return.
- Models TCP state machine transitions and UNIX peer waking.
- Properties:
  - `safety_no_double_fin` — repeated SHUT_WR ⟹ at most one FIN observed by peer.
  - `safety_shutdown_visible` — return success ⟹ sk_shutdown reflects requested bits.
  - `safety_peer_observes_eof` — UNIX-stream SHUT_WR ⟹ eventually peer.sk_shutdown has RCV_SHUTDOWN set.
  - `safety_state_machine_monotonic` — sk_state only transitions per valid TCP edges.
  - `liveness_terminates` — shutdown returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: how ∈ {0,1,2} | `sys_shutdown` |
| post: sk.sk_shutdown ⊇ (how+1) | per-AF shutdown |
| post: TCP_ESTABLISHED ∧ SHUT_WR ⟹ tcp_send_fin called exactly once | `tcp_shutdown` |
| post: UNIX peer non-null ⟹ peer.sk_shutdown ⊇ symmetric_mask(how) | `unix_shutdown` |
| post: lock acquired ⟹ lock released on every return path | `tcp_shutdown` / `unix_shutdown` |

### Layer 4: Verus/Creusot functional

Per-`tcp_shutdown`, `unix_shutdown` semantic equivalence with upstream. TCP state-machine edges match `tcp_close_state`/`tcp_send_fin`. UNIX peer-mask computation (`how != SHUT_RD ⟹ peer RCV_SHUTDOWN`, etc.) exactly matches `net/unix/af_unix.c:unix_shutdown`.

## Hardening

- **`how` bounds-checked before any kernel-state mutation** — defense against per-OOB-write into `sk_shutdown` bitmask via attacker-supplied `how`.
- **`sk_shutdown` modified under `lock_sock` / `unix_state_lock`** — defense against per-race with concurrent send/recv observing partial flag set.
- **`tcp_send_fin` gated to `SEND_SHUTDOWN` and only-on-state-transition** — defense against per-FIN-flood emitted by repeated shutdown calls.
- **TCP `sk_state` checked pre-FIN — TCP_LISTEN / TCP_SYN_SENT rejected** — defense against per-stale-state FIN emission.
- **`unix_peer_get` + `sock_put` balanced** — defense against per-refleak on the peer.
- **Peer waitqueue woken explicitly after `peer.sk_shutdown |=`** — defense against per-stuck-recv on the peer.
- **LSM `socket_shutdown`** — defense against per-MAC bypass.
- **Receive queue NOT freed on `SHUT_RD`** — defense against per-data-loss for buffered data the local reader could still consume (Linux semantics).
- **`POLLHUP` delivered to peer on bilateral shutdown** — defense against per-stuck-epoll.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized; per-AF shutdown stack frames placed at randomized offsets.
- **PaX UDEREF** — `how` is a register argument, no userland deref required; nothing to UDEREF-protect.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on shutdown.
- **GRKERNSEC_BLACKHOLE** — shutdown emits a FIN; blackhole policy decides whether incoming FIN_ACK to a shut-down socket gets a RST. Hardened build: silently drop, matching upstream + grsec blackhole.
- **GRKERNSEC_RANDNET** — irrelevant on shutdown (no port allocation).
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — shutdown does not re-check capabilities; the original socket creation gated them.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX shutdown across user-namespace boundaries: peer `sk_shutdown` propagation is unaffected; SCM_RIGHTS in-flight at shutdown time still completes (no premature fp[] drop).
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on shutdown (no fd creation).
- **SCM_RIGHTS recursion bound** — irrelevant on shutdown (no scm send).
- **getsockopt info-leak prevention** — `SO_ERROR` after `SHUT_RDWR` may report `ECONNRESET` or `EPIPE`; ensure we return the deliberate error code, not residual stack data.
- **sendmmsg compound-bound limit / splice fd-permission boundary** — irrelevant on shutdown.

## Open Questions

- For `SHUT_RD` on TCP: Linux historically does not discard receive queue, allowing local reader to drain. Some hardened policies prefer discard-on-SHUT_RD to immediately free memory. Upstream + grsec: no discard. We follow.
- For UNIX-DGRAM shutdown: the semantics are weak (no on-wire signal); we set `sk_shutdown` on both peers and propagate `POLLHUP`; matches upstream.

## Out of Scope

- `close(2)` — separate Tier-5 (different semantics; drops last reference).
- TCP state machine - Tier-3 `net/ipv4/tcp_state.md`.
- `SO_LINGER` interaction with shutdown — Tier-3 `net/core/sock_options.md`.
- Per-AF `sock.ops.shutdown` implementations — Tier-3 per family.
- Implementation code.
