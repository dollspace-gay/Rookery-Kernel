# Tier-3: net/netfilter/nat-helpers — per-helper NAT counterparts (FTP/SIP/IRC/TFTP/Amanda)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nf_nat_helper.c
  - net/netfilter/nf_nat_amanda.c
  - net/netfilter/nf_nat_ftp.c
  - net/netfilter/nf_nat_irc.c
  - net/netfilter/nf_nat_sip.c
  - net/netfilter/nf_nat_tftp.c
  - net/netfilter/nf_nat_proto.c
-->

## Summary
Tier-3 design for the per-helper NAT counterparts. When a conntrack helper (cross-ref `net/netfilter/conntrack-helper.md`) detects application-layer addresses/ports in protocol payloads, the per-helper NAT module is responsible for rewriting those payload values when NAT is applied to the connection. Without per-helper NAT, FTP-NAT'd clients would announce internal-only IPs in PORT/PASV commands, and the receiver would fail to connect; SIP/IRC/TFTP face similar issues.

Each per-helper NAT module exports a callback registered via `nf_nat_helper_register` that the conntrack helper invokes when NAT mangling is needed. The shared `nf_nat_helper.c` infrastructure provides per-direction-aware payload rewriting + TCP sequence-number adjustment integration (cross-ref `conntrack-netlink.md` REQ-6).

Note: Per upstream's modern layout, **PPTP** and **H.323** NAT helpers are folded into their respective conntrack helper modules (`nf_conntrack_pptp.c` and `nf_conntrack_h323_main.c`), not separate `nf_nat_pptp.c` / `nf_nat_h323.c` files.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-helper.md` (helpers that emit application-layer expectations), `net/netfilter/nat-core.md` (NAT mapping consumed by these helpers).

## Upstream references in scope

| Helper NAT | Upstream path | Pairing helper |
|---|---|---|
| Generic NAT helper framework | `nf_nat_helper.c` | (shared) |
| Amanda | `nf_nat_amanda.c` | `nf_conntrack_amanda.c` |
| FTP | `nf_nat_ftp.c` | `nf_conntrack_ftp.c` |
| IRC (DCC) | `nf_nat_irc.c` | `nf_conntrack_irc.c` |
| SIP (RTP/RTCP) | `nf_nat_sip.c` | `nf_conntrack_sip.c` |
| TFTP | `nf_nat_tftp.c` | `nf_conntrack_tftp.c` |
| PPTP NAT | (folded into `nf_conntrack_pptp.c`) | `nf_conntrack_pptp.c` |
| H.323 NAT | (folded into `nf_conntrack_h323_main.c`) | `nf_conntrack_h323_main.c` |
| Per-l4proto NAT | `nf_nat_proto.c` (cross-ref `nat-core.md`) | (shared) |

## Compatibility contract

### Shared NAT-helper framework (`nf_nat_helper.c`)

```c
int nf_nat_mangle_tcp_packet(struct sk_buff *skb, struct nf_conn *ct, enum ip_conntrack_info ctinfo,
                              unsigned int protoff, unsigned int match_offset, unsigned int match_len,
                              const char *rep_buffer, unsigned int rep_len);
int nf_nat_mangle_udp_packet(struct sk_buff *skb, struct nf_conn *ct, enum ip_conntrack_info ctinfo,
                              unsigned int protoff, unsigned int match_offset, unsigned int match_len,
                              const char *rep_buffer, unsigned int rep_len);
```

Common payload-rewriting entry points used by per-helper NAT modules. Replaces `match_len` bytes at `match_offset` with `rep_len` bytes from `rep_buffer`. Updates:
- skb data + length
- L4 checksum (incremental update)
- TCP sequence-adjustment for downstream segments (via seqadj extension)

Identical signatures + behavior.

### Per-helper NAT registration

```c
int nf_ct_helper_set(struct nf_conntrack_helper *helper, const char *name);
void nf_nat_helper_register(struct nf_conntrack_nat_helper *nat);
```

Per-protocol module registers a `nf_conntrack_nat_helper` carrying a callback invoked from the conntrack helper's expectation-detection code:
```c
struct nf_conntrack_nat_helper {
    struct list_head list;
    char name[NF_CT_HELPER_NAME_LEN];
    struct module *module;
};
```

Each per-helper module also exports a per-helper hook function (e.g., `nat_seq_adjust`, `ftp_nat_rewrite_addr`).

### FTP NAT (`nf_nat_ftp.c`)

Rewrites IP+port in:
- Active mode: client's `PORT a,b,c,d,e,f` command
- Passive mode: server's `227 Entering Passive Mode (a,b,c,d,e,f)` reply
- Extended PORT/EPSV variants

Algorithm:
1. Conntrack helper detects PORT/PASV; calls per-helper NAT
2. NAT module computes new IP+port from per-flow NAT mapping
3. Calls `nf_nat_mangle_tcp_packet` to rewrite payload
4. Updates expectation tuple with rewritten address+port
5. Emits seqadj if payload length changed

