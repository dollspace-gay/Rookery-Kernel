# Tier-3: net/netfilter/ipset/types-hash — hash family (11 set-type variants)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/ipset/ip_set_hash_gen.h
  - net/netfilter/ipset/ip_set_hash_ip.c
  - net/netfilter/ipset/ip_set_hash_mac.c
  - net/netfilter/ipset/ip_set_hash_net.c
  - net/netfilter/ipset/ip_set_hash_netiface.c
  - net/netfilter/ipset/ip_set_hash_netport.c
  - net/netfilter/ipset/ip_set_hash_netnet.c
  - net/netfilter/ipset/ip_set_hash_netportnet.c
  - net/netfilter/ipset/ip_set_hash_ipmac.c
  - net/netfilter/ipset/ip_set_hash_ipmark.c
  - net/netfilter/ipset/ip_set_hash_ipport.c
  - net/netfilter/ipset/ip_set_hash_ipportip.c
  - net/netfilter/ipset/ip_set_hash_ipportnet.c
-->

## Summary
Tier-3 design for the ipset hash family — 11 set-type variants implementing O(1)-average set-membership lookup over various key types. The single most-used ipset family: every Cilium pod-IP allow-list, Calico ipset-based egress policy, fail2ban dynamic block-list, firewalld zone-based allow/deny, and distro-specific mass-IP-blocklist setups depend on hash:* sets.

All hash variants share template-generated code from `ip_set_hash_gen.h` — a header that declares the per-element struct + per-key hash function via macros, included multiple times by per-variant .c files (similar pattern to C++ template specialization).

Sub-tier-3 of `net/netfilter/ipset/00-overview.md`. Pairs with `net/netfilter/ipset/core.md` (consumes `ip_set_type_variant` registration framework).

## Upstream references in scope

| Variant | Upstream path | Key |
|---|---|---|
| Template framework | `ip_set_hash_gen.h` | (shared) |
| `hash:ip` | `ip_set_hash_ip.c` | IP address (v4 or v6) |
| `hash:mac` | `ip_set_hash_mac.c` | MAC address (Ethernet) |
| `hash:net` | `ip_set_hash_net.c` | IP prefix (CIDR) |
| `hash:net,iface` | `ip_set_hash_netiface.c` | (CIDR, iface-name) |
| `hash:net,port` | `ip_set_hash_netport.c` | (CIDR, L4-port) |
| `hash:net,net` | `ip_set_hash_netnet.c` | (CIDR-src, CIDR-dst) |
| `hash:net,port,net` | `ip_set_hash_netportnet.c` | (CIDR-src, port, CIDR-dst) |
| `hash:ip,mac` | `ip_set_hash_ipmac.c` | (IP, MAC) — typical for ARP-spoofing-ACL |
| `hash:ip,mark` | `ip_set_hash_ipmark.c` | (IP, fwmark) |
| `hash:ip,port` | `ip_set_hash_ipport.c` | (IP, L4-port) |
| `hash:ip,port,ip` | `ip_set_hash_ipportip.c` | (src-IP, port, dst-IP) — flow-tuple |
| `hash:ip,port,net` | `ip_set_hash_ipportnet.c` | (src-IP, port, dst-CIDR) |

## Compatibility contract

### Common per-element structure pattern

Each hash variant declares a per-element struct via the template macros:
```c
/* hash:ip,port,ip example */
struct hash_ipportip4_elem {
    __be32 ip;            /* src IP */
    __be32 ip2;           /* dst IP */
    __be16 port;
    u8 proto;
    u8 padding;
};
```

The `ip_set_hash_gen.h` template generates `hash_*_test`, `hash_*_add`, `hash_*_del`, `hash_*_kadt`, `hash_*_uadt` based on the per-element struct.

### Hash function

Per `IPSET_CMD_CREATE` with optional `IPSET_ATTR_HASHSIZE` (default 1024 buckets) + `IPSET_ATTR_MAXELEM` (default 65536 elements). Per-element hash via `jhash` over the element-bytes. Resize on threshold via per-set GC.

### IPv4 vs IPv6 dual-mode

Each hash variant has separate `*4` and `*6` element structs (e.g., `hash_ip4_elem`, `hash_ip6_elem`). Per-set `family` field selects which is used; cannot mix in same set.

