# Tier-3: kernel/livepatch/core.c — Livepatch core (patch/object/func lifecycle)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/livepatch/00-overview.md
upstream-paths:
  - kernel/livepatch/core.c (~1387 lines)
  - kernel/livepatch/core.h
  - kernel/livepatch/patch.c
  - kernel/livepatch/patch.h
  - include/linux/livepatch.h
-->

## Summary

The **livepatch core** glues the user-supplied `struct klp_patch` (containing `struct klp_object` per kernel-object — `vmlinux` or a module — each holding an array of `struct klp_func` mapping `old_name`/`old_sympos` to `new_func`) into the live kernel. Per-`klp_enable_patch()` it (1) verifies the module carries the `LIVEPATCH` ELF marker via `is_livepatch_module()`, (2) creates the per-patch sysfs hierarchy under `/sys/kernel/livepatch/<patch>/`, (3) per-object resolves `old_func` addresses through `kallsyms_on_each_match_symbol()` honouring `old_sympos` for non-unique symbols, (4) writes the `.klp.rela.*` relocations via `apply_relocate_add()` to bind patch text to vmlinux/module private symbols, (5) registers ftrace handlers via `klp_patch_object()` so that calls to `old_func` are redirected through the patch's ftrace ops `func_stack` (LIFO patch-stack — newest patch wins), and (6) hands off to the consistency-model transition (`klp_init_transition()` → `klp_start_transition()` → `klp_try_complete_transition()`). Per-`klp_mutex` serialises all bookkeeping; the global `klp_patches` list holds enabled + in-transition patches. Per-`klp_module_coming/going()` hooks let late-loaded modules pick up patches that target them, and let unloading modules cleanly revert their patched state. Per-`patch->replace` mode atomically supersedes all prior patches and synthesises NOP `klp_func` entries for functions still patched by old patches but no longer in the new patch. Critical for: zero-downtime CVE remediation, ABI-compatible function replacement, ftrace-based redirection without full kernel reboot.

