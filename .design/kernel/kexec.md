# Tier-3: kernel/kexec_core.c — kexec / crash kernel boot

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/kexec_core.c (~1375 lines)
  - kernel/kexec.c (SYSCALL_DEFINE4 kexec_load)
  - kernel/kexec_file.c (SYSCALL_DEFINE5 kexec_file_load, kexec_calculate_store_digests, purgatory mgmt)
  - kernel/crash_core.c (__crash_kexec, crash_kexec)
  - kernel/kexec_internal.h
  - include/linux/kexec.h
  - include/uapi/linux/kexec.h (KEXEC_*)
-->

## Summary

`kexec_load(2)` / `kexec_file_load(2)` syscalls stage a new kernel image into
RAM so that the running kernel can warm-reboot into it (or, for the crash
variant `KEXEC_ON_CRASH`, jump into a preloaded crash kernel after panic).
Per-image context: `struct kimage` (segments, control pages, indirection
pages, head/entry list, control_code_page, type). Per-`SYSCALL_DEFINE4
kexec_load` path uses user-supplied source buffers; per-`SYSCALL_DEFINE5
kexec_file_load` uses an in-kernel `kernel_read_file` of a `.bzImage`-style
file with a `purgatory_info` blob whose SHA-256 digests are stored via
`kexec_calculate_store_digests`. Per-`sanity_check_segment_list` validates
segment alignment, overlap, and ≤50%-of-RAM budget. Per-`kimage_load_segment`
copies into either normally-allocated relocate-aware pages (default) or
reserved crashk_res memory (crash). Per-`machine_kexec` performs the
no-return jump after CPU offlining + console suspend + kmsg dump.
Per-`crash_kexec` is invoked from `panic()` / oops path. Critical for:
warm reboot, kdump, livepatch fallback, secure-boot-aware reboot.

