# Tier-3: drivers/net/wireguard/{device,send,receive,noise,...}.c — WireGuard VPN driver

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - drivers/net/wireguard/main.c (~78 lines)
  - drivers/net/wireguard/device.c (~475 lines)
  - drivers/net/wireguard/send.c
  - drivers/net/wireguard/receive.c
  - drivers/net/wireguard/socket.c
  - drivers/net/wireguard/noise.c (handshake)
  - drivers/net/wireguard/cookie.c (DDoS mitigation)
  - drivers/net/wireguard/allowedips.c (per-peer route table)
  - drivers/net/wireguard/timers.c
  - drivers/net/wireguard/queueing.c
  - include/uapi/linux/wireguard.h (genetlink ABI)
-->

## Summary

WireGuard is a modern minimalist L3 VPN with: per-peer (X25519, public-key) authenticated encryption (ChaCha20-Poly1305 AEAD); Noise IK handshake (lightweight, no negotiation); per-peer `allowedips` route-table (CIDR → peer); per-skb tx encrypts to per-peer endpoint UDP-encap; per-skb rx authenticates and decrypts; per-peer keepalive + rekey. Per-(peer, src-IP) cookie-mac DDoS mitigation. Per-net-namespace per-wireguard-iface. Critical for: secure cross-cloud VPN, mesh networking, ZeroTrust overlays.

This Tier-3 covers `device.c` (~475 lines) + per-related files.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct wg_device` | per-wg netdev | `WgDevice` |
| `struct wg_peer` | per-peer state | `WgPeer` |
| `struct wg_socket` | per-AF UDP-encap socket | `WgSocket` |
| `struct allowedips` | per-(peer) CIDR → peer route-table | `Allowedips` |
| `struct noise_handshake` | per-peer handshake state | `NoiseHandshake` |
| `struct noise_keypair` | per-peer encrypt/decrypt keys | `NoiseKeypair` |
| `wg_xmit()` | per-skb TX (encap-encrypt) | `Wg::xmit` |
| `wg_packet_send_keepalive()` | per-peer keepalive | `Wg::send_keepalive` |
| `wg_packet_send_handshake_initiation()` | per-peer init | `Wg::send_handshake_initiation` |
| `wg_packet_consume_data()` | per-skb RX (decrypt-decap) | `Wg::packet_consume_data` |
| `wg_socket_init()` | per-net UDP encap socket | `WgSocket::init` |
| `wg_set_device()` / `wg_get_device()` | per-genetlink set/get | `Wg::set_device` / `get_device` |
| `wg_genl_family` | per-genetlink family | `Wg::genl_family` |
| `WG_CMD_*` | per-genl-cmd | UAPI |
| `WGDEVICE_A_*` / `WGPEER_A_*` / `WGALLOWEDIP_A_*` | per-attr | UAPI |

## Compatibility contract

REQ-1: Per-wg_device:
- name: e.g. "wg0".
- private_key: per-instance X25519 priv-key.
- public_key: derived.
- listen_port: UDP port.
- fwmark: per-skb-mark for routing.
- peer_list: per-wg list of peers.

REQ-2: Per-wg_peer:
- handshake: NoiseHandshake (initiator/responder state, latest send-time).
- keypairs: NoiseKeypairs (current + previous + next, for re-key transition).
- endpoint: peer last-known UDP IP+port (updated per RX).
- allowedips: per-peer CIDR ranges.
- persistent_keepalive_interval: u16 (0 = off).
- timers: rekey-timer, persistent-keepalive-timer, handshake-retry-timer.

REQ-3: Per-genetlink ABI:
- WG_CMD_GET_DEVICE: dump per-iface state.
- WG_CMD_SET_DEVICE: install peers, allowedips, keys.
- struct wgdevice (NLA_NESTED) per-iface.
- struct wgpeer (NLA_NESTED) per-peer.
- struct wgallowedip (NLA_NESTED) per-CIDR.

REQ-4: Per-skb TX (wg_xmit):
- skb is L3 packet (no eth hdr per pure-L3 tunnel).
- peer = allowedips_lookup_dst(skb).
- if !peer: drop.
- if !peer.handshake.healthy: trigger handshake.
- encrypt skb with peer.keypair.tx_key.
- udp_tunnel_xmit_skb to peer.endpoint.

REQ-5: Per-skb RX (per UDP encap):
- Type byte (1) + per-message:
  - HANDSHAKE_INITIATION (1): trigger response.
  - HANDSHAKE_RESPONSE (2): consume.
  - HANDSHAKE_COOKIE (3): mitigation.
  - DATA (4): decrypt + deliver.
- Per-decrypt: validate authtag; replay-protection per-counter.
- skb rebuilt as L3 inner packet; netif_rx.

REQ-6: Per-handshake (Noise IK):
- Initiator: WG_HANDSHAKE_INITIATION (148 bytes).
- Responder: WG_HANDSHAKE_RESPONSE (92 bytes).
- Per-handshake derives session-keys.

REQ-7: Per-cookie:
- Per-receiver UDP-flood: requires pre-shared cookie.
- WG_HANDSHAKE_COOKIE (64 bytes): tag for handshake-flood mitigation.

REQ-8: Per-allowedips (cidr-trie):
- Patricia-trie per-AF (v4/v6).
- Per-(CIDR, peer) lookup.
- Per-shorter-prefix-fallback for not-most-specific match.

REQ-9: Per-rekey:
- Re-handshake every 2 minutes (REJECT_AFTER_TIME).
- Or after 2^60 messages (REJECT_AFTER_MESSAGES).
- Per-keypair lifetime: ~3 minutes max.

REQ-10: Per-namespace:
- Per-net wg_socket; per-net wg_devices.

REQ-11: Per-peer keepalive:
- persistent_keepalive_interval: send empty WG_DATA every N seconds.

REQ-12: Per-MTU:
- IPv4 default 1420; IPv6 default 1400.

## Acceptance Criteria

- [ ] AC-1: `ip link add wg0 type wireguard`: wg netdev created.
- [ ] AC-2: `wg set wg0 private-key`: WG_CMD_SET_DEVICE installs key.
- [ ] AC-3: `wg set wg0 peer ... allowed-ips 10.0.0.0/24 endpoint 1.2.3.4:5678`: peer + allowedips installed.
- [ ] AC-4: TX skb with dst 10.0.0.5: encrypted + sent to 1.2.3.4:5678.
- [ ] AC-5: RX UDP packet on listen-port: decrypted + delivered as inner.
- [ ] AC-6: Handshake: WG_HANDSHAKE_INITIATION → WG_HANDSHAKE_RESPONSE within RTT.
- [ ] AC-7: Per-rekey: every 2 minutes or 2^60 packets.
- [ ] AC-8: Cookie response under DDoS: subsequent handshakes carry cookie.
- [ ] AC-9: persistent_keepalive_interval=25: WG_DATA empty every 25s.
- [ ] AC-10: ndo_start_xmit + xfrm-stack disabled: WG handles encap.

## Architecture

Per-wg_device:

```
struct WgDevice {
  dev: *NetDev,
  static_identity: NoiseStaticIdentity,           // priv + pub
  preshared_keys: [u8; 32],
  listen_port: __be16,
  socket4: *WgSocket,
  socket6: *WgSocket,
  peer_allowedips: AllowedIps,
  peer_list: ListHead<WgPeer>,
  num_peers: AtomicI32,
  cookie_checker: WgCookieChecker,
  packet_crypt_wq: *Workqueue,
  handshake_send_wq: *Workqueue,
  ...
}