This Tier-3 covers `kernel/livepatch/core.c` (~1387 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct klp_patch` | per-patch root (mod, objs, replace, enabled, forced) | `KlpPatch` |
| `struct klp_object` | per-object (vmlinux or module) | `KlpObject` |
| `struct klp_func` | per-function replacement record | `KlpFunc` |
| `struct klp_state` | per-system-state cookie | `KlpState` |
| `struct klp_find_arg` | per-kallsyms-callback context | `KlpFindArg` |
| `klp_mutex` | per-global big-lock | `KLP_MUTEX` |
| `klp_patches` (LIST_HEAD) | per-global enabled-patch list | `KLP_PATCHES` |
| `klp_root_kobj` | per-sysfs `/sys/kernel/livepatch` root | `Klp::root_kobj` |
| `klp_enable_patch()` | per-public-API enable | `Klp::enable_patch` |
| `__klp_enable_patch()` | per-inner enable | `Klp::enable_patch_inner` |
| `__klp_disable_patch()` | per-inner disable | `Klp::disable_patch_inner` |
| `klp_init_patch()` | per-patch sysfs + objects | `Klp::init_patch` |
| `klp_init_patch_early()` | per-patch list/kobj init | `Klp::init_patch_early` |
| `klp_init_object()` | per-object init | `Klp::init_object` |
| `klp_init_object_loaded()` | per-object late init when module present | `Klp::init_object_loaded` |
| `klp_init_func()` | per-func sysfs | `Klp::init_func` |
| `klp_find_object_symbol()` | per-kallsyms resolve w/ sympos | `Klp::find_object_symbol` |
| `klp_resolve_symbols()` | per-`.klp.sym.*` ELF resolution | `Klp::resolve_symbols` |
| `klp_write_section_relocs()` | per-`.klp.rela.*` apply/clear | `Klp::write_section_relocs` |
| `klp_apply_section_relocs()` | per-public apply | `Klp::apply_section_relocs` |
| `klp_add_nops()` | per-replace mode NOP synthesis | `Klp::add_nops` |
| `klp_alloc_object_dynamic()` | per-dynamic-object alloc | `Klp::alloc_object_dynamic` |
| `klp_alloc_func_nop()` | per-NOP func alloc | `Klp::alloc_func_nop` |
| `klp_free_patch_start()` / `_finish()` / `_async()` | per-patch teardown | `Klp::free_patch_*` |
| `klp_free_replaced_patches_async()` | per-replace cleanup | `Klp::free_replaced_patches_async` |
| `klp_unpatch_replaced_patches()` | per-supersede unpatch | `Klp::unpatch_replaced_patches` |
| `klp_discard_nops()` | per-discard synthesised NOPs | `Klp::discard_nops` |
| `klp_module_coming()` | per-module-load hook | `Klp::module_coming` |
| `klp_module_going()` | per-module-unload hook | `Klp::module_going` |
| `klp_cleanup_module_patches_limited()` | per-module patch-revert | `Klp::cleanup_module_patches_limited` |
| `klp_find_section_by_name()` | per-ELF section lookup | `Klp::find_section_by_name` |
| `klp_is_patch_compatible()` | per-version-compat check | `Klp::is_patch_compatible` |
| `enabled_store/show` | per-sysfs `enabled` attr | `Klp::sysfs_enabled` |
| `transition_show` | per-sysfs `transition` attr | `Klp::sysfs_transition` |
| `force_store` | per-sysfs `force` attr | `Klp::sysfs_force` |
| `replace_show` | per-sysfs `replace` attr | `Klp::sysfs_replace` |
| `stack_order_show` | per-sysfs `stack_order` attr | `Klp::sysfs_stack_order` |
| `patched_show` | per-sysfs object `patched` attr | `Klp::sysfs_patched` |
| `klp_init()` | per-subsystem init (root kobj) | `Klp::init` |

## Compatibility contract

REQ-1: struct klp_patch:
- mod: per-patch backing struct module.
- objs: per-static-NULL-terminated array of klp_object.
- states: per-static klp_state array (optional, per-patch system-state).
- replace: per-bool atomic-replace mode flag.
- list: per-`klp_patches` list_head.
- kobj: per-sysfs kobject under `/sys/kernel/livepatch/`.
- obj_list: per-internal list head over static + dynamic klp_object.
- enabled: per-bool patch is currently the topmost active patch.
- forced: per-bool transition was forcibly completed (refcount held).
- free_work: per-deferred-free work_struct.
- finish: per-completion for sysfs teardown.

REQ-2: struct klp_object:
- name: per-object module name (NULL ⟹ vmlinux).
- funcs: per-static-NULL-terminated array of klp_func.
- callbacks: per-pre_patch/post_patch/pre_unpatch/post_unpatch hooks.
- kobj: per-sysfs kobject under patch directory.
- func_list: per-internal list head over static + dynamic klp_func.
- mod: per-resolved struct module (set when object's module is loaded).
- node: per-`patch->obj_list` list_head.
- patched: per-bool object is currently patched.
- dynamic: per-bool object was synthesised for `replace` mode.

REQ-3: struct klp_func:
- old_name: per-symbol-name target.
- new_func: per-replacement-function pointer.
- old_sympos: per-1-based occurrence-index for non-unique symbols (0 == 1 for unique).
- old_func: per-resolved old-function address.
- old_size / new_size: per-function-size in bytes (via `kallsyms_lookup_size_offset`).
- kobj: per-sysfs `<function,sympos>` kobject.
- stack_node: per-`klp_ops->func_stack` list_head.
- node: per-`obj->func_list` list_head.
- patched: per-bool func is on the ops stack.
- transition: per-bool consistency-model in-transition.
- nop: per-bool synthesised NOP for `replace` mode.

REQ-4: klp_enable_patch(patch):
- /* Validate */
- if !patch ∨ !patch.mod ∨ !patch.objs: return -EINVAL.
- for_each_static_object(patch, obj): if !obj.funcs: return -EINVAL.
- /* ELF marker */
- if !is_livepatch_module(patch.mod): pr_err; return -EINVAL.
- /* Subsystem ready */
- if !klp_initialized(): return -ENODEV.
- /* Reliable-stack warning (not error) */
- if !klp_have_reliable_stack(): pr_warn (transition may never complete).
- /* Serialise */
- mutex_lock(klp_mutex).
- /* Compat with existing patches */
- if !klp_is_patch_compatible(patch): mutex_unlock; return -EINVAL.
- /* Pin patch module */
- if !try_module_get(patch.mod): mutex_unlock; return -ENODEV.
- klp_init_patch_early(patch).
- ret = klp_init_patch(patch).
- if ret: goto err.
- ret = __klp_enable_patch(patch).
- if ret: goto err.
- mutex_unlock; return 0.
- err: klp_free_patch_start; mutex_unlock; klp_free_patch_finish; return ret.

REQ-5: __klp_enable_patch(patch):
- if klp_transition_patch: return -EBUSY (only one transition at a time).
- if WARN_ON(patch.enabled): return -EINVAL.
- klp_init_transition(patch, KLP_TRANSITION_PATCHED).
- smp_wmb (order func.transition vs ops->func_stack writes).
- klp_for_each_object(patch, obj):
  - if !klp_is_object_loaded(obj): continue.
  - klp_pre_patch_callback(obj); if err goto err.
  - klp_patch_object(obj) (register ftrace ops); if err goto err.
- klp_start_transition.
- patch.enabled = true.
- klp_try_complete_transition.
- return 0.
- err: pr_warn; klp_cancel_transition; return ret.

REQ-6: __klp_disable_patch(patch):
- if WARN_ON(!patch.enabled): return -EINVAL.
- if klp_transition_patch: return -EBUSY.
- klp_init_transition(patch, KLP_TRANSITION_UNPATCHED).
- klp_for_each_object: if obj.patched: klp_pre_unpatch_callback(obj).
- smp_wmb.
- klp_start_transition; patch.enabled = false; klp_try_complete_transition.
- return 0.

REQ-7: klp_init_patch_early(patch):
- INIT_LIST_HEAD(patch.list); INIT_LIST_HEAD(patch.obj_list).
- kobject_init(patch.kobj, &klp_ktype_patch).
- patch.enabled = false; patch.forced = false.
- INIT_WORK(patch.free_work, klp_free_patch_work_fn).
- init_completion(patch.finish).
- klp_for_each_object_static(patch, obj):
  - klp_init_object_early(patch, obj).
  - klp_for_each_func_static(obj, func): klp_init_func_early(obj, func).

REQ-8: klp_init_patch(patch):
- kobject_add(patch.kobj, klp_root_kobj, patch.mod.name).
- if patch.replace: klp_add_nops(patch).
- klp_for_each_object(patch, obj): klp_init_object(patch, obj).
- list_add_tail(patch.list, &klp_patches).

REQ-9: klp_init_object(patch, obj):
- if klp_is_module(obj) ∧ strlen(obj.name) >= MODULE_NAME_LEN: return -EINVAL.
- obj.patched = false; obj.mod = NULL.
- klp_find_object_module(obj) (sets obj.mod if module is `klp_alive`).
- kobject_add(obj.kobj, &patch.kobj, obj.name | "vmlinux").
- klp_for_each_func(obj, func): klp_init_func(obj, func).
- if klp_is_object_loaded(obj): klp_init_object_loaded(patch, obj).

REQ-10: klp_init_object_loaded(patch, obj):
- if klp_is_module(obj): klp_apply_object_relocs(patch, obj) — writes `.klp.rela.{module}.*` only.
- klp_for_each_func(obj, func):
  - klp_find_object_symbol(obj.name, func.old_name, func.old_sympos, &func.old_func).
  - kallsyms_lookup_size_offset(func.old_func, &func.old_size, NULL) — must succeed.
  - if func.nop: func.new_func = func.old_func.
  - kallsyms_lookup_size_offset(func.new_func, &func.new_size, NULL) — must succeed.

REQ-11: klp_find_object_symbol(objname, name, sympos, *addr):
- args = { name, addr=0, count=0, pos=sympos }.
- if objname: module_kallsyms_on_each_symbol(objname, klp_find_callback, &args).
- else: kallsyms_on_each_match_symbol(klp_match_callback, name, &args).
- if args.addr == 0: pr_err "not found"; return -EINVAL.
- if args.count > 1 ∧ sympos == 0: pr_err "unresolvable ambiguity"; return -EINVAL.
- if sympos > 0 ∧ args.count != sympos: pr_err "position not found"; return -EINVAL.
- *addr = args.addr; return 0.

REQ-12: klp_resolve_symbols(sechdrs, strtab, symndx, relasec, sec_objname):
- BUILD_BUG_ON(MODULE_NAME_LEN < 56 ∨ KSYM_NAME_LEN != 512).
- for each Elf_Rela in relasec:
  - sym = sym-table[ELF_R_SYM(rela.r_info)].
  - if sym.st_shndx != SHN_LIVEPATCH: pr_err; return -EINVAL.
  - sscanf(strtab + sym.st_name, ".klp.sym.%55[^.].%511[^,],%lu", sym_objname, sym_name, &sympos).
  - if cnt != 3: pr_err "incorrectly formatted name"; return -EINVAL.
  - sym_vmlinux = !strcmp(sym_objname, "vmlinux").
  - sec_vmlinux = !strcmp(sec_objname, "vmlinux").
  - if !sec_vmlinux ∧ sym_vmlinux: pr_err "invalid access to vmlinux symbol from module-specific rela"; return -EINVAL.
  - klp_find_object_symbol(sym_vmlinux ? NULL : sym_objname, sym_name, sympos, &addr).
  - sym.st_value = addr.

REQ-13: klp_write_section_relocs(pmod, sechdrs, ..., apply):
- sscanf section name ".klp.rela.%55[^.]" → sec_objname.
- if strcmp(objname ?: "vmlinux", sec_objname) != 0: return 0 (not for this object).
- if apply: klp_resolve_symbols then apply_relocate_add(sechdrs, strtab, symndx, secndx, pmod).
- else: clear_relocate_add(sechdrs, strtab, symndx, relsec, pmod).

REQ-14: Atomic-replace + NOPs:
- klp_add_nops(new_patch): klp_for_each_patch(old_patch): klp_for_each_object(old_patch, old_obj): klp_add_object_nops(new_patch, old_obj).
- klp_add_object_nops finds (or dynamically allocates) matching object in new patch and adds klp_func NOP entries for any old_func not present in new patch — these NOPs simply return to the original function once activated.
- klp_alloc_object_dynamic / klp_alloc_func_nop allocate the synthesised entries; obj.dynamic / func.nop set.
- After transition completes successfully and patch.replace == true: klp_unpatch_replaced_patches drops ftrace ops on old patches; klp_discard_nops removes dynamic NOPs.

REQ-15: klp_module_coming(mod):
- WARN_ON(mod.state != MODULE_STATE_COMING).
- if name == "vmlinux": pr_err; return -EINVAL.
- mutex_lock(klp_mutex).
- mod.klp_alive = true.
- klp_for_each_patch(patch): klp_for_each_object(patch, obj):
  - if !is_module(obj) ∨ strcmp(obj.name, mod.name): continue.
  - obj.mod = mod.
  - klp_init_object_loaded(patch, obj); on err goto err.
  - klp_pre_patch_callback(obj); on err goto err.
  - klp_patch_object(obj); on err: klp_post_unpatch_callback; goto err.
  - if patch != klp_transition_patch: klp_post_patch_callback(obj).
- mutex_unlock; return 0.
- err: pr_warn "refusing to load"; mod.klp_alive = false; obj.mod = NULL; klp_cleanup_module_patches_limited(mod, patch); mutex_unlock; return ret.

REQ-16: klp_module_going(mod):
- WARN_ON(mod.state != GOING ∧ mod.state != COMING).
- mutex_lock(klp_mutex).
- mod.klp_alive = false.
- klp_cleanup_module_patches_limited(mod, NULL) — for every patch's matching object: klp_pre_unpatch_callback (if not the active transition), klp_unpatch_object, klp_post_unpatch_callback, klp_clear_object_relocs, klp_free_object_loaded.
- mutex_unlock.

REQ-17: Sysfs interface (`/sys/kernel/livepatch/`):
- `<patch>/enabled` RW: write 0 disables; write 1 only valid to re-enable a not-yet-applied patch; writing to the in-transition patch reverses the transition; writing to a disabled patch errors.
- `<patch>/transition` RO: 1 if patch == klp_transition_patch else 0.
- `<patch>/force` WO: write 1 to force-complete a stalled transition; only valid for the in-transition patch.
- `<patch>/replace` RO: patch.replace value.
- `<patch>/stack_order` RO: 1-based index in klp_patches list.
- `<patch>/<object>/patched` RO: obj.patched value.
- `<patch>/<object>/<function,sympos>/` — per-function kobject (no extra attrs at this layer).

REQ-18: klp_init (module_init): klp_root_kobj = kobject_create_and_add("livepatch", kernel_kobj); -ENOMEM on failure.

## Acceptance Criteria

- [ ] AC-1: klp_enable_patch with a non-livepatch module returns -EINVAL and emits "not marked as a livepatch module".
- [ ] AC-2: klp_enable_patch creates `/sys/kernel/livepatch/<patch>/enabled` after init_patch.
- [ ] AC-3: klp_find_object_symbol with sympos=0 and a non-unique symbol returns -EINVAL ("unresolvable ambiguity").
- [ ] AC-4: klp_find_object_symbol with sympos=N matches the Nth occurrence; sympos count mismatch returns -EINVAL.
- [ ] AC-5: klp_resolve_symbols rejects symbols whose st_shndx != SHN_LIVEPATCH with -EINVAL.
- [ ] AC-6: klp_resolve_symbols rejects module-specific rela referencing vmlinux symbol with -EINVAL.
- [ ] AC-7: A second concurrent klp_enable_patch while another is in transition returns -EBUSY.
- [ ] AC-8: After successful enable, /sys/kernel/livepatch/<patch>/enabled reads "1".
- [ ] AC-9: Writing "0" to `enabled` of the only-active patch triggers __klp_disable_patch and transitions to UNPATCHED.
- [ ] AC-10: Writing "1" to `force` of the in-transition patch invokes klp_force_transition; writing to a non-transition patch returns -EINVAL.
- [ ] AC-11: `stack_order` returns 1-based index matching position in klp_patches list.
- [ ] AC-12: replace mode: after transition completes, replaced old patches' enabled is false and their funcs are off the ftrace stack.
- [ ] AC-13: klp_module_coming applies pending patches to matching objects; failure refuses module load (returns ret).
- [ ] AC-14: klp_module_going unpatches and clears all matching objects across all patches.
- [ ] AC-15: kallsyms_lookup_size_offset returning 0 for new_func or old_func causes klp_init_object_loaded to return -ENOENT.

## Architecture

```
struct KlpPatch {
  mod: *Module,
  objs: *[KlpObject],                 // static, NULL-terminated
  states: *[KlpState],                // optional
  replace: bool,
  list: ListHead,                     // klp_patches
  kobj: Kobject,
  obj_list: ListHead,
  enabled: bool,
  forced: bool,
  free_work: WorkStruct,
  finish: Completion,
}

struct KlpObject {
  name: Option<&'static str>,         // None = vmlinux
  funcs: *[KlpFunc],
  callbacks: KlpCallbacks,
  kobj: Kobject,
  func_list: ListHead,
  mod: Option<*Module>,
  node: ListHead,
  patched: bool,
  dynamic: bool,
}

struct KlpFunc {
  old_name: *const u8,
  new_func: *const (),
  old_sympos: u64,
  old_func: *const (),
  old_size: u64,
  new_size: u64,
  kobj: Kobject,
  stack_node: ListHead,               // klp_ops.func_stack
  node: ListHead,                     // obj.func_list
  patched: bool,
  transition: bool,
  nop: bool,
}

struct KlpFindArg {
  name: *const u8,
  addr: u64,
  count: u64,
  pos: u64,
}
```

`Klp::enable_patch(patch) -> Result<(), i32>`:
1. /* Validate */
2. if !patch ∨ !patch.mod ∨ !patch.objs: return Err(-EINVAL).
3. for_each_static_object(patch, obj): if !obj.funcs: return Err(-EINVAL).
4. if !is_livepatch_module(patch.mod): pr_err; return Err(-EINVAL).
5. if !Klp::initialized(): return Err(-ENODEV).
6. if !klp_have_reliable_stack(): pr_warn.
7. KLP_MUTEX.lock().
8. if !Klp::is_patch_compatible(patch): unlock; return Err(-EINVAL).
9. if !try_module_get(patch.mod): unlock; return Err(-ENODEV).
10. Klp::init_patch_early(patch).
11. Klp::init_patch(patch).map_err(|e| goto err).
12. Klp::enable_patch_inner(patch).map_err(|e| goto err).
13. unlock; return Ok.
14. err: Klp::free_patch_start(patch); unlock; Klp::free_patch_finish(patch); return Err(ret).

`Klp::enable_patch_inner(patch) -> Result<(), i32>`:
1. if klp_transition_patch.is_some(): return Err(-EBUSY).
2. WARN_ON(patch.enabled).
3. Klp::init_transition(patch, KLP_TRANSITION_PATCHED).
4. smp_wmb.
5. for obj in patch.objs: if obj.is_loaded():
   - Klp::pre_patch_callback(obj)?
   - Klp::patch_object(obj)?  // register ftrace ops
6. Klp::start_transition.
7. patch.enabled = true.
8. Klp::try_complete_transition.
9. return Ok.

`Klp::find_object_symbol(objname, name, sympos) -> Result<u64, i32>`:
1. args = KlpFindArg { name, addr: 0, count: 0, pos: sympos }.
2. if objname.is_some():
   - module_kallsyms_on_each_symbol(objname, Klp::find_callback, &args).
3. else:
   - kallsyms_on_each_match_symbol(Klp::match_callback, name, &args).
4. if args.addr == 0: pr_err; return Err(-EINVAL).
5. if args.count > 1 ∧ sympos == 0: pr_err; return Err(-EINVAL).
6. if sympos > 0 ∧ args.count != sympos: pr_err; return Err(-EINVAL).
7. return Ok(args.addr).

`Klp::resolve_symbols(sechdrs, strtab, symndx, relasec, sec_objname) -> Result<(), i32>`:
1. for rela in relasec:
   - sym = sym_table[ELF_R_SYM(rela.r_info)].
   - if sym.st_shndx != SHN_LIVEPATCH: pr_err; return Err(-EINVAL).
   - sscanf("{KLP_SYM_PREFIX}{56s}.{512s},{lu}", &sym_objname, &sym_name, &sympos).
   - if cnt != 3: return Err(-EINVAL).
   - if !sec_vmlinux ∧ sym_vmlinux: return Err(-EINVAL).
   - addr = Klp::find_object_symbol(if sym_vmlinux { None } else { Some(sym_objname) }, sym_name, sympos)?
   - sym.st_value = addr.
2. return Ok.

`Klp::init_object_loaded(patch, obj) -> Result<(), i32>`:
1. if obj.is_module(): Klp::apply_object_relocs(patch, obj)?  // .klp.rela.{module}.*
2. for func in obj.funcs:
   - func.old_func = Klp::find_object_symbol(obj.name, func.old_name, func.old_sympos)?
   - (func.old_size, _) = kallsyms_lookup_size_offset(func.old_func).ok_or(-ENOENT)?
   - if func.nop: func.new_func = func.old_func.
   - (func.new_size, _) = kallsyms_lookup_size_offset(func.new_func).ok_or(-ENOENT)?
3. return Ok.

`Klp::add_nops(patch)`:
1. for old_patch in klp_patches: for old_obj in old_patch.objs:
   - Klp::add_object_nops(patch, old_obj).
2. Klp::add_object_nops:
   - obj = Klp::find_object(patch, old_obj) or Klp::alloc_object_dynamic.
   - for old_func in old_obj.funcs:
     - if !Klp::find_func(obj, old_func): Klp::alloc_func_nop(old_func, obj).

`Klp::module_coming(mod) -> Result<(), i32>`:
1. WARN_ON(mod.state != MODULE_STATE_COMING).
2. if mod.name == "vmlinux": return Err(-EINVAL).
3. KLP_MUTEX.lock().
4. mod.klp_alive = true.
5. for patch in klp_patches: for obj in patch.objs:
   - if !obj.is_module() ∨ obj.name != mod.name: continue.
   - obj.mod = Some(mod).
   - Klp::init_object_loaded(patch, obj).map_err(|e| goto err).
   - Klp::pre_patch_callback(obj).map_err(|e| goto err).
   - Klp::patch_object(obj).map_err(|e| Klp::post_unpatch_callback(obj); goto err).
   - if patch != klp_transition_patch: Klp::post_patch_callback(obj).
6. unlock; return Ok.
7. err: mod.klp_alive = false; obj.mod = None; Klp::cleanup_module_patches_limited(mod, patch); unlock; return Err(ret).

`Klp::module_going(mod)`:
1. WARN_ON(mod.state ∉ {GOING, COMING}).
2. KLP_MUTEX.lock().
3. mod.klp_alive = false.
4. Klp::cleanup_module_patches_limited(mod, None).
5. unlock.

`Klp::sysfs_enabled_store(kobj, buf) -> Result<usize, i32>`:
1. enabled = kstrtobool(buf)?
2. patch = container_of(kobj, KlpPatch, kobj).
3. KLP_MUTEX.lock().
4. if patch.enabled == enabled: unlock; return Err(-EINVAL).
5. if patch == klp_transition_patch: Klp::reverse_transition().
6. else if !enabled: Klp::disable_patch_inner(patch).
7. else: unlock; return Err(-EINVAL).
8. unlock; return Ok(count).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `module_refcount_balanced` | INVARIANT | per-enable_patch: try_module_get on success ↔ module_put on free (unless forced). |
| `sympos_resolves_uniquely` | INVARIANT | per-find_object_symbol: sympos=0 ⟹ args.count == 1; sympos=N ⟹ args.count == N. |
| `livepatch_module_marker_required` | INVARIANT | per-enable_patch: !is_livepatch_module ⟹ -EINVAL. |
| `mutex_held_during_bookkeeping` | INVARIANT | per-enable_patch/init_patch/module_coming/going: klp_mutex held. |
| `relocs_only_in_livepatch_section` | INVARIANT | per-resolve_symbols: sym.st_shndx == SHN_LIVEPATCH. |
| `module_specific_rela_no_vmlinux_sym` | INVARIANT | per-resolve_symbols: !sec_vmlinux ∧ sym_vmlinux ⟹ -EINVAL. |
| `one_transition_at_a_time` | INVARIANT | per-enable/disable_patch_inner: klp_transition_patch == NULL precondition. |

### Layer 2: TLA+

`kernel/livepatch/core.tla`:
- States: per-patch lifecycle {Disabled, Initialized, Enabling, Transitioning, Enabled, Disabling, FreePending, Freed}.
- Actions: enable_patch, disable_patch, module_coming, module_going, transition_complete, transition_cancel, sysfs_enabled_store, sysfs_force_store, sysfs_replace_transition_reverse.
- Properties:
  - `safety_single_transition_patch` — per-global: at-most-one patch with klp_transition_patch != NULL.
  - `safety_enabled_implies_in_klp_patches` — per-patch: enabled ⟹ ∈ klp_patches list.
  - `safety_module_get_balanced` — per-enable: try_module_get count == module_put count (unless forced).
  - `safety_replace_supersedes_old` — per-replace transition: post-complete, old patches.enabled == false.
  - `safety_module_going_unpatches_all` — per-module-going: ∀ patch: patch.objs matching mod.name: obj.patched == false.
  - `liveness_enabled_eventually_transitions_or_errors` — per-enable_patch: returns ⟹ patch.enabled ∈ {true, false} ∧ klp_transition_patch == NULL eventually.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Klp::enable_patch` post: Ok ⟹ patch.enabled = true ∨ in transition; Err ⟹ patch freed, module_put | `Klp::enable_patch` |
| `Klp::init_patch` post: kobject_add succeeded; klp_patches contains patch | `Klp::init_patch` |
| `Klp::init_object_loaded` post: all funcs have non-NULL old_func ∧ old_size > 0 ∧ new_size > 0 | `Klp::init_object_loaded` |
| `Klp::find_object_symbol` post: Ok(addr) ⟹ addr != 0 ∧ matches sympos | `Klp::find_object_symbol` |
| `Klp::resolve_symbols` post: ∀ rela: sym.st_value = resolved-kallsyms addr | `Klp::resolve_symbols` |
| `Klp::add_nops` post: every old-patch func not in new-patch ⟹ NOP synthesised in new-patch | `Klp::add_nops` |
| `Klp::module_coming` post: Ok ⟹ ∀ matching obj: obj.patched ∨ patch in transition | `Klp::module_coming` |
| `Klp::module_going` post: ∀ matching obj across all patches: obj.patched == false ∧ relocs cleared | `Klp::module_going` |

### Layer 4: Verus/Creusot functional

`Per-klp_enable_patch → init_patch_early → init_patch (sysfs + init_object loop) → __klp_enable_patch (init_transition + pre_patch + patch_object + start_transition + try_complete_transition) → patch.enabled = true` semantic equivalence with upstream: per-Documentation/livepatch/livepatch.rst and Documentation/livepatch/api.rst. Per-`klp_module_coming/going` semantic equivalence with kernel/module/main.c notifier integration.

## Hardening

(Inherits row-1 features from `kernel/livepatch/00-overview.md` § Hardening.)

Livepatch-core reinforcement:

- **Per-LIVEPATCH ELF marker required** — defense against per-arbitrary-module being treated as patch.
- **Per-SHN_LIVEPATCH symbol gate** — defense against per-rela referencing non-livepatch ELF symbols.
- **Per-module-specific rela cannot reference vmlinux symbol** — defense against per-init-ordering races.
- **Per-sympos disambiguation strict** — defense against per-ambiguous-kallsyms picking wrong same-name function.
- **Per-module_get on enable** — defense against per-patch-module unloaded while still installed.
- **Per-klp_mutex serialises all bookkeeping** — defense against per-concurrent enable/disable/module-coming/going races.
- **Per-single transition_patch enforcement** — defense against per-overlapping transitions corrupting `task->patch_state`.
- **Per-`is_livepatch_module` check** — defense against per-malformed patch module.
- **Per-kallsyms_lookup_size_offset == 0 ⟹ -ENOENT** — defense against per-zero-sized function bypassing stack check.
- **Per-clear_relocate_add on module-going** — defense against per-stale relocations holding addresses into unloaded module.
- **Per-forced patch keeps module_put deferred** — defense against per-ftrace-handler still seeing module text post-unload.
- **Per-MODULE_NAME_LEN/KSYM_NAME_LEN BUILD_BUG_ON** — defense against per-sscanf-overflow in symbol parsing.
- **Per-replace-mode NOP synthesis** — defense against per-old-patch function leaks when atomic-replace skips them.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-task transition state machine (covered in `transition.md` Tier-3)
- ftrace handler internals (covered in `kernel/livepatch/patch.md` Tier-3)
- klp_state per-system-state cookie (covered in `kernel/livepatch/state.md` Tier-3)
- klp_shadow variables (covered in `kernel/livepatch/shadow.md` Tier-3)
- kernel/module loader integration (covered in `kernel/module/` Tier-3 docs)
- arch_apply_relocate_add / arch_klp_init_object_loaded (covered in arch Tier-3 docs)
- Implementation code
