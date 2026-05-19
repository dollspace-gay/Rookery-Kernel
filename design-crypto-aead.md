---
title: "Tier-3: crypto/aead.c — Authenticated Encryption with Associated Data API"
tags: ["tier-3", "crypto", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`crypto/aead.c` is the **Authenticated Encryption with Associated Data (AEAD)** front-end of the Linux Crypto API. AEAD bundles a symmetric cipher and a MAC into a single primitive that produces both a ciphertext and an authentication tag covering (associated_data || ciphertext); decryption either returns the plaintext or fails with -EBADMSG. Per-tfm handle: `struct crypto_aead` (with its embedded `crypto_tfm base`, an `authsize` field, and a per-algorithm reqsize). Per-algorithm vtable: `struct aead_alg` with `setkey / setauthsize / encrypt / decrypt / init / exit` and metadata `ivsize / chunksize / maxauthsize`. Per-request descriptor: `struct aead_request` (caller-allocated, sized at `crypto_aead_reqsize(tfm)`) carrying scatterlists `src` and `dst`, `cryptlen`, `assoclen`, `iv`, and a base `crypto_async_request` for async completion. The single funnel: `crypto_alloc_aead(name, type, mask)` ⟹ algorithm lookup via the `crypto_aead_type` frontend ⟹ `crypto_alloc_tfm`. Encryption/decryption: `crypto_aead_encrypt(req)` / `crypto_aead_decrypt(req)` validate NEED_KEY and authsize, then call into `alg->encrypt / decrypt`. Templates `gcm(aes)`, `ccm(aes)`, `rfc4106(gcm(aes))`, `chacha20poly1305(chacha20)`, etc. register an `aead_alg` via `crypto_register_aead` (or `aead_register_instance` for template-spawned instances). Critical for: IPsec (ESP), TLS in-kernel offload, dm-crypt authenticated mode, fscrypt's optional AEAD modes.

This Tier-3 covers `crypto/aead.c` (~315 lines).

### Acceptance Criteria

- [ ] AC-1: `crypto_alloc_aead("gcm(aes)", 0, 0)` returns a valid `crypto_aead *` when CRYPTO_GCM=y and CRYPTO_AES=y; returns ERR_PTR(-ENOENT) otherwise.
- [ ] AC-2: `crypto_alloc_sync_aead("aead-name-with-async-impl", 0, 0)` skips async implementations (mask ASYNC).
- [ ] AC-3: `crypto_alloc_sync_aead` WARNs and returns -EINVAL when the chosen impl has `reqsize > MAX_SYNC_AEAD_REQSIZE`.
- [ ] AC-4: `crypto_aead_setkey` with key misaligned to `crypto_aead_alignmask(tfm)` routes through `setkey_unaligned` and the shadow buffer is `kfree_sensitive`'d.
- [ ] AC-5: `crypto_aead_setkey` failure leaves `CRYPTO_TFM_NEED_KEY` set; success clears it.
- [ ] AC-6: `crypto_aead_setauthsize(tfm, 0)` returns -EINVAL when `maxauthsize > 0`; `crypto_aead_setauthsize(tfm, maxauthsize + 1)` returns -EINVAL.
- [ ] AC-7: `crypto_aead_encrypt(req)` returns -ENOKEY when `CRYPTO_TFM_NEED_KEY` is set.
- [ ] AC-8: `crypto_aead_decrypt(req)` returns -ENOKEY before checking cryptlen, and returns -EINVAL when `cryptlen < authsize`.
- [ ] AC-9: `crypto_aead_decrypt(req)` returns -EBADMSG (propagated from `alg->decrypt`) on tag mismatch; plaintext buffer must not be released to caller in success state.
- [ ] AC-10: `crypto_aead_init_tfm` sets `CRYPTO_TFM_NEED_KEY`, sets `authsize = alg->maxauthsize`, installs `exit` only when `alg->exit != NULL`, and runs `alg->init` when present.
- [ ] AC-11: `aead_prepare_alg` rejects algorithms with `max(maxauthsize, ivsize, chunksize) > PAGE_SIZE/8`.
- [ ] AC-12: `aead_register_instance` WARNs and returns -EINVAL when `inst->free == NULL`.
- [ ] AC-13: `crypto_register_aeads` rolls back fully on partial failure (unregistering already-registered entries).
- [ ] AC-14: `/proc/crypto` entry for a registered AEAD reports type=aead with correct `blocksize`, `ivsize`, `maxauthsize`.
- [ ] AC-15: CRYPTO_USER `CRYPTO_MSG_GETALG` on an AEAD returns `CRYPTOCFGA_REPORT_AEAD` with `type="aead"`, `geniv="<none>"`, `blocksize`, `maxauthsize`, `ivsize`.

### Architecture

```
struct CryptoAead {
  base: CryptoTfm,                       // template, alg pointer, flags, ctxbuf
  authsize: u32,                          // current tag length
  reqsize: u32,                           // per-request tail bytes
}

struct AeadAlg {
  setkey: Option<fn(&CryptoAead, &[u8]) -> i32>,
  setauthsize: Option<fn(&CryptoAead, u32) -> i32>,
  encrypt: fn(&mut AeadRequest) -> i32,
  decrypt: fn(&mut AeadRequest) -> i32,
  init: Option<fn(&CryptoAead) -> i32>,
  exit: Option<fn(&CryptoAead)>,
  ivsize: u32,
  maxauthsize: u32,
  chunksize: u32,
  base: CryptoAlg,
}

struct AeadRequest {
  base: CryptoAsyncRequest,               // tfm, complete, data, flags
  assoclen: u32,
  cryptlen: u32,
  iv: *mut u8,                            // length = crypto_aead_ivsize(tfm)
  src: *const ScatterList,
  dst: *mut ScatterList,
  // trailing bytes of size crypto_aead_reqsize(tfm) for alg-private context
}
```

`Aead::TYPE: CryptoType`:
```
CryptoType {
  extsize: crypto_alg_extsize,
  init_tfm: Aead::init_tfm,
  free: Aead::free_instance,
  show: Aead::show,            // CONFIG_PROC_FS
  report: Aead::report,        // CONFIG_CRYPTO_USER
  maskclear: !CRYPTO_ALG_TYPE_MASK,
  maskset: CRYPTO_ALG_TYPE_MASK,
  type: CRYPTO_ALG_TYPE_AEAD,
  tfmsize: offset_of!(CryptoAead, base),
  algsize: offset_of!(AeadAlg, base),
}
```

`Aead::setkey(tfm, key) -> Result<(), i32>`:
1. alignmask = tfm.alignmask().
2. err = if (key.as_ptr() as usize) & alignmask != 0:
   - Aead::setkey_unaligned(tfm, key).
   } else:
   - (tfm.alg().setkey)(tfm, key).
