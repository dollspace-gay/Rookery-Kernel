# Tier-2: net/xfrm — IPSec / XFRM transformation framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/
  - net/key/af_key.c
  - net/ipv4/ah4.c
  - net/ipv4/esp4.c
  - net/ipv4/ipcomp.c
  - net/ipv4/xfrm4_*.c
  - net/ipv6/ah6.c
  - net/ipv6/esp6.c
  - net/ipv6/ipcomp6.c
  - net/ipv6/xfrm6_*.c
  - include/net/xfrm.h
  - include/uapi/linux/xfrm.h
-->

## Summary
Tier-2 overview for the XFRM (IPSec) transformation framework: the per-netns Security Association Database (SADB), Security Policy Database (SPD), the per-packet RX/TX transform pipeline (`xfrm_input` / `xfrm_output`), per-packet replay-window tracking, AH/ESP/IPCOMP per-protocol transforms (per RFCs 4302, 4303, 3173) for both IPv4 and IPv6, the IKE-userspace cooperation interfaces (XFRM netlink — `xfrm_user.c` per RFC 7619, plus the legacy PF_KEYv2 — `af_key.c` per RFC 2367), the encrypted-IKE-over-TCP `espintcp` (RFC 8229), the cipher / authenticator algorithm dispatch (`xfrm_algo.c`), the optional NIC-offload path (`xfrm_device.c`), `xfrm_interface` (XFRMi virtual netdev — RFC 9347 IPTFS-included), NAT keepalive, BPF integration for state inspection, and the per-netns sysctl surface.

XFRM is what "Linux IPSec" actually is in 2026: a pluggable transformation framework where IKEv2 daemons (strongSwan, libreswan, iked, wireguard-go-fallback) install SA + policies, and the kernel applies them to RX/TX packets at the L3 boundary.

Sub-tier-2 of `net/00-overview.md`.

## Scope

This Tier-2 governs **all** of `/home/doll/linux-src/net/xfrm/` (~25 source files), `net/key/af_key.c`, the per-AF IPSec adapters in `net/ipv4/{ah,esp,ipcomp,xfrm4_*}.c` and `net/ipv6/{ah,esp,ipcomp,xfrm6_*}.c`, plus the public API + UAPI headers. Per-file Tier-3 docs live under `.design/net/xfrm/`.

## Compatibility contract — outline

### XFRM Netlink (NETLINK_XFRM family)

`xfrm_user.c` exposes the modern userspace control interface. Message types per `include/uapi/linux/xfrm.h`:
- `XFRM_MSG_NEWSA` / `UPDSA` / `DELSA` / `GETSA` (SA mutation)
- `XFRM_MSG_NEWPOLICY` / `UPDPOLICY` / `DELPOLICY` / `GETPOLICY` (policy mutation)
- `XFRM_MSG_FLUSHSA` / `FLUSHPOLICY` (bulk)
- `XFRM_MSG_ALLOCSPI` / `EXPIRE` / `ACQUIRE` (helper messages)
- `XFRM_MSG_REPORT` / `MIGRATE` / `MAPPING` / `SETSPDINFO` / `GETSPDINFO`
- `XFRM_MSG_NEWAE` / `GETAE` (anti-replay info)

Wire format byte-identical so strongSwan, libreswan, iked, networkd-style tooling work unchanged.

### PF_KEYv2 (legacy — `af_key.c`)

RFC 2367 / RFC 4301 §B; `socket(PF_KEY, SOCK_RAW, PF_KEY_V2)`. Messages: SADB_ADD/DELETE/GET/EXPIRE/ACQUIRE etc. Wire format byte-identical so legacy IKEv1 daemons (racoon, ipsec-tools) work.

### `/proc/net/xfrm_stat`

