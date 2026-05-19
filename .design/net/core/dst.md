# Tier-3: net/core/dst.c — destination cache (per-route dst_entry + per-skb dst-ref + RCU + GC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/core/dst.c
  - include/net/dst.h
  - net/ipv4/route.c (dst-related)
  - net/ipv6/route.c
-->

## Summary

`dst_entry` is the per-route forwarding cache binding a packet's resolved next-hop to per-route metadata (output func, MTU, hop-limit, neighbour, etc.). Per-skb `_skb_refdst` stores ref to dst-entry for that skb's path. dst.c provides per-dst alloc/free, refcount + RCU mgmt, dst-cache invalidation on route-change, ops-table dispatch (input vs output). Per-RT-genid invalidation on routing-table change. Critical for: every routed packet attaches a dst-entry; per-NIC TX uses dst.dev for transmit.

This Tier-3 covers `net/core/dst.c` (~355 lines) + `include/net/dst.h` (~621 lines) — focused on dst lifecycle.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dst_entry` | per-route cache entry | `net::core::dst::DstEntry` |
| `struct dst_ops` | per-AF dst vtable | `DstOps` |
| `dst_alloc(ops, dev, flags)` | per-dst alloc | `Dst::alloc` |
| `dst_init(dst, ops, dev, flags)` | per-dst init | `Dst::init` |
| `dst_destroy(dst)` | per-dst free | `Dst::destroy` |
| `dst_release(dst)` | per-dst dec-ref | `Dst::release` |
| `dst_release_immediate(dst)` | bypass-RCU release | `Dst::release_immediate` |
| `dst_hold(dst)` | per-dst inc-ref | `Dst::hold` |
| `dst_hold_safe(dst)` | per-dst safe-inc-ref | `Dst::hold_safe` |
| `dst_clone(dst)` | per-dst inc-ref + return | `Dst::clone` |
| `__dst_destroy_metrics_generic(dst)` | per-dst metric cleanup | `Dst::destroy_metrics_generic` |
| `dst_metric_set(dst, metric, val)` | per-dst metric update | `Dst::metric_set` |
| `dst_metric_lock(dst, metric)` | per-dst metric lock-bit | `Dst::metric_lock` |
| `dst_input(skb)` | per-skb input dispatch | `Dst::input` |
| `dst_output(net, sk, skb)` | per-skb output dispatch | `Dst::output` |
| `dst_link_failure(skb)` | per-skb dst-error notify | `Dst::link_failure` |
| `dst_blackhole_input(skb)` | drop-input | `Dst::blackhole_input` |
| `dst_blackhole_output(net, sk, skb)` | drop-output | `Dst::blackhole_output` |
| `dst_dev_put(dst)` | per-dst dev-detach | `Dst::dev_put` |

## Compatibility contract

REQ-1: Per-dst `dst_entry`:
- `next` (rcu chain).
- `ops` (KArc<DstOps>).
- `_metrics` (per-dst metric cache; pointer to read-only or per-dst).
- `expires` (per-dst expiry jiffies).
- `dev` (KArc<NetDevice>).
- `input` (per-skb input func).
- `output` (per-skb output func).
- `flags` (DST_*: NOXFRM, NOPOLICY, OBSOLETE, NOCOUNT, FAKE_RTABLE, etc.).
- `obsolete` (per-RT_GENID gen-counter).
- `header_len` (L2 header length).
- `trailer_len` (L2 trailer).
- `__refcnt` (atomic refcount).
- `__use` (use counter).
- `lastuse` (jiffies last-used).

REQ-2: Per-AF `dst_ops`:
- `family` (AF_INET / AF_INET6 / etc.).
- `protocol` (IPv4: 0x0800).
- `gc_thresh` (GC threshold).
- `gc` (per-AF GC fn).
- `check` (per-dst check + reuse).
- `default_advmss` (default advertised MSS).
- `mtu` (per-dst MTU getter).
- `cow_metrics` (per-dst metric COW).
- `destroy` (per-dst custom destroy).
- `ifdown` (per-NIC down callback).
- `negative_advice` (per-dst negative advice).
- `link_failure` (per-skb error).
- `update_pmtu` (per-skb MTU update).
- `redirect` (ICMP-redirect).
- `local_out` (per-skb local output).
- `neigh_lookup` (per-dst neighbour lookup).
- `confirm_neigh` (per-dst neigh-confirm).
- `kmem_cachep` (per-AF slab).
- `pcpuc_entries` (per-CPU entry counter).

REQ-3: Per-dst alloc:
1. dst_alloc(ops, dev, flags):
   - dst := kmem_cache_alloc(ops->kmem_cachep, GFP_ATOMIC).
   - dst_init(dst, ops, dev, flags).
   - dst->__refcnt = 1.
   - Return dst.

REQ-4: Per-dst init:
- dst.dev = dev (dev_hold).
- dst.ops = ops.
- dst.input = blackhole_input (per-AF override).
- dst.output = dst_blackhole_output (per-AF override).
- dst.flags = flags.

REQ-5: Per-dst refcount:
- dst_hold: atomic_inc(__refcnt).
- dst_release: atomic_dec; if 0: schedule destroy via call_rcu.
- dst_clone: dst_hold + return dst.

REQ-6: Per-dst RCU destroy:
- After atomic_dec → 0: call_rcu(&dst.rcu_head, dst_destroy_rcu).
- Per-RCU sync ensures readers complete before free.

REQ-7: Per-AF dispatch:
- dst.input(skb): typically ip_local_deliver / ip_forward (per-route).
- dst.output(net, sk, skb): typically ip_output / ip6_output.

REQ-8: Per-dst metrics:
- Per-RTA_METRICS array (12 entries: MTU, RTT, RTTVAR, etc.).
- Per-AF metric COW on first write.

REQ-9: Per-dst genid invalidation:
- per-net rt_genid bumped on route-change.
- Per-dst.obsolete tracked; on dispatch: if mismatch: refresh dst.

REQ-10: Per-dst link-failure:
- Per-NIC carrier-off → dst_link_failure(skb).
- Per-AF dispatches ICMP / SO_ERROR / sk_error_report.

REQ-11: Per-dst neighbour binding:
- Per-dst neighbour-cache: dst.ops->neigh_lookup(dst, skb, daddr).
- Per-route resolution: ARP/NDP via neigh.

REQ-12: Per-dst PMTU:
- Per-dst metric MTU updated via update_pmtu.
- Per-route ICMP frag-needed reduces dst.mtu.

## Acceptance Criteria

- [ ] AC-1: Per-skb dst-attach: ip_route_output_flow → dst_alloc → skb_dst_set.
- [ ] AC-2: Per-skb dst-input dispatch: ip_local_deliver vs ip_forward via dst.input.
- [ ] AC-3: Per-skb dst-output dispatch: ip_output via dst.output.
- [ ] AC-4: Per-dst refcount: dst_hold + dst_release pair; release-on-zero schedules call_rcu.
- [ ] AC-5: Per-dst genid invalidation: route-change bumps rt_genid; per-skb dst.obsolete mismatch detected.
- [ ] AC-6: Per-dst metric: ip_route_output → dst.metrics[RTAX_MTU] = path-MTU.
- [ ] AC-7: Per-NIC ifdown: dst_dev_put detaches per-dst from dev.
- [ ] AC-8: Per-dst neighbour: dst.ops->neigh_lookup returns matching neigh from arp/ndp table.
- [ ] AC-9: Per-dst link-failure: NIC carrier-off; per-skb dst_link_failure dispatched.
- [ ] AC-10: 100K-route stress: per-skb dst alloc/release < 100ns.

## Architecture

`DstEntry`:

```
struct DstEntry {
  next: KAtomicPtr<DstEntry>,
  ops: KArc<DstOps>,
  _metrics: KAtomicPtr<DstMetrics>,
  expires: u64,
  dev: KArc<NetDevice>,
  input: DstInputFn,
  output: DstOutputFn,
  flags: u32,
  obsolete: i32,
  header_len: u16,
  trailer_len: u16,
  __refcnt: AtomicI32,
  __use: AtomicI32,
  lastuse: u64,
  rcu_head: RcuHead,
  ...
}

