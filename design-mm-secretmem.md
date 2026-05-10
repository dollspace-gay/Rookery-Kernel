---
title: "Tier-3: mm/secretmem.c — memfd_secret(2) anonymous secret memory"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`memfd_secret(2)` returns an anonymous file descriptor whose pages, once faulted in via `mmap`, are **excluded from the kernel direct map** for the lifetime of the mapping. Each fault allocates a fresh `GFP_HIGHUSER | __GFP_ZERO` order-0 folio, calls `set_direct_map_invalid_noflush()` on it, inserts it into the file's `address_space`, and flushes the kernel TLB so that no kernel mapping of the page remains. Pages are pinned-unmovable (migration returns `-EBUSY`), unevictable, unswappable, and accounted against `RLIMIT_MEMLOCK`. The mapping is forced `VM_LOCKED | VM_DONTDUMP` and may only be created `MAP_SHARED` (`VMA_SHARED` or `VMA_MAYSHARE` bit). On folio free, the direct map is restored and the page is zeroed. Critical for: protecting user secrets (keys, tokens, password hashes) from kernel-resident speculative reads, Spectre-class side channels, and direct-map disclosure attacks; pairs with `F_SEAL_*` (memfd seals) to forbid resize/write/shrink. Hardening-only feature: requires `secretmem.enable=1` and `can_set_direct_map()` arch probe; otherwise `memfd_secret(2)` returns `-ENOSYS`.

This Tier-3 covers `mm/secretmem.c` (~270 lines).

### Acceptance Criteria

- [ ] AC-1: `memfd_secret(0)` returns a non-negative fd on a kernel with `secretmem.enable=1` and `can_set_direct_map() == true`.
- [ ] AC-2: `memfd_secret(0)` returns `-ENOSYS` when `secretmem.enable=0` or `can_set_direct_map() == false`.
- [ ] AC-3: `memfd_secret(O_CLOEXEC | 1)` returns `-EINVAL` (unknown flag bits).
- [ ] AC-4: `mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE, fd, 0)` on a secretmem fd returns `-EINVAL` (MAP_SHARED required).
- [ ] AC-5: After `ftruncate(fd, 4096); mmap(..., MAP_SHARED, fd, 0); *p = 0xaa;`, the faulted page is removed from the kernel direct map (kernel virtual `__va(pa)` mapping invalid).
- [ ] AC-6: After `munmap` + `close`, the page is restored to the direct map and zeroed.
- [ ] AC-7: `ftruncate(fd, 0)` then `ftruncate(fd, 8192)` once after first non-zero size returns `-EINVAL` (resize from non-zero forbidden).
- [ ] AC-8: A migration attempt (e.g., compaction) on a secretmem folio returns `-EBUSY`; folio is never moved.
- [ ] AC-9: Reading `/proc/$$/maps` shows the VMA flagged `lo` (VM_LOCKED) and `dd` (VM_DONTDUMP).
- [ ] AC-10: Coredump of a process with a secretmem VMA omits the VMA contents (VM_DONTDUMP).
- [ ] AC-11: `mlock`-future fail: when RLIMIT_MEMLOCK is exceeded by `len`, `mmap` returns `-EAGAIN`.
- [ ] AC-12: A read on the fd (no mmap) returns `-EINVAL` (no `.read_iter`).
- [ ] AC-13: `fs/proc/$$/smaps` shows the VMA as non-swappable (mapping unevictable).
- [ ] AC-14: `flush_tlb_kernel_range` is called per page after `set_direct_map_invalid_noflush` (kernel TLB invalidation occurs before fault returns).
- [ ] AC-15: `secretmem_users` counter increments and decrements symmetrically across open/close.

### Architecture

```
struct SecretMem {
  mount: *VfsMount,                  // secretmem_mnt
  users: AtomicI32,                  // secretmem_users
  enabled: bool,                     // secretmem_enable (module_param, RO post init)
}
```

