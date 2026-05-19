# Tier-3: drivers/thunderbolt/{tb.c,ctl.c,switch.c,tunnel.c} — Connection Manager, control channel, switch + tunnel mgmt

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/thunderbolt/00-overview.md
upstream-paths:
  - drivers/thunderbolt/tb.c
  - drivers/thunderbolt/tb.h
  - drivers/thunderbolt/ctl.c
  - drivers/thunderbolt/ctl.h
  - drivers/thunderbolt/switch.c
  - drivers/thunderbolt/tunnel.c
  - drivers/thunderbolt/tunnel.h
-->

## Summary

The native Linux Connection Manager (CM) — the brain of the Thunderbolt subsystem. Discovers + manages the router topology, talks to each router via the per-domain control channel, builds + tears down per-protocol tunnels across the discovered tree. Four files:

- `tb.c` (~3400 lines): `struct tb_cm` per-domain state, hotplug event loop, tunnel allocation orchestration, DP resource tracking, bandwidth-group management.
- `ctl.c` (~1200 lines): `struct tb_ctl` control channel; TX/RX rings on NHI hop 0; config-space read/write packets, request/response matching, hotplug event delivery.
- `switch.c` (~4000 lines): `struct tb_switch` per-router model; UUID + NVM + capabilities + ports + DROM + sysfs; per-router authorization; child-router enumeration.
- `tunnel.c` (~2600 lines): `struct tb_tunnel` per-protocol tunnels (PCIe, DP, USB3, DMA); path construction across the route; per-hop credit/priority programming.

