# Tier-3: drivers/interconnect/{core,bulk}.c — ICC framework (provider-node graph + bandwidth aggregation)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/interconnect/core.c
  - drivers/interconnect/bulk.c
  - drivers/interconnect/internal.h
  - drivers/interconnect/trace.h
  - drivers/interconnect/debugfs-client.c
  - include/linux/interconnect.h
  - include/linux/interconnect-provider.h
-->

## Summary

The Interconnect (ICC) framework abstracts SoC bus / NoC bandwidth requests so consumer drivers (display, GPU, video codec, NVMe, USB, cpufreq policy) can declare their per-path bandwidth needs to an arbitrating provider (typically Qualcomm BCM, MediaTek SCMI, Samsung Exynos, IMX, etc.) without each consumer knowing the topology. Bandwidth is expressed as `(avg_bw, peak_bw)` in kBps along a directed path between named nodes (e.g. "DDR ← MNOC ← CPU"). The provider aggregates all consumer requests per node, computes new bus / PLL / DCVS targets, and applies them in a `set()` callback.

This Tier-3 covers `drivers/interconnect/core.c` (~1252 lines: the provider/node/path data model, `icc_set_bw` aggregation, OF parsing, debugfs, sync_state), `bulk.c` (~159 lines: array helpers for consumers carrying many paths), plus the consumer-facing `include/linux/interconnect.h` and provider-facing `include/linux/interconnect-provider.h` contracts.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct icc_provider` | per-provider control block (registered by SoC driver) | `drivers::interconnect::Provider` |
| `struct icc_node` | per-endpoint / per-segment node in the bus graph | `drivers::interconnect::Node` |
| `struct icc_node_data` | OF cell descriptor → node id mapping | `drivers::interconnect::NodeData` |
| `struct icc_path` | per-consumer resolved path (list of nodes) + per-node `req_node` request | `drivers::interconnect::Path` |
| `struct icc_req` | per-path-per-node bandwidth request entry | `drivers::interconnect::Req` |
| `struct icc_onecell_data` / `icc_xlate_extended_data` | OF translation tables | `drivers::interconnect::OnecellData` |
| `icc_provider_add(provider)` / `icc_provider_del(provider)` | provider register/unregister | `Provider::add` / `_del` |
| `icc_node_create(id)` / `icc_node_create_dyn()` / `icc_node_destroy(id)` | node alloc/free | `Node::create` / `_destroy` |
| `icc_node_add(node, provider)` / `icc_node_del(node)` | add/remove node from provider | `Provider::add_node` |
| `icc_link_create(node, dst_id)` / `icc_link_nodes(src, dst_node)` / `icc_link_destroy(src, dst)` | per-edge add/remove | `Node::link_create` / `_destroy` |
| `of_icc_get(dev, name)` / `_get_by_index(dev, idx)` / `devm_of_icc_get(dev, name)` | consumer path resolution from DT phandle | `Path::of_get` |
| `icc_get_by_index(dev, idx)` (sysfs `interconnects` property) | platform / sysfs variant | `Path::get_by_index` |
| `icc_set_bw(path, avg_bw, peak_bw)` | per-path bandwidth request | `Path::set_bw` |
| `icc_enable(path)` / `icc_disable(path)` | enable/disable path (pin bandwidth) | `Path::enable` / `_disable` |
| `icc_set_tag(path, tag)` | per-path tag annotation passed to aggregator | `Path::set_tag` |
| `icc_put(path)` | release path | `Path::put` |
| `icc_bulk_set_bw(num, paths)` / `icc_bulk_enable(num, paths)` / `icc_bulk_disable(...)` / `icc_bulk_put(...)` | array-helper variants | `bulk::set_bw` |
| `icc_std_aggregate(node, tag, avg_bw, peak_bw, agg_avg, agg_peak)` | default sum-avg / max-peak aggregator | `Provider::std_aggregate` |
| `icc_sync_state(dev)` | provider's `sync_state` ack after boot consumers register | `Provider::sync_state` |
| `provider->set(src, dst)` | per-edge HW programming callback | `ProviderOps::set` |
| `provider->aggregate(node, tag, avg, peak, agg_avg, agg_peak)` | per-node aggregation callback (defaults to `icc_std_aggregate`) | `ProviderOps::aggregate` |
| `provider->pre_aggregate(node)` | reset per-node accumulators | `ProviderOps::pre_aggregate` |

## Compatibility contract

REQ-1: Provider registration: `icc_provider_add(provider)` validates `provider->set` non-NULL, attaches to global `icc_providers` list under `icc_lock` (mutex); nodes added later via `icc_node_add`.

REQ-2: Node identity: each `icc_node` has a per-provider id (`u16`) unique within the provider; global identity is `(provider, id)`; OF `#interconnect-cells` controls translation arity (typically 1 or 2 cells).