struct DstOps {
  family: u32,
  protocol: u16,
  gc_thresh: u32,
  gc: DstGcFn,
  check: DstCheckFn,
  default_advmss: DstDefaultAdvmssFn,
  mtu: DstMtuFn,
  cow_metrics: DstCowMetricsFn,
  destroy: DstDestroyFn,
  ifdown: DstIfdownFn,
  negative_advice: DstNegativeAdviceFn,
  link_failure: DstLinkFailureFn,
  update_pmtu: DstUpdatePmtuFn,
  redirect: DstRedirectFn,
  local_out: DstLocalOutFn,
  neigh_lookup: DstNeighLookupFn,
  confirm_neigh: DstConfirmNeighFn,
  kmem_cachep: KArc<KmemCache>,
  pcpuc_entries: PerCpu<i32>,
}
```

`Dst::alloc(ops, dev, flags)`:
1. dst := kmem_cache_alloc(ops.kmem_cachep, GFP_ATOMIC).
2. If !dst: return NULL.
3. dst_init(dst, ops, dev, flags).
4. dst.__refcnt = 1.
5. atomic_inc(per_cpu(ops.pcpuc_entries, cpu)).
6. Return dst.

`Dst::release(dst)`:
1. If !dst: return.
2. n := atomic_dec_return(&dst.__refcnt).
3. If n == 0:
   - call_rcu(&dst.rcu_head, dst_destroy_rcu).

`Dst::destroy(dst)`:
1. If dst.dev: dev_hold-balance dev_put.
2. dst.ops.destroy(dst).
3. atomic_dec(per_cpu(dst.ops.pcpuc_entries, cpu)).
4. kmem_cache_free(dst.ops.kmem_cachep, dst).

`Dst::hold_safe(dst)`:
1. Loop:
   - n := atomic_read(&dst.__refcnt).
   - If n <= 0: return false (being destroyed).
   - If atomic_cmpxchg(&dst.__refcnt, n, n + 1) == n: return true.

`Dst::input(skb)`:
1. dst := skb_dst(skb).
2. dst.input(skb).

`Dst::output(net, sk, skb)`:
1. dst := skb_dst(skb).
2. Return dst.output(net, sk, skb).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dst_refcount_no_underflow` | INVARIANT | per-dst __refcnt ≥ 0; defense against double-release. |
| `dst_destroy_after_zero_ref` | UAF | dst-destroy only after refcount reaches 0. |
| `rcu_protected_destroy` | UAF | per-dst destroy via call_rcu; readers safe. |
| `metrics_array_idx_bounded` | OOB | per-RTAX_* idx < RTAX_MAX. |
| `dst_obsolete_check_pre_use` | INVARIANT | per-dst.obsolete checked vs net.rt_genid before per-skb dispatch. |

