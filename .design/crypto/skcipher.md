# Tier-3: crypto/skcipher.c — Symmetric Key Cipher API

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: crypto/00-overview.md
upstream-paths:
  - crypto/skcipher.c (~888 lines)
  - crypto/skcipher.h (in-tree private)
  - include/crypto/skcipher.h
  - include/crypto/internal/skcipher.h
  - crypto/scatterwalk.c (helper)
-->

## Summary

`crypto/skcipher.c` is the **symmetric key cipher** (block-cipher mode) front-end of the Linux Crypto API — the layer above raw block ciphers (AES, DES, …) that handles a *mode* (CBC, CTR, XTS, ECB, …) with chunked scatterlist I/O, IV management, and key install. Per-tfm handle: `struct crypto_skcipher` (with embedded `crypto_tfm base`). Per-algorithm vtable: `struct skcipher_alg` with `setkey / encrypt / decrypt / init / exit / export / import` and metadata `min_keysize / max_keysize / ivsize / chunksize / walksize / statesize`. Per-request descriptor: `struct skcipher_request` (caller-sized at `crypto_skcipher_reqsize(tfm)`) carrying `src` / `dst` scatterlists, `cryptlen`, `iv`, base `crypto_async_request`. Per-walk descriptor: `struct skcipher_walk` (caller-allocated, often on stack) which iterates the request's scatterlists in chunks of `walksize`, providing flat virtual-address windows (`walk.{src,dst}.virt.addr`) for the algorithm to operate on. The walk handles three modes: **FAST** (in-place, properly aligned, same physical page) — direct map; **COPY** (cross-page or misaligned) — buffered through a 1-page bounce buffer; **SLOW** (no contiguous block) — heap-allocated aligned buffer. `skcipher_walk_virt` initializes the walk; `skcipher_walk_done` finishes one step and either advances or terminates. The skcipher type accepts both *native* skcipher implementations and *lskcipher* (linear-state-only skcipher) drivers, dispatching through `cra_type` check. The single funnel: `crypto_alloc_skcipher(name, type, mask)` ⟹ `crypto_alloc_tfm`. Encryption/decryption: `crypto_skcipher_encrypt(req)` / `crypto_skcipher_decrypt(req)` validate NEED_KEY then dispatch to either `crypto_lskcipher_{en,de}crypt_sg` (lskcipher backend) or `alg->{encrypt,decrypt}` (native). Templates `cbc(aes)`, `ctr(aes)`, `xts(aes)`, `ecb(aes)`, etc. register an `skcipher_alg` via `crypto_register_skcipher` (or `skcipher_register_instance` for template-spawned). The convenience helper `skcipher_alloc_instance_simple` instantiates a single-cipher mode template (cbc/ecb-style) deriving keysize/blocksize/alignmask from the underlying cipher. Critical for: dm-crypt full-disk-encryption, fscrypt, IPsec, in-kernel TLS legacy modes, WireGuard secondary KDF.

