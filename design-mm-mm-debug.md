---
title: "Tier-3: mm/debug.c + mm/debug_*.c — Memory subsystem debug + introspection helpers"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

mm/debug.c and siblings provide **debug + introspection** helpers for the mm subsystem: per-folio/page debug-dump (`dump_page`), per-VMA dump (`dump_vma`), per-mm_struct dump (`dump_mm`), per-page-table dump (`dump_pte`), page-flag debug-decode helpers, per-page-ref trace (debug_page_ref), per-pgtable-test (boot-time PMD/PUD/PTE-walker correctness), rodata-test (verify `.rodata` is RO). Per-CONFIG_DEBUG_VM enables panic-on-invariant-fail (VM_BUG_ON_*) for catching mm-state corruption. Critical for: developer debug + production triage (crash-dump analysis).

This Tier-3 covers `debug.c` (~364 lines) + sibling debug files.

### Acceptance Criteria

- [ ] AC-1: dump_page on valid folio: per-fields printed; PG_* flags decoded.
- [ ] AC-2: dump_vma: per-field printed.
- [ ] AC-3: dump_mm: per-counter printed.
- [ ] AC-4: VM_BUG_ON_FOLIO(folio.PG_locked != 1, folio): triggers BUG when condition false.
- [ ] AC-5: CONFIG_DEBUG_VM off: VM_BUG_ON compiled-out.
- [ ] AC-6: debug_vm_pgtable_init boot: per-pgtable walker validated.
- [ ] AC-7: rodata_test: per-write to .rodata faults.
- [ ] AC-8: debug_page_ref trace enabled: per-ref-count change traced.
- [ ] AC-9: page_init_poison: pre-real-use page has 0xAA pattern.
- [ ] AC-10: oops dump on UAF / NPE: dump_page invoked from oops-printer.

### Architecture

Per-flag name table:

```
static PAGEFLAG_NAMES: &[&'static str] = &[
  "locked", "writeback", "referenced", "uptodate", "dirty",
  "lru", "active", "workingset", "waiters", "error",
  "slab", "owner_priv_1", "arch_1", "reserved", "private",
  "private_2", "writeback", "head", "swapcache", "swapbacked",
  "unevictable", "mlocked", "uncached", "hwpoison", ...
];
```

`MmDebug::dump_page(page, reason)`:
1. folio = page_folio(page).
2. pr_warn("page:%p refcount:%d mapcount:%d mapping:%p index:%lx pfn:%lx\n",
   page, page_ref_count(page), page_mapcount(page), page.mapping, page.index, page_to_pfn(page)).
3. dump_flags(page.flags).
4. if reason: pr_warn("page dumped because: %s\n", reason).

`MmDebug::dump_vma(vma)`:
1. pr_warn("vma %p start %lx end %lx mm %p prot %lx anon_vma %p vm_ops %p file %p\n",
   vma, vma.vm_start, vma.vm_end, vma.vm_mm, pgprot_val(vma.vm_page_prot), vma.anon_vma, vma.vm_ops, vma.vm_file).

`MmDebug::dump_mm(mm)`:
1. pr_warn("mm %p mmap %p pgd %p mm_users %d mm_count %d ...\n", mm, mm.mmap, mm.pgd, ...).

`MmDebug::dump_pte(pte, addr)`:
1. pr_warn("pte @%lx: %llx [%s]\n", addr, pte_val(pte), pte_present(pte) ? "present" : "not_present").
2. if pte_present(pte): decode bits.

`MmDebug::pgtable_init()` (CONFIG_DEBUG_VM_PGTABLE):
1. Allocate synthetic mm + vma.
2. For each level (PGD/PUD/PMD/PTE): test walker.
3. Verify pte_clear/set semantics.
4. Free synthetic.

`MmDebug::rodata_test()` (CONFIG_DEBUG_RODATA_TEST):
1. attempted_write = &rodata_marker.
2. *attempted_write = 0xDEAD.  // should fault
3. if no-fault: panic("rodata not protected!").

`MmDebug::page_init_poison(page, size)`:
1. memset(page, PAGE_POISON_VALUE, size).  // 0xAA

### Out of Scope

