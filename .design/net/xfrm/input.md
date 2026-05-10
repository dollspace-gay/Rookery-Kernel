# Tier-3: net/xfrm/input — RX transform pipeline (decap → decrypt → policy-check → re-input)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_input.c
  - net/xfrm/xfrm_inout.h
  - net/ipv4/xfrm4_input.c
  - net/ipv6/xfrm6_input.c
  - include/net/xfrm.h
-->

## Summary
Tier-3 design for the XFRM RX pipeline: when a packet arriving as ESP/AH/IPCOMP is detected (via `inet_protos[IPPROTO_ESP]` etc. → `xfrm4_rcv` / `xfrm6_rcv`), the pipeline finds the matching SA via SPI, performs decapsulation + decryption, runs the per-protocol type handler (`x->type->input`), validates the anti-replay window, optionally walks more transforms (nested SA stacks), pushes through `__xfrm_policy_check` to validate against the configured IN policy, decrements skb's nf-conntrack-zone if applicable, and re-injects the inner packet via `netif_rx`-equivalent.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (SA lookup), `net/xfrm/policy.md` (policy check), `net/xfrm/replay.md` (sequence validation), `net/xfrm/output.md` (TX counterpart). Hot path for every IPSec-encapsulated received packet.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Generic RX pipeline: `xfrm_input`, `xfrm_input_resume`, async-completion handling, sub-skb-list walking | `net/xfrm/xfrm_input.c` |
| Internal inout struct definitions | `net/xfrm/xfrm_inout.h` |
| IPv4 ESP/AH/IPCOMP RX entry: `xfrm4_rcv`, `xfrm4_transport_finish`, `xfrm4_udp_encap_rcv` (NAT-T) | `net/ipv4/xfrm4_input.c` |
| IPv6 RX entry: `xfrm6_rcv`, `xfrm6_input_addr` | `net/ipv6/xfrm6_input.c` |
| Public API | `include/net/xfrm.h` |

## Compatibility contract

### RX entry: `inet_protos[IPPROTO_ESP/AH/IPCOMP] = xfrm4_rcv`

For IPv4: `inet_protos[]` registers `xfrm4_rcv` (which calls into generic `xfrm_input(skb, IPPROTO_ESP, spi=0, encap_type=0)`); same pattern for AH (50), ESP (51), IPCOMP (108), and UDP-encapsulated ESP (NAT-T).

For IPv6: `inet6_protos[]` registers `xfrm6_rcv` similarly.

Identical registration so packets dispatched to xfrm path identically.

### Generic RX pipeline (`xfrm_input`)

Algorithm:
1. Extract SPI from packet (per-proto: ESP at offset 4, AH at offset 4, IPCOMP at offset 2)
2. Lookup SA: `xfrm_state_lookup(daddr, spi, proto, family)`
3. If no SA found → drop + counter `XfrmInNoStates++`
4. Acquire `x->lock`
5. Validate SA state == VALID; on EXPIRED → drop + `XfrmInStateExpired++`
6. Validate per-SA selector against skb (saddr/daddr/dport/sport/ipproto)
7. Anti-replay check via `xfrm_replay_check` (cross-ref `net/xfrm/replay.md`)
8. Async-capable: call `x->type->input(x, skb)`; if returns `-EINPROGRESS` (HW offload or async crypto), pause pipeline + resume on completion
9. On crypto success: advance replay window via `xfrm_replay_advance`
10. Walk inner header; if it's another XFRM proto → loop (nested SAs)
11. Final `__xfrm_policy_check(skb, dir=IN, family)` against IN policy
12. Re-inject inner packet via per-AF `transport_finish` → `netif_rx_ni` / `ip_local_deliver_finish` / `ip6_input_finish`

Identical sequence + counters at every decision point.

### `xfrm_input_resume` async completion

When `x->type->input` returns `-EINPROGRESS` (HW offload), driver completion calls `xfrm_input_resume(skb, err)` to restart from step 9. The `XFRM_DEV_RESUME` skb-cb flag tracks resumed state.

Identical async semantics so HW-offload paths re-enter sync state predictably.

### NAT-Traversal RX (`xfrm4_udp_encap_rcv`)

UDP socket configured with `UDP_ENCAP_ESPINUDP_NON_IKE` or `UDP_ENCAP_ESPINUDP` (RFC 3948) callback `xfrm4_udp_encap_rcv`:
1. Strip UDP header
2. Detect non-IKE marker (IKE 4-byte non-ESP marker `0x00 0x00 0x00 0x00`)
3. If ESP-in-UDP: redirect skb to `xfrm4_rcv` with proto IPPROTO_ESP

Identical algorithm so strongSwan/libreswan NAT-T tunnels work unchanged.

