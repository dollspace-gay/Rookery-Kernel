# Tier-3: net/xfrm/replay — anti-replay window (RFC 4302/4303 § 3.4.3 + RFC 4304 ESN)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_replay.c
  - include/net/xfrm.h
  - include/uapi/linux/xfrm.h
-->

## Summary
Tier-3 design for the per-SA anti-replay window: the bitmap-based sliding window that tracks accepted sequence numbers per SA, supporting three modes — legacy 32-bit replay (RFC 4302/4303 § 3.4.3), Extended Sequence Number (ESN — RFC 4304 § 2.2.1) for 64-bit sequences, and a no-replay-protection mode for explicit-DTLS-style or transport-only deployments. Owns the mandatory `models/net/xfrm_replay.tla` TLA+ model that proves window-monotonicity + no-double-deliver invariants.

Three behaviors per `xfrm_replay`:
- **`xfrm_replay_legacy_*`**: 32-bit seq + 32-bit-window bitmap; when seq wraps around `2^32` without rekey → SA expires
- **`xfrm_replay_bmp_*`**: 64-bit ESN; per-RFC 4304 the high-32-bits are implicit; window can be configured up to 4096 bits
- **`xfrm_replay_esn_*`**: 64-bit ESN with explicit ESN tracking; same window-bitmap mechanics

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (replay state lives in xfrm_state), `net/xfrm/input.md` (RX consumer of replay-check), `net/xfrm/output.md` (TX consumer of replay-allocate).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Three replay-modes registration + per-mode check/advance/notify | `net/xfrm/xfrm_replay.c` |
| Public API (struct xfrm_replay_state / xfrm_replay_state_esn) | `include/net/xfrm.h` |
| UAPI (XFRMA_REPLAY_VAL / XFRMA_REPLAY_ESN_VAL / XFRMA_REPLAY_THRESH / XFRMA_REPLAY_THRESH_BYTES) | `include/uapi/linux/xfrm.h` |

## Compatibility contract

### `struct xfrm_replay_state` (legacy 32-bit)

```c
struct xfrm_replay_state {
    __u32 oseq;          /* outgoing seq next to use */
    __u32 seq;           /* incoming seq highest accepted */
    __u32 bitmap;        /* sliding window of recently-accepted seqs (32-bit) */
};
```

Layout-byte-identical.

### `struct xfrm_replay_state_esn` (RFC 4304)

```c
struct xfrm_replay_state_esn {
    unsigned int bmp_len;     /* in 32-bit words; max 128 (4096 bits) */
    __u32        oseq;        /* low 32 bits of outgoing seq */
    __u32        seq;         /* low 32 bits of incoming highest-accepted */
    __u32        oseq_hi;     /* high 32 bits of outgoing seq (ESN) */
    __u32        seq_hi;      /* high 32 bits of incoming highest-accepted (ESN) */
    __u32        replay_window; /* configured window size in bits */
    __u32        bmp[];       /* sliding window bitmap, length = bmp_len 32-bit words */
};
```

Layout-byte-identical with a flexible array member.

### Replay-mode operations

Each mode registers a `struct xfrm_replay`:
```c
struct xfrm_replay {
    void (*advance)(struct xfrm_state *x, __be32 net_seq);
    int  (*check)(struct xfrm_state *x, struct sk_buff *skb, __be32 net_seq);
    int  (*recheck)(struct xfrm_state *x, struct sk_buff *skb, __be32 net_seq);
    void (*notify)(struct xfrm_state *x, int event);
    int  (*overflow)(struct xfrm_state *x, struct sk_buff *skb);
};
```

- **`check`**: validates incoming seq is within window + not already accepted; returns 0 (accept) or `-EINVAL` (replay/out-of-window)
- **`recheck`**: post-decrypt re-validation (in case of async crypto reordering)
- **`advance`**: slides window forward + sets bit for newly-accepted seq
- **`notify`**: emits `XFRM_MSG_NEWAE` (anti-replay event) when AE event fires (per `XFRMA_REPLAY_THRESH`)
- **`overflow`**: TX-side sequence allocator + rollover detection

Identical per-mode method tables.

### `XFRMA_REPLAY_THRESH` (anti-replay event threshold)

Per RFC 4304 § 6: when accepted-seq advances past threshold (default `(window_size * 4) / 5`), kernel emits `XFRM_MSG_NEWAE` event to userspace daemon with current `seq`/`seq_hi`/`bitmap` (so daemon can persist replay state across IKE rekeys). Threshold tunable via `XFRMA_REPLAY_THRESH` NLA on NEWSA/UPDSA.

