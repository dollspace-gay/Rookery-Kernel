# Tier-3: net/xfrm/device — IPSec hardware offload (`xfrm_device.c`)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_device.c
  - include/net/xfrm.h
  - include/linux/netdevice.h
-->

## Summary
Tier-3 design for the IPSec NIC-offload bridge: when an SA is installed with `XFRMA_OFFLOAD_DEV` (cross-ref `net/xfrm/user.md` REQ-10), the kernel asks the named netdev's driver to install the SA on its hardware crypto engine. After install, ESP encrypt/decrypt + replay-window tracking + per-SA byte counter happen on the NIC; software path becomes a thin shim. Supported by Mellanox/NVIDIA ConnectX-6+, Intel ixgbe/i40e/ice (limited), Chelsio T6, and any driver implementing the `xdo_dev_*` netdev ops.

Two modes:
- **Crypto-only** (`XFRM_OFFLOAD_INBOUND/OUTBOUND` flags): NIC does ESP encrypt/decrypt, kernel still does encap/decap framing
- **Full** (`XFRM_OFFLOAD_FULL`): NIC does everything from L4-payload to wire (full ESP including header)

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (offload-aware SA install), `net/xfrm/ah-esp-ipv4.md` + `net/xfrm/ah-esp-ipv6.md` (offload paths in esp4_offload.c + esp6_offload.c), `net/skbuff.md` (per-skb offload state).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-netdev offload registration, SA-install/teardown bridging, GSO segmentation for offloaded SAs | `net/xfrm/xfrm_device.c` |
| Public API: `struct xfrmdev_ops`, `XFRM_OFFLOAD_*` flags, `struct xfrm_user_offload`, `struct xfrm_state.xso` | `include/net/xfrm.h`, `include/linux/netdevice.h` |

## Compatibility contract

### `struct xfrmdev_ops` (driver vtable)

```c
struct xfrmdev_ops {
    int  (*xdo_dev_state_add)    (struct xfrm_state *x);
    void (*xdo_dev_state_delete) (struct xfrm_state *x);
    void (*xdo_dev_state_free)   (struct xfrm_state *x);
    bool (*xdo_dev_offload_ok)   (struct sk_buff *skb, struct xfrm_state *x);
    void (*xdo_dev_state_advance_esn)(struct xfrm_state *x);

    int  (*xdo_dev_policy_add)   (struct xfrm_policy *x, struct netlink_ext_ack *extack);
    void (*xdo_dev_policy_delete)(struct xfrm_policy *x);
    void (*xdo_dev_policy_free)  (struct xfrm_policy *x);
};
```

Per-driver implementation; identical contract so existing drivers (ConnectX-6, ice, etc.) bind unchanged.

### `XFRM_OFFLOAD_*` flags

| Flag | Constant | Semantic |
|---|---|---|
| `XFRM_OFFLOAD_IPV6` | 1 | SA is IPv6 (vs. IPv4) |
| `XFRM_OFFLOAD_INBOUND` | 2 | RX direction offloaded |
| `XFRM_OFFLOAD_FULL` | 4 | Full-offload (incl. encap/decap) |
| `XFRM_OFFLOAD_PACKET` | 8 | Packet-offload mode (per-packet xfrm rather than per-SA) |

### `struct xfrm_user_offload` (UAPI)

```c
struct xfrm_user_offload {
    int  ifindex;
    __u8 flags;     /* XFRM_OFFLOAD_* */
};
```

User passes via `XFRMA_OFFLOAD_DEV` NLA on NEWSA / NEWPOLICY.

### Per-SA `xso` state (kernel side)

`struct xfrm_state.xso`:
```c
struct xfrm_dev_offload {
    struct net_device *dev;
    struct net_device *real_dev;
    unsigned long flags;
    enum xfrm_dev_offload_type type;
    enum xfrm_dev_offload_dir  dir;
};
```

Tracks the netdev + offload state per SA. Identical layout.

### Offload-eligible SA gate

A SA is offload-eligible if and only if:
1. `XFRMA_OFFLOAD_DEV` was supplied with a valid ifindex
2. The named netdev's `xdo_dev_state_add` returned 0 (driver accepted)
3. SA's algorithm + key-size + flags are within driver-supported set
4. SA's mode (transport/tunnel) is supported by driver

