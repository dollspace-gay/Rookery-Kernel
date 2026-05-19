# Tier-3: net/xfrm/espintcp — RFC 8229 ESP-in-TCP encapsulation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/espintcp.c
  - include/net/espintcp.h
  - include/net/xfrm.h
-->

## Summary
Tier-3 design for ESP-in-TCP (RFC 8229) — wraps ESP packets inside a TCP stream so IPSec traffic can traverse middleboxes that block UDP/4500 (NAT-T) or that deeply inspect ESP packets. The kernel-side implementation: a stream-parser-attached TCP socket where userspace daemon (strongSwan, libreswan) cooperatively maintains the connection while the kernel injects encrypted ESP packets into the TX queue and pulls received-frame'd ESP packets from the RX side via `strparser`.

Particularly relevant in countries with restrictive Internet middleboxes, hotel/airport WiFi captive portals, and mobile networks with aggressive NAT/firewall behavior.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (per-SA `encap` member uses `UDP_ENCAP_ESPINTCP`), `net/xfrm/output.md` (TX writes through espintcp socket), `net/xfrm/input.md` (RX reads from espintcp socket).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| ESP-in-TCP socket-attached state, strparser callbacks, TX/RX integration | `net/xfrm/espintcp.c` |
| Public API | `include/net/espintcp.h` |
| Cross-ref | `include/net/xfrm.h` (encap struct extension) |

## Compatibility contract

### RFC 8229 wire format

ESP packets carried over a TCP connection between IKE peers. Each frame on the TCP stream:
```
+---------------+---------------+
|         Length (2 bytes)      | (network byte order; total length of following ESP packet)
+---------------+---------------+
|                                |
|     ESP packet (variable)     |
|                                |
+---------------+---------------+
```

(Length=0 is reserved for keepalive; identical RFC 8229 § 4 framing.)

### Setup sequence

1. Userspace daemon establishes TCP connection to peer (typically port 4500)
2. After IKE_SA_INIT, daemon configures kernel SA with `XFRMA_ENCAP` carrying `UDP_ENCAP_ESPINTCP` + the established TCP socket fd
3. Kernel installs `espintcp_sock` callback on the TCP socket via `tcp_set_ulp("espintcp")` (ULP layer)
4. Kernel attaches `strparser` callbacks to extract per-frame ESP packets from the TCP byte stream
5. ESP TX from xfrm_output: looks up matching SA → if encap=ESPINTCP → write framed ESP into the attached socket's send queue
6. RX from socket: strparser delivers per-frame ESP packets to xfrm_input

### `tcp_set_ulp("espintcp")` (Upper Layer Protocol)

The TCP-ULP framework lets a per-socket ULP module hook into TCP RX/TX. ESP-in-TCP registers as a ULP; once attached, the socket cannot be used for plain-text TCP recv/send (RFC 8229 § 5 specifies dedicated TCP connection per-tunnel).

Identical ULP attach mechanism so existing strongSwan/libreswan setup sequences work.

### `strparser` integration

`strparser` (kernel stream-parser framework, cross-ref `net/strparser/00-overview.md`) extracts per-RFC-8229 frames from the TCP stream:
- `parse_msg`: read 2-byte length prefix → return frame size (or `-EAGAIN` if not enough)
- `rcv_msg`: deliver complete ESP frame to `xfrm_input` for processing

Identical strparser callbacks.

### `socket_lock_t` and concurrency

The ULP-attached TCP socket is shared between userspace daemon (which sends IKE keepalives over the same connection — per RFC 8229 length=0 framing for keepalive, longer-than-2-byte for IKE) and kernel ESP TX/RX. The kernel takes the socket lock for ESP injection; daemon's sendmsg also takes the lock. Per-socket FIFO order preserved.

### Cooperative IKE-traffic interleaving

RFC 8229 § 4 specifies: IKE messages on the same TCP connection use the same length-prefix framing. The kernel-side ESPINTCP only consumes frames where the first byte after length-prefix indicates ESP (per byte=0 sentinel test or specific protocol marker — kernel uses RFC 8229 § 4 mandatory non-ESP-marker check). Otherwise the frame is queued for daemon recvmsg.

Identical demultiplexing.

### Per-SA `encap` extension

`xfrm_state.encap` extended with `UDP_ENCAP_ESPINTCP` value:
```c
#define UDP_ENCAP_ESPINTCP   7   /* (post-NAT-T encap values) */
```

Identical UAPI value.

## Requirements