REQ-3: Graph edges: `icc_link_create(node, dst_id)` appends `dst_node` to `node->links` array (per-node `num_links` capped at SoC topology); destination node must be registered or queued via `icc_node_link_orphan`.

REQ-4: Path resolution: `of_icc_get(dev, name)` parses `interconnects = <&prov src_id &prov dst_id>` (and `interconnect-names`), runs BFS from src to dst across all providers using shortest-hop, allocates `icc_path` containing the ordered node list + per-node `icc_req`.

REQ-5: Per-path-per-node request: `icc_req { node, dev, enabled, tag, avg_bw, peak_bw }`; per-node `req_list` aggregates all consumer requests under `node->mutex`.

REQ-6: Bandwidth aggregation: on `icc_set_bw`, for each node in path: `pre_aggregate(node)`; walk `node->req_list` calling `provider->aggregate(node, req.tag, req.avg_bw, req.peak_bw, &agg_avg, &agg_peak)`; default aggregator sums avg + maxes peak; per-tag separation supported.

REQ-7: HW apply: per-edge `provider->set(src, dst)` invoked walking the path under the provider's per-provider lock; per-provider serialization prevents racing programming.

REQ-8: `icc_enable` / `icc_disable` lazy-pin: `enabled=false` requests still aggregate at zero so a future `enable` does not require recomputation; `disable` clears `enabled` and re-aggregates.

REQ-9: Bulk helpers (`bulk.c`): `icc_bulk_set_bw(num, paths)` iterates array applying per-path `icc_set_bw`; failure aborts halfway and unwinds prior bw to previous values (no partial commit semantic — see Open Questions).

REQ-10: `sync_state` integration: providers declare `dev->driver->sync_state = icc_sync_state`; called after all OF consumers have probed (or timeout), letting provider drop its initial-floor bandwidth.

REQ-11: Debugfs (`debugfs-client.c`): per-provider `interconnect/` directory exposes consumer's requested paths + aggregated per-node bw; control nodes for synthetic test requests.

REQ-12: Per-provider tag space: `tag` is opaque to core; providers (e.g. Qualcomm) use it to split requests into "active-only" vs "wake" sets.

## Acceptance Criteria

- [ ] AC-1: On SDM845 / 8cx / SM8250 reference board (Qualcomm BCM-noc provider), boot succeeds with `interconnect` consumers (display, GPU, NVMe) loaded; `ls /sys/kernel/debug/interconnect/` shows provider nodes.
- [ ] AC-2: `cat /sys/kernel/debug/interconnect/interconnect_summary` shows per-consumer `(avg, peak)` requests rolled up to per-node totals matching expected sums.
- [ ] AC-3: GPU drm/msm requesting peak bandwidth on render triggers a node bw bump observable in the QCOM BCM trace.
- [ ] AC-4: `sync_state` test: a provider with `keepalive` initial bw correctly drops its floor after all consumers report.
- [ ] AC-5: Bulk helper: NVMe driver registering 4-path bulk array succeeds; `icc_bulk_set_bw` applies all four atomically wrt sync ordering with other consumers.
- [ ] AC-6: Path teardown: `icc_put(path)` removes per-node requests + triggers re-aggregation; aggregated totals decrement correctly.
- [ ] AC-7: Hot-reprobe (rebind GPU driver) leaks no per-path requests; per-node `req_list` empties on detach.
- [ ] AC-8: KUnit `icc-kunit` (`drivers/interconnect/icc-kunit.c`) passes.

## Architecture

`Provider` + `Node` + `Path` core layout:

```
struct Provider {
  provider_list: ListHead,
  nodes: ListHead,
  set: fn(src: &Node, dst: &Node) -> Result<()>,
  aggregate: Option<fn(&Node, u32, u32, u32, &mut u32, &mut u32) -> Result<()>>,
  pre_aggregate: Option<fn(&Node)>,
  xlate: fn(&OfPhandleArgs, &Provider) -> Result<&Node>,
  xlate_extended: Option<fn(...) -> NodeData>,
  get_bw: Option<fn(&Node) -> (u32, u32)>,
  dev: Arc<Device>,
  users: AtomicI32,
  inter_set: bool,
  data: *mut c_void,
  refcnt: Refcount,
}

struct Node {
  id: u16,
  name: KString,
  links: Vec<Arc<Node>>,
  num_links: u32,
  provider: Arc<Provider>,
  node_list: ListHead,
  search_list: ListHead,
  reverse: Option<&Node>,
  is_traversed: bool,
  req_list: ListHead,             // per-path req entries (locked by icc_bw_lock)
  avg_bw: u32,                    // last applied
  peak_bw: u32,
  init_avg: u32,
  init_peak: u32,
  data: *mut c_void,
}

struct Path {
  name: KString,
  num_nodes: usize,
  reqs: Vec<Req>,                  // one per node along path
  tag: u32,
}

struct Req {
  node: Arc<Node>,
  dev: Arc<Device>,
  enabled: bool,
  tag: u32,
  avg_bw: u32,
  peak_bw: u32,
}
```

Provider registration `Provider::add`:
1. Lock `icc_lock`; insert into global `icc_providers`.
2. Process pending orphan node-links registered before this provider appeared.
3. Mark provider->users = 0; expose via `of_icc_*` lookup.

Path resolution `of_icc_get(dev, name)`:
1. Resolve `interconnects = <&prov src ...>` per `interconnect-names = "name"` index.
2. Look up `(prov_phandle, args)` → `src_node` via `provider->xlate(args, provider)`.
3. Repeat for dst pair.
4. BFS via `path_find(src, dst)`: queue-based BFS marking `is_traversed`, terminating at dst, reconstructing via `reverse` links.
5. Allocate `Path` with N+1 `Req` entries (one per visited node).
6. Insert each `Req` into `node->req_list` under `icc_bw_lock`.

Bandwidth request `icc_set_bw(path, avg, peak)`:
1. Acquire `icc_bw_lock` (mutex).
2. For each node in path: `req.avg_bw = avg; req.peak_bw = peak`.
3. For each node: provider->pre_aggregate(node); walk `node->req_list` calling provider->aggregate accumulating `(agg_avg, agg_peak)`.
4. For each edge in path: `provider->set(src, dst)` apply HW (or aggregated).
5. Update `node->avg_bw` / `node->peak_bw`.
6. On any error: revert request to previous values, re-aggregate, propagate error.

Aggregation default `icc_std_aggregate(node, tag, avg, peak, &agg_avg, &agg_peak)`:
- `*agg_avg += avg`
- `*agg_peak = max(*agg_peak, peak)`

Per-provider variants override: Qualcomm uses per-tag aggregation maintaining separate "active" + "wake" sums.

`icc_enable(path)`: per-req `enabled=true` (re-aggregates); `icc_disable`: per-req `enabled=false` (re-aggregates excluding disabled).

`sync_state(dev)`: provider driver registers `dev->driver->sync_state = icc_sync_state`; ICC core walks providers with `inter_set=true` ensuring all OF consumers have probed; provider then drops its `init_avg/init_peak` floor.

Debugfs (`debugfs-client.c`): synthetic test interface — userland can create dummy consumer paths + post bw requests for fault injection / characterization.

## Hardening

