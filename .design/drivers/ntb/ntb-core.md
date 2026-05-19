# Tier-3: drivers/ntb/{core,ntb_transport}.c — Non-Transparent Bridge core + transport-over-MW

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/ntb/core.c
  - drivers/ntb/ntb_transport.c
  - drivers/ntb/msi.c
  - drivers/ntb/hw/
  - include/linux/ntb.h
  - include/linux/ntb_transport.h
-->

## Summary

A Non-Transparent Bridge (NTB) is a PCIe bridge that exposes the *other side*'s memory as a translated PCIe BAR on the local side, while terminating PCIe configuration / enumeration locally so neither host sees the other host's PCIe topology. The kernel framework abstracts the per-vendor NTB (Intel SkyLake/IceLake, AMD, Switchtec, Idt, EPF) behind `struct ntb_dev_ops`, exposes the primitives every NTB exposes — scratchpad registers (per-side small message bytes), doorbell bits (per-side IPI), and Memory Windows (per-side BARs that translate to remote-side physical addresses) — and supplies one in-tree client (`ntb_transport`) that builds a virtual point-to-point packet pipe on top: doorbells signal queue-pair arrival, scratchpads negotiate queue layout, MWs hold the shared ring buffers.

Sources: `core.c` (~317 lines, `ntb_bus_type`, client/device registration, default port helpers), `ntb_transport.c` (~2566 lines, virtual queue-pair transport, RX/TX rings with DMA engine offload, link-state machine), `msi.c` (MSI-X across NTB for per-MW interrupts), `hw/{intel,amd,switchtec,idt,epf}/` (per-vendor backends), `test/` (ntb_perf, ntb_pingpong, ntb_tool).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ntb_dev` (`include/linux/ntb.h`) | per-NTB device control block | `drivers::ntb::NtbDev` |
| `struct ntb_client` | NTB-bus client driver | `drivers::ntb::NtbClient` |
| `struct ntb_dev_ops` | per-vendor backend vtable (`port_number`, `peer_port_count`, `link_is_up`, `link_enable`, `link_disable`, `mw_count`, `mw_get_align`, `mw_set_trans`, `mw_clear_trans`, `peer_mw_count`, `peer_mw_get_addr`, `peer_mw_set_trans`, `db_valid_mask`, `db_vector_count`, `db_vector_mask`, `db_read`, `db_clear`, `db_set_mask`, `db_clear_mask`, `db_read_mask`, `peer_db_addr`, `peer_db_set`, `spad_count`, `spad_read`, `spad_write`, `peer_spad_read`, `peer_spad_write`, `msg_count`, `msg_read`, `msg_write`, `msg_post`, ...) | `drivers::ntb::NtbDevOps` |
| `struct ntb_ctx_ops` | client callbacks: `link_event`, `db_event`, `msg_event` | `drivers::ntb::NtbCtxOps` |
| `ntb_register_device(ntb)` / `ntb_unregister_device(ntb)` | per-vendor backend register | `NtbDev::register` / `_unregister` |
| `__ntb_register_client(client, owner, mod_name)` / `ntb_unregister_client(client)` | NTB-bus client register | `NtbClient::register` / `_unregister` |
| `ntb_set_ctx(ntb, ctx, ops)` / `ntb_clear_ctx(ntb)` | bind client callbacks to ntb device | `NtbDev::set_ctx` / `_clear_ctx` |
| `ntb_link_event(ntb)` / `ntb_db_event(ntb, vector)` / `ntb_msg_event(ntb)` | per-vendor → core event injection | `NtbDev::link_event` / `_db_event` / `_msg_event` |
| `ntb_default_port_number(ntb)` / `ntb_default_peer_port_count(ntb)` / `ntb_default_peer_port_idx(ntb, port)` / `ntb_default_peer_port_number(ntb, pidx)` | trivial 2-port helpers for backends without multi-port topology | `Subsystem::default_*` |
| `struct ntb_transport_client` / `ntb_transport_register_client(...)` / `_unregister_client(...)` | transport-layer client | `drivers::ntb::transport::Client` |
| `ntb_transport_create_queue(data, client_dev, handlers)` / `ntb_transport_free_queue(qp)` | per-QP virtual queue-pair lifecycle | `transport::Qp::create` / `_free` |
| `ntb_transport_link_up(qp)` / `ntb_transport_link_down(qp)` / `ntb_transport_link_query(qp)` | per-QP enable/disable/state | `transport::Qp::link_up` / `_link_down` / `_link_query` |
| `ntb_transport_tx_enqueue(qp, cb, data, len)` / `ntb_transport_rx_enqueue(qp, cb, data, len)` | per-QP submit TX / pre-post RX buffer | `transport::Qp::tx_enqueue` / `_rx_enqueue` |
| `ntb_transport_tx_free_entry(qp)` / `ntb_transport_max_size(qp)` / `ntb_transport_qp_num(qp)` | queue stats | `transport::Qp::*` |

## Compatibility contract

REQ-1: Per-vendor backend exposes a `struct ntb_dev_ops` covering link, MW, doorbell, scratchpad and optional message primitives; default helpers (`ntb_default_*`) cover trivial 2-port topologies.

REQ-2: Topology enumeration: `enum ntb_topo` (`NTB_TOPO_PRI`, `NTB_TOPO_SEC`, `NTB_TOPO_B2B_USD`, `NTB_TOPO_B2B_DSD`, `NTB_TOPO_SWITCH`, `NTB_TOPO_CROSSLINK`); per-device `peer_port_count(ntb)` returns ≥ 1 with `peer_port_idx`/`peer_port_number` indirection.

REQ-3: Link control: `link_enable(ntb, max_speed, max_width)` brings the NTB link up to negotiated PCIe gen+width; `link_is_up(ntb, &speed, &width)` returns bitmap of which peers are up; `link_event(ntb)` invoked by backend on transition.

REQ-4: Memory windows: per-direction (local-as-target vs. peer-as-target) MW count via `mw_count(ntb, pidx)` and `peer_mw_count(ntb)`; per-MW alignment via `mw_get_align(ntb, pidx, widx, &addr_align, &size_align, &size_max)`; programming via `mw_set_trans(ntb, pidx, widx, addr, size)` (set local BAR translation to peer-physical address) and `peer_mw_set_trans(ntb, pidx, widx, addr, size)` (program peer's BAR target).

REQ-5: Doorbells: per-side `db_valid_mask(ntb)` reports valid bit indices; `db_set_mask`/`db_clear_mask` control which bits raise IRQ; `peer_db_set(ntb, mask)` raises remote doorbells; per-vector dispatch via `db_vector_count(ntb)` + `db_vector_mask(ntb, vec)`.

REQ-6: Scratchpads: per-side `spad_count(ntb)` 32-bit registers; `spad_read`/`spad_write` on local, `peer_spad_read`/`peer_spad_write` on remote; used as a small fixed-layout out-of-band channel.

REQ-7: Optional messages: per-vendor `msg_count(ntb)` / `msg_read` / `msg_write` / `msg_post` provide queued message passing instead of polled scratchpads on hardware that supports it.

REQ-8: `ntb_transport` builds a virtual point-to-point pipe per QP: each QP has TX ring + RX ring in MW memory; doorbell bit pair (TX-done, RX-ready) raised on submit; per-side scratchpads negotiate QP count, ring size, MW layout at link-up.

REQ-9: `ntb_transport` supports optional `dmaengine` offload: TX copies into MW and RX copies out can use a `dma_chan` (typically Intel IOAT) to bypass CPU memcpy; falls back to memcpy when no chan.

REQ-10: `ntb_transport` link state machine: DOWN → INIT (each side writes its version + QP layout to scratchpads) → READY → UP; mismatch triggers DOWN.

REQ-11: Per-QP buffer model: `ntb_transport_rx_enqueue(qp, cb, data, len)` posts a sink buffer; on RX completion `cb(qp, data, hdr, len)` fires with frame in pre-posted buffer; `ntb_transport_tx_enqueue(qp, cb, data, len)` enqueues a frame, callback fires on TX-side ring slot reuse.

REQ-12: MSI-X across NTB (`msi.c`): per-QP MSI-X interrupt that fires *on the remote host* when a doorbell is raised — implemented by exposing the local MSI-X table base in a MW and having the remote write to it.

## Acceptance Criteria

- [ ] AC-1: Modprobe vendor backend (e.g. `ntb_hw_intel`) on a host with an Intel Xeon NTB pair; `/sys/bus/ntb/devices/0000:ff:0e.0/` appears with non-zero `mw_count`, `db_valid_mask`, `spad_count`.
- [ ] AC-2: Modprobe `ntb_transport` on both hosts; `ntb_transport_link_query(qp)` returns true on both sides after the link-state handshake.
- [ ] AC-3: `ntb_tool` (in-tree) on both sides can write a scratchpad on host A and read it from host B.
- [ ] AC-4: `ntb_pingpong` measures < 5 μs doorbell-to-doorbell latency on a directly-connected pair.
- [ ] AC-5: `ntb_perf` achieves > 5 GB/s throughput over MW on Gen3 x16 with DMA-engine offload enabled.
- [ ] AC-6: `ntb_netdev` (out-of-tree but built against this core) exposes a virtual netdev that pings across the NTB.
- [ ] AC-7: Link bounce (admin disables the link on one side via debugfs, then re-enables): both sides recover to UP without QP leak.
- [ ] AC-8: Hot-eject of one host's NTB function (e.g. surprise removal): `link_event` fires with link-down, queued frames drain, QPs reclaimed.

## Architecture

`NtbDev` is the per-NTB-PCIe-function runtime, registered by per-vendor backends:

```
struct NtbDev {
  dev: Device,                          // ntb_bus_type
  pdev: Arc<PciDev>,
  topo: NtbTopo,
  ops: &'static NtbDevOps,
  ctx: AtomicPtr<()>,                   // client context
  ctx_ops: AtomicPtr<NtbCtxOps>,
  ctx_lock: SpinLock<()>,               // guard concurrent set/clear vs. event
}

