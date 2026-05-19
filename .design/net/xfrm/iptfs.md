# Tier-3: net/xfrm/iptfs — IP-TFS aggregation/fragmentation (RFC 9347)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_iptfs.c
  - net/xfrm/trace_iptfs.h
  - include/net/xfrm.h
  - include/uapi/linux/xfrm.h
-->

## Summary
Tier-3 design for IP-TFS (IP Traffic Flow Security — RFC 9347): a Traffic Flow Confidentiality (TFC) mode for IPSec ESP that defeats traffic-pattern analysis by aggregating multiple inner packets into fixed-size outer ESP datagrams (with padding when input rate is below configured rate) and fragmenting oversize inner packets across outer datagrams. The result: an attacker observing the encrypted stream sees a constant-rate constant-size flow that reveals nothing about inner packet boundaries, sizes, or arrival patterns.

Configured per-SA via `XFRMA_IPTFS_*` NLAs (`XFRMA_IPTFS_PKT_SIZE`, `XFRMA_IPTFS_MAX_QUEUE_SIZE`, `XFRMA_IPTFS_DROP_TIME`, `XFRMA_IPTFS_REORDER_WINDOW`, `XFRMA_IPTFS_INIT_DELAY`, `XFRMA_IPTFS_DONT_FRAG`). Inner packets queued on TX → packed into outer ESP frames sized at `pkt_size` → emitted at configured rate. RX fragments reassembled per RFC 9347 § 7.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (per-SA IPTFS state), `net/xfrm/output.md` (TX integration), `net/xfrm/input.md` (RX reassembly), `net/xfrm/user.md` (XFRMA_IPTFS_* NLA parsing).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| IP-TFS aggregation/fragmentation core: per-SA queue, packet assembler, RX reassembler, fragment-tracker | `net/xfrm/xfrm_iptfs.c` |
| Tracepoints for IP-TFS observability | `net/xfrm/trace_iptfs.h` |
| Public API | `include/net/xfrm.h` |
| UAPI: XFRMA_IPTFS_* NLA enum | `include/uapi/linux/xfrm.h` |

## Compatibility contract

### IP-TFS payload format (RFC 9347 § 2.1)

Outer ESP payload is an IP-TFS Payload Header followed by aggregated inner-packet sub-payloads:
```
+---------------------------------------------------------------+
| Subtype |    Reserved (rsv)   |       Block Offset (BO)       |   <-- Payload Header (4 bytes)
+---------------------------------------------------------------+
|                                                               |
|              Aggregated Inner Packet 1 (variable)             |
|                                                               |
+---------------------------------------------------------------+
|                                                               |
|              Aggregated Inner Packet 2 (variable)             |
|                                                               |
+---------------------------------------------------------------+
|                       Padding (if any)                        |
+---------------------------------------------------------------+
```

`Subtype` = 0 (basic IPTFS); 1 (CC — congestion-control mode RFC 9347 § 2.2.2). `Block Offset` is the byte offset within the outer payload of the first complete inner packet — used for RFC 9347 § 7 reassembly when prior outer datagram contained a fragment.

Wire format byte-identical so RFC-compliant peers interoperate.

### TX algorithm (RFC 9347 § 6)

1. Inner packet enters per-SA TX queue (bounded by `XFRMA_IPTFS_MAX_QUEUE_SIZE`)
2. Periodic timer (initial delay = `XFRMA_IPTFS_INIT_DELAY`, then per-SA rate from upper layer) fires
3. Build outer payload of `XFRMA_IPTFS_PKT_SIZE` bytes:
   - Append IPTFS Payload Header
   - Pop inner packets from queue, append until next packet would overflow
   - If next packet would overflow AND `XFRMA_IPTFS_DONT_FRAG=0` → fragment that packet across this and next outer
   - If `XFRMA_IPTFS_DONT_FRAG=1` AND next packet is too big → drop with counter increment
   - Pad remaining bytes with zero
4. Hand assembled payload to `xfrm_output` for ESP encrypt + emit

### RX algorithm (RFC 9347 § 7)

