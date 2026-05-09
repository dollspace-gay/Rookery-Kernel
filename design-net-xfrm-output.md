---
title: "Tier-3: net/xfrm/output — TX transform pipeline (bundle → encap → encrypt → xmit)"
tags: ["design-doc", "tier-3", "net", "ipsec"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the XFRM TX pipeline: when a TX skb has an attached XFRM bundle (built by `net/xfrm/policy.md` `xfrm_lookup` on outgoing route resolution), the pipeline walks the bundle's compiled `dst_entry` chain, applying each per-stage `x->type->output` to perform encapsulation + encryption (ESP/AH/IPCOMP/IPTFS), advancing the per-SA replay counter (`xfrm_replay_overflow`), running NAT-T encapsulation if applicable (UDP-encap of ESP), and finally pushing through the per-AF xmit hook (`xfrm4_output_finish` / `xfrm6_output_finish`) into the standard transport-layer xmit path.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/input.md` (RX counterpart), `net/xfrm/state.md` (SA consumer), `net/xfrm/policy.md` (bundle producer), `net/xfrm/replay.md` (sequence emission).

### Requirements

- REQ-1: TX entry dispatch: per-AF `xfrm4_output` / `xfrm6_output` registered as dst_output for XFRM bundles.
- REQ-2: Generic `xfrm_output` pipeline: bundle-walk → per-stage state-validate → replay-seq-allocate → type->output → SA-curlft-bump; identical sequence.
- REQ-3: SA-state validate at TX: VALID gate; EXPIRED → drop + XfrmOutStateExpired++; INVALID → drop + XfrmOutStateInvalid++.
- REQ-4: Replay sequence allocation: per-stage `xfrm_replay_overflow`; on legacy-32-bit-rollover without ESN/rekey → drop + XfrmOutStateSeqError++.
- REQ-5: Async crypto completion: `x->type->output` returning `-EINPROGRESS` pauses pipeline; `xfrm_output_resume` restarts at next bundle stage.
- REQ-6: Bundle-chain walk: walk all `xfrm_dst` stages in order; each stage applies one transform; final stage hands off to per-AF output_finish.
- REQ-7: NAT-T encap: when SA's `encap` is set, prepend UDP header per RFC 3948 § 3.
- REQ-8: Soft + hard lifetime accounting: per-stage curlft bump; threshold checks emit XFRM_MSG_EXPIRE; hard threshold marks SA EXPIRED + drops subsequent packets.
- REQ-9: Per-AF output-finish: `xfrm4_output_finish` → `ip_output` regular xmit; `xfrm6_output_finish` → `ip6_output` regular xmit.
- REQ-10: Counter set per `/proc/net/xfrm_stat` bumped at identical decision points; format byte-identical.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: ESP TX test: with installed SA + matching OUT policy, sendmsg from socket → skb leaves NIC ESP-encapsulated; tcpdump on outer interface shows valid ESP w/ correct SPI + sequence number. (covers REQ-1, REQ-2, REQ-4, REQ-6)
- [ ] AC-2: Sequence-rollover test: with legacy 32-bit replay, send `2^32 - 1` packets through SA → next packet trips XfrmOutStateSeqError++; with ESN enabled, sequence rolls past 2^32 cleanly. (covers REQ-4)
- [ ] AC-3: Async crypto test: HW-offload-capable cipher returns `-EINPROGRESS`; completion fires `xfrm_output_resume` with success → packet xmits correctly. (covers REQ-5)
- [ ] AC-4: Bundle-chain test: ESP+IPCOMP nested SA bundle → tcpdump shows both transforms applied in order (IPCOMP outer, ESP inner of compress, IP inner of ESP). (covers REQ-6)
- [ ] AC-5: NAT-T TX test: SA with UDP_ENCAP_ESPINUDP → tcpdump shows UDP/4500 wrapping ESP. (covers REQ-7)
- [ ] AC-6: Soft-lifetime test: SA with `lifetime soft byte 1024` → after 1024 bytes through TX → daemon receives XFRM_MSG_EXPIRE soft; SA still valid. (covers REQ-8)
- [ ] AC-7: Hard-lifetime test: SA with `lifetime hard byte 2048` → after 2048 bytes through TX → SA EXPIRED; subsequent TX → XfrmOutStateExpired++. (covers REQ-8)
- [ ] AC-8: Per-AF dispatch test: v4 + v6 SAs in same netns; v4 packet → xfrm4_output; v6 packet → xfrm6_output (verifiable via /proc/net/xfrm_stat per-namespace counters). (covers REQ-1, REQ-9)
- [ ] AC-9: `cat /proc/net/xfrm_stat` byte-identical content after equivalent TX traffic. (covers REQ-10)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::xfrm::output::XfrmOutput` — generic TX pipeline entrypoint
- `kernel::net::xfrm::output::BundleWalker` — bundle-chain iteration
- `kernel::net::xfrm::output::StageApply` — per-stage state-validate + replay-allocate + type->output
- `kernel::net::xfrm::output::AsyncCrypto` — `-EINPROGRESS` pause/resume
- `kernel::net::xfrm::output::Lifetime` — curlft bump + soft/hard threshold check
- `kernel::net::xfrm::output::NatT` — UDP-encap wrap (RFC 3948)
- `kernel::net::xfrm::output::v4::Xfrm4Output` — IPv4 entry + finish
- `kernel::net::xfrm::output::v6::Xfrm6Output` — IPv6 entry + finish
- `kernel::net::xfrm::output::counters::OutCounters` — `XfrmOut*` bumper

### Locking and concurrency

- **Per-`xfrm_state` lock** (spinlock): held during state-validate + replay-allocate + curlft-bump
- **Per-skb pipeline state** (lockless via skb-cb): tracks resumed state across async crypto
- **No new global locks**

### Error handling

- `Err(EINVAL)` — bad bundle / bad SA
- `Err(EAGAIN)` — async crypto in flight; pipeline paused
- `Err(EOVERFLOW)` — replay-sequence rollover without ESN/rekey
- `Err(ENOMEM)` — alloc fail
- skb dropped via kfree_skb_reason for telemetry

### Out of Scope

- RX pipeline (cross-ref `net/xfrm/input.md`)
- SADB / SPD (cross-ref `net/xfrm/state.md`, `net/xfrm/policy.md`)
- Replay-window detail (cross-ref `net/xfrm/replay.md`)
- Per-protocol AH/ESP/IPCOMP transforms (cross-ref `net/xfrm/ah-esp-ipv4.md`, `net/xfrm/ah-esp-ipv6.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Generic TX pipeline: `xfrm_output`, `xfrm_output2`, `xfrm_output_resume`, async-completion handling | `net/xfrm/xfrm_output.c` |
| Internal inout struct definitions | `net/xfrm/xfrm_inout.h` |
| IPv4 TX entry: `xfrm4_output`, `xfrm4_output_finish`, `xfrm4_extract_output` | `net/ipv4/xfrm4_output.c` |
| IPv6 TX entry: `xfrm6_output`, `xfrm6_output_finish`, `xfrm6_extract_output` | `net/ipv6/xfrm6_output.c` |
| Public API | `include/net/xfrm.h` |

### compatibility contract

### TX entry: per-AF dst_output hook

Per-AF dst_output for an XFRM bundle is `xfrm4_output` (IPv4) / `xfrm6_output` (IPv6). When `ip_output` (or `ip6_output`) sees skb_dst_entry whose ops are XFRM-aware, it dispatches to xfrm output rather than direct device xmit.

Identical dispatch contract.

### Generic TX pipeline (`xfrm_output`)

Algorithm:
1. Validate skb header (`xfrm_extract_input` for the inner IP header)
2. Loop over bundle's chain of `xfrm_dst` entries:
   a. For each `x = bundle->xfrm_state`:
      - Acquire `x->lock`
      - Validate SA state == VALID; on EXPIRED → drop + counter `XfrmOutStateExpired++`
      - Allocate sequence number via `xfrm_replay_overflow`; on overflow + no rekey → drop + `XfrmOutStateSeqError++`
      - Async-capable: call `x->type->output(x, skb)`; if returns `-EINPROGRESS` (HW offload or async crypto), pause + resume
      - On crypto success: SA counters bumped (`x->curlft.bytes` / `x->curlft.packets`)
3. After all stages applied: per-AF `xfrm4_output_finish` / `xfrm6_output_finish` push into `dev_queue_xmit` chain
4. NAT-T (RFC 3948): if `x->encap` is set with UDP_ENCAP_ESPINUDP, wrap final ESP packet in UDP header on port 4500 before xmit

Identical sequence + counter at every decision point.

### `xfrm_output_resume` async completion

When `x->type->output` returns `-EINPROGRESS` (HW offload or async crypto), driver completion calls `xfrm_output_resume(skb, err)` to restart from next-stage in chain. The `XFRM_DEV_RESUME` skb-cb flag tracks resumed state.

Identical async semantics.

### NAT-T TX

When SA's `encap` member has `encap_type=UDP_ENCAP_ESPINUDP` (or `_NON_IKE`):
1. After ESP encap, prepend UDP header (sport/dport from encap config; default 4500)
2. UDP length = ESP-payload + 8
3. UDP checksum: 0 for v4 (RFC 3948 § 3 permits) / required nonzero per `udp_no_check6_tx` policy for v6

Identical wrap algorithm.

### Soft / hard lifetime checks at TX

Per-packet, post-stage:
- `x->curlft.bytes += skb->len`
- `x->curlft.packets += 1`
- If crosses `x->lft.soft_byte_limit` / `soft_packet_limit` → emit `XFRM_MSG_EXPIRE` (soft) once
- If crosses `x->lft.hard_byte_limit` / `hard_packet_limit` → mark SA EXPIRED + emit `XFRM_MSG_EXPIRE` (hard) + drop subsequent packets via early `XfrmOutStateExpired` check

Identical lifetime accounting.

### Counters bumped (per `/proc/net/xfrm_stat`)

`XfrmOutError`, `XfrmOutBundleGenError`, `XfrmOutBundleCheckError`, `XfrmOutNoStates`, `XfrmOutStateProtoError`, `XfrmOutStateModeError`, `XfrmOutStateSeqError`, `XfrmOutStateExpired`, `XfrmOutPolBlock`, `XfrmOutPolDead`, `XfrmOutPolError`, `XfrmFwdHdrError`, `XfrmOutStateInvalid`, `XfrmOutStateDirError`, `XfrmOutNoQueueSpace`. Format byte-identical.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Bundle-walker iteration (depth-bounded by XFRM_MAX_DEPTH) | `kani::proofs::net::xfrm::output::bundle_safety` |
| Per-stage state-validate (no read-after-DEAD) | `kani::proofs::net::xfrm::output::state_safety` |
| Async-resume re-entry (no double-allocate of replay seq) | `kani::proofs::net::xfrm::output::async_safety` |
| NAT-T UDP-header push (skb-bounds-checked) | `kani::proofs::net::xfrm::output::nat_t_safety` |
| Lifetime curlft-bump arithmetic (no overflow on 64-bit byte counter) | `kani::proofs::net::xfrm::output::lifetime_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/xfrm_replay.tla` from `net/xfrm/replay.md` and `models/net/xfrm_state_machine.tla` from `net/xfrm/state.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Pipeline state machine | exactly one of {INIT, IN_PROGRESS, RESUMED, DONE} at any observable instant; transitions are forward-only | `kani::proofs::net::xfrm::output::state_invariants` |
| Per-bundle xfrm chain | depth ≤ XFRM_MAX_DEPTH; every stage's SA refcount > 0 throughout walk | `kani::proofs::net::xfrm::output::bundle_invariants` |
| Per-SA curlft accounting | post-TX `curlft.bytes` ≥ pre-TX value (monotonic); `curlft.packets ≥ pre-TX + 1` after successful TX | `kani::proofs::net::xfrm::output::lifetime_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/xfrm/00-overview.md` Layer 4)

- **TX pipeline soundness theorem** via TLA+ refinement — proves: every emitted packet traversed all bundle stages in order; no stage skipped; per-SA seq allocated exactly once per packet.
- **Async-resume idempotence theorem** via Verus — proves: `xfrm_output_resume` invoked twice on same skb is a no-op on second invocation (no double-emit).

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | freed-after-encrypt skbs cleared (pre-encrypt plaintext is sensitive) | § Default-on configurable off |
| **SIZE_OVERFLOW** | curlft byte counter + replay-seq arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY**: per-protocol-type output handler dispatch via `static const fn-ptr` array
- **SIZE_OVERFLOW**: see above
- **KERNEXEC**: type->output dispatch via static const fn-ptr only

### Row-2 / GR-RBAC integration

- LSM hooks: `security_xfrm_decode_session` (already standard upstream); GR-RBAC policy can deny session-decode per-flow (default empty).
- Useful default GR-RBAC policy: empty so behavior matches upstream.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