This Tier-3 covers `crypto/skcipher.c` (~888 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct crypto_skcipher` | per-tfm handle | `CryptoSkcipher` |
| `struct skcipher_alg` | per-algorithm vtable + metadata | `SkcipherAlg` |
| `struct skcipher_alg_common` | shared base (with lskcipher) | `SkcipherAlgCommon` |
| `struct skcipher_request` | per-op descriptor | `SkcipherRequest` |
| `struct skcipher_walk` | per-step iteration cursor | `SkcipherWalk` |
| `struct skcipher_instance` | per-template instance | `SkcipherInstance` |
| `struct skcipher_ctx_simple` | per-simple-mode ctx (cipher *) | `SkcipherCtxSimple` |
| `struct crypto_skcipher_spawn` | per-template spawn binding | `CryptoSkcipherSpawn` |
| `struct crypto_sync_skcipher` | per-sync-only handle alias | `CryptoSyncSkcipher` |
| `crypto_skcipher_type` | per-frontend descriptor | `Skcipher::TYPE` |
| `crypto_alloc_skcipher()` | per-tfm allocation | `Skcipher::alloc` |
| `crypto_alloc_sync_skcipher()` | per-sync-only allocation (mask ASYNC + LARGE_REQSIZE) | `Skcipher::alloc_sync` |
| `crypto_has_skcipher()` | per-availability probe | `Skcipher::has` |
| `crypto_free_skcipher()` (inline) | per-tfm release | `Skcipher::free` |
| `crypto_skcipher_setkey()` | per-tfm key install (alignment-aware, range-checked) | `Skcipher::setkey` |
| `skcipher_setkey_unaligned()` | per-tfm misaligned-key shadow buffer | `Skcipher::setkey_unaligned` |
| `skcipher_set_needkey()` | per-tfm NEED_KEY mark helper | `Skcipher::set_needkey` |
| `crypto_skcipher_encrypt()` | per-req encrypt entry | `Skcipher::encrypt` |
| `crypto_skcipher_decrypt()` | per-req decrypt entry | `Skcipher::decrypt` |
| `crypto_skcipher_export()` / `_import()` | per-req IV/state export/import | `Skcipher::export` / `import` |
| `crypto_lskcipher_export()` / `_import()` | per-lskcipher state shim | `Skcipher::lskcipher_export` / `import` |
| `skcipher_noexport()` / `skcipher_noimport()` | no-op state helpers (statesize == 0) | `Skcipher::no_export` / `no_import` |
| `crypto_skcipher_init_tfm()` | per-tfm init hook | `Skcipher::init_tfm` |
| `crypto_skcipher_exit_tfm()` | per-tfm exit hook | `Skcipher::exit_tfm` |
| `crypto_skcipher_extsize()` | per-tfm context-extension sizing | `Skcipher::extsize` |
| `crypto_skcipher_show()` | per-/proc/crypto print | `Skcipher::show` |
| `crypto_skcipher_report()` | per-CRYPTO_USER netlink report | `Skcipher::report` |
| `crypto_skcipher_free_instance()` | per-instance free | `Skcipher::free_instance` |
| `crypto_grab_skcipher()` | per-template spawn-by-name | `Skcipher::grab` |
| `skcipher_prepare_alg()` | per-alg pre-register validation | `Skcipher::prepare_alg` |
| `skcipher_prepare_alg_common()` | shared validation w/ lskcipher | `Skcipher::prepare_alg_common` |
| `crypto_register_skcipher()` | per-alg register | `Skcipher::register_alg` |
| `crypto_unregister_skcipher()` | per-alg unregister | `Skcipher::unregister_alg` |
| `crypto_register_skciphers()` / `crypto_unregister_skciphers()` | bulk register/unregister | `Skcipher::register_many` / `unregister_many` |
| `skcipher_register_instance()` | per-template-instance register | `Skcipher::register_instance` |
| `skcipher_walk_virt()` | per-walk init for skcipher_request | `SkcipherWalk::init_virt` |
| `skcipher_walk_aead_encrypt()` / `_aead_decrypt()` | per-walk init for AEAD payload | `SkcipherWalk::init_aead_encrypt` / `decrypt` |
| `skcipher_walk_aead_common()` (static) | shared AEAD walk init | `SkcipherWalk::init_aead_common` |
| `skcipher_walk_done()` | per-step finish + advance | `SkcipherWalk::done` |
| `skcipher_walk_first()` (static) | per-walk first step entry | `SkcipherWalk::first` |
| `skcipher_walk_next()` (static) | per-walk next step | `SkcipherWalk::next` |
| `skcipher_next_fast()` (static) | per-step fast path | `SkcipherWalk::next_fast` |
| `skcipher_next_copy()` (static) | per-step copy path | `SkcipherWalk::next_copy` |
| `skcipher_next_slow()` (static) | per-step slow / aligned-alloc path | `SkcipherWalk::next_slow` |
| `skcipher_copy_iv()` (static) | per-walk misaligned-IV bounce | `SkcipherWalk::copy_iv` |
| `skcipher_walk_gfp()` (static inline) | per-walk GFP selector | `SkcipherWalk::gfp` |
| `skcipher_setkey_simple()` (static) | per-simple-mode key delegate | `Skcipher::setkey_simple` |
| `skcipher_init_tfm_simple()` (static) | per-simple-mode tfm init | `Skcipher::init_tfm_simple` |
| `skcipher_exit_tfm_simple()` (static) | per-simple-mode tfm exit | `Skcipher::exit_tfm_simple` |
| `skcipher_free_instance_simple()` (static) | per-simple-mode instance free | `Skcipher::free_instance_simple` |
| `skcipher_alloc_instance_simple()` | per-simple-mode instance alloc | `Skcipher::alloc_instance_simple` |
| `CRYPTO_ALG_TYPE_SKCIPHER` | per-cra_flags type bit | shared |
| `CRYPTO_ALG_TYPE_SKCIPHER_MASK` | type-mask constant (0x0000000e) | shared |
| `MAX_SYNC_SKCIPHER_REQSIZE` | per-sync-stack reqsize ceiling | shared |
| `CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE` | per-large-reqsize bit (excluded from sync alloc) | shared |
| `CRYPTOCFGA_REPORT_BLKCIPHER` | netlink report attribute | shared |
| `SKCIPHER_WALK_SLOW / _COPY / _DIFF / _SLEEP` | per-walk flag bits | `SkcipherWalk::Flags` |

## Compatibility contract

REQ-1: struct crypto_skcipher:
- `base`: embedded `struct crypto_tfm` (template, alg, flags, ctxbuf).
- reqsize: per-tfm tail bytes after `skcipher_request` (set via `crypto_skcipher_set_reqsize`).

REQ-2: struct skcipher_alg:
- `co`: `struct skcipher_alg_common` embedded — shares fields w/ lskcipher (min_keysize, max_keysize, ivsize, chunksize, statesize, base).
- `setkey(tfm, key, keylen) -> int`.
- `encrypt(req) -> int` / `decrypt(req) -> int`.
- `export(req, out) -> int` / `import(req, in) -> int` (optional; defaulted to noop if statesize == 0).
- `init(tfm) -> int` (optional) / `exit(tfm)` (optional).
- `walksize`: per-step preferred byte count (≥ chunksize; ≤ PAGE_SIZE/8); defaulted to `chunksize` if zero.

REQ-3: struct skcipher_alg_common:
- `min_keysize` / `max_keysize`: accepted key length range.
- `ivsize`: per-op IV byte count (≤ PAGE_SIZE/8).
- `chunksize`: per-op natural processing block (≤ PAGE_SIZE/8); defaulted to `cra_blocksize`.
- `statesize`: per-export/import state buffer length (≤ PAGE_SIZE/2; ivsize+statesize ≤ PAGE_SIZE/2).
- `base`: embedded `struct crypto_alg`.

REQ-4: struct skcipher_request:
- `base`: `struct crypto_async_request`.
- `cryptlen`: byte length of payload (src == dst length).
- `iv`: pointer to IV buffer of length `crypto_skcipher_ivsize(tfm)`.
- `src`, `dst`: scatterlists.
- trailing reqsize bytes: per-alg context (often includes IV scratch + state).

REQ-5: struct skcipher_walk:
- `in`, `out`: scatter_walk cursors (with `.addr`, `.offset`, `.sg`).
- `iv`, `oiv`: working IV pointer + original IV pointer (for write-back on done).
- `buffer`: heap bounce buffer (SLOW path).
- `page`: single-page bounce buffer (COPY path).
- `total`: bytes remaining across the entire request.
- `nbytes`: bytes available in the current step.
- `blocksize`: algorithm's block size.
- `stride`: per-step preferred chunk (= alg->walksize for native, alg->chunksize for lskcipher).
- `ivsize`: IV byte count.
- `alignmask`: caller / alg alignment mask.
- `flags`: bitmap of SKCIPHER_WALK_{SLOW,COPY,DIFF,SLEEP}.

REQ-6: SKCIPHER_WALK flag enum:
- `SKCIPHER_WALK_SLOW = 1<<0` — heap-allocated aligned buffer (size = `bsize + (alignmask & ~(ctx_align - 1))`).
- `SKCIPHER_WALK_COPY = 1<<1` — single-page bounce buffer.
- `SKCIPHER_WALK_DIFF = 1<<2` — fast path: src/dst on same page but different addrs (need separate map of `in`).
- `SKCIPHER_WALK_SLEEP = 1<<3` — set when `CRYPTO_TFM_REQ_MAY_SLEEP` ∧ !atomic; selects `GFP_KERNEL` and allows `cond_resched`.

REQ-7: skcipher_walk_gfp(walk):
- return (walk.flags & SLEEP) ? GFP_KERNEL : GFP_ATOMIC.

REQ-8: skcipher_walk_virt(walk, req, atomic):
- tfm = `crypto_skcipher_reqtfm(req)`.
- `might_sleep_if(req->base.flags & CRYPTO_TFM_REQ_MAY_SLEEP)`.
- alg = `crypto_skcipher_alg(tfm)`.
- walk.total = req.cryptlen.
- walk.nbytes = 0.
- walk.iv = walk.oiv = req.iv.
- walk.flags = ((req.base.flags & MAY_SLEEP) ∧ !atomic) ? SLEEP : 0.
- if !walk.total: return 0 /* no data */.
- `scatterwalk_start(&walk.in, req.src)`; `scatterwalk_start(&walk.out, req.dst)`.
- walk.blocksize = `crypto_skcipher_blocksize(tfm)`.
- walk.ivsize = `crypto_skcipher_ivsize(tfm)`.
- walk.alignmask = `crypto_skcipher_alignmask(tfm)`.
- /* lskcipher vs native */
- walk.stride = (alg.co.base.cra_type != &crypto_skcipher_type) ? alg.co.chunksize : alg.walksize.
- return `skcipher_walk_first(walk)`.

REQ-9: skcipher_walk_aead_common(walk, req, atomic):
- tfm = `crypto_aead_reqtfm(req)`.
- walk.nbytes = 0; walk.iv = walk.oiv = req.iv.
- flags: same as walk_virt.
- if !walk.total: return 0.
- `scatterwalk_start_at_pos(&walk.in, req.src, req.assoclen)` /* skip AD prefix */.
- `scatterwalk_start_at_pos(&walk.out, req.dst, req.assoclen)`.
- walk.blocksize = `crypto_aead_blocksize(tfm)`.
- walk.stride = `crypto_aead_chunksize(tfm)`.
- walk.ivsize = `crypto_aead_ivsize(tfm)`.
- walk.alignmask = `crypto_aead_alignmask(tfm)`.
- return `skcipher_walk_first(walk)`.

REQ-10: skcipher_walk_aead_encrypt(walk, req, atomic):
- walk.total = req.cryptlen.
- return `skcipher_walk_aead_common(walk, req, atomic)`.

REQ-11: skcipher_walk_aead_decrypt(walk, req, atomic):
- tfm = `crypto_aead_reqtfm(req)`.
- walk.total = req.cryptlen - `crypto_aead_authsize(tfm)` /* strip tag */.
- return `skcipher_walk_aead_common(walk, req, atomic)`.

REQ-12: skcipher_walk_first(walk):
- `WARN_ON_ONCE(in_hardirq())` ⟹ -EDEADLK (walks must not run in hard-IRQ).
- walk.buffer = NULL.
- if `((unsigned long)walk.iv & walk.alignmask)`: `skcipher_copy_iv(walk)`; on err return err.
- walk.page = NULL.
- return `skcipher_walk_next(walk)`.

REQ-13: skcipher_copy_iv(walk):
- alignmask = walk.alignmask; ivsize = walk.ivsize.
- aligned_stride = `ALIGN(walk.stride, alignmask + 1)`.
- size = aligned_stride + ivsize + (alignmask & ~(crypto_tfm_ctx_alignment() - 1)).
- walk.buffer = `kmalloc(size, skcipher_walk_gfp(walk))`; -ENOMEM on fail.
- iv = `PTR_ALIGN(walk.buffer, alignmask + 1) + aligned_stride`.
- walk.iv = `memcpy(iv, walk.iv, ivsize)`.
- return 0.

REQ-14: skcipher_walk_next(walk):
- n = walk.total.
- bsize = `min(walk.stride, max(n, walk.blocksize))`.
- n = `scatterwalk_clamp(&walk.in, n)`.
- n = `scatterwalk_clamp(&walk.out, n)`.
- if `unlikely(n < bsize)`:
  - if `unlikely(walk.total < walk.blocksize)`: return `skcipher_walk_done(walk, -EINVAL)`.
  - goto slow_path: return `skcipher_next_slow(walk, bsize)`.
- walk.nbytes = n.
- if `unlikely((walk.in.offset | walk.out.offset) & walk.alignmask)`:
  - if !walk.page: walk.page = `(void *)__get_free_page(skcipher_walk_gfp(walk))`; if !page: goto slow_path.
  - walk.flags |= COPY.
  - return `skcipher_next_copy(walk)`.
- return `skcipher_next_fast(walk)`.

REQ-15: skcipher_next_fast(walk):
- diff = `offset_in_page(in.offset) - offset_in_page(out.offset)`.
- diff |= page-pointer difference between in.sg + (in.offset >> PAGE_SHIFT) and out.sg + (out.offset >> PAGE_SHIFT).
- `scatterwalk_map(&walk.out)`.
- walk.in.__addr = walk.out.__addr /* same address baseline */.
- if diff: walk.flags |= DIFF; `scatterwalk_map(&walk.in)` /* separate src map */.
- return 0.

REQ-16: skcipher_next_copy(walk):
- tmp = walk.page.
- `scatterwalk_map(&walk.in)`; memcpy(tmp, walk.in.addr, walk.nbytes); `scatterwalk_unmap(&walk.in)`.
- /* walk.in advanced later when actual processed-count is known. */
- walk.in.__addr = walk.out.__addr = tmp.
- return 0.

REQ-17: skcipher_next_slow(walk, bsize):
- alignmask = walk.alignmask.
- if !walk.buffer: walk.buffer = walk.page.
- buffer = walk.buffer.
- if !buffer:
  - n = bsize + (alignmask & ~(crypto_tfm_ctx_alignment() - 1)).
  - buffer = `kzalloc(n, skcipher_walk_gfp(walk))`; on fail: `skcipher_walk_done(walk, -ENOMEM)`.
  - walk.buffer = buffer.
- buffer = `PTR_ALIGN(buffer, alignmask + 1)`.
- `memcpy_from_scatterwalk(buffer, &walk.in, bsize)`.
- walk.out.__addr = buffer; walk.in.__addr = walk.out.addr.
- walk.nbytes = bsize; walk.flags |= SLOW.
- return 0.

REQ-18: skcipher_walk_done(walk, res):
- n = walk.nbytes; total = 0.
- if !n: goto finish.
- if `likely(res >= 0)`: n -= res; total = walk.total - n /* res = bytes-not-processed */.
- if `likely(!(walk.flags & (SLOW | COPY | DIFF)))`:
  - `scatterwalk_advance(&walk.in, n)` /* fast in-place */.
- else if walk.flags & DIFF:
  - `scatterwalk_done_src(&walk.in, n)`.
- else if walk.flags & COPY:
  - `scatterwalk_advance(&walk.in, n)`.
  - `scatterwalk_map(&walk.out)`; memcpy(walk.out.addr, walk.page, n).
- else /* SLOW */:
  - if res > 0:
    - /* didn't process all bytes: alg is broken or final partial block — fail */.
    - res = -EINVAL; total = 0.
  - else: `memcpy_to_scatterwalk(&walk.out, walk.out.addr, n)`.
  - goto dst_done.
- `scatterwalk_done_dst(&walk.out, n)`.
- dst_done:
- if res > 0: res = 0.
- walk.total = total; walk.nbytes = 0.
- if total:
  - if walk.flags & SLEEP: `cond_resched()`.
  - walk.flags &= ~(SLOW | COPY | DIFF).
  - return `skcipher_walk_next(walk)`.
- finish:
- /* common fast path: no buffer/page */
- if (!walk.buffer ∧ !walk.page): goto out.
- if walk.iv != walk.oiv: `memcpy(walk.oiv, walk.iv, walk.ivsize)` /* write-back IV */.
- if walk.buffer != walk.page: `kfree(walk.buffer)`.
- if walk.page: `free_page((unsigned long)walk.page)`.
- out: return res.

REQ-19: skcipher_set_needkey(tfm):
- if `crypto_skcipher_max_keysize(tfm) != 0`: `crypto_skcipher_set_flags(tfm, CRYPTO_TFM_NEED_KEY)`.
- /* algorithms with max_keysize == 0 (no key) skip NEED_KEY. */

REQ-20: skcipher_setkey_unaligned(tfm, key, keylen):
- alignmask = `crypto_skcipher_alignmask(tfm)`.
- cipher = `crypto_skcipher_alg(tfm)`.
- absize = keylen + alignmask.
- buffer = `kmalloc(absize, GFP_ATOMIC)`; -ENOMEM on fail.
- alignbuffer = `ALIGN(buffer, alignmask + 1)`.
- memcpy(alignbuffer, key, keylen).
- ret = `cipher->setkey(tfm, alignbuffer, keylen)`.
- `kfree_sensitive(buffer)`.
- return ret.

REQ-21: crypto_skcipher_setkey(tfm, key, keylen):
- cipher = `crypto_skcipher_alg(tfm)`; alignmask = `crypto_skcipher_alignmask(tfm)`.
- /* lskcipher dispatch */
- if cipher.co.base.cra_type != `&crypto_skcipher_type`:
  - ctx = `crypto_skcipher_ctx(tfm)` /* points to `struct crypto_lskcipher **` */.
  - `crypto_lskcipher_clear_flags(*ctx, CRYPTO_TFM_REQ_MASK)`.
  - `crypto_lskcipher_set_flags(*ctx, crypto_skcipher_get_flags(tfm) & CRYPTO_TFM_REQ_MASK)`.
  - err = `crypto_lskcipher_setkey(*ctx, key, keylen)`.
  - goto out.
- /* native skcipher */
- if keylen < cipher.min_keysize ∨ keylen > cipher.max_keysize: return -EINVAL.
- if ((unsigned long)key & alignmask): err = `skcipher_setkey_unaligned(tfm, key, keylen)`.
- else: err = `cipher->setkey(tfm, key, keylen)`.
- out:
- if `unlikely(err)`: `skcipher_set_needkey(tfm)`; return err.
- `crypto_skcipher_clear_flags(tfm, CRYPTO_TFM_NEED_KEY)`; return 0.

REQ-22: crypto_skcipher_encrypt(req):
- tfm = `crypto_skcipher_reqtfm(req)`; alg = `crypto_skcipher_alg(tfm)`.
- if `crypto_skcipher_get_flags(tfm) & CRYPTO_TFM_NEED_KEY`: return -ENOKEY.
- if alg.co.base.cra_type != `&crypto_skcipher_type`: return `crypto_lskcipher_encrypt_sg(req)`.
- return `alg->encrypt(req)`.

REQ-23: crypto_skcipher_decrypt(req):
- analogous; lskcipher-backed ⟹ `crypto_lskcipher_decrypt_sg(req)`; else `alg->decrypt(req)`.

REQ-24: crypto_skcipher_export(req, out):
- alg = `crypto_skcipher_alg(crypto_skcipher_reqtfm(req))`.
- if alg.co.base.cra_type != `&crypto_skcipher_type`: return `crypto_lskcipher_export(req, out)`.
- return `alg->export(req, out)`.

REQ-25: crypto_skcipher_import(req, in):
- analogous; lskcipher-backed ⟹ `crypto_lskcipher_import(req, in)`; else `alg->import(req, in)`.

REQ-26: crypto_lskcipher_export(req, out) (static):
- tfm = `crypto_skcipher_reqtfm(req)`; ivs = `skcipher_request_ctx(req)`.
- ivs = `PTR_ALIGN(ivs, crypto_skcipher_alignmask(tfm) + 1)`.
- `memcpy(out, ivs + crypto_skcipher_ivsize(tfm), crypto_skcipher_statesize(tfm))`.
- return 0.

REQ-27: crypto_lskcipher_import(req, in) (static):
- analogous in the opposite direction.

REQ-28: skcipher_noexport(req, out) / skcipher_noimport(req, in):
- return 0 /* statesize == 0 alg: no-op. */

REQ-29: crypto_skcipher_init_tfm(tfm):
- skcipher = `__crypto_skcipher_cast(tfm)`; alg = `crypto_skcipher_alg(skcipher)`.
- `skcipher_set_needkey(skcipher)`.
- /* lskcipher backend */
- if tfm.__crt_alg.cra_type != `&crypto_skcipher_type`:
  - am = `crypto_skcipher_alignmask(skcipher)`.
  - reqsize = am & ~(crypto_tfm_ctx_alignment() - 1).
  - reqsize += `crypto_skcipher_ivsize(skcipher)` + `crypto_skcipher_statesize(skcipher)`.
  - `crypto_skcipher_set_reqsize(skcipher, reqsize)`.
  - return `crypto_init_lskcipher_ops_sg(tfm)`.
- /* native */
- `crypto_skcipher_set_reqsize(skcipher, crypto_tfm_alg_reqsize(tfm))`.
- if alg.exit: skcipher.base.exit = `crypto_skcipher_exit_tfm`.
- if alg.init: return `alg->init(skcipher)`.
- return 0.

REQ-30: crypto_skcipher_exit_tfm(tfm):
- skcipher = cast; alg = `crypto_skcipher_alg(skcipher)`; `alg->exit(skcipher)`.

REQ-31: crypto_skcipher_extsize(alg):
- if alg.cra_type != `&crypto_skcipher_type`: return sizeof(`struct crypto_lskcipher *`).
- return `crypto_alg_extsize(alg)`.

REQ-32: crypto_skcipher_show(m, alg):
- prints: type=skcipher, async, blocksize, min_keysize, max_keysize, ivsize, chunksize, walksize, statesize.

REQ-33: crypto_skcipher_report(skb, alg):
- rblkcipher zeroed; type="skcipher"; geniv="<none>".
- blocksize / min_keysize / max_keysize / ivsize set.
- `nla_put(skb, CRYPTOCFGA_REPORT_BLKCIPHER, ...)`.

REQ-34: crypto_skcipher_type frontend:
```
{
  .extsize  = crypto_skcipher_extsize,
  .init_tfm = crypto_skcipher_init_tfm,
  .free     = crypto_skcipher_free_instance,
  .show     = crypto_skcipher_show,        // CONFIG_PROC_FS
  .report   = crypto_skcipher_report,      // CONFIG_CRYPTO_USER
  .maskclear = ~CRYPTO_ALG_TYPE_MASK,
  .maskset   =  CRYPTO_ALG_TYPE_SKCIPHER_MASK, // 0x0000000e
  .type      =  CRYPTO_ALG_TYPE_SKCIPHER,
  .tfmsize   = offsetof(struct crypto_skcipher, base),
  .algsize   = offsetof(struct skcipher_alg, base),
}
```

REQ-35: crypto_grab_skcipher(spawn, inst, name, type, mask):
- spawn.base.frontend = `&crypto_skcipher_type`.
- return `crypto_grab_spawn(&spawn->base, inst, name, type, mask)`.

REQ-36: crypto_alloc_skcipher(alg_name, type, mask):
- return `crypto_alloc_tfm(alg_name, &crypto_skcipher_type, type, mask)`.

REQ-37: crypto_alloc_sync_skcipher(alg_name, type, mask):
- mask |= `CRYPTO_ALG_ASYNC` | `CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE`.
- type &= ~(CRYPTO_ALG_ASYNC | CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE).
- tfm = `crypto_alloc_tfm(alg_name, &crypto_skcipher_type, type, mask)`.
- if !IS_ERR(tfm) ∧ `WARN_ON(crypto_skcipher_reqsize(tfm) > MAX_SYNC_SKCIPHER_REQSIZE)`:
  - `crypto_free_skcipher(tfm)`; return ERR_PTR(-EINVAL).
- return (struct crypto_sync_skcipher *)tfm.

REQ-38: crypto_has_skcipher(name, type, mask):
- `crypto_type_has_alg(name, &crypto_skcipher_type, type, mask)`.

REQ-39: skcipher_prepare_alg_common(alg):
- base = &alg.base.
- if alg.ivsize > PAGE_SIZE/8 ∨ alg.chunksize > PAGE_SIZE/8 ∨ alg.statesize > PAGE_SIZE/2 ∨ (alg.ivsize + alg.statesize) > PAGE_SIZE/2: -EINVAL.
- if !alg.chunksize: alg.chunksize = base.cra_blocksize.
- base.cra_flags &= ~CRYPTO_ALG_TYPE_MASK.
- return 0.

REQ-40: skcipher_prepare_alg(alg):
- err = `skcipher_prepare_alg_common(&alg->co)`; if err: return err.
- if alg.walksize > PAGE_SIZE/8: -EINVAL.
- if !alg.walksize: alg.walksize = alg.chunksize.
- if !alg.statesize:
  - alg.import = `skcipher_noimport`; alg.export = `skcipher_noexport`.
- else if !(alg.import ∧ alg.export): -EINVAL.
- base.cra_type = `&crypto_skcipher_type`.
- base.cra_flags |= CRYPTO_ALG_TYPE_SKCIPHER.
- return 0.

REQ-41: crypto_register_skcipher(alg):
- err = `skcipher_prepare_alg(alg)`; if err: return err.
- return `crypto_register_alg(&alg->base)`.

REQ-42: crypto_unregister_skcipher(alg):
- `crypto_unregister_alg(&alg->base)`.

REQ-43: crypto_register_skciphers(algs, count):
- for i in 0..count: ret = `crypto_register_skcipher(&algs[i])`; if ret: `crypto_unregister_skciphers(algs, i)`; return ret.
- return 0.

REQ-44: crypto_unregister_skciphers(algs, count):
- for i = count-1 down to 0: `crypto_unregister_skcipher(&algs[i])`.

REQ-45: skcipher_register_instance(tmpl, inst):
- if `WARN_ON(!inst->free)`: return -EINVAL.
- err = `skcipher_prepare_alg(&inst->alg)`; if err: return err.
- return `crypto_register_instance(tmpl, skcipher_crypto_instance(inst))`.

REQ-46: skcipher_alloc_instance_simple(tmpl, tb):
- err = `crypto_check_attr_type(tb, CRYPTO_ALG_TYPE_SKCIPHER, &mask)`.
- inst = `kzalloc(sizeof(*inst) + sizeof(*spawn), GFP_KERNEL)`.
- spawn = `skcipher_instance_ctx(inst)`.
- `crypto_grab_cipher(spawn, skcipher_crypto_instance(inst), crypto_attr_alg_name(tb[1]), 0, mask)` /* underlying cipher */.
- cipher_alg = `crypto_spawn_cipher_alg(spawn)`.
- `crypto_inst_setname(skcipher_crypto_instance(inst), tmpl->name, cipher_alg)`.
- inst.free = `skcipher_free_instance_simple`.
- /* Default properties (overridable) */
- inst.alg.base.cra_blocksize = cipher_alg.cra_blocksize.
- inst.alg.base.cra_alignmask = cipher_alg.cra_alignmask.
- inst.alg.base.cra_priority = cipher_alg.cra_priority.
- inst.alg.min_keysize = cipher_alg.cra_cipher.cia_min_keysize.
- inst.alg.max_keysize = cipher_alg.cra_cipher.cia_max_keysize.
- inst.alg.ivsize = cipher_alg.cra_blocksize /* default: full block as IV */.
- inst.alg.base.cra_ctxsize = sizeof(`struct skcipher_ctx_simple`).
- inst.alg.setkey = `skcipher_setkey_simple`.
- inst.alg.init = `skcipher_init_tfm_simple`.
- inst.alg.exit = `skcipher_exit_tfm_simple`.
- return inst /* caller still needs to register */.

REQ-47: skcipher_setkey_simple(tfm, key, keylen):
- cipher = `skcipher_cipher_simple(tfm)`.
- `crypto_cipher_clear_flags(cipher, CRYPTO_TFM_REQ_MASK)`.
- `crypto_cipher_set_flags(cipher, crypto_skcipher_get_flags(tfm) & CRYPTO_TFM_REQ_MASK)`.
- return `crypto_cipher_setkey(cipher, key, keylen)`.

REQ-48: skcipher_init_tfm_simple(tfm) / skcipher_exit_tfm_simple(tfm):
- init: spawn = `skcipher_instance_ctx(inst)`; cipher = `crypto_spawn_cipher(spawn)`; ctx.cipher = cipher; return 0.
- exit: `crypto_free_cipher(ctx.cipher)`.

REQ-49: skcipher_free_instance_simple(inst):
- `crypto_drop_cipher(skcipher_instance_ctx(inst))`.
- `kfree(inst)`.

REQ-50: Module metadata:
- `MODULE_LICENSE("GPL")`.
- `MODULE_DESCRIPTION("Symmetric key cipher type")`.
- `MODULE_IMPORT_NS("CRYPTO_INTERNAL")`.

REQ-51: Sync vs async semantics:
- Async impls (CRYPTO_ALG_ASYNC) may return -EINPROGRESS / -EBUSY from encrypt/decrypt; completion delivered via `req->base.complete(req, err)` callback.
- Sync impls return 0 / negative errno on completion.
- `crypto_alloc_sync_skcipher` masks ASYNC ∧ SKCIPHER_REQSIZE_LARGE to guarantee on-stack `SYNC_SKCIPHER_REQUEST_ON_STACK` usability.

REQ-52: Walk-buffer chunking guarantees:
- Per-step `walk.nbytes` is a multiple of `walk.blocksize` (except possibly the final step on a non-block-aligned request, in which case SLOW path with -EINVAL termination protects the alg).
- Per-step `walk.{in,out}.addr` provides a flat virtual mapping safe to read/write for `walk.nbytes` bytes.
- IV: alg may mutate `walk.iv` in-place; on `walk.total == 0` finish, written back to `walk.oiv == req.iv`.

## Acceptance Criteria

- [ ] AC-1: `crypto_alloc_skcipher("cbc(aes)", 0, 0)` returns a valid tfm when CRYPTO_CBC=y and CRYPTO_AES=y.
- [ ] AC-2: `crypto_alloc_sync_skcipher` masks both `CRYPTO_ALG_ASYNC` and `CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE`, and rejects (WARN + -EINVAL) any chosen impl with `reqsize > MAX_SYNC_SKCIPHER_REQSIZE`.
- [ ] AC-3: `crypto_skcipher_setkey` returns -EINVAL when `keylen < min_keysize` or `keylen > max_keysize` (native path).
- [ ] AC-4: `crypto_skcipher_setkey` with misaligned key routes through `skcipher_setkey_unaligned` and uses `kfree_sensitive` on the shadow buffer.
- [ ] AC-5: `crypto_skcipher_setkey` failure leaves `CRYPTO_TFM_NEED_KEY` set (via `skcipher_set_needkey`); success clears it.
- [ ] AC-6: `crypto_skcipher_encrypt` / `_decrypt` return -ENOKEY when NEED_KEY is set.
- [ ] AC-7: lskcipher-backed algs route encrypt/decrypt through `crypto_lskcipher_{encrypt,decrypt}_sg` (`cra_type != crypto_skcipher_type`).
- [ ] AC-8: `crypto_skcipher_init_tfm` for an lskcipher backend computes `reqsize = align_pad + ivsize + statesize` and installs `crypto_init_lskcipher_ops_sg`.
- [ ] AC-9: `crypto_skcipher_init_tfm` for a native skcipher installs `crypto_skcipher_exit_tfm` only when `alg->exit != NULL`.
- [ ] AC-10: `skcipher_walk_virt` on an empty request (cryptlen == 0) returns 0 immediately without touching the scatterlists.
- [ ] AC-11: `skcipher_walk_first` returns -EDEADLK with WARN_ON_ONCE when called from a hard-IRQ context.
- [ ] AC-12: A misaligned IV triggers `skcipher_copy_iv` which kmallocs `aligned_stride + ivsize + ctx-align padding` and sets `walk.iv` to the aligned shadow buffer.
- [ ] AC-13: `skcipher_walk_next` selects FAST when src and dst share a page and offsets meet alignmask; selects COPY (single-page bounce) when alignment fails; selects SLOW (heap aligned buffer) when scatterwalk_clamp returns less than bsize.
- [ ] AC-14: `skcipher_walk_done` writes back IV (`walk.iv != walk.oiv` ⟹ memcpy oiv ← iv) and frees both `walk.buffer` (if separate from `walk.page`) and `walk.page` at the final step.
- [ ] AC-15: On SLOW path with `res > 0` (alg didn't process full block), `skcipher_walk_done` upgrades to -EINVAL (alg is broken or non-block-aligned final).
- [ ] AC-16: `skcipher_walk_done` between steps with `SKCIPHER_WALK_SLEEP` flag calls `cond_resched()` before advancing.
- [ ] AC-17: `skcipher_prepare_alg` rejects `ivsize > PAGE_SIZE/8`, `chunksize > PAGE_SIZE/8`, `statesize > PAGE_SIZE/2`, `ivsize + statesize > PAGE_SIZE/2`, or `walksize > PAGE_SIZE/8`.
- [ ] AC-18: `skcipher_prepare_alg` defaults `chunksize = cra_blocksize` if zero, `walksize = chunksize` if zero; if `statesize == 0` defaults import/export to no-op helpers; if `statesize != 0` but either import or export is NULL: -EINVAL.
- [ ] AC-19: `skcipher_register_instance` returns -EINVAL (with WARN) when `inst->free == NULL`.
- [ ] AC-20: `crypto_register_skciphers` rolls back fully on partial failure.
- [ ] AC-21: `skcipher_alloc_instance_simple` returns an instance with defaults: `cra_blocksize`/`cra_alignmask`/`cra_priority` from the underlying cipher, `min_keysize`/`max_keysize` from `cra_cipher.cia_{min,max}_keysize`, `ivsize = cra_blocksize`, ctx size = `sizeof(skcipher_ctx_simple)`, default setkey/init/exit installed.

## Architecture

```
struct CryptoSkcipher {
  base: CryptoTfm,                       // template, alg, flags, ctxbuf
}

struct SkcipherAlgCommon {
  min_keysize: u32,
  max_keysize: u32,
  ivsize: u32,
  chunksize: u32,
  statesize: u32,
  base: CryptoAlg,
}

struct SkcipherAlg {
  setkey: Option<fn(&CryptoSkcipher, &[u8]) -> i32>,
  encrypt: fn(&mut SkcipherRequest) -> i32,
  decrypt: fn(&mut SkcipherRequest) -> i32,
  export: fn(&SkcipherRequest, *mut u8) -> i32,
  import: fn(&mut SkcipherRequest, *const u8) -> i32,
  init: Option<fn(&CryptoSkcipher) -> i32>,
  exit: Option<fn(&CryptoSkcipher)>,
  walksize: u32,
  co: SkcipherAlgCommon,
}

struct SkcipherRequest {
  base: CryptoAsyncRequest,
  cryptlen: u32,
  iv: *mut u8,                            // length = crypto_skcipher_ivsize(tfm)
  src: *const ScatterList,
  dst: *mut ScatterList,
  // trailing crypto_skcipher_reqsize(tfm) bytes alg-private
}

struct SkcipherWalk {
  in_: ScatterWalk,
  out: ScatterWalk,
  iv: *mut u8,
  oiv: *mut u8,                           // original; write-back target
  buffer: *mut u8,                        // SLOW heap alloc
  page: *mut u8,                          // COPY single-page bounce
  total: u32,
  nbytes: u32,
  blocksize: u32,
  stride: u32,
  ivsize: u32,
  alignmask: u32,
  flags: u32,                             // SKCIPHER_WALK_{SLOW,COPY,DIFF,SLEEP}
}

enum SkcipherWalkFlag {
  SLOW  = 1 << 0,
  COPY  = 1 << 1,
  DIFF  = 1 << 2,
  SLEEP = 1 << 3,
}
```

`Skcipher::TYPE: CryptoType`:
```
CryptoType {
  extsize: Skcipher::extsize,
  init_tfm: Skcipher::init_tfm,
  free: Skcipher::free_instance,
  show: Skcipher::show,
  report: Skcipher::report,
  maskclear: !CRYPTO_ALG_TYPE_MASK,
  maskset: CRYPTO_ALG_TYPE_SKCIPHER_MASK,
  type: CRYPTO_ALG_TYPE_SKCIPHER,
  tfmsize: offset_of!(CryptoSkcipher, base),
  algsize: offset_of!(SkcipherAlg, base),
}
```

`Skcipher::setkey(tfm, key) -> Result<(), i32>`:
1. cipher = tfm.alg().
2. /* lskcipher dispatch */
3. if cipher.co.base.cra_type != &Skcipher::TYPE:
   - ctx: &&mut CryptoLskcipher = tfm.ctx();
   - ctx.clear_flags(CRYPTO_TFM_REQ_MASK);
   - ctx.set_flags(tfm.flags() & CRYPTO_TFM_REQ_MASK);
   - err = CryptoLskcipher::setkey(*ctx, key);
   - goto out.
4. /* native */
5. if key.len() < cipher.co.min_keysize ∨ key.len() > cipher.co.max_keysize: return Err(-EINVAL).
6. err = if (key.as_ptr() as usize) & tfm.alignmask() != 0:
   - Skcipher::setkey_unaligned(tfm, key).
   } else:
   - (cipher.setkey.unwrap())(tfm, key).
7. out:
8. if err.is_err():
   - Skcipher::set_needkey(tfm);
   - return err.
9. tfm.clear_flag(CRYPTO_TFM_NEED_KEY).
10. Ok(()).

`Skcipher::setkey_unaligned(tfm, key) -> Result<(), i32>`:
1. alignmask = tfm.alignmask().
2. cipher = tfm.alg().
3. absize = key.len() + alignmask.
4. buffer = KVec::with_capacity(absize, GFP_ATOMIC).ok_or(-ENOMEM)?.
5. aligned = align_up_ptr(buffer.as_mut_ptr(), alignmask + 1).
6. memcpy(aligned, key.as_ptr(), key.len()).
7. ret = (cipher.setkey)(tfm, slice::from_raw_parts(aligned, key.len())).
8. kfree_sensitive(buffer).
9. ret.

`Skcipher::set_needkey(tfm)`:
1. if tfm.max_keysize() != 0: tfm.set_flag(CRYPTO_TFM_NEED_KEY).

`Skcipher::encrypt(req) -> i32`:
1. tfm = req.reqtfm(); alg = tfm.alg().
2. if tfm.flags() & CRYPTO_TFM_NEED_KEY: return -ENOKEY.
3. if alg.co.base.cra_type != &Skcipher::TYPE: return CryptoLskcipher::encrypt_sg(req).
4. (alg.encrypt)(req).

`Skcipher::decrypt(req) -> i32`:
1. analogous; lskcipher ⟹ CryptoLskcipher::decrypt_sg.

`Skcipher::export(req, out) -> i32`:
1. alg = req.reqtfm().alg().
2. if alg.co.base.cra_type != &Skcipher::TYPE: return Skcipher::lskcipher_export(req, out).
3. (alg.export)(req, out).

`Skcipher::import(req, in_) -> i32`:
1. analogous.

`Skcipher::lskcipher_export(req, out) -> i32`:
1. tfm = req.reqtfm().
2. ivs = req.ctx() as *mut u8.
3. ivs = align_up_ptr(ivs, tfm.alignmask() + 1).
4. memcpy(out, ivs + tfm.ivsize(), tfm.statesize()).
5. 0.

`Skcipher::lskcipher_import(req, in_) -> i32`:
1. analogous (opposite direction).

`Skcipher::init_tfm(tfm) -> i32`:
1. skcipher = CryptoSkcipher::cast(tfm).
2. alg = skcipher.alg().
3. Skcipher::set_needkey(skcipher).
4. if tfm.__crt_alg.cra_type != &Skcipher::TYPE:
   - am = skcipher.alignmask().
   - reqsize = am & !(crypto_tfm_ctx_alignment() - 1).
   - reqsize += skcipher.ivsize() + skcipher.statesize().
   - skcipher.set_reqsize(reqsize).
   - return crypto_init_lskcipher_ops_sg(tfm).
5. skcipher.set_reqsize(tfm.alg_reqsize()).
6. if alg.exit.is_some(): skcipher.base.exit = Skcipher::exit_tfm.
7. if let Some(init) = alg.init: return init(skcipher).
8. 0.

`Skcipher::exit_tfm(tfm)`:
1. skcipher = CryptoSkcipher::cast(tfm).
2. (skcipher.alg().exit.unwrap())(skcipher).

`Skcipher::extsize(alg) -> u32`:
1. if alg.cra_type != &Skcipher::TYPE: size_of::<*mut CryptoLskcipher>() as u32.
2. else: crypto_alg_extsize(alg).

`Skcipher::free_instance(inst)`:
1. skcipher = SkcipherInstance::from_inst(inst).
2. (skcipher.free)(skcipher).

`Skcipher::grab(spawn, inst, name, type, mask) -> Result<(), i32>`:
1. spawn.base.frontend = &Skcipher::TYPE.
2. crypto_grab_spawn(&spawn.base, inst, name, type, mask).

`Skcipher::alloc(name, type, mask)`:
1. crypto_alloc_tfm(name, &Skcipher::TYPE, type, mask).

`Skcipher::alloc_sync(name, type, mask)`:
1. mask |= CRYPTO_ALG_ASYNC | CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE.
2. type &= !(CRYPTO_ALG_ASYNC | CRYPTO_ALG_SKCIPHER_REQSIZE_LARGE).
3. tfm = crypto_alloc_tfm(name, &Skcipher::TYPE, type, mask)?.
4. if tfm.reqsize() > MAX_SYNC_SKCIPHER_REQSIZE:
   - WARN_ONCE; Skcipher::free(tfm); return Err(-EINVAL).
5. Ok(tfm as *mut CryptoSyncSkcipher).

`Skcipher::prepare_alg_common(alg) -> Result<(), i32>`:
1. if alg.ivsize > PAGE_SIZE/8 ∨ alg.chunksize > PAGE_SIZE/8 ∨ alg.statesize > PAGE_SIZE/2 ∨ (alg.ivsize + alg.statesize) > PAGE_SIZE/2: return Err(-EINVAL).
2. if alg.chunksize == 0: alg.chunksize = alg.base.cra_blocksize.
3. alg.base.cra_flags &= !CRYPTO_ALG_TYPE_MASK.
4. Ok(()).

`Skcipher::prepare_alg(alg) -> Result<(), i32>`:
1. Skcipher::prepare_alg_common(&alg.co)?.
2. if alg.walksize > PAGE_SIZE/8: return Err(-EINVAL).
3. if alg.walksize == 0: alg.walksize = alg.co.chunksize.
4. if alg.co.statesize == 0:
   - alg.import = Skcipher::no_import;
   - alg.export = Skcipher::no_export.
   else if alg.import.is_none() ∨ alg.export.is_none(): return Err(-EINVAL).
5. alg.base.cra_type = &Skcipher::TYPE.
6. alg.base.cra_flags |= CRYPTO_ALG_TYPE_SKCIPHER.
7. Ok(()).

`Skcipher::register_alg(alg)` / `unregister_alg(alg)`: trivial wrappers around `crypto_register_alg` / `crypto_unregister_alg` with `prepare_alg` in front.

`Skcipher::register_many(algs)` / `unregister_many(algs)`: register in order, on first failure roll back via `unregister_many(algs, i)`.

`Skcipher::register_instance(tmpl, inst)`:
1. if inst.free.is_none(): WARN_ON; return -EINVAL.
2. Skcipher::prepare_alg(&inst.alg)?.
3. crypto_register_instance(tmpl, SkcipherInstance::to_crypto(inst)).

### Walk machinery

`SkcipherWalk::init_virt(walk, req, atomic) -> Result<(), i32>`:
1. tfm = req.reqtfm().
2. if req.base.flags & CRYPTO_TFM_REQ_MAY_SLEEP: might_sleep().
3. alg = tfm.alg().
4. walk.total = req.cryptlen.
5. walk.nbytes = 0.
6. walk.iv = walk.oiv = req.iv.
7. walk.flags = if (req.base.flags & MAY_SLEEP) ∧ !atomic { SLEEP } else { 0 }.
8. if walk.total == 0: return Ok(()) /* nothing to do */.
9. ScatterWalk::start(&walk.in_, req.src).
10. ScatterWalk::start(&walk.out, req.dst).
11. walk.blocksize = tfm.blocksize().
12. walk.ivsize = tfm.ivsize().
13. walk.alignmask = tfm.alignmask().
14. walk.stride = if alg.co.base.cra_type != &Skcipher::TYPE { alg.co.chunksize } else { alg.walksize }.
15. SkcipherWalk::first(walk).

`SkcipherWalk::init_aead_common(walk, req, atomic)`:
1. tfm = req.reqtfm().
2. walk.nbytes = 0; walk.iv = walk.oiv = req.iv; walk.flags = ... (SLEEP gating).
3. if walk.total == 0: Ok(()).
4. ScatterWalk::start_at_pos(&walk.in_, req.src, req.assoclen).
5. ScatterWalk::start_at_pos(&walk.out, req.dst, req.assoclen).
6. walk.blocksize = tfm.blocksize(); walk.stride = tfm.chunksize(); walk.ivsize = tfm.ivsize(); walk.alignmask = tfm.alignmask().
7. SkcipherWalk::first(walk).

`SkcipherWalk::init_aead_encrypt(walk, req, atomic)`:
1. walk.total = req.cryptlen; SkcipherWalk::init_aead_common(walk, req, atomic).

`SkcipherWalk::init_aead_decrypt(walk, req, atomic)`:
1. walk.total = req.cryptlen - req.reqtfm().authsize(); SkcipherWalk::init_aead_common(walk, req, atomic).

`SkcipherWalk::first(walk) -> i32`:
1. if in_hardirq(): WARN_ON_ONCE; return -EDEADLK.
2. walk.buffer = null.
3. if (walk.iv as usize) & walk.alignmask != 0: SkcipherWalk::copy_iv(walk)?.
4. walk.page = null.
5. SkcipherWalk::next(walk).

`SkcipherWalk::copy_iv(walk) -> Result<(), i32>`:
1. aligned_stride = align_up(walk.stride, walk.alignmask + 1).
2. size = aligned_stride + walk.ivsize + (walk.alignmask & !(crypto_tfm_ctx_alignment() - 1)).
3. walk.buffer = kmalloc(size, SkcipherWalk::gfp(walk)).ok_or(-ENOMEM)?.
4. iv_dst = align_up_ptr(walk.buffer, walk.alignmask + 1) + aligned_stride.
5. memcpy(iv_dst, walk.iv, walk.ivsize).
6. walk.iv = iv_dst.
7. Ok(()).

`SkcipherWalk::next(walk) -> i32`:
1. n = walk.total.
2. bsize = min(walk.stride, max(n, walk.blocksize)).
3. n = ScatterWalk::clamp(&walk.in_, n); n = ScatterWalk::clamp(&walk.out, n).
4. if n < bsize:
   - if walk.total < walk.blocksize: return SkcipherWalk::done(walk, -EINVAL).
   - return SkcipherWalk::next_slow(walk, bsize).
5. walk.nbytes = n.
6. if (walk.in_.offset | walk.out.offset) & walk.alignmask != 0:
   - if walk.page.is_null():
     - walk.page = __get_free_page(SkcipherWalk::gfp(walk));
     - if walk.page.is_null(): return SkcipherWalk::next_slow(walk, bsize).
   - walk.flags |= COPY;
   - return SkcipherWalk::next_copy(walk).
7. SkcipherWalk::next_fast(walk).

`SkcipherWalk::next_fast(walk)`:
1. diff = offset_in_page(walk.in_.offset) - offset_in_page(walk.out.offset).
2. diff |= (sg_page(walk.in_.sg) + (walk.in_.offset >> PAGE_SHIFT) as ptr) - (sg_page(walk.out.sg) + (walk.out.offset >> PAGE_SHIFT) as ptr).
3. ScatterWalk::map(&walk.out).
4. walk.in_.addr = walk.out.addr.
5. if diff != 0:
   - walk.flags |= DIFF;
   - ScatterWalk::map(&walk.in_).
6. 0.

`SkcipherWalk::next_copy(walk)`:
1. tmp = walk.page.
2. ScatterWalk::map(&walk.in_); memcpy(tmp, walk.in_.addr, walk.nbytes); ScatterWalk::unmap(&walk.in_).
3. walk.in_.addr = walk.out.addr = tmp.
4. 0.

`SkcipherWalk::next_slow(walk, bsize)`:
1. if walk.buffer.is_null(): walk.buffer = walk.page.
2. buffer = walk.buffer.
3. if buffer.is_null():
   - n = bsize + (walk.alignmask & !(crypto_tfm_ctx_alignment() - 1));
   - buffer = kzalloc(n, SkcipherWalk::gfp(walk));
   - if buffer.is_null(): return SkcipherWalk::done(walk, -ENOMEM);
   - walk.buffer = buffer.
4. buffer = align_up_ptr(buffer, walk.alignmask + 1).
5. memcpy_from_scatterwalk(buffer, &walk.in_, bsize).
6. walk.out.addr = buffer; walk.in_.addr = walk.out.addr.
7. walk.nbytes = bsize; walk.flags |= SLOW.
8. 0.

`SkcipherWalk::done(walk, res) -> i32`:
1. n = walk.nbytes; total = 0.
2. if n == 0: goto finish.
3. if res >= 0: n -= res; total = walk.total - n.
4. if walk.flags & (SLOW | COPY | DIFF) == 0:
   - ScatterWalk::advance(&walk.in_, n) /* fast in-place */.
5. else if walk.flags & DIFF != 0:
   - ScatterWalk::done_src(&walk.in_, n).
6. else if walk.flags & COPY != 0:
   - ScatterWalk::advance(&walk.in_, n);
   - ScatterWalk::map(&walk.out); memcpy(walk.out.addr, walk.page, n).
7. else /* SLOW */:
   - if res > 0: res = -EINVAL; total = 0;
     else: memcpy_to_scatterwalk(&walk.out, walk.out.addr, n).
   - goto dst_done.
8. ScatterWalk::done_dst(&walk.out, n).
9. dst_done:
10. if res > 0: res = 0.
11. walk.total = total; walk.nbytes = 0.
12. if total > 0:
    - if walk.flags & SLEEP: cond_resched();
    - walk.flags &= !(SLOW | COPY | DIFF);
    - return SkcipherWalk::next(walk).
13. finish:
14. if walk.buffer.is_null() ∧ walk.page.is_null(): return res /* fast path: nothing to free */.
15. if walk.iv != walk.oiv: memcpy(walk.oiv, walk.iv, walk.ivsize) /* IV write-back */.
16. if walk.buffer != walk.page: kfree(walk.buffer).
17. if !walk.page.is_null(): free_page(walk.page).
18. res.

### Simple-mode instance

`Skcipher::alloc_instance_simple(tmpl, tb) -> Result<*mut SkcipherInstance, i32>`:
1. mask = crypto_check_attr_type(tb, CRYPTO_ALG_TYPE_SKCIPHER)?.
2. inst = kzalloc(sizeof(SkcipherInstance) + sizeof(CryptoCipherSpawn), GFP_KERNEL).ok_or(-ENOMEM)?.
3. spawn = inst.ctx() as *mut CryptoCipherSpawn.
4. crypto_grab_cipher(spawn, inst.to_crypto(), tb[1].alg_name(), 0, mask).map_err(|e| { kfree(inst); e })?.
5. cipher_alg = spawn.cipher_alg().
6. crypto_inst_setname(inst.to_crypto(), tmpl.name, cipher_alg)?.
7. inst.free = Skcipher::free_instance_simple.
8. inst.alg.base.cra_blocksize = cipher_alg.cra_blocksize.
9. inst.alg.base.cra_alignmask = cipher_alg.cra_alignmask.
10. inst.alg.base.cra_priority = cipher_alg.cra_priority.
11. inst.alg.min_keysize = cipher_alg.cra_cipher.cia_min_keysize.
12. inst.alg.max_keysize = cipher_alg.cra_cipher.cia_max_keysize.
13. inst.alg.ivsize = cipher_alg.cra_blocksize.
14. inst.alg.base.cra_ctxsize = size_of::<SkcipherCtxSimple>().
15. inst.alg.setkey = Skcipher::setkey_simple.
16. inst.alg.init = Skcipher::init_tfm_simple.
17. inst.alg.exit = Skcipher::exit_tfm_simple.
18. Ok(inst).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `setkey_keysize_bounded` | INVARIANT | per-setkey (native): keylen ∈ [min_keysize, max_keysize] else -EINVAL. |
| `setkey_misalign_routed` | INVARIANT | per-setkey: (key & alignmask) != 0 ⟹ setkey_unaligned path with kfree_sensitive. |
| `setkey_failure_sets_needkey` | INVARIANT | per-setkey: err != 0 ⟹ NEED_KEY remains/becomes set (via skcipher_set_needkey). |
| `setkey_success_clears_needkey` | INVARIANT | per-setkey: err == 0 ⟹ NEED_KEY cleared. |
| `encrypt_decrypt_needkey_gate` | INVARIANT | per-encrypt/decrypt: NEED_KEY set ⟹ -ENOKEY. |
| `lskcipher_dispatch` | INVARIANT | per-encrypt/decrypt/setkey/export/import: cra_type != crypto_skcipher_type ⟹ lskcipher path. |
| `init_tfm_sets_needkey` | INVARIANT | per-init_tfm: skcipher_set_needkey called; for lskcipher backend, reqsize computed from ivsize + statesize + align padding. |
| `init_tfm_installs_exit_iff_alg_exit` | INVARIANT | per-init_tfm (native): base.exit = exit_tfm iff alg.exit != NULL. |
| `walk_first_rejects_hardirq` | INVARIANT | per-skcipher_walk_first: in_hardirq() ⟹ -EDEADLK (WARN). |
| `walk_iv_alignment` | INVARIANT | per-walk_first: misaligned iv ⟹ copy_iv with kmalloc'd aligned shadow. |
| `walk_next_path_choice` | INVARIANT | per-walk_next: (in.offset \| out.offset) & alignmask == 0 ∧ n ≥ bsize ⟹ FAST; misaligned ⟹ COPY (page bounce); n < bsize ⟹ SLOW (heap aligned). |
| `walk_done_iv_writeback` | INVARIANT | per-walk_done finish: walk.iv != walk.oiv ⟹ memcpy(oiv, iv, ivsize). |
| `walk_done_frees_buffer_page` | INVARIANT | per-walk_done finish: buffer != page ⟹ kfree(buffer); page ⟹ free_page(page). |
| `walk_done_slow_partial_is_einval` | INVARIANT | per-walk_done SLOW: res > 0 ⟹ res = -EINVAL, total = 0. |
| `walk_done_cond_resched_under_sleep` | INVARIANT | per-walk_done: SKCIPHER_WALK_SLEEP ∧ total > 0 ⟹ cond_resched before next. |
| `prepare_alg_size_bounds` | INVARIANT | per-prepare_alg: ivsize/chunksize ≤ PAGE_SIZE/8; statesize ≤ PAGE_SIZE/2; ivsize+statesize ≤ PAGE_SIZE/2; walksize ≤ PAGE_SIZE/8. |
| `prepare_alg_chunksize_default` | INVARIANT | per-prepare_alg: chunksize == 0 ⟹ chunksize = cra_blocksize. |
| `prepare_alg_walksize_default` | INVARIANT | per-prepare_alg: walksize == 0 ⟹ walksize = chunksize. |
| `prepare_alg_state_consistency` | INVARIANT | per-prepare_alg: statesize == 0 ⟹ noimport/noexport; statesize > 0 ⟹ both import ∧ export non-NULL. |
| `register_instance_requires_free` | INVARIANT | per-skcipher_register_instance: inst.free == NULL ⟹ -EINVAL (WARN). |
| `register_many_rollback` | INVARIANT | per-crypto_register_skciphers: failure at i ⟹ entries [0..i) unregistered. |
| `sync_skcipher_reqsize_bound` | INVARIANT | per-alloc_sync_skcipher: reqsize ≤ MAX_SYNC_SKCIPHER_REQSIZE. |

### Layer 2: TLA+

`crypto/skcipher.tla`:
- States: Tfm { UNINIT, NEED_KEY, KEY_LOADED, OPERATING, EXIT_PENDING, FREED }.
- States: Request { ALLOC, CONFIGURED, IN_FLIGHT, COMPLETE_OK, COMPLETE_ERR }.
- States: Walk { INIT, STEP_FAST, STEP_COPY, STEP_SLOW, ADVANCING, DONE }.
- Actions: alloc, setkey, encrypt, decrypt, walk_first, walk_next (path: fast/copy/slow), walk_done (advance or finish), iv_writeback, free.
- Properties:
  - `safety_no_op_without_key` — per-encrypt/decrypt: pre(NEED_KEY) ⟹ post(-ENOKEY).
  - `safety_setkey_range` — per-setkey: keylen ∈ [min, max] else -EINVAL (native path).
  - `safety_walk_first_not_hardirq` — per-walk_first: pre(in_hardirq) ⟹ post(-EDEADLK).
  - `safety_walk_iv_writeback_on_finish` — per-walk_done with total==0 ∧ walk.iv != walk.oiv ⟹ oiv updated.
  - `safety_walk_resource_cleanup` — per-walk_done finish: all of {buffer, page} freed exactly once.
  - `safety_walk_slow_partial_einval` — per-walk_done SLOW with res > 0 ⟹ -EINVAL.
  - `liveness_walk_eventually_terminates` — per-walk: total strictly decreases each step; eventually total == 0 ∨ error.
  - `safety_register_many_atomic` — per-register_skciphers: all-or-rollback.
  - `safety_lskcipher_dispatch_consistent` — per-encrypt/decrypt/setkey: cra_type check matches dispatch target.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Skcipher::setkey` post: err == 0 ↔ !NEED_KEY; native path range-checked | `Skcipher::setkey` |
| `Skcipher::encrypt` post: !NEED_KEY ⟹ dispatch to alg.encrypt (native) or lskcipher.encrypt_sg | `Skcipher::encrypt` |
| `Skcipher::decrypt` post: !NEED_KEY ⟹ dispatch to alg.decrypt or lskcipher.decrypt_sg | `Skcipher::decrypt` |
| `Skcipher::init_tfm` post: NEED_KEY set; reqsize set (native = alg_reqsize, lskcipher = align + iv + state) | `Skcipher::init_tfm` |
| `Skcipher::prepare_alg` post: cra_type = SKCIPHER_TYPE; cra_flags type bits = SKCIPHER; walksize ≤ PAGE_SIZE/8 | `Skcipher::prepare_alg` |
| `Skcipher::register_many` post: returns 0 ⟹ all registered; returns err ⟹ none registered | `Skcipher::register_many` |
| `SkcipherWalk::init_virt` post: walk.total == req.cryptlen; walk.stride ∈ {alg.walksize, alg.co.chunksize}; walk.iv == walk.oiv == req.iv | `SkcipherWalk::init_virt` |
| `SkcipherWalk::first` post: misaligned iv ⟹ walk.iv != req.iv (shadow buffer aligned to alignmask+1) | `SkcipherWalk::first` |
| `SkcipherWalk::done` post: total == 0 ⟹ walk.buffer/page freed; walk.oiv updated from walk.iv (if separate) | `SkcipherWalk::done` |
| `SkcipherWalk::done` post: total > 0 ⟹ flags cleared of {SLOW, COPY, DIFF} before next step | `SkcipherWalk::done` |

### Layer 4: Verus/Creusot functional

`Per-crypto_alloc_skcipher(name) → frontend lookup (crypto_skcipher_type) → crypto_alloc_tfm → init_tfm sets NEED_KEY + reqsize → setkey (range-check, alignment-route) ⟹ clear NEED_KEY → encrypt/decrypt dispatch (native vs lskcipher)` semantic equivalence: per-Documentation/crypto/api-skcipher.rst.

`Per-walk machinery (skcipher_walk_virt → skcipher_walk_first → loop { skcipher_walk_next selects FAST/COPY/SLOW → alg processes walk.nbytes → skcipher_walk_done advances or finishes with IV write-back + free })` semantic equivalence to upstream walk contract documented in include/crypto/internal/skcipher.h.

`Per-simple-mode template (skcipher_alloc_instance_simple → defaults from underlying cipher → skcipher_setkey_simple / init_tfm_simple / exit_tfm_simple)` parity with cbc/ecb/etc. template wrappers in crypto/cbc.c, crypto/ecb.c.

## Hardening

(Inherits row-1 features from `crypto/00-overview.md` § Hardening.)

skcipher reinforcement:

- **Per-`CRYPTO_TFM_NEED_KEY` gate on encrypt/decrypt** — defense against per-uninitialized-key use.
- **Per-`kfree_sensitive` on unaligned setkey shadow buffer** — defense against per-key-residue leak.
- **Per-keysize range-check (min ≤ keylen ≤ max) on native skcipher** — defense against per-truncated/over-long key acceptance.
- **Per-`skcipher_walk_first` `in_hardirq()` reject** — defense against per-allocator deadlock in hard-IRQ context.
- **Per-IV alignment shadow (`skcipher_copy_iv`)** — defense against per-misaligned-IV undefined behavior in asm primitives.
- **Per-SLOW path heap buffer aligned to `alignmask + 1`** — defense against per-asm SIGBUS on architectures with strict alignment.
- **Per-SLOW partial-process ⟹ -EINVAL** — defense against per-non-block-aligned final write leaking stale buffer bytes.
- **Per-walk-done IV write-back to `oiv`** — defense against per-CBC/CTR chained-call IV staleness (silent corruption).
- **Per-walk-done resource cleanup (buffer ∧ page)** — defense against per-walk memory leak.
- **Per-`cond_resched` in walk-done under `MAY_SLEEP`** — defense against per-large-request scheduler starvation.
- **Per-alg size bounds (ivsize/chunksize/walksize ≤ PAGE_SIZE/8; statesize ≤ PAGE_SIZE/2)** — defense against per-overflow / huge-allocation on alg registration.
- **Per-`skcipher_register_instance` requires `inst->free`** — defense against per-template instance leak.
- **Per-`crypto_register_skciphers` atomic rollback** — defense against per-partial-state on bulk registration failure.
- **Per-`crypto_alloc_sync_skcipher` masks ASYNC ∧ REQSIZE_LARGE + reqsize ceiling** — defense against per-on-stack `SYNC_SKCIPHER_REQUEST_ON_STACK` overflow.
- **Per-statesize consistency (statesize > 0 ⟹ both import ∧ export required)** — defense against per-partial-state-API hole.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `crypto/lskcipher.c` linear-state cipher backend (covered separately if expanded)
- `crypto/cbc.c`, `crypto/ctr.c`, `crypto/xts.c`, `crypto/ecb.c` template implementations (covered separately if expanded)
- `crypto/scatterwalk.c` low-level scatterlist helpers (covered separately if expanded)
- `crypto/algapi.c` algorithm registration core (covered in `algapi.md` Tier-3)
- `crypto/api.c` tfm allocation core (covered in `api.md` Tier-3)
- `crypto/aead.c` AEAD frontend (covered in `aead.md` Tier-3; AEAD-walk helpers summarized here for cross-cutting context)
- `crypto/af_alg.c` AF_ALG socket frontend (covered in `af-alg.md` Tier-3)
- `crypto/testmgr.c` self-tests (covered separately if expanded)
- Hardware acceleration glue (arch/x86/crypto/*, drivers/crypto/*) — arch / driver Tier-3
- dm-crypt / fscrypt consumers — out of crypto subsystem
- Implementation code