### CIDR handling for `hash:net*` variants

Per-element + per-set `cidr[33]` array (or `cidr[129]` for IPv6) tracking which CIDR sizes have entries. Per-test, walk only relevant CIDR sizes (those with non-zero count) to avoid wasteful per-prefix walks.

Identical optimization.

### `hash:net,iface` interface matching

Per-element interface name string (IFNAMSIZ) plus optional flag for `physdev` matching (bridge ingress/egress dev) vs regular dev.

### `hash:ip,mark` fwmark integration

Element matches if (skb-IP, skb-mark) matches stored (IP, mark). Useful for fwmark-tagged-flow allow-lists from BPF or earlier rules.

### Per-set features (extensions)

All hash variants support the full per-element extension set (TIMEOUT / COUNTERS / COMMENT / SKBINFO) per the per-set `extensions` flag-bitmap (cross-ref `core.md`).

### Per-element timeout + concurrent GC

GC walks per-bucket; expired elements removed lockless via RCU; consumers hold rcu_read_lock during test. Resize during GC handled via temporary growth.

### Hash-collision attack defense

Per-set `initval` random-seeded perturbation (LATENT_ENTROPY-CSPRNG) added to jhash inputs — defense vs. malicious hash-collision attacks (CVE-class hash-DoS exploiting predictable hash). Identical mitigation.

## Requirements

