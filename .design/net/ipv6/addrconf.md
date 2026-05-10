# Tier-3: net/ipv6/addrconf — IPv6 address autoconfiguration (SLAAC, DAD, lifetimes)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/addrconf.c
  - net/ipv6/addrconf_core.c
  - net/ipv6/addrlabel.c
  - net/ipv6/anycast.c
  - include/net/addrconf.h
-->

## Summary
Tier-3 design for IPv6 address autoconfiguration: stateless address autoconfiguration (RFC 4862 SLAAC), Duplicate Address Detection (RFC 4862 § 5.4), per-prefix lifetimes (preferred / valid), privacy extensions (RFC 8981 — temporary addresses with rotating IIDs), router advertisement consumption + RDNSS extraction (RFC 8106), `ip -6 addr add/del` user-driven control, RTM_NEWADDR/DELADDR NETLINK_ROUTE wire interface, IPv6 address selection policy table (RFC 6724), per-interface IPv6 device state (`struct inet6_dev`), and IPv6 anycast group joining.

Sub-tier-3 of `net/ipv6/00-overview.md`. Cooperates with `net/ipv6/ndisc.md` (which delivers RA messages to addrconf) and `net/ipv6/route.md` (RA-derived routes installed). Anchors the boot-time IPv6 connectivity path on every modern Linux box.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| SLAAC + DAD + RA consumer + RTM_*ADDR + per-inet6_dev state | `net/ipv6/addrconf.c` |
| Architecture-independent helpers | `net/ipv6/addrconf_core.c` |
| RFC 6724 source-address selection labels | `net/ipv6/addrlabel.c` |
| Anycast group registry | `net/ipv6/anycast.c` |
| Public API | `include/net/addrconf.h` |

## Compatibility contract

### `ip -6 addr add 2001:db8::1/64 dev eth0` (RTM_NEWADDR)

NLA attributes: `IFA_ADDRESS`, `IFA_LOCAL`, `IFA_LABEL`, `IFA_FLAGS`, `IFA_CACHEINFO` (struct ifa_cacheinfo: ifa_prefered + ifa_valid + tstamp + cstamp), `IFA_RT_PRIORITY`, `IFA_PROTO`, `IFA_TARGET_NETNSID`. Wire format byte-identical so iproute2 + NetworkManager + systemd-networkd work unchanged.

### `/proc/net/if_inet6`

Per-address line: `addr ifindex prefixlen scope flags ifname`. Format-byte-identical so existing tools (`ifconfig -6`, custom monitoring scripts) work.

### sysctls — `/proc/sys/net/ipv6/conf/{all,default,<iface>}/`

Per-iface knobs: `accept_ra`, `accept_ra_defrtr`, `accept_ra_pinfo`, `accept_ra_rt_info_max_plen`, `accept_redirects`, `autoconf`, `dad_transmits`, `forwarding`, `mtu`, `optimistic_dad`, `disable_ipv6`, `temp_prefered_lft`, `temp_valid_lft`, `regen_max_retry`, `max_desync_factor`, `addr_gen_mode`, `stable_secret`, `use_oif_addrs_only`, `force_mld_version`, `keep_addr_on_down`, `seg6_enabled`, `seg6_require_hmac`, `enhanced_dad`, `addr_notify`, `accept_untracked_na`, `ndisc_evict_nocarrier`. Format-byte-identical so distro `sysctl.d` configs work.

### SLAAC pipeline (RFC 4862)

1. Interface `up` → kernel sends Router Solicitation (RS) to `ff02::2`
2. Router responds with Router Advertisement (RA) carrying Prefix Information Options
3. For each PIO with `A=1` (autonomous) bit:
   - Combine prefix + interface IID → candidate address
   - Run DAD (send NS to solicited-node multicast; if NA-conflict → mark `IFA_F_DADFAILED`; else wait `dad_transmits` × `RetransTimer` then mark `IFA_F_PERMANENT`)
   - Address transitions: tentative → preferred (within `prefered_lifetime`) → deprecated (within `valid_lifetime` after preferred expires) → invalid (after valid_lifetime; auto-removed)
