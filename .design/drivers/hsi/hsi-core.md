# Tier-3: drivers/hsi/{hsi_core,hsi_boardinfo}.c — HSI framework (High-Speed Synchronous Serial Interface)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/hsi/hsi_core.c
  - drivers/hsi/hsi_boardinfo.c
  - drivers/hsi/hsi_core.h
  - include/linux/hsi/hsi.h
-->

## Summary

The HSI (High-Speed Synchronous Serial Interface, MIPI Alliance DigRFv3 / OMAP4 era) framework — a high-speed bidirectional point-to-point serial interconnect between the application processor and a cellular modem. Each controller exposes one or more `hsi_port` (typically 1 per controller) that hosts one `hsi_client` per protocol (`ssi_protocol`, `hsi_char`). Messages are scatter-gather buffers (`hsi_msg`) submitted asynchronously via `hsi_async`; HW wake-up notification routed through a blocking notifier chain.

This Tier-3 covers `hsi_core.c` (~761 lines: bus_type, controller/port/client lifecycle, OF + board-info enumeration, msg alloc/free, async submit, port claim/release with sharing, event notifier registration, channel-id-by-name lookup) and `hsi_boardinfo.c` (~49 lines: per-board static `hsi_board_info` list registration for non-DT platforms).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hsi_controller` | per-bus controller (top-level) | `drivers::hsi::Controller` |
| `struct hsi_port` | per-port child of controller (mutex, notifier chain, async/setup/flush/start_tx/stop_tx/release callbacks) | `drivers::hsi::Port` |
| `struct hsi_client` | per-protocol-client child of port (tx_cfg, rx_cfg, ehandler, nb) | `drivers::hsi::Client` |
| `struct hsi_msg` | per-async-message (sgt, ttype, channel, status, actual_len, complete, destructor) | `drivers::hsi::Msg` |
| `struct hsi_config` | per-direction config (mode, channels, num_channels, num_hw_channels, speed, flow, arb_mode) | `drivers::hsi::Config` |
| `struct hsi_channel` | per-channel descriptor (id, name) | `drivers::hsi::Channel` |
| `struct hsi_board_info` | per-board static client descriptor | `drivers::hsi::BoardInfo` |
| `hsi_alloc_controller(n_ports, flags)` | alloc controller + ports | `Controller::alloc` |
| `hsi_register_controller(hsi)` / `_unregister_controller(hsi)` | per-controller lifecycle | `Controller::register` / `_unregister` |
| `hsi_put_controller(hsi)` | release before successful register | `Controller::put` |
| `hsi_new_client(port, info)` | per-board-info client registration | `Port::new_client` |
| `hsi_add_clients_from_dt(port, clients)` | DT-tree client enumeration | `Port::add_clients_from_dt` |
| `hsi_remove_client(dev, _)` / `hsi_port_unregister_clients(port)` | per-port client teardown | `Port::unregister_clients` |
| `hsi_alloc_msg(nents, flags)` / `hsi_free_msg(msg)` | per-message lifecycle | `Msg::alloc` / `_free` |
| `hsi_async(cl, msg)` | submit async transfer | `Client::async` |
| `hsi_claim_port(cl, share)` / `hsi_release_port(cl)` | per-port claim with sharing | `Client::claim_port` / `_release_port` |
| `hsi_register_port_event(cl, handler)` / `_unregister_port_event(cl)` | per-client event notifier | `Client::register_port_event` |
| `hsi_event(port, event)` | controller raises HSI_EVENT_{START_RX,STOP_RX} | `Port::event` |
| `hsi_get_channel_id_by_name(cl, name)` | per-name channel id lookup | `Client::get_channel_id_by_name` |
| `hsi_register_client_driver(drv)` | client driver register | `ClientDriver::register` |
| `hsi_register_board_info(info[], len)` | per-board static client list | `BoardInfo::register` |

## Compatibility contract

REQ-1: Bus topology — single point-to-point link per controller; each controller has fixed `num_ports` (typically 1); each port hosts 0..N clients (one per protocol).

REQ-2: Per-port claim semantics — `claimed` refcount + `shared` bool; multi-client claim requires every claimer to pass `share=1`; exclusive claim refused if already claimed.

REQ-3: Per-direction config — `hsi_config` carries `mode ∈ {STREAM, FRAME}`, `flow ∈ {SYNC, PIPE}`, `arb_mode ∈ {RR, PRIO}`, `speed` (kbps), `channels[]` + `num_channels` + `num_hw_channels`; tx/rx configs independent.

REQ-4: Per-message scatter-gather — `hsi_alloc_msg(nents)` builds `sgt` of `nents` entries; nents=0 valid (read-without-consume); client owns buffers, framework owns only the descriptor.

REQ-5: Per-message contract for `hsi_async` — caller must set `channel`, `ttype ∈ {TX, RX}`, `complete`, `destructor` before submit; `WARN_ON_ONCE` fires if `complete` or `destructor` is NULL.

REQ-6: Port-claim required before async — `hsi_async` returns -EACCES if `cl->pclaimed` is zero; `claim_port` must precede first transfer.

REQ-7: Per-client event handler — `register_port_event` requires prior port claim; handler called from notifier chain (`blocking_notifier_chain`); single handler per client.

REQ-8: HSI events — `HSI_EVENT_START_RX` (wake-line high) and `HSI_EVENT_STOP_RX` (wake-line down); the spec-mandated wake-line race workaround relies on these being delivered to every claimer.

REQ-9: Module pin — per-port claim takes `try_module_get(controller->owner)`; release calls `module_put` — defeats unload-while-claimed.

REQ-10: DT enumeration — `hsi-mode/hsi-rx-mode/hsi-tx-mode` ∈ {"stream","frame"}; `hsi-speed-kbps` u32; `hsi-flow` ∈ {"synchronized","pipeline"}; `hsi-arb-mode` ∈ {"round-robin","priority"}; `hsi-channel-ids` + `hsi-channel-names` parallel arrays.

REQ-11: `hsi_char` device auto-registered on every DT port (always-present userland char-device for raw HSI client).

REQ-12: Per-board-info list `hsi_board_list` (cross-ref `hsi_boardinfo.c`) — per-board static `hsi_board_info` entries keyed by `(hsi_id, port)`; scanned on controller register.

## Acceptance Criteria

- [ ] AC-1: Nokia N950/N9 boot (OMAP3-based) probes HSI controller; `dmesg | grep hsi` shows controller + port + ssi_protocol/hsi_char clients.
- [ ] AC-2: `hsi_async` round-trip on `hsi_char` `/dev/hsi-char-X` returns expected bytes via complete-callback.
- [ ] AC-3: Two clients claim the same port with `share=1`; exclusive third claim (`share=0`) is rejected with -EBUSY.
- [ ] AC-4: Unloading controller module is refused while a port claim is outstanding; succeeds after release.
- [ ] AC-5: HSI_EVENT_START_RX fires notifier registered via `hsi_register_port_event`; receiver dec ehandler.
- [ ] AC-6: Channel lookup `hsi_get_channel_id_by_name(cl, "mcsaab-control")` returns the configured channel id; returns -ENXIO on unknown name.

## Architecture

```
struct Controller {
  device: Device,                       // bus = hsi_bus_type
  id: u32,
  num_ports: u32,
  owner: &'static Module,
  port: [&Port; num_ports],             // flexible-array tail
}