`SecretMem::sys_memfd_secret(flags: u32) -> Result<RawFd, Errno>`:
1. /* Build-time invariant */
2. const_assert!(SECRETMEM_FLAGS_MASK & O_CLOEXEC == 0).
3. if !SecretMem::ENABLED.load(Relaxed) || !arch::can_set_direct_map(): return Err(ENOSYS).
4. if flags & !(SECRETMEM_FLAGS_MASK | O_CLOEXEC) != 0: return Err(EINVAL).
5. if SecretMem::USERS.load(Relaxed) < 0: return Err(ENFILE).
6. file = SecretMem::file_create(flags)?
7. fd_add(flags & O_CLOEXEC, file)

`SecretMem::file_create(flags) -> Result<File, Errno>`:
1. inode = anon_inode_make_secure_inode(MNT.sb, "[secretmem]", None)?
2. file = alloc_file_pseudo(inode, MNT, "secretmem", O_RDWR | O_LARGEFILE, &F_OPS).map_err(|e| { iput(inode); e })?
3. mapping_set_gfp_mask(inode.i_mapping, GFP_HIGHUSER).
4. mapping_set_unevictable(inode.i_mapping).
5. inode.i_op = &I_OPS.
6. inode.i_mapping.a_ops = &A_OPS.
7. inode.i_mode |= S_IFREG.
8. inode.i_size = 0.
9. USERS.fetch_add(1, Relaxed).
10. Ok(file).

`SecretMem::mmap_prepare(desc: &mut VmAreaDesc) -> Result<(), Errno>`:
1. len = desc.size().
2. if !desc.test_any_flags(VMA_SHARED_BIT | VMA_MAYSHARE_BIT): return Err(EINVAL).
3. desc.set_flags(VMA_LOCKED_BIT | VMA_DONTDUMP_BIT).
4. if !mlock_future_ok(desc.mm, /*locked=*/true, len): return Err(EAGAIN).
5. desc.vm_ops = &VM_OPS.
6. Ok(()).

`SecretMem::fault(vmf: &mut VmFault) -> VmFaultRet`:
1. mapping = vmf.vma.vm_file.f_mapping.
2. inode = file_inode(vmf.vma.vm_file).
3. offset = vmf.pgoff; gfp = vmf.gfp_mask.
4. if (offset as u64) << PAGE_SHIFT >= i_size_read(inode): return vmf_error(-EINVAL).
5. filemap_invalidate_lock_shared(mapping).
6. ret = (|| loop {
     match filemap_lock_folio(mapping, offset) {
       Ok(folio) => { vmf.page = folio_file_page(folio, offset); return VM_FAULT_LOCKED; }
       Err(_) => {
         folio = folio_alloc(gfp | __GFP_ZERO, /*order=*/0).ok_or(VM_FAULT_OOM)?;
         arch::set_direct_map_invalid_noflush(folio.page(0)).map_err(|e| { folio_put(folio); vmf_error(e) })?;
         __folio_mark_uptodate(folio);
         match filemap_add_folio(mapping, folio, offset, gfp) {
           Ok(_) => {
             addr = folio.address() as u64;
             arch::flush_tlb_kernel_range(addr, addr + PAGE_SIZE);
             vmf.page = folio_file_page(folio, offset);
             return VM_FAULT_LOCKED;
           }
           Err(-EEXIST) => { arch::set_direct_map_default_noflush(folio.page(0)); folio_put(folio); continue; }
           Err(e) => { arch::set_direct_map_default_noflush(folio.page(0)); folio_put(folio); return vmf_error(e); }
         }
       }
     }
   })();
7. filemap_invalidate_unlock_shared(mapping).
8. ret.

`SecretMem::free_folio(folio)`:
1. arch::set_direct_map_default_noflush(folio.page(0)).
2. folio_zero_segment(folio, 0, folio.size()).

`SecretMem::migrate_folio(mapping, dst, src, mode) -> i32`:
1. -EBUSY  /* secretmem is never migratable */.

`SecretMem::setattr(idmap, dentry, iattr) -> Result<(), Errno>`:
1. inode = d_inode(dentry); mapping = inode.i_mapping.
2. filemap_invalidate_lock(mapping).
3. ret = if (iattr.ia_valid & ATTR_SIZE) != 0 && inode.i_size != 0 { Err(EINVAL) } else { simple_setattr(idmap, dentry, iattr) }.
4. filemap_invalidate_unlock(mapping).
5. ret.

