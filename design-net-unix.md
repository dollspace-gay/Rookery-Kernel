---
title: "Tier-3: net/unix — AF_UNIX socket family (stream/dgram/seqpacket + SCM_RIGHTS)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the AF_UNIX socket family — Unix-domain sockets used for local IPC. Implements all three socket types (`SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_SEQPACKET`), pathname (`/run/foo.sock`) + abstract (`@foo`) namespace, peer credentials (`SCM_CREDENTIALS`/`SO_PEERCRED`/`SO_PEERSEC`), and the FD-passing primitive `SCM_RIGHTS` plus its file-descriptor garbage collector (cyclic-FD-graph collection in `net/unix/garbage.c`).

Sub-tier-3 of `net/00-overview.md`. The most heavily-used socket family on a typical Linux system: systemd's UNIX-socket activation, Docker / containerd's control plane, X11/Wayland, dbus, journald, syslog, and most application-server <-> local-tool IPC. SCM_RIGHTS is the kernel's only mechanism for passing open file descriptors between unrelated processes.

### Requirements

- REQ-1: `struct sockaddr_un` byte-identical layout; `UNIX_PATH_MAX=108`.
- REQ-2: All three socket types implemented: SOCK_STREAM, SOCK_DGRAM, SOCK_SEQPACKET; per-type semantics identical.
- REQ-3: Pathname namespace: bind creates S_IFSOCK FS inode with bind-time uid/gid/mode; connect resolves via VFS path-resolution.
- REQ-4: Abstract namespace: per-netns abstract-name hash table; bind registers + connect resolves identically.
- REQ-5: SCM_RIGHTS: receiver gets new FDs in own FD table referencing same `struct file` (refcount bump on send, drop on close); RLIMIT_NOFILE check enforced.
- REQ-6: SCM_CREDENTIALS: pid/uid/gid sender-supplied but kernel-verified against sending task's actual creds.
- REQ-7: SO_PEERCRED, SO_PEERSEC, SO_PEERPIDFD: peer creds captured at connect-time; immutable until reconnect.
- REQ-8: FD-passing GC (`net/unix/garbage.c`): periodic GC + on-close trigger; mark-and-sweep over reachable FDs identifies cyclic-only references; reclaims.
- REQ-9: sock_diag NETLINK_SOCK_DIAG for AF_UNIX wire-format identical.
- REQ-10: `/proc/net/unix` format identical.
- REQ-11: sysctl `max_dgram_qlen` default + r/w semantics identical.
- REQ-12: BPF sockmap integration (`net/unix/unix_bpf.c`): SOCK_STREAM/SEQPACKET sockets attachable to sockmap via `BPF_MAP_UPDATE_ELEM`. Identical UAPI.
- REQ-13: TLA+ model `models/net/unix_scm_gc.tla` (mandatory per `net/00-overview.md` Layer 2 — FD-passing safety) — proves: SCM_RIGHTS GC reclaims iff cyclic-only AND no live FD-table holder.
- REQ-14: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct sockaddr_un` byte-identical. (covers REQ-1)
- [ ] AC-2: For each of SOCK_STREAM/DGRAM/SEQPACKET: a connect-send-recv round-trip succeeds; payload byte-identical; per-type framing semantics (stream byte-stream, dgram boundary-preserving, seqpacket boundary+ordered) verifiable. (covers REQ-2)
- [ ] AC-3: A pathname bind to `/tmp/xs.sock` creates an inode with `mode = S_IFSOCK | bind-mode` and the bind-time fsuid/fsgid; `stat` confirms. (covers REQ-3)
- [ ] AC-4: An abstract-bind on `\0foo` in netns A is invisible from netns B (separate abstract namespaces per netns). (covers REQ-4)
- [ ] AC-5: SCM_RIGHTS test: process A passes an FD to B via sendmsg; B's recvmsg returns a new FD; both FDs refer to the same inode (via stat). RLIMIT_NOFILE-1 in B causes EMFILE on receive. (covers REQ-5)
- [ ] AC-6: SCM_CREDENTIALS: receiver always sees correct sender pid/uid/gid; spoofed pid (sender supplies wrong pid in cmsg without CAP_SYS_ADMIN) returns EPERM. (covers REQ-6)
- [ ] AC-7: SO_PEERCRED + SO_PEERPIDFD on a connected socket return matching uid/gid/pid; the pidfd-poll signals when peer exits. (covers REQ-7)
- [ ] AC-8: Cyclic-FD GC test: A passes FD to B who passes back to A; both close non-cyclic refs; GC reclaims within bounded time. (covers REQ-8)
- [ ] AC-9: `ss -x` output byte-identical between Rookery + upstream after equivalent boot. (covers REQ-9, REQ-10)
- [ ] AC-10: `cat /proc/sys/net/unix/max_dgram_qlen` returns `512`; write `1024` then read returns `1024`. (covers REQ-11)
- [ ] AC-11: BPF sockmap test: attach a SOCK_STREAM AF_UNIX pair to a sockmap; redirect via `bpf_sk_redirect_map` works. (covers REQ-12)
- [ ] AC-12: TLA+ `models/net/unix_scm_gc.tla` proves: ∀ FD `f`, GC reclaims `f` ⇔ (`f` is in a cycle) ∧ (no FD-table outside cycle holds `f`). (covers REQ-13)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-14)

### Architecture

### Rust module organization

- `kernel::net::unix::AfUnix` — socket family root
- `kernel::net::unix::sock::UnixSock` — `struct unix_sock`
- `kernel::net::unix::stream::Stream`, `dgram::Dgram`, `seqpacket::SeqPacket` — per-type send/recv
- `kernel::net::unix::name::PathnameTable` — pathname-bound (FS-rooted)
- `kernel::net::unix::name::AbstractTable` — per-netns abstract-name hash
- `kernel::net::unix::scm::ScmHandler` — SCM_RIGHTS / SCM_CREDENTIALS / SCM_SECURITY
- `kernel::net::unix::gc::UnixGc` — FD-passing cyclic-graph GC
- `kernel::net::unix::diag::UnixDiag` — sock_diag responder
- `kernel::net::unix::bpf::UnixBpf` — sockmap integration

### Locking and concurrency

- **`unix_table_locks[]`** (per-hash-bucket spinlock): pathname + abstract name registries
- **Per-`unix_sock` `iolock`** (mutex): serializes sendmsg / recvmsg
- **`unix_gc_lock`** (spinlock): GC walking
- **`unix_socket_table` rwlock**: GC vs. mutator coordination

### Error handling

- `Err(EADDRINUSE)` — bind path/abstract name already in use
- `Err(EACCES)` — pathname directory permission
- `Err(EISCONN)` — connect on already-connected SEQPACKET/STREAM
- `Err(ENOTCONN)` — sendmsg without connect
- `Err(EMFILE)` — SCM_RIGHTS recv hits RLIMIT_NOFILE
- `Err(EAGAIN)` — non-blocking recvmsg with no message
- `Err(EPIPE)` — peer closed; SIGPIPE generated to sender

### Out of Scope

- TIPC / vsock socket families
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| AF_UNIX core: socket family, bind, connect, sendmsg, recvmsg, accept, listen | `net/unix/af_unix.c` |
| FD-passing GC (cyclic-graph collection) | `net/unix/garbage.c` |
| sock_diag NETLINK reporter | `net/unix/diag.c` |
| sysctl tunables (max_dgram_qlen) | `net/unix/sysctl_net_unix.c` |
| BPF integration (sockmap-style) | `net/unix/unix_bpf.c` |
| Generic SCM (auxiliary control message) handling | `net/core/scm.c` |
| Public API | `include/net/af_unix.h` |
| UAPI | `include/uapi/linux/un.h` (struct sockaddr_un, UNIX_PATH_MAX=108) |

### compatibility contract

### `socket(AF_UNIX, type, 0)`

`type` ∈ `{SOCK_STREAM, SOCK_DGRAM, SOCK_SEQPACKET}`. Identical.

### `struct sockaddr_un`

```c
struct sockaddr_un {
    sa_family_t sun_family;       /* AF_UNIX */
    char        sun_path[108];    /* pathname or abstract */
};
```

Layout-identical. `UNIX_PATH_MAX = 108` (kept; not increased). Three name forms:
1. **Pathname**: `sun_path[0] != '\0'` and rest filesystem-resolved
2. **Abstract**: `sun_path[0] == '\0'` and `sun_path[1..]` is the abstract name; netns-scoped
3. **Unnamed**: zero-length address (transient, e.g., post-`socket()` pre-`bind()`)

### bind / connect / listen / accept

Identical semantics. Pathname binds create an FS inode of mode S_IFSOCK; abstract binds register only in the netns abstract-name table. Peer credentials captured at connect-time are immutable (`getsockopt SO_PEERCRED`).

### sendmsg / recvmsg + cmsg

| Cmsg | Semantics |
|---|---|
| `SCM_RIGHTS` | array of FDs; receiver gets new FDs in its FD table referencing same `struct file` |
| `SCM_CREDENTIALS` | sender pid+uid+gid (verified by kernel against sender; trusted) |
| `SCM_SECURITY` | LSM-supplied security context (e.g., SELinux label) |

Format byte-identical so libsystemd's `sd_notify` and `sd_listen_fds` work unchanged.

### `SO_PEERCRED`, `SO_PEERSEC`, `SO_PASSCRED`, `SO_PASSSEC`, `SO_PEERPIDFD`

`getsockopt`/`setsockopt` on `SOL_SOCKET` semantics identical. `SO_PEERPIDFD` returns a pidfd to the peer's task at connect-time (race-free vs. SO_PEERCRED's pid which can wrap).

### FD-passing GC

When SCM_RIGHTS passes FDs that themselves refer to AF_UNIX sockets, the kernel maintains a cyclic-graph GC (`net/unix/garbage.c`) to detect + reclaim unreachable FD cycles. Algorithm: per-FD reference count + mark-and-sweep on garbage-prone events.

Identical algorithm so userspace pathological-cycle test cases behave identically.

### sysctl `/proc/sys/net/unix/max_dgram_qlen`

Default `512`. Tunable. Identical.

### `/proc/net/unix`

Per-socket diag table. Format-identical so `ss -x`, `lsof -U`, `netstat -x` work unchanged.

### NETLINK_SOCK_DIAG family for AF_UNIX

`UNIX_DIAG_*` request/response identical wire format.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Pathname/abstract hash-bucket linked-list manipulation | `kani::proofs::net::unix::name_table_safety` |
| SCM_RIGHTS FD-table install + refcount | `kani::proofs::net::unix::scm_install_safety` |
| FD-passing GC walk + reclaim | `kani::proofs::net::unix::gc_safety` |
| sendmsg/recvmsg buffer copy via copy_{to,from}_iter | `kani::proofs::net::unix::msg_copy_safety` |

### Layer 2: TLA+ models

- `models/net/unix_scm_gc.tla` (mandatory per `net/00-overview.md` Layer 2) — proves: GC correctness as stated above. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Pathname/abstract registries | each (netns, name) maps to ≤1 listening unix_sock | `kani::proofs::net::unix::name_uniqueness` |
| Per-`unix_sock` peer-link | bidirectional: A.peer == B ⇒ B.peer == A (until disconnect) | `kani::proofs::net::unix::peer_link_invariants` |
| SCM_RIGHTS embedded-FD count | per-skbuff embedded-FD count matches passed cmsg array length | `kani::proofs::net::unix::scm_count_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/00-overview.md` Layer 4)

- **SCM_RIGHTS GC soundness** via Verus — proves: every reclaimed FD was unreachable from any FD-table; no reachable FD is reclaimed.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-socket + per-skbuff embedded-FD refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`unix_sock` slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed unix_sock + per-skbuff embedded-FD arrays cleared | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: `unix_dgram_ops`, `unix_stream_ops`, `unix_seqpacket_ops` proto_ops vtables `static const`
- **USERCOPY**: sendmsg/recvmsg payload movement uses `copy_to_iter`/`copy_from_iter` exclusively
- **SIZE_OVERFLOW**: cmsg-length arithmetic uses checked operators (defense vs. CVE-2014-9529-class)

### Row-2 / GR-RBAC integration

- LSM hook `security_unix_stream_connect` + `security_unix_may_send` (already in upstream); GR-RBAC policy can deny per-subject (default empty).
- LSM hook `security_socket_getpeersec_dgram` for SCM_SECURITY emission.
- A useful default GR-RBAC policy fragment (defined in `security/grsec/00-overview.md`, default empty): deny SCM_RIGHTS FD passing across role boundaries unless explicitly allowed — mitigates the "cap escalation via SCM_RIGHTS" class of attacks.

### Userspace-visible behavior changes

None beyond upstream defaults (GR-RBAC's default empty policy permits all UNIX socket operations).

### Verification

(See § Verification above.)