- REQ-1: Template framework `ip_set_hash_gen.h` generates per-variant test/add/del/kadt/uadt routines from per-element struct; identical signatures.
- REQ-2: All 11 hash variants registered (per the table) with byte-identical per-element struct layouts.
- REQ-3: Per-element jhash with random per-set perturbation seeded from kernel CSPRNG.
- REQ-4: IPv4 + IPv6 dual-mode: per-AF per-element struct; per-set `family` selects.
- REQ-5: `hash:net*` CIDR tracking: per-set `cidr[]` count array; per-test walk only relevant CIDR sizes.
- REQ-6: `hash:net,iface` interface matching: per-element IFNAMSIZ string + optional `physdev` flag.
- REQ-7: `hash:ip,mark` fwmark match: per-element u32 mark + IP.
- REQ-8: Per-set bucket count via `IPSET_ATTR_HASHSIZE` (default 1024); resize on threshold via GC.
- REQ-9: Per-set element cap via `IPSET_ATTR_MAXELEM` (default 65536); add fails with `EOVERFLOW` at cap (or evict-oldest if `FORCEADD`).
- REQ-10: All per-element extensions (TIMEOUT/COUNTERS/COMMENT/SKBINFO) supported per per-set `extensions` flag.
- REQ-11: Per-set GC: timeout-based eviction; RCU-side test-during-walk.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: hash:ip test: `ipset create allow hash:ip; ipset add allow 1.2.3.4; ipset test allow 1.2.3.4` returns 0; `ipset test allow 5.6.7.8` returns 1. (covers REQ-1, REQ-2)
- [ ] AC-2: hash:ip IPv6 test: `ipset create allow6 hash:ip family inet6; ipset add allow6 2001:db8::1`; lookup matches. (covers REQ-2, REQ-4)
- [ ] AC-3: hash:net CIDR test: `ipset create nets hash:net; ipset add nets 10.0.0.0/24`; lookup of 10.0.0.5 matches; 11.0.0.5 doesn't. (covers REQ-2, REQ-5)
- [ ] AC-4: hash:net,iface test: `ipset create allow hash:net,iface; ipset add allow 10.0.0.0/24,eth0`; matches packets with src ∈ 10.0.0.0/24 AND iif=eth0. (covers REQ-2, REQ-6)
- [ ] AC-5: hash:net,port test: `ipset create allow hash:net,port; ipset add allow 10.0.0.0/24,tcp:80`; matches packets src ∈ 10.0.0.0/24 AND TCP port 80. (covers REQ-2)
- [ ] AC-6: hash:net,net test: `ipset create flow hash:net,net; ipset add flow 10.0.0.0/24,192.168.1.0/24`; matches packets where src ∈ first prefix AND dst ∈ second. (covers REQ-2)
- [ ] AC-7: hash:ip,mac test: `ipset create arpsec hash:ip,mac; ipset add arpsec 10.0.0.5,aa:bb:cc:dd:ee:ff`; matches ARP frame with sender=10.0.0.5 AND src-MAC=aa:bb:cc:dd:ee:ff. (covers REQ-2)
- [ ] AC-8: hash:ip,mark test: `ipset create marked hash:ip,mark; ipset add marked 1.2.3.4,0x100`; matches packets src=1.2.3.4 AND skb->mark=0x100. (covers REQ-2, REQ-7)
- [ ] AC-9: hash:ip,port,ip test: `ipset create flow hash:ip,port,ip; ipset add flow 1.2.3.4,tcp:80,5.6.7.8`; matches 5-tuple. (covers REQ-2)
- [ ] AC-10: Bucket resize test: load 100K elements into hash:ip set with default hashsize=1024; bucket-walk depth stays bounded; resize occurs without breaking concurrent lookups. (covers REQ-8)
- [ ] AC-11: MAXELEM cap test: `ipset create cap hash:ip maxelem 100`; 101st add returns EOVERFLOW; with `forceadd`, oldest evicted. (covers REQ-9)
- [ ] AC-12: TIMEOUT test: `ipset create timed hash:ip timeout 5`; `ipset add timed 1.2.3.4`; wait 6s; `ipset test timed 1.2.3.4` returns 1. (covers REQ-10, REQ-11)
- [ ] AC-13: Hash-collision defense test: synthetic colliding inputs to a hash:ip set; per-set perturbation makes attacker-controlled collisions infeasible. (covers REQ-3)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::netfilter::ipset::types::hash::Framework` — template-equivalent generic generator
- `kernel::net::netfilter::ipset::types::hash::ip::HashIp4`, `HashIp6`
- `kernel::net::netfilter::ipset::types::hash::mac::HashMac`
- `kernel::net::netfilter::ipset::types::hash::net::HashNet4`, `HashNet6`
- `kernel::net::netfilter::ipset::types::hash::netiface::HashNetiface4`, `HashNetiface6`
- `kernel::net::netfilter::ipset::types::hash::netport::HashNetport4`, `HashNetport6`
- `kernel::net::netfilter::ipset::types::hash::netnet::HashNetnet4`, `HashNetnet6`
- `kernel::net::netfilter::ipset::types::hash::netportnet::HashNetportnet4`, `HashNetportnet6`
- `kernel::net::netfilter::ipset::types::hash::ipmac::HashIpmac4`, `HashIpmac6`
- `kernel::net::netfilter::ipset::types::hash::ipmark::HashIpmark4`, `HashIpmark6`
- `kernel::net::netfilter::ipset::types::hash::ipport::HashIpport4`, `HashIpport6`
- `kernel::net::netfilter::ipset::types::hash::ipportip::HashIpportip4`, `HashIpportip6`
- `kernel::net::netfilter::ipset::types::hash::ipportnet::HashIpportnet4`, `HashIpportnet6`
- `kernel::net::netfilter::ipset::types::hash::Resize` — bucket-resize logic
- `kernel::net::netfilter::ipset::types::hash::Cidr` — per-set CIDR-count tracking

### Locking and concurrency

- **Per-set `lock`** (cross-ref `core.md`): held during mutations
- **Per-bucket spinhash**: rhashtable-style per-bucket lock
- **RCU**: per-element test-during-walk RCU-side
- **Per-set GC timer**: hrtimer; lockless per-element teardown via RCU-defer

### Error handling

- `Err(EINVAL)` — bad NLA / element type mismatch
- `Err(EEXIST)` — duplicate element
- `Err(ENOENT)` — element not found (for delete)
- `Err(EOVERFLOW)` — set at maxelem cap
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-element jhash arithmetic (per-set perturbation; no integer overflow on partial-element bytes) | `kani::proofs::net::netfilter::ipset::types::hash::jhash_safety` |
| Per-bucket insert/erase under per-bucket lock (rhashtable-style) | `kani::proofs::net::netfilter::ipset::types::hash::bucket_safety` |
| Resize during concurrent test (RCU-side reader sees old-or-new bucket layout, never partial) | `kani::proofs::net::netfilter::ipset::types::hash::resize_safety` |
| `hash:net*` CIDR-count tracking (per-CIDR count consistent with element count) | `kani::proofs::net::netfilter::ipset::types::hash::cidr_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-set hashtable | every element appears in exactly the bucket determined by `jhash(key, perturbation) mod buckets` | `kani::proofs::net::netfilter::ipset::types::hash::table_invariants` |
| Per-set CIDR-count | `cidr[i] = count of elements with prefix-length i` | `kani::proofs::net::netfilter::ipset::types::hash::cidr_invariants` |
| Per-set element count | matches Σ per-bucket element counts | `kani::proofs::net::netfilter::ipset::types::hash::count_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Hash-perturbation soundness** via Verus — proves: per-set perturbation seeded from kernel CSPRNG; attacker without read-access to the seed cannot construct colliding inputs efficiently.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-element refcount via per-bucket atomic counters | § Mandatory |
| **AUTOSLAB** | per-variant element slab caches | § Mandatory |
| **CONSTIFY** | per-variant `ip_set_type_variant` instances `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed elements cleared (carry IPs/MACs/marks) | § Default-on configurable off |
| **LATENT_ENTROPY** | per-set hash perturbation seeded from kernel CSPRNG (defense vs hash-collision DoS) | § Default-on configurable off |
| **SIZE_OVERFLOW** | per-bucket count + element-count arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE, LATENT_ENTROPY, SIZE_OVERFLOW**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors (uadt path)
- **KERNEXEC**: per-variant dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `core.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — hash-type kadt/uadt nlattr ingress (IPSET_ATTR_IP, _CIDR, _PORT, _PROTO, _MAC, _IFACE, _MARK) bounded by per-type NLA policies.
- **PAX_KERNEXEC** — per-type `ip_set_type_variant` op pointer arrays (hash:ip, hash:net, hash:ip,port, hash:ip,port,net, hash:net,iface, etc.) live in R-X text.
- **PAX_RANDKSTACK** — `hash_ip4_kadt` / `hash_net6_kadt` / `hash_ipport4_kadt` family stacks re-randomised; per-call key + jhash seed locals unpredictable.
- **PAX_REFCOUNT** — per-set element count vs `maxelem` saturate-trap; per-set ref + extension refs cannot wrap under add storms.
- **PAX_MEMORY_SANITIZE** — hash bucket entries + extensions (counter, comment, skbinfo, timeout) zeroed on free so stale IP/port cannot leak to a recycled slot.
- **PAX_UDEREF** — every CIDR/netmask path validates 0..32 / 0..128 bounds before deref.
- **PAX_RAP/kCFI** — `htype->variant->kadt/uadt` indirect dispatch signed with hash-family CFI signature; cross-family vtable trap.
- **GRKERNSEC_HIDESYM** — hash-set bucket pointers never leak via netlink list dump.
- **GRKERNSEC_DMESG** — element-overflow / parse-error messages rate-limited.
- **Per-set `maxelem` strict clamp on add** — defense against per-set OOM via unbounded growth.
- **Per-set `hashsize` clamp + power-of-two enforcement** — defense against per-bucket-storm pathological collisions.
- **Per-set jhash seed per-instance random** — defense against per-precomputed-collision DoS.
- **Per-set `IPSET_ATTR_CIDR` 0..32/128 bound** — defense against per-out-of-range CIDR triggering UB in netmask shift.
- **Per-set `IPSET_ATTR_PROTO` whitelist (TCP/UDP/UDPLite/SCTP/ICMP)** — defense against per-unhandled-proto path.
- **Per-set timeout-extension monotonic clock guard** — defense against per-time-warp expiry skew.

Rationale: hash families are the most-used ipset implementation and are exposed to attacker-controlled keys + CIDR + port tuples; PAX_REFCOUNT on maxelem, per-instance jhash seed, and PAX_RAP on kadt/uadt dispatch are the critical hardening against bucket-collision DoS and vtable confusion across hash subtypes.

## Open Questions

(none — hash-family ABI exhaustively specified by upstream + libipset regression-test corpus)

## Out of Scope

- bitmap family (cross-ref `net/netfilter/ipset/types-bitmap.md`)
- list family (cross-ref `net/netfilter/ipset/types-list.md`)
- 32-bit-only paths
- Implementation code