The CM operates above NHI HW (`nhi.c`, see `drivers/thunderbolt/domain.md`) and below the bus / sysfs layer (`domain.c`, also in `domain.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tb_cm` | per-domain CM state: tunnel_list, dp_resources, hotplug_active, remove_work, groups[MAX_GROUPS] | `thunderbolt::Cm` |
| `tb_handle_hotplug(work)` | hotplug event dispatch | `Cm::handle_hotplug` |
| `tb_scan_port(port)` | walk downstream of a port discovering switches | `Cm::scan_port` |
| `tb_scan_switch(sw)` | recursively scan a switch's children | `Cm::scan_switch` |
| `struct tb_switch` | per-router model (route, depth, uuid, vendor, device, security_level, authorized, key, ports[], nvm) | `thunderbolt::Switch` |
| `tb_switch_alloc(tb, parent, route)` / `_add` / `_remove` | router lifecycle | `Switch::alloc` / `_add` / `_remove` |
| `tb_switch_configure(sw)` | program router-level config registers | `Switch::configure` |
| `tb_switch_get_uid(sw)` / `_get_drom(sw)` / `_read_caps(sw)` | per-router identity + DROM + caps | `Switch::get_uid` / `_get_drom` / `_read_caps` |
| `tb_switch_authorize(sw, level)` | per-router authorize gate | `Switch::authorize` |
| `tb_switch_nvm_read(sw, off, buf, len)` / `_write(sw, ...)` | NVM read/write via control channel | `Switch::nvm_read` / `_write` |
| `struct tb_port` | per-port adapter state | `thunderbolt::Port` |
| `tb_port_alloc_hopid(port, in, ...)` / `_release_hopid(...)` | per-port hopid allocation | `Port::alloc_hopid` / `_release_hopid` |
| `struct tb_ctl` | control channel: nhi, tx ring, rx ring, frame_pool (DMA pool), request_queue | `thunderbolt::Ctl` |
| `tb_ctl_alloc(nhi, ...)` / `tb_ctl_start(ctl)` / `tb_ctl_stop(ctl)` / `tb_ctl_free(ctl)` | per-domain ctl lifecycle | `Ctl::alloc` / `_start` / `_stop` / `_free` |
| `tb_cfg_read(ctl, buf, route, port, space, off, len)` / `tb_cfg_write(...)` | config-space access | `Ctl::cfg_read` / `_write` |
| `tb_cfg_request(ctl, req, timeout)` / `_response(ctl, ...)` | request/response framework | `Ctl::request` / `_response` |
| `struct tb_tunnel` | per-tunnel state (type, src, dst, paths[]) | `thunderbolt::Tunnel` |
| `tb_tunnel_alloc_pci(tb, up, down)` / `_alloc_dp(...)` / `_alloc_usb3(...)` / `_alloc_dma(...)` | per-protocol tunnel ctor | `Tunnel::alloc_*` |
| `tb_tunnel_activate(tunnel)` / `_deactivate(tunnel)` | program path entries on every router | `Tunnel::activate` / `_deactivate` |
| `tb_tunnel_consumed_bandwidth(tunnel, ...)` / `_alloc_bandwidth(...)` | DP/USB3 bandwidth accounting | `Tunnel::*_bandwidth` |
| `struct tb_bandwidth_group` | per-Group_ID set of DP tunnels sharing bandwidth | `thunderbolt::BandwidthGroup` |

## Compatibility contract

REQ-1: Per-domain `struct tb_cm` sits at `tb->privdata` (allocated as `sizeof(tb_cm)` trailing the `tb`); accessed via `tcm = tb_priv(tb)`. Lifetime = domain lifetime.

REQ-2: Hotplug event delivery: NHI IRQ → CTL RX → CTL `tb_cfg_event_callback` → `tb_queue_hotplug` → `tb_handle_hotplug` work item; per-event work bounded by `TB_TIMEOUT = 100 ms`.

REQ-3: Router discovery: `tb_scan_port` walks each downstream port; per-port `tb_switch_alloc` reads UID + DROM + caps via CTL; if discovery fails it's retried up to `TB_BW_ALLOC_RETRIES = 3` times.

REQ-4: Each switch has a `route` (64-bit path string of per-hop port numbers), `depth` (1..N from host), `uuid` (read from router's UUID-cap), and `device_id` / `vendor_id` from router config space.

REQ-5: `tb_switch_add` calls `device_add` to attach to the thunderbolt bus; `authorized` defaults to 0 unless `tb->security_level == TB_SECURITY_NONE` in which case it's auto-set to 1.

REQ-6: `tb_switch_authorize(sw, 1)`: caller has CAP_SYS_ADMIN (enforced at `sysfs_store` layer in `domain.c`); on success builds PCIe tunnel via `tb_tunnel_alloc_pci(tb, sw->ports[PCIE_USP], parent_sw->ports[PCIE_DSP])` + `tb_tunnel_activate`.

REQ-7: `tb_switch_authorize(sw, 2)`: with key — challenge-response: random nonce + HMAC-SHA256(key, nonce) sent to router; router responds with HMAC over (nonce, router-uid).

REQ-8: Control channel: TX ring on hop 0, RX ring on hop 0. Frames are `ctl_pkg` DMA pool allocations (≤ 256 bytes per packet including header + CRC32). `TB_CTL_RX_PKG_COUNT = 10` packets pre-allocated.

REQ-9: `tb_cfg_read`/`_write` build a `cfg_pkg` (route, port, config-space, offset, len) packet; submit to TX ring; await matching response in `request_queue` with `timeout_msec` timeout; retry up to `TB_CTL_RETRIES = 4` on TIMEOUT.

REQ-10: PCIe tunnel: 2 paths (UP + DOWN), HopID 8 on both directions; priority 3, weight 1; minimum `TB_MIN_PCIE_CREDITS = 6` per path.

REQ-11: USB3 tunnel: 2 paths (UP + DOWN), HopID 8 in both directions; priority 3, weight 2; USB4 v2 reserves `USB4_V2_USB3_MIN_BANDWIDTH = 1500 * 2 = 3000 Mb/s`.

REQ-12: DP tunnel: 3 paths (video out, AUX out, AUX in); video HopID 9, AUX HopID 8; video priority 1 / weight 1; AUX priority 2 / weight 1; DPRX establishment timeout 12s after activate.

REQ-13: DMA tunnel: variable HopID; `TB_DMA_CREDITS = 14` default, min `TB_MIN_DMA_CREDITS = 1`; used for XDomain peer-to-peer.

REQ-14: Bandwidth-group management: up to `MAX_GROUPS = 7` groups; each DP tunnel assigned a Group_ID; per-group total bandwidth budget enforced at `tb_tunnel_alloc_bandwidth`.

REQ-15: Asymmetric link reconfig: bandwidth requests above `TB_ASYM_THRESHOLD = 45000 Mb/s` cause link to switch to asymmetric mode; below `TB_ASYM_MIN = 36000 Mb/s` link returns to symmetric.

## Acceptance Criteria

- [ ] AC-1: Plug TB3 dock → hotplug event → switch discovered → sysfs `device`/`vendor`/`uuid`/`authorized=0` exposed within 200 ms.
- [ ] AC-2: `echo 1 > authorized` on a `TB_SECURITY_USER` domain → PCIe tunnel up → downstream NVMe enumerated within 500 ms.
- [ ] AC-3: `echo 2 > authorized` with stored key on `TB_SECURITY_SECURE` domain → challenge-response succeeds → tunnel up.
- [ ] AC-4: Plug 4K@60 monitor in dock → GPU emits DP source hotplug → DP tunnel activates → DPRX completes within 12s.
- [ ] AC-5: Two USB3 devices in dock → both enumerated through host xHCI via single USB3 tunnel; bandwidth shared per USB4 v2 reservation.
- [ ] AC-6: XDomain peer connect → DMA tunnel via `tb_tunnel_alloc_dma` → `tbnet0` link up between two hosts.
- [ ] AC-7: Unplug TB3 dock → all tunnels through that branch deactivated within 100 ms → child switches removed from sysfs.
- [ ] AC-8: Bandwidth above 45 Gb/s on DP tunnel → link switches to asymmetric; drop below 36 Gb/s → returns to symmetric (observable via `link_speed` sysfs).
- [ ] AC-9: CTL CRC32 mismatch on RX → packet dropped; ctl.request_queue waiter times out + retries up to 4 times.
- [ ] AC-10: NVM read of a 64 KB switch NVM image via `/sys/bus/thunderbolt/devices/<sw>/nvm_non_active/nvmem` succeeds.

## Architecture

### CM event loop (`tb.c`)

`struct tb_cm` (per-domain, lives at `tb->privdata`) carries `tunnel_list`, `dp_resources`, `hotplug_active`, `remove_work` (delayed), and `groups[MAX_GROUPS]`. Hotplug events are queued as `struct tb_hotplug_event { delayed_work, tb, route, port, unplug, retry }`.

Hotplug handler (`tb_handle_hotplug`):
1. `mutex_lock(&tb->lock)` (one big lock per domain).
2. Resolve `sw = tb_switch_find_by_route(tb, ev->route)`.
3. If `ev->unplug`: tear down all tunnels passing through sw + descendants, `tb_switch_remove(sw)`.
4. If plug: `tb_scan_port(&sw->ports[ev->port])` → discovers any downstream switch + recurses.
5. After scan: rebalance DP bandwidth groups, possibly reconfigure link symmetry.

### Switch model (`switch.c`)

`struct tb_switch` carries: `dev`, `tb`, `parent`, `route` (64-bit hop path), `depth`, `uuid`, `vendor`, `device`, `security_level`, `authorized`, `key[TB_SWITCH_KEY_SIZE]` (HMAC), `ports[port_count]`, `nvm`, `dma_port`, `nboot_acl`, `generation` (TB1/2/3, USB4 v1/v2), `hopids` (ida).

UID & key flow: `tb_switch_get_uid` reads router UID via CTL `tb_cfg_read(SPACE_ROUTER_UID)`. `nvm_set_auth_status` / `_get_auth_status` cache per-UUID auth status in `nvm_auth_status_cache` so power-cycled routers retain failure state across resume. Secure-level authorize: `crypto_shash_setkey(hmac_sha256, sw->key)` over random nonce → 32-byte digest sent via control packet → router HMAC verified on return.

### Control channel (`ctl.c`)

`struct tb_ctl` carries: `nhi`, `tx`/`rx` rings (NHI hop 0), `frame_pool` (DMA pool), `rx_packets[TB_CTL_RX_PKG_COUNT]`, `request_queue` + `request_queue_lock`, `running`, `timeout_msec`, `callback`/`callback_data`, `index`.

Packet wire format: `[header (route, port, space, offset, len, ack, error, type)][payload (≤ 256-byte frames)][CRC32]`.

Request lifecycle: `tb_cfg_request(ctl, &req, timeout)` builds packet → allocates frame from `frame_pool` → pushes to TX ring → enqueues onto `request_queue` keyed by (route, type, seq). NHI raises TX-complete IRQ; frame returned to pool. RX IRQ + ring callback: CRC32 check → match request_queue entry → complete via `req->completion`; unmatched packet = notification → invoke `ctl->callback`. On timeout: dequeue + `-ETIMEDOUT`; caller retries up to `TB_CTL_RETRIES = 4`. `tb_cfg_request` uses `kref_get/put` under `tb_cfg_request_lock` so concurrent cancel is safe.

### Tunnel model (`tunnel.c`)

`struct tb_tunnel` carries: `tb`, `callback`/`callback_data`, `src_port`/`dst_port`, `paths[npaths]`, vtable (`activate`, `deactivate`, `maximum_bandwidth`, `allocated_bandwidth`, `alloc_bandwidth`, `consumed_bandwidth`, `release_unused_bandwidth`, `reclaim_available_bandwidth`), `list` (in `tcm.tunnel_list`), `type`, `bw_group`.

Activation (`tb_tunnel_activate`): per path[i] program `IN/OUT_HOP_ID`, `PRIORITY`, `WEIGHT`, `CREDITS`, `INITIAL_CREDITS` on every router along the route (CTL `tb_cfg_write` to `SPACE_PATH`); per protocol additional adapter-level config (PCIe ATS enable, DP `set_active`, USB3 `set_link`); set `tunnel->active = true`; add to `tcm.tunnel_list`.

Bandwidth-group: `struct tb_bandwidth_group { tb, index (1..MAX_GROUPS), reserved (Mb/s), ports (DP IN adapters) }`. Per-group reservation enforced at `tb_tunnel_alloc_bandwidth`; consumers above the group's budget yield to existing tunnels.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tunnel_list_no_uaf` | UAF | `tcm->tunnel_list` traversal always under `tb->lock`; deactivate removes from list before free |
| `ctl_request_queue_no_uaf` | UAF | `tb_cfg_request_lock` + `kref` keep request alive across cancel race |
| `ctl_crc_validated` | CRC | every RX packet's CRC32 validated before dispatch; reject on mismatch |
| `hopid_no_collision` | UNIQUENESS | `sw->hopids` ida ensures HopID never re-issued while in use |
| `switch_route_unique` | UNIQUENESS | each `tb_switch` has unique `route` within a domain |
| `bandwidth_group_capped` | BOUND | per-group `reserved + consumed ≤ link_max` |
| `tunnel_paths_bounded` | OOB | `tunnel->npaths ≤ TB_MAX_PATHS` per protocol |

### Layer 2: TLA+

`models/thunderbolt/cm_hotplug.tla` (parent-declared): models N hotplug events interleaved with M authorize requests under `tb->lock`. Properties: (a) tunnels through an unplugged branch are deactivated before switches are removed, (b) `authorized=1` followed by unplug-during-tunnel-activate races correctly to a clean state.

`models/thunderbolt/ctl_request.tla`: models CTL request/response queue under concurrent submit + cancel + RX-callback. Property: every successful response matches exactly one in-flight request; cancelled requests return without UAF on `req` memory.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cm::handle_hotplug` post: on `unplug`, no tunnels in `tcm.tunnel_list` reference removed sw or descendants | `Cm::handle_hotplug` |
| `Switch::authorize(level=1)` post: PCIe tunnel exists in `tcm.tunnel_list` with src/dst matching sw + parent | `Switch::authorize` |
| `Switch::authorize(level=2)` post: HMAC-SHA256 challenge-response verified; tunnel built only on success | `Switch::authorize` |
| `Tunnel::activate` post: every router along route has path-config-space entries matching `tunnel.paths[*]` | `Tunnel::activate` |
| `Ctl::cfg_read` post: response payload CRC32 verified ∧ matches request seq | `Ctl::cfg_read` |

### Layer 4: Verus/Creusot functional

Plug → discover → authorize → PCIe tunnel up: encoded as a state-machine refinement from `domain.empty()` through hotplug event → switch added → authorize → tunnel activate → NVMe device PCI-enumerated. Every step's invariant verified; failure injection (CTL timeout, NVM-auth fail, group-budget exceeded) modeled as alternate transitions producing clean rollback.

## Hardening

(Inherits from `drivers/thunderbolt/00-overview.md` § Hardening.)

`tb.c` / `ctl.c` / `switch.c` / `tunnel.c` specific reinforcement:

- **Single `tb->lock` per domain** — serializes hotplug, authorize, tunnel mgmt; defense against UAF across concurrent unplug + authorize.
- **CTL TX/RX ring DMA pool sized to `TB_CTL_RX_PKG_COUNT`** — bounded; defense against per-packet DMA-pool exhaustion.
- **CRC32 verified on every RX packet** — `tb_ctl_handle_event` rejects mismatched frames.
- **Request/response matching by (route, type, seq)** — defense against response-injection from a misbehaving router.
- **`hotplug_active` flag** — set false during CTL shutdown; defense against in-flight hotplug work after `tb_ctl_stop`.
- **`TB_TIMEOUT = 100 ms` per CTL request** — defense against router stalling forever.
- **HMAC-SHA256 over (router-uid ‖ nonce ‖ key)** — challenge-response uses cryptographic mac.
- **NVM auth status cached per-UUID** — failure persists across power cycle; refuses re-enroll without explicit forget.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `tb`, `tb_cm`, `tb_switch`, `tb_port`, `tb_tunnel`, `tb_path`, `tb_ctl`, `tb_cfg_request`, `tb_hotplug_event`, and the DMA-pool `ctl_pkg` frames.
- **PAX_KERNEXEC** — CM core, CTL handler, switch model, tunnel ctor in W^X kernel text; `tb_cm_ops`, `tb_tunnel_callbacks`, and the per-protocol activate vtables `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `tb_handle_hotplug`, `tb_switch_add`, `tb_switch_authorize`, `tb_tunnel_activate`, `tb_ctl_rx_callback`, and `tb_cfg_request`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `tb_switch`, `tb_xdomain`, `tb_tunnel`, `tb_cfg_request`, `tb_service`, and the per-DP-resource handle; overflow trap defeats double-authorize / double-deactivate races.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `tb_switch` (UUID + key), `tb_cfg_request` (config-space payload), `ctl_pkg` DMA frames (control-plane data), and `tb_tunnel` (HopID + path tables) so stale credentials and topology bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs `_store` callback in switch + domain, every nvmem read/write, and every XDomain ioctl.
- **PAX_RAP / kCFI** — `tb_cm_ops`, `tb_tunnel_callbacks`, `tb_cfg_request_release`, `ctl->callback`, switch sysfs ops, and the per-tunnel `activate`/`deactivate`/`alloc_bandwidth` vtables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-switch UID / NVM-key / route disclosure behind CAP_SYSLOG; suppress `%p` in CTL trace events.
- **GRKERNSEC_DMESG** — restrict authorize-fail, tunnel-activate-fail, CRC-fail, "device unplugged" banners to CAP_SYSLOG.
- **PCIe tunnel CAP_SYS_ADMIN** — `tb_switch_authorize` rejects non-CAP_SYS_ADMIN callers; sysfs `authorized` store gated; only after authorization can a PCIe tunnel exist + DMA flow.
- **Control-packet validation** — `tb_ctl_handle_event` rejects packets whose `route` does not resolve to a known router in `tb_cm`, whose `port` exceeds `sw->port_count`, or whose payload length mismatches `header.len`; defense against router-side packet-injection.
- **Switch UID nonce** — challenge-response uses 32-byte cryptographically-random nonce per attempt; defense against replay; HMAC verification refuses on any mismatch.
- **Per-router HopID space bounded** — `sw->hopids` ida bounded by HW max (TB3: 8, USB4: 64); refuse to allocate beyond the cap; defense against HopID-exhaustion DoS.
- **Bandwidth-group budget enforced** — `tb_tunnel_alloc_bandwidth` refuses allocations that would push group total above link capacity; defense against starvation attack via crafted DP requests.
- **Tunnel teardown ordered** — `tb_handle_hotplug(unplug=true)` walks `tcm.tunnel_list` and deactivates from leaf-toward-host order before `tb_switch_remove`; defense against in-flight DMA continuing after switch removal.
- **CTL RX bounded** — RX ring frames pre-allocated; per-event work item bounded by `TB_TIMEOUT`; defense against CTL-flood causing soft-lockup.

Rationale: the CM is where router authorization, tunnel construction, and CTL packet handling meet. A missed CAP_SYS_ADMIN check on `authorize` lets any process open a DMA path. A racy unplug can leave a tunnel referencing freed switches (UAF + DMA after free). A weak nonce in challenge-response allows replay. PAX_REFCOUNT + kCFI on the ops vtables + bounded HopID/RX/timeout + cryptographic challenge-response convert this large, async, lock-heavy core into a verified consent + isolation pipeline.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- NHI HW (`nhi.c`, `nhi_ops.c`, `nhi_regs.h`): covered in `thunderbolt/domain.md`.
- Bus / sysfs layer (`domain.c`): covered in `thunderbolt/domain.md`.
- ICM-mode CM (`icm.c`): future `thunderbolt/icm.md` Tier-3.
- USB4 router ops detail (`usb4.c`, `usb4_port.c`): future `thunderbolt/usb4.md`.
- XDomain peer protocol (`xdomain.c`): future `thunderbolt/xdomain.md`.
- TMU / CLx / LC / retimer / NVM internals: future per-feature Tier-3s.
- Implementation code.