- REQ-1: RFC 8229 framing: 2-byte length prefix + ESP packet; length=0 reserved for keepalive; identical wire format.
- REQ-2: TCP-ULP integration: `tcp_set_ulp("espintcp")` registers ULP; per-socket attach succeeds for unattached TCP sockets.
- REQ-3: `strparser` callbacks: `parse_msg` reads length prefix, returns frame size or -EAGAIN; `rcv_msg` delivers complete ESP frames to `xfrm_input`.
- REQ-4: ESP TX via espintcp: `xfrm_output` for SA with `encap_type=UDP_ENCAP_ESPINTCP` writes RFC 8229-framed ESP into the attached socket's send queue.
- REQ-5: ESP RX via espintcp: strparser delivers frames; non-ESP frames (IKE) queued for daemon recvmsg; ESP frames delivered to `xfrm_input`.
- REQ-6: Cooperative IKE/ESP demultiplexing per RFC 8229 § 4: kernel inspects framed packet's first byte (or non-ESP-marker) to route between daemon-recvmsg and ESP-decap path.
- REQ-7: Socket-lock serialization: kernel ESP injection + daemon sendmsg + strparser RX coordinated via `lock_sock`/`release_sock`; per-socket FIFO preserved.
- REQ-8: SA encap extension: `UDP_ENCAP_ESPINTCP=7` UAPI; `xfrm_state.encap.encap_type` field accepts this value.
- REQ-9: Detach handling: when daemon closes the socket → kernel cleanly releases ULP attach + flushes pending TX; SAs using this socket fall to error state (XfrmOutNoStates on subsequent TX).
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: TCP-ULP attach test: `setsockopt(TCP_ULP, "espintcp", 8)` on a connected TCP socket → success; subsequent attach attempts on already-ULP'd socket return EBUSY. (covers REQ-2)
- [ ] AC-2: RFC 8229 framing test: install SA with `encap_type=UDP_ENCAP_ESPINTCP`; trigger ESP TX; tcpdump on outer TCP shows 2-byte length prefix + ESP. (covers REQ-1, REQ-4)
- [ ] AC-3: strparser test: simulated RX of 2 framed ESP packets concatenated → strparser splits + delivers each separately to `xfrm_input`. (covers REQ-3)
- [ ] AC-4: IKE-interleave test: daemon sends an IKE message via `sendmsg` on the espintcp socket → message framed with length prefix; receiver kernel sees non-ESP marker, queues to daemon's recvmsg; kernel ESP TX continues unaffected. (covers REQ-6)
- [ ] AC-5: Concurrent test: daemon sendmsg + kernel ESP TX both racing → `lock_sock` serializes; no corruption; tcpdump shows interleaved frames. (covers REQ-7)
- [ ] AC-6: Daemon-close test: daemon closes the espintcp socket → kernel releases ULP attach; subsequent ESP TX with the SA returns error counter increment. (covers REQ-9)
- [ ] AC-7: Wire-compat test: a Rookery-host using espintcp interoperates with a stock-strongSwan / libreswan peer over the same TCP connection. (covers REQ-1, REQ-4, REQ-5)
- [ ] AC-8: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::xfrm::espintcp::EspInTcp` — top-level
- `kernel::net::xfrm::espintcp::Ulp` — TCP-ULP attach (`tcp_set_ulp("espintcp")`)
- `kernel::net::xfrm::espintcp::Strparser` — `parse_msg` + `rcv_msg` callbacks
- `kernel::net::xfrm::espintcp::Tx` — kernel ESP injection (write framed ESP into socket sendq)
- `kernel::net::xfrm::espintcp::Rx` — strparser-delivered frame dispatch
- `kernel::net::xfrm::espintcp::Demux` — IKE vs ESP demultiplexing (RFC 8229 § 4 marker check)
- `kernel::net::xfrm::espintcp::Detach` — clean teardown on socket close

### Locking and concurrency

- **Per-socket lock_sock/release_sock**: serializes daemon sendmsg + kernel ESP TX
- **Strparser per-socket workqueue**: RX side; runs single-threaded per socket
- **Per-SA `encap` field**: read under SA lock (cross-ref `net/xfrm/state.md`)

### Error handling

- `Err(EBUSY)` — socket already ULP-attached
- `Err(EOPNOTSUPP)` — espintcp not enabled (CONFIG)
- `Err(EINVAL)` — bad framing (length prefix exceeds reasonable limit)
- `Err(ECONNRESET)` — peer closed TCP connection mid-frame
- `Err(EBADMSG)` — IKE-marker validation failed
- `Err(EAGAIN)` — partial frame; strparser retains state

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Length-prefix parser (no integer overflow on 2-byte length) | `kani::proofs::net::xfrm::espintcp::framing_safety` |
| TCP-ULP attach (no double-attach, no detach-during-use) | `kani::proofs::net::xfrm::espintcp::ulp_safety` |
| Strparser frame dispatch (frame ≤ socket recv buffer) | `kani::proofs::net::xfrm::espintcp::strparser_safety` |
| Demux marker check (no out-of-bounds read) | `kani::proofs::net::xfrm::espintcp::demux_safety` |
| Detach cleanup (no use-after-free of pending TX) | `kani::proofs::net::xfrm::espintcp::detach_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-socket espintcp state | when `socket->sk_user_data` points to espintcp_sock, ULP is attached; when NULL, ULP is detached | `kani::proofs::net::xfrm::espintcp::state_invariants` |
| Strparser per-socket queue | parse-state advances monotonically; never reads past length-prefix when frame body not yet complete | `kani::proofs::net::xfrm::espintcp::parse_invariants` |