`SecretMem::release(_inode, _file) -> i32`:
1. USERS.fetch_sub(1, Relaxed).
2. 0.

`SecretMem::init_fs_context(fc) -> Result<(), Errno>`:
1. ctx = init_pseudo(fc, SECRETMEM_MAGIC).ok_or(ENOMEM)?
2. fc.s_iflags |= SB_I_NOEXEC.
3. fc.s_iflags |= SB_I_NODEV.
4. Ok(()).

`SecretMem::init() -> Result<(), Errno>` (fs_initcall):
1. if !ENABLED.load(Relaxed) || !arch::can_set_direct_map(): return Ok(()).
2. MNT.store(kern_mount(&FS_TYPE)?).
3. Ok(()).

### Out of Scope

- mm/memfd.c F_SEAL_* implementation (covered separately if expanded; `memfd_secret` uses pseudo-FS file, not `memfd_create` seals path, though seals semantics overlap)
- arch/x86/mm/pat/set_memory.c direct-map manipulation (covered in arch design)
- arch/arm64/mm/pageattr.c direct-map manipulation (covered in arch design)
- mm/gup.c pinning of secretmem pages (`secretmem_active()` consulted by GUP — covered in `gup.md` Tier-3)
- mm/migrate.c migration framework (covered in `migration-compaction.md` Tier-3)
- mm/mlock.c lock accounting (covered in `mlock.md` Tier-3)
- kernel/power/* hibernation interaction with `secretmem_active()` (covered separately if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE1(memfd_secret, flags)` | per-syscall entry | `SecretMem::sys_memfd_secret` |
| `secretmem_file_create()` | per-fd allocate | `SecretMem::file_create` |
| `secretmem_fault()` | per-fault page-in | `SecretMem::fault` |
| `secretmem_mmap_prepare()` | per-mmap VMA setup | `SecretMem::mmap_prepare` |
| `secretmem_release()` | per-close hook | `SecretMem::release` |
| `secretmem_setattr()` | per-truncate/size-change | `SecretMem::setattr` |
| `secretmem_migrate_folio()` | per-migration block (`-EBUSY`) | `SecretMem::migrate_folio` |
| `secretmem_free_folio()` | per-free restore | `SecretMem::free_folio` |
| `vma_is_secretmem()` | per-VMA probe | `SecretMem::vma_is_secretmem` |
| `secretmem_active()` | per-system probe | `SecretMem::active` |
| `secretmem_init_fs_context()` | per-pseudo-FS mount | `SecretMem::init_fs_context` |
| `secretmem_init()` | per-fs_initcall | `SecretMem::init` |
| `can_set_direct_map()` | per-arch probe | shared |
| `set_direct_map_invalid_noflush()` | per-page direct-map removal | shared |
| `set_direct_map_default_noflush()` | per-page direct-map restore | shared |
| `flush_tlb_kernel_range()` | per-page kernel TLB flush | shared |
| `mlock_future_ok()` | per-RLIMIT_MEMLOCK pre-check | shared |
| `mapping_set_unevictable()` | per-mapping unevictable | shared |
| `secretmem_vm_ops` (struct) | per-VMA ops vector | `SecretMem::VM_OPS` |
| `secretmem_aops` (struct) | per-address_space ops | `SecretMem::A_OPS` |
| `secretmem_fops` (struct) | per-file ops | `SecretMem::F_OPS` |
| `secretmem_iops` (struct) | per-inode ops | `SecretMem::I_OPS` |
| `secretmem_fs` (struct) | per-pseudo-FS type | `SecretMem::FS_TYPE` |

### compatibility contract

REQ-1: SYSCALL_DEFINE1(memfd_secret, flags):
- BUILD_BUG_ON: `SECRETMEM_FLAGS_MASK & O_CLOEXEC` must be 0 (no overlap).
- if `!secretmem_enable ∨ !can_set_direct_map()`: return `-ENOSYS`.
- if `flags & ~(SECRETMEM_FLAGS_MASK | O_CLOEXEC)`: return `-EINVAL`.
- if `atomic_read(&secretmem_users) < 0`: return `-ENFILE`.
- return `FD_ADD(flags & O_CLOEXEC, secretmem_file_create(flags))`.

REQ-2: `SECRETMEM_FLAGS_MASK == SECRETMEM_MODE_MASK == 0x0`:
- Only valid extra bit on syscall is `O_CLOEXEC`.

REQ-3: secretmem_file_create(flags):
- anon_inode = `anon_inode_make_secure_inode(secretmem_mnt->mnt_sb, "[secretmem]", NULL)`.
- file = `alloc_file_pseudo(inode, secretmem_mnt, "secretmem", O_RDWR | O_LARGEFILE, &secretmem_fops)`.
- `mapping_set_gfp_mask(inode->i_mapping, GFP_HIGHUSER)`.
- `mapping_set_unevictable(inode->i_mapping)`.
- `inode->i_op = &secretmem_iops`.
- `inode->i_mapping->a_ops = &secretmem_aops`.
- `inode->i_mode |= S_IFREG`.
- `inode->i_size = 0`.
- `atomic_inc(&secretmem_users)`.
- return file (or ERR_PTR on failure path with `iput(inode)`).

REQ-4: secretmem_mmap_prepare(desc):
- len = `vma_desc_size(desc)`.
- if `!vma_desc_test_any(desc, VMA_SHARED_BIT, VMA_MAYSHARE_BIT)`: return `-EINVAL` (must be MAP_SHARED).
- `vma_desc_set_flags(desc, VMA_LOCKED_BIT, VMA_DONTDUMP_BIT)`.
- if `!mlock_future_ok(desc->mm, /*is_vma_locked=*/true, len)`: return `-EAGAIN` (RLIMIT_MEMLOCK).
- `desc->vm_ops = &secretmem_vm_ops`.
- return 0.

REQ-5: secretmem_fault(vmf):
- mapping = `vmf->vma->vm_file->f_mapping`.
- inode = `file_inode(vmf->vma->vm_file)`.
- offset = `vmf->pgoff`; gfp = `vmf->gfp_mask`.
- if `((loff_t)offset << PAGE_SHIFT) >= i_size_read(inode)`: return `vmf_error(-EINVAL)` (past EOF).
- `filemap_invalidate_lock_shared(mapping)`.
- retry:
  - folio = `filemap_lock_folio(mapping, offset)`.
  - if `IS_ERR(folio)`:
    - folio = `folio_alloc(gfp | __GFP_ZERO, /*order=*/0)`.
    - if !folio: ret = `VM_FAULT_OOM`; goto out.
    - err = `set_direct_map_invalid_noflush(folio_page(folio, 0))`.
    - if err: `folio_put(folio)`; ret = `vmf_error(err)`; goto out.
    - `__folio_mark_uptodate(folio)`.
    - err = `filemap_add_folio(mapping, folio, offset, gfp)`.
    - if err:
      - `set_direct_map_default_noflush(folio_page(folio, 0))` (large-page split already happened during invalidation, guarantees this call cannot fail).
      - `folio_put(folio)`.
      - if err == `-EEXIST`: goto retry.
      - ret = `vmf_error(err)`; goto out.
    - addr = `(unsigned long)folio_address(folio)`.
    - `flush_tlb_kernel_range(addr, addr + PAGE_SIZE)`.
- `vmf->page = folio_file_page(folio, offset)`.
- ret = `VM_FAULT_LOCKED`.
- out: `filemap_invalidate_unlock_shared(mapping)`; return ret.

REQ-6: secretmem_vm_ops:
- `.fault = secretmem_fault`.
- (no `.huge_fault`, `.page_mkwrite`, `.access`, etc. — explicit denial of THP / direct-access.)

REQ-7: vma_is_secretmem(vma):
- return `vma->vm_ops == &secretmem_vm_ops`.

REQ-8: secretmem_release(inode, file):
- `atomic_dec(&secretmem_users)`.
- return 0.

REQ-9: secretmem_aops:
- `.dirty_folio = noop_dirty_folio` (no writeback path).
- `.free_folio = secretmem_free_folio`.
- `.migrate_folio = secretmem_migrate_folio` (returns `-EBUSY`).

REQ-10: secretmem_free_folio(folio):
- `set_direct_map_default_noflush(folio_page(folio, 0))` — restore direct map.
- `folio_zero_segment(folio, 0, folio_size(folio))` — scrub before return to allocator.

REQ-11: secretmem_migrate_folio(mapping, dst, src, mode):
- return `-EBUSY` — secretmem folios are never migratable (would re-expose to direct map).

REQ-12: secretmem_setattr(idmap, dentry, iattr):
- inode = `d_inode(dentry)`; mapping = `inode->i_mapping`; ia_valid = `iattr->ia_valid`.
- `filemap_invalidate_lock(mapping)`.
- if `(ia_valid & ATTR_SIZE) ∧ inode->i_size != 0`: ret = `-EINVAL` (cannot resize once non-zero — emulates F_SEAL_SHRINK / F_SEAL_GROW semantics; size grows implicitly via faults but explicit `ftruncate` only allowed from 0).
- else: ret = `simple_setattr(idmap, dentry, iattr)`.
- `filemap_invalidate_unlock(mapping)`.
- return ret.

REQ-13: secretmem_iops: `.setattr = secretmem_setattr`.

REQ-14: secretmem_fops:
- `.release = secretmem_release`.
- `.mmap_prepare = secretmem_mmap_prepare`.
- (No `.read_iter`, no `.write_iter`, no `.fallocate`, no `.llseek`: read/write/seek on the fd return `-EINVAL` via VFS default for non-supplied ops; only `mmap` produces useful access.)

REQ-15: secretmem_fs (file_system_type):
- `.name = "secretmem"`.
- `.init_fs_context = secretmem_init_fs_context`.
- `.kill_sb = kill_anon_super`.

REQ-16: secretmem_init_fs_context(fc):
- ctx = `init_pseudo(fc, SECRETMEM_MAGIC)`; if !ctx: return `-ENOMEM`.
- `fc->s_iflags |= SB_I_NOEXEC` — secretmem pages never executable.
- `fc->s_iflags |= SB_I_NODEV` — no devices.
- return 0.

REQ-17: secretmem_init (fs_initcall):
- if `!secretmem_enable ∨ !can_set_direct_map()`: return 0 (silent disable).
- `secretmem_mnt = kern_mount(&secretmem_fs)`.
- if `IS_ERR(secretmem_mnt)`: return `PTR_ERR(secretmem_mnt)`.
- return 0.

REQ-18: secretmem_active / secretmem_users counter:
- `atomic_t secretmem_users` increments on file_create, decrements on release.
- `secretmem_active()` returns `!!atomic_read(&secretmem_users)`; consulted by hibernation and similar paths to know whether unmapped-direct-map pages exist.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `direct_map_invalidated_before_user_visible` | INVARIANT | per-fault: `set_direct_map_invalid_noflush` returns Ok before folio is inserted into mapping. |
| `direct_map_restored_on_error_path` | INVARIANT | per-fault: on `filemap_add_folio` failure, `set_direct_map_default_noflush` called before `folio_put`. |
| `direct_map_restored_on_free` | INVARIANT | per-free_folio: `set_direct_map_default_noflush` called and folio zeroed. |
| `tlb_flushed_after_invalidation` | INVARIANT | per-fault: `flush_tlb_kernel_range` called after successful `filemap_add_folio`. |
| `enosys_when_arch_unsupported` | INVARIANT | per-sys_memfd_secret: `!can_set_direct_map()` ⟹ ENOSYS. |
| `flags_mask_disjoint_from_o_cloexec` | INVARIANT | const-assert: `SECRETMEM_FLAGS_MASK & O_CLOEXEC == 0`. |
| `mmap_requires_shared` | INVARIANT | per-mmap_prepare: `!(SHARED|MAYSHARE)` ⟹ EINVAL. |
| `vma_locked_dontdump_set` | INVARIANT | per-mmap_prepare success: VMA_LOCKED and VMA_DONTDUMP flags set on desc. |
| `migration_always_busy` | INVARIANT | per-migrate_folio: return EBUSY for all modes. |
| `users_balanced` | INVARIANT | per-file_create / release: USERS counter delta == 0 across pair. |
| `resize_from_nonzero_forbidden` | INVARIANT | per-setattr: `inode.i_size != 0 ∧ ATTR_SIZE` ⟹ EINVAL. |

### Layer 2: TLA+

`mm/secretmem.tla`:
- Per-fd-create + per-mmap + per-fault + per-free + per-release lifecycle.
- Properties:
  - `safety_no_direct_map_aliased_user_page` — per-folio: while in mapping, no kernel direct-map entry exists.
  - `safety_zeroed_on_return_to_allocator` — per-free_folio: folio zeroed before buddy receives it.
  - `safety_migration_blocked` — per-folio: never relocated.
  - `safety_locked_dontdump_invariant` — per-VMA: VM_LOCKED ∧ VM_DONTDUMP throughout lifetime.
  - `safety_rlimit_memlock_enforced` — per-mmap: total locked ≤ RLIMIT_MEMLOCK.
  - `liveness_per_fault_terminates` — per-fault: either success, `VM_FAULT_OOM`, or `vmf_error`.
  - `safety_users_counter_nonnegative` — per-system: `secretmem_users >= 0`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_memfd_secret` post: fd ≥ 0 ∨ Err ∈ {ENOSYS, EINVAL, ENFILE, EMFILE} | `SecretMem::sys_memfd_secret` |
