# Tier-3: kernel/kallsyms.c — Kernel symbol table (compressed name lookup, /proc/kallsyms, BPF iter)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/kallsyms.c (~937 lines)
  - kernel/kallsyms_internal.h
  - kernel/kallsyms_selftest.c
  - include/linux/kallsyms.h
  - scripts/kallsyms.c (build-time generator)
-->

## Summary

**kallsyms** is the in-kernel **symbol table** used for oops/backtrace symbolization, `/proc/kallsyms`, kprobe/BPF/ftrace symbol resolution, and live-debug `kallsyms_lookup_name()`. The vmlinux symbol set is encoded at link time by `scripts/kallsyms.c` into compressed tables: `kallsyms_names` (per-symbol stem-compressed bytes), `kallsyms_token_table` + `kallsyms_token_index` (per-token dictionary for the BPE-like compression), `kallsyms_offsets` (per-symbol PC offsets, encoded relative to kernel base via `offset_to_ptr` when CONFIG_RELOCATABLE / CONFIG_64BIT), `kallsyms_markers` (per-256-symbol jumpstart marker for `get_symbol_offset`), `kallsyms_seqs_of_names` (3-byte name-sorted index permutation for binary search by name), and `kallsyms_num_syms`. PC → symbol uses binary search on `kallsyms_offsets` (`get_symbol_pos`); name → PC uses binary search on `kallsyms_seqs_of_names` decompressed (`kallsyms_lookup_names`). Modules, ftrace-allocated trampolines, BPF-JITed images, and kprobes-instructions each have a parallel `*_kallsyms_lookup_name` / `*_get_kallsym` API merged at iteration time via `kallsym_iter` (with `pos_mod_end`, `pos_ftrace_mod_end`, `pos_bpf_end` watermarks). `/proc/kallsyms` is emitted via seq_file with kptr-restrict respected via `kallsyms_show_value(cred)`. `kallsyms_lookup_name` is **deprecated and not GPL-exported** for security (BPF helper `ksym()` uses a BPF iterator instead). Critical for: oops / backtrace, live-debug, BPF, ftrace.