3. if err.is_err():
   - tfm.set_flag(CRYPTO_TFM_NEED_KEY).
   - return err.
4. tfm.clear_flag(CRYPTO_TFM_NEED_KEY).
5. Ok(()).

`Aead::setkey_unaligned(tfm, key) -> Result<(), i32>`:
1. alignmask = tfm.alignmask().
2. absize = key.len() + alignmask.
3. buffer = KVec::with_capacity(absize, GFP_ATOMIC).ok_or(-ENOMEM)?.
4. aligned = align_up_ptr(buffer.as_mut_ptr(), alignmask + 1).
5. memcpy(aligned, key.as_ptr(), key.len()).
6. ret = (tfm.alg().setkey)(tfm, slice::from_raw_parts(aligned, key.len())).
7. kfree_sensitive(buffer) /* drop with explicit zeroize */.
8. ret.

`Aead::setauthsize(tfm, authsize) -> Result<(), i32>`:
1. if (authsize == 0 ∧ tfm.maxauthsize() != 0) ∨ authsize > tfm.maxauthsize(): return Err(-EINVAL).
2. if let Some(setauthsize) = tfm.alg().setauthsize:
   - setauthsize(tfm, authsize)?.
3. tfm.authsize = authsize.
4. Ok(()).

`Aead::encrypt(req) -> i32`:
1. aead = req.reqtfm().
2. if aead.flags() & CRYPTO_TFM_NEED_KEY != 0: return -ENOKEY.
3. (aead.alg().encrypt)(req).

`Aead::decrypt(req) -> i32`:
1. aead = req.reqtfm().
2. if aead.flags() & CRYPTO_TFM_NEED_KEY != 0: return -ENOKEY.
3. if req.cryptlen < aead.authsize(): return -EINVAL.
4. (aead.alg().decrypt)(req).