struct Port {
  device: Device,                       // child of controller
  num: u32,
  claimed: i32,                         // refcount of claimers
  shared: bool,                         // true iff first claim passed share=1
  lock: Mutex<()>,                      // claim/release serialization
  n_head: BlockingNotifierHead,         // event notifier chain
  async:     fn(*mut Msg) -> i32,
  setup:     fn(*mut Client) -> i32,
  flush:     fn(*mut Client) -> i32,
  start_tx:  fn(*mut Client) -> i32,
  stop_tx:   fn(*mut Client) -> i32,
  release:   fn(*mut Client) -> i32,
}

struct Client {
  device: Device,                       // bus = hsi_bus_type, parent = port.device
  tx_cfg: Config, rx_cfg: Config,
  pclaimed: bool,                       // this client holds the port claim
  nb: NotifierBlock,                    // entry in port.n_head
  ehandler: Option<fn(&Client, u64 event)>,
}

struct Msg {
  sgt: SgTable,
  cl: *mut Client,
  channel: u32, ttype: enum { TX, RX },
  status: i32, actual_len: usize,
  complete: fn(*mut Msg),
  destructor: fn(*mut Msg),
}
```

Per-controller alloc `hsi_alloc_controller(n_ports, gfp)`:
1. Flex-allocate `hsi_controller` with `port` array of length `n_ports`.
2. `device_initialize(&hsi->device)`; per-port: alloc `hsi_port`, init mutex + notifier head, install dummy ops (`hsi_dummy_msg`/`hsi_dummy_cl`).

Per-controller register `hsi_register_controller(hsi)`:
1. `device_add(&hsi->device)` + per-port `device_add`.
2. `hsi_scan_board_info(hsi)` — walk `hsi_board_list`, dispatch per-board-info matching `hsi_id` + `port`.

Per-client from DT `hsi_add_client_from_dt(port, np)`:
1. Read `hsi-mode` or `hsi-{rx,tx}-mode` → STREAM/FRAME.
2. Read `hsi-speed-kbps` → rx/tx speed.
3. Read `hsi-flow` + `hsi-arb-mode`.
4. Read `hsi-channel-ids` + `hsi-channel-names`; kalloc per-direction channel arrays.
5. `device_register(&cl->device)` with parent = port.device.

Per-port claim `hsi_claim_port(cl, share)`:
1. Take `port->lock`.
2. If already claimed and (!port->shared || !share): -EBUSY.
3. `try_module_get(controller->owner)`; -ENODEV on fail.
4. Increment claimed; set shared = !!share; set `cl->pclaimed = 1`.

Per-port release `hsi_release_port(cl)`:
1. Take `port->lock`.
2. Call `port->release(cl)` (hw-level cleanup).
3. Decrement claimed if pclaimed; `BUG_ON(claimed < 0)`.
4. Clear `cl->pclaimed`; if claimed reaches 0, clear `shared`.
5. `module_put(controller->owner)`.

Per-async submit `hsi_async(cl, msg)`:
1. Get port from `cl`.
2. Validate `cl->pclaimed` (-EACCES otherwise).
3. `WARN_ON_ONCE` if `complete` or `destructor` is NULL.
4. `msg->cl = cl`; dispatch `port->async(msg)`.

Event notifier `hsi_register_port_event(cl, handler)`:
1. Validate non-NULL handler + no prior `ehandler`.
2. Validate port claim.
3. Set `cl->ehandler`, init `cl->nb`, `blocking_notifier_chain_register(&port->n_head, &cl->nb)`.

`hsi_event(port, event)` invokes `blocking_notifier_call_chain` which fans out to each registered client's `ehandler` via `hsi_event_notifier_call`.

## Hardening

(Inherits row-1 features from `drivers/bus/00-overview.md` § Hardening.)

hsi-core specific reinforcement:

- **Per-port mutex serializes claim/release** — defeats refcount/shared-flag races.
- **`try_module_get` on every claim** — defeats unload-while-claimed UAF.
- **`BUG_ON(port->claimed < 0)`** — detects double-release immediately.
- **`hsi_async` validates pclaimed + complete + destructor** — refuses to submit without proper teardown handlers.
- **DT parser rejects invalid mode/flow/arb_mode strings** — refuses unsupported configs at probe.
- **Per-controller flex-array bounds** at alloc — `kzalloc_flex` sized exactly to `num_ports`.
- **`hsi_char` always registered on DT** — predictable userland surface; absence indicates port init failure.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `hsi_controller` (flex-array of `hsi_port*`), `hsi_port`, `hsi_client`, and `hsi_msg`; per-message `sgt` is allocated via `sg_alloc_table` (kernel-owned); client buffers passed via `sgt` are caller-provided.
- **PAX_KERNEXEC** — HSI framework core in W^X kernel text; `hsi_bus_type`, per-port ops (`async/setup/flush/start_tx/stop_tx/release`), and notifier-chain dispatch live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `hsi_async`, `hsi_event`, `hsi_claim_port`, `hsi_release_port`, and `hsi_event_notifier_call`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-controller / per-port / per-client `dev` refcount + per-port `claimed` claim-count + per-controller `owner` module refcount; overflow trap defeats claim/release race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `hsi_msg`, `hsi_client`, `hsi_port`, `hsi_controller` and per-direction `channels` arrays so stale channel ids, sgt pointers, and notifier blocks cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every userland entry through `/dev/hsi-char-*` (delivered by the `hsi_char` client driver, not this framework); framework helpers themselves take only kernel-owned buffers.
- **PAX_RAP / kCFI** — `hsi_client_driver` ops and per-port callback table (`async/setup/flush/start_tx/stop_tx/release`) marked `__ro_after_init` with kCFI-typed indirect dispatch; per-msg `complete`/`destructor` callbacks dispatched only through kCFI-typed thunk.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-controller MMIO-base disclosure behind CAP_SYSLOG; suppress `%p` in async-completion / notifier-chain tracepoints.
- **GRKERNSEC_DMESG** — restrict claim-failure, register-client-failure, and DT-parse-error banners to CAP_SYSLOG so attackers cannot probe modem-link topology via dmesg.
- **`hsi_msg` PAX_USERCOPY** — `sgt` is a kernel-owned scatterlist; payload bytes are caller-provided and never directly copied into framework-owned slab; per-msg `actual_len` validated < total `sgt` length before `complete()`.
- **Port/client ID validated** — `hsi_id` + `port` for board-info matching validated against `controller->id` + `controller->num_ports`; DT enumeration validates `hsi-channel-ids` array length divisible by `sizeof(u32)`.
- **Async-callback RAP** — `msg->complete` and `msg->destructor` invoked only through kCFI-typed dispatch; per-msg `cl` backref validated before each invocation; `WARN_ON_ONCE` on submit catches missing callback.
- **dma-buf REFCOUNT** — `sgt` entries pointing to dma-mapped client buffers participate in the framework's `hsi_free_msg` `sg_free_table` cleanup; client is responsible for unmapping the underlying pages before `hsi_free_msg`, preventing dma-after-free.
- **Notifier-chain ordering** — `blocking_notifier_call_chain` invokes per-client handlers in registration order; framework refuses double-register (`cl->ehandler` non-NULL pre-check), defeating duplicate-handler invocation.
- **Module pin / unload guard** — `try_module_get(controller->owner)` on `hsi_claim_port` and `module_put` on `hsi_release_port` are paired across every branch including release-from-error; `hsi_remove_client` walks both ways and never leaks the pin.

Rationale: HSI is the AP↔cellular-modem control plane on legacy mobile platforms; the modem side often runs an opaque RTOS, making the AP-side framework the only attack-defense gate. A claim/release imbalance, a missing `pclaimed` check before `hsi_async`, or a notifier-chain double-register can corrupt modem state, leak sgt pointers across reuse, or wedge the modem heartbeat. RAP/kCFI on async-callback dispatch, refcount-overflow trapping on per-port claim, kCFI-typed `hsi_client_driver` ops, and module-pin pairing turn the framework into a structural enforcement boundary over the modem interconnect.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `claim_refcount_nonneg` | INVARIANT | `port->claimed >= 0` always; release on unclaimed BUG_ONs. |
| `async_requires_claim` | PRE | `hsi_async` returns -EACCES iff `cl->pclaimed == 0`. |
| `channels_within_bound` | OOB | `cl->{rx,tx}_cfg.num_channels` == number of entries in DT `hsi-channel-ids` (parsed length / sizeof(u32)). |
| `notifier_no_double_register` | INVARIANT | `register_port_event` fails with -EINVAL if `cl->ehandler` already set. |

### Layer 2: TLA+

`models/hsi/port_claim.tla` (parent-declared): proves shared-claim invariant — once `shared` is true, exclusive subsequent claim is impossible until all shared claimers release; refcount returns to 0 → shared resets to false.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `hsi_async(cl, msg)` post: `msg->cl == cl`; framework never observes `complete` returning before `port->async` is dispatched | `Client::async` |
| `hsi_release_port` post: `module_put` called exactly once per matching `try_module_get` from `hsi_claim_port` | `Port::release_port` |
| `hsi_alloc_msg(0, _)` post: returned msg has empty sgt; `hsi_free_msg(NULL)` is a no-op | `Msg::alloc` / `_free` |

### Layer 4: Verus/Creusot functional

Claim/submit/release cycle: `claim_port(share) → async(msg) → complete(msg) → release_port` preserves `claimed` refcount monotonicity and modem-link module pin parity; encoded as Verus invariant chained with controller-driver `port->async` postcondition.

## Out of Scope

- `ssi_protocol.c` client driver (Nokia ModEM SSI protocol) — separate future Tier-3
- `hsi_char.c` userland char-device — separate future Tier-3
- Per-controller drivers (`omap_ssi.c`, etc.) — separate future Tier-3
- 32-bit-only paths
- Implementation code
