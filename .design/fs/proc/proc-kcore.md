# Tier-3: fs/proc/proc-kcore — /proc/kcore (kernel core ELF for live-debug + drgn)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/proc/kcore.c
  - fs/proc/vmcore.c
  - include/linux/kcore.h
  - include/uapi/linux/elf.h
-->

## Summary
Tier-3 design for `/proc/kcore` — a virtual ELF-format file representing live kernel memory. Used by:
- `gdb /usr/lib/debug/lib/modules/.../vmlinux /proc/kcore` for live-kernel debugging
- `drgn` (Programmable kernel debugger) for live-kernel state inspection
- `crash` utility (live-kernel mode)
- `eu-readelf -a /proc/kcore` for ELF inspection of live kernel memory

The file presents itself as a `ET_CORE` ELF executable with program headers describing memory regions:
- Direct-mapped low memory (`__START_KERNEL_map`)
- vmalloc + vmap regions
- Per-CPU areas
- Per-NUMA-node memory
- Modules' code + data
- Kernel symbols' addresses

Plus, on architectures supporting it, additional notes for CPU register state captured at the `/proc/kcore` open time (mostly relevant for /proc/vmcore from a kdump but kcore inherits the format).

`/proc/vmcore` is the related file for crash-dump (kdump) scenarios — covered briefly here since they share `vmcore.c` and kcore-format-emit code.

Sub-tier-3 of `fs/proc/00-overview.md`. Read access requires CAP_SYS_RAWIO + CAP_SYS_PTRACE per `kptr_restrict` policy.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| /proc/kcore: ELF file emit, page-table walk, per-region program-header generation | `fs/proc/kcore.c` |
| /proc/vmcore: kdump-side ELF emit (similar format) | `fs/proc/vmcore.c` |
| Public API | `include/linux/kcore.h` |
| ELF UAPI | `include/uapi/linux/elf.h` |

## Compatibility contract

### `/proc/kcore` ELF format

Structure:
```
ELF Header (Elf64_Ehdr): e_type = ET_CORE, e_machine = EM_X86_64 (or per-arch)
Program Headers (Elf64_Phdr): one per virtual-memory region
  - PT_LOAD (kernel virtual segments — usually 4-6 of these)
  - PT_NOTE (CPU register state, vmcoreinfo, etc.)
Section Headers: usually absent (ELF core-file convention)
```