4. Privacy extensions (RFC 8981, optional): generate temporary address with random IID; rotate at `temp_prefered_lft`

Identical state machine + timers so userspace observes identical behavior.

### `struct inet6_dev` layout

`include/net/if_inet6.h`: per-netdev IPv6 state. Fields `dev`, `addr_list`, `mc_list`, `mc_tomb`, `mc_lock`, `mc_dad_count`, `mc_v1_seen`, `mc_qrv`, `mc_maxdelay`, `mc_qi`, `mc_qri`, `mc_v1_seen`, `mc_v2_seen`, `cnf` (struct ipv6_devconf — sysctl mirror), `stats`, `ndisc_nl`, `addr_gen_mode`, `desync_factor`. Layout-equivalent for first cache-line.

### `struct inet6_ifaddr` layout

`include/net/if_inet6.h`: per-IPv6-address state. Fields `addr`, `prefix_len`, `rt_priority`, `tstamp`, `cstamp`, `valid_lft`, `prefered_lft`, `flags` (IFA_F_*), `scope`, `idev`, `ifpub` (back-pointer for RFC 8981 temp), `if_next`, `lst`, `dad_list`, `dad_probes`, `dad_nonce`. Layout-equivalent.

### Address state flags

`IFA_F_TEMPORARY`, `IFA_F_DEPRECATED`, `IFA_F_TENTATIVE`, `IFA_F_PERMANENT`, `IFA_F_HOMEADDRESS`, `IFA_F_NODAD`, `IFA_F_OPTIMISTIC`, `IFA_F_DADFAILED`, `IFA_F_NOPREFIXROUTE`, `IFA_F_MANAGETEMPADDR`, `IFA_F_STABLE_PRIVACY`. Identical bit values.

### `addr_gen_mode` (per-interface, sysctl-tunable)

| Value | Constant | IID generation |
|---|---|---|
| 0 | `IN6_ADDR_GEN_MODE_EUI64` | EUI-64 from MAC (default historical; deprecated for privacy) |
| 1 | `IN6_ADDR_GEN_MODE_NONE` | userspace-supplied only |
| 2 | `IN6_ADDR_GEN_MODE_STABLE_PRIVACY` | RFC 7217 stable-but-opaque from `stable_secret` + prefix |
| 3 | `IN6_ADDR_GEN_MODE_RANDOM` | uniformly random + DAD |

Identical mode set + behavior.

## Requirements

