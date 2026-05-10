# Tier-2: net/tls — Kernel TLS (kTLS sw + device offload + strparser + TOE + TLS 1.2/1.3)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/tls/
  - include/net/tls.h
  - include/uapi/linux/tls.h
-->

## Summary

Tier-2 wrapper for kernel TLS — `setsockopt(SOL_TLS, TLS_TX, ...)` + `TLS_RX` move TLS record-layer encryption/decryption into the kernel after userspace OpenSSL/etc. completed the handshake. Enables: zero-copy `sendfile(2)` to TLS sockets (web servers nginx + apache + haproxy + envoy serving TLS-static-content from disk without per-record CPU encryption), AF_KTLS-style packet capture from kernel, NIC offload of TLS encrypt/decrypt (mlx5 / nfp / bnxt). Supports TLS 1.2 + TLS 1.3 with AES-128-GCM / AES-256-GCM / ChaCha20-Poly1305 / AES-128-CCM / AES-256-CCM / SM4-GCM / SM4-CCM / ARIA-128-GCM / ARIA-256-GCM ciphers.

Components: `tls_main.c` (subsystem init + setsockopt(SOL_TLS) handlers), `tls_sw.c` (software TLS encrypt/decrypt path using crypto API), `tls_device.c` + `tls_device_fallback.c` (device-offloaded TLS path; per-NIC tx_offload_ctx + rx_offload_ctx; software-fallback when NIC indicates can't offload a record), `tls_toe.c` (TLS Offload Engine for full-stack offload), `tls_strp.c` (TLS strparser — collects skb chunks into TLS-record boundaries), `tls_proc.c` (`/proc/net/tls_stat`), `trace.c` + `trace.h` (tracepoints `tls:*`), `tls.h` (private header).

## Compatibility contract — outline

- `setsockopt(SOL_TLS, TLS_TX, ...)` + `TLS_RX` UAPI byte-identical: `struct tls_crypto_info_aes_gcm_128`, `_256`, `tls12_crypto_info_chacha20_poly1305`, `_aes_ccm_128`, `_aes_ccm_256`, `_sm4_*`, `_aria_*`. Wire format byte-identical so OpenSSL kTLS + mbedTLS-kTLS + GnuTLS-kTLS + nginx-kTLS + envoy-kTLS consume unchanged.
- `getsockopt(SOL_TLS, TLS_TX/RX)` returns config back identically.
- TLS_RX_EXPECT_NO_PAD socket option byte-identical.
- ULP layer (Upper Layer Protocol) `setsockopt(SOL_TCP, TCP_ULP, "tls", ...)` byte-identical.
- `/proc/net/tls_stat` byte-identical fields.
- `events/tls/` tracepoints byte-identical.
- NIC offload negotiation via NETIF_F_HW_TLS_TX/RX/RECORD feature flags byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `net/tls/main.md` | `tls_main.c`: subsystem init + setsockopt + ULP registration |
| `net/tls/sw.md` | `tls_sw.c`: software TLS encrypt/decrypt via crypto API |
| `net/tls/device.md` | `tls_device.c` + `tls_device_fallback.c`: NIC-offloaded TLS path + sw fallback |
| `net/tls/toe.md` | `tls_toe.c`: TOE full-stack offload |
| `net/tls/strp.md` | `tls_strp.c`: strparser — TLS record-boundary detection |
| `net/tls/proc.md` | `tls_proc.c`: `/proc/net/tls_stat` |

## Compatibility outline

- REQ-O1: setsockopt(SOL_TLS, TLS_TX/RX) UAPI byte-identical (OpenSSL/mbedTLS/GnuTLS/nginx kTLS consumers work unchanged).
- REQ-O2: TCP_ULP "tls" registration byte-identical.
- REQ-O3: All TLS 1.2 + TLS 1.3 cipher configs supported.
- REQ-O4: NIC offload negotiation via netif_features byte-identical.
- REQ-O5: `/proc/net/tls_stat` fields byte-identical.
- REQ-O6: TLA+ models (TLS record-boundary strparser; TX-rekey while in-flight; RX device offload + sw fallback handoff; per-record AEAD nonce monotonic).
- REQ-O7: Hardening: keys never copied back to userspace post-install; per-record AEAD nonce monotonic enforcement; ChaCha20-Poly1305 default for software path.

## Acceptance Criteria

- [ ] AC-O1: `nginx` with `ssl_conf_command Options KTLS` serves https zero-copy via sendfile(2).
- [ ] AC-O2: OpenSSL `s_client -connect` + `BIO_set_ktls(...)` works.
- [ ] AC-O3: TLS-RX offload test on mlx5: tcpdump shows decrypted plaintext on socket reads.
- [ ] AC-O4: kselftest `tools/testing/selftests/net/tls/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/tls/strparser.tla` | `net/tls/strp.md` (proves: TLS record-boundary strparser collects skb chunks into complete records; concurrent recv + parse never produces torn record or missed record) |
| `models/tls/aead_nonce.tla` | `net/tls/sw.md` (proves: per-record AEAD nonce monotonically increases per-tx-key; rekey replaces nonce-counter atomically; never reuse nonce within key) |
| `models/tls/device_fallback.tla` | `net/tls/device.md` (proves: per-record device-offload OR software-fallback; NIC indicates "can't offload" per-skb correctly handed off to sw path; record ordering preserved) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-tls_context refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-cipher `tls_offload_resync_*` ops `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-record length + AEAD-tag arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed tls_context cleared (carries TLS keys) | § Default-on configurable |

TLS-specific reinforcement: keys passed to setsockopt copied + zeroed in user-buffer immediately (defense against keys lingering in user-RAM); AEAD nonce monotonicity enforced at tls_sw + device-offload layer (defense against nonce-reuse breaking AEAD security); rekey serialized via per-context lock; NIC offload requires CAP_NET_ADMIN per-NIC enable + LSM hook `security_socket_setsockopt`.

## Open Questions
(none at Tier-2)

## Out of Scope
- Implementation code
- TLS handshake (lives in userspace OpenSSL/mbedTLS/GnuTLS)
- 32-bit-only paths