This Tier-3 covers `kernel/kexec_core.c` (~1375 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE4(kexec_load, ...)` | per-syscall (buffer-based) | `KexecSys::kexec_load` |
| `SYSCALL_DEFINE5(kexec_file_load, ...)` | per-syscall (fd-based) | `KexecSys::kexec_file_load` |
| `struct kimage` | per-image staging context | `Kimage` |
| `struct kexec_segment` | per-segment user descriptor | `KexecSegment` |
| `do_kimage_alloc_init()` | per-image kzalloc init | `Kimage::alloc_init` |
| `kimage_alloc_init()` | per-load alloc + sanity + control pages | `Kimage::full_init` |
| `sanity_check_segment_list()` | per-segments validation | `Kimage::sanity_check` |
| `kimage_load_segment()` | per-segment copy dispatch | `Kimage::load_segment` |
| `kimage_load_normal_segment()` | per-non-crash copy | `Kimage::load_normal_segment` |
| `kimage_load_crash_segment()` | per-crash copy into crashk_res | `Kimage::load_crash_segment` |
| `kimage_load_cma_segment()` | per-CMA-backed segment | `Kimage::load_cma_segment` |
| `kimage_alloc_control_pages()` | per-control-page alloc (relocation-aware) | `Kimage::alloc_control_pages` |
| `kimage_alloc_normal_control_pages()` | per-normal control-page alloc | `Kimage::alloc_normal_control_pages` |
| `kimage_alloc_crash_control_pages()` | per-crash control-page alloc | `Kimage::alloc_crash_control_pages` |
| `kimage_alloc_page()` | per-page-aware reloc-safe alloc | `Kimage::alloc_page` |
| `kimage_alloc_pages()` | per-order alloc | `Kimage::alloc_pages` |
| `kimage_free()` | per-image teardown | `Kimage::free` |
| `kimage_terminate()` | per-list end marker (IND_DONE) | `Kimage::terminate` |
| `kimage_add_entry()` / `kimage_add_page()` / `kimage_set_destination()` | per-relocation list build | `Kimage::add_entry` / `add_page` / `set_destination` |
| `kimage_is_destination_range()` | per-overlap query | `Kimage::is_destination_range` |
| `kimage_map_segment()` / `kimage_unmap_segment()` | per-IMA vmap of staged seg | `Kimage::map_segment` / `unmap_segment` |
| `kexec_calculate_store_digests()` | per-SHA256 over segments + purgatory store | `Kimage::calculate_store_digests` |
| `kexec_purgatory` / `kexec_purgatory_size` | per-arch trampoline blob | `KEXEC_PURGATORY` const |
| `machine_kexec()` | per-arch no-return jump | arch `machine_kexec` |
| `machine_kexec_prepare()` / `_post_load()` / `_cleanup()` | per-arch hooks | arch hooks |
| `machine_crash_shutdown()` | per-arch crash-only setup | arch hook |
| `kernel_kexec()` | per-warm-reboot entry | `Kexec::kernel_kexec` |
| `crash_kexec()` / `__crash_kexec()` | per-panic jump | `Kexec::crash_kexec` |
| `kexec_should_crash(p)` | per-task crash policy | `Kexec::should_crash` |
| `kexec_in_progress` | per-flag (freeze) | `KEXEC_IN_PROGRESS` |
| `__kexec_lock` / `kexec_trylock()` / `kexec_unlock()` | per-syscall serialization | `KEXEC_LOCK` |
| `kexec_load_permitted()` | per-CAP_SYS_BOOT + lockdown + limit | `Kexec::load_permitted` |
| `kexec_load_disabled` | per-sysctl boolean (one-way) | `KEXEC_LOAD_DISABLED` |
| `struct kexec_load_limit` | per-(reboot/panic) counter | `KexecLoadLimit` |
| `kexec_image` / `kexec_crash_image` | per-staged kimage | `KEXEC_IMAGE` / `KEXEC_CRASH_IMAGE` |
| `kimage_crash_copy_vmcoreinfo()` | per-vmcoreinfo copy | `Kimage::copy_vmcoreinfo` |
| `migrate_to_reboot_cpu()` / `cpu_hotplug_enable()` | per-CPU freeze | shared |
| `freeze_processes()` / `thaw_processes()` | per-userspace freeze (KEXEC_JUMP) | `pm/process.c` |
| `dpm_suspend_start` / `dpm_suspend_end` / `syscore_suspend` / `syscore_shutdown` | per-device pipeline | drivers/base/power |

## Compatibility contract

REQ-1: struct kimage (key fields):
- entry: pointer-into-head, current write cursor.
- last_entry: end-of-current-indirection-page.
- head: kimage_entry_t — root of the relocation list.
- control_page: starting search address for crash-kernel control allocator (default ~0).
- start: u64 — entry-point physical address (image->start, taken from syscall `entry`).
- type: KEXEC_TYPE_DEFAULT | KEXEC_TYPE_CRASH.
- preserve_context: bool — KEXEC_PRESERVE_CONTEXT (kexec-jump).
- hotplug_support: bool — KEXEC_FILE_NO_AUTO_RELOAD analogue (crash hotplug).
- nr_segments: u8 ≤ KEXEC_SEGMENT_MAX (16).
- segment[KEXEC_SEGMENT_MAX]: kexec_segment { buf, bufsz, mem, memsz, kbuf }.
- segment_cma[KEXEC_SEGMENT_MAX]: struct page * (per-CMA-allocated segs).
- control_pages: list_head — list of allocated control pages.
- dest_pages: list_head — destination-page cache.
- unusable_pages: list_head — out-of-range alloc leftovers.
- control_code_page: struct page * — trampoline code home.
- swap_page: struct page * — KEXEC_TYPE_DEFAULT only.
- file_mode: bool — set if loaded via kexec_file_load.
- purgatory_info: struct purgatory_info — file-load only.
- vmcoreinfo_data_copy: void * — crash-only.
- elfcorehdr_index / elfcorehdr_updated / hp_action: CRASH_HOTPLUG.

REQ-2: SYSCALL kexec_load(entry, nr_segments, segments_user, flags):
- kexec_load_check(nr_segments, flags):
  - kexec_load_permitted(image_type) — CAP_SYS_BOOT ∧ !kexec_load_disabled ∧ per-(reboot/panic) limit counter.
  - security_kernel_load_data(LOADING_KEXEC_IMAGE, false) — LSM hook.
  - security_locked_down(LOCKDOWN_KEXEC) — refuse if lockdown.
  - flags ⊆ KEXEC_FLAGS ∪ KEXEC_ARCH_MASK.
  - nr_segments ≤ KEXEC_SEGMENT_MAX.
- Verify ((flags & KEXEC_ARCH_MASK) == KEXEC_ARCH ∨ == KEXEC_ARCH_DEFAULT).
- ksegments = memdup_array_user(segments, nr_segments, sizeof(kexec_segment)).
- do_kexec_load(entry, nr_segments, ksegments, flags).
- kfree(ksegments).

REQ-3: do_kexec_load(entry, nr_segments, segments, flags):
- if !kexec_trylock(): -EBUSY.
- dest_image = (flags & KEXEC_ON_CRASH) ? &kexec_crash_image : &kexec_image.
- if nr_segments == 0: kimage_free(xchg(dest_image, NULL)); return 0.   /* unload */
- if KEXEC_ON_CRASH: kimage_free(xchg(&kexec_crash_image, NULL)) — replace.
- kimage_alloc_init(&image, entry, nr_segments, segments, flags):
  - do_kimage_alloc_init → kzalloc; init head/list; type=DEFAULT.
  - image.start = entry; nr_segments + memcpy segments[].
  - if KEXEC_ON_CRASH: image.control_page = crashk_res.start; image.type = KEXEC_TYPE_CRASH.
  - sanity_check_segment_list(image).
  - image.control_code_page = kimage_alloc_control_pages(image, get_order(KEXEC_CONTROL_PAGE_SIZE)).
  - if !KEXEC_ON_CRASH: image.swap_page = kimage_alloc_control_pages(image, 0).
- if flags & KEXEC_PRESERVE_CONTEXT: image.preserve_context = 1.
- if CONFIG_CRASH_HOTPLUG ∧ KEXEC_ON_CRASH ∧ arch_crash_hotplug_support: image.hotplug_support = 1.
- machine_kexec_prepare(image).
- kimage_crash_copy_vmcoreinfo(image).
- For i in 0..nr_segments: kimage_load_segment(image, i).
- kimage_terminate(image) — append IND_DONE.
- machine_kexec_post_load(image).
- image = xchg(dest_image, image); kimage_free(image); kexec_unlock.

REQ-4: SYSCALL kexec_file_load(kernel_fd, initrd_fd, cmdline_len, cmdline_ptr, flags):
- kexec_load_permitted (same caps).
- kernel_read_file_from_fd(kernel_fd, &image.kernel_buf, INT_MAX, NULL, READING_KEXEC_IMAGE).
- if !(flags & KEXEC_FILE_NO_INITRAMFS): kernel_read_file_from_fd(initrd_fd).
- copy cmdline.
- arch_kexec_kernel_image_probe — match an arch loader (bzImage / elf / etc.).
- arch_kexec_kernel_image_load — produce in-kernel-allocated segments.
- kexec_calculate_store_digests(image) — SHA-256 over each segment (skipping purgatory + elfcorehdr_index + ima_segment_index); write digest + sha_regions[] into purgatory via kexec_purgatory_get_set_symbol.
- machine_kexec_prepare; kimage_load_segment loop; kimage_terminate.

REQ-5: sanity_check_segment_list(image):
- For each segment i:
  - mstart = mem; mend = mem + memsz.
  - if mstart > mend: -EADDRNOTAVAIL.
  - if mstart ≠ PAGE_ALIGN ∨ mend ≠ PAGE_ALIGN: -EADDRNOTAVAIL.
  - if mend ≥ KEXEC_DESTINATION_MEMORY_LIMIT: -EADDRNOTAVAIL.
- For each pair (i,j<i): overlap → -EINVAL.
- For each i: bufsz > memsz → -EINVAL.
- For each i: PAGE_COUNT(memsz) > totalram_pages()/2 → -EINVAL.
- total_pages = Σ PAGE_COUNT(memsz); if total > totalram/2 → -EINVAL.
- if CRASH ∧ image.type == KEXEC_TYPE_CRASH:
  - For each i: mstart < phys_to_boot_phys(crashk_res.start) ∨ mend > phys_to_boot_phys(crashk_res.end) → -EADDRNOTAVAIL.
- accept_memory(mem, memsz) — pre-accept (TDX/SEV).

REQ-6: kimage_load_segment(image, idx):
- DEFAULT: kimage_load_normal_segment.
- CRASH: kimage_load_crash_segment.

REQ-7: kimage_load_normal_segment(image, idx):
- If image.segment_cma[idx]: kimage_load_cma_segment.
- kimage_set_destination(image, maddr).
- While mbytes > 0:
  - page = kimage_alloc_page(image, GFP_HIGHUSER, maddr).
  - kimage_add_page(image, page_to_boot_pfn(page) << PAGE_SHIFT).
  - kmap_local_page; clear_page; copy_from_user (or memcpy for file_mode); kunmap_local.
  - maddr += mchunk; mbytes -= mchunk.

REQ-8: kimage_load_crash_segment(image, idx):
- For each PAGE_SIZE chunk of destination maddr in reserved crashk_res:
  - page = boot_pfn_to_page(maddr >> PAGE_SHIFT).
  - arch_kexec_post_alloc_pages; kmap_local_page; zero trailing region; copy_from_user / memcpy; kexec_flush_icache_page; kunmap_local; arch_kexec_pre_free_pages.

REQ-9: kimage_alloc_control_pages(image, order):
- KEXEC_TYPE_DEFAULT: kimage_alloc_normal_control_pages — loop alloc_pages until got pages not overlapping any segment destination and below KEXEC_CONTROL_MEMORY_LIMIT.
- KEXEC_TYPE_CRASH: kimage_alloc_crash_control_pages — first-hole search in [image.control_page .. crashk_res.end], rejecting overlap with segments.

REQ-10: kimage_alloc_page(image, gfp_mask, destination):
- Search image.dest_pages for cached page at destination addr; if found, list_del + return.
- Else loop:
  - page = kimage_alloc_pages(gfp_mask, 0).
  - if pfn > KEXEC_SOURCE_MEMORY_LIMIT: list_add image.unusable_pages; continue.
  - addr = pfn << PAGE_SHIFT.
  - if addr == destination: break.
  - if !kimage_is_destination_range(image, addr, addr+PAGE_SIZE-1): break.
  - /* addr is some other destination → see if a source already maps it */
  - old = kimage_dst_used(image, addr).
  - if old: copy_highpage(page, old_page); rewrite *old = addr | flags; pick old_page as our return (subject to __GFP_HIGHMEM honor).
  - else: list_add(&page->lru, &image.dest_pages); continue.

REQ-11: kimage_add_entry(image, entry):
- if *image.entry != 0: advance image.entry.
- if image.entry == image.last_entry:
  - alloc new indirection page; *image.entry = virt_to_boot_phys(ind) | IND_INDIRECTION.
  - image.entry = ind_page; image.last_entry = ind_page + (PAGE_SIZE/sizeof(entry)) - 1.
- *image.entry = entry; image.entry++; *image.entry = 0 (sentinel).

REQ-12: Entry-list flags:
- IND_DESTINATION — sets sliding destination cursor.
- IND_SOURCE — copy src→destination.
- IND_INDIRECTION — chained list page.
- IND_DONE — terminator.

REQ-13: kimage_terminate(image):
- if *image.entry != 0: image.entry++.
- *image.entry = IND_DONE.

REQ-14: kimage_free(image):
- vunmap vmcoreinfo_data_copy if any.
- kimage_free_extra_pages — drop dest_pages + unusable_pages.
- Walk entries: free IND_SOURCE pages; free indirection pages.
- machine_kexec_cleanup(image).
- kimage_free_page_list(&image.control_pages).
- kimage_free_cma — dma_release_from_contiguous per segment_cma[].
- if image.file_mode: kimage_file_post_load_cleanup — kfree(image.kernel_buf, initrd_buf, cmdline_buf, purgatory_info segments).
- kfree(image).

REQ-15: kexec_calculate_store_digests(image):
- if !CONFIG_ARCH_SUPPORTS_KEXEC_PURGATORY: return 0.
- Allocate sha_regions[KEXEC_SEGMENT_MAX].
- sha256_init.
- For each segment i:
  - if i == image.elfcorehdr_index (CRASH_HOTPLUG): skip.
  - if ksegment.kbuf == purgatory_buf: skip.
  - if check_ima_segment_index(image, i): skip.
  - sha256_update(ksegment.kbuf, ksegment.bufsz).
  - Pad remainder (memsz - bufsz) with ZERO_PAGE bytes.
  - sha_regions[j++] = { mem, memsz }.
- sha256_final → digest.
- kexec_purgatory_get_set_symbol(image, "purgatory_sha_regions", sha_regions, sha_region_sz, 0).
- kexec_purgatory_get_set_symbol(image, "purgatory_sha256_digest", digest, SHA256_DIGEST_SIZE, 0).

REQ-16: kernel_kexec() — warm reboot:
- if !kexec_trylock(): -EBUSY.
- if !kexec_image: -EINVAL.
- liveupdate_reboot.
- KEXEC_JUMP path (kexec_image.preserve_context):
  - pm_prepare_console; freeze_processes; console_suspend_all.
  - dpm_suspend_start(PMSG_FREEZE); dpm_suspend_end(PMSG_FREEZE).
  - suspend_disable_secondary_cpus.
  - local_irq_disable; syscore_suspend.
- Else:
  - kexec_in_progress = true.
  - kernel_restart_prepare("kexec reboot").
  - migrate_to_reboot_cpu — pin to a CPU, offline the rest.
  - syscore_shutdown.
  - cpu_hotplug_enable.
  - machine_shutdown.
- kmsg_dump(KMSG_DUMP_SHUTDOWN).
- machine_kexec(kexec_image) — no return on success.
- KEXEC_JUMP resume path (only if preserve_context returned).

REQ-17: crash_kexec(regs) — panic path:
- panic_try_start (atomic compare-and-swap on panic_cpu).
- __crash_kexec(regs):
  - if kexec_trylock():
    - if kexec_crash_image:
      - crash_setup_regs(&fixed_regs, regs) — capture register state.
      - crash_save_vmcoreinfo.
      - machine_crash_shutdown(&fixed_regs) — IPI all CPUs, halt them, save per-CPU state into crash notes.
      - crash_cma_clear_pending_dma — mdelay(CMA_DMA_TIMEOUT_SEC * 1000) if crashk_cma_cnt > 0.
      - machine_kexec(kexec_crash_image) — no return on success.
    - kexec_unlock.
- panic_reset.

REQ-18: kexec_should_crash(p):
- if crash_kexec_post_notifiers: 0 (deferred).
- if in_interrupt ∨ !p->pid ∨ is_global_init(p) ∨ panic_on_oops: 1.
- else: 0.

REQ-19: kexec_load_permitted(image_type):
- !CAP_SYS_BOOT ∨ kexec_load_disabled: false.
- limit = (image_type == CRASH) ? load_limit_panic : load_limit_reboot.
- mutex_lock(&limit.mutex). if !limit.limit: false. if limit.limit != -1: limit.limit--. mutex_unlock.
- return true.

REQ-20: kexec_in_progress (boolean global):
- Set true at start of non-JUMP kernel_kexec path; never reset (no return).
- Read by various subsystems (e.g. workqueue, console) to short-circuit teardown.

REQ-21: __kexec_lock / kexec_trylock / kexec_unlock:
- atomic_t __kexec_lock = ATOMIC_INIT(0).
- kexec_trylock = atomic_cmpxchg(&__kexec_lock, 0, 1) == 0.
- kexec_unlock = atomic_set(&__kexec_lock, 0).

## Acceptance Criteria

- [ ] AC-1: kexec_load(2) with CAP_SYS_BOOT + valid segments → kexec_image populated; second syscall with nr_segments=0 frees it.
- [ ] AC-2: kexec_load(2) without CAP_SYS_BOOT → -EPERM.
- [ ] AC-3: kexec_load(2) with kexec_load_disabled=1 → -EPERM.
- [ ] AC-4: kexec_load(2) with overlapping segments → -EINVAL.
- [ ] AC-5: kexec_load(2) with unaligned `mem` or `memsz` → -EADDRNOTAVAIL.
- [ ] AC-6: kexec_load(2) with bufsz > memsz → -EINVAL.
- [ ] AC-7: kexec_load(2) total pages > totalram/2 → -EINVAL.
- [ ] AC-8: KEXEC_ON_CRASH with destination outside crashk_res → -EADDRNOTAVAIL.
- [ ] AC-9: kexec_file_load(2): purgatory SHA-256 digest matches segment contents (excluding purgatory itself + elfcorehdr).
- [ ] AC-10: kexec_load_limit_reboot=N: after N successful loads, next load → false.
- [ ] AC-11: reboot -kexec → kernel_kexec → machine_kexec executed; no return.
- [ ] AC-12: panic() with kexec_crash_image loaded → __crash_kexec → machine_kexec(kexec_crash_image); machine_crash_shutdown halts secondary CPUs.
- [ ] AC-13: panic() with no crash image → __crash_kexec is a no-op.
- [ ] AC-14: Concurrent kexec_load attempts: kexec_trylock returns -EBUSY for the loser.
- [ ] AC-15: LOCKDOWN_KEXEC: kexec_load_check → -EPERM.

## Architecture

```
struct Kimage {
  entry: *KimageEntry,
  last_entry: *KimageEntry,
  head: KimageEntry,
  control_page: u64,
  start: u64,
  type: KexecType,            // DEFAULT / CRASH
  preserve_context: bool,
  hotplug_support: bool,
  nr_segments: u32,
  segment: [KexecSegment; KEXEC_SEGMENT_MAX],
  segment_cma: [Option<*Page>; KEXEC_SEGMENT_MAX],
  control_pages: ListHead,
  dest_pages: ListHead,
  unusable_pages: ListHead,
  control_code_page: *Page,
  swap_page: *Page,
  file_mode: bool,
  purgatory_info: PurgatoryInfo,
  vmcoreinfo_data_copy: *u8,
  // CRASH_HOTPLUG
  elfcorehdr_index: i32,
  elfcorehdr_updated: bool,
  hp_action: u32,
}

struct KexecSegment {
  buf: *const u8,             // user-space pointer (kexec_load)
  bufsz: usize,
  mem: u64,                   // physical destination
  memsz: usize,
  kbuf: *const u8,            // kernel pointer (kexec_file_load)
}

const IND_DESTINATION: u64 = 1 << 0;
const IND_INDIRECTION: u64 = 1 << 1;
const IND_DONE       : u64 = 1 << 2;
const IND_SOURCE     : u64 = 1 << 3;
```

`KexecSys::kexec_load(entry, nr_segments, segments, flags) -> i32`:
1. kexec_load_check — caps, lockdown, LSM, flag mask, nr_segments cap.
2. Verify (flags & KEXEC_ARCH_MASK) == KEXEC_ARCH ∨ == DEFAULT.
3. ksegments = memdup_array_user(segments, nr_segments).
4. do_kexec_load(entry, nr_segments, ksegments, flags).
5. kfree(ksegments).

`Kexec::do_kexec_load(entry, nr_segments, segments, flags) -> i32`:
1. kexec_trylock; else -EBUSY.
2. dest_image = (flags & KEXEC_ON_CRASH) ? &kexec_crash_image : &kexec_image.
3. if nr_segments == 0: kimage_free(xchg(dest_image, NULL)); return 0.
4. if KEXEC_ON_CRASH: kimage_free(xchg(&kexec_crash_image, NULL)).
5. kimage_alloc_init(&image, entry, nr_segments, segments, flags):
   - do_kimage_alloc_init.
   - image.start = entry; copy segments.
   - if CRASH: image.control_page = crashk_res.start; type = CRASH.
   - sanity_check_segment_list(image).
   - image.control_code_page = kimage_alloc_control_pages(image, get_order(KEXEC_CONTROL_PAGE_SIZE)).
   - if !CRASH: image.swap_page = kimage_alloc_control_pages(image, 0).
6. if KEXEC_PRESERVE_CONTEXT: image.preserve_context = 1.
7. if CRASH_HOTPLUG ∧ CRASH ∧ arch_crash_hotplug_support: image.hotplug_support = 1.
8. machine_kexec_prepare(image).
9. kimage_crash_copy_vmcoreinfo(image).
10. for i in 0..nr_segments: kimage_load_segment(image, i).
11. kimage_terminate(image).
12. machine_kexec_post_load(image).
13. image = xchg(dest_image, image); kimage_free(image).
14. kexec_unlock; return.

`Kimage::sanity_check(image) -> i32`:
1. For each i in 0..nr_segments:
   - mstart = segment[i].mem; mend = mstart + memsz.
   - mstart > mend → -EADDRNOTAVAIL.
   - !PAGE_ALIGN(mstart) ∨ !PAGE_ALIGN(mend) → -EADDRNOTAVAIL.
   - mend ≥ KEXEC_DESTINATION_MEMORY_LIMIT → -EADDRNOTAVAIL.
2. For each pair: overlap → -EINVAL.
3. For each i: bufsz > memsz → -EINVAL.
4. For each i: PAGE_COUNT(memsz) > totalram_pages()/2 → -EINVAL.
5. total > totalram/2 → -EINVAL.
6. CRASH: each segment ⊆ crashk_res or → -EADDRNOTAVAIL.
7. accept_memory(mem, memsz) per segment.

`Kimage::load_segment(image, idx) -> i32`:
1. Match image.type:
   - DEFAULT: kimage_load_normal_segment.
   - CRASH:   kimage_load_crash_segment.

`Kimage::load_normal_segment(image, idx) -> i32`:
1. If segment_cma[idx]: kimage_load_cma_segment.
2. kimage_set_destination(image, maddr).
3. Loop while mbytes > 0:
   - page = kimage_alloc_page(image, GFP_HIGHUSER, maddr).
   - kimage_add_page(image, page_to_boot_pfn(page) << PAGE_SHIFT).
   - kmap_local_page(page); clear_page; copy_from_user (or memcpy if file_mode); kunmap_local.
   - maddr += PAGE_SIZE; mbytes -= mchunk.

`Kimage::alloc_control_pages(image, order) -> *Page`:
1. DEFAULT: alloc_normal_control_pages — loop alloc until non-destination, in-range.
2. CRASH: alloc_crash_control_pages — first-hole scan over image.control_page..crashk_res.end.

`Kimage::calculate_store_digests(image) -> i32` (per kexec_file.c):
1. Allocate sha_regions[KEXEC_SEGMENT_MAX].
2. sha256_init.
3. For each segment i (skipping elfcorehdr_index, purgatory_buf, ima_segment_index):
   - sha256_update(kbuf, bufsz).
   - Pad (memsz - bufsz) bytes via ZERO_PAGE in PAGE_SIZE chunks.
   - sha_regions[j++] = { mem, memsz }.
4. sha256_final → digest[32].
5. kexec_purgatory_get_set_symbol(image, "purgatory_sha_regions", sha_regions, ...).
6. kexec_purgatory_get_set_symbol(image, "purgatory_sha256_digest", digest, 32, 0).

`Kexec::kernel_kexec() -> i32`:
1. kexec_trylock; else -EBUSY.
2. if !kexec_image: -EINVAL.
3. liveupdate_reboot.
4. KEXEC_JUMP: freeze_processes; console_suspend; dpm_suspend; suspend_disable_secondary_cpus; local_irq_disable; syscore_suspend.
5. ELSE: kexec_in_progress = true; kernel_restart_prepare; migrate_to_reboot_cpu; syscore_shutdown; cpu_hotplug_enable; machine_shutdown.
6. kmsg_dump(KMSG_DUMP_SHUTDOWN).
7. machine_kexec(kexec_image).  /* no return */

`Kexec::crash_kexec(regs)`:
1. panic_try_start (cmpxchg panic_cpu).
2. __crash_kexec(regs):
   - kexec_trylock.
   - if kexec_crash_image:
     - crash_setup_regs(&fixed_regs, regs).
     - crash_save_vmcoreinfo.
     - machine_crash_shutdown(&fixed_regs).
     - crash_cma_clear_pending_dma.
     - machine_kexec(kexec_crash_image).  /* no return */
   - kexec_unlock.
3. panic_reset.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kexec_caps_required` | INVARIANT | per-kexec_load_permitted: !CAP_SYS_BOOT ⟹ false. |
| `kexec_lockdown_blocks` | INVARIANT | per-kexec_load_check: LOCKDOWN_KEXEC ⟹ refused. |
| `kexec_load_disabled_oneway` | INVARIANT | per-sysctl: kexec_load_disabled monotonically 0 → 1 (proc_dointvec_minmax extra1=extra2=ONE). |
| `segment_alignment` | INVARIANT | per-sanity_check_segment_list: PAGE_ALIGN(mem) ∧ PAGE_ALIGN(memsz). |
| `segment_no_overlap` | INVARIANT | per-sanity_check_segment_list: ∀ i≠j, [mem_i,mem_i+memsz_i) ∩ [mem_j,mem_j+memsz_j) = ∅. |
| `segment_budget` | INVARIANT | per-sanity_check_segment_list: Σ PAGE_COUNT(memsz_i) ≤ totalram_pages()/2. |
| `crash_segments_in_crashkres` | INVARIANT | per-CRASH type: ∀ i, segment_i ⊆ crashk_res. |
| `kexec_lock_mutex_exclusion` | INVARIANT | per-kexec_trylock: at most one in-progress kexec_load / kernel_kexec / crash_kexec. |
| `kimage_alloc_no_destination` | INVARIANT | per-kimage_alloc_normal_control_pages: returned page ∉ kimage_is_destination_range. |
| `digest_excludes_purgatory_itself` | INVARIANT | per-kexec_calculate_store_digests: skip if ksegment.kbuf == purgatory_buf. |
| `crash_kexec_serialized` | INVARIANT | per-__crash_kexec: kexec_trylock held; only panic_cpu enters. |
| `kexec_in_progress_one_way` | INVARIANT | per-non-JUMP kernel_kexec: kexec_in_progress transitions false→true exactly once. |

### Layer 2: TLA+

`kernel/kexec.tla`:
- Per-kexec_load / kexec_file_load / kernel_kexec / crash_kexec + concurrent panics.
- Properties:
  - `safety_lock_exclusion` — per-__kexec_lock: at most one holder.
  - `safety_image_atomic_swap` — per-do_kexec_load: dest_image swapped only after full load.
  - `safety_segments_validated_before_load` — per-load: kimage_load_segment unreachable until sanity_check_segment_list passes.
  - `safety_crash_path_pinned_cpu` — per-crash_kexec: only panic_cpu reaches machine_kexec.
  - `safety_no_kexec_after_load_disabled` — per-kexec_load_disabled=1: subsequent loads refused.
  - `safety_limit_counter_monotonic` — per-kexec_load_permitted: limit.limit only decreases.
  - `safety_digest_covers_loaded_segments` — per-kexec_file_load: digest captures every segment in sha_regions list (modulo skip rules).
  - `liveness_kexec_load_terminates` — per-kexec_load: either error or kexec_image populated.
  - `liveness_panic_kexec_progresses` — per-crash_kexec: either machine_kexec (no return) or unlock.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Kimage::sanity_check` post: ret==0 ⟹ all segments aligned + non-overlapping + ≤half RAM | `Kimage::sanity_check` |
| `Kimage::alloc_control_pages` post: returned page below KEXEC_CONTROL_MEMORY_LIMIT ∧ not destination | `Kimage::alloc_control_pages` |
| `Kimage::add_entry` post: image.entry advances; *image.entry==0 sentinel after | `Kimage::add_entry` |
| `Kimage::terminate` post: *image.entry == IND_DONE | `Kimage::terminate` |
| `Kimage::load_normal_segment` post: for each src page, kimage_dst_used can locate it | `Kimage::load_normal_segment` |
| `Kimage::load_crash_segment` post: writes only inside crashk_res | `Kimage::load_crash_segment` |
| `Kimage::calculate_store_digests` post: SHA-256 over { segments \ {purgatory, elfcorehdr, ima} } stored in purgatory.sha_regions / sha256_digest | `Kimage::calculate_store_digests` |
| `Kexec::kernel_kexec` post: ret==0 ⟹ no return (machine_kexec) | `Kexec::kernel_kexec` |
| `Kexec::__crash_kexec` post: kexec_crash_image set ⟹ machine_kexec invoked | `Kexec::__crash_kexec` |
| `Kexec::load_permitted` post: ret ⟹ CAP_SYS_BOOT ∧ !kexec_load_disabled ∧ limit > 0 | `Kexec::load_permitted` |

### Layer 4: Verus/Creusot functional

`Per-kexec_load syscall → kexec_load_check (caps + lockdown + flag mask) → kimage_alloc_init (do_kimage_alloc_init + sanity_check_segment_list + control-pages alloc) → machine_kexec_prepare → kimage_load_segment loop → kimage_terminate → machine_kexec_post_load → atomic xchg into kexec_image / kexec_crash_image`, and `Per-panic → __crash_kexec → machine_crash_shutdown → machine_kexec(kexec_crash_image)`, and `Per-reboot -f kexec → kernel_kexec → migrate_to_reboot_cpu → syscore_shutdown → machine_kexec(kexec_image)` semantic equivalence: per-Documentation/admin-guide/kdump/kdump.rst + Documentation/admin-guide/mm/kexec.rst.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

kexec reinforcement:

- **Per-CAP_SYS_BOOT requirement** — defense against per-unprivileged kernel-replace.
- **Per-LOCKDOWN_KEXEC** — defense against per-secure-boot bypass via custom kernel.
- **Per-LSM hook (security_kernel_load_data + IMA)** — defense against per-policy-circumvention.
- **Per-kexec_load_disabled one-way sysctl (extra1=extra2=ONE)** — defense against per-runtime unlock.
- **Per-(reboot / panic) load-limit counters** — defense against per-DoS via repeated reloads.
- **Per-sanity_check_segment_list strict** — defense against per-malicious-segment-overlap / per-out-of-range write.
- **Per-segment ≤ half-RAM cap** — defense against per-alloc soft-lockup.
- **Per-PAGE_ALIGN destination** — defense against per-arbitrary-byte-overwrite.
- **Per-KEXEC_DESTINATION_MEMORY_LIMIT** — defense against per-high-mem clobber.
- **Per-CRASH segments must be ⊆ crashk_res** — defense against per-crash-image-into-live-kernel.
- **Per-KEXEC_CONTROL_MEMORY_LIMIT control-page bound** — defense against per-control-page in unsafe range.
- **Per-kexec_trylock (atomic_t)** — defense against per-concurrent kexec races.
- **Per-purgatory SHA-256 digest** — defense against per-segment tamper between load and boot.
- **Per-LOADING_KEXEC_IMAGE LSM/IMA load_data hook** — defense against per-untrusted-blob.
- **Per-memdup_array_user with overflow check** — defense against per-integer-overflow on size.
- **Per-accept_memory(mem, memsz)** — defense against per-unaccepted-memory (TDX/SEV-SNP) write.
- **Per-kimage_alloc_normal_control_pages reject destination overlap** — defense against per-trampoline-overwrite.
- **Per-machine_crash_shutdown halts secondaries** — defense against per-CPU-still-running during crash dump.
- **Per-crash_cma_clear_pending_dma mdelay** — defense against per-in-flight-DMA corrupting crash kernel.
- **Per-kexec_in_progress freeze flag** — defense against per-subsystem teardown reentry.
- **Per-kexec_load_check rejects unknown flag bits** — defense against per-forward-incompatible-flag attack.
- **Per-KEXEC_SEGMENT_MAX cap** — defense against per-unbounded-list inflation.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — copy-in of `kexec_segment[]` and image-buffer fragments is bounds-checked; `memdup_array_user` overflow checks remain mandatory.
- **PAX_KERNEXEC** — purgatory and kimage control pages are W^X; the live kernel never permits W+X on any segment during load.
- **PAX_RANDKSTACK** — randomize kernel stack on `sys_kexec_load`/`sys_kexec_file_load` entry.
- **PAX_REFCOUNT** — saturating atomics on `kexec_in_progress`, `crash_kexec_post_notifiers`, and image refcounts.
- **PAX_MEMORY_SANITIZE** — scrub all kexec segment staging pages on load failure or replacement so a previous user's blob cannot leak into a new load.
- **PAX_UDEREF** — strict user/kernel split when chasing user-supplied segment buffers.
- **PAX_RAP / kCFI** — type-signed indirect calls for `kexec_file_loaders[]` (`->probe`, `->load`, `->cleanup`, `->verify_sig`) and arch-specific `machine_kexec_*` hooks.
- **GRKERNSEC_HIDESYM** — hide `kexec_image`, `kexec_crash_image`, `crashk_res`, `machine_kexec_cleanup` from non-root /proc/kallsyms.
- **GRKERNSEC_DMESG** — restrict kexec load / purgatory diagnostics to CAP_SYSLOG.
- **kexec_load CAP_SYS_BOOT + signature verify** — `sys_kexec_load` (the legacy raw-segment path) is *disabled at build time* under hardened configs; only `kexec_file_load` is accepted, which mandates kernel signature verification via `KEXEC_SIG`/`KEXEC_SIG_FORCE` and rejects unsigned images even for CAP_SYS_BOOT.
- **kernel_lockdown integrity mode** — under lockdown=integrity or =confidentiality, kexec_file_load additionally requires the image be signed by a key in the platform keyring; secondary keyring trust is opt-in.
- **purgatory SHA-256 anchored** — the digest is computed at load and re-verified just before `machine_kexec`; mismatch panics rather than boots.
- **crashk_res containment hardened** — every segment destination address is re-checked against `crashk_res` and `crashk_low_res` immediately before relocation; trampoline-overwrite attempts panic.
- **machine_crash_shutdown halts secondaries** — confirmed under PAX_REFCOUNT-protected IPI fences; no secondary CPU may continue executing live-kernel text during crash transition.
- **kexec_in_progress freeze flag** — set under saturating refcount; reentry from a subsystem teardown notifier is denied with `WARN_ON_ONCE` and the kexec aborted.
- **TDX/SEV-SNP accept_memory** — every staged segment that lands in a confidential-VM is accepted before write; unaccepted-memory writes are rejected.
- **Rationale**: kexec replaces the running kernel — any unsigned blob loaded here defeats every other hardening in the system. Forcing kexec_file_load with KEXEC_SIG_FORCE plus lockdown-integrity platform-keyring anchoring makes "kexec a malicious kernel" a non-path even with CAP_SYS_BOOT.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/kexec_file.c parser internals + bzImage / image-PE loaders (covered in `kexec-file.md` Tier-3 if expanded)
- arch/x86/kernel/{machine_kexec_64.c,relocate_kernel_64.S} (covered in `arch/x86/machine-kexec.md` Tier-3 if expanded)
- kernel/crash_core.c vmcoreinfo + crash_notes (covered in `crash-core.md` Tier-3 if expanded)
- Documentation/admin-guide/kdump userspace tooling (out of scope — userspace)
- purgatory build glue (arch/x86/purgatory/*.c, S) — userspace-style trampoline (covered in `purgatory.md` Tier-3 if expanded)
- liveupdate_reboot pipeline (covered in `liveupdate.md` Tier-3 if expanded)
- Implementation code
