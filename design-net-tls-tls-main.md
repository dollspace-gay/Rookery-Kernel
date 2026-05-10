---
title: "Tier-3: net/tls/tls_main.c — Kernel TLS (KTLS) socket-layer offload"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KTLS (Kernel TLS) is in-kernel TLS-record framing + encryption: userspace performs handshake (OpenSSL/wolfSSL/etc.); per-`setsockopt(TLS_TX/TLS_RX, crypto_info)` installs per-direction key + IV + seq; kernel then encrypts/decrypts per-record on send/recv. Per-mode: TLS_SW (software in kernel via AES-NI/ChaCha20-Poly1305), TLS_HW (NIC-offloaded). Per-version: TLS_1_2 / TLS_1_3. Per-cipher: AES_GCM_128, AES_GCM_256, AES_CCM_128, CHACHA20_POLY1305. Per-record: per-record-seq monotonic. Per-RX-strp tls_strp uses skb-strparser to frame TLS-records on TCP-stream. Critical for: server-side sendfile-TLS (NGINX kTLS), userspace-bypass for TLS-heavy workloads, HTTPS at line-rate.

This Tier-3 covers `tls_main.c` (~1274 lines).

### Acceptance Criteria

- [ ] AC-1: setsockopt(SOL_TCP, TCP_ULP, "tls", 4): socket switches to TLS ULP.
- [ ] AC-2: setsockopt(SOL_TLS, TLS_TX, &aes_gcm_128_info): TX offload activated.
- [ ] AC-3: TX userspace data → encrypted TLS-records on wire.
- [ ] AC-4: setsockopt(SOL_TLS, TLS_RX, &info): RX offload.
- [ ] AC-5: Incoming TLS-records: decrypted before recvmsg.
- [ ] AC-6: Auth-fail TLS-record: socket reports SO_ERROR.
- [ ] AC-7: sendfile on TLS-sock: zerocopy encrypt.
- [ ] AC-8: TLS_1_3 cipher: per-record AEAD with seq-derived nonce.
- [ ] AC-9: HW-offload (TLS_HW): NIC handles encryption.
- [ ] AC-10: NGINX kTLS sendfile: serves HTTPS at line-rate.

### Architecture

Per-context:

```
struct TlsContext {
  sk_proto: *Proto,                              // base TCP proto
  push_pending_record: fn(sk, flags),
  sk: *Sock,
  sk_destruct: fn(sk),
  conf: TlsProtInfo,                              // per-direction crypto-info
  prot_info: TlsProtInfo,
  cipher_type: u16,
  pending_open_record_frags: bool,
  conn_disp: ConnDisp,
  recv_attribs: RecvAttribs,
  tx: TlsSwContextTx,
  rx: TlsSwContextRx,
  netdev: *NetDev,                                // for HW-offload
  ulp_data: u8[TLS_ULP_DATA_SIZE],
  hash: *CryptoShash,
  aead_send: *CryptoAead,
  aead_recv: *CryptoAead,
}

struct TlsCryptoInfo {
  version: u16,                                  // TLS_1_2_VERSION / TLS_1_3_VERSION
  cipher_type: u16,                              // TLS_CIPHER_*
}

struct Tls12CryptoInfoAesGcm128 {
  info: TlsCryptoInfo,
  iv: [u8; 8],
  key: [u8; 16],
  salt: [u8; 4],
  rec_seq: [u8; 8],
}
```

`Tls::set_sw_offload(sk, ctx, tx) -> Result<()>`:
1. crypto_info = (TlsCryptoInfo*)ctx.crypto.
2. /* Parse per-cipher info */
3. err = crypto_aead_setkey(ctx.aead_send, &cipher_info.key, key_size).
4. err = crypto_aead_setauthsize(ctx.aead_send, TLS_TAG_SIZE).
5. /* Allocate per-direction state */
6. ctx.tx = alloc TlsSwContextTx; ctx.tx.iv = cipher_info.iv; ctx.tx.salt = cipher_info.salt; ctx.tx.rec_seq = cipher_info.rec_seq.
7. /* Hook tls_sw_sendmsg */
8. sk.sk_write_space = tls_write_space.