If any check fails: SA still installed in kernel, but `xso.dev = NULL` → all packets through software path. Identical fall-back logic so misconfigured offload doesn't break IPSec.

### Per-skb offload tracking (`skb->sp`)

`struct sec_path` per-skb stores per-stage SA pointer + offload state. On RX from offload-capable NIC, driver sets `XFRM_GRO` flag on skb to indicate ESP already verified. Software path on RX consults this flag to skip redundant verification. Identical algorithm.

### GSO segmentation for offloaded SAs

When TX skb is GSO-flagged + SA is offloaded: software pre-segments the inner payload (mss-sized chunks), each chunk gets its own xfrm-bundle pass; per-chunk ESP done on NIC. `esp4_gso_segment` / `esp6_gso_segment` (in `esp4_offload.c` / `esp6_offload.c`) handle the per-chunk wrap.

### Driver capability discovery

`netdev->features & NETIF_F_HW_ESP` indicates ESP-encrypt/decrypt-capable; `NETIF_F_HW_ESP_TX_CSUM` indicates per-skb checksum offload; `NETIF_F_GSO_ESP` indicates ESP-aware GSO. Identical feature flags.

### Counters

Per-netdev `xstats` carry per-SA offloaded packet/byte counters; exposed via `ip xfrm state list` `crypto-offload-stats` block. Identical format.

## Requirements

- REQ-1: `struct xfrmdev_ops` vtable: all methods (`xdo_dev_state_add/delete/free`, `xdo_dev_offload_ok`, `xdo_dev_state_advance_esn`, `xdo_dev_policy_*`) per upstream signatures.
- REQ-2: `XFRM_OFFLOAD_IPV6/INBOUND/FULL/PACKET` flags identical bit values.
- REQ-3: `struct xfrm_user_offload` UAPI byte-identical layout.
- REQ-4: Per-SA `xso` (`struct xfrm_dev_offload`) layout-equivalent.
- REQ-5: Offload-eligible gate: valid ifindex + `xdo_dev_state_add` success + driver feature support; on any failure → SA installed without offload (`xso.dev = NULL`).
- REQ-6: Per-skb `sec_path` tracks offload state; RX `XFRM_GRO` flag short-circuits software verify when set by driver.
- REQ-7: GSO segmentation for offloaded SAs: per-chunk xfrm bundle pass; ESP wrap on NIC; software pre-segmentation in `esp4_gso_segment` / `esp6_gso_segment`.
- REQ-8: Driver capability discovery via `NETIF_F_HW_ESP / HW_ESP_TX_CSUM / GSO_ESP` features.
- REQ-9: Per-SA offloaded counters: `xfrmdev_ops`-driven `xfrm_dev_state_advance_esn` + per-driver xstats; format-identical.
- REQ-10: Driver hot-unplug: when offload netdev is removed, kernel calls `xdo_dev_state_delete` for all offloaded SAs; SAs revert to software path.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct xfrm_user_offload` byte-identical layout. (covers REQ-3)
- [ ] AC-2: Driver-bind test: load an `xfrmdev_ops`-implementing driver; install SA with `XFRMA_OFFLOAD_DEV`; `ip xfrm state list` shows offload flag + ifindex. (covers REQ-1, REQ-2, REQ-5)
- [ ] AC-3: HW-fail-fallback test: install SA with offload to a netdev whose `xdo_dev_state_add` returns -EINVAL → SA installed but `xso.dev = NULL`; subsequent traffic goes through software path. (covers REQ-5)
- [ ] AC-4: GRO short-circuit test: simulated NIC sets `XFRM_GRO` on skb after HW verify; software RX path skips redundant verify. (covers REQ-6)
- [ ] AC-5: GSO test: TX 64KB skb through offloaded SA + `NETIF_F_GSO_ESP`-capable driver → pre-segmented to MSS-sized chunks; each chunk ESP-wrapped on NIC. (covers REQ-7)
- [ ] AC-6: Capability test: query `ethtool -k <iface>` for `esp-hw-offload`; capable NIC reports on, non-capable off. (covers REQ-8)
- [ ] AC-7: Counter test: send 1000 ESP packets through offloaded SA → `ip xfrm state list` shows offloaded byte/packet counters increment. (covers REQ-9)
- [ ] AC-8: Hot-unplug test: install offloaded SA on netdev N; `ip link delete dev N` (or unplug PCI) → kernel cleanly tears down offload, SA falls back; no kernel oops. (covers REQ-10)
- [ ] AC-9: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::net::xfrm::device::Offload` — top-level entrypoint
- `kernel::net::xfrm::device::Eligibility` — offload-eligible gate
- `kernel::net::xfrm::device::Install` — `xdo_dev_state_add` bridge
- `kernel::net::xfrm::device::Teardown` — `xdo_dev_state_delete/free` bridge
- `kernel::net::xfrm::device::SecPath` — per-skb `sec_path` mutator + `XFRM_GRO` flag
- `kernel::net::xfrm::device::Gso` — GSO pre-segmentation for offloaded SAs
- `kernel::net::xfrm::device::Capability` — per-netdev capability discovery
- `kernel::net::xfrm::device::Counters` — per-SA offloaded packet/byte counters
- `kernel::net::xfrm::device::HotUnplug` — netdev hot-removal → SA software fallback
- `kernel::net::xfrm::device::PolicyOffload` — `xdo_dev_policy_*` bridge