- REQ-1: SLAAC pipeline: PIO consumption from RA → candidate-address generation per `addr_gen_mode` → DAD → address-state transitions on lifetime expiry; identical to upstream RFC 4862.
- REQ-2: DAD: `dad_transmits` Neighbor Solicitations (default 1) sent to solicited-node multicast; on NA-conflict → IFA_F_DADFAILED; otherwise → IFA_F_PERMANENT after retransmit period.
- REQ-3: Optimistic DAD (RFC 4429): with `optimistic_dad=1`, address is usable in IFA_F_OPTIMISTIC state during DAD; on conflict, downgrade to IFA_F_DADFAILED.
- REQ-4: Privacy extensions (RFC 8981): `use_tempaddr=2` generates temporary addresses; `temp_prefered_lft` (default 86400s) + `temp_valid_lft` (default 604800s); regen on lifetime expiry.
- REQ-5: RTM_NEWADDR/DELADDR/GETADDR wire format byte-identical.
- REQ-6: Address state lifecycle: tentative → preferred → deprecated → invalid; transitions driven by `prefered_lifetime` + `valid_lifetime` from PIO or user.
- REQ-7: `addr_gen_mode` modes implemented identically: EUI-64, NONE, STABLE_PRIVACY (RFC 7217), RANDOM.
- REQ-8: `struct inet6_dev` + `struct inet6_ifaddr` first-cache-line layout-equivalent.
- REQ-9: RFC 6724 source-address selection labels (`/proc/net/ipv6_addr_label`); `ip -6 addrlabel` r/w.
- REQ-10: Anycast group join (`ipv6_sock_ac_join`/`drop`); per-netdev anycast list maintained; ND solicitations for anycast addresses delivered identically.
- REQ-11: TLA+ model `models/net/ipv6_addrconf.tla` (mandatory per `net/ipv6/00-overview.md` Layer 2) — proves: address state machine has no "stuck" states (every reachable state has a transition); DAD-on-conflict is sound (no two interfaces hold same IFA_F_PERMANENT address on the same link).
- REQ-12: Per-iface sysctls `accept_ra*`, `autoconf`, `dad_transmits`, `optimistic_dad`, `disable_ipv6`, `temp_*_lft`, `addr_gen_mode`, `stable_secret`, `keep_addr_on_down` — semantics identical.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A virtual netdev, `accept_ra=1`, `autoconf=1`; inject an RA carrying PIO `2001:db8::/64 a=1 onlink=1 prefered_lt=14400 valid_lt=86400` → kernel auto-assigns `2001:db8::<EUI64>` to the interface within `dad_transmits × RetransTimer + ε`. (covers REQ-1, REQ-2, REQ-7)
- [ ] AC-2: Same scenario with `addr_gen_mode=2` (stable_privacy) and `stable_secret` set → IID is RFC 7217 hash, not EUI-64; reproducible across reboots given same secret. (covers REQ-7)
- [ ] AC-3: DAD conflict test: pre-populate ND-cache with conflicting NA target=`2001:db8::1`; kernel SLAAC of `2001:db8::1/64` results in IFA_F_DADFAILED. (covers REQ-2)
- [ ] AC-4: Optimistic DAD (`optimistic_dad=1`): address usable in IFA_F_OPTIMISTIC immediately; conflict downgrades. (covers REQ-3)
- [ ] AC-5: Privacy extensions test (`use_tempaddr=2`): temporary address generated alongside permanent SLAAC; regenerates after `temp_prefered_lft`. (covers REQ-4)
- [ ] AC-6: `ip -6 addr show dev eth0` output byte-identical to upstream after equivalent operations; tstamp/cstamp/valid_lft/prefered_lft fields populated identically. (covers REQ-5, REQ-6)
- [ ] AC-7: `pahole struct inet6_dev` + `struct inet6_ifaddr` byte-identical first cache-line. (covers REQ-8)
- [ ] AC-8: `ip -6 addrlabel list` byte-identical content. (covers REQ-9)
- [ ] AC-9: Anycast join test: `setsockopt(IPV6_JOIN_ANYCAST, &group)` adds group; ND solicitation for the anycast address is responded to. (covers REQ-10)
- [ ] AC-10: TLA+ `models/net/ipv6_addrconf.tla` proves: state machine has no stuck states; DAD soundness invariant holds. (covers REQ-11)
- [ ] AC-11: For each per-iface sysctl in the table, a r/w round-trip returns the written value; behavior changes accordingly (e.g., `accept_ra=0` ignores RA). (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::ipv6::addrconf::Addrconf` — top-level entrypoint
- `kernel::net::ipv6::addrconf::slaac::Slaac` — RA → PIO → candidate
- `kernel::net::ipv6::addrconf::iid::IidGenerator` — EUI64 / STABLE_PRIVACY / RANDOM
- `kernel::net::ipv6::addrconf::dad::Dad` — Duplicate Address Detection
- `kernel::net::ipv6::addrconf::lifetime::LifetimeTimer` — preferred/valid expiry
- `kernel::net::ipv6::addrconf::tempaddr::Tempaddr` — RFC 8981 privacy extensions
- `kernel::net::ipv6::addrconf::dev::Inet6Dev` — `struct inet6_dev`
- `kernel::net::ipv6::addrconf::ifaddr::Inet6Ifaddr` — `struct inet6_ifaddr`
- `kernel::net::ipv6::addrconf::netlink::RtmAddrHandler` — RTM_*ADDR wire
- `kernel::net::ipv6::addrconf::sysctl::AddrconfSysctl` — per-iface sysctls
- `kernel::net::ipv6::addrlabel::AddrLabel` — RFC 6724 labels
- `kernel::net::ipv6::anycast::AnycastList` — per-iface anycast registry

### Locking and concurrency

- **Per-`inet6_dev` `lock`** (rwlock): protects per-iface address list
- **`addrconf_hash_lock`** (per-bucket spinlock): per-netns addr-hash for fast lookup
- **Per-`inet6_ifaddr` `lock`** (spinlock): protects per-address state
- **`pneigh_lock`** (cross-ref `net/core/00-overview.md`): proxy-ND for anycast
- **`addrconf_lock`** (mutex): netlink-driven RTM_NEWADDR serialization

### Error handling

- `Err(EEXIST)` — RTM_NEWADDR for an address already present
- `Err(ENODEV)` — bad ifindex
- `Err(EINVAL)` — bad NLA attributes / bad lifetime
- `Err(EADDRNOTAVAIL)` — RTM_DELADDR for non-existent
- `Err(EOPNOTSUPP)` — interface doesn't support IPv6 (e.g., loopback-only-v4)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Address-state transition state machine (tentative → preferred → deprecated → invalid) | `kani::proofs::net::ipv6::addrconf::state_safety` |
| DAD NS-build + NA-receive matching (sanity of nonce) | `kani::proofs::net::ipv6::addrconf::dad_safety` |
| RA PIO parser (bounded extension-header walk) | `kani::proofs::net::ipv6::addrconf::pio_parse_safety` |
| RFC 7217 stable-IID HMAC build | `kani::proofs::net::ipv6::addrconf::iid_stable_safety` |
| Lifetime-timer arithmetic (no overflow on jiffies addition for max-uint32 lifetimes) | `kani::proofs::net::ipv6::addrconf::lifetime_safety` |

### Layer 2: TLA+ models

- `models/net/ipv6_addrconf.tla` (mandatory per `net/ipv6/00-overview.md` Layer 2) — proves state-machine soundness + DAD non-collision. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-`inet6_dev` address list | every entry's `idev` back-pointer matches the holding inet6_dev | `kani::proofs::net::ipv6::addrconf::list_invariants` |
| Address state flags | `IFA_F_TENTATIVE` and `IFA_F_PERMANENT` are mutually exclusive at every observable state | `kani::proofs::net::ipv6::addrconf::flag_invariants` |
| Privacy-extension parent/child | every IFA_F_TEMPORARY ifaddr has `ifpub` pointing to a non-temp ifaddr on the same idev | `kani::proofs::net::ipv6::addrconf::tempaddr_invariants` |

### Layer 4: Functional correctness (opt-in)

- **DAD soundness theorem** via Verus — proves: no address completes DAD as IFA_F_PERMANENT if ND received an NA matching the target with a different L2 address.
- **RFC 7217 stable-IID determinism theorem** via Verus — proves: same `(stable_secret, prefix, ifname, dad_counter)` tuple always yields same IID; no global counter dependency.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | inet6_ifaddr + inet6_dev refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-inet6_ifaddr + per-inet6_dev slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed inet6_ifaddr cleared (the address itself is potentially network-routable info) | § Default-on configurable off |
| **LATENT_ENTROPY** | RFC 7217 `stable_secret` initialization seeded from kernel CSPRNG; `dad_nonce` similarly | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, LATENT_ENTROPY**: see above
- **CONSTIFY**: addrconf netlink-handler vtable `static const`
- **USERCOPY**: NLA parsing uses `nla_*` accessors
- **SIZE_OVERFLOW**: lifetime-timer + delta arithmetic uses checked operators (CVE class: jiffies-overflow → permanent address)

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for RTM_NEWADDR/DELADDR; GR-RBAC policy can deny per-subject (default empty).
- LSM hook `security_socket_setsockopt` for IPV6_JOIN_ANYCAST.
- Useful default policy: deny RTM_NEWADDR outside gradm-marked `network_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — addrconf semantics are exhaustively specified by RFCs 4862, 4429, 6724, 7217, 8981 + upstream)

## Out of Scope

- ND state machine itself (cross-ref `net/ipv6/ndisc.md`)
- IPv6 routing (cross-ref `net/ipv6/route.md`)
- 32-bit-only paths
- Implementation code