`TlsSw::sendmsg(sk, msg, size) -> Result<usize>`:
1. ctx = tls_get_ctx(sk).
2. /* Build TLS record: header + payload, encrypted */
3. ret = tls_push_data(sk, msg, size, 0, MSG_*).
4. if ret == -ENOMEM: -ENOMEM.
5. return size-actually-sent.

`TlsSw::push_data(sk, msg, size, flags)`:
1. for each chunk:
   - Build TLS record header { type=APPLICATION_DATA, version, length }.
   - Build AAD (additional auth data).
   - nonce = ctx.tx.iv ++ ctx.tx.rec_seq.
   - ciphertext = AEAD_encrypt(aead_send, plaintext, AAD, nonce, key).
   - tcp_sendmsg ciphertext.
   - ctx.tx.rec_seq++.

`TlsStrp::msg(strp, skb)`:
1. /* Per-TCP-stream skb arrives; check if full TLS-record present */
2. hdr = parse_tls_record_header(skb).
3. if !skb-has hdr.length bytes: defer; return.
4. /* Full record */
5. err = decrypt_skb(skb, ctx.rx).
6. if err: tls_err_abort.
7. /* Plain skb queued for recvmsg */

`TlsSw::recvmsg(sk, msg, size, ...)`:
1. /* Get plaintext skb from ctx.rx.list */
2. /* Copy to msg.iov */
3. Update bytes-received.

`Tls::setsockopt(sk, level, optname, optval, optlen) -> Result<()>`:
1. ctx = tls_get_ctx(sk).
2. switch optname:
   - TLS_TX: copy_from_user(crypto_info); Tls::set_sw_offload(sk, ctx, TX).
   - TLS_RX: copy_from_user(crypto_info); Tls::set_sw_offload(sk, ctx, RX).
3. validate cipher_type, version.

### Out of Scope

- net/tls/{tls_sw, tls_device, tls_strp, tls_proc}.c (covered separately if expanded)
- crypto AEAD subsystem (covered in `crypto/00-overview.md` separately)
- OpenSSL / wolfSSL userspace handshake (out-of-tree)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tls_context` | per-socket state | `TlsContext` |
| `struct tls_sw_context_tx` / `_rx` | per-direction SW state | `TlsSwContextTx` / `Rx` |
| `tls_init()` | per-net init | `Tls::init` |
| `tls_setsockopt()` / `tls_getsockopt()` | per-TLS_* sockopt | `Tls::setsockopt` / `getsockopt` |
| `tls_set_sw_offload()` | per-SW mode setup | `Tls::set_sw_offload` |
| `tls_set_device_offload()` | per-HW mode setup | `Tls::set_device_offload` |
| `tls_sw_sendmsg()` / `tls_sw_recvmsg()` | per-SW send/recv | `TlsSw::sendmsg` / `recvmsg` |
| `tls_sw_close()` | per-SW close | `TlsSw::close` |
| `tls_strp_init()` | per-RX-strp init | `TlsStrp::init` |
| `tls_strp_msg()` | per-rx-decode | `TlsStrp::msg` |
| `tls_setup_from_iter()` | per-msg encrypt | `Tls::setup_from_iter` |
| `decrypt_skb()` | per-recv decrypt | `Tls::decrypt_skb` |
| `tls_ctx_create()` | per-socket create | `TlsContext::create` |
| `TLS_TX` / `TLS_RX` | sockopts | UAPI |
| `TLS_VERSION_*` / `TLS_CIPHER_*` | per-config | UAPI |

### compatibility contract

REQ-1: setsockopt(SOL_TCP, TCP_ULP, "tls", 4):
- Switches socket to TLS ULP (upper-layer protocol).
- Per-tcp_sock proto = &tls_prots[tls_arr_idx].

REQ-2: setsockopt(SOL_TLS, TLS_TX, &crypto_info):
- crypto_info: struct tls_crypto_info { version, cipher_type } + per-cipher subfields (key, iv, salt, rec_seq).
- Per-tls-cipher size:
  - tls12_crypto_info_aes_gcm_128: 36 bytes.
  - tls12_crypto_info_aes_gcm_256: 52 bytes.
  - tls12_crypto_info_chacha20_poly1305: 52 bytes.

REQ-3: setsockopt(SOL_TLS, TLS_RX, &crypto_info):
- Symmetric for RX direction.

REQ-4: TLS_HW_RECORD / TLS_HW: HW-offload modes.
- TLS_HW_RECORD: record-only offload.
- TLS_HW: full offload (entire conn).

REQ-5: Per-tx-flow:
- userspace sendmsg/send → tls_sw_sendmsg.
- buffer userspace data; build TLS-record header (type, version, length).
- AEAD-encrypt per-record (key, nonce=iv++rec_seq, AAD=type+version+length).
- tcp_sendmsg ciphertext.

REQ-6: Per-rx-flow:
- TCP-stream-data arrives → tls_strp framing.
- per-TLS-record-complete: decrypt_skb (key, nonce=iv++rec_seq).
- if auth-fail: bad-mac → close.
- expose plaintext via tls_sw_recvmsg.

REQ-7: Per-sendfile-TLS:
- sendfile(socket, file_fd, offset, len) on TLS-sock: kernel zerocopy + encrypt-in-place.

REQ-8: Per-record-version:
- TLS_1_2: AEAD with explicit-IV; per-record-seq in plaintext.
- TLS_1_3: AEAD with implicit-IV from seq; no explicit-IV in record.

REQ-9: Per-rekey:
- userspace renegotiate: setsockopt new crypto_info.
- Per-direction independently rekey.

REQ-10: Per-namespace:
- Per-net TLS instance.

REQ-11: Per-ULP type:
- /sys/kernel/debug/tcp_ulp_strings exposes "tls", "espintcp", etc.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rec_seq_monotonic` | INVARIANT | per-tx rec_seq strictly-increasing. |
| `cipher_type_supported` | INVARIANT | crypto_info.cipher_type ∈ {AES_GCM_128, AES_GCM_256, AES_CCM_128, CHACHA20_POLY1305}. |
| `version_supported` | INVARIANT | version ∈ {TLS_1_2, TLS_1_3}. |
| `tls_record_len_le_max` | INVARIANT | per-record-len ≤ TLS_MAX_PAYLOAD_SIZE. |
| `ctx_lifetime` | INVARIANT | tls_ctx exists ⟺ TCP socket has TLS ULP. |