1. Receive ESP-decap'd payload
2. Read IPTFS Payload Header → extract Block Offset
3. If Block Offset > 0 → completes a fragment from previous outer; reassemble + deliver
4. Walk aggregated inner packets in payload; for each:
   - If complete (no fragment-marker on tail) → deliver via `xfrm_input` re-inject path
   - If fragmented at tail → buffer; await next outer datagram with matching Block Offset
5. Per-fragment timeout (`XFRMA_IPTFS_DROP_TIME`) drops orphan fragments

### Reorder window

`XFRMA_IPTFS_REORDER_WINDOW` (default per upstream, in number of out-of-order outer datagrams tolerable). Sequence numbers from ESP replay window order outer datagrams; reorder buffer holds out-of-order fragments until in-order delivery completes.

### Per-SA congestion-control mode (Subtype 1, CC)

Per RFC 9347 § 2.2.2: outer datagrams carry congestion-control feedback; sender adjusts emission rate. Optional; opt-in via per-SA flag.

### Tracepoints (`trace_iptfs.h`)

Per-SA tracepoints for: enqueue, frag-buffered, frag-completed, frag-dropped, outer-emitted, outer-received. Identical tracepoint surface so existing tracing tools (bpftrace, perf trace) work.

## Requirements

- REQ-1: IP-TFS Payload Header layout per RFC 9347 § 2.1: Subtype + rsv + Block Offset; byte-identical wire format.
- REQ-2: TX aggregation: per-SA TX queue (bounded by `XFRMA_IPTFS_MAX_QUEUE_SIZE`); periodic timer fires at configured rate; assembles outer datagrams of `XFRMA_IPTFS_PKT_SIZE`; pads with zero when underused.
- REQ-3: TX fragmentation: when next inner packet exceeds remaining outer-payload room AND `XFRMA_IPTFS_DONT_FRAG=0` → split across this outer and the next.
- REQ-4: TX DF mode: `XFRMA_IPTFS_DONT_FRAG=1` + oversize inner → drop with counter increment (cross-ref XfrmOutIptfsError equivalent).
- REQ-5: RX aggregation walk: read Payload Header → walk inner packets; deliver complete; buffer fragments.
- REQ-6: RX reassembly: per-`XFRMA_IPTFS_REORDER_WINDOW`-sized reorder buffer; per-fragment timeout `XFRMA_IPTFS_DROP_TIME`; orphan fragments dropped + counter (`XfrmInIptfsError`).
- REQ-7: Sequence-number-driven reorder: ESP replay window provides total-ordering across outer datagrams; reassembler uses ESN-aware seq comparisons.
- REQ-8: Per-SA initial-delay `XFRMA_IPTFS_INIT_DELAY` honored before first outer-datagram emission.
- REQ-9: Subtype 0 (basic IPTFS) supported; Subtype 1 (CC) DEFERRED for v0 (declared in this Tier-3's Open Questions or future work).
- REQ-10: `trace_iptfs.h` tracepoints emit at identical decision points; identical tracepoint signatures.
- REQ-11: Per-`/proc/net/xfrm_stat` counters: `XfrmInIptfsError` + `XfrmOutNoQueueSpace` (covers IPTFS queue-full case) bumped at decision points.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: IP-TFS Payload Header wire test: capture an outer ESP packet from an IP-TFS-enabled SA; bytes 0-3 match RFC 9347 § 2.1 layout. (covers REQ-1)
- [ ] AC-2: TX aggregation test: send 3 small (100-byte) packets through SA configured `pkt_size=1500` → kernel emits 1 outer of 1500 bytes containing all 3 inner packets + padding. (covers REQ-2)
- [ ] AC-3: TX fragmentation test: send a 2000-byte packet through SA `pkt_size=1500` + `dont_frag=0` → kernel splits across 2 outers; second outer's Block Offset > 0. (covers REQ-3)
- [ ] AC-4: TX DF mode test: same packet + `dont_frag=1` → kernel drops + counter increment. (covers REQ-4)
- [ ] AC-5: RX aggregation test: receive an outer with 3 inner packets aggregated → kernel delivers all 3 to the upper-layer in order. (covers REQ-5)
- [ ] AC-6: RX reassembly test: receive 2 outers carrying fragmented 2000-byte packet → kernel reassembles + delivers. (covers REQ-6)
- [ ] AC-7: RX out-of-order test: receive 3 outers in order [seq=2, seq=1, seq=3] within reorder window → kernel buffers seq=2, delivers in order seq=1, 2, 3. (covers REQ-7)
- [ ] AC-8: RX timeout test: receive only seq=1 of 2-fragment packet; wait `drop_time + 1s` → orphan fragment dropped + counter. (covers REQ-6)
- [ ] AC-9: Constant-rate test: input rate = 0; outer emission still proceeds at configured rate (with all-pad outer datagrams). (covers REQ-2)
- [ ] AC-10: Tracepoint test: `bpftrace -e 'tracepoint:xfrm:iptfs_outer_emitted { print(args); }'` shows per-emitted-outer event. (covers REQ-10)
- [ ] AC-11: Counter test: queue-full scenario → `XfrmOutNoQueueSpace++`; corrupted RX fragment → `XfrmInIptfsError++`. (covers REQ-11)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::xfrm::iptfs::Iptfs` — top-level
- `kernel::net::xfrm::iptfs::Tx` — TX aggregator (queue + assembler + timer)
- `kernel::net::xfrm::iptfs::TxQueue` — per-SA bounded TX queue
- `kernel::net::xfrm::iptfs::PayloadAssembler` — outer-payload pack
- `kernel::net::xfrm::iptfs::Fragmenter` — inner-packet fragmentation
- `kernel::net::xfrm::iptfs::Rx` — RX reassembler (per-SA reorder buffer)
- `kernel::net::xfrm::iptfs::Reassembler` — fragment reassembly
- `kernel::net::xfrm::iptfs::ReorderBuffer` — out-of-order outer storage
- `kernel::net::xfrm::iptfs::Timer` — periodic TX timer + per-fragment timeout
- `kernel::net::xfrm::iptfs::Trace` — tracepoint emitters

### Locking and concurrency

- **Per-SA `iptfs.lock`** (spinlock): protects per-SA TX queue + RX reorder buffer
- **Per-SA hrtimer**: drives periodic TX emission
- **Per-fragment timeout timer**: drops orphan fragments
- **No new global locks**

### Error handling

- `Err(ENOBUFS)` — TX queue full → drop + XfrmOutNoQueueSpace++
- `Err(EBADMSG)` — RX malformed payload header (e.g., Block Offset > pkt_size)
- `Err(ETIMEDOUT)` — fragment-reassembly orphan timeout → drop
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| TX assembler outer-payload-build (no overrun on pad-to-pkt_size) | `kani::proofs::net::xfrm::iptfs::tx_safety` |
| TX fragmenter (split-byte-count = inner-len; no double-emit) | `kani::proofs::net::xfrm::iptfs::frag_safety` |
| RX reassembler buffer (no out-of-bounds-read on Block Offset) | `kani::proofs::net::xfrm::iptfs::reasm_safety` |
| Reorder buffer insert/erase under iptfs.lock | `kani::proofs::net::xfrm::iptfs::reorder_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/xfrm_replay.tla` for sequence ordering)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| TX queue | length ≤ XFRMA_IPTFS_MAX_QUEUE_SIZE; FIFO ordered | `kani::proofs::net::xfrm::iptfs::txq_invariants` |
| Reorder buffer | window holds at most REORDER_WINDOW out-of-order outers; in-order outers delivered immediately | `kani::proofs::net::xfrm::iptfs::reorder_invariants` |
| Fragment buffer | ∀ buffered fragment, holding-time ≤ DROP_TIME (else drop) | `kani::proofs::net::xfrm::iptfs::frag_buf_invariants` |

### Layer 4: Functional correctness (opt-in)

- **TFC traffic-pattern theorem** via Verus — proves: outer-datagram emission rate is independent of inner packet rate (sender-side) when input ≤ configured rate; outer-datagram size is constant `pkt_size`.
- **Reassembly soundness theorem** via Verus — proves: ∀ fragmented inner packet `p`, reassembled output equals `p` byte-for-byte; never delivers spurious bytes.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | freed-after-emit TX-queue skbs cleared (carries pre-encrypt plaintext); freed reassembly buffers cleared | § Default-on configurable off |
| **SIZE_OVERFLOW** | Block Offset + fragment-length arithmetic uses checked operators (CVE class: malformed Block Offset → out-of-bounds read) | § Mandatory |
| **LATENT_ENTROPY** | padding bytes seeded from kernel CSPRNG (defense vs. fingerprinting via constant-pad-pattern; per RFC 9347 § 6.6 SHOULD use random padding) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW, LATENT_ENTROPY**: see above
- **CONSTIFY**: per-SA IPTFS ops vtable `static const`
- **USERCOPY**: NLA parsing for XFRMA_IPTFS_* uses bound-checked accessors

### Row-2 / GR-RBAC integration

- LSM hook: none directly (per-SA configuration; LSM gate at NETLINK_XFRM mutation level — `user.md`).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds rtnl/NETLINK_XFRM `XFRMA_IPTFS_*` payload copyin so attacker-supplied IPTFS knobs cannot OOB the parser.
- **PAX_KERNEXEC** — keeps the `static const` IPTFS per-SA ops vtable and packetizer/depacketizer dispatch text W^X.
- **PAX_RANDKSTACK** — randomises stack per `iptfs_input`/`iptfs_output` entry; defeats ROP under aggressive aggregator-flood inputs.
- **PAX_REFCOUNT** — wraps the per-IPTFS-SA refcounts (TX queue + RX reassembly state) against UAF on SA delete during reassembly.
- **PAX_MEMORY_SANITIZE** — zeroes freed TX-queue skbs (pre-encrypt plaintext) and freed reassembly buffers on free; padding-byte buffers wiped too.
- **PAX_UDEREF** — protects netlink-driven IPTFS configuration paths from userland deref on malformed `XFRMA_IPTFS_*`.
- **PAX_RAP / kCFI** — protects indirect dispatch through the IPTFS ops vtable and the underlying ESP `xfrm_type->output` it wraps.
- **GRKERNSEC_HIDESYM** — hides `iptfs_*`, `xfrm_iptfs_*` symbols from unprivileged readers.
- **GRKERNSEC_DMESG** — restricts dmesg so IPTFS reassembly errors (Block Offset / fragment-length) don't leak SA SPIs or framing details.
- **IPsec SAD/SPD CAP_NET_ADMIN** — IPTFS-enabled SAs install only via the CAP_NET_ADMIN-gated XFRM netlink path; an unprivileged user cannot enable aggregation on a foreign SA.
- **XFRM key MEMORY_SANITIZE** — IPTFS itself does not hold SA keys; the AEAD/ESP key buffers it consumes are wiped by `ah-esp-*` and `state.md` on SA destroy.
- **Replay-window protected** — IPTFS replay (Block Offset uniqueness) is checked under `x->lock`; PAX_MEMORY_SANITIZE wipes the window on SA free.
- **Block Offset SIZE_OVERFLOW** — Block Offset + fragment-length arithmetic is range-checked so a malformed offset cannot OOB-read into a neighbouring reassembly buffer.
- **LATENT_ENTROPY-padded bytes** — padding bytes are seeded from the kernel CSPRNG (RFC 9347 § 6.6 SHOULD), defeating fingerprinting via constant pad patterns.

Rationale: IPTFS is the newest XFRM tunnel feature (RFC 9347) and reintroduces application-supplied length+offset arithmetic into the IPsec hot path; combining SIZE_OVERFLOW on Block Offset, MEMORY_SANITIZE on TX/reassembly buffers, REFCOUNT on per-SA state, and LATENT_ENTROPY-driven padding collapses the new attack surface to the same envelope as classic ESP.

## Open Questions

(none in v0 — IPTFS Subtype 0 fully specified; Subtype 1 CC mode marked as future work in REQ-9 since RFC 9347 § 2.2.2 is recently-finalized and not yet a deployment-blocker for typical IPSec installations)

## Out of Scope

- IP-TFS Subtype 1 (CC mode) — declared in REQ-9 as future-work for v0
- Underlying ESP transforms (cross-ref `net/xfrm/ah-esp-ipv4.md`, `net/xfrm/ah-esp-ipv6.md`)
- 32-bit-only paths
- Implementation code