This Tier-3 covers `kernel/kallsyms.c` (~937 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kallsyms_num_syms` | per-vmlinux symbol count | `KALLSYMS_NUM_SYMS` |
| `kallsyms_offsets` | per-symbol PC offsets | `KALLSYMS_OFFSETS` |
| `kallsyms_names` | per-compressed name stream | `KALLSYMS_NAMES` |
| `kallsyms_markers` | per-256-stride name-stream marker | `KALLSYMS_MARKERS` |
| `kallsyms_token_table` | per-BPE dictionary | `KALLSYMS_TOKEN_TABLE` |
| `kallsyms_token_index` | per-byte → token offset | `KALLSYMS_TOKEN_INDEX` |
| `kallsyms_seqs_of_names` | per-name-sort permutation (3 bytes/idx) | `KALLSYMS_SEQS_OF_NAMES` |
| `_einittext` / `_etext` / `_end` | per-section bounds | shared link symbols |
| `kallsyms_expand_symbol()` | per-name decompress | `Kallsyms::expand_symbol` |
| `kallsyms_get_symbol_type()` | per-name first-char (type) | `Kallsyms::get_symbol_type` |
| `get_symbol_offset()` | per-position name-stream offset | `Kallsyms::get_symbol_offset` |
| `kallsyms_sym_address()` | per-index PC computation | `Kallsyms::sym_address` |
| `get_symbol_seq()` | per-index sorted-permutation lookup | `Kallsyms::get_symbol_seq` |
| `kallsyms_lookup_names()` | per-name binary search → idx range | `Kallsyms::lookup_names` |
| `kallsyms_lookup_name()` | per-name → PC (DEPRECATED) | `Kallsyms::lookup_name` |
| `kallsyms_on_each_symbol()` | per-symbol iterator | `Kallsyms::on_each_symbol` |
| `kallsyms_on_each_match_symbol()` | per-name match iterator | `Kallsyms::on_each_match_symbol` |
| `get_symbol_pos()` | per-PC → idx binary search | `Kallsyms::get_symbol_pos` |
| `kallsyms_lookup_size_offset()` | per-PC → {size, offset} | `Kallsyms::lookup_size_offset` |
| `kallsyms_lookup_buildid()` | per-PC → {name, size, offset, modname, buildid} | `Kallsyms::lookup_buildid` |
| `kallsyms_lookup()` | per-PC → name (public) | `Kallsyms::lookup` |
| `lookup_symbol_name()` | per-PC → name (no offset/size) | `Kallsyms::lookup_symbol_name` |
| `is_ksym_addr()` | per-addr-in-kernel-text predicate | shared |
| `__sprint_symbol()` | per-buffer sprint helper | `Kallsyms::sprint_symbol_inner` |
| `sprint_symbol()` | per-public sprint w/ offset | `Kallsyms::sprint_symbol` |
| `sprint_symbol_build_id()` | per-sprint + buildid | `Kallsyms::sprint_symbol_build_id` |
| `sprint_symbol_no_offset()` | per-sprint, no offset | `Kallsyms::sprint_symbol_no_offset` |
| `sprint_backtrace()` | per-backtrace sprint (addr-1) | `Kallsyms::sprint_backtrace` |
| `sprint_backtrace_build_id()` | per-backtrace sprint + buildid | `Kallsyms::sprint_backtrace_build_id` |
| `append_buildid()` | per-buildid printf | `Kallsyms::append_buildid` |
| `struct kallsym_iter` | per-/proc/kallsyms seq_file iterator | `KallsymIter` |
| `get_ksymbol_core()` | per-vmlinux fetch | `Kallsyms::get_ksymbol_core` |
| `get_ksymbol_mod()` | per-module fetch | `Kallsyms::get_ksymbol_mod` |
| `get_ksymbol_ftrace_mod()` | per-ftrace-page fetch | `Kallsyms::get_ksymbol_ftrace_mod` |
| `get_ksymbol_bpf()` | per-BPF-JIT fetch | `Kallsyms::get_ksymbol_bpf` |
| `get_ksymbol_kprobe()` | per-kprobes-insn-page fetch | `Kallsyms::get_ksymbol_kprobe` |
| `update_iter()` / `update_iter_mod()` | per-pos advance | `Kallsyms::update_iter` / `_mod` |
| `reset_iter()` | per-iter init | `Kallsyms::reset_iter` |
| `s_start` / `s_next` / `s_stop` / `s_show` | per-seq_file ops | `KALLSYMS_SEQ_OPS` |
| `kallsyms_open()` | per-/proc/kallsyms open | `Kallsyms::open` |
| `kallsyms_show_value(cred)` | per-kptr-restrict gate | shared |
| `kallsyms_proc_ops` | per-/proc/kallsyms proc_ops | `KALLSYMS_PROC_OPS` |
| `kallsyms_init()` | per-device_initcall proc_create | `Kallsyms::init` |
| `kdb_walk_kallsyms()` | per-KGDB-KDB iterator | `Kallsyms::kdb_walk` |
| `bpf_iter_ksym_ops` + `bpf_ksym_iter_register()` | per-BPF-iterator target | `BPF_KSYM_ITER` |
| `ksym_iter_seq_info` | per-BPF iter seq_info | `KSYM_ITER_SEQ_INFO` |
| `ksym_iter_reg_info` | per-BPF iter reg | `KSYM_ITER_REG_INFO` |
| `btf_ksym_iter_id` | per-BTF id list singleton | `BTF_KSYM_ITER_ID` |
| `module_address_lookup` | per-module fallback | (kernel/module) |
| `bpf_address_lookup` | per-BPF-JIT fallback | (kernel/bpf) |
| `ftrace_mod_address_lookup` | per-ftrace-trampoline fallback | (kernel/trace/ftrace) |
| `module_kallsyms_lookup_name` | per-module name → PC fallback | (kernel/module) |
| `lookup_module_symbol_name` | per-module PC → name fallback | (kernel/module) |
| `kprobe_get_kallsym` | per-kprobes-insn iter | (kernel/kprobes) |
| `bpf_get_kallsym` | per-BPF-JIT iter | (kernel/bpf) |
| `module_get_kallsym` | per-module iter | (kernel/module) |
| `ftrace_mod_get_kallsym` | per-ftrace-page iter | (kernel/trace) |

## Compatibility contract

REQ-1: Compressed name stream (`kallsyms_names`):
- Per-symbol layout: `[len_lo (u8) [len_hi (u8) iff bit7 of len_lo]] [byte_0 ... byte_(len-1)]`.
- Each byte is a token index into `kallsyms_token_index[]` (lookup into `kallsyms_token_table[]` for the token's bytes).
- First byte of decompressed name is the **symbol type char** (e.g., 'T', 't', 'D', 'd', ...).

REQ-2: kallsyms_expand_symbol(off, result, maxlen):
- data = &kallsyms_names[off]; len = *data++; off++.
- if len & 0x80: len = (len & 0x7F) | (*data << 7); data++; off++. /* big-symbol form */
- off += len.
- For each compressed byte: tptr = &kallsyms_token_table[kallsyms_token_index[*data]]; iterate tptr (NUL-terminated token).
- First emitted char is the symbol-type and is consumed (skipped_first=1) — not written to result.
- Subsequent chars written into result, bounded by maxlen.
- result NUL-terminated; return next-symbol offset.

REQ-3: kallsyms_get_symbol_type(off):
- if kallsyms_names[off] & 0x80: off++. /* skip big-len high byte */
- return kallsyms_token_table[kallsyms_token_index[kallsyms_names[off + 1]]]. /* first char of first token */

REQ-4: get_symbol_offset(pos):
- name = &kallsyms_names[kallsyms_markers[pos >> 8]]. /* every 256-th symbol */
- for i in 0..(pos & 0xFF):
  - len = *name; if len & 0x80: len = ((len & 0x7F) | (name[1] << 7)) + 1.
  - name += len + 1.
- return name - kallsyms_names.

REQ-5: kallsyms_sym_address(idx):
- if !CONFIG_64BIT && !CONFIG_RELOCATABLE: return (u32)kallsyms_offsets[idx]. /* raw addr */
- else: return (unsigned long)offset_to_ptr(kallsyms_offsets + idx). /* PC-relative */

REQ-6: get_symbol_seq(index):
- seq = 0; for i in 0..3: seq = (seq << 8) | kallsyms_seqs_of_names[3*index + i].
- /* Decodes 3-byte big-endian permutation index */

REQ-7: kallsyms_lookup_names(name, start, end):
- low = 0; high = kallsyms_num_syms - 1.
- /* Binary search on sorted permutation */
- while low <= high:
  - mid = low + (high - low) / 2.
  - seq = get_symbol_seq(mid); off = get_symbol_offset(seq); kallsyms_expand_symbol(off, namebuf, ...).
  - cmp = strcmp(name, namebuf).
  - if cmp > 0: low = mid + 1; elif cmp < 0: high = mid - 1; else: break.
- if low > high: return -ESRCH.
- /* Expand to first match */
- low = mid; while low: probe low-1; if matches, low--; else break.
- *start = low.
- if end: high = mid; while high < num-1: probe high+1; if matches, high++; else break. *end = high.
- return 0.

REQ-8: kallsyms_lookup_name(name):
- /* DEPRECATED: NOT EXPORT_SYMBOL_GPL */
- if !*name: return 0.
- ret = kallsyms_lookup_names(name, &i, NULL).
- if !ret: return kallsyms_sym_address(get_symbol_seq(i)).
- return module_kallsyms_lookup_name(name). /* Fallback to modules */

REQ-9: kallsyms_on_each_symbol(fn, data):
- for i in 0..kallsyms_num_syms:
  - off = kallsyms_expand_symbol(off, namebuf, KSYM_NAME_LEN).
  - ret = fn(data, namebuf, kallsyms_sym_address(i)).
  - if ret != 0: return ret.
  - cond_resched().
- return 0.

REQ-10: kallsyms_on_each_match_symbol(fn, name, data):
- ret = kallsyms_lookup_names(name, &start, &end).
- if ret: return 0.
- for i in start..=end:
  - ret = fn(data, kallsyms_sym_address(get_symbol_seq(i))).
  - cond_resched().
- return ret.

REQ-11: get_symbol_pos(addr, symbolsize, offset) -> idx:
- /* Binary search on kallsyms_offsets array (sorted by PC) */
- low = 0; high = kallsyms_num_syms.
- while high - low > 1:
  - mid = low + (high - low) / 2.
  - if kallsyms_sym_address(mid) <= addr: low = mid; else: high = mid.
- /* Backtrack to first alias */
- while low > 0 && kallsyms_sym_address(low-1) == kallsyms_sym_address(low): low--.
- symbol_start = kallsyms_sym_address(low).
- /* Forward to next non-alias for symbol_end */
- for i in (low+1)..num: if kallsyms_sym_address(i) > symbol_start: symbol_end = ...; break.
- /* Section-end fallback */
- if !symbol_end:
  - if is_kernel_inittext(addr): symbol_end = _einittext.
  - elif CONFIG_KALLSYMS_ALL: symbol_end = _end.
  - else: symbol_end = _etext.
- if symbolsize: *symbolsize = symbol_end - symbol_start.
- if offset: *offset = addr - symbol_start.
- return low.

REQ-12: kallsyms_lookup_size_offset(addr, symbolsize, offset):
- if is_ksym_addr(addr): get_symbol_pos(addr, ...); return 1.
- return !!module_address_lookup(addr, ..., namebuf) || !!bpf_address_lookup(addr, ..., namebuf).

REQ-13: kallsyms_lookup_buildid(addr, symbolsize, offset, modname, modbuildid, namebuf) -> len:
- namebuf[0] = 0; if modname: *modname = NULL; if modbuildid: *modbuildid = NULL.
- if is_ksym_addr(addr):
  - pos = get_symbol_pos(addr, symbolsize, offset).
  - kallsyms_expand_symbol(get_symbol_offset(pos), namebuf, KSYM_NAME_LEN).
  - return strlen(namebuf).
- ret = module_address_lookup(addr, symbolsize, offset, modname, modbuildid, namebuf).
- if !ret: ret = bpf_address_lookup(addr, symbolsize, offset, namebuf).
- if !ret: ret = ftrace_mod_address_lookup(addr, symbolsize, offset, modname, modbuildid, namebuf).
- return ret.

REQ-14: kallsyms_lookup(addr, ..., modname, namebuf):
- ret = kallsyms_lookup_buildid(addr, symbolsize, offset, modname, NULL, namebuf).
- if !ret: return NULL.
- return namebuf.

REQ-15: __sprint_symbol(buffer, address, symbol_offset, add_offset, add_buildid):
- guard(rcu)() /* RAII: prevents module removal during modname access */.
- address += symbol_offset.
- len = kallsyms_lookup_buildid(address, &size, &offset, &modname, &buildid, buffer).
- if !len: return sprintf(buffer, "0x%lx", address - symbol_offset). /* unknown */
- offset -= symbol_offset.
- if add_offset: len += sprintf(buffer + len, "+%#lx/%#lx", offset, size).
- if modname:
  - len += sprintf(buffer + len, " [%s", modname).
  - if add_buildid: len += append_buildid(buffer + len, modname, buildid).
  - len += sprintf(buffer + len, "]").
- return len.

REQ-16: Per-public sprint variants:
- sprint_symbol(b, a) = __sprint_symbol(b, a, 0, 1, 0).
- sprint_symbol_build_id(b, a) = __sprint_symbol(b, a, 0, 1, 1).
- sprint_symbol_no_offset(b, a) = __sprint_symbol(b, a, 0, 0, 0).
- sprint_backtrace(b, a) = __sprint_symbol(b, a, -1, 1, 0). /* addr-1 for noreturn callee */
- sprint_backtrace_build_id(b, a) = __sprint_symbol(b, a, -1, 1, 1).
- Per-EXPORT_SYMBOL_GPL: sprint_symbol, sprint_symbol_build_id, sprint_symbol_no_offset.

REQ-17: append_buildid(buffer, modname, buildid):
- if !modname: return 0.
- if !buildid: pr_warn_once; return 0.
- static_assert sizeof(typeof_member(struct module, build_id)) == 20.
- return sprintf(buffer, " %20phN", buildid). /* 20-byte SHA-1 build_id */
- /* CONFIG_STACKTRACE_BUILD_ID gated; else stub returning 0 */

REQ-18: struct kallsym_iter (per-/proc/kallsyms seq_file state):
- pos: loff_t — current position (0..num_total_symbols).
- pos_mod_end, pos_ftrace_mod_end, pos_bpf_end: per-section end watermarks.
- value: PC of current symbol.
- nameoff: per-vmlinux offset into kallsyms_names (when iterating core).
- type, name[KSYM_NAME_LEN], module_name[MODULE_NAME_LEN], exported, show_value.

REQ-19: reset_iter(iter, new_pos):
- iter.name[0] = 0; iter.nameoff = get_symbol_offset(new_pos); iter.pos = new_pos.
- if new_pos == 0: iter.pos_mod_end = iter.pos_ftrace_mod_end = iter.pos_bpf_end = 0.

REQ-20: update_iter(iter, pos):
- if pos >= kallsyms_num_syms: return update_iter_mod(iter, pos).
- if pos != iter.pos: reset_iter(iter, pos).
- iter.nameoff += get_ksymbol_core(iter); iter.pos++; return 1.

REQ-21: update_iter_mod(iter, pos):
- iter.pos = pos.
- if (!iter.pos_mod_end || iter.pos_mod_end > pos) && get_ksymbol_mod(iter): return 1.
- if (!iter.pos_ftrace_mod_end || iter.pos_ftrace_mod_end > pos) && get_ksymbol_ftrace_mod(iter): return 1.
- if (!iter.pos_bpf_end || iter.pos_bpf_end > pos) && get_ksymbol_bpf(iter): return 1.
- return get_ksymbol_kprobe(iter).

REQ-22: get_ksymbol_core(iter):
- off = iter.nameoff.
- iter.module_name[0] = 0.
- iter.value = kallsyms_sym_address(iter.pos).
- iter.type = kallsyms_get_symbol_type(off).
- off = kallsyms_expand_symbol(off, iter.name, ARRAY_SIZE(iter.name)).
- return off - iter.nameoff.

REQ-23: get_ksymbol_{mod,ftrace_mod,bpf,kprobe}:
- module: module_get_kallsym(pos - kallsyms_num_syms, ...).
- ftrace_mod: ftrace_mod_get_kallsym(pos - pos_mod_end, ...) /* "__builtin__ftrace" pseudo-modname */.
- bpf: module_name = "bpf"; bpf_get_kallsym(pos - pos_ftrace_mod_end, ...).
- kprobe: module_name = "__builtin__kprobes"; kprobe_get_kallsym(pos - pos_bpf_end, ...).
- Each: on ret < 0, set respective pos_*_end watermark and return 0; else return 1.

REQ-24: s_show(m, p):
- if !iter.name[0]: return 0. /* Skip nameless debugging symbols */
- value = iter.show_value ? (void*)iter.value : NULL. /* kptr-restrict gate */
- if iter.module_name[0]:
  - type = iter.exported ? toupper(iter.type) : tolower(iter.type).
  - seq_printf(m, "%px %c %s\t[%s]\n", value, type, iter.name, iter.module_name).
- else:
  - seq_printf(m, "%px %c %s\n", value, iter.type, iter.name).

REQ-25: kallsyms_open(inode, file):
- iter = __seq_open_private(file, &kallsyms_op, sizeof(*iter)) || -ENOMEM.
- reset_iter(iter, 0).
- iter.show_value = kallsyms_show_value(file.f_cred). /* Cache at open-time */
- return 0.

REQ-26: kallsyms_proc_ops:
- proc_open = kallsyms_open; proc_read = seq_read; proc_lseek = seq_lseek; proc_release = seq_release_private.

REQ-27: kallsyms_init (device_initcall):
- proc_create("kallsyms", 0444, NULL, &kallsyms_proc_ops). /* 0444: world-readable but values masked unless privileged */

REQ-28: Per-CONFIG_KGDB_KDB kdb_walk_kallsyms(pos):
- static kdb_walk_kallsyms_iter.
- if *pos == 0: memset + reset_iter.
- while (1): update_iter; ++*pos; if iter.name[0]: return iter.name.

REQ-29: Per-CONFIG_BPF_SYSCALL BPF iter "ksym":
- struct bpf_iter__ksym { meta, ksym }.
- ksym_prog_seq_show: get bpf_prog from meta; bpf_iter_run_prog(prog, &ctx).
- bpf_iter_ksym_init: reset_iter(iter, 0); iter.show_value = kallsyms_show_value(current_cred()).
- ksym_iter_seq_info = { seq_ops = bpf_iter_ksym_ops, init_seq_private = bpf_iter_ksym_init, seq_priv_size = sizeof(kallsym_iter) }.
- ksym_iter_reg_info = { target = "ksym", feature = BPF_ITER_RESCHED, ctx_arg_info = [{ offset of ksym, PTR_TO_BTF_ID_OR_NULL }], seq_info }.
- BTF_ID_LIST_SINGLE(btf_ksym_iter_id, struct, kallsym_iter).
- bpf_ksym_iter_register (late_initcall): ksym_iter_reg_info.ctx_arg_info[0].btf_id = *btf_ksym_iter_id; bpf_iter_reg_target.

## Acceptance Criteria

- [ ] AC-1: kallsyms_lookup_name("kfree") returns address of `kfree` if statically linked.
- [ ] AC-2: kallsyms_lookup(addr) for addr inside `printk`: returns "printk" with size & offset.
- [ ] AC-3: kallsyms_lookup_size_offset for unknown addr: returns 0 (and falls through to module / BPF).
- [ ] AC-4: Aliased symbols (same PC): get_symbol_pos returns first-alias idx.
- [ ] AC-5: kallsyms_on_each_symbol visits exactly kallsyms_num_syms entries.
- [ ] AC-6: kallsyms_on_each_match_symbol("schedule") visits all sym entries named "schedule" (alias group).
- [ ] AC-7: /proc/kallsyms read by unprivileged: %px renders as 0000000000000000 (kallsyms_show_value(cred) == false).
- [ ] AC-8: /proc/kallsyms read by CAP_SYSLOG-capable: real addresses emitted.
- [ ] AC-9: /proc/kallsyms includes module symbols with "[module_name]" suffix.
- [ ] AC-10: /proc/kallsyms includes BPF-JITed images with "[bpf]" suffix.
- [ ] AC-11: /proc/kallsyms includes ftrace-mod entries with "[__builtin__ftrace]" suffix.
- [ ] AC-12: /proc/kallsyms includes kprobe insn slots with "[__builtin__kprobes]" suffix.
- [ ] AC-13: sprint_symbol(buf, &printk) renders "printk+0x0/0x..." (no module suffix).
- [ ] AC-14: sprint_backtrace(buf, ret_to_noreturn): renders caller name (not next function past noreturn).
- [ ] AC-15: kallsyms_expand_symbol handles big-symbol form (MSB=1 in len byte) correctly.
- [ ] AC-16: BPF iter "ksym": iterates all symbols and passes valid kallsym_iter pointer (BTF id matches).
- [ ] AC-17: kallsyms_lookup_name is **not** EXPORT_SYMBOL_GPL (unexported; modules cannot link to it).

## Architecture

```
struct KallsymIter {
  pos: loff_t,
  pos_mod_end: loff_t,
  pos_ftrace_mod_end: loff_t,
  pos_bpf_end: loff_t,
  value: u64,                       // PC of current sym
  nameoff: u32,                     // offset into KALLSYMS_NAMES (core only)
  type: u8,                         // 'T', 't', 'D', 'd', 'B', 'b', ...
  name: [u8; KSYM_NAME_LEN],
  module_name: [u8; MODULE_NAME_LEN],
  exported: i32,
  show_value: i32,                  // cached kptr-restrict gate
}

// Build-time tables (from scripts/kallsyms.c output)
extern const KALLSYMS_NUM_SYMS: u32;
extern const KALLSYMS_OFFSETS: [PrelOff32; _];  // PC offsets or raw u32 addrs
extern const KALLSYMS_NAMES:   [u8; _];          // compressed stream
extern const KALLSYMS_MARKERS: [u32; _];         // every-256 marker
extern const KALLSYMS_TOKEN_TABLE: [u8; _];      // BPE dictionary
extern const KALLSYMS_TOKEN_INDEX: [u16; 256];   // byte → table-offset
extern const KALLSYMS_SEQS_OF_NAMES: [u8; 3 * _]; // 3-byte name-sorted permutation
```

`Kallsyms::lookup(addr, size_out, off_out, modname_out, namebuf) -> Option<&str>`:
1. ret = Kallsyms::lookup_buildid(addr, size_out, off_out, modname_out, None, namebuf).
2. if ret == 0: return None.
3. return Some(namebuf).

`Kallsyms::lookup_buildid(addr, size, off, modname, buildid, namebuf) -> usize`:
1. namebuf[0] = 0.
2. if let Some(m) = modname { *m = None; }
3. if let Some(b) = buildid { *b = None; }
4. if is_ksym_addr(addr):
   - pos = Kallsyms::get_symbol_pos(addr, size, off).
   - Kallsyms::expand_symbol(Kallsyms::get_symbol_offset(pos), namebuf, KSYM_NAME_LEN).
   - return strlen(namebuf).
5. ret = Module::address_lookup(addr, size, off, modname, buildid, namebuf).
6. if ret == 0: ret = Bpf::address_lookup(addr, size, off, namebuf).
7. if ret == 0: ret = FtraceMod::address_lookup(addr, size, off, modname, buildid, namebuf).
8. return ret.

`Kallsyms::get_symbol_pos(addr, size, off) -> usize`:
1. low = 0; high = KALLSYMS_NUM_SYMS.
2. while high - low > 1:
   - mid = low + (high - low) / 2.
   - if Kallsyms::sym_address(mid) <= addr: low = mid; else: high = mid.
3. while low > 0 && Kallsyms::sym_address(low - 1) == Kallsyms::sym_address(low): low -= 1.
4. symbol_start = Kallsyms::sym_address(low).
5. /* Forward scan */
6. for i in (low+1)..KALLSYMS_NUM_SYMS:
   - if Kallsyms::sym_address(i) > symbol_start: symbol_end = Kallsyms::sym_address(i); break.
7. if symbol_end == 0:
   - if is_kernel_inittext(addr): symbol_end = _einittext.
   - else if CONFIG_KALLSYMS_ALL: symbol_end = _end.
   - else: symbol_end = _etext.
8. if let Some(s) = size: *s = symbol_end - symbol_start.
9. if let Some(o) = off: *o = addr - symbol_start.
10. return low.

`Kallsyms::lookup_names(name, start, end) -> Result<(), Errno>`:
1. low = 0; high = KALLSYMS_NUM_SYMS - 1.
2. loop:
   - if low > high: return Err(ESRCH).
   - mid = low + (high - low) / 2.
   - seq = Kallsyms::get_symbol_seq(mid).
   - off = Kallsyms::get_symbol_offset(seq).
   - Kallsyms::expand_symbol(off, namebuf, KSYM_NAME_LEN).
   - cmp = strcmp(name, namebuf).
   - if cmp > 0: low = mid + 1; elif cmp < 0: high = mid - 1; else: break.
3. /* Backtrack to first match */
4. low = mid; while low > 0:
   - probe = Kallsyms::probe_name(low - 1); if probe != name { break; } low -= 1.
5. *start = low.
6. if let Some(e) = end:
   - high = mid; while high < KALLSYMS_NUM_SYMS - 1:
     - probe = Kallsyms::probe_name(high + 1); if probe != name { break; } high += 1.
   - *e = high.
7. return Ok(()).

`Kallsyms::lookup_name(name) -> u64`:    /* DEPRECATED — NOT EXPORTED */
1. if name.is_empty(): return 0.
2. match Kallsyms::lookup_names(name, &mut i, None):
   - Ok(()): return Kallsyms::sym_address(Kallsyms::get_symbol_seq(i)).
   - Err(_): return Module::kallsyms_lookup_name(name).

`Kallsyms::expand_symbol(off, result, maxlen) -> u32`:
1. data = &KALLSYMS_NAMES[off]; len = *data; data++; off++.
2. if len & 0x80 != 0: len = (len & 0x7F) | (*data << 7); data++; off++.
3. off += len.
4. skipped_first = false.
5. while len > 0:
   - tptr = &KALLSYMS_TOKEN_TABLE[KALLSYMS_TOKEN_INDEX[*data]].
   - data++; len--.
   - while *tptr != 0:
     - if !skipped_first { skipped_first = true; } /* drop type-char */
     - else { if maxlen <= 1 { goto tail; } *result++ = *tptr; maxlen--; }
     - tptr++.
6. tail: if maxlen > 0 { *result = 0; }
7. return off.

`Kallsyms::sprint_symbol_inner(buffer, address, symbol_offset, add_offset, add_buildid) -> usize`:
1. guard(rcu)(); /* RAII */
2. address += symbol_offset.
3. len = Kallsyms::lookup_buildid(address, &size, &offset, &modname, &buildid, buffer).
4. if len == 0: return sprintf(buffer, "0x%lx", address - symbol_offset).
5. offset -= symbol_offset.
6. if add_offset: len += sprintf(buffer + len, "+%#lx/%#lx", offset, size).
7. if let Some(m) = modname:
   - len += sprintf(buffer + len, " [{}", m).
   - if add_buildid: len += Kallsyms::append_buildid(buffer + len, m, buildid).
   - len += sprintf(buffer + len, "]").
8. return len.

`Kallsyms::s_show(seq, p)`:
1. iter = seq.private as &KallsymIter.
2. if iter.name[0] == 0: return 0.
3. value = if iter.show_value { iter.value as *const () } else { ptr::null() }.
4. if iter.module_name[0] != 0:
   - type = if iter.exported { toupper(iter.type) } else { tolower(iter.type) }.
   - seq_printf(seq, "%px %c %s\t[%s]\n", value, type, iter.name, iter.module_name).
5. else: seq_printf(seq, "%px %c %s\n", value, iter.type, iter.name).
6. return 0.

`Kallsyms::open(inode, file) -> Result<(), Errno>`:
1. iter = __seq_open_private(file, KALLSYMS_OP, size_of::<KallsymIter>()).ok_or(ENOMEM)?.
2. Kallsyms::reset_iter(iter, 0).
3. iter.show_value = kallsyms_show_value(file.f_cred).
4. return Ok(()).

`Kallsyms::init()`:
1. proc_create("kallsyms", 0o444, None, KALLSYMS_PROC_OPS).
2. return 0.

`BPF_KSYM_ITER_REGISTER (late_initcall)`:
1. KSYM_ITER_REG_INFO.ctx_arg_info[0].btf_id = *BTF_KSYM_ITER_ID.
2. bpf_iter_reg_target(&KSYM_ITER_REG_INFO).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `expand_symbol_bounds` | INVARIANT | per-expand_symbol: writes at most maxlen-1 chars + NUL. |
| `expand_symbol_off_increment` | INVARIANT | per-expand_symbol: returned off > input off. |
| `get_symbol_offset_in_names` | INVARIANT | per-get_symbol_offset: result < sizeof(KALLSYMS_NAMES). |
| `sym_address_in_text` | INVARIANT | per-sym_address(idx < num): result ∈ [_stext, _end]. |
| `lookup_names_binsearch_monotone` | INVARIANT | per-lookup_names: low ≤ mid ≤ high invariant maintained. |
| `lookup_names_returns_first_match` | INVARIANT | per-lookup_names: *start refers to smallest idx where strcmp == 0. |
| `get_symbol_pos_first_alias` | INVARIANT | per-get_symbol_pos: returned idx is smallest where sym_address == symbol_start. |
| `get_symbol_pos_symbol_end_section_fallback` | INVARIANT | per-get_symbol_pos: no next-non-alias ⟹ symbol_end ∈ {_einittext, _end, _etext}. |
| `sprint_symbol_unknown_addr_format` | INVARIANT | per-sprint_symbol: no symbol ⟹ "0x%lx" form. |
| `sprint_symbol_modname_format` | INVARIANT | per-sprint_symbol w/ modname: " [modname]" suffix. |
| `sprint_backtrace_addr_minus_1` | INVARIANT | per-sprint_backtrace: address fed as addr-1 internally. |
| `show_value_kptr_restrict` | INVARIANT | per-s_show: !show_value ⟹ %px renders NULL. |
| `kallsyms_open_caches_show_value` | INVARIANT | per-open: iter.show_value cached from f_cred (not re-evaluated on each line). |
| `iter_section_watermarks_monotone` | INVARIANT | per-update_iter_mod: pos_mod_end ≤ pos_ftrace_mod_end ≤ pos_bpf_end. |
| `lookup_name_not_exported_gpl` | INVARIANT | per-build-time: kallsyms_lookup_name lacks EXPORT_SYMBOL_GPL declaration. |

### Layer 2: TLA+

`kernel/kallsyms.tla`:
- Per-build-time symbol stream (KALLSYMS_NAMES with token compression) and lookup operations (lookup_name, lookup, on_each_symbol, sprint_symbol).
- Properties:
  - `safety_lookup_round_trip` — per-symbol s in {0..N-1}: lookup(sym_address(s)) returns s.
  - `safety_lookup_name_round_trip` — per-name n in symbols: lookup_name(name(s)) returns sym_address(s).
  - `safety_expand_symbol_total` — per-valid off: expand_symbol terminates with NUL-terminated string ≤ maxlen.
  - `safety_first_alias_returned` — per-aliased addr a: get_symbol_pos returns smallest idx i with sym_address(i) == a.
  - `safety_section_end_fallback` — per-last-symbol PC ≥ _stext: symbol_end falls through to section end.
  - `safety_kptr_restrict_obeyed` — per-/proc/kallsyms read: !kallsyms_show_value(cred) ⟹ all addresses printed as NULL.
  - `safety_module_iter_disjoint` — per-iter watermarks pos_mod_end / pos_ftrace_mod_end / pos_bpf_end never overlap their predecessors.
  - `liveness_on_each_symbol_terminates` — per-on_each_symbol: visits all syms or early-exits on fn != 0.
  - `liveness_seq_file_emits_all` — per-/proc/kallsyms read: emits exactly kallsyms_num_syms + module-syms + ftrace-mod + bpf + kprobe entries.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Kallsyms::lookup` post: ret.is_some() ⟹ namebuf is NUL-terminated valid UTF-8 (or ASCII) | `Kallsyms::lookup` |
| `Kallsyms::lookup` post: ret.is_some() ⟹ size_out and off_out within sym bounds | `Kallsyms::lookup` |
| `Kallsyms::lookup_name` post: ret != 0 ⟹ sym_address(idx) found such that name(idx) == query | `Kallsyms::lookup_name` |
| `Kallsyms::lookup_names` post: start ≤ end ∧ ∀ i ∈ [start..end]: name(i) == query | `Kallsyms::lookup_names` |
| `Kallsyms::get_symbol_pos` post: sym_address(low) ≤ addr < sym_address(low+1) (or section end) | `Kallsyms::get_symbol_pos` |
| `Kallsyms::expand_symbol` post: result NUL-terminated; strlen(result) ≤ maxlen-1 | `Kallsyms::expand_symbol` |
| `Kallsyms::sprint_symbol_inner` post: len ≤ KSYM_SYMBOL_LEN | `Kallsyms::sprint_symbol_inner` |
| `Kallsyms::on_each_symbol` post: ret == 0 ⟹ visited all syms; ret != 0 ⟹ early-exit | `Kallsyms::on_each_symbol` |
| `Kallsyms::open` post: iter initialized via reset_iter(0); show_value cached | `Kallsyms::open` |
| `BPF_KSYM_ITER` post: ctx_arg_info[0].btf_id == BTF_KSYM_ITER_ID | `BPF_KSYM_ITER` |

### Layer 4: Verus/Creusot functional

`Per-PC → get_symbol_pos (binary search + alias walk + section-end fallback) → expand_symbol (token-table decompress) → sprint format` ↔ `scripts/kallsyms.c` output encoding semantic equivalence: per-Documentation/dev-tools/kallsyms.rst (if extant) + scripts/kallsyms.c compression spec (stem → token-table by Paulo Marques 2004).

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

kallsyms reinforcement:

- **Per-kallsyms_show_value(cred) gate** — defense against per-unprivileged-leak of kernel addresses (kptr_restrict policy).
- **Per-kallsyms_lookup_name NOT EXPORTED GPL** — defense against per-out-of-tree-module abusing symbol lookup for arbitrary function override.
- **Per-rcu-guard in __sprint_symbol** — defense against per-module-unload race during modname read (RAII guard).
- **Per-namebuf NUL-termination in expand_symbol** — defense against per-buffer-overrun on truncation.
- **Per-maxlen-bounded write in expand_symbol** — defense against per-write-past-namebuf.
- **Per-binary-search bounded loop in lookup_names / get_symbol_pos** — defense against per-runaway-iteration.
- **Per-section-end fallback in get_symbol_pos** — defense against per-out-of-range size computation for last symbol.
- **Per-pos_*_end watermark caches** — defense against per-redundant-call to module/ftrace/bpf lookup when those sections exhausted.
- **Per-/proc/kallsyms 0444 perm + kptr-restrict** — defense against per-easy-read by attackers.
- **Per-show_value cached at open** — defense against per-cred-change mid-read (file descriptor capabilities don't escalate post-open).
- **Per-cond_resched in iterators** — defense against per-large-symbol-table soft-lockup.
- **Per-static_assert sizeof(build_id) == 20** — defense against per-buildid format drift between scripts and runtime.
- **Per-BPF iter "ksym" through BTF_ID_LIST_SINGLE** — defense against per-BPF-program reading mis-typed iter context.
- **Per-bad-taint module skip in /proc/kallsyms backing** — handled in module subsystem (defense against per-fake-symbol-injection).
- **Per-CAP_SYSLOG check (via kallsyms_show_value)** — defense against per-policy-bypass for address disclosure.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `/proc/kallsyms` seq_file emission is bounds-checked per line; lookup name buffers are stack-allocated and length-validated.
- **PAX_KERNEXEC** — `kallsyms_names`, `kallsyms_addresses`, `kallsyms_token_table`, and `kallsyms_token_index` live in `__ro_after_init`/const sections; the symbol table cannot be patched at runtime.
- **PAX_RANDKSTACK** — randomize kernel stack on every `kallsyms_lookup`/`kallsyms_lookup_size_offset` entry.
- **PAX_REFCOUNT** — saturating atomics on iterator state and BPF "ksym" iter refcounts.
- **PAX_MEMORY_SANITIZE** — scrub temporary name buffers used during expand/compress (`expand_symbol`).
- **PAX_UDEREF** — strict user/kernel split on /proc/kallsyms read path; no user pointer ever enters the lookup core.
- **PAX_RAP / kCFI** — type-signed indirect calls for the symbol-iter `show`/`next`/`stop` ops and for the BPF iter context.
- **GRKERNSEC_HIDESYM** — primary control surface: when enabled, all addresses in `/proc/kallsyms` and `/proc/modules` show as 0 for non-CAP_SYSLOG readers, and `kptr_restrict=2` is the hard default (no override via sysctl from non-init userns).
- **GRKERNSEC_DMESG** — restrict kallsyms-derived stack traces and "?? at <addr>" backtraces to CAP_SYSLOG.
- **kallsyms_lookup_name removed for modules** — the function is no longer exported (matching upstream removal); attempts to relink it via livepatch or insmod are rejected at module load (modpost denies the symbol reference).
- **/proc/kallsyms 0400 root-only at boot** — perms tightened beyond the upstream 0444 default; reading requires CAP_SYSLOG even for root in a non-init userns.
- **show_value cached at file open** — capability snapshot is taken once at `open(2)`; later setresuid/seccomp on the fd cannot escalate disclosure.
- **No address disclosure via dmesg fallback** — kallsyms_show_value(NULL) returns false; printk's `%pK`/`%pS` honor the snapshot and never leak through dmesg under GRKERNSEC_DMESG.
- **BPF iter "ksym" gated by CAP_SYS_ADMIN + CAP_PERFMON** — even with BPF available, the symbol iter requires the address-disclosure capabilities; no BPF-side `bpf_kallsyms_lookup_name` helper is exposed.
- **modpost check** — build-time check refuses any kernel/module reference to `kallsyms_lookup_name` or `module_kallsyms_lookup_name` outside the symbol core itself.
- **Rationale**: kallsyms is the canonical KASLR-defeat oracle and the standard pre-step for ROP/data-only attacks. GRKERNSEC_HIDESYM with kptr_restrict=2 as the hard default plus removal of `kallsyms_lookup_name` from the module symbol space closes both the disclosure and the runtime-symbol-resolution paths.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- scripts/kallsyms.c build-time generator (covered separately in build infrastructure Tier-3)
- kallsyms_selftest.c (kselftest harness; out of scope)
- Module symbol-table backing (`kernel/module/kallsyms.c` covered in `module/` Tier-3)
- BPF JIT image kallsyms backing (`kernel/bpf/core.c` covered in `bpf/` Tier-3)
- ftrace trampoline kallsyms backing (`kernel/trace/ftrace.c` covered in `trace/` Tier-3)
- kprobe insn-slot kallsyms backing (`kernel/kprobes.c` covered in `kprobes.md` Tier-3)
- kptr-restrict sysctl + cred policy (`kernel/sysctl.c` / `lib/vsprintf.c` covered in their Tier-3)
- KGDB KDB consumer (out of scope unless KGDB feature enabled)
- Implementation code