struct ntb_transport_ctx {
  ndev: Arc<NtbDev>,
  client_dev_list: Mutex<Vec<Arc<TransportClientDev>>>,
  qp_list: Mutex<Vec<Arc<TransportQp>>>,
  mw_count: u32,
  qp_count: u32,
  qp_bitmap: AtomicU64,                 // free QP indices
  link_is_up: AtomicBool,
  link_work: DelayedWork,               // poll scratchpads until peer ready
  link_cleanup: Work,
  debugfs_node_dir: Option<DebugFsDir>,
}

struct TransportQp {
  qp_num: u32, ndev: Arc<NtbDev>, qp_link: QpLinkState,
  tx_mw: NonNull<u8>, tx_mw_size: usize, tx_index: AtomicU32, tx_max_entry: u32, tx_max_frame: u32,
  tx_dma_chan: Option<Arc<DmaChan>>, tx_free_q: List<TransportQEntry>, tx_pend_q: List<TransportQEntry>,
  rx_buff: NonNull<u8>, rx_buff_size: usize, rx_index: AtomicU32, rx_max_entry: u32, rx_max_frame: u32,
  rx_dma_chan: Option<Arc<DmaChan>>, rx_free_q: List<TransportQEntry>, rx_pend_q: List<TransportQEntry>,
  rx_post_q: List<TransportQEntry>, rx_work: Work,
  remote_rx_info: NonNull<RxInfo>,      // peer's RX progress (we write here)
  rx_info: NonNull<RxInfo>,             // our RX progress (peer writes here)
  rx_handler: fn(qp, *mut RxBuf), tx_handler: fn(qp, *mut TxBuf), event_handler: fn(qp, up: bool),
}
```

Backend registration `NtbDev::register`:
1. Per-vendor probe sets `ntb.topo`, allocates BAR resources, fills `ntb.ops`.
2. `device_register(&ntb.dev)` under `ntb_bus_type`.
3. `ntb_bus.match` walks registered `NtbClient`s; each client `.probe(ntb)` calls `ntb_set_ctx(ntb, client_data, &client_ops)` to register callbacks.

Per-vendor event injection:
1. Backend MMIO IRQ → identify event type (link change vs. doorbell vs. message).
2. Read `ntb.ctx` + `ntb.ctx_ops` under `ctx_lock`.
3. Call `ctx_ops.link_event(ctx)` / `db_event(ctx, vector)` / `msg_event(ctx)`.

`ntb_transport` QP create `Qp::create`:
1. Alloc free QP index from `transport_ctx.qp_bitmap`.
2. Compute per-QP TX/RX MW slice from negotiated layout.
3. `mw_set_trans(ntb, pidx, widx, qp.rx_buff_dma, qp.rx_buff_size)` — program local BAR so the peer's write to its TX address lands in our `rx_buff`.
4. Initialize TX/RX rings: empty queues, pre-allocated `TransportQEntry` pool.
5. Optionally `dma_request_chan(...)` for TX/RX offload.
6. Register `rx_work`, `tx_work` workqueues.

Link-state machine `transport_link_work` (delayed work):
1. Read peer scratchpads: `peer_spad_read(VERSION)`, `_(QP_LINKS)`, `_(NUM_QPS)`, `_(NUM_MWS)`, `_(MW0_SZ_HIGH)`, `_(MW0_SZ_LOW)`, ...
2. Validate version compatibility; on mismatch, set state DOWN.
3. If peer's MW layout matches ours: write our `QP_LINKS` mask to peer scratchpad → peer responds with same mask in its scratchpad.
4. Once both sides agree: per-QP `event_handler(qp, true)` fires.
5. On link-down (peer disabled, `link_event` says link_is_up==false): cancel work, transition all QPs to DOWN.

TX path `Qp::tx_enqueue`:
1. Pop a free `TransportQEntry` from `tx_free_q`.
2. Compute `tx_index` into peer's RX buffer; if `tx_index == remote_rx_info.entry`, ring full → push to `tx_pend_q`, return -EAGAIN (or block).
3. Write 4-byte header (length + flags + sequence) + payload into `tx_mw + tx_index*tx_max_frame` (PCIe write through NTB to peer's `rx_buff`).
4. Update `remote_rx_info.entry` (peer reads this to know new data).
5. Raise `peer_db_set(ntb, qp.tx_db_bit)` (doorbell into peer).
6. Bump `tx_pkts`, schedule `tx_handler(qp, entry)` callback.
7. If `tx_dma_chan`: issue DMA descriptor for the memcpy instead of CPU-copying.

RX path (doorbell on local side):
1. `db_event(ctx, vector)` from backend → `ntb_transport_isr` reads which QP bits set.
2. Per-QP `rx_work` scheduled.
3. `rx_work`: walk `rx_buff + rx_index*rx_max_frame`, validate header (length ≤ rx_max_frame, flags valid), copy payload into pre-posted user buffer from `rx_pend_q`.
4. Call `rx_handler(qp, entry)`.
5. Update `rx_info.entry` so peer's TX can advance.

MSI across NTB (`msi.c`):
1. Local side exposes MSI-X table base in a MW.
2. Remote side writes to that MW address with the MSI-X data; PCIe write triggers IRQ on local side.
3. Useful for replacing doorbell IPIs with direct per-vector MSI-X.

## Hardening

- `ntb_set_ctx`/`_clear_ctx` use `ntb.ctx_lock` to ensure event callbacks see a coherent `(ctx, ctx_ops)` pair; defense against torn pointer reads.
- `ntb_transport` validates RX frame header `length` against `rx_max_frame` before any copy; rejects malformed headers without advancing `rx_index`.
- Per-QP `qp_link` state guards TX submit: enqueue refused unless `READY` or `UP`.
- `mw_set_trans` validates `(addr, size)` against `mw_get_align` (alignment + max size); rejects bad params.
- `tx_pend_q` length bounded by `tx_max_entry`; defense against producer queue-flood.
- DMA engine usage path validates SG entries do not exceed per-MW size before submit.
- Scratchpad reads are atomic 32-bit MMIO; partial reads not observable.
- Link bounce sequence cancels delayed `link_work` and drains all `rx_work`/`tx_work` before re-init.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `ntb_dev`, `transport_qp`, and `TransportQEntry`; scratchpad reads/writes treated as MMIO-not-userland (no kuc).
- **PAX_KERNEXEC** — NTB core (`core.c`, `ntb_transport.c`, `msi.c`) text W^X; `ntb_dev_ops`, `ntb_ctx_ops`, `ntb_client.ops`, `ntb_transport_client.ops` placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel stack on `ntb_db_event`, `ntb_link_event`, transport ISR, `tx_enqueue`, `rx_work` entry.
- **PAX_REFCOUNT** — saturating `refcount_t` on `ntb_dev` lifetime, `transport_qp` lifetime, `dma_chan` refs held by transport; overflow trap defeats link-bounce-race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `transport_qp`, `TransportQEntry`, RX buffers in `rx_pend_q`/`rx_post_q` on QP teardown so prior-peer payloads cannot bleed into the next link cycle.
- **PAX_UDEREF** — SMAP/PAN enforced on debugfs `ntb_tool` write paths (`spad`, `peer_spad`, `db`, `peer_db`, `mw_trans`); userspace pointer deref restricted to canonical helpers.
- **PAX_RAP / kCFI** — `ntb_dev_ops`, `ntb_ctx_ops`, `ntb_client.ops`, `ntb_transport_client.ops`, `tx_handler`/`rx_handler`/`event_handler` callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms disclosure of `ntb_dev` and `transport_qp` pointers behind CAP_SYSLOG; suppress `%p` in NTB tracepoints and debugfs.
- **GRKERNSEC_DMESG** — restrict link-mismatch, MW-translation-fail, and peer-version-skew banners to CAP_SYSLOG.
- **Scratchpad PAX_USERCOPY** — debugfs `spad`/`peer_spad` write paths bounded to `u32` per write; sysfs read paths bounded; defense against userspace pasting unbounded blobs.
- **MW translation CAP_SYS_ADMIN** — `mw_set_trans` / `peer_mw_set_trans` via debugfs (or any non-`ntb_transport` client) gated by CAP_SYS_ADMIN; defense against unprivileged admin pointing a peer BAR at host kernel RAM.
- **Transport queue REFCOUNT** — `transport_qp` ref protected by saturating `refcount_t`; refused negative or overflow transitions.
- **Doorbell-mask validation** — `db_set_mask`/`db_clear_mask`/`peer_db_set` arguments masked to `db_valid_mask(ntb)` before write; defense against undefined-bit writes hitting reserved registers.
- **Header validation on RX** — every RX frame's `length` and `flags` validated against `rx_max_frame` before any copy into user-posted buffer; defense against malformed peer payload.
- **Link-bounce drain** — link-down transition synchronously drains all `rx_work`/`tx_work` before re-init; defense against half-quiesced QP receiving fresh doorbells.

Rationale: an NTB exposes peer-host physical addresses as local BARs and vice versa — a misprogrammed MW translation directly grants the peer-host DMA into our kernel RAM. Bounded MW programming behind CAP_SYS_ADMIN, doorbell masks validated against the hardware-valid mask, kCFI-typed `ntb_dev_ops` dispatch, refcount-overflow trapping on QP teardown, and zero-on-free on transport buffers turn the NTB from "trust the remote OS" into a bounded, validated cross-host channel where every write is policed before it touches kernel memory.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor backends (`ntb_hw_intel`, `ntb_hw_amd`, `ntb_hw_switchtec`, `ntb_hw_idt`, `ntb_hw_epf`) — Tier-4 each.
- `ntb_tool`, `ntb_perf`, `ntb_pingpong` test modules — separate Tier-4 each.
- `ntb_netdev` networking client — separate Tier-3.
- Implementation code.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mw_trans_bounded` | OOB | `mw_set_trans(addr, size)` arguments satisfy `mw_get_align` constraints; refused otherwise. |
| `db_mask_valid` | OOB | every doorbell mask op is subset of `db_valid_mask(ntb)`. |
| `rx_frame_bounded` | OOB | every RX frame `length <= rx_max_frame` before any copy. |
| `qp_no_uaf` | UAF | `TransportQp` outlives every in-flight `TransportQEntry` and pending `rx_work`/`tx_work`. |

