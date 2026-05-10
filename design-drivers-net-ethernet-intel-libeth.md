---
title: "Tier-3: drivers/net/ethernet/intel/libeth/ — Shared Intel ethernet lib (rx + tx + xdp + xsk + page-pool integration)"
tags: ["tier-3", "drivers-net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The shared Intel-NIC library — `libeth` (NEW, post-2023) — provides per-queue page-pool RX path, generic TX completion handling, generic XDP_DRV processing (XDP_TX / XDP_REDIRECT / XDP_PASS / XDP_DROP / XDP_ABORTED), AF_XDP zerocopy infrastructure (tx-ring + completion-ring + fill-ring + rx-ring), and generic skb-build helpers. Used by multiple Intel drivers (idpf, libie consumers via libeth, future ice migration). Replaces per-driver duplicated boilerplate that was scattered across e1000e/igb/ixgbe/i40e/ice/iavf.

This Tier-3 covers `drivers/net/ethernet/intel/libeth/` (~5 files: `rx.c`, `tx.c`, `xdp.c`, `xsk.c`, `priv.h`).

### Acceptance Criteria

- [ ] AC-1: idpf driver loads + binds + carries traffic at line-rate on reference Intel Mount Evans IPU; per-queue page-pool stats visible via `ethtool -S`.
- [ ] AC-2: AF_XDP zerocopy test: `xdpsock --rx --tx -i ens0f0 --zero-copy` achieves pps within 5% of upstream baseline.
- [ ] AC-3: XDP_DRV test: `xdp-loader load -m native ens0f0 xdp_redirect.o` redirects packets between two interfaces.
- [ ] AC-4: NAPI poll-budget stress: 10G line-rate sustained → poll respects budget, ksoftirqd runs when over-budget; throughput within 2% upstream.
- [ ] AC-5: skb-tx + xdp-tx + xsk-tx mixed workload: descriptor pool correctly recycled across all 3 paths; no leak.
- [ ] AC-6: kselftest `tools/testing/selftests/drivers/net/intel/` passes.

### Architecture

`Fq` lives in `drivers::intel::libeth::Fq`:

```
struct Fq {
  pp: Arc<PagePool>,                  // per-NAPI page-pool
  fqes: VarLenArray<libeth_fqe>,      // per-fill-descriptor entries
  count: u32,
  buf_len: u32,
  truesize: u32,
  nid: NumaNode,
  napi: Arc<Napi>,
}
```

Per-RX-queue init `Fq::create(fq, napi)`:
1. `page_pool_create(&pool_params)` with: per-driver `pool_params.flags = PP_FLAG_DMA_MAP | PP_FLAG_PAGE_FRAG`, `pool_params.pool_size = 4096`, `pool_params.nid = numa_node_id()`, `pool_params.dma_dir = DMA_FROM_DEVICE`.
2. Allocate per-fqe array of `libeth_fqe` (one per RX descriptor).
3. Pre-fill: for each fqe, `Fq::alloc(fq, i)` → page-pool returns page, DMA-map auto-applied.

Per-RX-descriptor `Fq::alloc(fq, i)`:
1. `page_pool_dev_alloc_pages(fq.pp, &page, &dma_addr)` → returns page-fragment + DMA-mapped addr.
2. Store in `fq.fqes[i] = { page, dma_addr, offset, truesize }`.
3. Return `dma_addr` for HW descriptor write.

Per-NAPI poll `NapiRx::poll(napi, budget)`:
1. Loop until `processed >= budget` OR no more completed descriptors:
   - Read RX completion from HW descriptor.
   - Look up `fq.fqes[desc_idx]` → page + dma_addr.
   - If XDP prog attached: `Xdp::run_prog(prog, fqe, packet_len)`:
     - Build `xdp_buff` from fqe page.
     - `bpf_prog_run_xdp(prog, &xdp_buff)` → action (PASS / DROP / TX / REDIRECT / ABORTED).
     - PASS: continue to skb-build.
     - DROP / ABORTED: `page_pool_put_page(fq.pp, page, false)` → recycle.
     - TX: enqueue on per-NAPI XDP TX-ring (`XdpTx::tx_buff(...)`).
     - REDIRECT: `xdp_do_redirect(...)` → cross-ifindex via xdp_do_redirect.
   - Else (no XDP prog): build skb from fqe; deliver via `napi_gro_receive(napi, skb)`.
2. If processed == budget: re-arm interrupt + return budget (NAPI re-schedules); else return processed (NAPI completes).

XDP_REDIRECT path `XdpRedirect::flush_after_napi_poll`:
1. Per-cpu `bpf_redirect_info` set during prog run.
2. `xdp_do_flush_map()` flushes per-cpu redirect-batch to target ifindex's xmit queue.

AF_XDP zerocopy: when ndo_bpf with XDP_SETUP_XSK_POOL succeeds, per-RX-queue + per-TX-queue switched to xsk-mode:
- RX-queue: `XskRx::poll(...)` reads UMEM frame via `xsk_buff_alloc(pool)` instead of page-pool; descriptor enqueued on userspace RX-ring.
- TX-queue: `XskXmit::xmit(...)` consumes from userspace TX-ring; submits via HW TX descriptor; on TX completion, `xsk_tx_completed(...)` writes to userspace completion-ring.

TX completion `TxComplete::poll(napi, budget)`:
1. Walk per-TX-queue completion descriptors.
2. Per-buf type:
   - SKB: `dma_unmap_single(...)` + `napi_consume_skb(...)`.
   - XDP-FRAME: `page_pool_put_page(pp, page, true)` (recycle for re-use).
   - XSK: `xsk_tx_completed(pool, n)` (write to completion-ring).

### Out of Scope

- Per-driver implementation (covered in `intel-ice.md`, `intel-idpf.md`, etc. future Tier-3s)
- Page-pool internals (covered in `net/core/page_pool.md` future Tier-3 if added)
- AF_XDP UMEM management (covered in `net/xdp/00-overview.md` future Tier-2 if added)
- Generic XDP infra (covered in `kernel/bpf/00-overview.md` parent)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct libeth_fq_fp` | per-queue fast-path fill-queue context | `drivers::intel::libeth::FqFp` |
| `struct libeth_fq` | per-queue fill-queue config | `Fq` |
| `libeth_rx_fq_create(fq, napi)` / `_destroy(fq)` | per-queue page-pool init/teardown | `Fq::create` / `_destroy` |
| `libeth_rx_alloc(fq, i)` | alloc + DMA-map a page from page-pool for RX descriptor i | `Fq::alloc` |
| `libeth_rx_recycle_slow(...)` | slow-path recycle (e.g., when page-pool is busy) | `Fq::recycle_slow` |
| `libeth_rx_pt_*` family | per-pkt-type RSS hash + checksum descriptor parsing | `RxPtype::*` |
| `libeth_napi_rx_*` | NAPI-budgeted RX poll | `NapiRx::*` |
| `libeth_tx_complete_*` family | TX completion handling (skb consume + page-pool put_page or dma_unmap) | `TxComplete::*` |
| `libeth_xdp_run_prog(...)` | per-buffer XDP prog dispatch | `Xdp::run_prog` |
| `libeth_xdp_tx_*` family | XDP_TX path (re-enqueue on TX ring) | `XdpTx::*` |
| `libeth_xdp_redirect_*` | XDP_REDIRECT path (cross-ifindex via xdp_do_redirect) | `XdpRedirect::*` |
| `libeth_xsk_buff_*` family | AF_XDP buffer-pool integration | `XskBuff::*` |
| `libeth_xsk_xmit_*` | AF_XDP TX-ring driver-side dispatch | `XskXmit::*` |
| `libeth_xsk_rx_*` | AF_XDP RX-ring producer-side fill | `XskRx::*` |

### compatibility contract

REQ-1: Per-queue page-pool integration: per-NAPI page-pool with optional dma-pre-mapping; per-page recycle-on-skb-free; backbone of zero-alloc RX path.

REQ-2: NAPI poll-budget enforcement: per-poll bounded by `napi_budget`; over-budget reschedules via softirq + `napi_schedule`.

REQ-3: XDP-DRV native processing on every supported buffer type (page-frag from page-pool); per-prog return code (XDP_PASS / DROP / TX / REDIRECT / ABORTED) handled identically.

REQ-4: AF_XDP zerocopy: when bound via `XDP_USE_NEED_WAKEUP`, RX descriptor refers directly to a UMEM frame from userspace; TX descriptor consumes from userspace TX-ring; completion-ring entries written back.

REQ-5: TX-completion: skb-tx → DMA-unmap-then-skb-consume; xdp-tx → page-pool-recycle; xsk-tx → completion-ring write.

REQ-6: Per-pkt-type RSS hash + checksum descriptor parsing: per-Intel-driver descriptor format decoded via shared lookup tables (libeth_rx_pt_*).

REQ-7: Per-page reference-count integrity: page-pool refcount + skb fragment refcount maintained; page only returned to pool when both reach 0.

REQ-8: DMA-map error propagation: alloc-time DMA-map failure produces NULL alloc; per-driver RX path handles via descriptor reschedule.

REQ-9: Per-queue config (page-pool order + page-size + max-len) source-compat for in-tree consumers.

REQ-10: Tracepoints for XDP transitions (xdp_exception, xdp_redirect, xdp_redirect_err, xdp_redirect_map, xdp_redirect_map_err) byte-identical names + format.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fqe_idx_no_oob` | OOB | per-fqe array bounded by Fq::count; descriptor idx < count. |
| `pp_no_uaf` | UAF | page-pool drained before Fq destroy; per-page refcount tracked; no stale dma_addr after unmap. |
| `xdp_buff_no_oob` | OOB | xdp_buff data + data_end + data_meta + data_hard_start checked against page-frag bounds. |
| `xsk_pool_no_uaf` | UAF | xsk_buff_pool refcount held during xsk_buff_alloc; pool destroy waits for in-flight buffers. |

### Layer 2: TLA+

`models/net-eth/page_pool.tla` (parent-declared): proves per-NAPI page-pool refcount + recycle path; concurrent skb-free returning page to pool + NAPI-poll allocating from pool never produces double-recycle or leaked page.
`models/net-eth/napi_budget.tla` (parent-declared): proves NAPI poll-budget enforcement under sustained burst.
`models/net-eth/af_xdp_umem.tla` (parent-declared): proves AF_XDP UMEM fill-ring + completion-ring acq-rel between userspace producer/consumer and kernel NAPI.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fq::alloc` post: returned dma_addr is a valid DMA mapping; fqe[i].page held with ref-count ≥ 1 | `Fq::alloc` |
| `XdpTx::tx_buff` post: page placed on TX-ring with proper recycle marker; will be put back to page-pool on TX completion | `XdpTx::tx_buff` |
| `XskRx::poll` post: each completed RX descriptor has its UMEM frame addr written to userspace RX-ring; head advanced atomically | `XskRx::poll` |

### Layer 4: Verus/Creusot functional

`Fq::alloc + Fq::recycle` round-trip: a page allocated from the pool, used in skb, freed via skb-free's page-pool callback returns to the pool with refcount restored. Encoded as Verus invariant on per-page ref state.

### hardening

(Inherits row-1 features from `drivers/net/ethernet/00-overview.md` § Hardening.)

libeth-specific reinforcement:

- **Page-pool DMA-mask validation** — per-pool init validates `dma_mask` reaches required range; refuse pool create on insufficient mask (defense against silent swiotlb-bounce thrashing).
- **AF_XDP UMEM permission** — `bind(XDP_USE_NEED_WAKEUP)` requires CAP_NET_RAW + per-iface allowlist (defense against userspace stealing all RX from an interface).
- **Per-NAPI poll-budget cap** — total XDP_REDIRECT batch per-poll capped (default 64); defense against single-poll consuming entire system bandwidth.
- **Per-XDP-prog buffer mutation bounded** — XDP prog attempts to grow buffer past page-frag tailroom rejected; defense against driver-prog interaction causing OOB write.
- **Per-TX completion budget cap** — TX completion poll bounded; defense against TX-ring becoming unbounded backlog.
- **xdp_do_flush rate-limited** — per-cpu redirect-batch flush bounded; defense against XDP_REDIRECT-flood DoS to other ifindex.

