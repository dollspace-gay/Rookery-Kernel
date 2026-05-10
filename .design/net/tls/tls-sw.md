# Tier-3: net/tls/tls_sw.c — software kTLS data path (encrypt/decrypt, sendmsg/recvmsg/splice, strparser, BPF sk_msg)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/tls/00-overview.md
upstream-paths:
  - net/tls/tls_sw.c (~2923 lines)
  - net/tls/tls.h (tls_sw_context_tx/rx, tls_rec, tls_msg, tls_prot_info)
  - net/tls/tls_strp.c (struct tls_strparser stream parser)
  - include/net/tls.h (tls_context, tls_crypto_info, tls_cipher_desc)
  - include/uapi/linux/tls.h (TLS_TX/TLS_RX setsockopt, TLS_*_VERSION, TLS_CIPHER_*)
  - include/crypto/aead.h (aead_request, crypto_aead, crypto_aead_encrypt/decrypt)
  - include/linux/skmsg.h (struct sk_msg, sk_msg_alloc/clone/zerocopy_from_iter)
-->

## Summary

**Software kTLS** (kernel TLS) plugs into a TCP socket as a ULP ("upper-layer protocol") and performs **TLS-record encrypt on send** and **TLS-record decrypt on receive** entirely in-kernel using the crypto-API AEAD interface — without hardware offload. Per-TX (`tls_sw_sendmsg` / `tls_sw_sendmsg_locked`): per-application-write the data is gathered into an open `struct tls_rec` (paired `msg_plaintext` + `msg_encrypted` `sk_msg` scatterlists), AAD is built per `tls_make_aad`, the TLS header is filled via `tls_fill_prepend`, the AEAD (AES-GCM-128/256, AES-CCM-128, CHACHA20-POLY1305, SM4-GCM/CCM) is invoked synchronously or asynchronously through `tls_do_encryption`, and completed records are pushed to TCP via `tls_tx_records` calling `tcp_sendmsg_locked`. Per-RX: the **TLS strparser** (`tls_strp`, `tls_rx_msg_size`) frames TCP byte stream into TLS records; `tls_sw_recvmsg` dequeues a record via `tls_rx_one_record` → `tls_decrypt_sw` → `tls_do_decryption` → `crypto_aead_decrypt` (zero-copy into iov when `ZC_RX` capable), strips padding/content-type per `tls_padding_length`, advances the record sequence number, and delivers plaintext to userspace. Per-`splice` paths: `tls_sw_splice_eof` (handle write-side EOF) and `tls_sw_splice_read` (pipe-out plaintext after decrypt). Per-async crypto: AEAD callbacks (`tls_encrypt_done`, `tls_decrypt_done`) wake `crypto_wait_t` waiters and tally `encrypt_pending` / `decrypt_pending`. Per-BPF integration: `bpf_exec_tx_verdict` runs an attached `sk_msg` program over the plaintext sk_msg before encryption (verdict can DROP / REDIRECT / PASS); `sk_psock_tls_strp_read` runs strparser-mode programs on plaintext skbs before recv delivery. Per-content-type: `TLS_RECORD_TYPE_DATA` (23), `TLS_RECORD_TYPE_HANDSHAKE` (22 — used for in-band TLS 1.3 KeyUpdate), control records are surfaced via `TLS_GET_RECORD_TYPE` cmsg. Critical for: kernel-fast-path TLS, sendfile/splice over TLS, BPF policy enforcement at TLS-record boundary, hardware-free TLS rollover.