struct WgPeer {
  device: *WgDevice,
  endpoint: WgEndpoint,                           // last-known IP+port
  endpoint_lock: SpinLock,
  handshake: NoiseHandshake,
  keypairs: NoiseKeypairs,                        // current/prev/next
  is_dead: bool,
  persistent_keepalive_interval: u16,
  timer_rekey: TimerList,
  timer_persistent_keepalive: TimerList,
  ...
}
```

Per-allowedips (Patricia-trie):

```
struct AllowedipsNode {
  peer: Option<*WgPeer>,
  bit: BitMaskBlob,                              // CIDR bits
  cidr: u8,                                      // prefix length
  bit_at_a: u8,                                  // tree-walking
  bit_at_b: u8,
  parent: *AllowedipsNode,
  bit_l: *AllowedipsNode,
  bit_r: *AllowedipsNode,
}

struct AllowedIps {
  root4: *AllowedipsNode,
  root6: *AllowedipsNode,
}
```

`Wg::xmit(skb, dev) -> NetdevTxT`:
1. wg = wg_priv(dev).
2. /* L3 dst lookup via allowedips */
3. peer = allowedips_lookup_dst(skb, &wg.peer_allowedips).
4. if !peer: drop.
5. if !peer.keypairs.current_keypair ∨ peer.keypairs.expired:
   - wg_packet_send_handshake_initiation(peer).
   - drop or queue (depends on policy).
6. /* Encrypt */
7. wg_packet_send_data(peer, skb).

`Wg::packet_send_data(peer, skb)`:
1. /* AEAD encrypt */
2. nonce = peer.keypairs.tx_counter++.
3. cipher_skb = chacha20_poly1305_encrypt(skb, key=peer.keypairs.tx_key, nonce, aad).
4. cipher_skb prepended with type byte + nonce + receiver-index.
5. udp_tunnel_xmit_skb(rt, wg.socket, cipher_skb, ..., peer.endpoint.dst_port).

`Wg::packet_consume_data(skb, peer)`:
1. /* Validate decrypt */
2. nonce = msg.nonce; receiver_idx = msg.receiver_index.
3. keypair = lookup_keypair(receiver_idx).
4. if !keypair: drop.
5. plain_skb = chacha20_poly1305_decrypt(skb, key=keypair.rx_key, nonce, aad).
6. if !plain_skb: drop (auth-fail).
7. /* Replay protection */
8. if !replay_check(keypair.replay_window, nonce): drop.
9. /* Update peer.endpoint from src */
10. update_peer_endpoint(peer, src_ip, src_port).
11. skb = plain_skb; skb.dev = wg.dev.
12. netif_rx(skb).

`Wg::send_handshake_initiation(peer)`:
1. /* Construct Noise IK init message */
2. msg = build_noise_initiation(peer, wg.static_identity).
3. udp_tunnel_xmit_skb to peer.endpoint.
4. timer_rekey arm.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `private_key_secret` | INVARIANT | wg.static_identity.priv_key never copied to non-priv buffer. |
| `nonce_monotonic` | INVARIANT | per-peer.keypairs.tx_counter monotonically increasing. |
| `replay_window_anti_replay` | INVARIANT | per-rx: nonce > last-window-floor or in window. |
| `peer_in_one_list` | INVARIANT | per-WgPeer in exactly one wg.peer_list. |
| `keypair_lifetime_le_180s` | INVARIANT | per-keypair: max-lifetime ≤ 180s. |

### Layer 2: TLA+

`drivers/net/wireguard/wireguard.tla`:
- Per-peer Noise IK handshake + per-keypair lifetime + per-msg encrypt/decrypt.
- Properties:
  - `safety_no_replay` — per-(receiver_idx, nonce) seen at most once.
  - `safety_handshake_authenticated` — per-keypair installed ⟹ both peers have matching pub-keys.
  - `safety_endpoint_updated_only_post_decrypt` — per-endpoint update only after auth-success.
  - `liveness_handshake_eventually_succeeds` — per-peer + reachable + correct-keys ⟹ handshake completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Wg::xmit` post: skb encrypted + udp_tunnel_xmit_skb invoked OR handshake triggered | `Wg::xmit` |