- mm/00-overview (Tier-2)
- mm/debug_page_alloc.c (per-allocator debug; covered separately if expanded)
- mm/debug_page_ref.c (per-tracepoint; covered separately)
- mm/rodata_test.c (covered separately)
- /proc/<pid>/{maps, smaps, status} (covered in fs/proc/ separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `dump_page()` | per-folio/page debug-dump | `MmDebug::dump_page` |
| `__dump_page()` | per-page internal | `MmDebug::__dump_page` |
| `dump_vma()` | per-VMA dump | `MmDebug::dump_vma` |
| `dump_mm()` | per-mm_struct dump | `MmDebug::dump_mm` |
| `dump_pte()` / `dump_pmd()` / `dump_pud()` / `dump_pgd()` | per-page-table-entry dump | `MmDebug::dump_pte` etc. |
| `__page_flag_name` | per-flag name lookup | `MmDebug::page_flag_name` |
| `page_init_poison()` | per-struct page poison-init | `MmDebug::page_init_poison` |
| `pageflag_names[]` | per-PG_* name table | shared |
| `debug_page_ref` | per-page-refcount trace (TRACE_EVENT) | shared |
| `debug_vm_pgtable_init()` | per-boot pgtable test | `MmDebug::pgtable_init` |
| `rodata_test_*` | per-boot rodata-protection test | `MmDebug::rodata_test` |
| `mark_rodata_ro()` | per-arch hook | `MmDebug::mark_rodata_ro` |
| `VM_BUG_ON_*` macro family | per-invariant-failure | macros |

### compatibility contract

REQ-1: Per-CONFIG_DEBUG_VM:
- Enables VM_BUG_ON, VM_WARN_ON.
- Per-page-flag-mask invariants checked.
- Per-VMA invariants checked.
- Performance overhead ~5-10% on heavy mm code-paths.

REQ-2: dump_page(page, reason):
- "page:<vaddr> refcount:N mapcount:M mapping:<ptr> index:N pfn:N".
- per-PG_* flag-names decoded.
- per-folio: dump_folio.

REQ-3: dump_vma(vma):
- vm_start..vm_end vm_prot vm_flags + vm_file + vm_ops.
- Per-vm_pgoff per-mapping-offset.

REQ-4: dump_mm(mm):
- mmap, total_vm, locked_vm, exec_vm, stack_vm, data_vm.
- mm_count + mm_users refcounts.
- pgd phys-addr.

REQ-5: dump_pte / dump_pmd / dump_pud / dump_pgd:
- Per-entry decode: present/write/user/access/dirty/global/no-exec.
- pte_pfn extraction.

REQ-6: debug_vm_pgtable_init:
- Per-boot walk synthetic VMA; verify page-table-walker.
- Checks per-arch pgd/pud/pmd/pte invariants.

REQ-7: rodata_test:
- Per-boot: attempt write to .rodata section.
- Per-arch: should fault.

REQ-8: page_init_poison:
- Per-struct page early-init poison-pattern (0xAA).
- Subsequent first-real-use must zero-or-init.
- Catches per-uninit-page-access bugs.

REQ-9: VM_BUG_ON_FOLIO / VM_BUG_ON_PAGE / VM_BUG_ON_VMA / VM_BUG_ON_MM:
- Per-CONFIG_DEBUG_VM: panic on assertion.
- Per-non-DEBUG_VM: compiled-out.

REQ-10: Per-debug_page_ref tracepoint:
- Per-get_page / put_page / atomic-inc / dec → trace.
- Enabled via /sys/kernel/debug/tracing/events/page_ref/*.

REQ-11: Per-userspace ABI:
- /proc/<pid>/{maps, smaps, status, oom_adj}: per-mm introspection.
- /sys/kernel/debug/page_alloc_debug.
- /proc/buddyinfo, /proc/zoneinfo, /proc/pagetypeinfo.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pageflag_names_align` | INVARIANT | per-PG_* index < pageflag_names.len(). |
| `vm_bug_on_only_in_DEBUG_VM` | INVARIANT | per-non-DEBUG_VM: VM_BUG_ON evaluates to no-op. |
| `dump_page_handles_null` | INVARIANT | dump_page(NULL): doesn't deref; just prints message. |
| `page_init_poison_value_unique` | INVARIANT | PAGE_POISON_VALUE not zero (so detectable). |

### Layer 2: TLA+

`mm/mm_debug.tla`:
- Per-dump_* / VM_BUG_ON invariant-check.
- Properties:
  - `safety_no_bug_on_normal_path` — per-correct-state code-path: no VM_BUG_ON fires.
  - `safety_dump_doesnt_corrupt` — per-dump-fn read-only.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MmDebug::dump_page` post: read-only side-effect (log only) | `MmDebug::dump_page` |
| `MmDebug::pgtable_init` post: synthetic mm freed | `MmDebug::pgtable_init` |
| `MmDebug::rodata_test` post: panic if rodata writable; else continue | `MmDebug::rodata_test` |

### Layer 4: Verus/Creusot functional

`Per-VM_BUG_ON / dump_* / debug_vm_pgtable_init triggers at runtime per-CONFIG_DEBUG_VM` semantic equivalence: per-Documentation/dev-tools/mm-debug.rst.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

mm-debug-specific reinforcement:

- **Per-VM_BUG_ON in CONFIG_DEBUG_VM only** — defense against per-prod halt-on-warning.
- **Per-dump-fn null-checks** — defense against per-NULL deref during oops.
- **Per-rodata_test gated by CONFIG_DEBUG_RODATA_TEST** — defense against per-prod overhead.
- **Per-page_init_poison value-unique** — defense against per-zero-init confusion.
- **Per-debug_vm_pgtable_init synthetic-mm freed** — defense against per-boot leak.
- **Per-debug_page_ref via tracepoint** — defense against per-trace-flood.
- **Per-PAGE_POISON 0xAA bytes recognizable** — defense against per-oops misdiagnosis.
- **Per-dump_pte / dump_pmd / dump_pud / dump_pgd use arch-specific decoders** — defense against per-arch bit-misinterpret.

