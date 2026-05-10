---
title: "Tier-3: kernel/module/main.c — Module load/unload core"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`init_module(2)` / `finit_module(2)` / `delete_module(2)` syscalls implement
runtime kernel-module load and unload. The core loader takes a user-supplied
ELF blob (or fd to a `.ko` file), runs a strict ELF validity cache, allocates
core/init/RO-after-init/exec/RX/RW memory regions, lays out sections,
resolves and symbol-checks `__ksymtab`-exported names, applies SHT_REL/RELA
relocations, runs `do_init_module()` to invoke the module's
`module_init()` callback, and links the module into `LIST_HEAD(modules)`
under `DEFINE_MUTEX(module_mutex)`. Per-module signature is verified by
`module_sig_check()` before any header parsing. Per-module reference
count drives `try_module_get()` / `module_put()`. Per-`TAINT_*` flags
record license-violation / out-of-tree / unsigned / forced-load. Per-
`MODULE_STATE_*` state machine (UNFORMED → COMING → LIVE → GOING) governs
the visibility of symbols and the eligibility of unload. Critical for:
device-driver loading, runtime feature enablement, livepatch, security
posture (signature + lockdown), and `kmod`-style autoload.

This Tier-3 covers `kernel/module/main.c` (~3988 lines).

### Acceptance Criteria

- [ ] AC-1: init_module(2) loads valid ELF module → state LIVE; `lsmod` shows entry.
- [ ] AC-2: init_module(2) with unsigned blob ∧ CONFIG_MODULE_SIG_FORCE → -EKEYREJECTED.
- [ ] AC-3: finit_module(2) idempotent: two concurrent loads of same inode → single load, second waits for completion.
- [ ] AC-4: delete_module(2) with refcnt > 0 → -EBUSY (without O_TRUNC/force).
- [ ] AC-5: delete_module(2) of dependent's parent → -EWOULDBLOCK.
- [ ] AC-6: SHN_UNDEF symbol that no exporter provides → load fails -ENOENT.
- [ ] AC-7: Module with duplicate __ksymtab entry already loaded → -ENOEXEC.
- [ ] AC-8: Non-GPL module references EXPORT_SYMBOL_GPL → -ENOENT and pr_warn.
- [ ] AC-9: Init failure (do_one_initcall returns -E): module dropped, state GOING → freed; module_wq waiters awoken.
- [ ] AC-10: After init: MOD_INIT_* memory freed; MOD_RO_AFTER_INIT flipped to RO; subsequent write traps.
- [ ] AC-11: Module without GPL-compat license → TAINT_PROPRIETARY_MODULE set on kernel.
- [ ] AC-12: Concurrent load of same module name: second loader waits on module_wq, completes with -EEXIST when first reaches LIVE.
- [ ] AC-13: try_module_get on GOING module returns false.
- [ ] AC-14: request_module_nowait("foo") spawns modprobe asynchronously; load_module respected.

### Architecture

```
struct LoadInfo {
  hdr: *ElfEhdr,
  len: usize,
  sechdrs: *ElfShdr,
  secstrings: *const u8,
  strtab: *const u8,
  mod: *Module,                 // pointer into ELF copy
  index: SectionIndex {
    sym, str, mod, info, pcpu, vers, vers_ext_crc, vers_ext_name: u32,
  },
  name: *const u8,
  sig_ok: bool,
}

struct Module {
  name: [u8; MODULE_NAME_LEN],
  state: ModuleState,           // UNFORMED / COMING / LIVE / GOING
  list: ListHead,
  mem: [ModuleMemory; MOD_MEM_NUM_TYPES],
  refcnt: AtomicI32,
  init: Option<fn() -> i32>,
  exit: Option<fn()>,
  kp: *KernelParam, num_kp: u32,
  syms: *KernelSymbol, num_syms: u32,
  crcs: *u32,
  flagstab: *u8,
  ctors: *fn(), num_ctors: u32,
  jump_entries: *JumpEntry, num_jump_entries: u32,
  args: *u8,
  source_list: ListHead, target_list: ListHead,
  taints: u64,
  async_probe_requested: bool,
  // ... arch + sysfs + livepatch + ftrace fields
}
```

`ModuleSys::init_module(umod, len, uargs) -> i32`:
1. may_init_module() — CAP_SYS_MODULE ∧ !modules_disabled, else -EPERM.
2. copy_module_from_user(umod, len, &info) — vmalloc info.hdr; copy_from_user.
3. load_module(&info, uargs, 0).

