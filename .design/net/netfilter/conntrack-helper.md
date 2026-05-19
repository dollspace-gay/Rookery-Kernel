# Tier-3: net/netfilter/conntrack-helper — application-layer helper modules (FTP/SIP/IRC/H.323/PPTP/...)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nf_conntrack_helper.c
  - net/netfilter/nf_conntrack_amanda.c
  - net/netfilter/nf_conntrack_ftp.c
  - net/netfilter/nf_conntrack_irc.c
  - net/netfilter/nf_conntrack_pptp.c
  - net/netfilter/nf_conntrack_sip.c
  - net/netfilter/nf_conntrack_tftp.c
  - net/netfilter/nf_conntrack_sane.c
  - net/netfilter/nf_conntrack_h323_main.c
  - net/netfilter/nf_conntrack_h323_asn1.c
  - net/netfilter/nf_conntrack_netbios_ns.c
  - net/netfilter/nf_conntrack_snmp.c
  - net/netfilter/nf_conntrack_broadcast.c
  - include/net/netfilter/nf_conntrack_helper.h
-->

## Summary
Tier-3 design for the conntrack helper framework + the per-protocol helper modules. Helpers are application-layer-aware modules that parse the data stream of a primary connection and pre-register expectations for related child connections (FTP control connection's PORT command → expectation for incoming DCC TCP connection from announced port; SIP INVITE → expectations for media RTP/RTCP streams; etc.). Without helpers, application protocols that negotiate ephemeral child connections out-of-band wouldn't pass through stateful firewalls.

Modern note: per upstream's 4.7+ default, helpers are NOT auto-attached to flows; they must be explicitly attached via `iptables CT --helper <name>` (legacy) or `nft ct helper set "name"` (modern). This is a defense-in-depth response to several CVE-class helper-bypass attacks where attackers crafted traffic on unrelated ports that helpers tried to parse and ended up registering attacker-controlled expectations.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-core.md` (consumes nf_conntrack_expect via helpers), `net/netfilter/nat-core.md` (per-helper NAT counterparts in `nat-helpers.md`).

## Upstream references in scope

| Helper | Upstream path | Application |
|---|---|---|
| Framework | `nf_conntrack_helper.c` | Helper registration + auto-attach gate (default off) + per-helper hash |
| Amanda | `nf_conntrack_amanda.c` | Amanda backup protocol (RFC 8492-style) |
| FTP | `nf_conntrack_ftp.c` | RFC 959 FTP active mode (PORT/EPRT) |
| IRC | `nf_conntrack_irc.c` | DCC negotiation via PRIVMSG |
| PPTP | `nf_conntrack_pptp.c` | PPTP control + GRE data channel |
| SIP | `nf_conntrack_sip.c` | SIP INVITE → RTP/RTCP media expectations |
| TFTP | `nf_conntrack_tftp.c` | TFTP request → ephemeral data port |
| SANE | `nf_conntrack_sane.c` | SANE scanner protocol |
| H.323 | `nf_conntrack_h323_main.c` | H.225 + H.245 signaling for VoIP |
| H.323 ASN.1 | `nf_conntrack_h323_asn1.c` | ASN.1 PER-encoded H.323 message parser |
| NetBIOS-NS | `nf_conntrack_netbios_ns.c` | NetBIOS name-service broadcasts |
| SNMP | `nf_conntrack_snmp.c` | SNMP request/reply pairing |
| Broadcast helper | `nf_conntrack_broadcast.c` | Generic broadcast → unicast-replies expectation |
| Public API | `include/net/netfilter/nf_conntrack_helper.h` | nf_conntrack_helper struct + register/unregister API |

## Compatibility contract

### `struct nf_conntrack_helper` registration

```c
struct nf_conntrack_helper {
    struct hlist_node hnode;
    char name[NF_CT_HELPER_NAME_LEN];
    refcount_t refcnt;
    struct module *me;
    const struct nf_conntrack_expect_policy *expect_policy;
    struct nf_conntrack_tuple tuple;
    int (*help)(struct sk_buff *skb, unsigned int protoff, struct nf_conn *ct, enum ip_conntrack_info ctinfo);
    void (*from_nlattr)(struct nlattr *attr, struct nf_conn *ct);
    int  (*to_nlattr)(struct sk_buff *skb, const struct nf_conn *ct);
    unsigned int expect_class_max;
    unsigned int flags;
    unsigned int queue_num;     /* userspace queue number */
    u16 data_len;
    char nat_mod_name[NF_CT_HELPER_NAME_LEN];
};
```

Layout-byte-identical so existing helper modules' static instances work.

### Helper auto-attach default (CHANGED in 4.7)

Default `nf_conntrack_helper` sysctl: `0` (helpers NOT auto-attached). Operators must explicitly attach via:
- iptables: `iptables -A PREROUTING -p tcp --dport 21 -j CT --helper ftp`
- nftables: `nft add rule ip filter prerouting tcp dport 21 ct helper set "ftp"`

Identical default. Boot-time + sysctl-tunable.

### Per-helper `expect_policy`

```c
struct nf_conntrack_expect_policy {
    unsigned int max_expected;
    unsigned int timeout;
    char name[NF_CT_HELPER_NAME_LEN];
};
```

Per-helper-class limits: how many concurrent expectations per parent (defends against expectation-flooding attacks); per-class default expectation timeout. Identical defaults.

### `help` callback contract

Called from PRE_ROUTING / LOCAL_IN hooks at conntrack-helper priority for skbs whose conntrack has helper attached:

```rust
fn help(skb, protoff, ct, ctinfo) -> NF_VERDICT {
    parse_application_payload(skb, protoff)?;
    for related_flow in detected_expectations {
        nf_ct_expect_alloc(...);
        nf_ct_expect_register(...);
    }
    return NF_ACCEPT;
}
```

Per-helper specifics:

#### FTP helper (`nf_conntrack_ftp.c`)

- Active mode: parse `PORT a,b,c,d,e,f` command in client-to-server stream → expectation for incoming TCP from `a.b.c.d:e*256+f`
- Passive mode: parse `227 Entering Passive Mode (a,b,c,d,e,f)` reply in server-to-client → expectation for outgoing TCP from server side
- Extended PORT/EPSV variants for IPv6
- Tracks state across packets (split-across-segments commands)

#### SIP helper (`nf_conntrack_sip.c`)

- Parse SIP-over-UDP/TCP/SCTP messages
- INVITE/200-OK SDP body → RTP + RTCP expectations
- BYE → expectation cleanup
- Per-call-ID state tracking
- Extensive RFC 3261 message parsing — historically the largest source of conntrack-helper CVEs

#### IRC helper (`nf_conntrack_irc.c`)

- Parse PRIVMSG containing `\01DCC ...\01` payload → expectation for DCC TCP

#### PPTP helper (`nf_conntrack_pptp.c`)

- Parse PPTP control messages (TCP/1723) → register expectation for GRE-encapsulated PPP data channel
- Cooperates with `nf_conntrack_proto_gre.c` for the GRE side

#### H.323 helper (`nf_conntrack_h323_main.c` + `_asn1.c`)

- ASN.1 PER-encoded H.225 + H.245 signaling
- RAS messages (registration / admission / status)
- Q.931 call setup → RTP/RTCP expectations
- Heaviest helper in terms of code complexity; long history of CVEs in ASN.1 parser

### Per-helper NAT counterpart (deferred)

Helpers that detect application-layer addresses/ports require NAT-aware rewriting of those payload values when NAT is applied. NAT counterparts in `nat-helpers.md` (separate Tier-3) call back into helper code to coordinate the rewrite.

### Userspace helper alternative (deprecated)

Per-helper `queue_num` field allows a queue-num for `cthelper` userspace helper (alternative implementation in userspace via libnetfilter_cthelper). Modern helpers retain this for backward-compat but in-kernel helpers are preferred.

### Per-skb helper attach via `nfct_helper_attach` (CT target)

```c
extern int nf_ct_helper_attach(struct nf_conn *ct, const char *name, gfp_t gfp);
```

Called from `iptables CT --helper` / `nft ct helper set` rule paths. Per-RCU lookup of helper by name; refcount-acquire; attach to ct's helper extension.

## Requirements

- REQ-1: `struct nf_conntrack_helper` registration via `nf_conntrack_helper_register / _unregister`; per-helper hash table for name→helper lookup.
- REQ-2: `nf_conntrack_helper` sysctl: default `0` (helpers NOT auto-attached); explicit attach required via CT target.
- REQ-3: Per-helper `expect_policy` (max_expected per class + per-class timeout); enforce limits at `nf_ct_expect_register` time.
- REQ-4: All listed helpers (Amanda / FTP / IRC / PPTP / SIP / TFTP / SANE / H.323 / NetBIOS-NS / SNMP / broadcast) implemented identically.
- REQ-5: FTP helper: PORT + EPRT + 227-PASV + EPSV parsing in both directions; per-flow state spans split segments.
- REQ-6: SIP helper: full SIP-over-UDP/TCP/SCTP parser; INVITE → RTP/RTCP expectations; BYE cleanup.
- REQ-7: H.323 helper: ASN.1 PER-encoded message parser; H.225 + H.245 + RAS message handling.
- REQ-8: PPTP helper: control-channel parsing → GRE-channel expectation; cooperates with `nf_conntrack_proto_gre.c`.
- REQ-9: Per-helper NAT counterpart bridge: helper detects application-layer addresses; NAT counterpart called for payload rewrite (cross-ref `nat-helpers.md`).
- REQ-10: `nf_ct_helper_attach`: per-name lookup; per-helper refcount-acquire; identical contract.
- REQ-11: Userspace helper queue (`queue_num`) backward-compat: per-helper queue field preserved; libnetfilter_cthelper userspace helpers continue to work (deprecated but supported).
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct nf_conntrack_helper` byte-identical layout. (covers REQ-1)
- [ ] AC-2: Default-off test: `cat /proc/sys/net/netfilter/nf_conntrack_helper` returns `0`; FTP without `CT --helper ftp` rule → no expectations registered, DCC connection dropped. (covers REQ-2)
- [ ] AC-3: Explicit-attach test: `iptables -A PREROUTING -p tcp --dport 21 -j CT --helper ftp`; FTP control connection PORT cmd → expectation registered; subsequent DCC connection → IP_CT_RELATED conntrack. (covers REQ-2, REQ-4, REQ-5)
- [ ] AC-4: SIP test: SIP INVITE with SDP `c=IN IP4 1.2.3.4` + `m=audio 10000 RTP/AVP 0` → RTP expectation registered; subsequent UDP from 1.2.3.4:10000 conntracked as RELATED. (covers REQ-6)
- [ ] AC-5: H.323 test: H.225 SETUP message with H.245 address → H.245 expectation registered; subsequent connection conntracked. (covers REQ-7)
- [ ] AC-6: PPTP test: PPTP CALL_REQUEST → expectation for GRE flow with negotiated call-ID; subsequent GRE → conntracked as RELATED. (covers REQ-8)
- [ ] AC-7: expect_policy limit test: helper with `max_expected=10`; 11th expectation registration → rejected with -EBUSY. (covers REQ-3)
- [ ] AC-8: TFTP test: TFTP RRQ → ephemeral-port expectation; subsequent UDP data → RELATED. (covers REQ-4)
- [ ] AC-9: NAT-helper integration test: install SNAT + FTP helper; FTP PORT command's IP/port rewritten to NAT'd values. (covers REQ-9)
- [ ] AC-10: nf_ct_helper_attach test: programmatic attach via CT target → ct->helper non-NULL; refcount on helper module incremented. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::netfilter::conntrack::helper::Registry` — per-system helper hash + register/unregister
- `kernel::net::netfilter::conntrack::helper::Helper` — `struct nf_conntrack_helper` wrapper
- `kernel::net::netfilter::conntrack::helper::ExpectPolicy` — per-class limits
- `kernel::net::netfilter::conntrack::helper::AutoAttach` — `nf_conntrack_helper` sysctl gate
- `kernel::net::netfilter::conntrack::helper::ftp::Ftp`
- `kernel::net::netfilter::conntrack::helper::irc::Irc`
- `kernel::net::netfilter::conntrack::helper::sip::Sip`
- `kernel::net::netfilter::conntrack::helper::sip::SdpParser`
- `kernel::net::netfilter::conntrack::helper::pptp::Pptp`
- `kernel::net::netfilter::conntrack::helper::tftp::Tftp`
- `kernel::net::netfilter::conntrack::helper::sane::Sane`
- `kernel::net::netfilter::conntrack::helper::h323::H323`
- `kernel::net::netfilter::conntrack::helper::h323::asn1::Asn1Parser`
- `kernel::net::netfilter::conntrack::helper::netbios_ns::NetbiosNs`
- `kernel::net::netfilter::conntrack::helper::snmp::Snmp`
- `kernel::net::netfilter::conntrack::helper::amanda::Amanda`
- `kernel::net::netfilter::conntrack::helper::broadcast::Broadcast`
- `kernel::net::netfilter::conntrack::helper::Attach` — `nf_ct_helper_attach`

### Locking and concurrency

- **`nf_conntrack_helper_mutex`** (mutex): per-system helper-hash mutator
- **Per-helper refcount** (Refcount): held while attached to any ct
- **RCU**: per-skb help-callback dispatch RCU-side; helper unregister waits for RCU sync

### Error handling

- `Err(EBUSY)` — expect_policy max_expected reached
- `Err(EEXIST)` — duplicate helper register
- `Err(ENOENT)` — helper not registered (for attach by name)
- `Err(EINVAL)` — bad helper config
- `Err(ENOMEM)` — alloc fail
- Application-layer parse failures: counted but not propagated (helper returns NF_ACCEPT to continue)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-helper hash insert/erase under helper_mutex | `kani::proofs::net::netfilter::conntrack::helper::registry_safety` |
| FTP PORT/PASV parser bounds checking | `kani::proofs::net::netfilter::conntrack::helper::ftp_safety` |
| SIP message parser bounds + per-call-ID state machine | `kani::proofs::net::netfilter::conntrack::helper::sip_safety` |
| H.323 ASN.1 PER decoder bounds | `kani::proofs::net::netfilter::conntrack::helper::h323_asn1_safety` |
| `nf_ct_helper_attach` refcount-acquire (no use-after-free) | `kani::proofs::net::netfilter::conntrack::helper::attach_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/conntrack_state_machine.tla` from `conntrack-proto.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system helper hash | every helper has unique `name[NF_CT_HELPER_NAME_LEN]` | `kani::proofs::net::netfilter::conntrack::helper::name_invariants` |
| Per-helper expect_policy | `max_expected` ≥ 1; `timeout` > 0 | `kani::proofs::net::netfilter::conntrack::helper::policy_invariants` |
| Per-helper refcount | refcount ≥ 1 between register and unregister | `kani::proofs::net::netfilter::conntrack::helper::refcount_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Helper parser bounded-walk theorem** via Verus — proves: per-helper application-layer parser terminates within bounded iterations on any input; rejects malformed input with NF_ACCEPT (no expectation registered, packet allowed) rather than infinite-loop.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-helper module refcount uses `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-helper `nf_conntrack_helper` instances `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-helper application-layer parser arithmetic uses checked operators (CVE class: H.323-ASN.1 buffer-overflow CVEs from history) | § Mandatory |
| **MEMORY_SANITIZE** | per-helper-extension state cleared on conntrack free (carries call-IDs, ftp-state, etc.) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, SIZE_OVERFLOW, MEMORY_SANITIZE**: see above
- **USERCOPY**: skb header read for help() callback uses bound-checked accessors
- **KERNEXEC**: per-helper dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for `iptables CT --helper` / `nft ct helper set` mutations.
- Default useful GR-RBAC policy: deny `nf_ct_helper_attach` outside gradm-marked `firewall_admin` role; helpers parse untrusted application payloads and have a long CVE history.
- Default-off helper auto-attach (`nf_conntrack_helper=0`) is itself a row-2 hardening default.

### Userspace-visible behavior changes

None beyond upstream defaults (helpers are off by default per upstream 4.7+).

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

Rationale: Conntrack helpers (ftp, sip, h.323, irc, tftp, pptp, sane, snmp, amanda, netbios-ns) deep-parse L7 protocols inside kernel context to inject expectations for related connections; they are the highest-risk parser surface in conntrack. Upstream 4.7+ defaults helpers off (`nf_conntrack_helper=0`) precisely because every shipped helper has had multiple CVEs from L7 parser bugs. Expectations are themselves a finite resource and a hostile helper-attached flow can flood the expectation table.

Baseline (cross-ref `net/netfilter/00-overview.md` § Hardening):
- **PAX_USERCOPY**: helper modules use no direct user-deref; ctnetlink `CTA_HELP_NAME` parse via NLA_POLICY.
- **PAX_KERNEXEC**: `nf_conntrack_helper.help`, per-helper `expect_policy[]` placed `__ro_after_init`.
- **PAX_RANDKSTACK**: re-randomise on every `nf_ct_helper` invocation from `nf_conntrack_confirm`.
- **PAX_REFCOUNT**: `nf_conntrack_expect.use` saturating; per-helper module refcount via try_module_get bounded.
- **PAX_MEMORY_SANITIZE**: freed expectation + per-conn helper extension zero-filled before slab reuse.
- **PAX_UDEREF**: ctnetlink helper-attach attribute parse via NLA_POLICY only.
- **PAX_RAP / kCFI**: `nf_conntrack_helper.help`, `from_nlattr`, `to_nlattr` indirect calls kCFI-tagged.
- **GRKERNSEC_HIDESYM**: helper + expectation pointers never rendered to `/proc/net/nf_conntrack_expect` (only %pK).
- **GRKERNSEC_DMESG**: helper-parse-fail warns ratelimited; per-helper attack patterns surfaced via MIB counter only.

conntrack-helper-specific reinforcement:
- **Helper-module CAP_NET_ADMIN strict-in-userns** — helper-attach via ctnetlink + iptables `CT --helper` refused from non-init userns.
- **Expectation-flood gate** — per-master-conn expectation count capped at GRSEC_NF_CT_EXPECT_PER_CONN_MAX (default 4); per-net expectation total at GRSEC_NF_CT_EXPECT_MAX.
- **Per-helper opt-in via /proc/sys/net/netfilter/nf_conntrack_helper_<name>** — every helper individually-enableable, not just the global flag; default deny.
- **L7 parse pull-up strict** — every helper invokes `skb_linearize`/`pskb_may_pull` to its parse-window max before any byte-deref; refuse on fail.
- **Expectation tuple validation** — refuse expectation insert whose tuple `dst_port`/`src_port` overlap reserved kernel ranges.
- **GR-RBAC `helper_admin` role** — only marked subjects may load + attach helpers; CAP_NET_ADMIN alone insufficient under default policy.
- **Audit-log on first expectation-create per helper per net** — defends silent rogue-helper deployment.

## Open Questions

(none — helper framework + per-helper modules exhaustively specified by upstream + extensive iptables-test / conntrack-tools test coverage)

## Out of Scope

- Per-helper NAT counterparts (cross-ref future `net/netfilter/nat-helpers.md`)
- Conntrack core (cross-ref `net/netfilter/conntrack-core.md`)
- 32-bit-only paths
- Implementation code