Identical algorithm.

### `XFRMA_REPLAY_THRESH_BYTES` (event threshold by bytes)

Alternative to seq-count threshold: fire AE event after N bytes processed. Used by HW-offload SAs to coordinate hardware-counter sync with software view. Identical UAPI.

### Window size selection

| Mode | Default window | Configurable |
|---|---|---|
| Legacy | 32 bits | no (bitmap is single u32) |
| ESN bmp | 32 bits | yes via `replay_window`; max 128*32 = 4096 bits |
| No-replay | n/a | n/a |

Identical defaults.

### Sysctl `/proc/sys/net/core/xfrm_aevent_etime`, `xfrm_aevent_rseqth`

- `xfrm_aevent_etime` (default 10): AE-event time threshold in 1/100 seconds
- `xfrm_aevent_rseqth` (default 2): AE-event seq-delta threshold (per-RFC default `(window*4)/5` for window=32 → ~25, sysctl overrides to 2 for tighter monitoring)

Format byte-identical.

## Requirements

- REQ-1: Three replay modes implemented identically: legacy (32-bit), ESN-bmp (64-bit ESN with bitmap), no-replay.
- REQ-2: `struct xfrm_replay_state` + `struct xfrm_replay_state_esn` layout-byte-identical.
- REQ-3: Per-mode `check` / `recheck` / `advance` / `notify` / `overflow` methods registered in `struct xfrm_replay`; per-SA dispatch via `x->repl`.
- REQ-4: RX `check`: reject seq < (highest_accepted - window + 1), reject already-set bit in bitmap; otherwise accept.
- REQ-5: RX `advance`: shift bitmap left by `(net_seq - highest_accepted)` if forward, set bit at appropriate offset; update `seq` / `seq_hi`.
- REQ-6: TX `overflow`: allocate next seq via `oseq++`; on legacy seq wrap (`oseq == 0`) without rekey → return -EOVERFLOW + SA marked DEAD; on ESN, `oseq_hi++` carry on `oseq` wrap.
- REQ-7: AE notification: when `seq - last_notify_seq` ≥ `XFRMA_REPLAY_THRESH` (or after `xfrm_aevent_etime` time elapsed) → emit `XFRM_MSG_NEWAE` to per-netns notify socket with current state.
- REQ-8: ESN window up to 4096 bits (`bmp_len ≤ 128`); identical bound.
- REQ-9: TLA+ model `models/net/xfrm_replay.tla` (mandatory per `net/xfrm/00-overview.md` Layer 2) — proves: ∀ accepted seq `s`, no later check accepts `s` again (no-double-deliver); window left edge is monotonically non-decreasing; per-SA seq allocator never returns same seq twice.
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct xfrm_replay_state` + `struct xfrm_replay_state_esn` byte-identical layout. (covers REQ-2)
- [ ] AC-2: Legacy mode replay test: deliver seq 1..100, then deliver seq 50 again → check returns -EINVAL + counter `XfrmInStateSeqError++`. (covers REQ-1, REQ-4)
- [ ] AC-3: ESN mode test: SA with ESN window=64; deliver seq 1..2^32-1, then 2^32 (causes seq_hi=1, seq=0); subsequent delivery of seq 2^32+1 succeeds. (covers REQ-1, REQ-4, REQ-5)
- [ ] AC-4: Out-of-window test: deliver seq 1000, then deliver seq 500 (window=32) → -EINVAL since 1000-32+1=969 > 500. (covers REQ-4)
- [ ] AC-5: Forward-jump test: deliver seq 1000, then seq 1100 → bitmap shifted, bit at offset 0 set, seq_high becomes 1100. (covers REQ-5)
- [ ] AC-6: TX seq-rollover test (legacy): TX 2^32 packets through SA → next TX returns -EOVERFLOW + SA DEAD; without ESN no further TX. (covers REQ-6)
- [ ] AC-7: TX ESN rollover test: SA with ESN; TX 2^32+1 packets → seq_hi increments correctly to 1, oseq=1 for the (2^32+1)-th. (covers REQ-6)
- [ ] AC-8: AE-event test: install SA with `XFRMA_REPLAY_THRESH=10`; receive 11 packets → daemon receives `XFRM_MSG_NEWAE` with current state. (covers REQ-7)
- [ ] AC-9: AE-event time test: install SA with `xfrm_aevent_etime=1`; idle for >1×10ms → daemon receives time-driven AE event. (covers REQ-7)
- [ ] AC-10: ESN max-window test: configure `replay_window=4096` (bmp_len=128); accept 4096 contiguous seqs; verify all bits set in bmp[]. (covers REQ-8)
- [ ] AC-11: TLA+ `models/net/xfrm_replay.tla` proves: no-double-deliver invariant + window-monotonicity invariant + per-SA seq-uniqueness invariant. (covers REQ-9)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::xfrm::replay::Replay` — generic dispatch (per `x->repl`)
- `kernel::net::xfrm::replay::Legacy` — `xfrm_replay_legacy_*`
- `kernel::net::xfrm::replay::Bmp` — `xfrm_replay_bmp_*` (ESN with bitmap)
- `kernel::net::xfrm::replay::Esn` — `xfrm_replay_esn_*` (ESN with explicit hi tracking)
- `kernel::net::xfrm::replay::NoReplay` — no-protection mode
- `kernel::net::xfrm::replay::Notify` — AE-event emitter
- `kernel::net::xfrm::replay::Window` — bitmap-window primitives (shared)