`ModuleSys::finit_module(fd, uargs, flags) -> i32`:
1. may_init_module().
2. fdget(fd) ⟶ file.
3. Validate flags ⊆ MODULE_INIT_IGNORE_MODVERSIONS | _IGNORE_VERMAGIC | _COMPRESSED_FILE.
4. idempotent_init_module(file, uargs, flags):
   - cookie = file_inode(file).
   - if first cookie holder: init_module_from_file → idempotent_complete.
   - else: idempotent_wait_for_completion.

`ModuleSys::delete_module(name_user, flags) -> i32`:
1. Capability + name copy + audit_log_kern_module.
2. mutex_lock_interruptible(&module_mutex).
3. mod = find_module(name); if NULL → -ENOENT.
4. if dependents → -EWOULDBLOCK.
5. if state != LIVE → -EBUSY.
6. if init ∧ !exit ∧ !try_force_unload(flags) → -EBUSY.
7. try_stop_module(mod, flags, &forced) — refcount check + GOING transition.
8. mutex_unlock.
9. mod.exit(); notifier → MODULE_STATE_GOING; klp_module_going; ftrace_release_mod; async_synchronize_full.
10. Save last_unloaded_module.{name, taints}.
11. free_module(mod); wake_up_all(&module_wq).

`ModuleLoader::load(info, uargs, flags) -> i32`:
1. module_sig_check(info, flags) — strip + verify sig.
2. elf_validity_cache_copy(info, flags) — ehdr/sechdrs/secstrings/index/strtab.
3. early_mod_check(info, flags) — blacklist; rewrite_section_headers; modstruct version; modinfo; patient_check_exists.
4. mod = layout_and_allocate(info, flags) — layout_sections; layout_symtab; move_module (execmem_alloc per MOD_* type).
5. audit_log_kern_module(info.name).
6. add_unformed_module(mod) — state UNFORMED; list_add_rcu; mod_tree_insert.
7. module_augment_kernel_taints(mod, info).
8. percpu_modalloc(mod, info).
9. module_unload_init(mod) — refcnt = 1; INIT_LIST_HEAD lists.
10. init_param_lock.
11. find_module_sections(mod, info).
12. check_export_symbol_sections.
13. setup_modinfo.
14. simplify_symbols(mod, info) — SHN_COMMON err; SHN_UNDEF → resolve_symbol_wait.
15. apply_relocations(mod, info).
16. post_relocation; flush_module_icache.
17. mod.args = strndup_user(uargs).
18. init_build_id; ftrace_module_init.
19. complete_formation(mod, info) — under module_mutex: verify_exported_symbols, bug/cfi finalize, enable_rodata_ro / data_nx / text_rox; state → COMING.
20. prepare_coming_module(mod) — ftrace + klp + notifier_call_chain MODULE_STATE_COMING.
21. parse_args.
22. mod_sysfs_setup.
23. if is_livepatch_module: copy_module_elf.
24. codetag_load_module.
25. free_copy(info, flags).
26. trace_module_load.
27. do_init_module(mod).
- On any error: cascading goto labels (sysfs_cleanup, coming_cleanup, bug_cleanup, ddebug_cleanup, free_arch_cleanup, free_modinfo, free_unload, unlink_mod, free_module, free_copy) — synchronize_rcu before releasing list; mod_stat_bump_invalid.

`ModuleLoader::do_init_module(mod) -> i32`:
1. freeinit = kmalloc(struct mod_initfree).
2. do_mod_ctors(mod).
3. if mod.init: ret = do_one_initcall(mod.init).
4. if ret < 0:
   - if ret == -EEXIST: ret = -EBUSY (UAPI reserves -EEXIST for already-loaded).
   - state → GOING; synchronize_rcu; module_put; notifier MODULE_STATE_GOING; klp_module_going; ftrace_release_mod; free_module; wake_up_all.
   - return ret.
5. state ← LIVE; notifier MODULE_STATE_LIVE; kobject_uevent KOBJ_ADD.
6. if !async_probe_requested: async_synchronize_full.
7. ftrace_free_mem(mod, MOD_INIT_TEXT base..end).
8. mutex_lock(&module_mutex); module_put(mod); trim_init_extable; rcu_assign_pointer(mod.kallsyms, &mod.core_kallsyms).
9. module_enable_rodata_ro_after_init.
10. mod_tree_remove_init.
11. module_arch_freeing_init.
12. For each MOD_INIT_* type: base=NULL, size=0.
13. llist_add(&freeinit.node, &init_free_list); schedule_work(&init_free_wq).
14. mutex_unlock; wake_up_all(&module_wq).
15. mod_stat_inc(&modcount).