### Per-AF IN-direction adapters

| AF | Path |
|---|---|
| IPv4 | `xfrm4_rcv` → `xfrm_input` → … → `xfrm4_transport_finish` → `ip_local_deliver_finish` |
| IPv6 | `xfrm6_rcv` → `xfrm_input` → … → `xfrm6_transport_finish` → `ip6_input_finish` |

Identical chain.

### Counters bumped (per `/proc/net/xfrm_stat`)

`XfrmInError`, `XfrmInBufferError`, `XfrmInHdrError`, `XfrmInNoStates`, `XfrmInStateProtoError`, `XfrmInStateModeError`, `XfrmInStateSeqError`, `XfrmInStateExpired`, `XfrmInStateMismatch`, `XfrmInStateInvalid`, `XfrmInTmplMismatch`, `XfrmInNoPols`, `XfrmInPolBlock`, `XfrmInPolError`, `XfrmInStateDirError`, `XfrmInIptfsError`. Format byte-identical.

## Requirements

- REQ-1: RX entry registration: `xfrm4_rcv` for IPv4 + `xfrm6_rcv` for IPv6 registered in `inet_protos[]`/`inet6_protos[]` for IPPROTO_AH/ESP/IPCOMP.
- REQ-2: Generic `xfrm_input` pipeline: SPI extract → SA lookup → SA-state validate → selector match → replay check → type->input → replay advance → nested walk → policy-check → re-inject; identical sequence.
- REQ-3: SA lookup: `xfrm_state_lookup(daddr, spi, proto, family)`; on miss → drop + XfrmInNoStates++.
- REQ-4: SA-state validate: VALID gate; EXPIRED → drop + XfrmInStateExpired++; INVALID → drop + XfrmInStateInvalid++.
- REQ-5: Selector validate: per-SA selector matched against skb's actual fields; mismatch → drop + XfrmInStateMismatch++.
- REQ-6: Anti-replay check: per-SA replay window (cross-ref `net/xfrm/replay.md`); on duplicate or out-of-window → drop + XfrmInStateSeqError++.
- REQ-7: Async crypto completion: `x->type->input` returning `-EINPROGRESS` pauses pipeline; `xfrm_input_resume` restarts at advance step.
- REQ-8: Nested-SA support: after inner-header parse, if inner is another XFRM proto → loop pipeline up to `XFRM_MAX_DEPTH=6` times.
- REQ-9: `__xfrm_policy_check`: per IN policy validation per `net/xfrm/policy.md` REQ-7; counters per matrix.
- REQ-10: NAT-T RX: `xfrm4_udp_encap_rcv` strips UDP, detects non-IKE marker, redirects to ESP path; identical.
- REQ-11: Per-AF transport-finish: `xfrm4_transport_finish` → `ip_local_deliver_finish`; `xfrm6_transport_finish` → `ip6_input_finish`.
- REQ-12: Counter set per `/proc/net/xfrm_stat` bumped at identical decision points; format byte-identical.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: ESP RX test: send a correctly-encrypted ESP packet with valid SPI + valid replay seq → kernel decrypts + delivers inner packet to upper layer; tcpdump on inner shows plaintext. (covers REQ-1, REQ-2, REQ-3, REQ-6)
- [ ] AC-2: Bad-SPI test: send ESP packet with unknown SPI → drop + `XfrmInNoStates++`. (covers REQ-3)
- [ ] AC-3: Bad-MAC test: send ESP packet with corrupted authenticator → `x->type->input` returns -EBADMSG → drop + `XfrmInError++`. (covers REQ-2)
- [ ] AC-4: Replay test: send a packet, replay it → drop + `XfrmInStateSeqError++`. (covers REQ-6)
- [ ] AC-5: Selector mismatch test: SA selector requires saddr=`10.0.0.1`; receive ESP from saddr=`10.0.0.2` → drop + `XfrmInStateMismatch++`. (covers REQ-5)
- [ ] AC-6: Async crypto test: HW-offload-capable cipher returns `-EINPROGRESS`; completion fires `xfrm_input_resume` with success → packet delivered. (covers REQ-7)
- [ ] AC-7: Nested-SA test: ESP-over-AH wrapped packet → both layers decrypt; inner packet delivered. (covers REQ-8)
- [ ] AC-8: Policy-check test: SA matches but no IN policy → drop + `XfrmInNoPols++`. (covers REQ-9)
- [ ] AC-9: NAT-T RX test: UDP-encapsulated ESP packet on port 4500 → kernel strips UDP, processes as ESP, delivers inner. (covers REQ-10)
- [ ] AC-10: `cat /proc/net/xfrm_stat` byte-identical content after equivalent traffic. (covers REQ-12)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::xfrm::input::XfrmInput` — generic pipeline entrypoint
- `kernel::net::xfrm::input::SpiExtractor` — per-proto SPI extraction
- `kernel::net::xfrm::input::SaResolve` — SA lookup + state-validate + selector-validate
- `kernel::net::xfrm::input::AsyncCrypto` — `-EINPROGRESS` pause/resume
- `kernel::net::xfrm::input::NestedWalker` — XFRM_MAX_DEPTH-bounded nested SA loop
- `kernel::net::xfrm::input::PolicyValidate` — `__xfrm_policy_check` invocation
- `kernel::net::xfrm::input::ReInject` — per-AF transport_finish dispatch
- `kernel::net::xfrm::input::v4::Xfrm4Rcv` — IPv4 entry
- `kernel::net::xfrm::input::v6::Xfrm6Rcv` — IPv6 entry
- `kernel::net::xfrm::input::nat_t::Xfrm4UdpEncapRcv` — NAT-T RX strip
- `kernel::net::xfrm::input::counters::InCounters` — `XfrmIn*` bumper

### Locking and concurrency

- **Per-`xfrm_state` lock** (spinlock): held during state-validate + replay-check + replay-advance
- **No new global locks**: pipeline is per-skb; SA refcount inherited from `xfrm_state_lookup`
- **`xfrm_state_lock`** (rwlock, cross-ref `net/xfrm/state.md`): RCU-side lookup only

### Error handling

- `Err(EINVAL)` — bad SPI / bad selector / bad replay
- `Err(EBADMSG)` — crypto authenticator mismatch
- `Err(EAGAIN)` — async crypto in flight; pipeline paused
- `Err(ENOMEM)` — alloc fail
- skb dropped via kfree_skb_reason for telemetry

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| SPI extraction (skb-bounds-checked) | `kani::proofs::net::xfrm::input::spi_safety` |
| SA-state validate transition (no read-after-DEAD) | `kani::proofs::net::xfrm::input::state_validate_safety` |
| Async-resume re-entry (no double-advance of replay window) | `kani::proofs::net::xfrm::input::async_safety` |
| Nested-walker depth bound (≤ XFRM_MAX_DEPTH iterations) | `kani::proofs::net::xfrm::input::nested_safety` |
| Policy-check SA-stack walk | `kani::proofs::net::xfrm::input::policy_check_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/xfrm_replay.tla` from `net/xfrm/replay.md` and `models/net/xfrm_state_machine.tla` from `net/xfrm/state.md` + `models/net/xfrm_policy_eval.tla` from `net/xfrm/policy.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Pipeline state machine | exactly one of {SYNC, IN_PROGRESS, RESUMED} at any observable instant; transitions are SYNC→IN_PROGRESS→RESUMED→{SYNC} | `kani::proofs::net::xfrm::input::state_invariants` |
| Nested-walker depth | walker depth ≤ XFRM_MAX_DEPTH at every loop iteration | `kani::proofs::net::xfrm::input::nested_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/xfrm/00-overview.md` Layer 4)

