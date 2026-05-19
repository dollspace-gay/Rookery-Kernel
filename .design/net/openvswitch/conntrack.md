# Tier-3: net/openvswitch/conntrack.c — OVS conntrack action (OVS_ACTION_ATTR_CT)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/openvswitch/datapath.md
upstream-paths:
  - net/openvswitch/conntrack.c (~2028 lines)
  - net/openvswitch/conntrack.h
  - include/uapi/linux/openvswitch.h (OVS_CT_ATTR_*)
-->

## Summary

OVS conntrack action (OVS_ACTION_ATTR_CT) integrates Linux Netfilter connection-tracking into the OVS datapath pipeline. Per-`CT` action: invoke nf_conntrack lookup → set per-skb ct_state in sw_flow_key (NEW/ESTABLISHED/RELATED/REPLY/INVALID/TRACKED) + ct_mark + ct_label + ct_zone. Per-CT-helper invocation (FTP/SIP/TFTP). Per-NAT (DNAT/SNAT) applied. Per-orig-tuple + per-reply-tuple stored. Per-recirculate downstream actions use ct_state for stateful matching ("allow established traffic"). Critical for: OVS-based stateful firewall (security-groups in OpenStack), service-mesh L4 LB, NAT-enabled overlays.

This Tier-3 covers `conntrack.c` (~2028 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ovs_ct_execute()` | per-CT-action entry | `OvsCt::execute` |
| `ovs_ct_copy_action()` | per-action parse | `OvsCt::copy_action` |
| `ovs_ct_init()` | per-net init | `OvsCt::init` |
| `ovs_ct_exit()` | per-net exit | `OvsCt::exit` |
| `ovs_ct_lookup()` | per-skb nf_conntrack lookup | `OvsCt::lookup` |
| `ovs_ct_commit()` | per-skb commit (per-OVS_CT_F_COMMIT) | `OvsCt::commit` |
| `ovs_ct_nat()` | per-DNAT/SNAT | `OvsCt::nat` |
| `ovs_ct_helper()` | per-CT-helper attach | `OvsCt::helper` |
| `ovs_ct_clear_action()` | per-OVS_ACTION_ATTR_CT_CLEAR | `OvsCt::clear_action` |
| `OVS_CT_ATTR_*` | per-attr UAPI | UAPI |
| `OVS_CS_F_*` | ct_state flags | UAPI |
| `ct_state` | sw_flow_key.ct_state field | shared |

## Compatibility contract

REQ-1: OVS_ACTION_ATTR_CT (nested attrs):
- OVS_CT_ATTR_COMMIT: commit to nf_conntrack table.
- OVS_CT_ATTR_ZONE: per-zone (default 0).
- OVS_CT_ATTR_MARK: per-mark + mask.
- OVS_CT_ATTR_LABELS: per-label + mask.
- OVS_CT_ATTR_HELPER: per-helper name (e.g. "ftp").
- OVS_CT_ATTR_NAT: per-NAT range.
- OVS_CT_ATTR_FORCE_COMMIT: commit even if existing.
- OVS_CT_ATTR_EVENTMASK: per-conntrack-event-mask.
- OVS_CT_ATTR_TIMEOUT: per-timeout policy.

REQ-2: ct_state flags (OVS_CS_F_*):
- NEW: first-pkt-in-conn (not yet committed).
- ESTABLISHED: in nf_conntrack table; matched.
- RELATED: related to existing (e.g. ICMP-error).
- REPLY: per-pkt reply-direction.
- INVALID: malformed; can't track.
- TRACKED: walked via CT (sets after).
- SRC_NAT / DST_NAT: NAT applied this pkt.

REQ-3: ovs_ct_execute(net, skb, key, info):
- /* Per-info.commit: full commit-path */
- /* Per-info.zone, mark, label, NAT params */
- nf_conntrack_in_compat(net, key.eth.type, &info, skb, hooknum).
- /* Sets skb._nfct */
- /* Per-helper: attach */
- if info.helper: ovs_ct_helper(skb, info.helper, info.zone).
- /* Per-NAT: apply */
- if info.nat: ovs_ct_nat(net, skb, info).
- /* Update sw_flow_key.ct_state from skb._nfct */
- ovs_ct_update_key(skb, info, key).
- /* Per-commit: confirm + insert into nf_conntrack table */
- if info.commit: nf_conntrack_confirm(skb).

REQ-4: Per-zone:
- Per-zone segregation: same 5-tuple in different zones = independent connections.
- Per-OVS_CT_ATTR_ZONE u16.

REQ-5: Per-NAT:
- OVS_NAT_ATTR_SRC / OVS_NAT_ATTR_DST: NAT direction.
- OVS_NAT_ATTR_IP_MIN / IP_MAX: target IP range.
- OVS_NAT_ATTR_PROTO_MIN / PROTO_MAX: target port range.
- OVS_NAT_ATTR_PERSISTENT / OVS_NAT_ATTR_PROTO_HASH / RANDOM: NAT algorithm.

REQ-6: Per-CT-helper:
- Per-helper name string ("ftp", "sip", "tftp", "irc").
- nf_conntrack_helper_try_module_get; attach to conn.
- Per-helper handles per-protocol app-layer (e.g. FTP-PORT command opens data-channel).

REQ-7: ovs_ct_update_key (sw_flow_key.ct_state):
- if !TRACKED: set TRACKED.
- if NEW conn: set NEW.
- if ESTABLISHED conn: set ESTABLISHED.
- if RELATED: set RELATED.
- if REPLY direction: set REPLY.
- if NAT applied: set SRC_NAT or DST_NAT.

REQ-8: Per-OVS_ACTION_ATTR_CT_CLEAR:
- Clear ct_state in key (without entering nf_conntrack).
- Used to pre-CT processing.

REQ-9: Per-namespace:
- Per-net conntrack table.
- OVS_CT_F_FORCE: force fresh tracking.

REQ-10: Per-event-mask:
- Per-conntrack-event subscription (NEW/REPLY/DESTROY/etc.).

## Acceptance Criteria

- [ ] AC-1: OVS_ACTION_ATTR_CT applied: skb._nfct set; ct_state in key.
- [ ] AC-2: OVS_CT_ATTR_COMMIT: subsequent same-5-tuple matches ESTABLISHED.
- [ ] AC-3: OVS_CT_ATTR_ZONE=42: per-zone scoped.
- [ ] AC-4: OVS_CT_ATTR_NAT DNAT: per-skb dst-IP changed.
- [ ] AC-5: OVS_CT_ATTR_HELPER="ftp": per-FTP-PORT-command opens related conn.
- [ ] AC-6: ct_state OVS_CS_F_INVALID: per-malformed pkt flagged.
- [ ] AC-7: ct_state OVS_CS_F_RELATED: per-ICMP-error tracked.
- [ ] AC-8: OVS_ACTION_ATTR_CT_CLEAR: per-skb ct_state cleared.
- [ ] AC-9: Stateful firewall rule (allow ESTABLISHED): per-recirc match.
- [ ] AC-10: Per-namespace: nsA / nsB conntrack tables independent.

## Architecture

Per-CT-action info:

```
struct OvsConntrackInfo {
  helper: Option<&NfConntrackHelper>,
  ct: Option<&NfConn>,
  zone: NfConntrackZone,
  family: u16,
  flags: u32,                                    // OVS_CT_F_*
  commit: bool,
  force: bool,
  helper_index: u32,
  labels: NfConntrackLabels,
  labels_mask: NfConntrackLabels,
  mark: u32,
  mark_mask: u32,
  nat: NfNat,
  timeout: u32,
  expect_classifier: ...
}
```

Per-sw_flow_key.ct_state:

```
const OVS_CS_F_NEW: u8 = 0x01;
const OVS_CS_F_ESTABLISHED: u8 = 0x02;
const OVS_CS_F_RELATED: u8 = 0x04;
const OVS_CS_F_REPLY_DIR: u8 = 0x08;
const OVS_CS_F_INVALID: u8 = 0x10;
const OVS_CS_F_TRACKED: u8 = 0x20;
const OVS_CS_F_SRC_NAT: u8 = 0x40;
const OVS_CS_F_DST_NAT: u8 = 0x80;
```

`OvsCt::execute(net, skb, key, info) -> i32`:
1. /* Pre-fragment IP if needed */
2. err = handle_fragments(net, key, info.zone.id, skb).
3. /* Per-info.commit: full pipeline */
4. if info.commit:
   - err = OvsCt::commit(net, skb, key, info).
5. else:
   - err = OvsCt::lookup(net, skb, key, info).
6. /* Per-helper / NAT applied within above */
7. /* Update key.ct_state from skb._nfct */
8. OvsCt::update_key(skb, info, key).
9. return err.

`OvsCt::lookup(net, skb, key, info) -> i32`:
1. /* Per-skb invoke nf_conntrack_in */
2. err = nf_conntrack_in(skb, &state).
3. /* Per-skb._nfct now set */
4. if err: return err.
5. ct = nf_ct_get(skb, &ctinfo).
6. /* Per-NAT: apply if requested */
7. if info.nat: OvsCt::nat(net, skb, info, ctinfo).

`OvsCt::commit(net, skb, key, info) -> i32`:
1. /* Mark + label set pre-commit */
2. if info.mark_mask: ovs_ct_set_mark(skb, info.mark, info.mark_mask).
3. if info.labels_mask: ovs_ct_set_labels(skb, info.labels, info.labels_mask).
4. /* Per-helper attach */
5. if info.helper: ovs_ct_helper(skb, info.helper, info.zone).
6. /* Commit to nf_conntrack table */
7. ret = nf_conntrack_confirm(skb).

`OvsCt::nat(net, skb, info, ctinfo) -> i32`:
1. range = nf_nat_range2 { min_addr, max_addr, min_proto, max_proto }.
2. if info.nat.src_nat:
   - nf_nat_setup_info(ct, &range, NF_NAT_MANIP_SRC).
3. if info.nat.dst_nat:
   - nf_nat_setup_info(ct, &range, NF_NAT_MANIP_DST).
4. nf_nat_packet(ct, ctinfo, hook, skb).

`OvsCt::update_key(skb, info, key)`:
1. nfct = skb._nfct.
2. if !nfct: clear ct_state; return.
3. key.ct_state = OVS_CS_F_TRACKED.
4. if nfct->status & IPS_CONFIRMED: key.ct_state |= ESTABLISHED.
5. else: key.ct_state |= NEW.
6. if ctinfo == IP_CT_RELATED ∨ IP_CT_RELATED_REPLY: key.ct_state |= RELATED.
7. if nfct.tuplehash[REPLY] matches: key.ct_state |= REPLY_DIR.
8. /* Per-NAT */
9. if nat_dst_applied: key.ct_state |= DST_NAT.
10. if nat_src_applied: key.ct_state |= SRC_NAT.
11. key.ct_zone = info.zone.id.
12. key.ct_mark = nfct.mark.
13. key.ct_label = nfct.labels.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ct_state_flags_valid` | INVARIANT | per-skb.ct_state bits ⊆ OVS_CS_F_*. |
| `commit_without_force_idempotent` | INVARIANT | per-commit twice same-tuple: same nf_conn. |
| `nat_mapping_consistent` | INVARIANT | per-NAT range: chosen addr/port ∈ range. |
| `zone_lookup_correct` | INVARIANT | per-(zone, tuple) lookup separate from other zones. |
| `helper_attached_pre_commit` | INVARIANT | per-info.helper: nf_conntrack_helper attached before confirm. |

### Layer 2: TLA+

`net/openvswitch/conntrack.tla`:
- Per-skb CT action: lookup or commit + NAT + helper + key-update.
- Properties:
  - `safety_no_double_commit` — per-tuple within zone: nf_conntrack_confirm called at most once.
  - `safety_NAT_applied_to_skb` — per-DNAT applied ⟹ skb.dst-IP modified.
  - `liveness_per_NEW_eventually_committed` — per-OVS_CT_F_COMMIT NEW pkt ⟹ nf_conntrack_confirm.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `OvsCt::execute` post: key.ct_state populated; skb._nfct set | `OvsCt::execute` |
| `OvsCt::lookup` post: per-skb nf_conntrack lookup result in skb._nfct | `OvsCt::lookup` |
| `OvsCt::commit` post: per-skb nf_conntrack_confirm called | `OvsCt::commit` |
| `OvsCt::nat` post: per-skb mangled per-NAT direction | `OvsCt::nat` |

### Layer 4: Verus/Creusot functional

`Per-skb OVS_ACTION_ATTR_CT: invoke nf_conntrack → set per-skb ct_state for downstream stateful matching` semantic equivalence: per-OVS Programmer's Reference + RFC-inspired stateful firewall semantics.

## Hardening

(Inherits row-1 features from `net/openvswitch/datapath.md` § Hardening.)

OVS-conntrack-specific reinforcement:

- **Per-zone scoping** — defense against cross-zone conntrack-leakage.
- **Per-helper-module-get atomic** — defense against per-helper-module-unload race.
- **Per-NAT range validated** — defense against per-NAT-config invalid range.
- **Per-CT_F_COMMIT requires nf_conntrack_confirm sequence** — defense against per-commit partial-state.
- **Per-skb._nfct refcount via NFCT_PUT** — defense against per-skb UAF on detach.
- **Per-OVS_CS_F_INVALID per-malformed only** — defense against per-arbitrary-skb claiming invalid.
- **Per-fragment reassembly before CT** — defense against per-fragmented-skb tracking-bypass.
- **Per-helper rate-limit** — defense against per-helper amplifier-DoS.
- **Per-CAP_NET_ADMIN for nf_conntrack module-load** — defense against unprivileged helper-injection.
- **Per-namespace per-net conntrack scoped** — defense against cross-ns leak.

## Grsecurity/PaX-style Reinforcement

- PAX_USERCOPY: OVS_CT_ATTR_* attribute payload (mark, label, helper, nat) copied via whitelisted slabs sized against NLA policy max.
- PAX_KERNEXEC: `ovs_ct_execute()`, `__ovs_ct_lookup()`, and nf_conntrack glue reside in .text; helper tables in .rodata.
- PAX_RANDKSTACK: stack-base randomization on action paths invoking `ovs_ct_execute()`.
- PAX_REFCOUNT: `nf_conn` reference counts via `nf_conntrack_get()`/`nf_conntrack_put()` use saturating refcount_t; cannot wrap on rapid CT churn.
- PAX_MEMORY_SANITIZE: CT zone / label storage scrubbed on `nf_conntrack_destroy()` callback before kfree.
- PAX_UDEREF: every CT attribute pointer validated before deref; `nla_get_*()` bounded by `nla_len()`.
- PAX_RAP / kCFI: `nf_conntrack_helper->help`, `nf_nat_ops`, and `ovs_ct_action` dispatch kCFI-typed.
- GRKERNSEC_HIDESYM: `/proc/net/nf_conntrack` and OVS CT dump redact kernel pointers; tuple hashes truncated where possible.
- GRKERNSEC_DMESG: CT table-full, helper-mismatch, and NAT-collision messages rate-limited.
- nf_conn PAX_REFCOUNT: `nf_conn->ct_general.use` saturating refcount_t; `nf_conntrack_confirm()` path cannot underflow on concurrent expiry.
- OVS_CT_ATTR_* validation: NLA policy enforces mark/label/helper/nat ranges; unknown attrs rejected under strict-mode.
- CAP_NET_ADMIN per-net: CT-attaching flow install gated by `ns_capable(net->user_ns, CAP_NET_ADMIN)`.
- Zone isolation: per-zone CT lookup ensures userns cannot poison parent-net CT entries.

Rationale: OVS conntrack ties nf_conn lifetime to flow lifetime, which is a recurring UAF surface (CVE-2022-4378-style refcount issues); saturating `nf_conn` refcounts plus strict OVS_CT_ATTR_* validation eliminate the documented vector.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- OVS datapath core (covered in `datapath.md` Tier-3)
- nf_conntrack core (covered in netfilter/ separately)
- nf_nat (covered separately)
- Per-helper protocol-specific (FTP/SIP/TFTP/IRC; covered separately)
- Implementation code