### Locking and concurrency

- **Per-netdev `xfrmdev_ops` register**: pointer set under `rtnl_lock` at driver init; read RCU-side
- **Per-SA `xso.dev` pointer**: written via `xdo_dev_state_add` callback under SA lock; read under SA lock
- **Hot-unplug serialization**: `unregister_netdevice` waits for all offloaded SAs to drain (per-driver vtable handles)

### Error handling

- `Err(EINVAL)` — bad XFRMA_OFFLOAD_DEV / unsupported flags
- `Err(ENODEV)` — bad ifindex
- `Err(EOPNOTSUPP)` — driver doesn't implement `xfrmdev_ops`
- `Err(EAGAIN)` — driver-side resource exhaustion (e.g., SA-table full)
- Any driver-returned error from `xdo_dev_state_add` → SA fall-back to software path (REQ-5)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `xfrm_dev_state_add` bridge (driver-vtable null-check + refcount sound) | `kani::proofs::net::xfrm::device::install_safety` |
| Per-SA `xso.dev` mutate under SA lock (no concurrent stale read) | `kani::proofs::net::xfrm::device::xso_safety` |
| Hot-unplug iteration over offloaded SAs (no double-free during teardown) | `kani::proofs::net::xfrm::device::hotunplug_safety` |
| GSO pre-segmentation (skb_clone bounds + per-chunk SA refcount sound) | `kani::proofs::net::xfrm::device::gso_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-SA `xso` state | `xso.dev` is non-NULL ⇔ SA was successfully offloaded; `xso.real_dev` is non-NULL when offload is via a virtual netdev (e.g., bond) | `kani::proofs::net::xfrm::device::xso_invariants` |
| Per-skb `sec_path` | when `XFRM_GRO` flag set, `sec_path` carries the offloaded SA pointer | `kani::proofs::net::xfrm::device::sp_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Offload fall-back soundness theorem** via Verus — proves: ∀ offload failure path (driver returns error / driver doesn't support feature), SA is correctly installed in software path; no inconsistent state where `xso.dev` is set but driver hasn't accepted.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-driver `struct xfrmdev_ops` instances `static const` (driver-side; documented contract) | § Mandatory |
| **MEMORY_SANITIZE** | per-SA HW-context buffers cleared on offload-teardown (driver-side may hold key material) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-SA + per-skb (cross-ref `net/xfrm/state.md`, `net/skbuff.md`)
- **CONSTIFY**: see above
- **SIZE_OVERFLOW**: per-SA byte counter arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook: none directly (offload-eligibility is determined by driver capability + SA configuration; LSM gate is at the SA-mutation level — `state.md` / `user.md`).
- Default GR-RBAC policy: empty so behavior matches upstream.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — `xfrmdev_ops` contract is exhaustively specified by upstream + the Linux IPSec offload API documentation)

## Out of Scope

- Per-driver implementations (cross-ref individual driver Tier-4 docs once Phase D begins)
- Underlying SA storage (cross-ref `net/xfrm/state.md`)
- Underlying ESP transforms (cross-ref `net/xfrm/ah-esp-ipv4.md`, `net/xfrm/ah-esp-ipv6.md`)
- 32-bit-only paths
- Implementation code