### Layer 2: TLA+

`models/ntb/transport_link.tla` (parent-declared): proves both sides converge to UP iff scratchpad-handshake matches; link-down on either side propagates without leaking QPs; no torn pointer reads on `ntb.ctx` between `ntb_set_ctx` and event callbacks.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ntb_set_ctx` post: subsequent `ntb_db_event`/`link_event`/`msg_event` always observe matching `(ctx, ctx_ops)` | `NtbDev::set_ctx` |
| `Qp::tx_enqueue` post: returned OK iff `tx_index` advanced exactly one slot and `remote_rx_info.entry` updated | `transport::Qp::tx_enqueue` |
| `Qp::rx_work` post: every consumed frame's `length` satisfied `<= rx_max_frame` AND `rx_index` advanced exactly that many slots | `transport::Qp::rx_work` |
| `link_is_up` invariant: UP only if both `our spad_QP_LINKS` and `peer spad_QP_LINKS` agree | `transport_link_work` |

### Layer 4: Verus/Creusot functional

Link bring-up: both sides publish (version, QP layout) into scratchpads → both observe agreement → `event_handler(qp, true)` fires once per QP → TX enqueue succeeds → frame appears at peer RX with header intact. Encoded as Verus refinement of an abstract handshake protocol with no observable inconsistency between sides.