### Layer 2: TLA+

`net/core/dst_lifecycle.tla`:
- Per-dst state ∈ {Alloc, Used, Stale, Released}.
- Properties:
  - `safety_no_use_after_release` — Released dst not subsequently dispatched.
  - `safety_obsolete_eventually_refreshed` — Stale dst eventually replaced via re-route.
  - `liveness_zero_ref_eventually_destroyed` — refcount=0 eventually Released after RCU sync.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Dst::alloc` post: dst allocated; refcount = 1; per-CPU counter incremented | `Dst::alloc` |
| `Dst::release` post: refcount decremented; on zero: call_rcu scheduled | `Dst::release` |
| `Dst::input` / `Dst::output` post: per-AF input/output func called with valid dst | `Dst::input` / `Dst::output` |
| Per-dst refcount paired with hold/release | invariants on hold/release |

### Layer 4: Verus/Creusot functional

`Per-skb: dst-attached + dispatched via dst.ops + correctly released after use` semantic equivalence: per-skb the eventual transmit/receive is governed by dst's recorded routing decision.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

dst-specific reinforcement:

- **RCU-protected dst-destroy** — defense against concurrent-use-during-free UAF.
- **Per-dst refcount atomic** — defense against torn refcount on multi-CPU access.
- **Per-AF kmem_cache** — defense against cross-AF allocation pollution.
- **Per-dst.obsolete check** — defense against stale-route bypass.
- **Per-dst.dev hold/put paired** — defense against per-NIC unregister-during-dst-active.
- **Per-AF GC threshold** — defense against unbounded dst-cache growth.
- **Per-dst link_failure dispatched** — defense against silent-drop on NIC down.
- **Per-namespace per-AF kmem_cache** — defense against cross-netns dst leak.
- **Per-dst metric COW** — defense against shared-state corruption.
- **Per-dst lastuse update on hot-path** — defense against stale-LRU eviction.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX posture inherited workspace-wide:

- **PAX_USERCOPY** — strict bounds on dst-metric copies surfaced via `RTM_GETROUTE` netlink replies.
- **PAX_KERNEXEC** — `.rodata` `dst_ops` per-AF tables (`ipv4_dst_ops`, `ip6_dst_ops`, `xfrm_dst_ops`, ...); W^X for the dst-output/input dispatch.
- **PAX_RANDKSTACK** — per-softirq randomisation across `dst_output` / `dst_input` invocations.
- **PAX_REFCOUNT** — saturating `refcount_t` on `struct dst_entry` (`__refcnt`).
- **PAX_MEMORY_SANITIZE** — zero-on-free for `dst_entry` slabs and per-AF derived buffers (`rtable`, `rt6_info`).
- **PAX_UDEREF** — enforced separation between any user-surfaced dst-metric attribute and the in-kernel `dst_entry`.
- **PAX_RAP / kCFI** — forward-edge CFI on `dst->input`, `dst->output`, `dst_ops->check`, `dst_ops->gc`, `dst_ops->update_pmtu`, `dst_ops->cow_metrics`, and `dst_ops->ifdown`.
- **GRKERNSEC_HIDESYM** — dst-internal symbols withheld from non-CAP_SYSLOG kallsym readers.
- **GRKERNSEC_DMESG** — dst WARN ("dst cache overflow") gated behind CAP_SYSLOG.

dst-specific reinforcement:

- **`struct dst_entry` PAX_REFCOUNT** — saturating against per-dst pin abuse from a hostile per-AF route holder.
- **`dst_ops` PAX_RAP / kCFI** — defense against indirect-call hijack on `dst->input`/`dst->output` (universal hot-path TX/RX primitive).
- **Per-AF GC threshold** — defense against unbounded dst-cache growth (memory-exhaustion DoS).
- **`dst_link_failure` dispatched on NIC-down** — defense against silent-drop / stale-route reuse.
- **Per-namespace per-AF `kmem_cache`** — defense against cross-netns dst leak.
- **Dst-metric COW** — defense against shared-state corruption when one socket clones a route and mutates per-route metrics.

Rationale: `dst_entry` is the per-route hot-path object on every TX/RX path in the network stack; its indirect calls (`->input`/`->output`) are the single most attractive call-target hijack surface. The grsec stack enforces RAP/kCFI on those indirect calls, saturating refcount on the dst itself, and bounded GC so the cache cannot be exhausted.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IPv4 routing (covered in `net/ipv4/route.md` Tier-3)
- IPv6 routing (covered separately)
- Neighbour subsystem (covered in `net/core/neighbour.md` Tier-3)
- per-skb skb_dst_set / _check (covered in skbuff.md Tier-2)
- Implementation code
