# Tier-3: net/core/flow_dissector.c — per-skb 5-tuple/header extraction (RPS/RFS hash + tc-flower / eBPF)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/core/dev.md
upstream-paths:
  - net/core/flow_dissector.c (~2098 lines)
  - include/net/flow_dissector.h (~488 lines)
-->

## Summary

`flow_dissector` walks a per-skb's network headers (Ethernet → VLAN → ARP/IPv4/IPv6 → TCP/UDP/SCTP/ICMP/AH/ESP/L2TPv3/GRE/MPLS/Geneve/VXLAN/etc.) extracting requested **flow keys** into a per-cb opaque target struct: 5-tuple (src/dst IP, src/dst port, proto), VLAN tag, MPLS label, GRE-key, encap inner-tuple, TCP flags, ICMP id/seq, IPSEC SPI. Uses per-(flow_dissector, dissector_key) target table — caller declares which keys it cares about. Critical for: skb_get_hash (RPS/RSS/RFS), tc-flower classification, eBPF flow-dissector programs (BPF_PROG_TYPE_FLOW_DISSECTOR), GSO segmentation (GSO_INNER_*).

This Tier-3 covers `flow_dissector.c` (~2098 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct flow_dissector` | per-config which-keys | `FlowDissector` |
| `struct flow_dissector_key` | per-key entry (ID + offset) | `FlowDissectorKey` |
| `struct flow_keys` | per-skb default 5-tuple+more target | `FlowKeys` |
| `enum flow_dissector_key_id` | per-key constant (CONTROL/BASIC/IPV4/IPV6/PORTS/...) | `FlowDissectorKeyId` |
| `__skb_flow_dissect()` | per-skb walk | `FlowDissector::dissect` |
| `skb_flow_dissect()` | per-skb default-target dissect | `FlowDissector::dissect_default` |
| `skb_flow_dissector_init()` | per-flow_dissector init | `FlowDissector::init` |
| `skb_flow_dissector_target()` | per-key offset → ptr | `FlowDissector::target` |
| `dissector_uses_key()` | per-flow_dissector has-key | `FlowDissector::uses_key` |
| `skb_get_hash()` | per-skb 4-tuple hash | `Skb::get_hash` |
| `__flow_hash_from_keys()` | per-FlowKeys jhash | `FlowDissector::hash_from_keys` |
| `skb_flow_dissect_meta()` | per-skb meta extract | `FlowDissector::dissect_meta` |
| `skb_flow_dissect_ct()` | per-conntrack extract | `FlowDissector::dissect_ct` |
| `bpf_flow_dissect()` | per-eBPF program path | `Bpf::flow_dissect` |
| `init_default_flow_dissectors()` | per-net default dissector | `FlowDissector::init_default` |

## Compatibility contract

REQ-1: `struct flow_dissector_key`:
- `key_id` (enum flow_dissector_key_id).
- `offset` (within target struct).
- Per-key registered statically.

REQ-2: `flow_dissector_key_id` constants (subset):
- FLOW_DISSECTOR_KEY_CONTROL (mandatory; flags + thoff + addr_type).
- FLOW_DISSECTOR_KEY_BASIC (n_proto + ip_proto).
- FLOW_DISSECTOR_KEY_IPV4_ADDRS / IPV6_ADDRS.
- FLOW_DISSECTOR_KEY_PORTS / PORTS_RANGE.
- FLOW_DISSECTOR_KEY_VLAN / CVLAN.
- FLOW_DISSECTOR_KEY_TCP (flags only).
- FLOW_DISSECTOR_KEY_ICMP (icmp.type + code + id).
- FLOW_DISSECTOR_KEY_ARP.
- FLOW_DISSECTOR_KEY_MPLS.
- FLOW_DISSECTOR_KEY_GRE_KEYID.
- FLOW_DISSECTOR_KEY_ENC_KEYID / ENC_IPV4_ADDRS / ENC_PORTS (for tunnels).
- FLOW_DISSECTOR_KEY_IPSEC.
- FLOW_DISSECTOR_KEY_L2TPV3.
- FLOW_DISSECTOR_KEY_META (skb metadata: ingress_ifindex / src_user_ns).
- FLOW_DISSECTOR_KEY_NUM (max).

REQ-3: `__skb_flow_dissect(net, skb, flow_dissector, target_container, data, proto, nhoff, hlen, flags)`:
- Walk per-skb header.
- Per-encountered protocol: extract per-key into target_container at flow_dissector[key].offset.
- Per-flag FLOW_DISSECTOR_F_PARSE_1ST_FRAG: stop at IP-fragment.
- Per-flag FLOW_DISSECTOR_F_STOP_AT_ENCAP: don't dive into tunnels.
- Per-flag FLOW_DISSECTOR_F_STOP_AT_FLOW_LABEL: stop at IPv6-flowlabel.

REQ-4: Per-Linux network dissection chain:
- L2: ETH_P_8021Q / 8021AD / IPV6 / IPV4 / ARP / MPLS / etc.
- L3 IPv4: walks options; reads per-pkt fields.
- L3 IPv6: walks ext-headers (Hop-By-Hop, Routing, Fragment, AH, Destination, MH).
- L4: TCP / UDP / SCTP / ICMP / GRE / IPIP / SIT / IPV6 (encap) / L2TPv3 / AH / ESP.
- Encap: inside GRE/VXLAN/Geneve/IP-in-IP → recurse with FLOW_DISSECTOR_KEY_ENC_*.

REQ-5: skb_get_hash flow:
- If skb.l4_hash: return skb.hash.
- Else: __skb_flow_dissect(net=NULL, skb, &flow_keys_dissector, &keys, ...) + jhash3(keys.basic.ip_proto, keys.addrs, keys.ports).
- Per-flow.hash := computed; skb.l4_hash := 1.

REQ-6: Per-eBPF override:
- Per-namespace BPF_PROG_TYPE_FLOW_DISSECTOR program.
- Per-skb dispatched via bpf_flow_dissect → BPF program returns per-key offsets.

REQ-7: Per-conntrack integration:
- skb_flow_dissect_ct(skb, dissector, target, ...) extracts per-skb's nf_conn_zone / nf_ct mark / labels.

REQ-8: Per-meta extraction:
- skb_flow_dissect_meta extracts skb.skb_iif (ingress_ifindex).

REQ-9: Per-tunnel extraction:
- FLOW_DISSECTOR_F_STOP_BEFORE_ENCAP: stop at outer.
- Else: dive via ip_tunnel_info_opts.

REQ-10: Per-encryption extraction:
- IPSEC: AH / ESP → FLOW_DISSECTOR_KEY_IPSEC.SPI.

## Acceptance Criteria

- [ ] AC-1: skb_flow_dissect on TCP/IPv4 skb: PORTS + IPV4_ADDRS extracted.
- [ ] AC-2: skb_flow_dissect on TCP/IPv6: IPV6_ADDRS + PORTS extracted.
- [ ] AC-3: skb_flow_dissect on VXLAN-encap: outer + inner ENC_* extracted.
- [ ] AC-4: skb_flow_dissect on IPv6 fragmented (FLOW_DISSECTOR_F_PARSE_1ST_FRAG): first-frag walked; subsequent frags' L4 not extracted.
- [ ] AC-5: skb_get_hash returns deterministic hash for same 5-tuple.
- [ ] AC-6: skb_get_hash returns same hash for fragmented and non-fragmented skb (with PARSE_1ST_FRAG).
- [ ] AC-7: BPF flow-dissector program: returns per-key offsets; called instead of native.
- [ ] AC-8: Per-MPLS label: extracted into FLOW_DISSECTOR_KEY_MPLS.
- [ ] AC-9: Per-VLAN-tagged: VLAN + CVLAN extracted.
- [ ] AC-10: Per-tc-flower: flower-classifier extracts FlowKeys via flow_dissector.

## Architecture

Per-flow_dissector config:

```
struct FlowDissector {
  used_keys: u64,                                 // bitmap of used FlowDissectorKeyId
  offset: [u16; FLOW_DISSECTOR_KEY_MAX],
}

struct FlowDissectorKey {
  key_id: FlowDissectorKeyId,
  offset: u16,                                    // within caller's target struct
}
```

Per-default target:

```
struct FlowKeys {
  control: FlowDissectorKeyControl,               // flags + thoff + addr_type
  basic: FlowDissectorKeyBasic,                   // n_proto + ip_proto
  addrs: FlowDissectorKeyAddrs,                   // v4 or v6
  ports: FlowDissectorKeyPorts,                   // src + dst u16
  tags: FlowDissectorKeyTags,
  vlan: FlowDissectorKeyVlan,
  cvlan: FlowDissectorKeyVlan,
  enc_addrs: FlowDissectorKeyAddrs,
  enc_ports: FlowDissectorKeyPorts,
  enc_keyid: FlowDissectorKeyKeyid,
  flow_label: u32,
  hash: u32,
  n_proto: u16,
  ip_proto: u8,
  ...
}
```

`FlowDissector::init(fd, keys, num_keys)`:
1. fd.used_keys = 0.
2. for k in keys[0..num_keys]:
   - fd.used_keys |= 1 << k.key_id.
   - fd.offset[k.key_id] = k.offset.

`FlowDissector::dissect(net, skb, fd, target, data, proto, nhoff, hlen, flags) -> bool`:
1. If BPF_PROG dissector enabled: bpf_flow_dissect() → return.
2. Otherwise: walk loop:
   ```
   loop {
     match proto {
       ETH_P_8021Q | ETH_P_8021AD => parse_vlan(); proto = inner_proto; nhoff += 4.
       ETH_P_IP => parse_ipv4(); set IPV4_ADDRS; proto = ip_proto; nhoff += iph_hlen.
       ETH_P_IPV6 => parse_ipv6(); walk extension-headers; set IPV6_ADDRS.
       ETH_P_MPLS_UC | ETH_P_MPLS_MC => parse_mpls(); set MPLS.
       ETH_P_ARP => parse_arp(); set ARP.
       _ => break.
     }
   }
   match ip_proto {
     IPPROTO_TCP => parse_tcp(); set PORTS, TCP.flags.
     IPPROTO_UDP | IPPROTO_UDPLITE => parse_udp(); set PORTS.
     IPPROTO_ICMP => parse_icmp(); set ICMP.
     IPPROTO_GRE => parse_gre(); set GRE_KEYID; encap recurse.
     IPPROTO_AH | IPPROTO_ESP => parse_ipsec(); set IPSEC.
     IPPROTO_L2TPV3 => parse_l2tpv3(); set L2TPV3.
     ...
   }
   if encap_dissect:
     recurse with FLOW_DISSECTOR_KEY_ENC_* prefix.
   ```

`FlowDissector::target(fd, key_id, target_container) -> &mut T`:
1. Assert(fd.used_keys & (1 << key_id)).
2. Return target_container + fd.offset[key_id].

`FlowDissector::uses_key(fd, key_id) -> bool`:
1. Return fd.used_keys & (1 << key_id) != 0.

`FlowDissector::hash_from_keys(keys) -> u32`:
1. ip_proto = keys.basic.ip_proto.
2. h = jhash3(keys.addrs.v4.src, keys.addrs.v4.dst, ip_proto | (keys.ports.src_dst << 8), seed).
3. Return h ?: 1 (avoid 0).

`Skb::get_hash(skb) -> u32`:
1. If skb.l4_hash: return skb.hash.
2. keys = FlowKeys::default().
3. FlowDissector::dissect(NULL, skb, &flow_keys_dissector, &keys, ..., 0).
4. skb.hash = hash_from_keys(&keys).
5. skb.l4_hash = 1.
6. Return skb.hash.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dissect_within_skb_len` | INVARIANT | per-walk: nhoff ≤ skb.len. |
| `key_id_lt_max` | INVARIANT | per-FlowDissectorKey.key_id < FLOW_DISSECTOR_KEY_MAX. |
| `target_offset_within_caller` | INVARIANT | per-target_container access uses fd.offset[key_id]. |
| `loop_terminates` | INVARIANT | per-walk terminates (bounded by header-walk depth). |
| `parse_1st_frag_stops_at_frag` | INVARIANT | flag PARSE_1ST_FRAG ⟹ subsequent frags' L4 not extracted. |

### Layer 2: TLA+

`net/core/flow_dissector.tla`:
- Per-skb header walk + per-key extraction.
- Properties:
  - `safety_no_oob_read` — per-walk: per-bytes accessed ⊆ skb.data.
  - `safety_per_key_extracted_iff_used` — per-target field set ⟹ fd.uses_key(key_id).
  - `liveness_walk_terminates` — per-skb ⟹ dissect terminates within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FlowDissector::init` post: per-key offsets recorded; used_keys bitmap | `FlowDissector::init` |
| `FlowDissector::dissect` post: target fields populated for fd.uses_key(key) | `FlowDissector::dissect` |
| `FlowDissector::target` post: returned ptr within caller's target_container | `FlowDissector::target` |
| `Skb::get_hash` post: returned hash deterministic for same 5-tuple | `Skb::get_hash` |

### Layer 4: Verus/Creusot functional

`Per-skb (TCP/IPv4) → FlowKeys{addrs.v4, ports.src/dst, basic.ip_proto=TCP} extracted with control.thoff = IP+TCP-header-len` semantic equivalence: per-Linux dissector matches RFC-defined per-protocol header layout.

## Hardening

(Inherits row-1 features from `net/core/dev.md` § Hardening.)

Flow-dissector-specific reinforcement:

- **Per-walk header-bounds enforced** — defense against malformed-pkt OOB read.
- **Per-loop iter-limit** — defense against infinite header-recursion.
- **Per-encap-stop flag honored** — defense against per-malicious-tunnel deep recursion DoS.
- **Per-frag PARSE_1ST_FRAG stops at frag** — defense against per-fragmented-skb OOB.
- **Per-eBPF program rate-limited** — defense against malicious BPF-flow-dissector.
- **Per-target_container size validated** — defense against per-key offset > target-size.
- **Per-IPSEC SPI extracted only on AH/ESP** — defense against type confusion.
- **Per-VLAN double-tag bounded** — defense against 802.1Q-stacking-bomb.
- **Per-MPLS label-stack bounded** — defense against MPLS-bomb.
- **Per-jhash seed per-boot random** — defense against hash-flood DoS.
- **Per-skb_get_hash cached** — defense against repeat-dissect cost amplification.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX posture inherited workspace-wide:

- **PAX_USERCOPY** — strict bounds on `BPF_PROG_TYPE_FLOW_DISSECTOR` user prog-load buffers and `bpf_flow_keys` introspection paths.
- **PAX_KERNEXEC** — `.rodata` per-protocol dissect tables (`flow_dissector_funcs`) and the per-class hash dispatch.
- **PAX_RANDKSTACK** — per-softirq randomisation on `__skb_flow_dissect` invocations.
- **PAX_REFCOUNT** — saturating `refcount_t` on `struct bpf_prog` attached as a per-netns flow dissector.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-skb `flow_keys` shadows allocated on overflow paths.
- **PAX_UDEREF** — enforced isolation between BPF user-prog buffers and the in-kernel flow-dissector call chain.
- **PAX_RAP / kCFI** — forward-edge CFI on the per-protocol dissect callbacks and the JIT-compiled BPF entry.
- **GRKERNSEC_HIDESYM** — flow-dissector internal + BPF JIT symbols withheld from non-CAP_SYSLOG kallsym readers.
- **GRKERNSEC_DMESG** — dissector WARN gated behind CAP_SYSLOG.

flow_dissector-specific reinforcement:

- **PAX_KERNEXEC W^X on the BPF flow-dissector JIT pages** — defense against W^X bypass that would turn an attached flow-dissector BPF prog into an arbitrary in-kernel exec primitive.
- **Per-protocol parser strictly bounded** — each `IPPROTO_*`/`ETH_P_*` dissect path length-checked against remaining `skb->len`; defense against per-OOB-read via crafted header chain.
- **IPSEC SPI extracted only on AH/ESP** — defense against type-confusion in the dissect-result.
- **VLAN double-tag bounded** — defense against 802.1Q-stacking-bomb.
- **MPLS label-stack walk bounded** — defense against MPLS-bomb.
- **`jhash` seed per-boot randomised** — defense against hash-flood DoS via predictable bucket selection.

Rationale: the flow dissector is invoked on the RX hot path for every skb, optionally through user-attached BPF. PAX_KERNEXEC W^X on the JIT page and forward-edge CFI on the per-protocol dissect callbacks ensure neither a hostile BPF prog nor a crafted packet chain can corrupt control flow or read past the linear region.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/core/dev (covered in `dev.md` Tier-3)
- BPF subsystem (covered separately)
- tc-flower classifier (covered separately)
- Conntrack subsystem (covered separately)
- Implementation code