Per-netns counters: `XfrmInError`, `XfrmInBufferError`, `XfrmInHdrError`, `XfrmInNoStates`, `XfrmInStateProtoError`, `XfrmInStateModeError`, `XfrmInStateSeqError`, `XfrmInStateExpired`, `XfrmInStateMismatch`, `XfrmInStateInvalid`, `XfrmInTmplMismatch`, `XfrmInNoPols`, `XfrmInPolBlock`, `XfrmInPolError`, `XfrmAcquireError`, `XfrmFwdHdrError`, `XfrmOutError`, `XfrmOutBundleGenError`, `XfrmOutBundleCheckError`, `XfrmOutNoStates`, `XfrmOutStateProtoError`, `XfrmOutStateModeError`, `XfrmOutStateSeqError`, `XfrmOutStateExpired`, `XfrmOutPolBlock`, `XfrmOutPolDead`, `XfrmOutPolError`, `XfrmFwdHdrError`, `XfrmOutStateInvalid`, `XfrmAcquireError`, `XfrmOutStateDirError`, `XfrmInStateDirError`, `XfrmInIptfsError`, `XfrmOutNoQueueSpace`. Format-byte-identical so existing parsers work.

### sysctls — `/proc/sys/net/core/xfrm_*`

Per-netns: `xfrm_aevent_etime`, `xfrm_aevent_rseqth`, `xfrm_larval_drop`, `xfrm_acq_expires`. Format byte-identical.

### XFRM virtual interface (XFRMi — RFC 9347-aware)

`ip link add type xfrm dev <underlying> if_id <id>` creates a netdev; packets entering it are classified by `if_id` matching XFRM policy templates. Identical iproute2 wire format.

### NIC offload (XFRM offload)

When `dev->features` includes `NETIF_F_HW_ESP` / `NETIF_F_HW_ESP_TX_CSUM` / `NETIF_F_HW_TLS_RX`: kernel offers SA installation to driver; driver may accept and handle ESP encrypt/decrypt in hardware. Identical UAPI + driver API.

### `struct xfrm_state` (SA) layout

`include/net/xfrm.h`: per-SA descriptor. Fields `id` (struct xfrm_id: daddr, spi, proto), `sel` (struct xfrm_selector), `props` (struct xfrm_state_walk: replay, lifetime, …), `lft`, `curlft`, `cur_curlft`, `replay`, `replay_esn`, `mark`, `if_id`, `tfcpad`, `xfrag` (frag-after-encrypt). Layout-equivalent for first cache-line.

### `struct xfrm_policy` (policy) layout

Per-policy descriptor. Fields `selector`, `priority`, `index`, `mark`, `if_id`, `bydst`, `byidx`, `lft`, `curlft`, `walk`, `xfrm_nr`, `family`, `action`, `flags`, `xfrm_vec[]` (template array). Layout-equivalent.

## Tier-3 docs governed by this Tier-2