### Layer 4: Functional correctness (opt-in)

- **RFC 8229 frame-format theorem** via Verus — proves: serialized frame on the wire equals the RFC 8229 § 4 specification for any ESP/IKE input.
- **Demux-correctness theorem** via Verus — proves: ∀ frame `f`, kernel routes to ESP-decap iff `f` has ESP marker (per RFC 8229 § 4); IKE frames go to daemon recvmsg.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | freed strparser frame buffers cleared (decrypted ESP payload sensitive) | § Default-on configurable off |
| **SIZE_OVERFLOW** | length-prefix arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-socket (cross-ref `net/sock.md`)
- **CONSTIFY**: ULP ops vtable + strparser callbacks `static const`
- **SIZE_OVERFLOW**: see above
- **USERCOPY**: TCP recv-buffer copy via `copy_to_iter`/`copy_from_iter`

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_setsockopt` for `TCP_ULP=espintcp`; GR-RBAC policy can deny per-subject (default empty).
- Useful default GR-RBAC policy: deny espintcp ULP attach outside gradm-marked `ipsec_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds TCP-recv buffer copies via `copy_to_iter` / `copy_from_iter` carrying length-prefixed ESP-in-TCP frames.
- **PAX_KERNEXEC** — keeps the `static const tcp_ulp_ops espintcp` registration and strparser callback tables text W^X.
- **PAX_RANDKSTACK** — randomises stack per `espintcp_rcv_message`/`espintcp_sendmsg` entry; defeats ROP under TCP-segmented ESP floods.
- **PAX_REFCOUNT** — wraps the inner `xfrm_state` refs and per-socket ULP refs taken across strparser callbacks against UAF on TCP close.
- **PAX_MEMORY_SANITIZE** — zeroes freed strparser frame buffers (decrypted ESP payload) and per-message scratch on free.
- **PAX_UDEREF** — protects setsockopt(`TCP_ULP=espintcp`) and any ULP-driven ioctl from userland deref on malformed args.
- **PAX_RAP / kCFI** — protects indirect dispatch through `tcp_ulp_ops` and strparser `cb.parse_msg`/`cb.rcv_msg` edges.
- **GRKERNSEC_HIDESYM** — hides `espintcp_*`, `strparser_*`, `tcp_ulp_*` symbols from unprivileged readers.
- **GRKERNSEC_DMESG** — restricts dmesg so length-prefix parse errors don't leak SA SPIs or peer addresses.
- **IPsec SAD/SPD CAP_NET_ADMIN** — the SAs that espintcp wraps are installed via the CAP_NET_ADMIN-gated XFRM netlink path at `state.md`/`user.md`.
- **XFRM key MEMORY_SANITIZE** — espintcp itself never touches SA keys, but the per-frame decrypted payload buffer is wiped on free so cleartext doesn't bleed across strparser frames.
- **Replay-window protected** — replay state of the underlying SA is owned by `state.md` and accessed under `x->lock`; espintcp never copies replay counters into the TCP stream.
- **TCP_ULP attach RBAC gate** — `TCP_ULP=espintcp` is restricted to gradm `ipsec_admin` role; an unprivileged sock cannot inject framing into a peer TCP flow.
- **Length-prefix SIZE_OVERFLOW** — RFC 8229 `len` prefix is range-checked against `INT_MAX` and against strparser's `max_msg_len`; truncated/oversized frames are dropped pre-decrypt.

Rationale: ESP-in-TCP exists specifically to traverse middleboxes where UDP+ESP is blocked, which means the kernel must accept attacker-influenced TCP segmentation around length-prefixed ciphertext. USERCOPY on the ULP recv path, REFCOUNT on the ULP socket+SA refs, MEMORY_SANITIZE on freed decrypted frames, RBAC gating of the ULP attach, and SIZE_OVERFLOW-checked length-prefix arithmetic collapse this to the underlying ESP transform.

## Open Questions

(none — RFC 8229 framing + Linux ULP integration exhaustively specified by RFC + upstream)

## Out of Scope

- IKE protocol semantics on the TCP connection (userspace daemon's responsibility)
- TCP-ULP framework details (cross-ref `net/strparser/` Tier-3 + `net/ipv4/tcp_ulp.md` Tier-3)
- 32-bit-only paths
- Implementation code