| `Wg::packet_consume_data` post: decrypted + replay-checked + endpoint-updated + delivered | `Wg::packet_consume_data` |
| `Wg::send_handshake_initiation` post: WG_HANDSHAKE_INITIATION emitted to peer.endpoint | `Wg::send_handshake_initiation` |
| `Allowedips::lookup_dst` post: returns peer with longest-prefix-match for skb.dst | `Allowedips::lookup_dst` |

### Layer 4: Verus/Creusot functional

`Per-skb encrypted via ChaCha20-Poly1305 AEAD; per-peer Noise IK handshake; per-allowedips longest-prefix-match routing` semantic equivalence: per-WireGuard whitepaper (2018 NDSS).

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

WireGuard-specific reinforcement:

- **Per-priv-key never logged** — defense against kernel-log private-key leak.
- **Per-AEAD authtag validated pre-deliver** — defense against per-tampered-pkt.
- **Per-replay-window 2^N tracking** — defense against per-replay attack.
- **Per-cookie response under flood** — defense against per-handshake DDoS.
- **Per-peer ratelimit on handshake** — defense against per-peer flood.
- **Per-allowedips longest-prefix unique** — defense against per-CIDR ambiguity.
- **Per-keypair-rotation 2-minute** — defense against per-key-fatigue.
- **Per-nonce reset on rekey** — defense against per-stale-nonce reuse.
- **Per-endpoint-update only after auth** — defense against per-spoofed src updating endpoint.
- **Per-namespace WG socket scoped** — defense against cross-ns leakage.
- **Per-CAP_NET_ADMIN for genl WG_CMD_SET** — defense against unprivileged WG-config.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/net/wireguard/{noise, cookie, allowedips, queueing, timers}.c (covered separately if expanded)
- ChaCha20-Poly1305 (lib/crypto; covered separately)
- Curve25519 (lib/crypto; covered separately)
- udp_tunnel framework (covered separately)
- Implementation code
