---
title: "Tier-3: crypto/algapi.c — Crypto algorithm registration + template instantiation"
tags: ["tier-3", "crypto", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The crypto **algapi** layer is the low-level registration and instantiation surface beneath every typed front-end (skcipher, ahash, shash, aead, akcipher, kpp, rng, scomp, acomp). Per-`struct crypto_alg` is the universal descriptor (`cra_name`, `cra_driver_name`, `cra_flags`, `cra_blocksize`, `cra_alignmask`, `cra_priority`, `cra_refcnt`, `cra_type`, `cra_module`). Per-`crypto_register_alg` validates with `crypto_check_alg`, appends to global `crypto_alg_list` under `crypto_alg_sem`, allocates a `crypto_larval` for self-test if `CONFIG_CRYPTO_SELFTESTS` and not `CRYPTO_ALG_INTERNAL`, then either schedules the test (if boot tests finished) or defers (boot-time). Per-`crypto_alg_tested(name, err)` is the callback the test harness invokes — on success it flips `CRYPTO_ALG_TESTED`, calls `crypto_alg_finish_registration` (which displaces lower-priority duplicates via `crypto_remove_spawns`), broadcasts `CRYPTO_MSG_ALG_LOADED`, and completes the larval's completion. Per-`crypto_register_template(tmpl)` registers a parameterized constructor (e.g. `cbc(...)`, `hmac(...)`); per-`crypto_register_instance(tmpl, inst)` registers an instantiated alg whose lifetime is anchored to spawns the instance grabbed via `crypto_grab_spawn`. Per-`crypto_remove_spawns` walks the `cra_users` tree depth-first to evict every instance that transitively depends on a removed alg, deferring instance free to a workqueue (`crypto_destroy_instance_workfn`) since `crypto_alg_sem` cannot be held across `inst->alg.cra_type->free`. Per-FIPS mode: `crypto_check_module_sig` panics if `fips_enabled` and the module's signature is not OK. Critical for: dynamic crypto module load/unload, parameterized algorithms (mode + cipher), correct refcount + spawn-dependency teardown, FIPS-mode chain-of-trust.

This Tier-3 covers `crypto/algapi.c` (~1119 lines).

### Acceptance Criteria

- [ ] AC-1: `crypto_register_alg` on a valid alg: alg appears in `crypto_alg_list`; refcnt = 1; `CRYPTO_ALG_TESTED` cleared until self-test passes (or set immediately if `CONFIG_CRYPTO_SELFTESTS=n` ∨ `CRYPTO_ALG_INTERNAL`).
- [ ] AC-2: `crypto_register_alg` with duplicate `cra_driver_name` (non-larval): returns -EEXIST.
- [ ] AC-3: `crypto_register_alg` with `cra_alignmask + 1` not a power of 2: returns -EINVAL.
- [ ] AC-4: `crypto_register_alg` with `cra_priority < 0`: returns -EINVAL.
- [ ] AC-5: `crypto_register_alg` with `cra_flags & CRYPTO_ALG_DUP_FIRST` and no `cra_destroy`: kmemdup's caller-stack alg and installs `crypto_free_alg` as destructor.
- [ ] AC-6: `crypto_alg_tested(name, 0)`: target alg gets `CRYPTO_ALG_TESTED`; `crypto_alg_finish_registration` runs; `CRYPTO_MSG_ALG_LOADED` fires; larval `complete_all`.
- [ ] AC-7: `crypto_alg_tested(name, -ECANCELED)`: target gets `CRYPTO_ALG_FIPS_INTERNAL`; still `TESTED`; finish_registration runs.
- [ ] AC-8: `crypto_alg_tested(name, err != 0 ∧ err != -ECANCELED)`: target stays untested; alg eventually shot via `crypto_shoot_alg`.
- [ ] AC-9: `crypto_register_template`: same template registered twice returns -EEXIST.
- [ ] AC-10: `crypto_unregister_template`: every instance moved from `tmpl->instances` to `tmpl->dead`; deferred `free_work` flushed before return.
- [ ] AC-11: `crypto_register_instance`: any spawn flagged `dead` aborts registration; refcounted module pins on remaining spawns are dropped via `crypto_mod_put`.
- [ ] AC-12: `crypto_grab_spawn` against moribund alg: returns -EAGAIN; module pin dropped.
- [ ] AC-13: `crypto_drop_spawn` for unregistered instance: drops module pin (`crypto_mod_put`) since registration never happened.
- [ ] AC-14: `crypto_remove_spawns(alg, list, nalg)`: every transitive instance with `(spawn->alg->cra_flags ^ nalg->cra_flags) & spawn->mask == 0` is marked dead; nalg's own deps spared.
- [ ] AC-15: `fips_enabled` ∧ unsigned module attempts `crypto_register_alg`: `crypto_check_module_sig` panics.

### Architecture

```
struct CryptoAlg {
  cra_list:         ListHead,
  cra_users:        ListHead,            // -> spawn.list of dependents
  cra_refcnt:       Refcount,
  cra_flags:        u32,                 // TYPE_* | DEAD | TESTED | INTERNAL | INSTANCE | FIPS_INTERNAL | DUP_FIRST
  cra_blocksize:    u32,
  cra_alignmask:    u32,                 // 2^k - 1
  cra_priority:     i32,                 // >= 0
  cra_ctxsize:      u32,
  cra_name:         [u8; CRYPTO_MAX_ALG_NAME],
  cra_driver_name:  [u8; CRYPTO_MAX_ALG_NAME],
  cra_type:         Option<*CryptoType>,
  cra_module:       Option<*Module>,
  cra_destroy:      Option<fn(*CryptoAlg)>,
}

struct CryptoTemplate {
  list:        ListHead,                  // in CRYPTO_TEMPLATE_LIST
  instances:   HListHead,                 // live instances
  dead:        HListHead,                 // pending free
  free_work:   WorkStruct,                // -> CryptoInstance::destroy_workfn
  module:      *Module,
  name:        &'static str,
  create:      fn(*CryptoTemplate, **RtAttr) -> Result<()>,  // populates instance + spawns
  // type-specific helpers (alloc_skcipher, alloc_shash, ...)
}

struct CryptoInstance {
  alg:      CryptoAlg,                    // embedded
  list:     HListNode,                    // in tmpl.instances / tmpl.dead
  tmpl:     *CryptoTemplate,
  spawns:   Option<*CryptoSpawn>,         // singly-linked via spawn.next
}

struct CryptoSpawn {
  list:        ListHead,                   // in spawn.alg.cra_users
  alg:         *CryptoAlg,                 // dependency target
  inst:        *CryptoInstance,            // dependent
  next:        Option<*CryptoSpawn>,
  frontend:    *CryptoType,
  mask:        u32,
  dead:        bool,
  registered:  bool,
}

struct CryptoLarval {
  alg:           CryptoAlg,                // embedded with cra_flags|=LARVAL
  adult:         Option<*CryptoAlg>,
  completion:    Completion,
  test_started:  bool,
}

struct CryptoType {
  ctxsize:      fn(*CryptoAlg, u32, u32) -> u32,
  extsize:      fn(*CryptoAlg) -> u32,
  init_tfm:     fn(*CryptoTfm) -> Result<()>,
  exit_tfm:     fn(*CryptoTfm),
  show:         fn(*SeqFile, *CryptoAlg),
  report:       fn(*SkBuff, *CryptoAlg) -> Result<()>,
  free:         fn(*CryptoInstance),
  type:         u32,
  maskclear:    u32,
  maskset:      u32,
  algsize:      u32,
  tfmsize:      u32,
}

struct CryptoQueue {
  list:        ListHead,
  backlog:     *ListHead,
  qlen:        u32,
  max_qlen:    u32,
}
```

`CryptoAlg::register(alg) -> Result<()>`:
1. alg.cra_flags &= ~CRYPTO_ALG_DEAD.
2. CryptoAlg::check(alg)?
3. if alg.cra_flags & CRYPTO_ALG_DUP_FIRST ∧ alg.cra_destroy.is_none():
   - kmemdup full (algsize + sizeof) into p; alg = (p + algsize); alg.cra_destroy = Some(CryptoAlg::free).
4. down_write(CRYPTO_ALG_SEM).
5. larval = CryptoAlg::register_inner(alg, &algs_to_put).
6. if larval is Ok(Some(l)):
   - test_started = crypto_boot_test_finished(); l.test_started = test_started.
7. up_write(CRYPTO_ALG_SEM).
8. if larval is Err(e): crypto_alg_put(alg); return Err(e).
9. if test_started: crypto_schedule_test(l).
10. else: CryptoAlg::remove_final(&algs_to_put).
11. return Ok(()).

`CryptoAlg::register_inner(alg, algs_to_put) -> Result<Option<*CryptoLarval>>`:
1. if crypto_is_dead(alg): return Err(-EAGAIN).
2. INIT_LIST_HEAD(alg.cra_users).
3. for q in CRYPTO_ALG_LIST:
   - if q == alg: return Err(-EEXIST).
   - if crypto_is_moribund(q): continue.
   - if crypto_is_larval(q):
     - if same cra_driver_name: return Err(-EEXIST).
     - continue.
   - if q.cra_driver_name in {alg.cra_name, alg.cra_driver_name}
     ∨ q.cra_name == alg.cra_driver_name: return Err(-EEXIST).
4. larval = CryptoAlg::alloc_test_larval(alg)?  // returns None when no test needed
5. list_add(alg.cra_list, CRYPTO_ALG_LIST).
6. if larval.is_some():
   - alg.cra_flags &= ~CRYPTO_ALG_TESTED.
   - list_add(larval.alg.cra_list, CRYPTO_ALG_LIST).
7. else:
   - alg.cra_flags |= CRYPTO_ALG_TESTED.
   - CryptoAlg::finish_registration(alg, algs_to_put).
8. return Ok(larval).

`CryptoAlg::tested(name, err)`:
1. down_write(CRYPTO_ALG_SEM).
2. find q with crypto_is_larval(q) ∧ q.cra_driver_name == name; if not found: pr_err; unlock; return.
3. q.cra_flags |= CRYPTO_ALG_DEAD; alg = q.adult.
4. if crypto_is_dead(alg): goto complete.
5. match err:
   - 0: alg.cra_flags &= ~CRYPTO_ALG_FIPS_INTERNAL; alg.cra_flags |= CRYPTO_ALG_TESTED; CryptoAlg::finish_registration(alg, &list).
   - -ECANCELED: alg.cra_flags |= (CRYPTO_ALG_FIPS_INTERNAL | CRYPTO_ALG_TESTED); CryptoAlg::finish_registration(alg, &list).
   - other: goto complete (test failed).
6. complete: list_del_init(q.cra_list); complete_all(&q.completion).
7. up_write(CRYPTO_ALG_SEM).
8. crypto_alg_put(&q.alg); CryptoAlg::remove_final(&list).

`CryptoAlg::remove_spawns(alg, list, nalg)`:
1. new_type = (nalg ?: alg).cra_flags.
2. /* Seed: every direct user matching new_type by mask */
3. for spawn in alg.cra_users:
   - if (spawn.alg.cra_flags ^ new_type) & spawn.mask: skip.
   - else list_move(spawn.list, top).
4. /* DFS via stack + secondary_spawns */
5. spawns = &top.
6. do:
   - inner: pop spawn; inst = spawn.inst.
   - list_move(spawn.list, &stack).
   - spawn.dead = !spawn.registered ∨ &inst.alg != nalg.
   - if !spawn.registered ∨ &inst.alg == nalg: break.
   - BUG_ON &inst.alg == alg.
   - spawns = &inst.alg.cra_users.
   - if spawns.next == NULL: break (uninit).
7. while (spawns = CryptoAlg::more_spawns(alg, &stack, &top, &secondary_spawns)).
8. for spawn in secondary_spawns:
   - if !spawn.dead: list_move(spawn.list, &spawn.alg.cra_users).
   - elif spawn.registered: CryptoInstance::remove(spawn.inst, list).

`CryptoInstance::register(tmpl, inst) -> Result<()>`:
1. CryptoAlg::check(&inst.alg)?
2. inst.alg.cra_module = tmpl.module.
3. inst.alg.cra_flags |= CRYPTO_ALG_INSTANCE.
4. inst.alg.cra_destroy = Some(CryptoInstance::destroy).
5. down_write(CRYPTO_ALG_SEM).
6. fips_internal = 0; larval = Err(-EAGAIN).
7. for spawn from inst.spawns:
   - if spawn.dead: goto unlock.
   - spawn.inst = inst; spawn.registered = true.
   - fips_internal |= spawn.alg.cra_flags.
   - crypto_mod_put(spawn.alg).
8. inst.alg.cra_flags |= (fips_internal & CRYPTO_ALG_FIPS_INTERNAL).
9. larval = CryptoAlg::register_inner(&inst.alg, &algs_to_put).
10. if larval is Err: goto unlock.
11. elif larval.is_some(): larval.test_started = true.
12. hlist_add_head(&inst.list, &tmpl.instances); inst.tmpl = tmpl.
13. unlock: up_write(CRYPTO_ALG_SEM).
14. if larval is Err(e): return Err(e).
15. if larval.is_some(): crypto_schedule_test(larval).
16. else: CryptoAlg::remove_final(&algs_to_put).
17. return Ok(()).

`CryptoSpawn::grab(spawn, inst, name, type, mask) -> Result<()>`:
1. if inst.is_null(): WARN_ON_ONCE; return Err(-EINVAL).
2. if name is Err: return name.
3. alg = crypto_find_alg(name, spawn.frontend, type | CRYPTO_ALG_FIPS_INTERNAL, mask)?
4. down_write(CRYPTO_ALG_SEM).
5. if !crypto_is_moribund(alg):
   - list_add(&spawn.list, &alg.cra_users).
   - spawn.alg = alg; spawn.mask = mask; spawn.next = inst.spawns; inst.spawns = Some(spawn).
   - inst.alg.cra_flags |= (alg.cra_flags & CRYPTO_ALG_INHERITED_FLAGS).
   - err = 0.
6. up_write(CRYPTO_ALG_SEM).
7. if err != 0: crypto_mod_put(alg).
8. return err.

`CryptoInstance::destroy_workfn(work)`:
1. tmpl = container_of(work, CryptoTemplate, free_work).
2. local list = HList::new().
3. down_write(CRYPTO_ALG_SEM).
4. for (inst, n) in tmpl.dead:
   - if refcount_read(&inst.alg.cra_refcnt) != -1: continue.
   - hlist_del(&inst.list); hlist_add_head(&inst.list, &list).
5. up_write(CRYPTO_ALG_SEM).
6. for inst in list: CryptoInstance::free(inst)  // dispatches inst.alg.cra_type.free.

`CryptoTemplate::register(tmpl) -> Result<()>`:
1. INIT_WORK(&tmpl.free_work, CryptoInstance::destroy_workfn).
2. down_write(CRYPTO_ALG_SEM).
3. CryptoAlg::check_module_sig(tmpl.module).
4. for q in CRYPTO_TEMPLATE_LIST: if q == tmpl: err = -EEXIST; goto out.
5. list_add(&tmpl.list, CRYPTO_TEMPLATE_LIST); err = 0.
6. out: up_write(CRYPTO_ALG_SEM); return err.

`CryptoTemplate::unregister(tmpl)`:
1. down_write(CRYPTO_ALG_SEM).
2. BUG_ON list_empty(&tmpl.list); list_del_init(&tmpl.list).
3. for inst in tmpl.instances: CryptoAlg::remove(&inst.alg, &users); BUG_ON err.
4. up_write(CRYPTO_ALG_SEM).
5. for inst in tmpl.instances: BUG_ON refcount != 1; CryptoInstance::free(inst).
6. CryptoAlg::remove_final(&users).
7. flush_work(&tmpl.free_work).

### Out of Scope

- `crypto/api.c` (high-level `crypto_alloc_tfm` / `crypto_find_alg` / larval wait machinery, `crypto_chain` notifier list) — covered in `crypto/api.md` Tier-3.
- `crypto/af_alg.c` (AF_ALG socket interface) — covered in `crypto/af-alg.md` Tier-3.
- Per-frontend type modules (`crypto/skcipher.c`, `crypto/shash.c`, `crypto/ahash.c`, `crypto/aead.c`, `crypto/akcipher.c`, `crypto/kpp.c`, `crypto/rng.c`, `crypto/scompress.c`, `crypto/acompress.c`) — each gets its own Tier-3 if expanded.
- `crypto/testmgr.c` self-test harness, `crypto_schedule_test`, `crypto_boot_test_finished` — covered separately if expanded.
- `crypto/proc.c` (`/proc/crypto`) — covered separately if expanded.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct crypto_alg` | per-alg descriptor | `CryptoAlg` |
| `struct crypto_template` | per-parameterized-constructor | `CryptoTemplate` |
| `struct crypto_instance` | per-instantiated alg (embeds crypto_alg) | `CryptoInstance` |
| `struct crypto_spawn` | per-dependency edge (instance → alg) | `CryptoSpawn` |
| `struct crypto_larval` | per-untested-alg placeholder | `CryptoLarval` |
| `struct crypto_type` | per-frontend vtable | `CryptoType` |
| `crypto_alg_list` | global registered list | `CRYPTO_ALG_LIST` |
| `crypto_template_list` | global template list | `CRYPTO_TEMPLATE_LIST` |
| `crypto_alg_sem` | global rwsem | `CRYPTO_ALG_SEM` |
| `crypto_chain` | notifier chain | `CRYPTO_CHAIN` |
| `crypto_check_alg` | per-register validation | `CryptoAlg::check` |
| `crypto_check_module_sig` | per-FIPS sig check | `CryptoAlg::check_module_sig` |
| `__crypto_register_alg` | per-register inner (sem held) | `CryptoAlg::register_inner` |
| `crypto_register_alg` | per-register entry | `CryptoAlg::register` |
| `crypto_unregister_alg` | per-unregister | `CryptoAlg::unregister` |
| `crypto_register_algs` | per-array register | `CryptoAlg::register_array` |
| `crypto_unregister_algs` | per-array unregister | `CryptoAlg::unregister_array` |
| `crypto_alg_finish_registration` | per-displace lower-priority + notify | `CryptoAlg::finish_registration` |
| `crypto_alg_tested` | per-test-result callback | `CryptoAlg::tested` |
| `crypto_alloc_test_larval` | per-larval alloc | `CryptoAlg::alloc_test_larval` |
| `crypto_remove_alg` | per-list-removal | `CryptoAlg::remove` |
| `crypto_remove_spawns` | per-cra_users depth-first eviction | `CryptoAlg::remove_spawns` |
| `crypto_more_spawns` | per-DFS step | `CryptoAlg::more_spawns` |
| `crypto_remove_instance` | per-instance dead-mark | `CryptoInstance::remove` |
| `crypto_remove_final` | per-list refcount drop | `CryptoAlg::remove_final` |
| `crypto_register_template` | per-template register | `CryptoTemplate::register` |
| `crypto_unregister_template` | per-template unregister | `CryptoTemplate::unregister` |
| `crypto_register_templates` | per-array register | `CryptoTemplate::register_array` |
| `crypto_unregister_templates` | per-array unregister | `CryptoTemplate::unregister_array` |
| `crypto_lookup_template` | per-name lookup + request_module | `CryptoTemplate::lookup` |
| `__crypto_lookup_template` | per-name lookup inner | `CryptoTemplate::lookup_inner` |
| `crypto_register_instance` | per-instance register | `CryptoInstance::register` |
| `crypto_unregister_instance` | per-instance unregister | `CryptoInstance::unregister` |
| `crypto_free_instance` | per-instance free dispatch | `CryptoInstance::free` |
| `crypto_destroy_instance_workfn` | per-deferred-free work | `CryptoInstance::destroy_workfn` |
| `crypto_destroy_instance` | per-cra_destroy on instance | `CryptoInstance::destroy` |
| `crypto_grab_spawn` | per-spawn alloc (instance grabs dep) | `CryptoSpawn::grab` |
| `crypto_drop_spawn` | per-spawn release | `CryptoSpawn::drop` |
| `crypto_spawn_alg` | per-spawn resolve to alg | `CryptoSpawn::resolve_alg` |
| `crypto_spawn_tfm` | per-spawn → tfm (legacy) | `CryptoSpawn::tfm` |
| `crypto_spawn_tfm2` | per-spawn → tfm (frontend) | `CryptoSpawn::tfm2` |
| `crypto_register_notifier` / `_unregister_notifier` | per-notifier reg | `Crypto::{register,unregister}_notifier` |
| `crypto_get_attr_type` | per-rtattr parse type | `Crypto::get_attr_type` |
| `crypto_check_attr_type` | per-template type validate + inherited mask | `Crypto::check_attr_type` |
| `crypto_attr_alg_name` | per-rtattr alg-name extract | `Crypto::attr_alg_name` |
| `__crypto_inst_setname` | per-instance name `tmpl(alg)` | `CryptoInstance::set_name` |
| `crypto_init_queue` | per-request-queue init | `CryptoQueue::init` |
| `crypto_enqueue_request` | per-request enqueue | `CryptoQueue::enqueue` |
| `crypto_enqueue_request_head` | per-request enqueue head | `CryptoQueue::enqueue_head` |
| `crypto_dequeue_request` | per-request dequeue | `CryptoQueue::dequeue` |
| `crypto_inc` | big-endian counter increment (CTR) | `Crypto::inc` |
| `crypto_alg_extsize` | per-frontend ctx-size | `CryptoAlg::extsize` |
| `crypto_type_has_alg` | per-name probe | `Crypto::type_has_alg` |
| `crypto_start_tests` | per-late_initcall scan unstarted larvals | `Crypto::start_tests` |
| `crypto_algapi_init` / `_exit` | per-module init/exit | `Crypto::{init,exit}` |
| `CRYPTO_ALG_DEAD` / `_TESTED` / `_INTERNAL` / `_INSTANCE` / `_FIPS_INTERNAL` / `_DUP_FIRST` | `cra_flags` bits | shared bits |
| `CRYPTO_ALG_INHERITED_FLAGS` | per-spawn inherited mask | shared bits |
| `MAX_ALGAPI_ALIGNMASK` / `MAX_ALGAPI_BLOCKSIZE` / `MAX_CIPHER_ALIGNMASK` / `MAX_CIPHER_BLOCKSIZE` | per-validation caps | shared constants |
| `CRYPTOA_TYPE` / `CRYPTOA_ALG` | rtattr tags | shared constants |
| `CRYPTO_MAX_ALG_NAME` | name buffer cap | shared constant |
| `CRYPTO_MSG_ALG_LOADED` | notifier message | shared msg |

### compatibility contract

REQ-1: struct crypto_alg (selected fields):
- cra_list: linkage in `crypto_alg_list`.
- cra_users: head of `crypto_spawn.list` for every instance depending on this alg.
- cra_refcnt: `refcount_t`, initialized 1 by `crypto_check_alg`; `-1` is the sentinel for "destroy scheduled".
- cra_flags: bitset; `CRYPTO_ALG_TYPE_MASK` selects type; `CRYPTO_ALG_DEAD` marks unregistration; `CRYPTO_ALG_TESTED` set after self-test passes; `CRYPTO_ALG_INTERNAL` opts out of self-test; `CRYPTO_ALG_INSTANCE` flags template-created algs; `CRYPTO_ALG_FIPS_INTERNAL` indicates FIPS-only-usable; `CRYPTO_ALG_DUP_FIRST` requests `kmemdup` on register.
- cra_blocksize / cra_alignmask: capped at `MAX_ALGAPI_BLOCKSIZE` / `MAX_ALGAPI_ALIGNMASK`; for type==CIPHER capped tighter at `MAX_CIPHER_*`.
- cra_priority: signed `int`; must be ≥ 0; higher displaces lower at finish_registration.
- cra_name / cra_driver_name: `CRYPTO_MAX_ALG_NAME`-bounded strings; both must be non-empty.
- cra_type: front-end vtable (`crypto_type`); NULL only for pre-frontend (legacy CIPHER/COMPRESS) algs.
- cra_module: owning module (for `try_module_get` and FIPS sig check).
- cra_destroy: optional destructor invoked when refcount drops to zero (set to `crypto_destroy_instance` for template instances, `crypto_free_alg` for `DUP_FIRST` registrations).

REQ-2: crypto_check_alg(alg):
- `crypto_check_module_sig(alg->cra_module)`: if `fips_enabled ∧ mod ∧ !module_sig_ok(mod)`: `panic("Module ... signature verification failed in FIPS mode")`.
- if `!cra_name[0] ∨ !cra_driver_name[0]`: return -EINVAL.
- if `cra_alignmask & (cra_alignmask + 1)`: return -EINVAL (must be `2^k - 1`).
- if `cra_alignmask > MAX_ALGAPI_ALIGNMASK`: return -EINVAL.
- if `cra_blocksize > MAX_ALGAPI_BLOCKSIZE`: return -EINVAL.
- if `!cra_type ∧ (cra_flags & TYPE_MASK) == TYPE_CIPHER`:
  - if `cra_alignmask > MAX_CIPHER_ALIGNMASK`: return -EINVAL.
  - if `cra_blocksize > MAX_CIPHER_BLOCKSIZE`: return -EINVAL.
- if `cra_priority < 0`: return -EINVAL.
- `refcount_set(&cra_refcnt, 1)`.

REQ-3: crypto_register_alg(alg):
- Clear `CRYPTO_ALG_DEAD`.
- `err = crypto_check_alg(alg)`; if err: return err.
- if `cra_flags & CRYPTO_ALG_DUP_FIRST ∧ !WARN_ON_ONCE(cra_destroy)`:
  - `algsize = cra_type->algsize`; `p = (u8*)alg - algsize`.
  - `p = kmemdup(p, algsize + sizeof(*alg), GFP_KERNEL)`; if !p: return -ENOMEM.
  - `alg = (void*)(p + algsize)`; `alg->cra_destroy = crypto_free_alg`.
- `down_write(&crypto_alg_sem)`.
- `larval = __crypto_register_alg(alg, &algs_to_put)`.
- if `!IS_ERR_OR_NULL(larval)`:
  - `test_started = crypto_boot_test_finished()`; `larval->test_started = test_started`.
- `up_write(&crypto_alg_sem)`.
- if `IS_ERR(larval)`: `crypto_alg_put(alg)`; return `PTR_ERR(larval)`.
- if `test_started`: `crypto_schedule_test(larval)` (post-boot path).
- else: `crypto_remove_final(&algs_to_put)` (boot-time, deferred test).
- return 0.

REQ-4: __crypto_register_alg(alg, algs_to_put):
- Must hold `crypto_alg_sem` (write).
- if `crypto_is_dead(alg)`: err = -EAGAIN; goto err.
- `INIT_LIST_HEAD(&alg->cra_users)`.
- err = -EEXIST.
- for each q in `crypto_alg_list`:
  - if q == alg: goto err.
  - if `crypto_is_moribund(q)`: continue.
  - if `crypto_is_larval(q)`:
    - if `!strcmp(alg->cra_driver_name, q->cra_driver_name)`: goto err.
    - continue.
  - if `q->cra_driver_name ∈ {alg->cra_name, alg->cra_driver_name} ∨ q->cra_name == alg->cra_driver_name`: goto err.
- `larval = crypto_alloc_test_larval(alg)`; if `IS_ERR(larval)`: goto out.
- `list_add(&alg->cra_list, &crypto_alg_list)`.
- if larval:
  - `alg->cra_flags &= ~CRYPTO_ALG_TESTED` (test pending).
  - `list_add(&larval->alg.cra_list, &crypto_alg_list)`.
- else:
  - `alg->cra_flags |= CRYPTO_ALG_TESTED`.
  - `crypto_alg_finish_registration(alg, algs_to_put)`.
- return larval (NULL ⟹ no test scheduled; non-NULL ⟹ test pending; ERR_PTR ⟹ failed).

REQ-5: crypto_alg_finish_registration(alg, algs_to_put):
- Must hold `crypto_alg_sem`.
- for each q in `crypto_alg_list`:
  - skip if `q == alg` ∨ `crypto_is_moribund(q)` ∨ `crypto_is_larval(q)`.
  - if `strcmp(alg->cra_name, q->cra_name) != 0`: continue.
  - if `strcmp(alg->cra_driver_name, q->cra_driver_name) != 0 ∧ q->cra_priority > alg->cra_priority`: continue (lower-prio loses only if names differ; higher-prio incumbent stays).
  - `crypto_remove_spawns(q, algs_to_put, alg)` (evict q's users that aren't shielded by alg).
- `crypto_notify(CRYPTO_MSG_ALG_LOADED, alg)` (notifier chain).

REQ-6: crypto_alloc_test_larval(alg):
- if `!IS_ENABLED(CONFIG_CRYPTO_SELFTESTS) ∨ (cra_flags & CRYPTO_ALG_INTERNAL)`: return NULL (no test needed).
- `larval = crypto_larval_alloc(alg->cra_name, alg->cra_flags | CRYPTO_ALG_TESTED, 0)`.
- if `IS_ERR(larval)`: return larval.
- `larval->adult = crypto_mod_get(alg)`; if NULL: kfree(larval); return -ENOENT.
- `refcount_set(&larval->alg.cra_refcnt, 1)`.
- copy `cra_driver_name`, `cra_priority`.
- return larval.

REQ-7: crypto_alg_tested(name, err):
- `down_write(&crypto_alg_sem)`.
- Find larval `q` whose `cra_driver_name == name` and `crypto_is_larval(q)`.
- if not found: `pr_err("alg: Unexpected test result for %s: %d")`; unlock; return.
- `q->cra_flags |= CRYPTO_ALG_DEAD` (larval is consumed).
- `alg = test->adult`.
- if `crypto_is_dead(alg)`: goto complete.
- if `err == -ECANCELED`: `alg->cra_flags |= CRYPTO_ALG_FIPS_INTERNAL`.
- elif err: goto complete (test failed; alg remains untested + DEAD path).
- else: `alg->cra_flags &= ~CRYPTO_ALG_FIPS_INTERNAL`.
- `alg->cra_flags |= CRYPTO_ALG_TESTED`.
- `crypto_alg_finish_registration(alg, &list)`.
- complete: `list_del_init(&test->alg.cra_list)`; `complete_all(&test->completion)`.
- `up_write(&crypto_alg_sem)`.
- `crypto_alg_put(&test->alg)`; `crypto_remove_final(&list)`.

REQ-8: crypto_remove_spawns(alg, list, nalg):
- `new_type = (nalg ?: alg)->cra_flags`.
- Initial: `spawns = &alg->cra_users`.
- for each spawn in `cra_users`: if `(spawn->alg->cra_flags ^ new_type) & spawn->mask`: skip; else move to `top`.
- DFS using stack/secondary_spawns:
  - inner loop pops `spawn` from current list, sets `spawn->dead = !spawn->registered ∨ &inst->alg != nalg`.
  - if `!spawn->registered`: break (skip walking into uninitialized cra_users).
  - BUG_ON `&inst->alg == alg`.
  - if `&inst->alg == nalg`: break (don't kill nalg's own deps).
  - descend into `&inst->alg.cra_users` (skip if `next == NULL` — instance not yet fully registered).
  - resume via `crypto_more_spawns` which pops from `stack`, repaints `dead = false` for survivor chains.
- After DFS: for each spawn in `secondary_spawns`:
  - if `!spawn->dead`: move back to `spawn->alg->cra_users` (resurrected).
  - elif `spawn->registered`: `crypto_remove_instance(spawn->inst, list)` (queued for free).

REQ-9: crypto_remove_instance(inst, list):
- if `crypto_is_dead(&inst->alg)`: return.
- `inst->alg.cra_flags |= CRYPTO_ALG_DEAD`.
- if `!tmpl`: return (raw alg).
- `list_del_init(&inst->alg.cra_list)`.
- `hlist_del(&inst->list)`; `hlist_add_head(&inst->list, &tmpl->dead)`.
- BUG_ON `!list_empty(&inst->alg.cra_users)` (instance must have no further users).
- `crypto_alg_put(&inst->alg)` (drop registration's hold).

REQ-10: crypto_destroy_instance / crypto_destroy_instance_workfn:
- `crypto_destroy_instance(alg)`:
  - `inst = container_of(alg, struct crypto_instance, alg)`.
  - `refcount_set(&alg->cra_refcnt, -1)` (sentinel — "destroy in flight").
  - `schedule_work(&tmpl->free_work)` (deferred — can't free under `crypto_alg_sem`).
- `crypto_destroy_instance_workfn(w)`:
  - `down_write(&crypto_alg_sem)`.
  - for each inst in `tmpl->dead`: if `cra_refcnt == -1`: detach to local `list`.
  - `up_write(&crypto_alg_sem)`.
  - for each inst in local `list`: `crypto_free_instance(inst)` = `inst->alg.cra_type->free(inst)`.

REQ-11: crypto_register_template(tmpl):
- `INIT_WORK(&tmpl->free_work, crypto_destroy_instance_workfn)`.
- `down_write(&crypto_alg_sem)`.
- `crypto_check_module_sig(tmpl->module)`.
- duplicate check: `for q in crypto_template_list: if q == tmpl: goto out`.
- `list_add(&tmpl->list, &crypto_template_list)`; err = 0.
- `up_write(&crypto_alg_sem)`.
- return err.

REQ-12: crypto_unregister_template(tmpl):
- `down_write(&crypto_alg_sem)`.
- BUG_ON `list_empty(&tmpl->list)`.
- `list_del_init(&tmpl->list)`.
- for each inst in `tmpl->instances`: `crypto_remove_alg(&inst->alg, &users)`; BUG_ON err.
- `up_write(&crypto_alg_sem)`.
- for each inst in `tmpl->instances`: BUG_ON `refcount_read(&inst->alg.cra_refcnt) != 1`; `crypto_free_instance(inst)`.
- `crypto_remove_final(&users)`.
- `flush_work(&tmpl->free_work)`.

REQ-13: crypto_register_instance(tmpl, inst):
- `err = crypto_check_alg(&inst->alg)`; if err: return err.
- `inst->alg.cra_module = tmpl->module`.
- `inst->alg.cra_flags |= CRYPTO_ALG_INSTANCE`.
- `inst->alg.cra_destroy = crypto_destroy_instance`.
- `down_write(&crypto_alg_sem)`.
- larval = ERR_PTR(-EAGAIN).
- for `spawn = inst->spawns; spawn; spawn = next`:
  - if `spawn->dead`: goto unlock.
  - `next = spawn->next`; `spawn->inst = inst`; `spawn->registered = true`.
  - `fips_internal |= spawn->alg->cra_flags`.
  - `crypto_mod_put(spawn->alg)`.
- `inst->alg.cra_flags |= (fips_internal & CRYPTO_ALG_FIPS_INTERNAL)`.
- `larval = __crypto_register_alg(&inst->alg, &algs_to_put)`.
- if `IS_ERR(larval)`: goto unlock.
- elif larval: `larval->test_started = true`.
- `hlist_add_head(&inst->list, &tmpl->instances)`; `inst->tmpl = tmpl`.
- unlock: `up_write(&crypto_alg_sem)`.
- if `IS_ERR(larval)`: return `PTR_ERR(larval)`.
- if larval: `crypto_schedule_test(larval)`.
- else: `crypto_remove_final(&algs_to_put)`.
- return 0.

REQ-14: crypto_unregister_instance(inst):
- `down_write(&crypto_alg_sem)`.
- `crypto_remove_spawns(&inst->alg, &list, NULL)` (evict everyone depending on this instance).
- `crypto_remove_instance(inst, &list)` (mark dead, splice to tmpl->dead).
- `up_write(&crypto_alg_sem)`.
- `crypto_remove_final(&list)`.

REQ-15: crypto_grab_spawn(spawn, inst, name, type, mask):
- WARN_ON_ONCE if `inst == NULL`: return -EINVAL.
- if `IS_ERR(name)`: return `PTR_ERR(name)` (allow direct pass-through from `crypto_attr_alg_name`).
- `alg = crypto_find_alg(name, spawn->frontend, type | CRYPTO_ALG_FIPS_INTERNAL, mask)`; if `IS_ERR(alg)`: return.
- `down_write(&crypto_alg_sem)`.
- if `!crypto_is_moribund(alg)`:
  - `list_add(&spawn->list, &alg->cra_users)`.
  - `spawn->alg = alg`; `spawn->mask = mask`; `spawn->next = inst->spawns`; `inst->spawns = spawn`.
  - `inst->alg.cra_flags |= (alg->cra_flags & CRYPTO_ALG_INHERITED_FLAGS)`.
  - err = 0.
- `up_write(&crypto_alg_sem)`.
- if err: `crypto_mod_put(alg)`.

REQ-16: crypto_drop_spawn(spawn):
- if `!spawn->alg`: return (not initialized).
- `down_write(&crypto_alg_sem)`.
- if `!spawn->dead`: `list_del(&spawn->list)`.
- `up_write(&crypto_alg_sem)`.
- if `!spawn->registered`: `crypto_mod_put(spawn->alg)` (instance never registered — undo the grab's module pin).

REQ-17: crypto_spawn_alg(spawn):
- `down_read(&crypto_alg_sem)`.
- if `!spawn->dead`:
  - `alg = spawn->alg`.
  - if `!crypto_mod_get(alg)`: `target = crypto_alg_get(alg)`; `shoot = true`; `alg = ERR_PTR(-EAGAIN)`.
- `up_read(&crypto_alg_sem)`.
- if shoot: `crypto_shoot_alg(target)`; `crypto_alg_put(target)`.
- return alg (or `ERR_PTR(-EAGAIN)`).

REQ-18: crypto_spawn_tfm / crypto_spawn_tfm2:
- per-`crypto_spawn_tfm(spawn, type, mask)`:
  - `alg = crypto_spawn_alg(spawn)`; if `IS_ERR(alg)`: return cast.
  - if `(alg->cra_flags ^ type) & mask`: tfm = ERR_PTR(-EINVAL); goto out_put.
  - `tfm = __crypto_alloc_tfm(alg, type, mask)`; if `IS_ERR(tfm)`: goto out_put.
  - return tfm.
- per-`crypto_spawn_tfm2(spawn)`:
  - `alg = crypto_spawn_alg(spawn)`; if `IS_ERR(alg)`: return cast.
  - `tfm = crypto_create_tfm(alg, spawn->frontend)`; if `IS_ERR(tfm)`: goto out_put.
  - return tfm.

REQ-19: crypto_lookup_template(name):
- `try_then_request_module(__crypto_lookup_template(name), "crypto-%s", name)`.
- `__crypto_lookup_template`: read-lock `crypto_alg_sem`; iterate `crypto_template_list`; return first `q` with matching name where `crypto_tmpl_get(q)` succeeds.

REQ-20: crypto_check_attr_type(tb, type, mask_ret):
- `algt = crypto_get_attr_type(tb)`; if `IS_ERR(algt)`: return `PTR_ERR(algt)`.
- if `(algt->type ^ type) & algt->mask`: return -EINVAL (user asked for incompatible front-end).
- `*mask_ret = crypto_algt_inherited_mask(algt)`.
- return 0.

REQ-21: __crypto_inst_setname(inst, name, driver, alg):
- `snprintf(inst->alg.cra_name, CRYPTO_MAX_ALG_NAME, "%s(%s)", name, alg->cra_name)`; if `≥ CRYPTO_MAX_ALG_NAME`: return -ENAMETOOLONG.
- `snprintf(inst->alg.cra_driver_name, CRYPTO_MAX_ALG_NAME, "%s(%s)", driver, alg->cra_driver_name)`; if `≥`: return -ENAMETOOLONG.

REQ-22: crypto_queue (request queue used by ahash/akcipher backlog):
- `crypto_init_queue(queue, max_qlen)`: INIT_LIST_HEAD; backlog = &queue->list; qlen = 0; max_qlen = max_qlen.
- `crypto_enqueue_request`:
  - if `qlen ≥ max_qlen`:
    - if `!(flags & CRYPTO_TFM_REQ_MAY_BACKLOG)`: return -ENOSPC.
    - err = -EBUSY; if `backlog == &list`: `backlog = &request->list`.
  - else: err = -EINPROGRESS.
  - `qlen++`; `list_add_tail(&request->list, &queue->list)`.
- `crypto_dequeue_request`:
  - if `!qlen`: return NULL.
  - `qlen--`; if `backlog != &list`: `backlog = backlog->next`.
  - return head.

REQ-23: crypto_inc(a, size):
- `b = (__be32 *)(a + size)`.
- if `HAVE_EFFICIENT_UNALIGNED_ACCESS ∨ IS_ALIGNED(b, __alignof__(*b))`:
  - for `size ≥ 4`: `c = be32_to_cpu(*--b) + 1`; `*b = cpu_to_be32(c)`; if `c != 0`: return.
- `crypto_inc_byte(a, size)` (byte-wise tail).

REQ-24: crypto_alg_extsize(alg):
- return `cra_ctxsize + (cra_alignmask & ~(crypto_tfm_ctx_alignment() - 1))`.

REQ-25: FIPS mode invariants:
- per-`crypto_check_module_sig`: `fips_enabled ∧ mod ∧ !module_sig_ok(mod) ⟹ panic`.
- per-`crypto_grab_spawn`: search mask always OR'd with `CRYPTO_ALG_FIPS_INTERNAL` so internal-only algs are reachable inside templates.
- per-`crypto_register_instance`: instance inherits `CRYPTO_ALG_FIPS_INTERNAL` from any spawn that has it.
- per-`crypto_alg_tested(err = -ECANCELED)`: sets `CRYPTO_ALG_FIPS_INTERNAL` (test deferred to FIPS-mode-only path).

REQ-26: Locking discipline:
- `crypto_alg_sem` (rwsem) protects `crypto_alg_list`, `crypto_template_list`, every `cra_list`, every `cra_users`, every `spawn` field.
- All registration / unregistration / spawn manipulation: write-lock.
- Lookup (`__crypto_lookup_template`, `crypto_spawn_alg`): read-lock.
- Never held across `tfm->cra_type->free` (deferred via `tmpl->free_work` workqueue).
- `crypto_chain` (notifier chain): blocking; callers must not hold `crypto_alg_sem`.

REQ-27: Late-initcall flow:
- `crypto_algapi_init` (late_initcall):
  - `crypto_init_proc()`.
  - `crypto_start_tests()`: if `IS_BUILTIN(CRYPTO_ALGAPI) ∧ CRYPTO_SELFTESTS`: `set_crypto_boot_test_finished()`; loop over `crypto_alg_list` scheduling any `crypto_is_test_larval(l) ∧ !l->test_started` via `crypto_schedule_test`.
- module_exit `crypto_algapi_exit`: `crypto_exit_proc()`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `alignmask_is_2k_minus_1` | INVARIANT | per-crypto_check_alg: `cra_alignmask & (cra_alignmask + 1) == 0`. |
| `alignmask_bounded` | INVARIANT | per-crypto_check_alg: `cra_alignmask ≤ MAX_ALGAPI_ALIGNMASK`. |
| `blocksize_bounded` | INVARIANT | per-crypto_check_alg: `cra_blocksize ≤ MAX_ALGAPI_BLOCKSIZE`. |
| `cipher_subcaps_bounded` | INVARIANT | per-crypto_check_alg: type==CIPHER ⟹ `cra_alignmask ≤ MAX_CIPHER_ALIGNMASK ∧ cra_blocksize ≤ MAX_CIPHER_BLOCKSIZE`. |
| `priority_nonneg` | INVARIANT | per-crypto_check_alg: `cra_priority ≥ 0`. |
| `name_non_empty` | INVARIANT | per-crypto_check_alg: `cra_name[0] != 0 ∧ cra_driver_name[0] != 0`. |
| `refcnt_balanced_on_register_failure` | INVARIANT | per-crypto_register_alg: error path drops the +1 set by `crypto_check_alg`. |
| `sem_held_in_register_inner` | INVARIANT | per-__crypto_register_alg: `crypto_alg_sem` write-held throughout. |
| `dead_sentinel_uniqueness` | INVARIANT | per-crypto_destroy_instance: `cra_refcnt == -1` exactly once per instance lifetime. |
| `spawn_list_under_sem` | INVARIANT | per-crypto_grab_spawn / crypto_drop_spawn: `spawn.list` mutations only under write-sem. |
| `inheritance_mask` | INVARIANT | per-crypto_grab_spawn: `inst.cra_flags & INHERITED_FLAGS ⊇ alg.cra_flags & INHERITED_FLAGS`. |
| `fips_panic_unsigned` | INVARIANT | per-crypto_check_module_sig: `fips_enabled ∧ mod ∧ !module_sig_ok(mod)` ⟹ panic. |
| `queue_qlen_bounded_unless_backlog` | INVARIANT | per-crypto_enqueue_request: `qlen ≥ max_qlen ∧ !MAY_BACKLOG ⟹ -ENOSPC ∧ qlen unchanged`. |
| `crypto_inc_be32_overflow_carries` | INVARIANT | per-crypto_inc: aligned be32 path correctly carries on `c == 0` continuation. |

### Layer 2: TLA+

`crypto/algapi.tla`:
- Per-register + per-test + per-instantiate + per-spawn-grab + per-unregister + per-defer-free.
- Properties:
  - `safety_unique_driver_name` — per-CRYPTO_ALG_LIST: no two non-larval, non-moribund entries share `cra_driver_name`.
  - `safety_no_register_after_dead` — per-alg in DEAD state: subsequent `register_inner` calls return -EAGAIN.
  - `safety_spawn_keeps_alg_live` — per-spawn registered: target alg's refcount ≥ 1 until `crypto_drop_spawn` or `crypto_remove_spawns`.
  - `safety_dead_instance_no_users` — per-`crypto_remove_instance`: `cra_users` empty (BUG_ON otherwise).
  - `safety_fips_panic_on_unsigned` — per-fips_enabled: every `crypto_check_module_sig` call returns ⟹ module sig OK ∨ no module.
  - `liveness_larval_completes` — per-larval: eventually `complete_all` fires (test passes/fails/canceled).
  - `liveness_instance_freed_after_tmpl_unreg` — per-`crypto_unregister_template`: every instance in `tmpl->dead` reaches `crypto_type->free` after finite work executions; `flush_work` ensures completion before return.
  - `liveness_remove_spawns_terminates` — per-DFS in `crypto_remove_spawns`: stack monotonically progresses; finite.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `CryptoAlg::check` post: returns 0 ⟹ all validation predicates hold ∧ refcnt = 1 | `CryptoAlg::check` |
| `CryptoAlg::register` post: returns 0 ⟹ alg ∈ CRYPTO_ALG_LIST under write-sem; CRYPTO_ALG_TESTED reflects test status | `CryptoAlg::register` |
| `CryptoAlg::register_inner` post: returns Ok(None) ⟹ `CRYPTO_ALG_TESTED` set + finish_registration called; returns Ok(Some) ⟹ TESTED cleared, larval in list | `CryptoAlg::register_inner` |
| `CryptoAlg::tested(name, 0)` post: target gains `CRYPTO_ALG_TESTED`; CRYPTO_MSG_ALG_LOADED notified | `CryptoAlg::tested` |
| `CryptoAlg::tested(name, -ECANCELED)` post: target gains `CRYPTO_ALG_FIPS_INTERNAL` and `CRYPTO_ALG_TESTED` | `CryptoAlg::tested` |
| `CryptoAlg::remove_spawns` post: every surviving spawn has `dead == false` ∧ in `spawn.alg.cra_users` | `CryptoAlg::remove_spawns` |
| `CryptoInstance::register` post: returns 0 ⟹ inst ∈ tmpl.instances ∧ every spawn registered ∧ inst.alg ∈ CRYPTO_ALG_LIST | `CryptoInstance::register` |
| `CryptoInstance::register` failure post: every successfully-registered spawn before failure remains backed by `crypto_find_alg` module-pin (will be dropped on caller's unwind) | `CryptoInstance::register` |
| `CryptoTemplate::unregister` post: tmpl ∉ CRYPTO_TEMPLATE_LIST ∧ every instance freed (flush_work observed) | `CryptoTemplate::unregister` |
| `CryptoSpawn::grab` post: returns 0 ⟹ spawn ∈ alg.cra_users ∧ inst.spawns chain extended | `CryptoSpawn::grab` |
| `CryptoSpawn::drop` post: spawn ∉ alg.cra_users (if was registered) ∧ module pin released only if !registered | `CryptoSpawn::drop` |
| `CryptoQueue::enqueue` post: qlen ≤ max_qlen ∨ (qlen ≤ max_qlen + 1 ∧ flags & MAY_BACKLOG); returns one of {EINPROGRESS, EBUSY, ENOSPC} | `CryptoQueue::enqueue` |
| `Crypto::inc` post: post-state encodes pre-state + 1 (mod 2^(8*size)) in big-endian | `Crypto::inc` |

### Layer 4: Verus/Creusot functional

`Per-template request "cbc(aes)" → crypto_lookup_template("cbc") → tmpl.create → grab_spawn("aes") → crypto_register_instance → __crypto_register_alg → alloc_test_larval → crypto_schedule_test → tester runs → crypto_alg_tested(driver_name, err) → crypto_alg_finish_registration → crypto_remove_spawns(lower-prio dups) → crypto_notify(CRYPTO_MSG_ALG_LOADED)` semantic equivalence: per-`Documentation/crypto/api-intro.rst` and per-RFC 2104 (hmac) / NIST SP 800-38A (cbc) / SP 800-38D (gcm) for tests run against the registered alg.

`Per-unregister flow: crypto_unregister_alg → crypto_remove_alg (cra_list del + set DEAD) → crypto_remove_spawns (DFS evict transitive dependents) → crypto_remove_final (drop list's refcount holds) → if instance: schedule tmpl.free_work → crypto_destroy_instance_workfn → cra_type.free` semantic equivalence: refcount-balanced + sem-safety.

### hardening

(Inherits row-1 features from `crypto/00-overview.md` § Hardening.)

algapi reinforcement:

- **Per-`crypto_check_module_sig` panic on unsigned module in FIPS mode** — defense against per-FIPS chain-of-trust bypass.
- **Per-`crypto_check_alg` rejects `(alignmask & (alignmask + 1)) != 0`** — defense against per-misaligned buffer crash and per-arithmetic invariant break.
- **Per-`MAX_ALGAPI_BLOCKSIZE` / `MAX_ALGAPI_ALIGNMASK` caps + tighter `MAX_CIPHER_*` for legacy CIPHER** — defense against per-resource-exhaustion via pathological alg metadata.
- **Per-`crypto_register_alg` rejects negative priority** — defense against per-`crypto_alg_finish_registration` displacement-rule corruption.
- **Per-`crypto_register_alg` rejects duplicate `cra_driver_name`** — defense against per-namespace-collision and per-non-deterministic lookup.
- **Per-`crypto_alg_sem` write-locked across every list / spawn mutation** — defense against per-concurrent registrar racing into corrupt list state.
- **Per-`crypto_alg_sem` released before `cra_type->free`** — defense against per-deadlock when free callback recursively touches algapi (deferred via `tmpl->free_work` workqueue).
- **Per-`crypto_destroy_instance` uses `refcnt = -1` sentinel** — defense against per-double-free and per-race-with-concurrent-drop.
- **Per-`crypto_remove_spawns` DFS evicts transitive dependents** — defense against per-stale-spawn referencing a removed alg.
- **Per-`crypto_remove_spawns` resurrection of survivors via `secondary_spawns`** — defense against per-over-eager eviction of instances still backed by an equivalent alg.
- **Per-`crypto_register_instance` checks every spawn.dead** — defense against per-race-window where an instance gets registered against an alg already in the process of being removed.
- **Per-`crypto_grab_spawn` adds `CRYPTO_ALG_FIPS_INTERNAL` to search mask** — defense against per-template-instantiation skipping internal-only algs and silently picking a weaker public alg.
- **Per-`crypto_register_instance` inherits `CRYPTO_ALG_FIPS_INTERNAL`** — defense against per-FIPS-policy violation if any dep is FIPS-internal-only.
- **Per-`flush_work(&tmpl->free_work)` in `crypto_unregister_template`** — defense against per-template-module-unload while instance free still pending.
- **Per-`crypto_enqueue_request` returns -ENOSPC unless `CRYPTO_TFM_REQ_MAY_BACKLOG`** — defense against per-unbounded queue growth.