This Tier-3 covers `net/tls/tls_sw.c` (~2923 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tls_decrypt_arg` | per-decrypt zc/async/tail flags + skb | `TlsDecryptArg` |
| `struct tls_decrypt_ctx` | per-decrypt IV/AAD/sg scratch | `TlsDecryptCtx` |
| `tls_err_abort()` | per-fatal-crypto sk error report | `TlsSw::err_abort` |
| `tls_sw_sendmsg()` | per-send entry (locks + sendmsg_locked) | `TlsSw::sendmsg` |
| `tls_sw_sendmsg_locked()` | per-send loop (gather, encrypt, push) | `TlsSw::sendmsg_locked` |
| `tls_sw_sendmsg_splice()` | per-MSG_SPLICE_PAGES gather | `TlsSw::sendmsg_splice` |
| `tls_sw_recvmsg()` | per-recv entry (decrypt loop + zc) | `TlsSw::recvmsg` |
| `tls_sw_splice_read()` | per-splice(2) read into pipe | `TlsSw::splice_read` |
| `tls_sw_splice_eof()` | per-splice-EOF flush | `TlsSw::splice_eof` |
| `tls_sw_read_sock()` | per-tcp_read_sock for BPF | `TlsSw::read_sock` |
| `tls_sw_push_pending_record()` | per-flush open record | `TlsSw::push_pending_record` |
| `tls_sw_sock_is_readable()` | per-poll readability | `TlsSw::sock_is_readable` |
| `tls_get_rec()` / `tls_free_rec()` / `tls_free_open_rec()` | per-record alloc/free | `TlsSw::get_rec` / `free_rec` |
| `tls_tx_records()` | per-tx_list flush to TCP | `TlsSw::tx_records` |
| `tls_push_record()` | per-record encode + encrypt + push | `TlsSw::push_record` |
| `tls_split_open_record()` / `tls_merge_open_record()` | per-record split/merge for full | `TlsSw::split_open_record` / `merge_open_record` |
| `tls_do_encryption()` | per-AEAD encrypt invocation | `TlsSw::do_encryption` |
| `tls_encrypt_done()` | per-AEAD encrypt callback | `TlsSw::encrypt_done` |
| `tls_encrypt_async_wait()` | per-tx-pending wait | `TlsSw::encrypt_async_wait` |
| `tls_do_decryption()` | per-AEAD decrypt invocation | `TlsSw::do_decryption` |
| `tls_decrypt_done()` | per-AEAD decrypt callback | `TlsSw::decrypt_done` |
| `tls_decrypt_async_wait()` | per-rx-pending wait | `TlsSw::decrypt_async_wait` |
| `tls_decrypt_sg()` | per-decrypt scatterlist setup | `TlsSw::decrypt_sg` |
| `tls_decrypt_sw()` / `tls_decrypt_device()` | per-SW vs device decrypt | `TlsSw::decrypt_sw` / `decrypt_device` |
| `tls_rx_one_record()` | per-record decrypt + sn advance | `TlsSw::rx_one_record` |
| `decrypt_skb()` | per-external sg decrypt entry | `TlsSw::decrypt_skb` |
| `process_rx_list()` | per-drain previously decrypted | `TlsSw::process_rx_list` |
| `tls_padding_length()` | per-TLS1.3 trailing-zero strip + content-type recovery | `TlsSw::padding_length` |
| `tls_check_pending_rekey()` | per-1.3 KeyUpdate / rekey notification | `TlsSw::check_pending_rekey` |
| `tls_record_content_type()` | per-recv content_type mux + cmsg | `TlsSw::record_content_type` |
| `tls_rx_msg_size()` | per-strparser TLS-record framing | `TlsSw::rx_msg_size` |
| `tls_rx_msg_ready()` | per-strparser data-ready bump | `TlsSw::rx_msg_ready` |
| `tls_data_ready()` | per-sk_data_ready override | `TlsSw::data_ready` |
| `tls_rx_reader_acquire()` / `_lock()` / `_release()` / `_unlock()` | per-reader exclusion | `TlsSw::rx_reader_*` |
| `tls_setup_from_iter()` | per-RX zero-copy iov setup | `TlsSw::setup_from_iter` |
| `bpf_exec_tx_verdict()` | per-tx sk_msg BPF program | `TlsSw::bpf_exec_tx_verdict` |
| `tls_read_flush_backlog()` | per-recv socket backlog flush | `TlsSw::read_flush_backlog` |
| `tls_alloc_encrypted_msg()` / `tls_clone_plaintext_msg()` / `tls_trim_both_msgs()` | per-tx sk_msg sizing | `TlsSw::alloc_encrypted_msg` / `clone_plaintext_msg` / `trim_both_msgs` |
| `tx_work_handler()` | per-deferred-tx workqueue | `TlsSw::tx_work_handler` |
| `tls_is_tx_ready()` | per-tx-list ready predicate | `TlsSw::is_tx_ready` |
| `tls_sw_write_space()` | per-sk_write_space override | `TlsSw::write_space` |
| `tls_sw_strparser_arm()` / `tls_sw_strparser_done()` | per-strparser lifecycle | `TlsSw::strparser_arm` / `strparser_done` |
| `tls_update_rx_zc_capable()` | per-RX zc capability recompute | `TlsSw::update_rx_zc_capable` |
| `init_ctx_tx()` / `init_ctx_rx()` | per-direction ctx alloc | `TlsSw::init_ctx_tx` / `init_ctx_rx` |
| `init_prot_info()` | per-cipher overheads from desc | `TlsSw::init_prot_info` |
| `tls_finish_key_update()` | per-1.3 rekey commit | `TlsSw::finish_key_update` |
| `tls_set_sw_offload()` | per-setsockopt SW-mode install | `TlsSw::set_sw_offload` |
| `tls_sw_release_resources_tx()` / `_rx()` / `free_ctx_tx()` / `free_ctx_rx()` / `free_resources_rx()` | per-teardown | `TlsSw::release_*` / `free_ctx_*` |
| `tls_sw_cancel_work_tx()` | per-shutdown cancel tx_work | `TlsSw::cancel_work_tx` |

## Compatibility contract

REQ-1: kTLS ULP install via setsockopt(SOL_TLS, TLS_TX | TLS_RX, struct tls_crypto_info):
- `tls_set_sw_offload(sk, tx, new_crypto_info)` allocates `init_ctx_tx`/`init_ctx_rx`, fetches `cipher_desc = get_cipher_desc(crypto_info->cipher_type)`, populates `tls_prot_info` (prepend_size, tag_size, iv_size, salt_size, overhead_size, aad_size, rec_seq_size, version), allocates AEAD via `crypto_alloc_aead(cipher_desc->cipher_name, 0, 0)`, sets key via `crypto_aead_setkey` and `crypto_aead_setauthsize`. RX-side additionally calls `tls_strp_init`. Per-rekey (`new_crypto_info != NULL`): only setkey replaces material; on failure leaves old state. Per-RX: sets `async_capable = (version != TLS_1_3 ∧ (tfm->__crt_alg->cra_flags & CRYPTO_ALG_ASYNC))`.

REQ-2: struct tls_sw_context_tx (per-direction TX state):
- aead_send: crypto_aead handle.
- tx_list: list_head of `struct tls_rec` pending submission.
- open_rec: current record-in-progress.
- tx_bitmask: BIT_TX_SCHEDULED, BIT_TX_CLOSING.
- tx_work: delayed_work for deferred record push.
- encrypt_pending: atomic_t outstanding async encrypts.
- async_capable: bool flagged after first async-success.
- async_wait: crypto_wait_t.
- tx_lock: held outside lock_sock during sendmsg.

REQ-3: struct tls_sw_context_rx (per-direction RX state):
- aead_recv: crypto_aead handle.
- strp: tls_strparser bound to socket.
- rx_list: skb_queue of decrypted but not yet delivered.
- decrypt_pending: atomic_t outstanding async decrypts.
- async_capable / zc_capable: bool capability flags.
- async_wait: crypto_wait_t.
- saved_data_ready: original sk_data_ready (restored on teardown).
- reader_lock / reader_present: per-recv-mutex exclusion.

REQ-4: struct tls_rec (per-record):
- list (in tx_list).
- tx_ready (READ_ONCE/WRITE_ONCE) — set true when crypto completes.
- tx_flags — passed-through MSG_* flags.
- msg_plaintext (struct sk_msg) — application bytes.
- msg_encrypted (struct sk_msg) — output buffer = plaintext + prepend + tag.
- sg_aead_in[2] / sg_aead_out[2] — chain headers for AAD + payload.
- sg_content_type (TLS 1.3 trailing byte).
- aad_space[TLS_MAX_AAD_SIZE] / iv_data[TLS_MAX_IV_SIZE].
- content_type — TLS_RECORD_TYPE_DATA by default.
- aead_req (struct aead_request).

REQ-5: `tls_sw_sendmsg(sk, msg, size)`:
- Reject `msg->msg_flags & ~(MSG_MORE | MSG_DONTWAIT | MSG_NOSIGNAL | MSG_CMSG_COMPAT | MSG_SPLICE_PAGES | MSG_EOR | MSG_SENDPAGE_NOPOLICY)` with -EOPNOTSUPP.
- `mutex_lock_interruptible(&tls_ctx->tx_lock)` → `lock_sock(sk)` → `tls_sw_sendmsg_locked` → `release_sock` → `mutex_unlock`.

REQ-6: `tls_sw_sendmsg_locked(sk, msg, size)` — main TX loop:
- timeo = sock_sndtimeo(sk, msg->msg_flags & MSG_DONTWAIT).
- record_type = TLS_RECORD_TYPE_DATA; if msg_controllen: `tls_process_cmsg` may override (for TLS_HANDSHAKE etc.).
- while msg_data_left(msg):
  - if sk->sk_err: return -sk->sk_err.
  - rec = ctx->open_rec ?: (ctx->open_rec = tls_get_rec(sk)).
  - try_to_copy = min(msg_data_left, tx_max_payload_len - msg_pl->sg.size); full_record = (try_to_copy == record_room).
  - required_size = msg_pl->sg.size + try_to_copy + prot->overhead_size.
  - if !sk_stream_memory_free: goto wait_for_sndbuf.
  - `tls_alloc_encrypted_msg(sk, required_size)` — sk_msg_alloc on msg_en.
  - if MSG_SPLICE_PAGES: `tls_sw_sendmsg_splice`; goto copied if full_record ∨ eor.
  - else if !is_kvec ∧ (full_record ∨ eor) ∧ !async_capable: try `sk_msg_zerocopy_from_iter` + `bpf_exec_tx_verdict` (zc fast path); on -ENOSPC and no cork_bytes: rollback_iter; on success accumulate copied.
  - else (slow path): `tls_clone_plaintext_msg` (share pages with msg_en) + `sk_msg_memcopy_from_iter` (kvec/copy_from_iter).
  - copied:
    - if full_record ∨ eor: `bpf_exec_tx_verdict(msg_pl, sk, full_record, record_type, &copied, msg_flags)`.
    - if BIT_TX_SCHEDULED: cancel_delayed_work; `tls_tx_records(sk, msg_flags)`.
- send_end / end: if num_async ∧ (num_zc ∨ eor): `tls_encrypt_async_wait(ctx)`. Final flush tx_records. `sk_stream_error(sk, msg_flags, ret)`.

REQ-7: `tls_push_record(sk, flags, record_type)`:
- split_point = msg_pl->apply_bytes; split = split_point ∧ split_point < msg_pl->sg.size.
- if msg_pl + overhead > msg_en: split_point = msg_en->sg.size; `tls_split_open_record(...)` allocates new rec.
- if TLS 1.3: append 1-byte content_type to sg chain (sg_content_type), no header content-type byte modulation.
- else (TLS 1.2): sg_mark_end on msg_pl tail.
- Build AAD: `tls_make_aad(rec->aad_space, msg_pl->sg.size + prot->tail_size, tls_ctx->tx.rec_seq, record_type, prot)`.
- Build prepend (TLS header): `tls_fill_prepend(tls_ctx, page_address(...)+offset, msg_pl_size + tail_size, record_type)` — writes 5-byte TLS header (type, version=TLS_1_2 always-on-wire, length).
- pending_open_record_frags = false.
- `tls_do_encryption(sk, tls_ctx, ctx, req, msg_pl->sg.size + prot->tail_size, i)`.
- On split: keep new rec as open_rec; ctx->async_capable = 1 on -EBADMSG abort path.
- Finally: `tls_tx_records(sk, flags)`.

REQ-8: `tls_do_encryption(sk, ctx, aead_req, data_len, start)`:
- For AES_CCM_128 / SM4_CCM: iv_data[0] = TLS_AES_CCM_IV_B0_BYTE / TLS_SM4_CCM_IV_B0_BYTE; iv_offset = 1.
- memcpy(iv_data + iv_offset, tls_ctx->tx.iv, iv_size + salt_size).
- `tls_xor_iv_with_seq(prot, iv_data + iv_offset, tls_ctx->tx.rec_seq)`.
- sge->offset += prepend; sge->length -= prepend (AEAD covers payload only, prepend = header).
- `aead_request_set_tfm(req, aead_send); set_ad(req, aad_size); set_crypt(req, sg_aead_in, sg_aead_out, data_len, iv_data)`.
- `aead_request_set_callback(req, MAY_BACKLOG, tls_encrypt_done, rec)`.
- list_add_tail(&rec->list, &tx_list); atomic_inc(encrypt_pending).
- rc = crypto_aead_encrypt(req).
- if rc == -EBUSY: `tls_encrypt_async_wait`; rc = err ?: -EINPROGRESS.
- if !rc ∨ rc != -EINPROGRESS: dec encrypt_pending; restore sge->offset/length.
- if !rc: WRITE_ONCE(rec->tx_ready, true).
- else if rc != -EINPROGRESS: list_del; return rc.
- ctx->open_rec = NULL; `tls_advance_record_sn(sk, prot, &tls_ctx->tx)`.

REQ-9: `tls_encrypt_done(data, err)` (AEAD callback context):
- Update sge offsets, set rec->tx_ready or set sk error via `tls_err_abort` on hard error.
- atomic_dec(encrypt_pending); if zero ∧ async waiter: complete crypto_wait.
- Schedule `tx_work` (BIT_TX_SCHEDULED) to push completed records on lock_sock holder.

REQ-10: `tls_tx_records(sk, flags)`:
- Walk tx_list head; for each rec with READ_ONCE(rec->tx_ready) == true:
  - `tcp_sendmsg_locked(sk, msg = { iov from msg_encrypted scatterlist }, size = msg_encrypted->sg.size)`.
  - On success: list_del(&rec->list); `tls_free_rec` (page_unref + kfree).
  - On error: leave at head; return error to caller.

REQ-11: `tls_sw_recvmsg(sk, msg, len, flags)`:
- if MSG_ERRQUEUE: sock_recv_errqueue(...).
- `tls_rx_reader_lock` (per-reader mutex acquire).
- if ctx->async_wait.err: goto end (broken connection).
- `process_rx_list(ctx, msg, &control, 0, len, is_peek, &rx_more)` — drain previously-decrypted rx_list first (copy into iov).
- target = sock_rcvlowat(sk, flags & MSG_WAITALL, len); len -= copied.
- zc_capable = !bpf_strp_enabled ∧ !is_kvec ∧ !is_peek ∧ ctx->zc_capable.
- while len ∧ (decrypted + copied < target ∨ tls_strp_msg_ready):
  - `tls_rx_rec_wait(sk, psock, MSG_DONTWAIT, released)` waits for strparser-framed record.
  - darg.zc = (zc_capable ∧ to_decrypt <= len ∧ tlm->control == TLS_DATA).
  - darg.async = (tlm->control == TLS_DATA ∧ !bpf_strp_enabled) ? ctx->async_capable : false.
  - `tls_rx_one_record(sk, msg, &darg)`:
    - `tls_decrypt_device` then `tls_decrypt_sw` → `tls_do_decryption`.
    - rxm->offset += prepend_size; rxm->full_len -= overhead_size.
    - `tls_advance_record_sn(sk, prot, &tls_ctx->rx)`.
    - `tls_check_pending_rekey(sk, tls_ctx, darg.skb)`.
  - `tls_record_content_type(msg, tls_msg(darg.skb), &control)` — first record sets control + `put_cmsg(SOL_TLS, TLS_GET_RECORD_TYPE)`; mismatched-type → break with MSG_EOR.
  - `tls_read_flush_backlog(sk, prot, ...)` periodically.
  - if !darg.zc: bpf_strp_enabled path runs `sk_psock_tls_strp_read`; else `skb_copy_datagram_msg(skb, rxm->offset, msg, chunk)`; consume_skb / partially_consumed handling.
  - if control != TLS_DATA: break with MSG_EOR.
- recv_end: if async: `tls_decrypt_async_wait`; drain remaining records into iov.
- `tls_rx_reader_unlock`.

REQ-12: `tls_decrypt_sg(sk, out_iov, sgout, darg)` (per-record decrypt SG setup):
- Allocate `tls_decrypt_ctx` with sg[] sized for input + output scatterlists.
- Build AAD per `tls_make_aad` (covers content_type, version, length, seq).
- Build IV: copy tls_ctx->rx.iv (+ salt for AES-GCM), xor with rx.rec_seq.
- if darg->zc: map plaintext into out_iov pages via `tls_setup_from_iter` (user-page pinning).
- else: allocate per-record decrypt output skb (`tls_alloc_clrtxt_skb`).
- `tls_do_decryption(sk, sgin, sgout, iv_recv, data_len, aead_req, darg)`:
  - aead_request_set_tfm(aead_recv); set_ad(aad_size); set_crypt(sgin, sgout, data_len + tag_size, iv_recv).
  - if darg->async: callback = tls_decrypt_done; atomic_inc(decrypt_pending); crypto_aead_decrypt; -EINPROGRESS → return 0; -EBUSY → tls_decrypt_async_wait then darg->async_done = true; darg->async = false.
  - else: synchronous via DECLARE_CRYPTO_WAIT + crypto_wait_req.
- On success: `tls_padding_length` strips trailing zero pad and recovers TLS 1.3 content_type from last non-zero byte; on TLS 1.2 records the cleartext is in place.

REQ-13: `tls_padding_length(prot, skb, darg)` (TLS 1.3 only):
- Scan from end of plaintext backward; first non-zero byte is content_type.
- If only zeros (length 0) or content_type is invalid: -EBADMSG.
- darg->tail = padding length; tlm->control = recovered content_type.
- TLS 1.2 has no padding-with-content-type; tlm->control = header[0].

REQ-14: `tls_rx_msg_size(strp, skb)` (strparser frame callback):
- if rxm->offset + prepend_size > skb->len: return 0 (need more).
- `skb_copy_bits(skb, rxm->offset, header, prepend_size)` — copy TLS header.
- data_len = (header[4] | (header[3] << 8)).
- cipher_overhead = tag_size; if version != TLS_1_3 ∧ cipher != CHACHA20_POLY1305: cipher_overhead += iv_size.
- if data_len > TLS_MAX_PAYLOAD_SIZE (16K) + cipher_overhead + tail_size: -EMSGSIZE.
- if data_len < cipher_overhead: -EBADMSG.
- if header[1] != TLS_1_2_VERSION_MINOR ∨ header[2] != TLS_1_2_VERSION_MAJOR (always-on-wire): -EINVAL.
- `tls_device_rx_resync_new_rec(sk, data_len + TLS_HEADER_SIZE, seq + offset)`.
- return data_len + TLS_HEADER_SIZE.
- read_failure: `tls_strp_abort_strp(strp, ret)`.

REQ-15: `tls_sw_splice_read(sock, ppos, pipe, len, flags)`:
- tls_rx_reader_lock.
- if !skb_queue_empty(rx_list): __skb_dequeue from rx_list.
- else: `tls_rx_rec_wait` + `tls_rx_one_record` (no msghdr, darg defaults).
- if tlm->control != TLS_DATA: -EINVAL, splice_requeue (push back to rx_list head).
- chunk = min(rxm->full_len, len); `skb_splice_bits(skb, sk, rxm->offset, pipe, chunk, flags)`.
- if partial: rxm->offset += copied; rxm->full_len -= copied; splice_requeue.
- else: consume_skb.
- tls_rx_reader_unlock.

REQ-16: `tls_sw_read_sock(sk, desc, read_actor)` (BPF stream/scatter):
- Rejects if sk_psock present (-EINVAL — caller must use BPF strp path).
- tls_rx_reader_acquire (no lock_sock blocking — strparser already holds).
- do { dequeue rx_list ∨ tls_rx_one_record; if !DATA: requeue; used = read_actor(desc, skb, offset, full_len); accumulate } while skb ∧ desc->count.

REQ-17: `bpf_exec_tx_verdict(msg, sk, full_record, record_type, copied, flags)`:
- If sk_psock present with attached sk_msg BPF program: run via `sk_psock_msg_verdict`.
- Verdicts: __SK_PASS → encrypt + push; __SK_DROP → trim_sgl; __SK_REDIRECT → redirect to peer ingress queue; SK_PASS with apply_bytes adjusts cork.
- If no BPF: enqueue tls_push_record or defer via tx_work.

REQ-18: `tls_sw_release_resources_tx(sk)`:
- cancel_delayed_work_sync(&tx_work).
- mutex_lock(tx_lock) [outside lock_sock] to drain encrypt_pending via tls_encrypt_async_wait.
- Walk tx_list: tls_free_rec each.
- crypto_free_aead(aead_send).
- `tls_sw_release_resources_rx`: strp_done; strp_stop; skb_queue_purge(rx_list); crypto_free_aead(aead_recv); restore saved_data_ready.

REQ-19: TLS_VERSION encoding on-wire:
- record-layer header always carries TLS 1.2 version bytes (header[1]=0x03, header[2]=0x03) regardless of negotiated TLS_1_3 (per RFC 8446 §5.1 middlebox compatibility); negotiated version stored in `prot->version`.

REQ-20: Per-`async_capable` invariants:
- TLS 1.3: async_capable = false (async disabled for TLS 1.3).
- TLS 1.2: async_capable = (CRYPTO_ALG_ASYNC set on tfm).
- Decrypt async path moves skbs to rx_list with content_type pre-classified.

## Acceptance Criteria

- [ ] AC-1: setsockopt(SOL_TLS, TLS_TX, &info) with valid crypto_info installs SW offload; subsequent send() goes through tls_sw_sendmsg.
- [ ] AC-2: setsockopt(SOL_TLS, TLS_RX, &info) installs SW strparser; subsequent recv() goes through tls_sw_recvmsg.
- [ ] AC-3: AES-GCM-128 / AES-GCM-256 / AES-CCM-128 / CHACHA20-POLY1305 / SM4-GCM / SM4-CCM all functional via tls_set_sw_offload.
- [ ] AC-4: TLS 1.2 record sent: 5-byte header (type=23, ver=0x0303, len) + explicit IV + payload + 16-byte tag.
- [ ] AC-5: TLS 1.3 record sent: 5-byte header (type=23, ver=0x0303, len) + payload + content_type byte + padding + 16-byte tag.
- [ ] AC-6: sendmsg with MSG_MORE corks the open record; subsequent send without MSG_MORE (or MSG_EOR) flushes.
- [ ] AC-7: sendmsg with MSG_SPLICE_PAGES gathers pages into msg_pl without copy.
- [ ] AC-8: Zero-copy RX: recvmsg with non-kvec, non-peek, no BPF strp → darg.zc = true → plaintext decrypted directly into iov pages.
- [ ] AC-9: TLS 1.3 in-band KeyUpdate (HANDSHAKE record after data): tls_check_pending_rekey triggers; new keys committed via tls_finish_key_update.
- [ ] AC-10: Async encrypt: crypto_aead_encrypt returns -EINPROGRESS; tls_encrypt_done callback marks tx_ready; tx_work pushes to TCP.
- [ ] AC-11: Async decrypt: crypto_aead_decrypt returns -EINPROGRESS; tls_decrypt_done callback fires; tls_decrypt_async_wait drains before final delivery.
- [ ] AC-12: strparser frames record boundaries: TLS records of mixed sizes correctly delimited per tls_rx_msg_size; oversized → -EMSGSIZE; undersized → -EBADMSG; wrong version → -EINVAL.
- [ ] AC-13: splice(2) read on TLS data record returns plaintext via skb_splice_bits; control records → -EINVAL.
- [ ] AC-14: BPF sk_msg attached: bpf_exec_tx_verdict invoked per record; DROP trims; PASS encrypts; REDIRECT forwards to peer.
- [ ] AC-15: BPF strparser attached: sk_psock_tls_strp_read invoked per record post-decrypt; non-PASS verdicts skip user delivery.
- [ ] AC-16: Crypto failure on RX: tls_err_abort sets sk_err = -EBADMSG; next recvmsg returns -EBADMSG.
- [ ] AC-17: Reader-side mutex (`reader_present`) prevents concurrent recvmsg from same socket.
- [ ] AC-18: Teardown: tls_sw_release_resources_tx/rx waits all pending async, frees recs, restores saved_data_ready.

## Architecture

```
struct TlsRec {
  list: ListNode,
  tx_ready: AtomicBool,                       // READ_ONCE/WRITE_ONCE on C side
  tx_flags: u32,
  msg_plaintext: SkMsg,                       // application payload, scatterlist
  msg_encrypted: SkMsg,                       // header + ciphertext + tag, scatterlist
  sg_aead_in: [Scatterlist; 2],
  sg_aead_out: [Scatterlist; 2],
  sg_content_type: Scatterlist,               // TLS 1.3 trailing byte
  aad_space: [u8; TLS_MAX_AAD_SIZE],
  iv_data: [u8; TLS_MAX_IV_SIZE],
  content_type: u8,
  aead_req: AeadRequest,
}

struct TlsSwContextTx {
  aead_send: Arc<CryptoAead>,
  tx_list: List<TlsRec>,
  open_rec: Option<*mut TlsRec>,
  tx_bitmask: AtomicU32,                      // BIT_TX_SCHEDULED | BIT_TX_CLOSING
  tx_work: DelayedWork,
  encrypt_pending: AtomicI32,
  async_capable: bool,
  async_wait: CryptoWait,
}

struct TlsSwContextRx {
  aead_recv: Arc<CryptoAead>,
  strp: TlsStrparser,
  rx_list: SkbQueue,
  decrypt_pending: AtomicI32,
  async_capable: bool,
  zc_capable: bool,
  async_wait: CryptoWait,
  saved_data_ready: Option<unsafe extern "C" fn(*mut Sock)>,
  reader_present: Mutex<()>,
}

struct TlsDecryptArg {
  zc: bool,
  async_: bool,
  async_done: bool,
  tail: u8,
  skb: *mut SkBuff,
}

struct TlsDecryptCtx {
  sk: *mut Sock,
  iv: [u8; TLS_MAX_IV_SIZE],
  aad: [u8; TLS_MAX_AAD_SIZE],
  tail: u8,
  free_sgout: bool,
  sg: [Scatterlist; 0],                       // flexible-array
}
```

`TlsSw::sendmsg(sk, msg, size)`:
1. Check msg_flags whitelist (MSG_MORE | MSG_DONTWAIT | MSG_NOSIGNAL | MSG_CMSG_COMPAT | MSG_SPLICE_PAGES | MSG_EOR | MSG_SENDPAGE_NOPOLICY).
2. tx_lock.lock_interruptible()?.
3. lock_sock(sk).
4. ret = TlsSw::sendmsg_locked(sk, msg, size).
5. release_sock; tx_lock.unlock.
6. return ret.

`TlsSw::sendmsg_locked(sk, msg, size)`:
1. timeo = sock_sndtimeo(sk, MSG_DONTWAIT).
2. record_type = TLS_DATA.
3. if msg->msg_controllen: tls_process_cmsg → record_type.
4. while msg_data_left(msg):
   - if sk->sk_err: return Err(-sk->sk_err).
   - rec = open_rec ?? get_rec().
   - try_to_copy = min(left, tx_max_payload_len - msg_pl.size).
   - alloc_encrypted_msg(required = pl.size + try_to_copy + overhead).
   - if MSG_SPLICE_PAGES: sendmsg_splice → copied.
   - else if !is_kvec ∧ (full_record ∨ eor) ∧ !async_capable:
     - sk_msg_zerocopy_from_iter(msg_pl, try_to_copy).
     - bpf_exec_tx_verdict(msg_pl, full_record, record_type).
     - on -ENOSPC and !cork_bytes: rollback_iter → fallback_to_reg_send.
   - else: clone_plaintext_msg + sk_msg_memcopy_from_iter.
   - if full_record ∨ eor: bpf_exec_tx_verdict → tx_records.
5. send_end: if num_async ∧ (num_zc ∨ eor): tls_encrypt_async_wait.
6. tx_records flush; return sk_stream_error(sk, msg_flags, ret) or copied.

`TlsSw::push_record(sk, flags, record_type)`:
1. rec = open_rec; if None: return 0.
2. split_point = msg_pl.apply_bytes; split = decide(msg_pl, msg_en, overhead).
3. if split: tls_split_open_record → tmp; maybe merge if buffer is single-large.
4. rec.tx_flags = flags; rec.content_type = record_type.
5. if prot.version == TLS_1_3: sg_chain msg_pl.sg + sg_content_type (append 1-byte content_type).
6. else: sg_mark_end(last(msg_pl)).
7. sg_chain rec.sg_aead_in, msg_pl.sg; sg_chain rec.sg_aead_out, msg_en.sg.
8. tls_make_aad(rec.aad_space, msg_pl.size + tail_size, tx.rec_seq, record_type, prot).
9. tls_fill_prepend(tls_ctx, ptr_in_msg_en, msg_pl.size + tail_size, record_type).
10. pending_open_record_frags = false.
11. tls_do_encryption(sk, tls_ctx, ctx, &rec.aead_req, msg_pl.size + tail_size, start).
12. On split + success: open_rec = tmp; pending = true.
13. tls_tx_records(sk, flags).

`TlsSw::do_encryption(sk, ctx, req, data_len, start)`:
1. For AES_CCM / SM4_CCM: rec.iv_data[0] = constant; iv_offset = 1.
2. memcpy(iv_data + iv_offset, tx.iv, iv_size + salt_size).
3. tls_xor_iv_with_seq(prot, iv, tx.rec_seq).
4. sge.offset += prepend; sge.length -= prepend.
5. aead_request_set_tfm/ad/crypt; set_callback(tls_encrypt_done, MAY_BACKLOG).
6. list_add_tail(&rec.list, &tx_list); encrypt_pending += 1.
7. rc = crypto_aead_encrypt(req).
8. if rc == -EBUSY: tls_encrypt_async_wait; rc = err ?: -EINPROGRESS; on err != -EINPROGRESS: list_del, return rc.
9. if !rc ∨ rc != -EINPROGRESS: encrypt_pending -= 1; restore sge.
10. if !rc: WRITE_ONCE(rec.tx_ready, true).
11. else if rc != -EINPROGRESS: list_del, return rc.
12. open_rec = None; tls_advance_record_sn(sk, prot, &tx).
13. return rc.

`TlsSw::recvmsg(sk, msg, len, flags)`:
1. if flags & MSG_ERRQUEUE: return sock_recv_errqueue.
2. tls_rx_reader_lock(sk, ctx, MSG_DONTWAIT)?.
3. psock = sk_psock_get(sk); bpf_strp_enabled = sk_psock_strp_enabled(psock).
4. if async_wait.err: goto end (err).
5. copied = process_rx_list(ctx, msg, &control, 0, len, is_peek, &rx_more)?.
6. if len <= copied ∨ rx_more ∨ (control ∧ control != TLS_DATA): goto end.
7. target = sock_rcvlowat; len -= copied.
8. zc_capable = !bpf_strp ∧ !is_kvec ∧ !is_peek ∧ ctx.zc_capable.
9. while len ∧ (decrypted + copied < target ∨ tls_strp_msg_ready):
   - tls_rx_rec_wait?.
   - rxm = strp_msg(tls_strp_msg(ctx)); tlm = tls_msg(...).
   - to_decrypt = rxm.full_len - prot.overhead_size.
   - darg.zc = zc_capable ∧ to_decrypt <= len ∧ tlm.control == TLS_DATA.
   - darg.async = (tlm.control == TLS_DATA ∧ !bpf_strp) ? ctx.async_capable : false.
   - tls_rx_one_record(sk, msg, &darg)?.
   - tls_record_content_type(msg, tls_msg(darg.skb), &control)?.
   - tls_read_flush_backlog.
   - rxm = strp_msg(darg.skb); chunk = rxm.full_len.
   - tls_rx_rec_done.
   - if !darg.zc: partially_consumed handling; skb_copy_datagram_msg(skb, offset, msg, chunk); peek/partial → put_on_rx_list.
   - decrypted += chunk; len -= chunk; msg.msg_flags |= MSG_EOR if !DATA; break if !DATA.
10. if async: tls_decrypt_async_wait; process_rx_list(... async_copy_bytes ...).
11. copied += decrypted.
12. end: tls_rx_reader_unlock; sk_psock_put.
13. return copied ?: err.

`TlsSw::rx_one_record(sk, msg, darg)`:
1. err = tls_decrypt_device(sk, msg, tls_ctx, darg); if !err: err = tls_decrypt_sw(sk, tls_ctx, msg, darg).
2. if err < 0: return err.
3. rxm = strp_msg(darg.skb); rxm.offset += prepend; rxm.full_len -= overhead.
4. tls_advance_record_sn(sk, prot, &rx).
5. return tls_check_pending_rekey(sk, tls_ctx, darg.skb).

`TlsSw::splice_read(sock, ppos, pipe, len, flags)`:
1. tls_rx_reader_lock(sk, ctx, SPLICE_F_NONBLOCK)?.
2. if !rx_list.is_empty: skb = rx_list.dequeue.
3. else: tls_rx_rec_wait; tls_rx_one_record(sk, None, &darg)?; tls_rx_rec_done; skb = darg.skb.
4. rxm = strp_msg(skb); tlm = tls_msg(skb).
5. if tlm.control != TLS_DATA: err = -EINVAL; goto splice_requeue.
6. chunk = min(rxm.full_len, len).
7. copied = skb_splice_bits(skb, sk, rxm.offset, pipe, chunk, flags)?.
8. if copied < rxm.full_len: rxm.offset += copied; rxm.full_len -= copied; goto splice_requeue.
9. consume_skb.
10. splice_read_end: tls_rx_reader_unlock; return copied ?: err.
11. splice_requeue: rx_list.push_head(skb); goto splice_read_end.

`TlsSw::rx_msg_size(strp, skb)` (strparser callback):
1. if rxm.offset + prepend > skb.len: return Ok(0) (need more bytes).
2. header[..prepend] = skb_copy_bits(skb, rxm.offset, prepend).
3. strp.mark = header[0].
4. data_len = (header[4] as u16) | ((header[3] as u16) << 8).
5. cipher_overhead = tag_size; if version != TLS_1_3 ∧ cipher != CHACHA20: cipher_overhead += iv_size.
6. if data_len > TLS_MAX_PAYLOAD_SIZE + cipher_overhead + tail_size: return Err(-EMSGSIZE).
7. if data_len < cipher_overhead: return Err(-EBADMSG).
8. if header[1] != TLS_1_2_VERSION_MINOR ∨ header[2] != TLS_1_2_VERSION_MAJOR: return Err(-EINVAL).
9. tls_device_rx_resync_new_rec(sk, data_len + TLS_HEADER_SIZE, seq + offset).
10. return Ok(data_len + TLS_HEADER_SIZE).
11. read_failure: tls_strp_abort_strp(strp, ret).

`TlsSw::set_sw_offload(sk, tx, new_crypto_info)`:
1. ctx = tls_get_ctx(sk); prot = &ctx.prot_info.
2. if !new_crypto_info: alloc init_ctx_tx (or init_ctx_rx) into priv_ctx_tx (or priv_ctx_rx).
3. (aead, cctx, crypto_info) = if tx { (aead_send, &tx, &crypto_send.info) } else { (aead_recv, &rx, &crypto_recv.info) }.
4. src = new_crypto_info ?: crypto_info.
5. cipher_desc = get_cipher_desc(src.cipher_type)?.
6. init_prot_info(prot, src, cipher_desc).
7. iv/key/salt/rec_seq = crypto_info_* helpers.
8. if !*aead: *aead = crypto_alloc_aead(cipher_desc.cipher_name, 0, 0)?.
9. ctx.push_pending_record = tls_sw_push_pending_record.
10. crypto_aead_setkey(*aead, key, cipher_desc.key)?.
11. if !new_crypto_info: crypto_aead_setauthsize(*aead, prot.tag_size)?.
12. if !tx ∧ !new_crypto_info:
    - tfm = crypto_aead_tfm(aead_recv).
    - tls_update_rx_zc_capable(ctx).
    - async_capable = (version != TLS_1_3) ∧ (tfm.cra_flags & CRYPTO_ALG_ASYNC).
    - tls_strp_init(&sw_ctx_rx.strp, sk)?.
13. memcpy(cctx.iv, salt + iv); memcpy(cctx.rec_seq, rec_seq).
14. if new_crypto_info: unsafe_memcpy(crypto_info, new, cipher_desc.crypto_info); memzero_explicit(new); if !tx: tls_finish_key_update(sk, ctx).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sendmsg_flags_whitelist` | INVARIANT | per-tls_sw_sendmsg: rejects flags outside (MSG_MORE | MSG_DONTWAIT | MSG_NOSIGNAL | MSG_CMSG_COMPAT | MSG_SPLICE_PAGES | MSG_EOR | MSG_SENDPAGE_NOPOLICY). |
| `iv_size_bounded` | INVARIANT | per-do_encryption: iv_size + salt_size <= TLS_MAX_IV_SIZE. |
| `aad_size_bounded` | INVARIANT | per-make_aad: aad_size <= TLS_MAX_AAD_SIZE. |
| `record_length_bounded` | INVARIANT | per-rx_msg_size: data_len <= TLS_MAX_PAYLOAD_SIZE + cipher_overhead + tail_size. |
| `record_length_min` | INVARIANT | per-rx_msg_size: data_len >= cipher_overhead. |
| `tls_header_version_pinned` | INVARIANT | per-rx_msg_size: header[1] == 0x03 ∧ header[2] == 0x03. |
| `prepend_skb_headlen_safe` | INVARIANT | per-fill_prepend: page_address + offset + prepend <= page boundary. |
| `aead_request_set_crypt_consistent` | INVARIANT | per-do_encryption / do_decryption: sg_len(in) ∧ sg_len(out) >= data_len. |
| `encrypt_pending_balanced` | INVARIANT | per-do_encryption: encrypt_pending inc paired with dec on sync return ∨ callback. |
| `decrypt_pending_balanced` | INVARIANT | per-do_decryption: decrypt_pending inc paired with dec on sync return ∨ callback. |
| `tx_list_only_modified_under_lock` | INVARIANT | per-tx_list: insertion/removal only under lock_sock ∨ tx_work_handler. |
| `rx_list_only_modified_under_reader_lock` | INVARIANT | per-rx_list: insertion/removal only under tls_rx_reader_lock. |
| `open_rec_set_or_free` | INVARIANT | per-encryption-completed: ctx.open_rec = NULL; rec moved to tx_list. |
| `padding_length_recovers_valid_content_type` | INVARIANT | per-padding_length: returned content_type ∈ {DATA, HANDSHAKE, ALERT, CHANGE_CIPHER_SPEC}. |
| `zc_only_for_data_records` | INVARIANT | per-recvmsg: darg.zc ⟹ tlm.control == TLS_DATA. |
| `async_disabled_for_tls13` | INVARIANT | per-recvmsg: prot.version == TLS_1_3 ⟹ darg.async == false. |
| `strparser_aborts_on_oversize_record` | INVARIANT | per-rx_msg_size: data_len > limit ⟹ tls_strp_abort_strp. |

### Layer 2: TLA+

`net/tls/tls-sw.tla`:
- Per-sendmsg + per-encrypt-async + per-recvmsg + per-decrypt-async + per-strparser-callback + per-bpf-verdict.
- Properties:
  - `safety_record_sn_monotonic_tx` — per-encrypt: tls_advance_record_sn called exactly once per successfully-completed record on tx side.
  - `safety_record_sn_monotonic_rx` — per-decrypt: tls_advance_record_sn called exactly once per successfully-decrypted record on rx side.
  - `safety_keystream_no_reuse` — per-record IV = static_iv XOR rec_seq; rec_seq monotonic ⟹ unique IV per record per direction.
  - `safety_tls13_padding_strip_before_userspace` — TLS 1.3 cleartext delivered has padding stripped and content_type recovered.
  - `safety_no_recvmsg_concurrent` — at most one recvmsg in critical section per socket (reader_present mutex).
  - `safety_no_partial_record_to_user` — partial record left in rx_list / rxm.offset advanced; never split delivery across records.
  - `safety_bpf_verdict_drop_no_encrypt` — BPF __SK_DROP ⟹ no AEAD encrypt invocation for that data.
  - `safety_aead_setkey_atomic_on_rekey` — rekey path: setkey is last fallible op; failure leaves prior key intact.
  - `liveness_pending_encrypt_eventually_drains` — encrypt_pending > 0 ∧ no error ⟹ eventually encrypt_pending == 0 (via tls_encrypt_async_wait or callback).
  - `liveness_pending_decrypt_eventually_drains` — analogous for decrypt_pending.
  - `liveness_tx_list_drains_on_close` — tls_sw_release_resources_tx terminates with empty tx_list.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `TlsSw::sendmsg_locked` post: copied >= 0 ∨ negative errno; sk_stream_error invoked exactly once | `TlsSw::sendmsg_locked` |
| `TlsSw::push_record` post: rec.content_type set; aad_space[..aad_size] = make_aad output; prepend bytes = (type | 0x0303 | be16 length) | `TlsSw::push_record` |
| `TlsSw::do_encryption` post: !rc ⟹ WRITE_ONCE(rec.tx_ready, true) ∧ open_rec = None ∧ tls_advance_record_sn invoked | `TlsSw::do_encryption` |
| `TlsSw::do_decryption` post: !rc ⟹ sgout contains plaintext; rc == -EINPROGRESS ⟹ darg.async ∧ decrypt_pending++ | `TlsSw::do_decryption` |
| `TlsSw::rx_one_record` post: rxm.offset += prepend; rxm.full_len -= overhead | `TlsSw::rx_one_record` |
| `TlsSw::rx_msg_size` post: returns 0 ∨ data_len + TLS_HEADER_SIZE ∨ negative errno; tls_strp_abort_strp invoked on error | `TlsSw::rx_msg_size` |
| `TlsSw::splice_read` post: control != TLS_DATA ⟹ skb requeued to rx_list head | `TlsSw::splice_read` |
| `TlsSw::set_sw_offload` post: success ⟹ aead initialized with key ∧ authsize ∧ (rx) strparser initialized | `TlsSw::set_sw_offload` |
| `TlsSw::tx_records` post: each pushed rec removed from tx_list ∧ tls_free_rec invoked | `TlsSw::tx_records` |
| `TlsSw::bpf_exec_tx_verdict` post: verdict ∈ {PASS encrypts, DROP trims, REDIRECT forwards} | `TlsSw::bpf_exec_tx_verdict` |
| `TlsSw::release_resources_tx` post: tx_list empty ∧ encrypt_pending == 0 ∧ aead_send freed | `TlsSw::release_resources_tx` |
| `TlsSw::release_resources_rx` post: rx_list empty ∧ decrypt_pending == 0 ∧ strp stopped ∧ saved_data_ready restored | `TlsSw::release_resources_rx` |

### Layer 4: Verus/Creusot functional

`Per-application-write → tls_sw_sendmsg → sendmsg_locked → tls_get_rec → sk_msg_alloc/zerocopy/memcopy → bpf_exec_tx_verdict → tls_push_record → tls_make_aad + tls_fill_prepend + tls_do_encryption (sync ∨ async + tls_encrypt_done) → tls_advance_record_sn → tls_tx_records → tcp_sendmsg_locked` equivalence with RFC 8446 §5 (Record Protocol) + RFC 5246 §6.2.

`Per-TCP-byte → tls_data_ready → tls_strp_data_ready → tls_rx_msg_size (frame) → tls_sw_recvmsg → tls_rx_one_record → tls_decrypt_sw → tls_do_decryption (sync ∨ async + tls_decrypt_done) → tls_padding_length (TLS 1.3) → tls_check_pending_rekey → record_content_type cmsg → skb_copy_datagram_msg / zc to iov → tls_advance_record_sn` equivalence with RFC 8446 §5 + RFC 5246 §6.2.

## Hardening

(Inherits row-1 features from `net/tls/00-overview.md` § Hardening.)

TLS-SW reinforcement:

- **Per-`tls_xor_iv_with_seq` constant-time** — defense against per-IV-leak timing side-channel.
- **Per-`memzero_explicit` on rekey new_crypto_info** — defense against per-stack-residue key leak after setsockopt rekey.
- **Per-`tls_err_abort` WRITE_ONCE(sk_err) + smp_wmb** — defense against per-recv-race-after-decrypt-fail.
- **Per-`READ_ONCE` / `WRITE_ONCE` on rec->tx_ready** — defense against per-tx_work torn read of cross-CPU state.
- **Per-`tls_rx_reader_lock` mutex** — defense against per-concurrent-recvmsg fragment-interleave.
- **Per-`tx_lock` mutex held outside lock_sock** — defense against per-tx_work vs sendmsg deadlock; ordering: tx_lock then lock_sock.
- **Per-record-length upper bound TLS_MAX_PAYLOAD_SIZE + overhead** — defense against per-overlong-record DoS.
- **Per-version pinned 0x0303 on wire** — defense against per-version-rollback (TLS 1.3 middlebox compatibility).
- **Per-`crypto_aead_setkey` last fallible op on rekey path** — defense against per-rekey partial-state.
- **Per-`atomic_inc(encrypt_pending)` before crypto_aead_encrypt** — defense against per-callback-before-counter race.
- **Per-`cancel_delayed_work_sync` in release_resources_tx** — defense against per-tx_work after free.
- **Per-`tls_strp_abort_strp` on malformed header** — defense against per-strparser livelock on attacker frames.
- **Per-async-disabled-for-TLS-1.3** — defense against per-1.3 in-band record-type recovery race.
- **Per-`unsafe_memcpy` with explicit size assertion on rekey crypto_info** — defense against per-rekey type confusion.

## Open Questions

- Whether Rookery binds AEAD via Linux `crypto/aead` shim or via a Rust-native AEAD trait (RustCrypto `aead` crate) — needs decision before implementation.
- TLS 1.3 0-RTT (Early Data) support: upstream kTLS-SW supports per-`TLS_TX_ZEROCOPY_RO` only; full 0-RTT is application-level.
- Hardware-offload fallback policy when device degrades (`tls_decrypt_device` returns non-zero) — should we fall back silently to SW or surface to userspace?
- Cipher migration on Linux 7.x: SM4 family is gated under CONFIG_TLS_SW; confirm whether Rookery includes SM4-GCM/CCM by default.
- BPF struct_ops for TLS hook integration vs. classic sk_psock — pick one for v1.

## Out of Scope

- net/tls/tls_main.c ULP registration, setsockopt routing (covered in `tls_main.md` Tier-3).
- net/tls/tls_device.c / tls_device_fallback.c hardware offload (covered in `tls_device.md` if expanded).
- net/tls/tls_strp.c stream parser (covered in `tls_strp.md` if expanded; this doc cites the API surface).
- net/tls/tls_proc.c procfs export (covered separately if expanded).
- crypto/aead.c AEAD core (covered in `crypto/aead.md` Tier-3).
- net/core/skmsg.c sk_msg machinery (covered in `net/core/skmsg.md` Tier-3).
- net/core/sock_map.c BPF sockmap (covered in `bpf/sockmap.md` if expanded).
- Userspace handshake (OpenSSL / GnuTLS / ktls-utils) — purely application-side.
- Implementation code.