### Locking and concurrency

- **Per-`xfrm_state` `lock`** (spinlock): held during check + advance (atomicity required for window mutation)
- **TX-side `oseq` allocation**: lockless via `atomic_add_return` on most modes (legacy uses x->lock since `oseq+bitmap` mutation is paired)

### Error handling

- `Err(EINVAL)` — seq out of window OR replay (already accepted)
- `Err(EOVERFLOW)` — TX seq rollover without ESN/rekey
- `Err(ENOBUFS)` — AE-event netlink emission failed (counter only; non-fatal)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Bitmap window shift arithmetic (no out-of-bounds bit access) | `kani::proofs::net::xfrm::replay::shift_safety` |
| Per-mode dispatch (no NULL fn-ptr deref) | `kani::proofs::net::xfrm::replay::dispatch_safety` |
| TX `overflow` rollover detection | `kani::proofs::net::xfrm::replay::overflow_safety` |
| ESN seq_hi carry on oseq wrap | `kani::proofs::net::xfrm::replay::esn_carry_safety` |
| AE-event emit (skb alloc + socket send under spinlock) | `kani::proofs::net::xfrm::replay::ae_emit_safety` |

### Layer 2: TLA+ models

- `models/net/xfrm_replay.tla` (mandatory per `net/xfrm/00-overview.md` Layer 2) — proves window-monotonicity + no-double-deliver + per-SA seq-uniqueness. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Replay window bitmap | bit at offset `i` set ⇔ seq `(highest_accepted - i)` was accepted; `0 ≤ i < window_size` | `kani::proofs::net::xfrm::replay::bitmap_invariants` |
| ESN seq_hi/seq pair | `(seq_hi << 32) | seq` is monotonically non-decreasing across accepts | `kani::proofs::net::xfrm::replay::esn_invariants` |
| TX `oseq` | `oseq` strictly increases per allocation; on legacy wrap → SA DEAD | `kani::proofs::net::xfrm::replay::oseq_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/xfrm/00-overview.md` Layer 4)

- **Anti-replay refinement theorem** via TLA+ refinement — proves: implementation refines abstract sliding-window model; ∀ packet `p`, `p` accepted exactly once.
- **ESN sequence-uniqueness theorem** via Verus — proves: under cooperative seq_hi++ on oseq wrap, ESN sequence space is `2^64` unique values per SA.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | seq + seq_hi + bitmap-offset arithmetic uses checked operators (CVE class: ESN-rollover misordering) | § Mandatory |
| **MEMORY_SANITIZE** | freed `xfrm_replay_state_esn` cleared (replay state may leak SA-traffic patterns) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: state struct allocated within parent xfrm_state (cross-ref `net/xfrm/state.md`)
- **CONSTIFY**: per-mode `struct xfrm_replay` instances `static const`
- **SIZE_OVERFLOW**: see above
- **KERNEXEC**: per-mode dispatch via static const fn-ptr only

### Row-2 / GR-RBAC integration

- LSM hook: none directly (replay state mutation is internal kernel transformation; AE-event is delivered to daemon's NETLINK_XFRM socket which has its own LSM gate).
- Default GR-RBAC policy: empty so behavior matches upstream.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — anti-replay window semantics are exhaustively specified by RFC 4302/4303 § 3.4.3 + RFC 4304)

## Out of Scope

- SADB (cross-ref `net/xfrm/state.md`)
- RX/TX integration (cross-ref `net/xfrm/input.md`, `net/xfrm/output.md`)
- 32-bit-only paths
- Implementation code