Per program header:
- `p_type` = `PT_LOAD` for memory regions, `PT_NOTE` for note section
- `p_offset` = file offset into kcore where this region's bytes start
- `p_vaddr` = kernel virtual address of region
- `p_paddr` = same as p_vaddr (or physical for vmcore)
- `p_filesz` = `p_memsz` = region size in bytes
- `p_flags` = `PF_R | PF_W | PF_X` (kernel pages are typically RWX from kernel's perspective)
- `p_align` = `PAGE_SIZE`

Identical ELF layout so `gdb` / `drgn` / `crash` parse correctly.

### Memory regions covered

Per-arch differs slightly. On x86_64:
1. Direct-mapped low memory: `__START_KERNEL_map` (typically `0xffffffff80000000` for 64-bit) up to end of kernel image
2. Direct-mapped physical memory: `PAGE_OFFSET` (typically `0xffff888000000000`) up to highest mapped RAM
3. vmalloc area: `VMALLOC_START` to `VMALLOC_END`
4. vmemmap: `VMEMMAP_START` to `VMEMMAP_END` (if enabled)
5. fixmap area
6. Per-CPU areas
7. Module addresses (`MODULES_VADDR` to `MODULES_END`)

Identical region-set per-arch.

### Read semantics

Reading from `/proc/kcore`:
1. Per-page lookup of the corresponding kernel virtual address
2. If page is mapped → read directly
3. If page is unmapped (e.g., guard page, hot-unplugged memory) → read returns zeros
4. Per-region byte offsets calculated via program-header `p_offset`/`p_vaddr` mapping

Identical algorithm.

### Lazy header generation

ELF headers (`Ehdr` + `Phdr` array) are generated on first read; cached in per-`kcore_list` array. On hot-{add,remove} of memory or modules, header is invalidated + regenerated next read.

### `kclist_add` / `kclist_del` (per-subsys region registration)

```c
void kclist_add(struct kcore_list *new, void *addr, size_t size, int type);
void kclist_del(struct kcore_list *m);
```

Per-subsystem (vmalloc, modules, etc.) registers its kernel-virtual region(s) at boot or hot-add time. `type` ∈ `KCORE_TEXT / KCORE_VMALLOC / KCORE_RAM / KCORE_VMEMMAP / KCORE_USER / KCORE_REMAP`.

Identical contract.

### `/proc/vmcore` (crash-dump format)

Similar format but for post-crash kdump scenarios. Userspace `makedumpfile` consumes /proc/vmcore to produce compressed crash dumps. Format-wise nearly identical to /proc/kcore but with physical addresses.

### Permission gate

Read access requires CAP_SYS_RAWIO + (CAP_SYS_PTRACE OR same-pid as init):
- `kptr_restrict=2` blocks even root (default for security-strict deployments)
- `kptr_restrict=1` allows root only
- `kptr_restrict=0` allows any user (insecure; never default)

Plus `lockdown` mode: when kernel is locked-down (`lockdown=integrity|confidentiality`), `/proc/kcore` is unreadable to prevent kernel-pointer leak even by root.

### vmcoreinfo note

`PT_NOTE` segment contains a `VMCOREINFO` note with kernel version, struct offsets, per-arch context. Used by `makedumpfile` for compressed crash dumps. Format byte-identical.

## Requirements

- REQ-1: ELF file structure: ET_CORE Ehdr + array of Phdrs (PT_LOAD per region + PT_NOTE for CPU/vmcoreinfo); per-arch Ehdr.
- REQ-2: Per-region program header: `p_type / p_offset / p_vaddr / p_paddr / p_filesz / p_memsz / p_flags / p_align` byte-identical to upstream.
- REQ-3: Memory regions covered (per-arch — direct-mapped low + direct-mapped physical + vmalloc + vmemmap + fixmap + per-cpu + modules) per-arch identical.
- REQ-4: Read semantics: per-page lookup; unmapped pages → zero-fill; per-region offset arithmetic correct.
- REQ-5: Lazy header generation: cached after first read; invalidated on memory/module hot-{add,remove}.
- REQ-6: `kclist_add` / `kclist_del` per-subsystem region registration; identical signatures.
- REQ-7: `/proc/vmcore` post-crash format compatibility — physical-address-based; identical Phdr layout.
- REQ-8: Permission gate: CAP_SYS_RAWIO + CAP_SYS_PTRACE; `kptr_restrict` honored; `lockdown` mode blocks read.
- REQ-9: VMCOREINFO note: per-arch struct offsets + kernel-version + build-id; format byte-identical.
- REQ-10: Hot-add/remove memory: invalidate header + regenerate; concurrent reader sees consistent (post-event) view.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `eu-readelf -h /proc/kcore` succeeds; e_type=CORE, e_machine matches host arch, valid Phdrs. (covers REQ-1, REQ-2)
- [ ] AC-2: gdb test: `gdb /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /proc/kcore` opens; can read kernel variables (e.g., `print init_task`). (covers REQ-1, REQ-2, REQ-4)
- [ ] AC-3: drgn test: `drgn` opens /proc/kcore; live-kernel object inspection works. (covers REQ-4)
- [ ] AC-4: per-region test: `eu-readelf -l /proc/kcore` shows direct-mapped + vmalloc + per-cpu + modules regions on x86_64. (covers REQ-3)
- [ ] AC-5: `kclist_add` test: load synthetic kernel module that registers a region; subsequent /proc/kcore read includes the new region's Phdr. (covers REQ-5, REQ-6)
- [ ] AC-6: hot-add test: hot-add memory; subsequent kcore read includes the added region. (covers REQ-10)
- [ ] AC-7: kptr_restrict test: with `kptr_restrict=2`, root read of /proc/kcore returns -EPERM. (covers REQ-8)
- [ ] AC-8: lockdown test: with `lockdown=integrity`, /proc/kcore returns -EPERM regardless of caps. (covers REQ-8)
- [ ] AC-9: VMCOREINFO test: extract PT_NOTE segment via `eu-readelf -n /proc/kcore`; vmcoreinfo present with kernel-version + offset-of-struct fields. (covers REQ-9)
- [ ] AC-10: vmcore makedumpfile test: kdump kernel running; `makedumpfile -d 31 /proc/vmcore vmcore.compressed` produces valid compressed dump. (covers REQ-7)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::fs::proc::kcore::Kcore` — top-level entry
- `kernel::fs::proc::kcore::ElfHeader` — Ehdr + Phdr generation
- `kernel::fs::proc::kcore::Region` — `struct kcore_list` + per-region descriptor
- `kernel::fs::proc::kcore::Read` — per-skb page-by-page read
- `kernel::fs::proc::kcore::HotAdd` — hot-add/remove memory invalidation
- `kernel::fs::proc::kcore::Permission` — CAP_SYS_RAWIO + lockdown gate
- `kernel::fs::proc::kcore::Vmcoreinfo` — PT_NOTE generation
- `kernel::fs::proc::vmcore::Vmcore` — post-crash kdump variant

### Locking and concurrency

- **`kclist_lock`** (rwlock): per-system kcore_list mutator
- **Per-region refcount**: hot-removed memory refcounted to prevent reader fault during teardown
- **RCU**: kcore_list traversal hot path RCU-side

### Error handling

- `Err(EPERM)` — non-CAP_SYS_RAWIO / kptr_restrict / lockdown
- `Err(EFAULT)` — read into invalid userspace buffer
- `Err(EINVAL)` — bad offset / size
- Unmapped pages return zero-fill (no error to caller)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| ELF Ehdr + Phdr build (no out-of-bounds; correct alignment) | `kani::proofs::fs::proc::kcore::elf_safety` |
| Per-page lookup (no use-after-free during read) | `kani::proofs::fs::proc::kcore::page_safety` |
| Hot-add/remove invalidation (header re-generation atomicity) | `kani::proofs::fs::proc::kcore::hotadd_safety` |
| `kclist_add`/`_del` under kclist_lock | `kani::proofs::fs::proc::kcore::kclist_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system kcore_list | regions non-overlapping in virtual address space | `kani::proofs::fs::proc::kcore::region_invariants` |
| Per-region Phdr | `p_filesz == p_memsz`; `p_offset` non-overlapping with prior Phdrs | `kani::proofs::fs::proc::kcore::phdr_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `fs/proc/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-arch Phdr-template + kcore proc_ops `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed read-buffer cleared (carries kernel memory bytes) | § Default-on configurable off |
| **KPTR_RESTRICT** | `/proc/kcore` read blocked at kptr_restrict ≥ 1 (default 1; or 2 for stricter) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above
- **USERCOPY**: per-region read uses `copy_to_user` with explicit page-bound check
- **SIZE_OVERFLOW**: per-region offset arithmetic uses checked operators
- **KERNEXEC**: per-region dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_SYS_RAWIO)` already required.
- LSM hook `security_locked_down(LOCKDOWN_KCORE)` for lockdown-mode block.
- Default useful GR-RBAC policy: deny `/proc/kcore` reads outside gradm-marked `kernel_debugger` role; arbitrary kernel-memory introspection is the highest-privilege debug surface.

### Userspace-visible behavior changes

None beyond upstream defaults (gdb / drgn / crash continue to work for authorized debugger users).

### Verification

(See § Verification above.)

## Open Questions

(none — kcore ELF format exhaustively specified by upstream + gdb/drgn/crash test corpus)

## Out of Scope

- gdb / drgn / crash userspace tooling (out of scope)
- 32-bit-only paths
- Implementation code