| `file_create` post: USERS incremented; aops/iops/fops wired | `SecretMem::file_create` |
| `mmap_prepare` post: VM_LOCKED ∧ VM_DONTDUMP set; vm_ops = VM_OPS | `SecretMem::mmap_prepare` |
| `fault` post: page direct-map invalid; folio in mapping; TLB flushed | `SecretMem::fault` |
| `free_folio` post: direct-map restored; folio zeroed | `SecretMem::free_folio` |
| `migrate_folio` post: returns EBUSY | `SecretMem::migrate_folio` |
| `setattr` post: resize-from-nonzero rejected | `SecretMem::setattr` |
| `release` post: USERS decremented exactly once | `SecretMem::release` |

### Layer 4: Verus/Creusot functional

`Per-fault (allocate folio → set_direct_map_invalid_noflush → filemap_add_folio → flush_tlb_kernel_range → return VM_FAULT_LOCKED)` semantic equivalence: per-Documentation/userspace-api/mfd_noexec.rst and Documentation/admin-guide/mm/secretmem.rst. `Per-free (set_direct_map_default_noflush → folio_zero_segment)` symmetric inverse.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Secretmem reinforcement:

- **Per-direct-map unmap on fault** — defense against per-direct-map disclosure / Spectre-class kernel speculative reads of user secrets.
- **Per-flush_tlb_kernel_range after invalidation** — defense against per-stale-TLB read of the now-secret page.
- **Per-folio_zero_segment on free** — defense against per-secret-leak via buddy reallocation to another process.
- **Per-set_direct_map_default_noflush on free** — defense against per-permanently-orphaned direct-map hole.
- **Per-MAP_SHARED required** — defense against per-MAP_PRIVATE COW double-mapping that could leave a stale CoW source visible.
- **Per-VM_LOCKED mandatory** — defense against per-swap-out leak (swap space could leak plaintext).
- **Per-VM_DONTDUMP mandatory** — defense against per-coredump exfiltration.
- **Per-mapping_set_unevictable** — defense against per-page-reclaim writing secret to swap.
- **Per-migrate_folio = EBUSY** — defense against per-compaction relocating a page that would briefly become direct-map-visible.
- **Per-SB_I_NOEXEC / SB_I_NODEV on pseudo-FS** — defense against per-exec-of-secretmem and per-device-tricks.
- **Per-`can_set_direct_map()` arch probe** — defense against per-arch silently accepting secretmem on systems that cannot enforce direct-map removal (returns ENOSYS instead of providing a false guarantee).
- **Per-RLIMIT_MEMLOCK accounting via `mlock_future_ok`** — defense against per-unbounded-pin DoS.
- **Per-resize-from-nonzero forbidden** — defense against per-truncate-then-recover attack window where freed-then-reallocated pages could leak.
- **Per-no `.read_iter`/`.write_iter`** — defense against per-fd-direct-read bypass of mmap-only access discipline.
- **Per-`secretmem.enable` boot/module param ro_after_init** — defense against per-post-boot toggle attack surface change.