Identical algorithm.

### SIP NAT (`nf_nat_sip.c`)

Rewrites SIP-message addresses + SDP body addresses:
- Via header (sender's local IP)
- Contact header
- SDP `c=IN IP4 ...` (connection-info IP)
- SDP `m=audio PORT RTP/AVP ...` (media port)
- SDP `o=user TIMESTAMP IPVER IP` (origin field)
- Per-INVITE/REGISTER/200-OK message types

UDP / TCP / SCTP variants handled. Identical algorithm.

### IRC NAT (`nf_nat_irc.c`)

Rewrites DCC `\01DCC ... a.b.c.d port\01` payload in PRIVMSG. Smaller scope than SIP; rewrites IP+port to NAT'd values.

### TFTP NAT (`nf_nat_tftp.c`)

TFTP server's reply uses ephemeral source port. NAT helper rewrites the expected reply tuple to use NAT'd port. Minimal mangling (TFTP RRQ/WRQ packets carry no IPs in payload — just conntrack expectation tuple needs adjustment).

### Amanda NAT (`nf_nat_amanda.c`)

Amanda backup protocol: parses Amanda-specific data ports → rewrites IPs in payload + per-data-port expectation.

### PPTP NAT (folded into `nf_conntrack_pptp.c`)

PPTP control-channel NAT rewrites Call-ID fields (PPTP's per-call identifier — analogous to per-flow IDs). NAT helper integrated rather than separate file. Identical semantics.

### H.323 NAT (folded into `nf_conntrack_h323_main.c`)

H.323's complex multi-channel architecture (Q.931 + H.245 + RAS + RTP/RTCP) requires extensive per-message-type NAT rewriting. ASN.1-PER encoded payloads → per-field rewriting. Folded into helper module since the ASN.1 parser is the same code.

### TCP sequence-number adjustment

When NAT helper rewrites TCP payload of length M with new content of length N (M ≠ N), all subsequent segments need seqno offset adjusted by (N - M). The seqadj extension (cross-ref `conntrack-netlink.md` REQ-6) tracks per-direction offset; applied per outgoing TCP segment.

## Requirements

- REQ-1: `nf_nat_mangle_tcp_packet` + `nf_nat_mangle_udp_packet` shared helpers — payload replacement + L4 checksum incremental update + seqadj integration; identical signatures.
- REQ-2: Per-helper NAT registration via `nf_nat_helper_register` (or per-helper-direct registration in conntrack helper for PPTP/H.323).
- REQ-3: FTP NAT: rewrite PORT / EPRT / 227-PASV / EPSV addresses in both directions; TCP-seqadj on length change.
- REQ-4: SIP NAT: rewrite Via / Contact / SDP `c=` / `m=` / `o=` addresses; UDP+TCP+SCTP transport variants.
- REQ-5: IRC NAT: rewrite DCC PRIVMSG embedded address + port.
- REQ-6: TFTP NAT: adjust expectation tuple to NAT'd reply-port; identical to upstream.
- REQ-7: Amanda NAT: per-message-type address rewrite.
- REQ-8: PPTP NAT (folded into conntrack helper): Call-ID rewriting for control-channel; GRE expectation tuple adjustment.
- REQ-9: H.323 NAT (folded into conntrack helper): ASN.1 PER-encoded per-field rewriting for Q.931/H.245/RAS messages.
- REQ-10: Per-l4proto NAT (cross-ref `nat-core.md`): per-TCP/UDP/SCTP/ICMP port + checksum mangling; consumed by per-helper modules.
- REQ-11: TCP seqadj application: per-direction offset tracking via `conntrack-netlink.md`'s seqadj extension.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: FTP active mode test: enable `nf_nat_ftp` + SNAT rule; FTP client behind NAT issues PORT 192.168.1.10,8,1; tcpdump on outer iface shows PORT command rewritten to public-IP+NAT'd-port; server connects back successfully. (covers REQ-1, REQ-3)
- [ ] AC-2: FTP passive mode test: server's 227-PASV reply rewritten as it traverses NAT; client connects to NAT'd outer IP+port, gets DNAT'd to server's internal IP+port. (covers REQ-3)
- [ ] AC-3: SIP NAT test: SIP INVITE with internal SDP `c=IN IP4 10.0.0.5` → tcpdump on outer iface shows SDP rewritten to `c=IN IP4 <public-ip>`; RTP flow correctly conntracked + de-NAT'd in reverse. (covers REQ-4)
- [ ] AC-4: IRC DCC test: `/dcc send` from internal IRC client → tcpdump shows DCC payload rewritten with NAT'd address+port; subsequent connection established through NAT. (covers REQ-5)
- [ ] AC-5: TFTP NAT test: internal TFTP client RRQ → server replies from ephemeral port; NAT helper adjusts expected reply tuple; server reply correctly de-NAT'd. (covers REQ-6)
- [ ] AC-6: PPTP NAT test: PPTP client CALL_REQUEST through NAT; control-channel Call-ID rewritten; subsequent GRE flow with rewritten Call-ID conntracked + NAT'd. (covers REQ-8)
- [ ] AC-7: H.323 NAT test: H.225 SETUP through NAT; ASN.1-encoded callee address rewritten; H.245 expected channels NAT'd correctly. (covers REQ-9)
- [ ] AC-8: TCP seqadj test: FTP NAT rewrites PORT command of length 30 with length 35 (5 bytes longer); subsequent TCP segments have seq adjusted by +5; receiver sees consistent bytestream. (covers REQ-1, REQ-11)
- [ ] AC-9: Amanda backup test: Amanda data-port negotiation through NAT → addresses rewritten; data flow conntracked. (covers REQ-7)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::netfilter::nat_helpers::Framework` — shared `nf_nat_mangle_tcp_packet` / `_udp_packet`
- `kernel::net::netfilter::nat_helpers::Registry` — per-helper NAT registration
- `kernel::net::netfilter::nat_helpers::ftp::FtpNat` — FTP NAT
- `kernel::net::netfilter::nat_helpers::sip::SipNat` — SIP NAT
- `kernel::net::netfilter::nat_helpers::sip::SdpRewriter` — SDP body rewriter
- `kernel::net::netfilter::nat_helpers::irc::IrcNat` — IRC DCC NAT
- `kernel::net::netfilter::nat_helpers::tftp::TftpNat` — TFTP NAT
- `kernel::net::netfilter::nat_helpers::amanda::AmandaNat` — Amanda NAT
- `kernel::net::netfilter::nat_helpers::pptp::PptpNat` — PPTP Call-ID NAT (folded with conntrack helper)
- `kernel::net::netfilter::nat_helpers::h323::H323Nat` — H.323 ASN.1 NAT (folded with conntrack helper)
- `kernel::net::netfilter::nat_helpers::SeqAdj` — TCP seqadj application

### Locking and concurrency

- **Per-conntrack `lock`** (cross-ref `conntrack-core.md`): held during NAT mangling
- **`nf_nat_helper_lock`** (mutex): per-system NAT-helper registry mutator
- **RCU**: per-helper NAT lookup RCU-side from conntrack helper

### Error handling

- `Err(EINVAL)` — bad payload format / parse failure (helper falls back to no-mangle, packet still passes)
- `Err(ENOMEM)` — alloc fail
- skb-extension fail → drop with counter

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `nf_nat_mangle_tcp_packet` payload-replace bounds (no skb-overrun on rep_len > match_len) | `kani::proofs::net::netfilter::nat_helpers::mangle_safety` |
| FTP/SIP/IRC payload-position arithmetic (no out-of-bounds skb-read) | `kani::proofs::net::netfilter::nat_helpers::parse_safety` |
| Per-helper NAT registry insert/erase under nf_nat_helper_lock | `kani::proofs::net::netfilter::nat_helpers::registry_safety` |
| TCP seqadj application (per-direction offset; no signed overflow) | `kani::proofs::net::netfilter::nat_helpers::seqadj_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/nat_translation.tla` from `nat-core.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-helper NAT registry | every entry has unique `name` | `kani::proofs::net::netfilter::nat_helpers::registry_invariants` |
| TCP seqadj per-direction state | offset_orig + offset_reply consistently applied; no double-apply on retransmit | `kani::proofs::net::netfilter::nat_helpers::seqadj_invariants` |

### Layer 4: Functional correctness (opt-in)

- **NAT-helper bidirectional consistency theorem** via Verus — proves: forward NAT helper rewrites payload to new addresses; reverse NAT helper de-NATs incoming payload back to original addresses; both directions consistent.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-helper NAT module instances `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-helper payload-position + seqadj offset arithmetic uses checked operators | § Mandatory |
| **MEMORY_SANITIZE** | freed-after-mangle scratch buffers cleared (carry pre-NAT addresses) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-conntrack (cross-ref `conntrack-core.md`)
- **CONSTIFY, SIZE_OVERFLOW, MEMORY_SANITIZE**: see above
- **USERCOPY**: skb header read/write uses bound-checked accessors
- **KERNEXEC**: per-helper dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook: same as `conntrack-helper.md` (CAP_NET_ADMIN gate; CT --helper-attached only).
- Default useful GR-RBAC policy: deny per-helper NAT registration outside gradm-marked `firewall_admin` role; helpers parse untrusted application payloads + perform mangling — historically frequent CVE source.

### Userspace-visible behavior changes

None beyond upstream defaults (helpers default-off per upstream 4.7+).

### Verification

(See § Verification above.)

## Open Questions

(none — per-helper NAT semantics exhaustively specified by upstream + iptables-test-suite NAT-helper coverage)

## Out of Scope

- Conntrack helpers (cross-ref `conntrack-helper.md`)
- NAT core (cross-ref `nat-core.md`)
- 32-bit-only paths
- Implementation code