### Layer 2: TLA+

`net/tls/tls.tla`:
- Per-socket setsockopt + per-direction encrypt/decrypt + per-record-seq.
- Properties:
  - `safety_per_record_seq_incremented` — per-tls_sw_sendmsg: rec_seq incremented.
  - `safety_no_decrypt_without_key` — per-recv: AEAD-decrypt only after key-installed.
  - `liveness_per_record_eventually_complete` — per-tls_strp partial-record: eventually full → decrypt.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tls::set_sw_offload` post: ctx.aead_send/recv set; rec_seq initialized | `Tls::set_sw_offload` |
| `TlsSw::sendmsg` post: per-chunk encrypted; tcp_sendmsg-ed; rec_seq++ | `TlsSw::sendmsg` |
| `TlsStrp::msg` post: per-record decrypted-and-queued OR abort-on-fail | `TlsStrp::msg` |
| `Tls::setsockopt` post: per-direction key set; crypto initialized | `Tls::setsockopt` |

### Layer 4: Verus/Creusot functional

`Per-userspace handshake completed → setsockopt crypto-info → kernel encrypts/decrypts per-record on tcp-stream` semantic equivalence: per-RFC 5246 (TLS 1.2) + RFC 8446 (TLS 1.3) record layer.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

KTLS reinforcement:

- **Per-key never logged** — defense against per-log secret-leak.
- **Per-record auth-fail closes socket** — defense against per-padding-oracle.
- **Per-rec_seq u64 monotonic** — defense against per-seq-replay.
- **Per-AAD includes type+version+length** — defense against per-cross-record confusion.
- **Per-version TLS_1_3 implicit IV** — defense against per-IV reuse.
- **Per-record-len cap (~16KB)** — defense against per-malicious-record allocation.
- **Per-direction key separation** — defense against per-key-mix.
- **Per-rekey serializes via socket-lock** — defense against per-mid-flight rekey race.
- **Per-NIC HW-offload caps gated** — defense against per-driver insufficient cap.
- **Per-namespace tls-ULP scoped** — defense against cross-ns leak.
- **Per-CAP_NET_ADMIN for ULP TCP_ULP set** — defense against unprivileged ULP-install.