`ModuleLoader::simplify_symbols(mod, info) -> i32`:
1. For each Elf_Sym sym in info.sechdrs[info.index.sym]:
   - Switch on sym.st_shndx:
     - SHN_COMMON → -ENOEXEC.
     - SHN_ABS → keep.
     - SHN_UNDEF → resolve_symbol_wait(mod, info, name); if !found ∧ !ignore_undef_symbol → -ENOENT.
     - Default → sym.st_value += info.sechdrs[sym.st_shndx].sh_addr.
2. return 0.

`ModuleLoader::apply_relocations(mod, info) -> i32`:
1. For i in 1..info.hdr.e_shnum:
   - infosec = info.sechdrs[i].sh_info.
   - if infosec ≥ e_shnum: skip.
   - if !(sechdrs[infosec].sh_flags & SHF_ALLOC) ∧ infosec != index.pcpu: skip.
   - if sechdrs[i].sh_flags & SHF_RELA_LIVEPATCH: klp_apply_section_relocs.
   - else if sechdrs[i].sh_type == SHT_REL: apply_relocate.
   - else if sechdrs[i].sh_type == SHT_RELA: apply_relocate_add.
2. return 0 / err.

`Module::try_get(module) -> bool`:
1. if module is None: true.
2. preempt_disable.
3. if module.state != LIVE: preempt_enable; return false.
4. atomic_inc(&module.refcnt).
5. preempt_enable; return true.

`Module::find_symbol(fsa) -> bool`:
1. rcu_read_lock.
2. Search kernel `__ksymtab` (vmlinux).
3. For each Module in `&modules` list: find_exported_symbol_in_section.
4. Honor fsa.gplok against per-symbol flagstab GPL bit.
5. rcu_read_unlock.

### Out of Scope