- **`icc_lock` + `icc_bw_lock` separation** — registration vs bw-set lock distinct so per-set aggregation does not block provider registration.
- **Per-provider lock invocation order** — set() callbacks invoked walking the path; providers do not re-enter ICC core to avoid recursion deadlock.
- **Node refcount via `kref`** — destroy refuses with -EBUSY if non-empty `req_list`.
- **Bandwidth arithmetic overflow** — `agg_avg`, `agg_peak` use `u32`; addition checked via `check_add_overflow` (see SIZE_OVERFLOW in grsec section).
- **Orphan-link bounded** — pending node-link list bounded by provider-count; refused after deadline.
- **OF parse strict** — `xlate` returns `-EPROBE_DEFER` if dst provider not yet registered; permanent failures (bad cell count) return `-EINVAL`.
- **Bulk-helper unwinds** — `icc_bulk_set_bw` failure partway through reverts already-applied paths to previous bw before returning error.
- **`sync_state` idempotent** — second call is no-op so device-driver reload does not double-decrement floor.
- **debugfs CAP_SYS_ADMIN** — synthetic-consumer creation gated, otherwise an unprivileged user could bw-starve a victim.
- **Path teardown ordering** — `icc_put` removes per-node req from list before drop so a concurrent aggregator never sees freed req.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `icc_provider`, `icc_node`, `icc_path`, `icc_req`, and `icc_bulk_data` arrays; reject user copies of node tables outside fixed-size debugfs helpers.
- **PAX_KERNEXEC** — ICC core in W^X kernel text; `provider->set`, `provider->aggregate`, `provider->pre_aggregate`, `provider->xlate` vtable dispatched via `__ro_after_init` indirect tables per-provider.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `icc_set_bw`, `icc_get`, `icc_put`, and `sync_state` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `icc_provider`, `icc_node`, and `icc_path`; defense against teardown-vs-set races that previously underflowed kref.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `icc_node`, `icc_req`, `icc_path` reqs vector, and per-provider data blobs so prior bw requests cannot bleed into reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on debugfs synthetic-consumer entries; reject user-pointer deref outside canonical seq_file helpers.
- **PAX_RAP / kCFI** — `provider->set`, `aggregate`, `pre_aggregate`, `xlate`, `xlate_extended`, and `get_bw` vtables marked `__ro_after_init` and dispatched via kCFI-typed indirect calls.
- **GRKERNSEC_HIDESYM** — gate disclosure of per-provider data pointer and per-node `data` pointer in debugfs behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict provider register / sync_state / bw-set-failed banners to CAP_SYSLOG so attackers cannot side-channel SoC bw activity.
- **Provider node PAX_REFCOUNT** — every `icc_node` registration takes a refcount; unregister blocked while paths reference it; underflow trapped.
- **Set-bw arithmetic SIZE_OVERFLOW** — `agg_avg += avg`, `agg_peak = max(...)` wrapped with `check_add_overflow`/`u32_max`; refuse the request on overflow rather than silently saturating.
- **sync_state lock** — `sync_state` callback serialized via per-provider mutex; refuse concurrent dual-call from racing late-probe events.
- **debugfs CAP_SYS_ADMIN** — `interconnect-client` synthetic consumer requires CAP_SYS_ADMIN; without it, unprivileged users could starve actual consumers of bandwidth.
- **OF xlate validated** — `#interconnect-cells` arity strictly checked; provider rejects malformed args without dereferencing OOB.
- **bulk-helper bounded** — `num_paths` bounded vs SoC topology max; OOB array index refused.

Rationale: ICC sits between every active consumer (GPU, display, NVMe) and the SoC's actual bus / DCVS knobs. A refcount underflow, an unguarded aggregation overflow, or unprivileged debugfs access lets an attacker either crash the box (negative bw demoting NoC below floor) or DoS / brown-out a victim (saturating bw budget). RAP/kCFI on the provider vtable, CAP_SYS_ADMIN on the synthetic-consumer surface, SIZE_OVERFLOW on the aggregator, and refcount discipline on provider/node/path lifetimes turn ICC into a structurally fenced QoS arbiter rather than a soft cooperative one.

## Open Questions

- Bulk-helper "atomic" semantic across multiple providers — current code unwinds per-path on failure but does not provide cross-provider ordering guarantees beyond "best effort". Worth a TLA model.

## Out of Scope

- Per-SoC provider drivers (`qcom/`, `mediatek/`, `samsung/`, `imx/`) — each gets its own future Tier-3.
- DCVS policy details (covered in per-SoC docs).
- 32-bit-only paths.
- Implementation code.