(Phase C will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `net/xfrm/state.md` | SADB + SA mutation + lookup (`xfrm_state.c`, `xfrm_hash.[ch]`) |
| `net/xfrm/policy.md` | SPD + policy mutation + lookup (`xfrm_policy.c`) |
| `net/xfrm/user.md` | NETLINK_XFRM wire (`xfrm_user.c`) |
| `net/xfrm/af-key.md` | PF_KEYv2 legacy (`net/key/af_key.c`) |
| `net/xfrm/input.md` | RX transform pipeline (`xfrm_input.c`) |
| `net/xfrm/output.md` | TX transform pipeline (`xfrm_output.c`) |
| `net/xfrm/replay.md` | anti-replay window (`xfrm_replay.c`) |
| `net/xfrm/algo.md` | crypto algo registration (`xfrm_algo.c`) |
| `net/xfrm/device.md` | NIC-offload (`xfrm_device.c`) |
| `net/xfrm/interface.md` | XFRMi virtual netdev (`xfrm_interface_core.c`) |
| `net/xfrm/iptfs.md` | IPTFS RFC 9347 (`xfrm_iptfs.c`) |
| `net/xfrm/espintcp.md` | RFC 8229 IKE-over-TCP (`espintcp.c`) |
| `net/xfrm/nat-keepalive.md` | RFC 3948 NAT-T keepalive (`xfrm_nat_keepalive.c`) |
| `net/xfrm/ah-esp-ipv4.md` | per-AF v4 transforms (`ah4.c`, `esp4.c`, `ipcomp.c`, `xfrm4_*.c`) |
| `net/xfrm/ah-esp-ipv6.md` | per-AF v6 transforms (`ah6.c`, `esp6.c`, `ipcomp6.c`, `xfrm6_*.c`) |
| `net/xfrm/bpf.md` | BPF state introspection (`xfrm_state_bpf.c`, `xfrm_interface_bpf.c`) |
| `net/xfrm/compat.md` | 32-bit-userspace compat (`xfrm_compat.c`) — out-of-scope for v0 (REQ-O3 below) |

## Compatibility outline (top-level)

- REQ-O1: NETLINK_XFRM wire format byte-identical for all message types listed above.
- REQ-O2: PF_KEYv2 (`af_key.c`) wire format byte-identical (legacy IKEv1 daemons).
- REQ-O3: 32-bit userspace compat (`xfrm_compat.c`) DEFERRED for v0 (Rookery is x86_64-only); revisit for v1+ if 32-bit userspace matters.
- REQ-O4: SADB lookup hot-path: `xfrm_state_lookup` / `xfrm_state_lookup_byaddr` / `xfrm_state_lookup_byspi` semantics identical; bucket sizing identical.
- REQ-O5: SPD lookup hot-path: `xfrm_policy_lookup_bytype` / `__xfrm_policy_check` / `__xfrm_route_forward` semantics identical.
- REQ-O6: Per-netns SADB + SPD; lookups within netns scope only; identical netns isolation.
- REQ-O7: AH (RFC 4302), ESP (RFC 4303), IPCOMP (RFC 3173) per-AF for IPv4 and IPv6; wire format byte-identical.
- REQ-O8: Anti-replay window (RFC 4302/4303 § 3.4.3 + RFC 4304 ESN extension) — both legacy 32-bit replay window and ESN replay-window paths.
- REQ-O9: NAT-Traversal (RFC 3948): UDP encapsulation of ESP on port 4500; NAT keepalive path; identical.
- REQ-O10: NIC offload (CONFIG_XFRM_OFFLOAD): driver-aware SA installation; identical contract for HW-supporting drivers.
- REQ-O11: XFRMi virtual netdev: per-`if_id` classification of TX/RX; identical to upstream.
- REQ-O12: TLA+ models declared at this Tier-2 (see § Verification — Layer 2 below).
- REQ-O13: Hardening: row-1 features applied; default-on configurable per `00-security-principles.md`.

## Acceptance Criteria (top-level)

- [ ] AC-O1: `ip xfrm state list` + `ip xfrm policy list` byte-identical content after equivalent IKEv2-driven setup. (covers REQ-O1, REQ-O4, REQ-O5)
- [ ] AC-O2: `setkey -DP` (PF_KEYv2) byte-identical content after equivalent setup. (covers REQ-O2)
- [ ] AC-O3: A strongSwan IKEv2 site-to-site tunnel between two Rookery hosts → traffic flows; pcap shows ESP-encapsulated traffic byte-identical to upstream's. (covers REQ-O1, REQ-O4, REQ-O5, REQ-O7)
- [ ] AC-O4: Anti-replay test: replay an already-seen ESP packet → drop + counter `XfrmInStateSeqError++`. (covers REQ-O8)
- [ ] AC-O5: NAT-T test: tunnel through a NAT box → UDP-encapsulated ESP on port 4500 transits; keepalive prevents NAT timeout. (covers REQ-O9)
- [ ] AC-O6: NIC-offload test on supporting NIC: XFRM_OFFLOAD-capable driver accepts SA install; `ip xfrm state list` shows offload flag; throughput exceeds CPU-only path. (covers REQ-O10)
- [ ] AC-O7: XFRMi test: `ip link add foo type xfrm dev eth0 if_id 0x42` + policy template with `if_id=0x42` → packets out of `foo` get policy-matched. (covers REQ-O11)
- [ ] AC-O8: TLA+ models per § Verification all pass `make tla`. (covers REQ-O12)
- [ ] AC-O9: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O13)

## Architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. The shared abstractions:

- `kernel::net::xfrm::state::Sadb` — per-netns SA database
- `kernel::net::xfrm::policy::Spd` — per-netns policy database
- `kernel::net::xfrm::input::XfrmInput` — RX transform pipeline
- `kernel::net::xfrm::output::XfrmOutput` — TX transform pipeline
- `kernel::net::xfrm::replay::Replay`, `ReplayEsn` — anti-replay windows
- `kernel::net::xfrm::algo::AlgoRegistry` — cipher / auth registration
- `kernel::net::xfrm::user::XfrmUserNetlink` — NETLINK_XFRM
- `kernel::net::xfrm::pfkey::PfKeyV2` — PF_KEYv2 legacy
- `kernel::net::xfrm::device::Offload` — NIC offload
- `kernel::net::xfrm::iface::Xfrmi` — XFRMi virtual netdev
- `kernel::net::xfrm::iptfs::Iptfs` — IPTFS (RFC 9347)
- `kernel::net::xfrm::espintcp::EspInTcp` — RFC 8229
- `kernel::net::xfrm::keepalive::NatKeepalive`

## Verification (top-level)

### Layer 1: Kani SAFETY proofs

Each Tier-3 child doc declares its own.

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/net/xfrm_replay.tla` | `net/xfrm/replay.md` (RFC 4302/4303 § 3.4.3 anti-replay window invariants — packet acceptance window monotonicity, no double-deliver, ESN sequence-roll handling) |
| `models/net/xfrm_state_machine.tla` | `net/xfrm/state.md` (SA lifecycle: VALID → EXPIRING → DEAD; transitions sound under concurrent xfrm_user mutations + RX/TX consumers) |
| `models/net/xfrm_policy_eval.tla` | `net/xfrm/policy.md` (policy-tree evaluation: priority + selector match + template-application atomicity) |

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **AH/ESP packet-format theorem** via Verus — proves: serialized AH/ESP wire format equals the RFC 4302/4303 spec for any input.
- **Anti-replay correctness theorem** via TLA+ refinement — proves: kernel never delivers a sequence number it already accepted.
- **Per-AF transform-pipeline isolation theorem** via Verus — proves: v4 and v6 transforms cannot leak state across AF.

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | xfrm_state + xfrm_policy + per-template + per-bundle refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-state + per-policy slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed xfrm_state + xfrm_policy cleared (sensitive: cipher keys, authenticator keys, replay window state) | § Default-on configurable off |
| **LATENT_ENTROPY** | xfrm_alloc_spi initial SPI seed from kernel CSPRNG (RFC 4303 § 2.1 — SPI must be unpredictable) | § Default-on configurable off |
| **SIZE_OVERFLOW** | replay-window sequence-number arithmetic uses checked operators (CVE class: ESN sequence-rollover misordering) | § Mandatory |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, LATENT_ENTROPY, SIZE_OVERFLOW**: see above
- **CONSTIFY**: `xfrm_state_afinfo`, `xfrm_policy_afinfo`, per-protocol `xfrm_type` ops vtables `static const`
- **USERCOPY**: NETLINK_XFRM + PF_KEYv2 message parsing uses bound-checked accessors
- **KERNEXEC**: per-protocol-type dispatch table via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hooks: `security_xfrm_policy_alloc`, `security_xfrm_policy_clone`, `security_xfrm_policy_free`, `security_xfrm_policy_delete`, `security_xfrm_state_alloc`, `security_xfrm_state_alloc_acquire`, `security_xfrm_state_delete`, `security_xfrm_state_free`, `security_xfrm_policy_lookup`, `security_xfrm_state_pol_flow_match`, `security_xfrm_decode_session` — all standard upstream.
- GR-RBAC policy can deny per-subject SA/policy mutation. Default empty.
- Useful default GR-RBAC policy fragment: deny NETLINK_XFRM + PF_KEY mutation outside gradm-marked `ipsec_admin` role; SA mutation gives kernel-level encrypt/decrypt-key control.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none at Tier-2 — defer to per-Tier-3)

## Out of Scope

- 32-bit userspace compat (`xfrm_compat.c`) — covered in `net/xfrm/compat.md` Tier-3, marked OUT-OF-SCOPE for v0
- Per-IKE-daemon userspace logic — kernel exposes the wire ABI, daemon is userspace
- Implementation code