- **RX pipeline soundness theorem** via TLA+ refinement — proves: every accepted packet traversed all required pipeline stages (SA-resolve, replay, type->input, advance, policy-check) in the right order; no stage skipped.
- **Async-resume idempotence theorem** via Verus — proves: `xfrm_input_resume` invoked twice on same skb is a no-op on the second invocation (no double-advance, no double-deliver).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | freed-after-decrypt skbs cleared (decrypted plaintext is sensitive) | § Default-on configurable off |
| **SIZE_OVERFLOW** | nested-walker depth + skb-pull byte arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`)
- **CONSTIFY**: per-protocol-type input handler dispatch via `static const fn-ptr` array (per `xfrm_type` registration)
- **SIZE_OVERFLOW**: see above
- **KERNEXEC**: type->input dispatch via static const fn-ptr only

### Row-2 / GR-RBAC integration

- LSM hooks: `security_xfrm_decode_session` (already standard upstream); GR-RBAC policy can deny session-decode per-flow (default empty).
- Useful default GR-RBAC policy: empty so behavior matches upstream.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — RX pipeline semantics are exhaustively specified by upstream + RFCs 4302, 4303, 3173, 3948)

## Out of Scope

- TX pipeline (cross-ref `net/xfrm/output.md`)
- SADB (cross-ref `net/xfrm/state.md`)
- SPD (cross-ref `net/xfrm/policy.md`)
- Replay-window detail (cross-ref `net/xfrm/replay.md`)
- Per-protocol AH/ESP/IPCOMP transforms (cross-ref `net/xfrm/ah-esp-ipv4.md`, `net/xfrm/ah-esp-ipv6.md`)
- 32-bit-only paths
- Implementation code