- kernel/module/signing.c sig verification internals (covered in `module-signing.md` Tier-3 if expanded)
- kernel/module/kallsyms.c symbol-table lookup details (covered in `module-kallsyms.md` Tier-3 if expanded)
- kernel/module/kmod.c request_module / modprobe spawn (covered in `kmod.md` Tier-3 if expanded)
- kernel/module/strict_rwx.c page-permission flipping (covered in `module-rwx.md` Tier-3 if expanded)
- kernel/livepatch (covered in `livepatch/00-overview.md`)
- ftrace_module_init (covered in `trace/ftrace.md`)
- kexec / hibernate freeze (covered in `kexec.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE3(init_module, ...)` | per-syscall (buffer-from-user) | `ModuleSys::init_module` |
| `SYSCALL_DEFINE3(finit_module, ...)` | per-syscall (fd-based) | `ModuleSys::finit_module` |
| `SYSCALL_DEFINE2(delete_module, ...)` | per-syscall (unload) | `ModuleSys::delete_module` |
| `struct module` | per-loaded-module descriptor | `Module` |
| `struct load_info` | per-load transient context | `LoadInfo` |
| `load_module()` | per-load main path | `ModuleLoader::load` |
| `module_sig_check()` | per-load signature verify | `ModuleSig::check` |
| `elf_validity_cache_copy()` | per-load ELF parse + cache | `ModuleLoader::elf_validity_cache` |
| `early_mod_check()` | per-blacklist + modstruct version | `ModuleLoader::early_check` |
| `layout_and_allocate()` | per-mem-layout + alloc | `ModuleLoader::layout_and_allocate` |
| `layout_sections()` / `__layout_sections()` | per-section bucketing | `ModuleLoader::layout_sections` |
| `layout_symtab()` | per-kallsyms layout | `ModuleLoader::layout_symtab` |
| `move_module()` | per-section move into final mem | `ModuleLoader::move_module` |
| `find_module_sections()` | per-section pointer discovery | `ModuleLoader::find_module_sections` |
| `simplify_symbols()` | per-symbol resolution | `ModuleLoader::simplify_symbols` |
| `apply_relocations()` | per-SHT_REL / SHT_RELA reloc | `ModuleLoader::apply_relocations` |
| `post_relocation()` | per-arch post hook | `ModuleLoader::post_relocation` |
| `complete_formation()` | per-RWX enable + COMING state | `ModuleLoader::complete_formation` |
| `prepare_coming_module()` | per-COMING notifier chain | `ModuleLoader::prepare_coming` |
| `do_init_module()` | per-call module_init() | `ModuleLoader::do_init_module` |
| `add_unformed_module()` | per-list insert + dedup | `ModuleLoader::add_unformed` |
| `module_patient_check_exists()` | per-wait for racing load | `ModuleLoader::patient_check_exists` |
| `finished_loading()` | per-LIVE/GOING check | `ModuleLoader::finished_loading` |
| `free_module()` | per-unload destructor | `Module::free` |
| `try_module_get()` | per-refcount inc (if LIVE) | `Module::try_get` |
| `module_put()` | per-refcount dec | `Module::put` |
| `__module_get()` | per-unchecked inc | `Module::get_raw` |
| `try_stop_module()` | per-unload refcount check | `Module::try_stop` |
| `verify_exported_symbols()` | per-load duplicate-export reject | `ModuleLoader::verify_exports` |
| `find_symbol()` | per-symbol lookup over modules | `Module::find_symbol` |
| `find_module()` / `find_module_all()` | per-name lookup | `Module::find` / `find_all` |
| `module_augment_kernel_taints()` | per-load taint propagation | `Module::augment_taints` |
| `add_taint_module()` | per-flag set | `Module::add_taint` |
| `module_license_taint_check()` | per-GPL-compat check | `Module::license_check` |
| `mod_sysfs_setup()` / `mod_sysfs_teardown()` | per-`/sys/module/<name>` | `Module::sysfs_setup` / `_teardown` |
| `module_memory_alloc()` / `_free()` | per-type exec/RO/RW alloc | `Module::mem_alloc` / `_free` |
| `module_arch_freeing_init()` | per-arch init free | `Module::arch_freeing_init` |
| `module_unload_init()` / `_free()` | per-source/target list | `Module::unload_init` / `_free` |
| `ref_module()` / `already_uses()` | per-module dependency | `Module::ref` / `already_uses` |
| `module_refcount()` | per-int read | `Module::refcount` |
| `EXPORT_SYMBOL()` / `_GPL()` | per-section emission (build) | `kernel_symbol` table |
| `THIS_MODULE` | per-build self-pointer | `THIS_MODULE` const |
| `request_module_nowait()` | per-modprobe spawn (kmod) | `Module::request_nowait` |
| `module_mutex` | per-load global lock | `MODULE_MUTEX` |
| `module_wq` | per-state-change wait | `MODULE_WQ` |
| `kexec`-OOM-suspend disable | shared | (cross-ref) |

### compatibility contract

REQ-1: struct module (key fields):
- name: char[MODULE_NAME_LEN] — unique module name.
- state: MODULE_STATE_LIVE / _COMING / _GOING / _UNFORMED.
- list: list_head — links into LIST_HEAD(modules).
- refcnt: atomic_t — incremented by try_module_get / __module_get.
- mem[MOD_MEM_NUM_TYPES]: array of `{ base, size }` for MOD_TEXT, MOD_DATA, MOD_RODATA, MOD_RO_AFTER_INIT, MOD_INIT_TEXT, MOD_INIT_DATA, MOD_INIT_RODATA.
- init, exit: function pointers.
- kp, num_kp: kernel_param array (module params).
- syms, num_syms: __ksymtab — exported.
- crcs: __kcrctab — modversions CRCs.
- flagstab: __kflagstab — per-symbol flags (e.g. GPL).
- ctors, num_ctors: C++ ctor array (CONFIG_CONSTRUCTORS).
- jump_entries, num_jump_entries: __jump_table.
- args: char *, copied from uargs.
- source_list, target_list: dependency lists.
- taints: unsigned long bitmask.
- async_probe_requested: bool.

REQ-2: struct load_info (key fields):
- hdr: Elf_Ehdr * — pointer to loaded ELF in temp buffer.
- len: size_t — total ELF size (signature truncated after sig-check).
- sechdrs: Elf_Shdr * — pointer to section headers.
- secstrings: char * — section-name string table.
- strtab: char * — symbol-name string table.
- mod: struct module * — pointer to in-ELF copy of module struct.
- index: { sym, str, mod, info, pcpu, vers, vers_ext_crc, vers_ext_name } — section indices.
- name: char * — module name (resolved from .modinfo).
- sig_ok: bool — set by module_sig_check.

REQ-3: SYSCALL init_module(umod, len, uargs):
- may_init_module() — requires CAP_SYS_MODULE ∧ !modules_disabled.
- copy_module_from_user(umod, len, &info) — vmalloc + copy_from_user.
- load_module(&info, uargs, 0).

REQ-4: SYSCALL finit_module(fd, uargs, flags):
- may_init_module().
- fdget(fd) → file *.
- idempotent_init_module(file, uargs, flags):
  - check `idem_hash` for racing load with same `file_inode(f)` cookie.
  - if first: init_module_from_file → kernel_read_file (optionally decompress per MODULE_INIT_COMPRESSED_FILE) → load_module(&info, uargs, flags) → idempotent_complete.
  - else: wait_for_completion_interruptible.
- flags allowed: MODULE_INIT_IGNORE_MODVERSIONS | MODULE_INIT_IGNORE_VERMAGIC | MODULE_INIT_COMPRESSED_FILE.

REQ-5: SYSCALL delete_module(name_user, flags):
- requires CAP_SYS_MODULE ∧ !modules_disabled.
- strncpy_from_user name; reject empty / overlong.
- audit_log_kern_module(name).
- mutex_lock_interruptible(&module_mutex).
- mod = find_module(name); if NULL → -ENOENT.
- if !list_empty(&mod.source_list) → -EWOULDBLOCK (dependents).
- if mod.state != MODULE_STATE_LIVE → -EBUSY.
- if mod.init ∧ !mod.exit: try_force_unload(flags) — else -EBUSY.
- try_stop_module(mod, flags, &forced) — refcount check; mod.state ← MODULE_STATE_GOING.
- mutex_unlock.
- mod.exit() if non-NULL.
- blocking_notifier_call_chain(&module_notify_list, MODULE_STATE_GOING, mod).
- klp_module_going; ftrace_release_mod; async_synchronize_full.
- Save name + taints to last_unloaded_module.
- free_module(mod); wake_up_all(&module_wq).

REQ-6: load_module(info, uargs, flags) main path:
- 1. module_sig_check(info, flags) — read trailing sig if present; verify per CONFIG_MODULE_SIG.
- 2. elf_validity_cache_copy(info, flags) — chain of:
  - elf_validity_ehdr — magic, machine, EI_CLASS, e_type == ET_REL.
  - elf_validity_cache_sechdrs — each sh_offset + sh_size ≤ info.len, sh_link / sh_info bounds.
  - elf_validity_cache_secstrings — sh_strtab valid.
  - elf_validity_cache_index — find {.sym, .str, .modinfo, __versions, .data..percpu, .modinfo}.
  - elf_validity_cache_strtab — strtab terminator + bounds.
- 3. early_mod_check(info, flags):
  - blacklisted(info.name) → -EPERM.
  - rewrite_section_headers — set sh_addr.
  - check_modstruct_version — verify struct module vermagic.
  - check_modinfo — license, vermagic, retpoline (warns if mismatched).
  - module_patient_check_exists — under module_mutex; if existing UNFORMED/COMING, wait_event_interruptible on module_wq.
- 4. layout_and_allocate(info, flags):
  - alloc struct module skeleton.
  - layout_sections → __layout_sections(is_init=false) then (is_init=true): bucket SHF_ALLOC sections into MOD_TEXT / MOD_RODATA / MOD_RO_AFTER_INIT / MOD_DATA (resp. MOD_INIT_*).
  - layout_symtab — reserve room for kallsyms.
  - move_module — module_memory_alloc(MOD_*); memcpy each SHF_ALLOC section into mod.mem[type].base + sh_entsize-encoded offset.
- 5. add_unformed_module(mod):
  - mod.state ← MODULE_STATE_UNFORMED.
  - module_patient_check_exists.
  - mod_update_bounds; list_add_rcu(&mod.list, &modules); mod_tree_insert.
- 6. module_augment_kernel_taints(mod, info) — TAINT_OOT_MODULE / TAINT_PROPRIETARY_MODULE / TAINT_CRAP / TAINT_LIVEPATCH / TAINT_TEST.
- 7. percpu_modalloc(mod, info) — separate per-CPU section.
- 8. module_unload_init(mod) — INIT_LIST_HEAD source_list, target_list; atomic_set(&mod.refcnt, 1).
- 9. init_param_lock(mod).
- 10. find_module_sections(mod, info) — populate mod.kp, mod.syms, mod.crcs, mod.flagstab, mod.ctors, mod.jump_entries, mod.tracepoints_ptrs, mod.bpf_raw_events, mod.extable, mod.dyndbg_info, mod.kunit_suites, etc.
- 11. check_export_symbol_sections — sanity __ksymtab / __kcrctab / __kflagstab sizes match.
- 12. setup_modinfo — version, srcversion, import_ns attrs.
- 13. simplify_symbols(mod, info) — for each Elf_Sym in .symtab:
  - SHN_COMMON → -ENOEXEC.
  - SHN_ABS → keep st_value.
  - SHN_UNDEF (extern) → resolve_symbol_wait → find_symbol in other modules; if !found ∧ !ignore_undef_symbol → -ENOENT.
  - Otherwise → st_value += sechdrs[st_shndx].sh_addr.
- 14. apply_relocations(mod, info) — for each section:
  - if !SHF_ALLOC ∧ infosec != index.pcpu: skip.
  - SHF_RELA_LIVEPATCH → klp_apply_section_relocs.
  - SHT_REL → apply_relocate (arch).
  - SHT_RELA → apply_relocate_add (arch).
- 15. post_relocation(mod, info) — arch hook (e.g. x86 alternatives).
- 16. flush_module_icache(mod) — arch icache invalidate.
- 17. mod.args = strndup_user(uargs).
- 18. init_build_id(mod, info) — read .note.gnu.build-id.
- 19. ftrace_module_init(mod) — discover mcount sites.
- 20. complete_formation(mod, info):
  - mutex_lock(&module_mutex).
  - verify_exported_symbols — reject duplicate __ksymtab names.
  - module_bug_finalize; module_cfi_finalize.
  - module_enable_rodata_ro / data_nx / text_rox — set page permissions per MOD_* type.
  - mod.state ← MODULE_STATE_COMING.
  - mutex_unlock.
- 21. prepare_coming_module(mod) — ftrace_module_enable; klp_module_coming; blocking_notifier_call_chain MODULE_STATE_COMING.
- 22. parse_args(mod.name, mod.args, mod.kp, mod.num_kp, ..., unknown_module_param_cb).
- 23. mod_sysfs_setup(mod, info, mod.kp, mod.num_kp) — create /sys/module/<name>, holders/, parameters/, sections/, notes/.
- 24. codetag_load_module(mod) — alloc-tagging support.
- 25. free_copy(info, flags) — release temp ELF buffer.
- 26. trace_module_load.
- 27. do_init_module(mod).

REQ-7: do_init_module(mod):
- Alloc mod_initfree { init_text, init_data, init_rodata } from mod.mem[MOD_INIT_*].base.
- do_mod_ctors(mod) — call each fn in mod.ctors[0..num_ctors].
- if mod.init: ret = do_one_initcall(mod.init).
- if ret < 0 (init failed):
  - if ret == -EEXIST: ret = -EBUSY.
  - goto fail_free_freeinit → MODULE_STATE_GOING → notifier → ftrace_release_mod → free_module → wake_up_all(&module_wq).
- if ret > 0: pr_warn (non-0/-E convention).
- mod.state ← MODULE_STATE_LIVE.
- blocking_notifier_call_chain MODULE_STATE_LIVE.
- kobject_uevent KOBJ_ADD.
- async_synchronize_full (unless async_probe_requested).
- ftrace_free_mem on mod.mem[MOD_INIT_TEXT].
- mutex_lock(&module_mutex); module_put(mod) — drop initial ref; trim_init_extable.
- module_enable_rodata_ro_after_init(mod) — flip MOD_RO_AFTER_INIT to RO.
- For each MOD_INIT_* type: mod.mem[type].base = NULL; .size = 0.
- llist_add(&freeinit->node, &init_free_list); schedule_work(&init_free_wq) — do_free_init defers execmem_free until RCU-safe.
- mutex_unlock; wake_up_all(&module_wq).

REQ-8: free_module(mod):
- trace_module_free.
- codetag_unload_module.
- mod_sysfs_teardown.
- mutex_lock(&module_mutex); mod.state ← MODULE_STATE_UNFORMED; mutex_unlock.
- module_arch_cleanup(mod).
- module_unload_free(mod) — iterate target_list / source_list, drop refs.
- module_destroy_params.
- if is_livepatch_module: free_module_elf.
- mutex_lock(&module_mutex); list_del_rcu(&mod.list); mod_tree_remove; module_bug_cleanup; synchronize_rcu; try_add_tainted_module; mutex_unlock.
- module_arch_freeing_init(mod) — arch may have copied init for later use.
- kfree(mod.args); percpu_modfree(mod).
- free_mod_mem(mod) — execmem_free each mod.mem[type].base.

REQ-9: module_sig_check(info, flags):
- if !CONFIG_MODULE_SIG: return 0.
- if info.len ≤ sizeof(MODULE_SIG_STRING): -ENODATA.
- if memcmp(info.hdr + info.len - sizeof(MODULE_SIG_STRING), MODULE_SIG_STRING) != 0: -ENODATA (unsigned).
- Else strip sig marker; verify sig per CONFIG_MODULE_SIG_FORMAT (PKCS#7).
- if sig_enforce ∧ unsigned: -EKEYREJECTED.
- if unsigned: add_taint(TAINT_UNSIGNED_MODULE) ∧ pr_notice.
- info.len -= sig_len; info.sig_ok = (verified).

REQ-10: try_module_get(module) -> bool:
- if module == NULL: true.
- preempt_disable.
- if module.state != MODULE_STATE_LIVE:
  - preempt_enable.
  - return false.
- atomic_inc(&module.refcnt).
- preempt_enable.
- return true.

REQ-11: module_put(module):
- if module:
  - if atomic_dec_if_positive(&module.refcnt) == 0: BUG (would underflow).
  - else: trace_module_put.

REQ-12: __module_get(module):
- if module: atomic_inc(&module.refcnt) — unchecked (caller asserts ref held).

REQ-13: try_stop_module(mod, flags, &forced):
- if mod.refcnt > 0: if try_force_unload(flags) → forced=1 else -EBUSY.
- mod.state ← MODULE_STATE_GOING.

REQ-14: find_symbol(fsa):
- Iterate { kernel_symtab (vmlinux), each loaded module.syms }:
  - find_exported_symbol_in_section(syms, name, fsa).
  - Honor gplok flag — GPL-only sym refused if owner not GPL.
- Returns true if found; sets fsa.{owner, sym, license, namespace}.

REQ-15: EXPORT_SYMBOL / _GPL emission contract:
- Compile-time: writes a `struct kernel_symbol { value_offset, name_offset, namespace_offset }` into `__ksymtab` / `__ksymtab_gpl` (deprecated section name; merged at build-time into `__ksymtab` with `__kflagstab` carrying the GPL bit).
- find_symbol consults flagstab to enforce GPL.

REQ-16: THIS_MODULE:
- Per-translation-unit constant `extern struct module __this_module`; resolves to the loaded module's struct in __mod_<...> section, or `NULL` for vmlinux-internal compilation.
- Used as owner pointer in file_operations / device_driver / etc.

REQ-17: module_augment_kernel_taints / TAINT_*:
- TAINT_PROPRIETARY_MODULE — license not GPL-compatible.
- TAINT_OOT_MODULE — `intree` modinfo absent.
- TAINT_FORCED_MODULE — `--force` modprobe.
- TAINT_CRAP — `staging:` license string.
- TAINT_UNSIGNED_MODULE — failed module_sig_check (when !sig_enforce).
- TAINT_LIVEPATCH — livepatch module.
- TAINT_TEST — modinfo `test` annotation.

REQ-18: request_module_nowait (kmod):
- Spawns userspace `modprobe` via call_usermodehelper, UMH_NO_WAIT.
- Synchronous variant request_module (UMH_WAIT_EXEC); uses kmod_concurrent throttle and finished_loading.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `init_module_caps_required` | INVARIANT | per-init_module / finit_module / delete_module: !CAP_SYS_MODULE ⟹ -EPERM. |
| `modules_disabled_blocks_load` | INVARIANT | per-may_init_module: modules_disabled=1 ⟹ -EPERM. |
| `elf_validity_strict` | INVARIANT | per-elf_validity_cache_copy: out-of-bounds section ⟹ -ENOEXEC. |
| `sig_enforce_rejects_unsigned` | INVARIANT | per-module_sig_check + sig_enforce: unsigned ⟹ -EKEYREJECTED. |
| `state_machine_monotonic` | INVARIANT | per-mod.state: UNFORMED → COMING → LIVE → GOING (no reverse). |
| `refcnt_balanced_init_path` | INVARIANT | per-load_module: refcnt starts at 1; do_init_module drops it; delete_module drops final. |
| `module_mutex_held_for_list_mut` | INVARIANT | per-list_add_rcu / list_del_rcu of &mod.list: module_mutex held. |
| `idempotent_inode_dedup` | INVARIANT | per-idempotent_init_module: only first holder of inode cookie runs init_module_from_file. |
| `kp_array_bounds_checked` | INVARIANT | per-parse_args: mod.kp / mod.num_kp consistent with __param section size. |
| `relocation_section_valid` | INVARIANT | per-apply_relocations: sh_info < e_shnum ∧ target SHF_ALLOC. |
| `init_text_freed_after_init` | INVARIANT | per-do_init_module: MOD_INIT_TEXT.base ← NULL post-init. |
| `ro_after_init_flipped` | INVARIANT | per-do_init_module: module_enable_rodata_ro_after_init invoked exactly once. |

### Layer 2: TLA+

`kernel/module-load.tla`:
- Per-syscall init_module / finit_module / delete_module + concurrent loaders + module_wq.
- Properties:
  - `safety_unique_module_name` — per-list: no two LIVE modules share `name`.
  - `safety_caps_required` — per-load: CAP_SYS_MODULE held.
  - `safety_sig_enforce` — per-sig_enforce: unsigned ⟹ refused.
  - `safety_state_monotonic` — per-mod.state: transitions only UNFORMED → COMING → LIVE → GOING.
  - `safety_refcount_nonnegative` — per-Module.refcnt: never negative.
  - `safety_no_load_during_unload` — per-state==GOING: no concurrent loader observes it as LIVE.
  - `safety_idempotent_finit` — per-(file inode) cookie: at most one in-flight load.
  - `liveness_loader_progresses` — per-load_module: terminates in -E or LIVE.
  - `liveness_init_wq_woken` — per-load failure / completion: wake_up_all(&module_wq).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ModuleLoader::load` post: ret == 0 ⟹ mod.state == LIVE ∧ mod ∈ modules | `ModuleLoader::load` |
| `ModuleLoader::elf_validity_cache` post: all section offsets ≤ info.len | `ModuleLoader::elf_validity_cache` |
| `ModuleLoader::simplify_symbols` post: no SHN_COMMON survives | `ModuleLoader::simplify_symbols` |
| `ModuleLoader::apply_relocations` post: each reloc target SHF_ALLOC | `ModuleLoader::apply_relocations` |
| `ModuleLoader::complete_formation` post: mod.mem[MOD_TEXT] is RX, MOD_RODATA is RO, MOD_DATA is NX | `ModuleLoader::complete_formation` |
| `ModuleLoader::do_init_module` post: ret==0 ⟹ state LIVE ∧ MOD_INIT_* base==NULL | `ModuleLoader::do_init_module` |
| `Module::try_get` post: ret ⟹ refcnt ≥ 1 ∧ state == LIVE | `Module::try_get` |
| `Module::put` post: refcnt > 0 pre-call | `Module::put` |
| `Module::free` post: !mod ∈ modules ∧ all mem[*].base freed | `Module::free` |
| `ModuleSig::check` post: sig_ok ⟹ sig verified by trusted key | `ModuleSig::check` |

### Layer 4: Verus/Creusot functional

`Per-init_module / per-finit_module → load_module pipeline (sig → elf_validity → early_check → layout_and_allocate → add_unformed → percpu_modalloc → unload_init → find_module_sections → simplify_symbols → apply_relocations → complete_formation → prepare_coming → mod_sysfs_setup → do_init_module → state LIVE) → per-delete_module pipeline (refcnt check → state GOING → exit → notifier → free_module → wake_up)` semantic equivalence: per-Documentation/admin-guide/module-signing.rst + Documentation/kbuild/modules.rst.

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Module-loader reinforcement:

- **Per-CAP_SYS_MODULE + modules_disabled** — defense against per-unprivileged module-injection.
- **Per-module_sig_check + sig_enforce** — defense against per-untrusted-code execution.
- **Per-elf_validity_cache strict bounds** — defense against per-malformed-ELF (out-of-bounds sh_offset, oversized e_shnum, truncated strtab).
- **Per-blacklist** — defense against per-known-bad module reload.
- **Per-modstruct_version + vermagic check** — defense against per-ABI-mismatch.
- **Per-MODULE_INIT_IGNORE_VERMAGIC requires force** — defense against per-cross-kernel module.
- **Per-rodata_ro / data_nx / text_rox enforcement post-formation** — defense against per-W^X violation.
- **Per-MOD_RO_AFTER_INIT flipped to RO after init** — defense against per-late-stage data tampering.
- **Per-CFI finalize (module_cfi_finalize)** — defense against per-indirect-call-target hijack.
- **Per-SHN_COMMON rejected** — defense against per-uninitialized-symbol overlap.
- **Per-duplicate __ksymtab rejected at complete_formation** — defense against per-symbol-shadow attack.
- **Per-GPL flag enforced at find_symbol** — defense against per-license-violation export grab.
- **Per-LOCKDOWN_KEXEC analog (LOCKDOWN_MODULE_PARAMETERS)** — defense against per-unsafe-parameter-injection.
- **Per-idempotent finit_module** — defense against per-double-load TOCTOU + per-DoS via re-load loops.
- **Per-module_mutex protects list mutation; list traversal under RCU** — defense against per-load/unload race UAF.
- **Per-percpu_modalloc separate alloc** — defense against per-percpu-out-of-bounds.
- **Per-init_free_list / do_free_init via workqueue** — defense against per-execmem free-in-interrupt.
- **Per-augmented taints (OOT / PROPRIETARY / CRAP / UNSIGNED / TEST / LIVEPATCH)** — defense against per-silent-trust degradation.
- **Per-request_module_nowait throttle (kmod_concurrent)** — defense against per-modprobe fork-bomb.
- **Per-audit_log_kern_module** — defense against per-stealth load.

