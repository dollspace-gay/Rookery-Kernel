# Tier-3: kernel/trace/ftrace.c — Function tracer (dyn_ftrace records + mcount/fentry patching + ftrace_ops + filters)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/trace/00-overview.md
upstream-paths:
  - kernel/trace/ftrace.c (~9426 lines)
  - kernel/trace/fgraph.c
  - kernel/trace/ftrace_internal.h
  - include/linux/ftrace.h
  - arch/x86/kernel/ftrace.c (arch_ftrace_update_code, ftrace_make_call/nop/modify_call)
  - arch/x86/kernel/ftrace_64.S (ftrace_caller, ftrace_regs_caller, ftrace_graph_caller, return_to_handler)
-->

## Summary

`kernel/trace/ftrace.c` is the **function tracer** core — the machinery that turns each compiler-inserted `mcount` / `__fentry__` call-site into a runtime-patchable hook. Per-build the compiler emits a 5-byte (or 4-byte on aarch64) call to `__fentry__`/`mcount` at every function entry; the `recordmcount` / build-time `-mfentry -mrecord-mcount` pass collects all call-site IPs into the `__mcount_loc` ELF section. On boot, `ftrace_init()` reads `__start_mcount_loc..__stop_mcount_loc`, allocates one `struct dyn_ftrace { ip, flags }` per call-site (packed into `struct ftrace_page` arrays), sorts/dedupes them, and patches every call site to a NOP (`ftrace_make_nop` / `ftrace_init_nop`). Per-callback registration: an `ftrace_ops { func, flags, func_hash { filter_hash, notrace_hash }, trampoline, ... }` is added via `register_ftrace_function(ops)` → `ftrace_startup(ops, 0)` → `ftrace_hash_rec_enable(ops)` (mark every `dyn_ftrace` whose ip ∈ filter_hash ∧ ip ∉ notrace_hash with a ref-count increment + `FTRACE_FL_*` flags) → `ftrace_run_update_code(command)` → `arch_ftrace_update_code()` (default `stop_machine(__ftrace_modify_code)`) → walk every dyn_ftrace, compute new vs current address via `ftrace_get_addr_new` / `_curr`, dispatch to `ftrace_make_call` / `ftrace_make_nop` / `ftrace_modify_call`. Per-record `FTRACE_FL_*` flags: ENABLED, REGS / REGS_EN, TRAMP / TRAMP_EN (per-ops trampoline), IPMODIFY (livepatch's KLP_FUNC), DIRECT / DIRECT_EN (BPF direct dispatch), CALL_OPS / CALL_OPS_EN, DISABLED, TOUCHED, MODIFIED. Per-filter: `set_ftrace_filter` writes an allow-list; `set_ftrace_notrace` writes a deny-list; both parse glob patterns matched against `kallsyms` resolved via `ftrace_match_record`. Per-`set_ftrace_pid` / `set_ftrace_notrace_pid` writes the per-instance `function_pids` / `_no_pids` `trace_pid_list`, gated through `ftrace_pid_func()` which short-circuits when current's pid is filtered. Per-`set_graph_function` / `set_graph_notrace` filter the function-graph tracer (which uses `fgraph.c` to hook return addresses via a per-task shadow stack of `struct ftrace_ret_stack`). Per-livepatch interaction: KLP installs an `ftrace_ops` with `FTRACE_OPS_FL_IPMODIFY` to actually rewrite the saved `pt_regs->ip` from the trampoline so when the trampoline returns it lands in the patched function — only one IPMODIFY ops per ip allowed; DIRECT and IPMODIFY interaction is mediated via `ops_func(op, ip, FTRACE_OPS_CMD_ENABLE_SHARE_IPMODIFY_PEER)`. Per-`ftrace_kill()` is the fault-handler ejector: sets `ftrace_disabled = 1` permanently — no further patches.

This Tier-3 covers `kernel/trace/ftrace.c` (~9426 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dyn_ftrace` | per-call-site record | `DynFtrace` |
| `struct ftrace_page` | array of dyn_ftrace (alloc unit) | `FtracePage` |
| `struct ftrace_ops` | per-callback registration | `FtraceOps` |
| `struct ftrace_ops_hash` | filter+notrace hash pair | `FtraceOpsHash` |
| `struct ftrace_hash` | per-hash bucket table | `FtraceHash` |
| `struct ftrace_func_entry` | per-ip-in-hash node | `FtraceFuncEntry` |
| `struct ftrace_rec_iter` | per-record walker | `FtraceRecIter` |
| `struct ftrace_glob` | per-match pattern | `FtraceGlob` |
| `struct ftrace_iterator` | per-tracefs seq_file iter | `FtraceIterator` |
| `struct ftrace_func_mapper` | per-ip → per-data hash | `FtraceFuncMapper` |
| `struct ftrace_func_probe` | per-probe (deprecated path) | `FtraceFuncProbe` |
| `ftrace_init()` | per-boot read __mcount_loc | `Ftrace::init` |
| `ftrace_process_locs()` | per-callsite record creation + sort + nop init | `Ftrace::process_locs` |
| `ftrace_allocate_pages()` / `_free_pages()` | per-ftrace_page slab | `Ftrace::allocate_pages` / `_free_pages` |
| `ftrace_allocate_records()` | per-ftrace_page records init | `Ftrace::allocate_records` |
| `ftrace_update_code()` | per-module-load record nop-init | `Ftrace::update_code` |
| `ftrace_nop_initialize()` | per-record call ⟹ nop on init | `Ftrace::nop_initialize` |
| `register_ftrace_function()` | per-ops register | `Ftrace::register_function` |
| `unregister_ftrace_function()` | per-ops unregister | `Ftrace::unregister_function` |
| `__register_ftrace_function()` | inner | `Ftrace::register_function_inner` |
| `__unregister_ftrace_function()` | inner | `Ftrace::unregister_function_inner` |
| `ftrace_startup()` | per-ops install hashes ⟹ patch | `FtraceOps::startup` |
| `ftrace_shutdown()` | per-ops uninstall hashes ⟹ unpatch | `FtraceOps::shutdown` |
| `ftrace_startup_subops()` / `_shutdown_subops()` | per-subops (subset of parent ops) | `FtraceOps::startup_subops` / `_shutdown_subops` |
| `ftrace_modify_all_code()` | per-command: update calls / start-graph / stop-graph | `Ftrace::modify_all_code` |
| `ftrace_replace_code()` | walk all records + patch | `Ftrace::replace_code` |
| `__ftrace_replace_code()` | per-record patch | `Ftrace::replace_code_one` |
| `ftrace_check_record()` | per-record FTRACE_FL diff | `DynFtrace::check_record` |
| `ftrace_update_record()` | per-record commit FTRACE_FL diff | `DynFtrace::update_record` |
| `ftrace_test_record()` | per-record dry-run check | `DynFtrace::test_record` |
| `ftrace_get_addr_new()` / `_curr()` | per-record target addr (new vs current) | `DynFtrace::addr_new` / `addr_curr` |
| `ftrace_make_call()` / `_make_nop()` / `_modify_call()` | arch primitives (per-record) | `arch::Ftrace::make_call` / `make_nop` / `modify_call` |
| `ftrace_init_nop()` | arch per-record nop install | `arch::Ftrace::init_nop` |
| `arch_ftrace_update_code()` | arch dispatcher (`__weak` ⟹ `ftrace_run_stop_machine`) | `arch::Ftrace::update_code` |
| `ftrace_run_stop_machine()` | per-stop_machine wrap | `Ftrace::run_stop_machine` |
| `__ftrace_modify_code()` | stop_machine callback | `Ftrace::modify_code_inner` |
| `ftrace_run_update_code()` | per-prepare + arch_update + post-process | `Ftrace::run_update_code` |
| `ftrace_arch_code_modify_prepare()` / `_post_process()` | per-arch hooks (text_poke mode, ROX→RW) | `arch::Ftrace::code_modify_{prepare,post_process}` |
| `update_ftrace_function()` | rebuild `ftrace_trace_function` global | `Ftrace::update_trace_function` |
| `ftrace_ops_list_func()` | global dispatcher when ≥ 2 ops | `Ftrace::ops_list_func` |
| `ftrace_pid_func()` | wrapper enforcing per-pid filter | `Ftrace::pid_func` |
| `ftrace_stub` | the no-op terminator | `Ftrace::stub` |
| `ftrace_caller` / `ftrace_regs_caller` | arch entry trampoline | `arch::Ftrace::caller` / `regs_caller` |
| `ftrace_ops_get_func()` / `_get_list_func()` | per-ops dispatch selection | `FtraceOps::get_func` / `get_list_func` |
| `ftrace_update_trampoline()` | per-ops alloc/free per-op trampoline | `FtraceOps::update_trampoline` |
| `ftrace_hash_rec_enable()` / `_disable()` | per-ops record refcount + flag update | `FtraceOps::hash_rec_enable` / `_disable` |
| `__ftrace_hash_rec_update()` | per-ops walk hash, set rec flags + refcount | `FtraceOps::hash_rec_update` |
| `ftrace_hash_ipmodify_enable()` / `_disable()` / `_update()` | per-IPMODIFY exclusion bookkeeping | `FtraceOps::ipmodify_enable` / `_disable` / `_update` |
| `__ftrace_hash_update_ipmodify()` | per-walk: enforce single IPMODIFY per ip | `FtraceOps::hash_update_ipmodify` |
| `ftrace_set_filter()` / `_notrace()` | per-ops string set | `FtraceOps::set_filter` / `set_notrace` |
| `ftrace_set_filter_ip()` / `_filter_ips()` | per-ops ip-array set | `FtraceOps::set_filter_ip` / `set_filter_ips` |
| `ftrace_set_global_filter()` / `_notrace()` | global_ops shortcut | `Ftrace::set_global_filter` / `set_global_notrace` |
| `ftrace_set_hash()` | per-hash assign | `FtraceOps::set_hash` |
| `ftrace_set_regex()` | per-string glob set | `FtraceOps::set_regex` |
| `ftrace_match()` / `ftrace_match_record()` | per-glob match | `Ftrace::match` / `match_record` |
| `match_records()` | per-pattern walk all records | `Ftrace::match_records` |
| `ftrace_match_records()` | inner | `Ftrace::match_records_inner` |
| `register_ftrace_function_probe()` | per-glob+probe (deprecated tracefs `set_ftrace_filter`-with-trigger) | `Ftrace::register_function_probe` |
| `unregister_ftrace_function_probe_func()` | per-probe-unreg | `Ftrace::unregister_function_probe_func` |
| `clear_ftrace_function_probes()` | per-instance clear | `Ftrace::clear_function_probes` |
| `register_ftrace_command()` / `unregister_ftrace_command()` | per-named ftrace probe command (`enable`, `disable`, `dump`, `cpudump`, `stacktrace`) | `Ftrace::register_command` / `unregister_command` |
| `ftrace_pid_follow_fork()` | per-instance: follow forks into pid filter | `FtraceArray::pid_follow_fork` |
| `clear_ftrace_pids()` | per-instance clear | `FtraceArray::clear_pids` |
| `ftrace_pid_open()` / `ftrace_no_pid_open()` / `pid_write()` | tracefs handlers | `FtraceArray::pid_open` / `no_pid_open` / `pid_write` |
| `register_ftrace_direct()` / `unregister_ftrace_direct()` | per-BPF-direct registration | `Ftrace::register_direct` / `unregister_direct` |
| `modify_ftrace_direct()` / `_nolock()` | per-direct-addr swap | `Ftrace::modify_direct` / `modify_direct_nolock` |
| `update_ftrace_direct_add/del/mod()` | per-multi-direct hash update | `Ftrace::direct_add` / `del` / `mod` |
| `prepare_direct_functions_for_ipmodify()` / `cleanup_direct_functions_after_ipmodify()` | per-IPMODIFY-vs-DIRECT mediator | `Ftrace::prepare_direct_for_ipmodify` / `cleanup_direct_after_ipmodify` |
| `register_ftrace_graph()` / `unregister_ftrace_graph()` (fgraph.c) | per-graph hook install (entry + return) | `Fgraph::register` / `unregister` |
| `ftrace_graph_set_hash()` | per-`set_graph_function` write | `Fgraph::set_hash` |
| `ftrace_enable_ftrace_graph_caller()` / `_disable()` (arch) | enable arch `ftrace_graph_caller` | `arch::Fgraph::enable_caller` / `disable_caller` |
| `prepare_ftrace_return()` (arch helper called from `ftrace_graph_caller`) | per-entry: push to shadow stack, return `return_to_handler` | `arch::Fgraph::prepare_return` |
| `return_to_handler()` (arch asm stub) | per-exit: pop shadow stack, invoke retfunc, jmp to real ret | `arch::Fgraph::return_to_handler` |
| `ftrace_text_reserved()` | per-arch text-reservation check | `Ftrace::text_reserved` |
| `ftrace_location_range()` / `ftrace_location()` | per-ip ⟹ rec lookup | `Ftrace::location_range` / `location` |
| `ftrace_lookup_ip()` | per-hash lookup | `Ftrace::lookup_ip` |
| `alloc_ftrace_hash()` / `free_ftrace_hash()` | per-hash alloc/free | `FtraceHash::alloc` / `free` |
| `alloc_and_copy_ftrace_hash()` | per-hash dup | `FtraceHash::dup` |
| `ftrace_hash_move()` | per-ops swap hash | `FtraceOps::hash_move` |
| `ftrace_release_mod()` | per-module-unload free records | `Ftrace::release_mod` |
| `ftrace_module_init()` / `_enable()` | per-module-load callbacks | `Ftrace::module_init` / `module_enable` |
| `ftrace_free_init_mem()` | per-`free_initmem` cleanup | `Ftrace::free_init_mem` |
| `ftrace_free_mem()` | per-mod free range | `Ftrace::free_mem` |
| `ftrace_kill()` / `ftrace_is_dead()` | per-fault permanent disable | `Ftrace::kill` / `is_dead` |
| `ftrace_lookup_symbols()` | per-sorted-name array lookup | `Ftrace::lookup_symbols` |
| `ftrace_enable_sysctl()` | sysctl `kernel.ftrace_enabled` writer | `Ftrace::enable_sysctl` |
| `function_profile_enabled` / `function_stat_show()` | per-function-profiler tracefs | `Ftrace::profile_enabled` / `stat_show` |
| `ftrace_bug()` | per-fault dump | `Ftrace::bug` |
| `FTRACE_FL_ENABLED` (1<<31) | per-record: at least one ops references | flag |
| `FTRACE_FL_REGS` (1<<30) | per-record: some referring ops wants SAVE_REGS | flag |
| `FTRACE_FL_REGS_EN` (1<<29) | per-record: SAVE_REGS variant currently installed | flag |
| `FTRACE_FL_TRAMP` (1<<28) | per-record: dedicated trampoline desired | flag |
| `FTRACE_FL_TRAMP_EN` (1<<27) | per-record: dedicated trampoline installed | flag |
| `FTRACE_FL_IPMODIFY` (1<<26) | per-record: an ops will rewrite `regs->ip` | flag |
| `FTRACE_FL_DISABLED` (1<<25) | per-record: skip (weak fn / kernel init free / kallsyms invalid) | flag |
| `FTRACE_FL_DIRECT` (1<<24) | per-record: a DIRECT ops desires direct dispatch | flag |
| `FTRACE_FL_DIRECT_EN` (1<<23) | per-record: direct currently installed | flag |
| `FTRACE_FL_CALL_OPS` (1<<22) | per-record: arch will pass ops via call-ops trampoline | flag |
| `FTRACE_FL_CALL_OPS_EN` (1<<21) | per-record: call-ops currently installed | flag |
| `FTRACE_FL_TOUCHED` (1<<20) | per-record: at least once enabled (sticky) | flag |
| `FTRACE_FL_MODIFIED` (1<<19) | per-record: was patched after initial nop | flag |
| `FTRACE_UPDATE_*` enum | per-record patch action (IGNORE / MAKE_CALL / MAKE_NOP / MODIFY_CALL) | enum |

## Compatibility contract

REQ-1: `struct dyn_ftrace`:
- `ip`: instruction pointer at the call site (post-`ftrace_call_adjust`).
- `flags`: bitfield: low bits = refcount (number of enabled ops referencing this rec); upper bits = `FTRACE_FL_*` (ENABLED, REGS, REGS_EN, TRAMP, TRAMP_EN, IPMODIFY, DISABLED, DIRECT, DIRECT_EN, CALL_OPS, CALL_OPS_EN, TOUCHED, MODIFIED).
- `arch`: optional arch-private (currently unused on x86).

REQ-2: `struct ftrace_page`:
- `next`: link in singly-linked list rooted at `ftrace_pages_start`.
- `records`: variable-length `struct dyn_ftrace records[]`.
- `index`: # of records currently filled.
- `order`: page order of this allocation.
- `size`: capacity (`ENTRIES_PER_PAGE_GROUP(order)`).
- Allocated in groups (max 1 page each, possibly multi-page for densely-traced kernels) by `ftrace_allocate_pages()`.

REQ-3: `struct ftrace_ops`:
- `func`: callback `void (*)(unsigned long ip, unsigned long parent_ip, struct ftrace_ops*, struct ftrace_regs*)`.
- `saved_func`: copy of `func` (so pid wrappers can be re-assigned without losing user's func).
- `next`: RCU-linked list `ftrace_ops_list`.
- `flags`: `FTRACE_OPS_FL_*` (DYNAMIC, SAVE_REGS, SAVE_REGS_IF_SUPPORTED, RECURSION, RCU, INITIALIZED, ENABLED, ADDING, REMOVING, MODIFYING, ALLOC_TRAMP, PID, PERMANENT, IPMODIFY, DIRECT, STUB, CONTROL, ...).
- `local_hash`: embedded `struct ftrace_ops_hash` (used when ops doesn't share hash).
- `func_hash`: pointer to active `struct ftrace_ops_hash { filter_hash, notrace_hash, regex_lock }`.
- `old_hash`: snapshot during modify (so the trampoline check sees consistent state across the patch transition).
- `trampoline`: arch-allocated trampoline address (when ALLOC_TRAMP).
- `trampoline_size`: bytes of trampoline image.
- `subop_list`: list of registered subops for nested filter parents.
- `managed`: parent if this is a subop.
- `ops_func`: optional `int (*)(struct ftrace_ops*, unsigned long ip, enum ftrace_ops_cmd)` (IPMODIFY-vs-DIRECT mediation).
- `private`: ops-private (e.g., `trace_array*` for PID-aware ops).

REQ-4: `struct ftrace_hash`:
- `size_bits`: log2 of bucket count.
- `buckets`: `hlist_head` array of `2^size_bits`.
- `count`: total entries.
- `flags`: `FTRACE_HASH_FL_MOD` if module-relative pending entries.
- `rcu`: deferred free head.
- `EMPTY_HASH` sentinel: trace-all (when filter_hash) or trace-none (when notrace_hash); empty means "no filter".

REQ-5: `ftrace_init()`:
- `local_irq_save(flags)`; `ftrace_dyn_arch_init()` (no-op on x86, validates on others); restore IRQs.
- On error ⟹ `ftrace_disabled = 1`.
- `count = __stop_mcount_loc - __start_mcount_loc`.
- If `count == 0` ⟹ `ftrace_disabled = 1`.
- `ftrace_process_locs(NULL, __start_mcount_loc, __stop_mcount_loc)`.
- `last_ftrace_enabled = ftrace_enabled = 1`.
- `set_ftrace_early_filters()`.

REQ-6: `ftrace_process_locs(mod, start, end)`:
- `count = end - start`.
- Per-sort: if `!CONFIG_BUILDTIME_MCOUNT_SORT || mod`: `sort(start, count, ftrace_cmp_ips)`; else verify build-time sort with `test_is_sorted`.
- `start_pg = ftrace_allocate_pages(count, &pages)`.
- `mutex_lock(&ftrace_lock)`.
- If `!mod`: this is vmlinux; assert first-call (`!ftrace_pages`); set `ftrace_pages = ftrace_pages_start = start_pg`.
- Else: append `start_pg` to tail of existing list.
- For each `p in [start, end)`:
  - `addr = *p++`.
  - If `addr == 0`: skip (linker padding).
  - If `!mod && !(is_kernel_text(addr) || is_kernel_inittext(addr))`: skip (weak fn zeroed by KASLR/linker).
  - `addr = ftrace_call_adjust(addr)` (arch tweak: x86 returns `addr+5` to point at the byte after the CALL; aarch64 returns unchanged).
  - Append `{ ip=addr, flags=0 }` into `pg->records[pg->index++]`; advance to next page when full.
- Trim unused tail page (`pg_unuse`).
- For !mod: `local_irq_save(flags)`; `ftrace_update_code(mod, start_pg)`; `local_irq_restore(flags)`.
- For mod: no IRQ save (module not yet runnable from other CPUs).
- `synchronize_rcu()` before freeing `pg_unuse` (since `ftrace_location_range` may have raced).
- `pr_info("ftrace: allocating %ld entries in %ld pages\n", ...)`.

REQ-7: `ftrace_update_code(mod, new_pgs)`:
- For each rec in new_pgs: `ftrace_nop_initialize(mod, rec)` ⟹ `ftrace_init_nop(mod, rec)` (arch primitive: overwrite the build-time call with a NOP). On failure: `ftrace_bug_type = FTRACE_BUG_INIT`; call `ftrace_bug(rec)`; return.
- Skip records whose ip is in `kernel_text + free_initmem` if applicable.
- Optionally clear `FTRACE_FL_DISABLED` for unreferenced weak functions.
- Update `ftrace_number_of_pages` + `ftrace_number_of_groups` accounting.

REQ-8: `register_ftrace_function(ops)`:
- `lock_direct_mutex()` (only if DIRECT-CALLS enabled, else no-op).
- `prepare_direct_functions_for_ipmodify(ops)`:
  - If `ops->flags & FTRACE_OPS_FL_IPMODIFY`: walk `ops->func_hash->filter_hash`; for each ip, if any DIRECT ops references it: call `op->ops_func(op, ip, FTRACE_OPS_CMD_ENABLE_SHARE_IPMODIFY_PEER)`; if returns 0 ⟹ OK; else propagate error.
  - Else: return 0.
- `register_ftrace_function_nolock(ops)`:
  - `ftrace_ops_init(ops)` (lazy mutex-init + initial flags).
  - `mutex_lock(&ftrace_lock)`.
  - `ret = ftrace_startup(ops, 0)`.
  - `mutex_unlock(&ftrace_lock)`.
- `unlock_direct_mutex()`.

REQ-9: `__register_ftrace_function(ops)` (inner, called by `ftrace_startup`):
- `if (ops->flags & FTRACE_OPS_FL_DELETED)` ⟹ `-EINVAL`.
- `WARN_ON` if `ENABLED` already.
- Validate `SAVE_REGS` arch availability; if no SAVE_REGS arch support and ops requires (not `_IF_SUPPORTED`): `-EINVAL`.
- If `!ftrace_enabled && (ops->flags & PERMANENT)`: `-EBUSY`.
- If ops is not in core data (module-allocated): set `FTRACE_OPS_FL_DYNAMIC`.
- `add_ftrace_ops(&ftrace_ops_list, ops)` (RCU-safe head insert).
- `ops->saved_func = ops->func`.
- If `ftrace_pids_enabled(ops)`: `ops->func = ftrace_pid_func`.
- `ftrace_update_trampoline(ops)` (alloc / re-alloc per-ops trampoline if requested).
- If `ftrace_enabled`: `update_ftrace_function()` (recompute `ftrace_trace_function` global).

REQ-10: `ftrace_startup(ops, command)`:
- `if (ftrace_disabled)` ⟹ `-ENODEV`.
- `__register_ftrace_function(ops)`.
- `ftrace_start_up++`.
- `ops->flags |= FTRACE_OPS_FL_ENABLED | FTRACE_OPS_FL_ADDING`.
- `ftrace_hash_ipmodify_enable(ops)`:
  - If `ops->flags & IPMODIFY`: walk filter_hash; for each ip, check no other ops on this ip already IPMODIFY ⟹ else `-EBUSY` rollback.
  - Set `FTRACE_FL_IPMODIFY` on each rec.
  - On failure: undo `__register_ftrace_function`, decrement `ftrace_start_up`, free trampoline.
- `ftrace_hash_rec_enable(ops)`:
  - `__ftrace_hash_rec_update(ops, true)`: walk every rec; if rec.ip ∈ filter_hash ∧ rec.ip ∉ notrace_hash: increment rec.flags low-bits (refcount) and set `FTRACE_FL_REGS` / `FTRACE_FL_TRAMP` / `FTRACE_FL_DIRECT` / `FTRACE_FL_CALL_OPS` if ops requests; return true if any rec needs `FTRACE_UPDATE_CALLS`.
- If hash_rec_enable returned true: `command |= FTRACE_UPDATE_CALLS`.
- `ftrace_startup_enable(command)`:
  - If `saved_ftrace_func != ftrace_trace_function`: `command |= FTRACE_UPDATE_TRACE_FUNC; saved_ftrace_func = ftrace_trace_function`.
  - If `!command || !ftrace_enabled`: return.
  - `ftrace_run_update_code(command)`.
- `ops->flags &= ~FTRACE_OPS_FL_ADDING`.

REQ-11: `ftrace_run_update_code(command)`:
- `ftrace_arch_code_modify_prepare()` (e.g., x86 transitions trampoline page from ROX to RW via text_poke setup).
- `arch_ftrace_update_code(command)` (`__weak` default ⟹ `ftrace_run_stop_machine(command)`).
- `ftrace_arch_code_modify_post_process()` (x86 transitions back to ROX, sync IPI).

REQ-12: `ftrace_run_stop_machine(command)`:
- `stop_machine(__ftrace_modify_code, &command, NULL)` — every CPU is parked outside kernel text execution; one CPU runs `__ftrace_modify_code` ⟹ `ftrace_modify_all_code(command)`.

REQ-13: `ftrace_modify_all_code(command)`:
- `update = command & FTRACE_UPDATE_TRACE_FUNC`.
- `mod_flags = (command & FTRACE_MAY_SLEEP) ? FTRACE_MODIFY_MAY_SLEEP_FL : 0`.
- If `update`: temporarily set `ftrace_trace_function = ftrace_ops_list_func` via `update_ftrace_func(...)` so any pending call hits the safe dispatcher.
- If `command & FTRACE_UPDATE_CALLS`: `ftrace_replace_code(mod_flags | FTRACE_MODIFY_ENABLE_FL)`.
- Else if `command & FTRACE_DISABLE_CALLS`: `ftrace_replace_code(mod_flags)`.
- If `update && ftrace_trace_function != ftrace_ops_list_func`:
  - `function_trace_op = set_function_trace_op`.
  - `smp_wmb()`.
  - If `!irqs_disabled()`: `smp_call_function(ftrace_sync_ipi, NULL, 1)` (IPI everyone so they see the new `function_trace_op` before they see the new `ftrace_trace_function`).
  - `update_ftrace_func(ftrace_trace_function)` (arch installs the new direct dispatcher, e.g., overwriting the single CALL in `ftrace_caller`).
- If `command & FTRACE_START_FUNC_RET`: `ftrace_enable_ftrace_graph_caller()` (arch patches `ftrace_graph_call` site).
- Else if `command & FTRACE_STOP_FUNC_RET`: `ftrace_disable_ftrace_graph_caller()`.

REQ-14: `ftrace_replace_code(mod_flags)`:
- `enable = mod_flags & FTRACE_MODIFY_ENABLE_FL`.
- `schedulable = mod_flags & FTRACE_MODIFY_MAY_SLEEP_FL`.
- `if (ftrace_disabled) return`.
- `do_for_each_ftrace_rec(pg, rec)`:
  - If `skip_record(rec)` (DISABLED + !ENABLED): continue.
  - `failed = __ftrace_replace_code(rec, enable)`.
  - If failed: `ftrace_bug(failed, rec)`; return.
  - If schedulable: `cond_resched()`.

REQ-15: `__ftrace_replace_code(rec, enable)`:
- `ftrace_addr = ftrace_get_addr_new(rec)`:
  - If `rec.flags & FTRACE_FL_DIRECT` ∧ `ftrace_rec_count(rec) == 1`: return `ftrace_find_rec_direct(rec.ip)` (direct target).
  - Else if `rec.flags & FTRACE_FL_TRAMP`: return `ftrace_find_tramp_ops_new(rec)->trampoline`.
  - Else if `rec.flags & FTRACE_FL_REGS`: return `FTRACE_REGS_ADDR` (ftrace_regs_caller).
  - Else: return `FTRACE_ADDR` (ftrace_caller).
- `ftrace_old_addr = ftrace_get_addr_curr(rec)` (uses `_EN` flags — current state).
- `ret = ftrace_update_record(rec, enable)` (commits flag transitions; returns FTRACE_UPDATE_IGNORE / _MAKE_CALL / _MAKE_NOP / _MODIFY_CALL).
- Dispatch:
  - `_IGNORE` ⟹ 0.
  - `_MAKE_CALL` ⟹ `ftrace_make_call(rec, ftrace_addr)`.
  - `_MAKE_NOP` ⟹ `ftrace_make_nop(NULL, rec, ftrace_old_addr)`.
  - `_MODIFY_CALL` ⟹ `ftrace_modify_call(rec, ftrace_old_addr, ftrace_addr)`.
- Set `ftrace_bug_type` per branch (CALL / NOP / UPDATE).

REQ-16: `ftrace_check_record(rec, enable, update)` (returns FTRACE_UPDATE_*):
- If `skip_record(rec)`: return `_IGNORE`.
- `flag = 0`.
- If `enable && ftrace_rec_count(rec)`: `flag = FTRACE_FL_ENABLED`.
- If `flag`:
  - Diff `REGS` vs `REGS_EN` ⟹ set `flag |= FTRACE_FL_REGS`.
  - Diff `TRAMP` vs `TRAMP_EN` ⟹ set `flag |= FTRACE_FL_TRAMP`.
  - If `rec_count == 1`: diff `DIRECT` vs `DIRECT_EN` ⟹ set `flag |= DIRECT`. Else if `DIRECT_EN`: `flag |= DIRECT`.
  - If `rec_count == 1`: diff `CALL_OPS` vs `CALL_OPS_EN` ⟹ set `flag |= CALL_OPS`. Else if `CALL_OPS_EN`: `flag |= CALL_OPS`.
- If `(rec.flags & ENABLED) == flag`: return `_IGNORE` (nothing changed).
- If `flag`:
  - `flag ^= rec.flags & ENABLED` (extract diff bits).
  - If `update`:
    - `rec.flags |= ENABLED | TOUCHED`.
    - Apply REGS / TRAMP `_EN` mirror: `rec.flags ⊕= REGS_EN/TRAMP_EN` per current bit.
    - If rec.flags has DIRECT or IPMODIFY: set MODIFIED.
    - If `flag & DIRECT`: if `rec_count == 1`, mirror `DIRECT_EN`; else clear `DIRECT_EN`.
    - If `flag & CALL_OPS`: similarly mirror `CALL_OPS_EN`.
  - If `flag & ENABLED`: `ftrace_bug_type = FTRACE_BUG_CALL`; return `_MAKE_CALL`.
  - Else: `ftrace_bug_type = FTRACE_BUG_UPDATE`; return `_MODIFY_CALL`.
- Else (clearing):
  - If `update`:
    - If `!rec_count`: `rec.flags &= FTRACE_NOCLEAR_FLAGS` (keep DISABLED + TOUCHED + MODIFIED).
    - Else: `rec.flags &= ~(ENABLED | TRAMP_EN | REGS_EN | DIRECT_EN | CALL_OPS_EN)` (other ops still ref).
  - `ftrace_bug_type = FTRACE_BUG_NOP`; return `_MAKE_NOP`.

REQ-17: `ftrace_shutdown(ops, command)`:
- `if (ftrace_disabled)` ⟹ `-ENODEV`.
- `__unregister_ftrace_function(ops)`; decrement `ftrace_start_up`.
- `ftrace_hash_ipmodify_disable(ops)` (clears `FTRACE_FL_IPMODIFY` on touched recs).
- `ftrace_hash_rec_disable(ops)` ⟹ if any rec needs update: `command |= FTRACE_UPDATE_CALLS`.
- `ops->flags &= ~ENABLED`.
- If `saved_ftrace_func != ftrace_trace_function`: `command |= UPDATE_TRACE_FUNC; saved_ftrace_func = ftrace_trace_function`.
- If `!command || !ftrace_enabled`: skip-modify.
- `ops->flags |= REMOVING; removed_ops = ops`.
- `ops->old_hash.filter_hash = ops->func_hash->filter_hash; .notrace_hash = ops->func_hash->notrace_hash` (so the trampoline-finder sees pre-removal state).
- `ftrace_run_update_code(command)`.
- If `ftrace_ops_list` now equals `&ftrace_list_end`: walk all records; WARN if any rec retains flags outside `FTRACE_NOCLEAR_FLAGS`.
- Clear `ops->old_hash.*`; `removed_ops = NULL; ops->flags &= ~REMOVING`.
- If `ops->flags & DYNAMIC`:
  - `synchronize_rcu_tasks_rude()` (flush in-flight pre-noinstr handlers).
  - `synchronize_rcu_tasks()` (flush preempted-on-trampoline tasks).
  - `ftrace_trampoline_free(ops)`.

REQ-18: `set_ftrace_filter` / `set_ftrace_notrace` (tracefs writers):
- `ftrace_regex_write(file, ubuf, ...)` ⟹ parse glob lines via `trace_get_user`.
- For each line: `ftrace_process_regex(iter, buff)`:
  - If line is `<cmd>:<pattern>:<glob>:<arg>` ⟹ dispatch to a registered `struct ftrace_func_command` (`enable`, `disable`, `dump`, `cpudump`, `stacktrace`).
  - Else `ftrace_match_records(hash, buff, len)`:
    - Build `ftrace_glob { search, len, type=MATCH_FULL|FRONT_ONLY|MIDDLE_ONLY|END_ONLY }`.
    - Iterate `do_for_each_ftrace_rec`: if `ftrace_match_record(rec, &func_g, mod_g, ...) == 1`: `enter_record(hash, rec, clear_filter)`.
- On `ftrace_regex_release`: `ftrace_hash_move_and_update_ops(ops, &ops->func_hash, hash, enable)` ⟹ swap into live hash + `ftrace_run_modify_code(ops, command, &old_hash)`.
- All paths take `ftrace_lock` mutex for the patch transition.

REQ-19: `set_ftrace_pid` / `set_ftrace_notrace_pid` (per-instance):
- `pid_open(inode, file, type)` ⟹ `ftrace_pid_open` or `ftrace_no_pid_open` set up `tr->function_pids` / `_no_pids` builder.
- `pid_write(filp, ubuf, ...)`:
  - Parse pids/tgids/groups.
  - Build new `trace_pid_list`.
  - Swap into `tr->function_pids` / `_no_pids` under RCU.
  - Recompute `ftrace_pids_enabled(ops)` and toggle `ops->func` between `ops->saved_func` and `ftrace_pid_func`.
  - `on_each_cpu_mask(... ignore_task_cpu)` to update per-cpu `ftrace_ignore_pid` for fast filtering.
- `ftrace_pid_follow_fork(tr, enable)`: register tracepoints on `sched_process_fork` (auto-add child to pid list) and `sched_process_exit` (auto-remove).
- `ftrace_pid_func(ip, pip, op, fregs)`:
  - `tr = op->private`.
  - `pid = this_cpu_read(tr->array_buffer.data->ftrace_ignore_pid)`.
  - If `pid == FTRACE_PID_IGNORE`: return.
  - If `pid != FTRACE_PID_TRACE && pid != current->pid`: return.
  - `op->saved_func(ip, pip, op, fregs)`.

REQ-20: `set_graph_function` / `set_graph_notrace`:
- `ftrace_graph_open` ⟹ `__ftrace_graph_open(inode, file, enable)` ⟹ alloc hash, copy from `ftrace_graph_hash` or `_notrace_hash`.
- `ftrace_graph_write` ⟹ parse glob ⟹ `ftrace_graph_set_hash(hash, buffer)`:
  - `ftrace_match_records` style walk; matched ips into hash.
- `ftrace_graph_release` ⟹ swap into live hash under `graph_lock` mutex.
- `set_graph_max_depth_function` cmdline parses `ftrace_graph_max_depth=`.
- Boot cmdlines: `ftrace_graph_filter=`, `ftrace_graph_notrace=`.

REQ-21: Function-graph hook (per-fgraph.c):
- `register_ftrace_graph(gops)`:
  - Validate gops index ≤ FGRAPH_ARRAY_SIZE (16 in 7.1.0-rc2).
  - Install into `fgraph_array[index]` slot; allocate per-task shadow stacks (`current->ret_stack`).
  - Issue `ftrace_modify_all_code(... | FTRACE_START_FUNC_RET)` ⟹ `ftrace_enable_ftrace_graph_caller()` (arch flips the `ftrace_graph_call` site to a CALL of `ftrace_graph_caller`).
- Per-entry mechanism (x86): `ftrace_caller` falls through into `ftrace_graph_caller` when graph enabled ⟹ ASM stashes regs, calls `prepare_ftrace_return(parent_ip_addr, self_ip, frame_pointer)`:
  - `prepare_ftrace_return`: per-task `ftrace_push_return_trace(ret, func, &index, frame_pointer, &retp)` pushes `struct ftrace_ret_stack { ret, func, calltime, subtime, fp, retp, type }` onto `current->ret_stack` (capped at `FTRACE_RETFUNC_DEPTH`); replaces `*parent_ip_addr` with `return_to_handler`.
- Per-exit mechanism: when the traced function returns, it jumps to `return_to_handler` (asm stub) which pops the `ftrace_ret_stack`, calls each registered `gops->retfunc(&trace, gops, regs)`, restores the original return address into the right register, jmps to it.
- `unregister_ftrace_graph(gops)`: per-array clear; `FTRACE_STOP_FUNC_RET` if last; `synchronize_rcu_tasks` to flush in-flight `return_to_handler` walks; free per-task shadow stacks via `ftrace_shutdown_ret_stack_per_task`.

REQ-22: Livepatch interaction:
- KLP allocates an `ftrace_ops` per patched function with `FTRACE_OPS_FL_IPMODIFY | _SAVE_REGS` and `func = klp_ftrace_handler`.
- `klp_ftrace_handler` rewrites `regs->ip` (or `ftrace_regs->regs.ip`) to the new function address; when the trampoline returns it lands in the replacement.
- `__ftrace_hash_update_ipmodify` enforces: at most one IPMODIFY ops per rec.ip (else `-EBUSY` at register time).
- When a DIRECT ops also targets the same ip (BPF trampolines), `prepare_direct_functions_for_ipmodify` calls the direct ops's `ops_func(FTRACE_OPS_CMD_ENABLE_SHARE_IPMODIFY_PEER)` to coordinate; symmetric `cleanup_direct_functions_after_ipmodify` on unregister.

REQ-23: Module load / unload integration:
- `ftrace_module_init(mod)`: registered as `MODULE_STATE_COMING` handler; calls `ftrace_process_locs(mod, mod->mcount_loc_start, mod->mcount_loc_end)` to nop-init the module's call sites + allocate `dyn_ftrace` records for them.
- `ftrace_module_enable(mod)`: registered as `MODULE_STATE_LIVE`; re-walks `ftrace_pages` to enable any rec belonging to this module that matches a currently-active ops's hash.
- `ftrace_release_mod(mod)`: clears mod-owned records from every ops hash, removes the per-module `ftrace_page` group, RCU-frees.

REQ-24: `ftrace_kill()`:
- `ftrace_disabled = 1; ftrace_enabled = 0; ftrace_trace_function = ftrace_stub`.
- `kprobe_ftrace_kill()` (cascade kill kprobes-via-ftrace).
- Permanent; no API to re-enable. Used by `ftrace_bug` on irrecoverable patch fault.

REQ-25: `ftrace_text_reserved(start, end)`:
- Per-arch: returns true if any byte in [start, end) is a known ftrace call site OR within a known trampoline OR within the special caller sites (`ftrace_caller`, `ftrace_regs_caller`, `ftrace_graph_caller`).
- Used by jump_label / static_call / kprobes to refuse overlap.

REQ-26: `ftrace_location_range(start, end)` / `ftrace_location(ip)`:
- Per-binary-search of sorted `ftrace_pages` records.
- Returns the matched rec's `ip` (or 0 if none).
- `lookup_rec` is the inner helper (binary-search within a page group).

REQ-27: `function_profile_enabled` (CONFIG_FUNCTION_PROFILER):
- Echo 1 to `/sys/kernel/tracing/function_profile_enabled` ⟹ `ftrace_profile_write`:
  - Allocate per-cpu `ftrace_profile_stat` (hash-table of profile records).
  - `register_ftrace_profiler()` ⟹ install profile-callback ops; if FUNCTION_GRAPH_TRACER, also install graph entry/return.
- `function_stat_show(seq, v)`: per `trace_stat` framework prints per-function "Hits | Time | Avg" sorted by `function_stat_cmp` (per-counter sort).

REQ-28: Direct ftrace (BPF JIT trampolines):
- `register_ftrace_direct(ops, addr)`:
  - `check_direct_multi(ops)` validates filter cardinality.
  - Set `FTRACE_OPS_FL_DIRECT`.
  - For each ip in filter: add `(ip, addr)` pair to `direct_functions` hash; set `FTRACE_FL_DIRECT` on rec.
  - `register_ftrace_function_nolock(ops)`.
- `modify_ftrace_direct(ops, addr)`: under `direct_mutex` swap addr; `text_poke` updates trampoline call target.
- `update_ftrace_direct_add/del/mod(ops, hash)`: multi-ip update with rollback on partial failure.

REQ-29: `update_ftrace_function()`:
- Recompute global `ftrace_trace_function` based on ops list:
  - `set_function_trace_op = ftrace_ops_list` (head).
  - If list empty: `func = ftrace_stub`.
  - Else if list has one ops (`ops_list->next == &ftrace_list_end`): `func = ftrace_ops_get_list_func(ops_list)` (the single ops directly, or list-func if DYNAMIC/RCU).
  - Else: `set_function_trace_op = &ftrace_list_end; func = ftrace_ops_list_func`.
- If no change: return.
- If new func is `ftrace_ops_list_func`: write directly.
- Else (static-tracing path, no DYNAMIC_FTRACE): transition through `ftrace_ops_list_func`, `synchronize_rcu_tasks_rude`, set `function_trace_op`, `smp_wmb`, IPI all CPUs, then write new `func`.
- Finally `ftrace_trace_function = func`.

REQ-30: `ftrace_enable_sysctl(table, write, ...)`:
- `mutex_lock(&ftrace_lock)`.
- Read user value into `ftrace_enabled`.
- If transitioning 0 → 1: `ftrace_startup_sysctl()` (re-apply any saved state).
- If transitioning 1 → 0: if `is_permanent_ops_registered()` ⟹ `-EBUSY` (rollback). Else `ftrace_shutdown_sysctl()`.
- `mutex_unlock`.

## Acceptance Criteria

- [ ] AC-1: `ftrace_init()` reads `__mcount_loc`, sorts, allocates `dyn_ftrace` records, patches every site to NOP; `enabled_functions` is empty post-init.
- [ ] AC-2: `register_ftrace_function(ops)` with empty filter (trace all): every rec gets ref+1, `FTRACE_FL_ENABLED` set, every site patched to call `FTRACE_ADDR` (or `FTRACE_REGS_ADDR` if SAVE_REGS).
- [ ] AC-3: `unregister_ftrace_function(ops)`: refs decrement; recs with ref=0 patched back to NOP; if DYNAMIC, trampoline freed after `synchronize_rcu_tasks_rude + _tasks`.
- [ ] AC-4: `set_ftrace_filter` with `vfs_read` ⟹ only `vfs_read` site enabled; reading `enabled_functions` returns `vfs_read`.
- [ ] AC-5: `set_ftrace_notrace` with `vfs_read` AND filter set to all: every site except `vfs_read` enabled.
- [ ] AC-6: `set_ftrace_pid` with current's pid ⟹ traces ONLY current; other procs' calls skipped via `ftrace_pid_func`.
- [ ] AC-7: `current_tracer = function_graph` ⟹ `FTRACE_START_FUNC_RET` issued; `ftrace_graph_caller` patched in; entry/exit recorded via shadow stack.
- [ ] AC-8: `set_graph_function vfs_read` ⟹ only `vfs_read`'s descendants traced.
- [ ] AC-9: Two ops register with overlapping filter ⟹ rec.flags refcount = 2; unregister one ⟹ refcount = 1, rec stays enabled.
- [ ] AC-10: Two ops register both with IPMODIFY for same ip ⟹ second register returns `-EBUSY`.
- [ ] AC-11: DIRECT ops + IPMODIFY ops on same ip ⟹ if DIRECT's `ops_func` returns 0, both coexist; if returns `-EBUSY`, registration fails.
- [ ] AC-12: Module insmod ⟹ `ftrace_module_init` records new mcount sites; `ftrace_module_enable` may re-trace if filter matches.
- [ ] AC-13: Module rmmod ⟹ `ftrace_release_mod` removes records; `ftrace_location(<module_ip>)` returns 0 after RCU grace.
- [ ] AC-14: `ftrace_kill()` ⟹ `ftrace_is_dead()` returns 1; further `register_ftrace_function` returns `-ENODEV`.
- [ ] AC-15: `kernel.ftrace_enabled = 0` while a PERMANENT ops is registered ⟹ `-EBUSY`.
- [ ] AC-16: Patch transition uses stop_machine on arches without text_poke_bp; on x86 uses `arch_ftrace_update_code` with `text_poke_bp`.

## Architecture

```
struct DynFtrace {
  ip:    u64,
  flags: u64,                 // low bits = refcount, upper bits = FTRACE_FL_*
}

struct FtracePage {
  next:    *FtracePage,
  records: *mut DynFtrace,    // [size]
  index:   u32,
  size:    u32,
  order:   u8,
}

struct FtraceOps {
  func:           FtraceFunc,
  saved_func:     FtraceFunc,
  next:           *FtraceOps,            // RCU
  flags:          u32,                   // FTRACE_OPS_FL_*
  local_hash:     FtraceOpsHash,
  func_hash:      *FtraceOpsHash,
  old_hash:       FtraceOpsHash,
  trampoline:     u64,
  trampoline_size:u32,
  subop_list:     List<FtraceOps>,
  managed:        Option<*FtraceOps>,
  ops_func:       Option<fn(*FtraceOps, u64 ip, FtraceOpsCmd) -> i32>,
  private:        *mut (),
}

struct FtraceOpsHash {
  filter_hash:  *FtraceHash,
  notrace_hash: *FtraceHash,
  regex_lock:   Mutex<()>,
}

struct FtraceHash {
  size_bits: u8,
  buckets:   *mut HListHead,      // [1 << size_bits]
  count:     u32,
  flags:     u32,
  rcu:       RcuHead,
}

struct FtraceFuncEntry {
  hlist:  HListNode,
  ip:     u64,
  direct: u64,                    // only for direct_functions hash
}

enum FtraceUpdate {
  Ignore       = 0,
  MakeCall     = 1,
  MakeNop      = 2,
  ModifyCall   = 3,
}

bitflags FtraceFlRec : u64 {
  ENABLED     = 1 << 31,
  REGS        = 1 << 30,
  REGS_EN     = 1 << 29,
  TRAMP       = 1 << 28,
  TRAMP_EN    = 1 << 27,
  IPMODIFY    = 1 << 26,
  DISABLED    = 1 << 25,
  DIRECT      = 1 << 24,
  DIRECT_EN   = 1 << 23,
  CALL_OPS    = 1 << 22,
  CALL_OPS_EN = 1 << 21,
  TOUCHED     = 1 << 20,
  MODIFIED    = 1 << 19,
  // FTRACE_NOCLEAR_FLAGS = DISABLED | TOUCHED | MODIFIED
}
```

`Ftrace::init()`:
1. `local_irq_save(flags); ftrace_dyn_arch_init(); local_irq_restore(flags)`.
2. `count = __stop_mcount_loc - __start_mcount_loc`.
3. If `count == 0` ⟹ `ftrace_disabled = 1; return`.
4. `ftrace_process_locs(NULL, __start_mcount_loc, __stop_mcount_loc)`.
5. `last_ftrace_enabled = ftrace_enabled = 1`.
6. `set_ftrace_early_filters()` (applies boot cmdlines `ftrace_filter=`, `ftrace_notrace=`, `ftrace_graph_filter=`, `ftrace_graph_notrace=`).

`Ftrace::process_locs(mod, start, end)`:
1. `count = end - start`; if 0 return.
2. If `!BUILDTIME_MCOUNT_SORT || mod`: `sort(start, count, ftrace_cmp_ips)`; else `test_is_sorted`.
3. `start_pg = ftrace_allocate_pages(count, &pages)`; if NULL ⟹ `-ENOMEM`.
4. `guard(mutex)(&ftrace_lock)`.
5. Insert `start_pg` at head (vmlinux) or tail (module).
6. For each addr in [start, end):
   - Skip zero (linker padding).
   - For !mod: skip if not in `kernel_text/inittext` (KASLR-zeroed weak fn).
   - `addr = ftrace_call_adjust(addr)`.
   - Append `{ ip=addr, flags=0 }` into current `pg->records[pg->index++]`; advance `pg` if full.
7. Truncate unused tail; record `pg_unuse`.
8. Local IRQ-save (only !mod); `ftrace_update_code(mod, start_pg)`; restore.
9. If pg_unuse: `synchronize_rcu(); ftrace_free_pages(pg_unuse)`.

`FtraceOps::startup(command) -> i32`:
1. If `ftrace_disabled` ⟹ `-ENODEV`.
2. `__register_ftrace_function(self)` (RCU-insert into `ftrace_ops_list`, set `saved_func`, install pid wrapper if needed, update trampoline, recompute `ftrace_trace_function`).
3. `ftrace_start_up++`.
4. `self.flags |= ENABLED | ADDING`.
5. `ret = ftrace_hash_ipmodify_enable(self)`; on failure: undo register, decrement, free trampoline if DYNAMIC.
6. If `ftrace_hash_rec_enable(self)`: `command |= UPDATE_CALLS`.
7. `ftrace_startup_enable(command)`:
   - If `saved_ftrace_func != ftrace_trace_function`: `command |= UPDATE_TRACE_FUNC; saved_ftrace_func = ftrace_trace_function`.
   - If `!command || !ftrace_enabled`: return.
   - `ftrace_run_update_code(command)`:
     - `ftrace_arch_code_modify_prepare()`.
     - `arch_ftrace_update_code(command)` ⟹ default `stop_machine(__ftrace_modify_code, &command, NULL)` ⟹ `ftrace_modify_all_code(command)`.
     - `ftrace_arch_code_modify_post_process()`.
8. If `ftrace_disabled` post-update: `__unregister_ftrace_function(self); return -ENODEV`.
9. `self.flags &= ~ADDING`.
10. Return 0.

`Ftrace::modify_all_code(command)`:
1. `update = command & UPDATE_TRACE_FUNC`.
2. `mod_flags = (command & MAY_SLEEP) ? MAY_SLEEP_FL : 0`.
3. If `update`: `update_ftrace_func(ftrace_ops_list_func)` (force list-func during transition).
4. If `command & UPDATE_CALLS`: `ftrace_replace_code(mod_flags | MODIFY_ENABLE_FL)`.
5. Else if `command & DISABLE_CALLS`: `ftrace_replace_code(mod_flags)`.
6. If `update && ftrace_trace_function != ftrace_ops_list_func`:
   - `function_trace_op = set_function_trace_op; smp_wmb()`.
   - If `!irqs_disabled()`: `smp_call_function(ftrace_sync_ipi, NULL, 1)`.
   - `update_ftrace_func(ftrace_trace_function)`.
7. If `command & START_FUNC_RET`: `ftrace_enable_ftrace_graph_caller()`.
8. Else if `command & STOP_FUNC_RET`: `ftrace_disable_ftrace_graph_caller()`.

`Ftrace::replace_code_one(rec, enable) -> i32`:
1. `ftrace_addr = DynFtrace::addr_new(rec)` (direct → trampoline → regs_caller → caller).
2. `ftrace_old_addr = DynFtrace::addr_curr(rec)`.
3. `action = DynFtrace::update_record(rec, enable)` (commit flags + return FtraceUpdate).
4. Switch:
   - `Ignore` ⟹ 0.
   - `MakeCall` ⟹ `arch::ftrace_make_call(rec, ftrace_addr)`.
   - `MakeNop` ⟹ `arch::ftrace_make_nop(NULL, rec, ftrace_old_addr)`.
   - `ModifyCall` ⟹ `arch::ftrace_modify_call(rec, ftrace_old_addr, ftrace_addr)`.

`DynFtrace::addr_new(rec) -> u64`:
1. If `rec.flags & DIRECT && ftrace_rec_count(rec) == 1`: `addr = ftrace_find_rec_direct(rec.ip)`; return.
2. If `rec.flags & TRAMP`: return `ftrace_find_tramp_ops_new(rec).trampoline`.
3. If `rec.flags & REGS`: return `FTRACE_REGS_ADDR`.
4. Return `FTRACE_ADDR`.

`DynFtrace::addr_curr(rec) -> u64`:
1. If `rec.flags & DIRECT_EN`: `addr = ftrace_find_rec_direct(rec.ip)`; return.
2. If `rec.flags & TRAMP_EN`: return `ftrace_find_tramp_ops_curr(rec).trampoline`.
3. If `rec.flags & REGS_EN`: return `FTRACE_REGS_ADDR`.
4. Return `FTRACE_ADDR`.

`Ftrace::kill()`:
1. `ftrace_disabled = 1`.
2. `ftrace_enabled = 0`.
3. `ftrace_trace_function = ftrace_stub`.
4. `kprobe_ftrace_kill()`.

`Fgraph::register(gops)` (per-fgraph.c, summarized):
1. Allocate slot in `fgraph_array[]`; assign `gops->idx`.
2. Allocate per-task shadow stacks (`current->ret_stack`) for all tasks via `alloc_ret_stack_tasks`.
3. `ftrace_modify_all_code(FTRACE_START_FUNC_RET | UPDATE_TRACE_FUNC)`:
   - `arch::ftrace_enable_ftrace_graph_caller()` patches `ftrace_graph_call` site to call `ftrace_graph_caller`.
4. On any traced function:
   - `ftrace_caller` saves regs, calls `ftrace_ops_list_func` (or per-ops direct).
   - If `FTRACE_START_FUNC_RET` is set globally, `ftrace_caller` falls into `ftrace_graph_call` ⟹ `ftrace_graph_caller` ⟹ `prepare_ftrace_return(parent_ip_addr, self_ip, frame_pointer)`.
   - `prepare_ftrace_return`: `ftrace_push_return_trace(ret=*parent_ip_addr, func=self_ip, &index, fp, &retp)` pushes onto `current->ret_stack`; `*parent_ip_addr = return_to_handler`.
5. On traced function return:
   - Jumps to `return_to_handler` (arch asm); pops `ftrace_ret_stack`; invokes each registered `gops->retfunc(&trace, gops, regs)`; restores original `ret`; jmp ret.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dyn_ftrace_refcount_bounded` | INVARIANT | per-rec: low-bits refcount ≤ `FTRACE_REF_MAX` (1<<19 - 1); never wraps. |
| `flag_bit_disjoint` | INVARIANT | per-rec: `REGS_EN ⟹ REGS`; `TRAMP_EN ⟹ TRAMP`; `DIRECT_EN ⟹ DIRECT`; `CALL_OPS_EN ⟹ CALL_OPS`. |
| `ipmodify_singleton_per_ip` | INVARIANT | per-rec: at most one ops with IPMODIFY references this rec. |
| `nop_after_zero_refcount` | INVARIANT | per-replace: rec_count drops to 0 ⟹ `_MAKE_NOP` issued + ENABLED cleared. |
| `text_poke_under_stop_machine_or_arch` | INVARIANT | per-modify_all_code: invoked from stop_machine or arch-equivalent quiescent context. |
| `tramp_freed_after_rcu_tasks` | INVARIANT | per-DYNAMIC ops shutdown: `synchronize_rcu_tasks_rude()` + `synchronize_rcu_tasks()` precede `ftrace_trampoline_free`. |
| `ftrace_pid_func_short_circuit` | INVARIANT | per-call: when filter exists and current's pid is not allowed, `saved_func` not invoked. |
| `ftrace_disabled_propagates` | INVARIANT | post-`ftrace_kill`: every register attempt returns `-ENODEV`. |
| `update_func_atomic_via_list_func` | INVARIANT | per-update_ftrace_function: transition through `ftrace_ops_list_func` precedes direct-func change. |

### Layer 2: TLA+

`kernel/trace/ftrace_patch.tla`:
- N CPUs concurrently executing kernel text + one CPU patching via `text_poke` (or `stop_machine`).
- States: rec ∈ {nop, call, regs_call, tramp_call, direct_call, mid_patch}.
- Properties:
  - `safety_no_torn_call_observed` — per-CPU: any decoded instruction at rec.ip is one of the legal states; never a partial overwrite.
  - `safety_refcount_matches_enabled` — per-rec: `rec_count > 0 ⟺ FTRACE_FL_ENABLED`.
  - `safety_ipmodify_singleton` — per-rec: at most one ops with IPMODIFY references this ip.
  - `safety_direct_singleton_or_listed` — per-rec: DIRECT_EN set iff `rec_count == 1` and DIRECT requested; else multi-DIRECT routes via list_func.
  - `liveness_register_eventually_active` — per-register_ftrace_function: bounded delay before `ftrace_trace_function` reflects the new ops.
  - `liveness_unregister_eventually_freed` — per-unregister: bounded delay before trampoline freed (post 2× rcu_tasks).

`kernel/trace/fgraph_shadow.tla`:
- Per-task shadow stack; entry pushes onto `ret_stack`; exit pops.
- Properties:
  - `safety_stack_balanced` — per-task: every entry has matching exit during the task's lifetime (or on task_exit, stack is reset).
  - `safety_no_overflow` — per-task: `current->curr_ret_stack < FTRACE_RETFUNC_DEPTH * FGRAPH_FRAME_OFFSET`.
  - `safety_return_to_handler_finds_frame` — per-pop: top-of-stack matches the calling function's frame; original `ret` restored before jmp.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ftrace_init` post: every record is in NOP state OR DISABLED | `Ftrace::init` |
| `process_locs` post: records sorted; no duplicates; ranges respect kernel_text | `Ftrace::process_locs` |
| `register_ftrace_function` post: ops in list AND every matching rec has ref+1 OR rollback | `Ftrace::register_function` |
| `unregister_ftrace_function` post: ops absent AND every matching rec has ref-1 OR ENABLED cleared if hit 0 | `Ftrace::unregister_function` |
| `ftrace_check_record` post: returned UPDATE matches the diff between current and desired flags | `DynFtrace::check_record` |
| `ftrace_update_record` post: rec.flags updated atomically with returned UPDATE | `DynFtrace::update_record` |
| `addr_new` correctness: returns direct ⊕ tramp ⊕ regs ⊕ default per priority | `DynFtrace::addr_new` |
| `addr_curr` correctness: mirrors `addr_new` but using `_EN` bits | `DynFtrace::addr_curr` |
| `ftrace_make_call` post: instruction at rec.ip is now `call ftrace_addr` (arch-specific encoding) | `arch::Ftrace::make_call` |
| `ftrace_make_nop` post: instruction at rec.ip is now arch-NOP of correct length | `arch::Ftrace::make_nop` |
| `ftrace_kill` post: every future `register_ftrace_function` returns `-ENODEV` | `Ftrace::kill` |
| `pid_func` post: only invokes saved_func when current.pid passes filter | `Ftrace::pid_func` |
| `ftrace_hash_move` post: new hash linked; old hash freed after RCU; ops sees consistent state | `FtraceOps::hash_move` |

### Layer 4: Verus/Creusot functional

`Per-ops register-and-trace lifecycle` semantic equivalence (per `Documentation/trace/ftrace-uses.rst` and `Documentation/trace/ftrace-design.rst`):

```
ftrace_init()
  __mcount_loc → process_locs → ftrace_pages list
  every site nop-init via ftrace_make_nop
  ftrace_enabled = 1
register_ftrace_function(ops)
  prepare_direct_functions_for_ipmodify(ops)   [if IPMODIFY]
  ftrace_startup(ops, 0):
    __register_ftrace_function: add to ftrace_ops_list, update_ftrace_function
    ftrace_hash_ipmodify_enable
    ftrace_hash_rec_enable: set ENABLED|REGS|TRAMP|DIRECT|CALL_OPS per rec
    ftrace_run_update_code(command):
      arch_code_modify_prepare
      stop_machine(__ftrace_modify_code)
      arch_code_modify_post_process
  every traced site now: CALL ftrace_caller (or ftrace_regs_caller, tramp, direct)
function executes
  → ftrace_caller (arch asm) → ftrace_trace_function
    = single-ops direct OR ftrace_ops_list_func walk
    → ops->func(ip, parent_ip, ops, fregs)
  if FTRACE_START_FUNC_RET:
    fall into ftrace_graph_caller → prepare_ftrace_return → push ret_stack, return_to_handler installed
  function returns → return_to_handler → pop ret_stack → call gops->retfunc → jmp orig_ret
unregister_ftrace_function(ops)
  ftrace_shutdown(ops, 0):
    __unregister_ftrace_function: remove from list, update_ftrace_function
    ftrace_hash_ipmodify_disable
    ftrace_hash_rec_disable
    ftrace_run_update_code: re-patch every released rec
    if DYNAMIC: synchronize_rcu_tasks_rude + synchronize_rcu_tasks + free trampoline
ftrace_kill()  [on fault]
  ftrace_disabled = 1, ftrace_enabled = 0, ftrace_trace_function = stub
```

## Hardening

(Inherits row-1 features from `kernel/trace/00-overview.md` § Hardening — STRICT_KERNEL_RWX text_poke ROX→RW transition is mandatory.)

ftrace-specific reinforcement:

- **Per-`stop_machine` (or arch `text_poke_bp` BP-trap) for patch** — defense against per-concurrent-execution observing torn instruction.
- **Per-`text_poke` ROX→RW→ROX transition** — defense against per-permanent-RW-mapping privilege escalation primitive.
- **Per-`ftrace_lock` mutex for every flag transition** — defense against per-concurrent-register/unregister race corrupting refcount.
- **Per-IPMODIFY singleton enforced at register time** — defense against per-livepatch-conflict undefined behavior.
- **Per-DIRECT-vs-IPMODIFY mediated via `ops_func(CMD_ENABLE_SHARE_IPMODIFY_PEER)`** — defense against per-BPF-trampoline-vs-KLP collision.
- **Per-`FTRACE_FL_DISABLED` for weak-fn / KASLR-zeroed / init-freed** — defense against per-invalid-ip patch attempt.
- **Per-`ftrace_kill` permanent disable on unrecoverable patch fault** — defense against per-cascading-corruption.
- **Per-`synchronize_rcu_tasks_rude + _tasks` before trampoline free** — defense against per-preempted-on-trampoline UAF.
- **Per-`text_reserved` check against jump_label / static_call / kprobes** — defense against per-overlapping-patch corruption.
- **Per-`security_locked_down(LOCKDOWN_TRACEFS)` at filter writers** — defense against per-secureboot-bypass.
- **Per-module init: pages frozen post-init** — defense against per-module-init-text patch attempt after `module_enable_ro`.
- **Per-`PERMANENT` ops blocks `ftrace_enabled = 0`** — defense against per-disable-while-livepatched cascading kernel-bug.
- **Per-`ftrace_rec_count` capped at `FTRACE_REF_MAX`** — defense against per-refcount-overflow flag corruption.
- **Per-pid-list RCU update + per-cpu `ftrace_ignore_pid` cache** — defense against per-pid-filter race observing torn list.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Ring buffer internals (covered in `ring-buffer.md` Tier-3).
- Trace core / tracefs / per-instance machinery (covered in `trace.md` Tier-3).
- Event class registration + filter/trigger/hist (covered in `trace-events.md` Tier-3).
- Function-graph internals (shadow stack details, FGRAPH_INDEX/DATA/BITMAP encoding) covered in `kernel/trace/fgraph.md` future Tier-3 (this doc covers only the patching/registration glue).
- fprobe / rethook (separate Tier-3 if expanded).
- BPF trampoline JIT details (covered in `kernel/bpf/trampoline.md` Tier-3).
- Livepatch consistency model (covered in `kernel/livepatch.md` Tier-3).
- Architecture-specific instruction encoding (`arch/x86/kernel/ftrace.c`, `ftrace_64.S`) — covered in `arch/x86/ftrace.md` Tier-3.
- Implementation code.