`Aead::init_tfm(tfm) -> i32`:
1. aead = CryptoAead::cast(tfm).
2. alg = aead.alg().
3. aead.set_flag(CRYPTO_TFM_NEED_KEY).
4. aead.set_reqsize(tfm.alg_reqsize()).
5. aead.authsize = alg.maxauthsize.
6. if alg.exit.is_some(): aead.base.exit = Aead::exit_tfm.
7. if let Some(init) = alg.init: return init(aead).
8. 0.

`Aead::exit_tfm(tfm)`:
1. aead = CryptoAead::cast(tfm).
2. (aead.alg().exit.unwrap())(aead).

`Aead::report(skb, alg) -> i32`:
1. raead = CryptoReportAead::zeroed().
2. raead.type = "aead".
3. raead.geniv = "<none>".
4. raead.blocksize = alg.cra_blocksize.
5. raead.maxauthsize = container_of!(alg, AeadAlg, base).maxauthsize.
6. raead.ivsize = container_of!(alg, AeadAlg, base).ivsize.
7. nla_put(skb, CRYPTOCFGA_REPORT_AEAD, &raead).

`Aead::show(m, alg)`:
1. print "type         : aead".
2. print "async        : <yes|no>" based on (alg.cra_flags & CRYPTO_ALG_ASYNC).
3. print blocksize / ivsize / maxauthsize.
4. print "geniv        : <none>".

`Aead::free_instance(inst)`:
1. aead = AeadInstance::from_inst(inst).
2. (aead.free)(aead).

`Aead::grab(spawn, inst, name, type, mask) -> Result<(), i32>`:
1. spawn.base.frontend = &Aead::TYPE.
2. crypto_grab_spawn(&spawn.base, inst, name, type, mask).

`Aead::alloc(name, type, mask) -> Result<*mut CryptoAead, i32>`:
1. crypto_alloc_tfm(name, &Aead::TYPE, type, mask).

`Aead::alloc_sync(name, type, mask) -> Result<*mut CryptoSyncAead, i32>`:
1. mask |= CRYPTO_ALG_ASYNC.
2. type &= !CRYPTO_ALG_ASYNC.
3. tfm = crypto_alloc_tfm(name, &Aead::TYPE, type, mask)?.
4. if tfm.reqsize() > MAX_SYNC_AEAD_REQSIZE:
   - WARN_ONCE.
   - Aead::free(tfm).
   - return Err(-EINVAL).
5. Ok(tfm as *mut CryptoSyncAead).

`Aead::has(name, type, mask) -> bool`:
1. crypto_type_has_alg(name, &Aead::TYPE, type, mask) != 0.

`Aead::prepare_alg(alg) -> Result<(), i32>`:
1. if max3(alg.maxauthsize, alg.ivsize, alg.chunksize) > PAGE_SIZE / 8: return Err(-EINVAL).
2. if alg.chunksize == 0: alg.chunksize = alg.base.cra_blocksize.
3. alg.base.cra_type = &Aead::TYPE.
4. alg.base.cra_flags &= !CRYPTO_ALG_TYPE_MASK.
5. alg.base.cra_flags |= CRYPTO_ALG_TYPE_AEAD.
6. Ok(()).

`Aead::register_alg(alg) -> i32`:
1. Aead::prepare_alg(alg)?.
2. crypto_register_alg(&alg.base).

`Aead::unregister_alg(alg)`:
1. crypto_unregister_alg(&alg.base).

`Aead::register_many(algs) -> i32`:
1. for (i, a) in algs.iter().enumerate():
   - if let Err(e) = Aead::register_alg(a):
     - for j in (0..i).rev(): Aead::unregister_alg(&algs[j]).
     - return e.
2. 0.

`Aead::unregister_many(algs)`:
1. for a in algs.iter().rev(): Aead::unregister_alg(a).

`Aead::register_instance(tmpl, inst) -> i32`:
1. if inst.free.is_none(): WARN_ON; return -EINVAL.
2. Aead::prepare_alg(&inst.alg)?.
3. crypto_register_instance(tmpl, AeadInstance::to_crypto(inst)).

### Out of Scope

- `crypto/gcm.c`, `crypto/ccm.c`, `crypto/chacha20poly1305.c` template implementations (covered separately if expanded)
- `crypto/algapi.c` algorithm registration core (covered in `algapi.md` Tier-3)
- `crypto/api.c` tfm allocation core (covered in `api.md` Tier-3)
- `crypto/af_alg.c` AF_ALG socket frontend (covered in `af-alg.md` Tier-3)
- `crypto/testmgr.c` self-tests (covered separately if expanded)
- `crypto/scompress.c`, `crypto/skcipher.c` other algorithm types (separate Tier-3)
- AEAD network consumers (net/ipv4/esp4.c, net/tls/*) — out of crypto subsystem
- Hardware acceleration glue (arch/x86/crypto/aesni-intel_*) — covered in arch-specific Tier-3
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct crypto_aead` | per-tfm handle (base + authsize + reqsize) | `CryptoAead` |
| `struct aead_alg` | per-algorithm vtable + metadata | `AeadAlg` |
| `struct aead_request` | per-op descriptor (sg, iv, lens) | `AeadRequest` |
| `struct aead_instance` | per-template instance | `AeadInstance` |
| `struct crypto_aead_spawn` | per-template spawn binding | `CryptoAeadSpawn` |
| `struct crypto_sync_aead` | per-sync-only handle alias | `CryptoSyncAead` |
| `crypto_aead_type` | per-frontend descriptor (init_tfm / free / show / report) | `Aead::TYPE` |
| `crypto_alloc_aead()` | per-tfm allocation | `Aead::alloc` |
| `crypto_alloc_sync_aead()` | per-sync-only allocation (mask ASYNC, bound reqsize) | `Aead::alloc_sync` |
| `crypto_has_aead()` | per-availability probe | `Aead::has` |
| `crypto_free_aead()` (inline in aead.h) | per-tfm release | `Aead::free` |
| `crypto_aead_setkey()` | per-tfm key install (alignment-aware) | `Aead::setkey` |
| `setkey_unaligned()` | per-tfm misaligned-key shadow buffer | `Aead::setkey_unaligned` |
| `crypto_aead_setauthsize()` | per-tfm tag-length install | `Aead::setauthsize` |
| `crypto_aead_encrypt()` | per-req encrypt entry | `Aead::encrypt` |
| `crypto_aead_decrypt()` | per-req decrypt entry | `Aead::decrypt` |
| `crypto_aead_init_tfm()` | per-tfm init hook | `Aead::init_tfm` |
| `crypto_aead_exit_tfm()` | per-tfm exit hook | `Aead::exit_tfm` |
| `crypto_aead_show()` | per-/proc/crypto print | `Aead::show` |
| `crypto_aead_report()` | per-CRYPTO_USER netlink report | `Aead::report` |
| `crypto_aead_free_instance()` | per-instance free | `Aead::free_instance` |
| `crypto_grab_aead()` | per-template spawn-by-name | `Aead::grab` |
| `aead_prepare_alg()` | per-alg pre-register validation | `Aead::prepare_alg` |
| `crypto_register_aead()` | per-alg register | `Aead::register_alg` |
| `crypto_unregister_aead()` | per-alg unregister | `Aead::unregister_alg` |
| `crypto_register_aeads()` / `crypto_unregister_aeads()` | bulk register/unregister | `Aead::register_many` / `unregister_many` |
| `aead_register_instance()` | per-template-instance register | `Aead::register_instance` |
| `CRYPTO_ALG_TYPE_AEAD` | per-cra_flags type bit | shared |
| `MAX_SYNC_AEAD_REQSIZE` | per-sync-stack reqsize ceiling | shared |
| `CRYPTOCFGA_REPORT_AEAD` | netlink report attribute | shared |

### compatibility contract

REQ-1: struct crypto_aead:
- `base`: embedded `struct crypto_tfm` (template, alg pointer, flags, refcount, ctxbuf via `__crt_ctx`).
- `authsize`: current tag length (≤ `alg->maxauthsize`); defaulted to `maxauthsize` in init.
- `reqsize`: per-tfm tail bytes appended after `aead_request` for algorithm-private state.

REQ-2: struct aead_alg:
- `setkey(tfm, key, keylen) -> int` (optional; some algs derive key internally).
- `setauthsize(tfm, authsize) -> int` (optional; default accepts any size up to maxauthsize).
- `encrypt(req) -> int`: returns 0 (sync done), -EINPROGRESS / -EBUSY (async).
- `decrypt(req) -> int`: returns 0 / -EINPROGRESS / -EBADMSG (tag mismatch) / -EBUSY.
- `init(tfm) -> int` (optional).
- `exit(tfm)` (optional; installed at base.exit only if non-NULL).
- `ivsize`: per-op IV byte count.
- `maxauthsize`: per-alg maximum tag length.
- `chunksize`: per-alg natural processing block (defaulted to `cra_blocksize` if zero).
- `base`: embedded `struct crypto_alg`.

REQ-3: struct aead_request:
- `base`: `struct crypto_async_request` (tfm, complete, data, flags).
- `assoclen`: byte length of associated data prefix in `src` / `dst`.
- `cryptlen`: byte length of plaintext (encrypt) or ciphertext+tag (decrypt).
- `iv`: pointer to IV buffer of length `crypto_aead_ivsize(tfm)`.
- `src`, `dst`: scatterlists — must contain assoclen || cryptlen bytes; for decrypt cryptlen includes authsize.
- Trailing bytes: algorithm-private context sized by `crypto_aead_reqsize(tfm)`.

REQ-4: setkey_unaligned(tfm, key, keylen):
- alignmask = `crypto_aead_alignmask(tfm)`.
- absize = keylen + alignmask.
- buffer = `kmalloc(absize, GFP_ATOMIC)`; -ENOMEM on fail.
- alignbuffer = `ALIGN(buffer, alignmask + 1)`.
- memcpy(alignbuffer, key, keylen).
- ret = `crypto_aead_alg(tfm)->setkey(tfm, alignbuffer, keylen)`.
- `kfree_sensitive(buffer)` /* zeroes before free */.
- return ret.

REQ-5: crypto_aead_setkey(tfm, key, keylen):
- alignmask = `crypto_aead_alignmask(tfm)`.
- if ((unsigned long)key & alignmask): err = `setkey_unaligned(tfm, key, keylen)`.
- else: err = `crypto_aead_alg(tfm)->setkey(tfm, key, keylen)`.
- if `unlikely(err)`: `crypto_aead_set_flags(tfm, CRYPTO_TFM_NEED_KEY)`; return err.
- `crypto_aead_clear_flags(tfm, CRYPTO_TFM_NEED_KEY)`; return 0.

REQ-6: crypto_aead_setauthsize(tfm, authsize):
- if ((!authsize ∧ `crypto_aead_maxauthsize(tfm)`) ∨ authsize > `crypto_aead_maxauthsize(tfm)`): -EINVAL.
- if `crypto_aead_alg(tfm)->setauthsize`: err = `alg->setauthsize(tfm, authsize)`; if err: return err.
- tfm.authsize = authsize; return 0.

REQ-7: crypto_aead_encrypt(req):
- aead = `crypto_aead_reqtfm(req)`.
- if `crypto_aead_get_flags(aead) & CRYPTO_TFM_NEED_KEY`: return -ENOKEY.
- return `crypto_aead_alg(aead)->encrypt(req)`.

REQ-8: crypto_aead_decrypt(req):
- aead = `crypto_aead_reqtfm(req)`.
- if `crypto_aead_get_flags(aead) & CRYPTO_TFM_NEED_KEY`: return -ENOKEY.
- if `req->cryptlen < crypto_aead_authsize(aead)`: return -EINVAL /* no room for tag */.
- return `crypto_aead_alg(aead)->decrypt(req)`.

REQ-9: crypto_aead_init_tfm(tfm):
- aead = `__crypto_aead_cast(tfm)`.
- alg = `crypto_aead_alg(aead)`.
- `crypto_aead_set_flags(aead, CRYPTO_TFM_NEED_KEY)` /* require explicit setkey */.
- `crypto_aead_set_reqsize(aead, crypto_tfm_alg_reqsize(tfm))`.
- aead.authsize = alg.maxauthsize.
- if alg.exit: `aead->base.exit = crypto_aead_exit_tfm`.
- if alg.init: return `alg->init(aead)`.
- return 0.

REQ-10: crypto_aead_exit_tfm(tfm):
- aead = `__crypto_aead_cast(tfm)`.
- alg = `crypto_aead_alg(aead)`.
- `alg->exit(aead)`.

REQ-11: crypto_aead_report(skb, alg) (CONFIG_CRYPTO_USER):
- raead = zeroed `struct crypto_report_aead`.
- raead.type = "aead"; raead.geniv = "<none>".
- raead.blocksize = alg.cra_blocksize; raead.maxauthsize = aead.maxauthsize; raead.ivsize = aead.ivsize.
- `nla_put(skb, CRYPTOCFGA_REPORT_AEAD, sizeof(raead), &raead)`.

REQ-12: crypto_aead_show(m, alg) (CONFIG_PROC_FS):
- prints: type=aead, async=str_yes_no(CRYPTO_ALG_ASYNC), blocksize, ivsize, maxauthsize, geniv=<none>.

REQ-13: crypto_aead_free_instance(inst):
- aead = `aead_instance(inst)`; `aead->free(aead)` /* per-template tear-down */.

REQ-14: crypto_aead_type frontend:
```
{
  .extsize  = crypto_alg_extsize,
  .init_tfm = crypto_aead_init_tfm,
  .free     = crypto_aead_free_instance,
  .show     = crypto_aead_show,        // CONFIG_PROC_FS
  .report   = crypto_aead_report,      // CONFIG_CRYPTO_USER
  .maskclear = ~CRYPTO_ALG_TYPE_MASK,
  .maskset   =  CRYPTO_ALG_TYPE_MASK,
  .type      =  CRYPTO_ALG_TYPE_AEAD,
  .tfmsize   = offsetof(struct crypto_aead, base),
  .algsize   = offsetof(struct aead_alg, base),
}
```

REQ-15: crypto_grab_aead(spawn, inst, name, type, mask):
- spawn.base.frontend = `&crypto_aead_type`.
- return `crypto_grab_spawn(&spawn->base, inst, name, type, mask)`.

REQ-16: crypto_alloc_aead(alg_name, type, mask):
- return `crypto_alloc_tfm(alg_name, &crypto_aead_type, type, mask)`.

REQ-17: crypto_alloc_sync_aead(alg_name, type, mask):
- mask |= `CRYPTO_ALG_ASYNC` /* forbid async impls */.
- type &= ~(CRYPTO_ALG_ASYNC).
- tfm = `crypto_alloc_tfm(alg_name, &crypto_aead_type, type, mask)`.
- if !IS_ERR(tfm) ∧ `WARN_ON(crypto_aead_reqsize(tfm) > MAX_SYNC_AEAD_REQSIZE)`:
  - `crypto_free_aead(tfm)`; return ERR_PTR(-EINVAL).
- return (struct crypto_sync_aead *)tfm.

REQ-18: crypto_has_aead(alg_name, type, mask):
- return `crypto_type_has_alg(alg_name, &crypto_aead_type, type, mask)`.

REQ-19: aead_prepare_alg(alg):
- base = &alg.base.
- if `max3(alg->maxauthsize, alg->ivsize, alg->chunksize) > PAGE_SIZE / 8`: -EINVAL.
- if !alg.chunksize: alg.chunksize = base.cra_blocksize.
- base.cra_type = `&crypto_aead_type`.
- base.cra_flags &= ~CRYPTO_ALG_TYPE_MASK.
- base.cra_flags |= CRYPTO_ALG_TYPE_AEAD.
- return 0.

REQ-20: crypto_register_aead(alg):
- err = `aead_prepare_alg(alg)`; if err: return err.
- return `crypto_register_alg(&alg->base)`.

REQ-21: crypto_unregister_aead(alg):
- `crypto_unregister_alg(&alg->base)`.

REQ-22: crypto_register_aeads(algs, count):
- for i in 0..count: ret = `crypto_register_aead(&algs[i])`; if ret: rollback (unregister 0..i-1); return ret.
- return 0.

REQ-23: crypto_unregister_aeads(algs, count):
- for i = count-1 down to 0: `crypto_unregister_aead(&algs[i])`.

REQ-24: aead_register_instance(tmpl, inst):
- if `WARN_ON(!inst->free)`: return -EINVAL /* templates must supply free */.
- err = `aead_prepare_alg(&inst->alg)`; if err: return err.
- return `crypto_register_instance(tmpl, aead_crypto_instance(inst))`.

REQ-25: aead_request lifecycle:
- alloc: `aead_request_alloc(tfm, gfp)` ⟹ kmalloc of `sizeof(aead_request) + crypto_aead_reqsize(tfm)`; init base.tfm via `aead_request_set_tfm`.
- set callback: `aead_request_set_callback(req, flags, complete, data)`.
- set crypt: `aead_request_set_crypt(req, src, dst, cryptlen, iv)`.
- set ad: `aead_request_set_ad(req, assoclen)`.
- encrypt/decrypt: `crypto_aead_encrypt(req)` / `crypto_aead_decrypt(req)`.
- free: `aead_request_free(req)` ⟹ `kfree_sensitive`.

REQ-26: IV-handling contract:
- IV buffer of length `crypto_aead_ivsize(tfm)` supplied at request-set time.
- Counter-mode AEADs (GCM, ChaCha20-Poly1305) treat IV as a 12-byte nonce (with 4-byte counter prefix internally).
- CCM treats IV as L+nonce per RFC 3610.
- Caller is responsible for nonce uniqueness per (key, nonce) pair.

REQ-27: Tag placement convention:
- Encrypt: src contains [assoc || plaintext] (cryptlen bytes plain); dst contains [assoc || ciphertext || tag] (cryptlen + authsize trailing bytes written).
- Decrypt: src contains [assoc || ciphertext || tag] (cryptlen includes tag); dst contains [assoc || plaintext] (cryptlen - authsize bytes written).
- In-place: src == dst supported.

REQ-28: Template consumers (out-of-file but contract-bound):
- `gcm(<blockcipher>)` — Galois/Counter Mode (RFC 5288); ivsize=12, maxauthsize=16, requires blocksize=16.
- `ccm(<blockcipher>)` — Counter with CBC-MAC (RFC 3610); ivsize=16, maxauthsize=16, requires blocksize=16.
- `rfc4106(gcm(aes))` — IPsec ESP GCM (RFC 4106).
- `rfc4543(gcm(aes))` — IPsec ESP GMAC (RFC 4543).
- `rfc7539(chacha20,poly1305)` — ChaCha20-Poly1305 (RFC 7539/8439); ivsize=12, authsize=16.
- `rfc7539esp(chacha20,poly1305)` — IPsec variant.
- Each template registers an `aead_alg` via `aead_register_instance`.

REQ-29: Module metadata:
- `MODULE_LICENSE("GPL")`.
- `MODULE_DESCRIPTION("Authenticated Encryption with Associated Data (AEAD)")`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `setkey_misalign_routed` | INVARIANT | per-setkey: (key & alignmask) != 0 ⟹ setkey_unaligned path with kfree_sensitive on exit. |
| `setkey_failure_sets_needkey` | INVARIANT | per-setkey: err != 0 ⟹ CRYPTO_TFM_NEED_KEY remains/becomes set. |
| `setkey_success_clears_needkey` | INVARIANT | per-setkey: err == 0 ⟹ CRYPTO_TFM_NEED_KEY cleared. |
| `setauthsize_bounded` | INVARIANT | per-setauthsize: authsize ∈ (0, maxauthsize] ∨ (0 ∧ maxauthsize == 0). |
| `encrypt_decrypt_needkey_gate` | INVARIANT | per-encrypt/decrypt: NEED_KEY set ⟹ -ENOKEY. |
| `decrypt_authsize_room_check` | INVARIANT | per-decrypt: cryptlen < authsize ⟹ -EINVAL before alg->decrypt. |
| `init_tfm_sets_needkey_and_default_authsize` | INVARIANT | per-init_tfm: NEED_KEY set; authsize = maxauthsize. |
| `init_tfm_installs_exit_iff_alg_exit` | INVARIANT | per-init_tfm: base.exit = exit_tfm iff alg.exit != NULL. |
| `prepare_alg_size_bound` | INVARIANT | per-prepare_alg: max(maxauthsize, ivsize, chunksize) ≤ PAGE_SIZE/8. |
| `prepare_alg_chunksize_default` | INVARIANT | per-prepare_alg: chunksize == 0 ⟹ chunksize = cra_blocksize post. |
| `register_instance_requires_free` | INVARIANT | per-aead_register_instance: inst.free == NULL ⟹ -EINVAL (WARN). |
| `register_many_rollback` | INVARIANT | per-crypto_register_aeads: failure at i ⟹ entries [0..i) unregistered. |
| `sync_aead_reqsize_bound` | INVARIANT | per-alloc_sync_aead: reqsize ≤ MAX_SYNC_AEAD_REQSIZE (else WARN + free + -EINVAL). |
| `sync_aead_forbids_async` | INVARIANT | per-alloc_sync_aead: mask |= ASYNC; type &= ~ASYNC. |

### Layer 2: TLA+

`crypto/aead.tla`:
- States: Tfm { UNINIT, NEED_KEY, KEY_LOADED, OPERATING, EXIT_PENDING, FREED }.
- States: Request { ALLOC, CONFIGURED, IN_FLIGHT, COMPLETE_OK, COMPLETE_EBADMSG, COMPLETE_OTHER }.
- Actions: alloc, setkey, setauthsize, encrypt, decrypt, complete_async, exit, free.
- Properties:
  - `safety_no_op_without_key` — per-encrypt/decrypt: pre(NEED_KEY) ⟹ post(-ENOKEY).
  - `safety_decrypt_authsize_room` — per-decrypt: cryptlen ≥ authsize else -EINVAL.
  - `safety_setauthsize_bounded` — per-setauthsize: authsize ≤ maxauthsize ∧ (authsize > 0 ∨ maxauthsize == 0).
  - `safety_setkey_unaligned_zeroize` — per-setkey_unaligned: kfree_sensitive guarantees shadow buffer zeroed.
  - `liveness_async_eventually_completes` — per-encrypt/decrypt returning -EINPROGRESS: complete fires.
  - `safety_register_many_atomic` — per-register_aeads: all-or-rollback.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Aead::setkey` post: err == 0 ↔ !NEED_KEY; err != 0 ↔ NEED_KEY | `Aead::setkey` |
| `Aead::setauthsize` post: err == 0 ⟹ tfm.authsize == authsize | `Aead::setauthsize` |
| `Aead::encrypt` post: !NEED_KEY ⟹ return = alg.encrypt(req) | `Aead::encrypt` |
| `Aead::decrypt` post: !NEED_KEY ∧ cryptlen ≥ authsize ⟹ return = alg.decrypt(req) | `Aead::decrypt` |
| `Aead::init_tfm` post: NEED_KEY set; authsize = maxauthsize; reqsize = alg_reqsize | `Aead::init_tfm` |
| `Aead::prepare_alg` post: cra_type = AEAD_TYPE; cra_flags type bits = AEAD | `Aead::prepare_alg` |
| `Aead::register_many` post: returns 0 ⟹ all registered; returns err ⟹ none registered | `Aead::register_many` |
| `Aead::alloc_sync` post: returned tfm has reqsize ≤ MAX_SYNC_AEAD_REQSIZE | `Aead::alloc_sync` |

### Layer 4: Verus/Creusot functional

`Per-crypto_alloc_aead(name) → frontend lookup (crypto_aead_type) → crypto_alloc_tfm → init_tfm sets NEED_KEY + maxauthsize → setkey ⟹ clear NEED_KEY → encrypt/decrypt dispatch through alg vtable` semantic equivalence: per-Documentation/crypto/api-aead.rst and the AEAD chapter of Documentation/crypto/architecture.rst.

`Template instantiation (gcm/ccm/chacha20poly1305) → aead_register_instance → aead_prepare_alg validates sizes → crypto_register_instance` parity with templates in crypto/gcm.c, crypto/ccm.c, crypto/chacha20poly1305.c (out of scope for this Tier-3 but contract-bound).

### hardening

(Inherits row-1 features from `crypto/00-overview.md` § Hardening.)

AEAD reinforcement:

- **Per-`CRYPTO_TFM_NEED_KEY` gate on encrypt/decrypt** — defense against per-uninitialized-key use.
- **Per-`kfree_sensitive` on unaligned setkey shadow buffer** — defense against per-key-residue leak after kfree.
- **Per-`crypto_aead_setauthsize` bounds (≤ maxauthsize, no 0 when maxauthsize > 0)** — defense against per-truncated-tag forgery.
- **Per-`crypto_aead_decrypt` cryptlen ≥ authsize pre-check** — defense against per-OOB-read during tag extraction.
- **Per-alg `max3(maxauthsize, ivsize, chunksize) ≤ PAGE_SIZE/8`** — defense against per-overflow / huge-allocation on alg registration.
- **Per-`aead_register_instance` requires `inst->free`** — defense against per-template instance leak.
- **Per-`crypto_register_aeads` atomic rollback** — defense against per-partial-state on bulk registration failure.
- **Per-`crypto_alloc_sync_aead` masks ASYNC + reqsize ceiling** — defense against per-on-stack reqsize overflow.
- **Per-`init_tfm` defaults `authsize = maxauthsize`** — defense against per-zero-authsize default (silent no-MAC).
- **Per-`crypto_aead_alg` indirect call only through frontend-validated `cra_type`** — defense against per-type-confusion across crypto subsystems.
- **Per-`-EBADMSG` propagation from `alg->decrypt`** — defense against per-tag-mismatch silent-pass.

