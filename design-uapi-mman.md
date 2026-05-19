---
title: "Tier-5 UAPI: include/uapi/linux/mman.h + include/uapi/asm-generic/mman{,-common}.h — Memory-management ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The mman UAPI is the syscall-edge contract for `mmap(2)`, `mmap2(2)`, `mremap(2)`, `mprotect(2)`, `pkey_mprotect(2)`, `msync(2)`, `madvise(2)`, `process_madvise(2)`, `mlock(2)`, `mlock2(2)`, `munlock(2)`, `mlockall(2)`, `munlockall(2)`, and `cachestat(2)`. It defines:

- **Protection bits** (`PROT_READ`, `PROT_WRITE`, `PROT_EXEC`, `PROT_SEM`, `PROT_NONE`, `PROT_GROWSDOWN`, `PROT_GROWSUP`).
- **Mapping-type and flag bits** (`MAP_SHARED`, `MAP_PRIVATE`, `MAP_SHARED_VALIDATE`, `MAP_TYPE`, `MAP_FIXED`, `MAP_FIXED_NOREPLACE`, `MAP_ANONYMOUS`, `MAP_GROWSDOWN`, `MAP_DENYWRITE`, `MAP_EXECUTABLE`, `MAP_LOCKED`, `MAP_NORESERVE`, `MAP_POPULATE`, `MAP_NONBLOCK`, `MAP_STACK`, `MAP_HUGETLB`, `MAP_SYNC`, `MAP_UNINITIALIZED`, `MAP_DROPPABLE`, `MAP_FILE` compatibility flag).
- **Huge-page encoding** (`MAP_HUGE_SHIFT`, `MAP_HUGE_MASK`, `MAP_HUGE_16KB` … `MAP_HUGE_16GB`).
- **`mremap(2)` flags** (`MREMAP_MAYMOVE`, `MREMAP_FIXED`, `MREMAP_DONTUNMAP`).
- **`msync(2)` flags** (`MS_ASYNC`, `MS_SYNC`, `MS_INVALIDATE`).
- **`mlockall(2)` flags** (`MCL_CURRENT`, `MCL_FUTURE`, `MCL_ONFAULT`) plus `mlock2(2)` `MLOCK_ONFAULT`.
- **`madvise(2)` advice values** (`MADV_NORMAL`, `MADV_RANDOM`, `MADV_SEQUENTIAL`, `MADV_WILLNEED`, `MADV_DONTNEED`, `MADV_FREE`, `MADV_REMOVE`, `MADV_DONTFORK`, `MADV_DOFORK`, `MADV_MERGEABLE`, `MADV_UNMERGEABLE`, `MADV_HUGEPAGE`, `MADV_NOHUGEPAGE`, `MADV_DONTDUMP`, `MADV_DODUMP`, `MADV_WIPEONFORK`, `MADV_KEEPONFORK`, `MADV_COLD`, `MADV_PAGEOUT`, `MADV_POPULATE_READ`, `MADV_POPULATE_WRITE`, `MADV_DONTNEED_LOCKED`, `MADV_COLLAPSE`, `MADV_GUARD_INSTALL`, `MADV_GUARD_REMOVE`, `MADV_HWPOISON`, `MADV_SOFT_OFFLINE`).
- **Overcommit policies** (`OVERCOMMIT_GUESS`, `OVERCOMMIT_ALWAYS`, `OVERCOMMIT_NEVER`).
- **Protection keys** (`PKEY_UNRESTRICTED`, `PKEY_DISABLE_ACCESS`, `PKEY_DISABLE_WRITE`, `PKEY_ACCESS_MASK`).
- **Shadow-stack flags** (`SHADOW_STACK_SET_TOKEN`, `SHADOW_STACK_SET_MARKER`).
- **`cachestat(2)` structures** (`struct cachestat_range`, `struct cachestat`).

Critical for: every userspace allocator (glibc/jemalloc/mimalloc/tcmalloc), every JIT compiler (V8, JVM, .NET, LuaJIT, sqlite3, Wine), every database that mmaps its data file (PostgreSQL, SQLite, RocksDB), every container runtime that has to set up shared memory, every sandboxing framework that uses `mprotect` to lock down code.

This Tier-5 covers `include/uapi/linux/mman.h` and consolidates the directly-related arch-generic headers `include/uapi/asm-generic/mman.h` and `mman-common.h` which carry the bulk of the ABI bits.

### Acceptance Criteria

- [ ] AC-1: `PROT_READ|PROT_WRITE == 3`; `PROT_NONE == 0`; `PROT_GROWSDOWN == 0x01000000`.
- [ ] AC-2: `mprotect(addr, len, PROT_READ | 0x10000)` returns `-EINVAL`.
- [ ] AC-3: `mmap(NULL, 4096, PROT_READ, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` succeeds; first byte reads zero.
- [ ] AC-4: `mmap(addr, 4096, PROT_READ, MAP_FIXED_NOREPLACE|MAP_ANONYMOUS|MAP_PRIVATE, -1, 0)` over an existing mapping returns `-EEXIST`.
- [ ] AC-5: `mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, fd, 0)` with `MAP_SYNC` and non-DAX `fd` returns `-EOPNOTSUPP`.
- [ ] AC-6: `mmap` with `MAP_HUGETLB | MAP_HUGE_2MB` on x86-64 succeeds when sysfs `nr_hugepages` is configured.
- [ ] AC-7: `mremap(old, oldsz, newsz, MREMAP_FIXED, newaddr)` without `MREMAP_MAYMOVE` returns `-EINVAL`.
- [ ] AC-8: `mremap(... MREMAP_DONTUNMAP ...)` on a private-anon mapping leaves the source range mapped as PROT_NONE placeholder.
- [ ] AC-9: `msync(..., MS_ASYNC | MS_SYNC)` returns `-EINVAL`.
- [ ] AC-10: `mlockall(MCL_ONFAULT)` returns `-EINVAL`; `mlockall(MCL_CURRENT|MCL_ONFAULT)` succeeds.
- [ ] AC-11: `mlock2(addr, len, MLOCK_ONFAULT)` returns 0 and the pages are not prefaulted.
- [ ] AC-12: `madvise(addr, len, MADV_HWPOISON)` without CAP_SYS_ADMIN returns `-EPERM`.
- [ ] AC-13: `madvise(addr, len, 999)` returns `-EINVAL`.
- [ ] AC-14: `madvise(addr, len, MADV_GUARD_INSTALL)` then touching `addr` delivers `SIGSEGV` with `SEGV_ACCERR`.
- [ ] AC-15: `madvise(addr, len, MADV_WIPEONFORK)` on private-anon: after fork, child reads zeros.
- [ ] AC-16: `madvise(addr, len, MADV_COLLAPSE)` returns 0 only after THP attempt completes (synchronous).
- [ ] AC-17: `MAP_DROPPABLE | MAP_PRIVATE | MAP_ANONYMOUS` succeeds; under memory pressure pages return zero on next read.
- [ ] AC-18: `sizeof(struct cachestat) == 40`; `sizeof(struct cachestat_range) == 16`.
- [ ] AC-19: `pkey_mprotect` with `pkey=-1` keeps current pkey; with valid pkey reassigns.

### Architecture

```
bitflags::bitflags! {
    pub struct Prot : u32 {
        const READ       = 0x1;
        const WRITE      = 0x2;
        const EXEC       = 0x4;
        const SEM        = 0x8;
        const NONE       = 0x0;
        const GROWSDOWN  = 0x0100_0000;
        const GROWSUP    = 0x0200_0000;
    }
}

bitflags::bitflags! {
    pub struct MapFlags : u32 {
        const SHARED            = 0x0001;
        const PRIVATE           = 0x0002;
        const SHARED_VALIDATE   = 0x0003;
        const DROPPABLE         = 0x0008;
        const FIXED             = 0x0010;
        const ANONYMOUS         = 0x0020;
        const GROWSDOWN         = 0x0100;
        const DENYWRITE         = 0x0800;
        const EXECUTABLE        = 0x1000;
        const LOCKED            = 0x2000;
        const NORESERVE         = 0x4000;
        const POPULATE          = 0x0000_8000;
        const NONBLOCK          = 0x0001_0000;
        const STACK             = 0x0002_0000;
        const HUGETLB           = 0x0004_0000;
        const SYNC              = 0x0008_0000;
        const FIXED_NOREPLACE   = 0x0010_0000;
        const UNINITIALIZED     = 0x0400_0000;
    }
}

pub const MAP_TYPE_MASK: u32 = 0x0f;
pub const MAP_HUGE_SHIFT: u32 = 26;
pub const MAP_HUGE_MASK: u32 = 0x3f;

#[repr(C)]
pub struct CachestatRange {
    pub off: u64,
    pub len: u64,
}

#[repr(C)]
pub struct Cachestat {
    pub nr_cache: u64,
    pub nr_dirty: u64,
    pub nr_writeback: u64,
    pub nr_evicted: u64,
    pub nr_recently_evicted: u64,
}
```

`SysMmap::sys_mmap(addr, len, prot, flags, fd, off) -> Result<usize>`:
1. /* Validate prot */
2. let known = Prot::READ | Prot::WRITE | Prot::EXEC | Prot::SEM | Prot::GROWSDOWN | Prot::GROWSUP.
3. if prot & !(known | arch_reserved) != 0: return Err(EINVAL).
4. if prot & (Prot::GROWSDOWN | Prot::GROWSUP) on mmap path: return Err(EINVAL) /* mprotect-only */.
5. /* Validate mapping type */
6. let mtype = flags & MAP_TYPE_MASK.
7. match mtype { 1 => SHARED, 2 => PRIVATE, 3 => SHARED_VALIDATE, _ => return Err(EINVAL) }.
8. /* If SHARED_VALIDATE: strictly reject unknown flag bits */
9. if mtype == 3 && (flags & !known_flag_bits) != 0: return Err(EOPNOTSUPP).
10. /* Validate MAP_FIXED_NOREPLACE vs existing */
11. if flags & MAP_FIXED_NOREPLACE && range_overlaps_existing(addr, len): return Err(EEXIST).
12. /* MAP_SYNC requires SHARED_VALIDATE + DAX */
13. if flags & MAP_SYNC && (mtype != SHARED_VALIDATE || !file_is_dax(fd)): return Err(EOPNOTSUPP).
14. /* Hugepage size decode */
15. if flags & MAP_HUGETLB {
        let log = (flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK;
        let sz = if log == 0 { default_hugepage_size() } else { 1usize << log };
        ensure!(arch_supports_hugepage(sz), Err(EINVAL));
    }
16. /* Per LSM */
17. security_mmap_file(fd, prot, flags)?.
18. /* Allocate VMA */
19. let vma = Vma::new(addr, len, prot, flags, fd, off)?.
20. /* MAP_LOCKED ⟹ try mlock */
21. if flags & MAP_LOCKED { mlock_vma_pages_range(&vma, ...)?; }
22. /* MAP_POPULATE ⟹ prefault */
23. if flags & MAP_POPULATE { populate_vma_page_range(&vma, ...)?; }
24. return Ok(vma.start).

`SysMmap::sys_mremap(old_addr, old_len, new_len, flags, new_addr) -> Result<usize>`:
1. if flags & !(MREMAP_MAYMOVE | MREMAP_FIXED | MREMAP_DONTUNMAP) != 0: return Err(EINVAL).
2. if (flags & MREMAP_FIXED) && !(flags & MREMAP_MAYMOVE): return Err(EINVAL).
3. if (flags & MREMAP_DONTUNMAP) {
       require flags & MREMAP_MAYMOVE;
       require source_vma.is_private_anon();
   }
4. /* Resize / relocate */
5. ...

`SysMmap::sys_madvise(addr, len, advice) -> Result<()>`:
1. match advice {
       MADV_NORMAL | MADV_RANDOM | MADV_SEQUENTIAL | MADV_WILLNEED | MADV_DONTNEED
       | MADV_FREE | MADV_REMOVE | MADV_DONTFORK | MADV_DOFORK
       | MADV_MERGEABLE | MADV_UNMERGEABLE | MADV_HUGEPAGE | MADV_NOHUGEPAGE
       | MADV_DONTDUMP | MADV_DODUMP | MADV_WIPEONFORK | MADV_KEEPONFORK
       | MADV_COLD | MADV_PAGEOUT | MADV_POPULATE_READ | MADV_POPULATE_WRITE
       | MADV_DONTNEED_LOCKED | MADV_COLLAPSE
       | MADV_GUARD_INSTALL | MADV_GUARD_REMOVE => proceed,
       MADV_HWPOISON | MADV_SOFT_OFFLINE => require_capability(CAP_SYS_ADMIN)?,
       _ => return Err(EINVAL),
   };
2. for vma in vmas_in_range(addr, len) { advice.apply(vma, addr, len)?; }
3. return Ok(()).

### Out of Scope

- `include/uapi/asm-generic/hugetlb_encode.h` (covered inline above)
- Per-arch `mman.h` overrides (`arch/<arch>/include/uapi/asm/mman.h`) — covered per-arch
- `mm/mmap.c` / `mm/mremap.c` / `mm/madvise.c` / `mm/mlock.c` implementations — covered in `mm/` Tier-3
- `mm/khugepaged.c` MADV_COLLAPSE worker (covered in `mm/khugepaged.md` Tier-3)
- `pkey_alloc(2)` / `pkey_free(2)` syscalls (covered in `uapi/syscalls/pkey.md` Tier-5)
- `cachestat(2)` implementation (covered in `mm/cachestat.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `PROT_*` constants | per-page protection | `mman::Prot` flagset |
| `MAP_*` constants | per-mmap-type/flag | `mman::MapFlags` flagset |
| `MAP_HUGE_*` constants | per-hugepage-size encoding | `mman::HugeSize` |
| `MS_*` constants | per-msync mode | `mman::MsFlags` |
| `MCL_*` constants | per-mlockall scope | `mman::MclFlags` |
| `MLOCK_ONFAULT` | per-mlock2 mode | `mman::MlockFlags` |
| `MADV_*` constants | per-madvise advice | `mman::MAdvice` enum |
| `MREMAP_*` constants | per-mremap flag | `mman::MremapFlags` |
| `OVERCOMMIT_*` constants | per-vm.overcommit_memory | shared |
| `PKEY_*` constants | per-pkey access bits | `mman::PkeyAccess` |
| `SHADOW_STACK_*` | per-shadow-stack token/marker | `mman::ShadowStackFlags` |
| `struct cachestat_range` | per-cachestat input range | `CachestatRange` |
| `struct cachestat` | per-cachestat counters | `Cachestat` |
| `HUGETLB_FLAG_ENCODE_SHIFT` | hugetlb encode shift (26) | shared |
| `HUGETLB_FLAG_ENCODE_MASK` | hugetlb encode mask (0x3f) | shared |

### abi surface (constants + structs)

### Protection bits (`asm-generic/mman-common.h:10-18`)

```text
PROT_READ       = 0x1            /* page can be read */
PROT_WRITE      = 0x2            /* page can be written */
PROT_EXEC       = 0x4            /* page can be executed */
PROT_SEM        = 0x8            /* page may be used for atomic ops */
PROT_NONE       = 0x0            /* no access */
PROT_GROWSDOWN  = 0x01000000     /* mprotect: extend to start of growsdown VMA */
PROT_GROWSUP    = 0x02000000     /* mprotect: extend to end of growsup VMA */
/* 0x10, 0x20 reserved arch-specific */
```

### Mapping type-and-flag bits (`uapi/linux/mman.h:17-20`, `asm-generic/mman-common.h:21-34`, `asm-generic/mman.h:7-11`)

```text
MAP_SHARED              = 0x0001  /* share changes */
MAP_PRIVATE             = 0x0002  /* COW private */
MAP_SHARED_VALIDATE     = 0x0003  /* share + validate extension flags */
MAP_TYPE                = 0x000f  /* mask: low 4 bits = mapping type */
MAP_DROPPABLE           = 0x0008  /* zero under memory pressure */
MAP_FIXED               = 0x0010  /* interpret addr exactly (may unmap) */
MAP_ANONYMOUS           = 0x0020  /* no backing file */
MAP_GROWSDOWN           = 0x0100  /* stack-like segment */
MAP_DENYWRITE           = 0x0800  /* historic ETXTBSY (now no-op) */
MAP_EXECUTABLE          = 0x1000  /* mark executable */
MAP_LOCKED              = 0x2000  /* lock pages (mmap-time mlock) */
MAP_NORESERVE           = 0x4000  /* don't reserve swap */
MAP_POPULATE            = 0x008000  /* prefault page tables */
MAP_NONBLOCK            = 0x010000  /* do not block on IO during MAP_POPULATE */
MAP_STACK               = 0x020000  /* stack-suitable address */
MAP_HUGETLB             = 0x040000  /* create huge-page mapping */
MAP_SYNC                = 0x080000  /* DAX synchronous faults */
MAP_FIXED_NOREPLACE     = 0x100000  /* MAP_FIXED, but EEXIST instead of unmap */
MAP_UNINITIALIZED       = 0x4000000 /* anon: skip zeroing (no-MMU only) */

MAP_FILE                = 0          /* historic compat alias */
```

`MAP_SHARED_VALIDATE` (3) is distinct from `MAP_SHARED|MAP_PRIVATE`; selecting it tells the kernel to **reject** unknown flag bits rather than silently ignore them. New mapping flags (e.g., `MAP_SYNC`) require `MAP_SHARED_VALIDATE`.

### Huge-page-size encoding (`uapi/linux/mman.h:29-44` + `asm-generic/hugetlb_encode.h`)

```text
MAP_HUGE_SHIFT  = 26      /* bit position of size in MAP_HUGETLB encoding */
MAP_HUGE_MASK   = 0x3f    /* 6-bit log2 of page size, shifted by MAP_HUGE_SHIFT */

MAP_HUGE_16KB   = (14 << 26)
MAP_HUGE_64KB   = (16 << 26)
MAP_HUGE_512KB  = (19 << 26)
MAP_HUGE_1MB    = (20 << 26)
MAP_HUGE_2MB    = (21 << 26)
MAP_HUGE_8MB    = (23 << 26)
MAP_HUGE_16MB   = (24 << 26)
MAP_HUGE_32MB   = (25 << 26)
MAP_HUGE_256MB  = (28 << 26)
MAP_HUGE_512MB  = (29 << 26)
MAP_HUGE_1GB    = (30 << 26)
MAP_HUGE_2GB    = (31 << 26)
MAP_HUGE_16GB   = (34 << 26)
```

The chosen huge-page size MUST be supported by the running architecture (e.g., x86-64 supports 2MB and 1GB; arm64 supports 2MB, 32MB, 512MB, 16GB depending on stage-1 granule).

### `mremap(2)` flags (`uapi/linux/mman.h:9-11`)

```text
MREMAP_MAYMOVE    = 1   /* may relocate mapping */
MREMAP_FIXED      = 2   /* new_address is authoritative; requires MAYMOVE */
MREMAP_DONTUNMAP  = 4   /* old mapping kept as PROT_NONE / lazy */
```

### `msync(2)` flags (`asm-generic/mman-common.h:41-43`)

```text
MS_ASYNC       = 1
MS_INVALIDATE  = 2
MS_SYNC        = 4
```

### `mlock` / `mlockall` (`asm-generic/mman.h:18-20`, `asm-generic/mman-common.h:39`)

```text
MCL_CURRENT    = 1   /* lock currently mapped pages */
MCL_FUTURE     = 2   /* lock future mappings */
MCL_ONFAULT    = 4   /* lock pages as they are faulted in */

MLOCK_ONFAULT  = 0x01  /* mlock2(2) only */
```

### `madvise(2)` advice values (`asm-generic/mman-common.h:45-83`)

```text
MADV_NORMAL         = 0
MADV_RANDOM         = 1
MADV_SEQUENTIAL     = 2
MADV_WILLNEED       = 3
MADV_DONTNEED       = 4
MADV_FREE           = 8
MADV_REMOVE         = 9
MADV_DONTFORK       = 10
MADV_DOFORK         = 11
MADV_MERGEABLE      = 12     /* KSM merge candidate */
MADV_UNMERGEABLE    = 13
MADV_HUGEPAGE       = 14     /* THP candidate */
MADV_NOHUGEPAGE     = 15
MADV_DONTDUMP       = 16     /* exclude from coredump */
MADV_DODUMP         = 17
MADV_WIPEONFORK     = 18     /* zero on fork, child only */
MADV_KEEPONFORK     = 19
MADV_COLD           = 20     /* deactivate */
MADV_PAGEOUT        = 21     /* reclaim */
MADV_POPULATE_READ  = 22     /* prefault readable */
MADV_POPULATE_WRITE = 23     /* prefault writable */
MADV_DONTNEED_LOCKED= 24     /* DONTNEED, drop locked pages too */
MADV_COLLAPSE       = 25     /* synchronous THP collapse */
MADV_HWPOISON       = 100    /* TEST: poison */
MADV_SOFT_OFFLINE   = 101    /* TEST: soft offline */
MADV_GUARD_INSTALL  = 102    /* fatal signal on touch */
MADV_GUARD_REMOVE   = 103    /* remove guard */
```

Note the gap at 5..7 (historical/obsolete); userspace MUST NOT pass values in those gaps.

### Overcommit policies (`uapi/linux/mman.h:13-15`)

```text
OVERCOMMIT_GUESS   = 0    /* heuristic (default) */
OVERCOMMIT_ALWAYS  = 1    /* never refuse */
OVERCOMMIT_NEVER   = 2    /* strict accounting */
```

Selected via `vm.overcommit_memory` sysctl.

### Protection keys (`asm-generic/mman-common.h:88-92`)

```text
PKEY_UNRESTRICTED    = 0x0
PKEY_DISABLE_ACCESS  = 0x1
PKEY_DISABLE_WRITE   = 0x2
PKEY_ACCESS_MASK     = PKEY_DISABLE_ACCESS | PKEY_DISABLE_WRITE  /* = 0x3 */
```

### Shadow stack (`asm-generic/mman.h:22-23`)

```text
SHADOW_STACK_SET_TOKEN  = (1ULL << 0)   /* shstk restore token */
SHADOW_STACK_SET_MARKER = (1ULL << 1)   /* top-of-stack marker */
```

### `cachestat(2)` structures (`uapi/linux/mman.h:46-57`)

```text
struct cachestat_range {
    __u64 off;
    __u64 len;
};

struct cachestat {
    __u64 nr_cache;
    __u64 nr_dirty;
    __u64 nr_writeback;
    __u64 nr_evicted;
    __u64 nr_recently_evicted;
};
```

Both fixed-size, no padding, naturally aligned on 8-byte boundaries on all arches.

### compatibility contract

REQ-1: `PROT_*` bit values MUST match: `PROT_READ=1, PROT_WRITE=2, PROT_EXEC=4, PROT_SEM=8, PROT_NONE=0`. `PROT_GROWSDOWN=0x01000000`, `PROT_GROWSUP=0x02000000` MUST be valid only on `mprotect(2)` (not `mmap(2)`).

REQ-2: `mmap(2)` MUST reject unknown `prot` bits (other than the documented set plus arch-reserved `0x10`/`0x20`) with `-EINVAL`. `PROT_GROWSDOWN | PROT_GROWSUP` simultaneously MUST return `-EINVAL`.

REQ-3: `MAP_TYPE == 0x0f` masks the mapping-type field of `flags`. Exactly one of `MAP_SHARED`, `MAP_PRIVATE`, `MAP_SHARED_VALIDATE` MUST be set; zero or two MUST return `-EINVAL`.

REQ-4: `MAP_SHARED_VALIDATE == 3 == MAP_SHARED|MAP_PRIVATE` is intentional: kernels predating `MAP_SHARED_VALIDATE` saw it as an invalid combination. Kernels supporting it MUST reject any unknown `flags` bit when this type is selected.

REQ-5: `MAP_FIXED` MUST silently unmap any existing mapping in the requested range. `MAP_FIXED_NOREPLACE` MUST return `-EEXIST` if any byte of the range is already mapped.

REQ-6: `MAP_ANONYMOUS` mappings MUST be zero-filled before the first read (unless `MAP_UNINITIALIZED` is set AND the kernel was built with `CONFIG_MMAP_ALLOW_UNINITIALIZED` — no-MMU only).

REQ-7: `MAP_HUGETLB` selects an unspecified default hugepage size unless OR'd with one of `MAP_HUGE_*`. The hugepage size = `1 << ((flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK)` bytes when `MAP_HUGE_*` is set.

REQ-8: `MAP_SYNC` MUST be used only with `MAP_SHARED_VALIDATE` and a DAX-backed file. Combinations like `MAP_PRIVATE | MAP_SYNC` MUST return `-EOPNOTSUPP`.

REQ-9: `MAP_GROWSDOWN` MUST only be valid with anonymous mappings; the kernel grows the VMA on a fault below the current start, bounded by `RLIMIT_STACK`.

REQ-10: `MAP_DENYWRITE` and `MAP_EXECUTABLE` MUST be silently ignored by `mmap(2)` (historic compatibility — kernel applies internally for ELF loader).

REQ-11: `MAP_LOCKED` MUST require `CAP_IPC_LOCK` if the locked-bytes total would exceed `RLIMIT_MEMLOCK`. Best-effort: `MAP_LOCKED` faults do not return `-ENOMEM` mid-call; mapping may end partially populated (use `mlock(2)` for strict semantics).

REQ-12: `MAP_DROPPABLE` (bit `0x08`) MUST be valid only with `MAP_ANONYMOUS` and `MAP_PRIVATE`. Pages MAY be dropped (zeroed) by reclaim under memory pressure; subsequent read returns zero, write re-allocates.

REQ-13: `MREMAP_FIXED` MUST require `MREMAP_MAYMOVE`. `MREMAP_DONTUNMAP` MUST require `MREMAP_MAYMOVE` AND the source mapping must be `MAP_PRIVATE | MAP_ANONYMOUS`.

REQ-14: `mremap(2)` returning a new `addr` MUST leave the old range unmapped (default) or as a PROT_NONE placeholder (`MREMAP_DONTUNMAP`).

REQ-15: `MS_ASYNC | MS_SYNC` simultaneously MUST return `-EINVAL`. Exactly one MUST be set; `MS_INVALIDATE` is orthogonal.

REQ-16: `MCL_FUTURE` alone is valid; `MCL_ONFAULT` MUST require `MCL_CURRENT` or `MCL_FUTURE` (otherwise `-EINVAL`).

REQ-17: `mlock2(2)` with `flags == 0` behaves as `mlock(2)`. `MLOCK_ONFAULT` is the only valid bit; other bits MUST return `-EINVAL`.

REQ-18: `madvise(2)` advice values MUST match the table. Unknown values MUST return `-EINVAL`. `MADV_HWPOISON` and `MADV_SOFT_OFFLINE` MUST require `CAP_SYS_ADMIN`.

REQ-19: `process_madvise(2)` with `MADV_DONTNEED`, `MADV_FREE`, `MADV_DONTNEED_LOCKED`, `MADV_COLD`, `MADV_PAGEOUT` against a remote process MUST require `PTRACE_MODE_READ` AND `CAP_SYS_NICE`.

REQ-20: `MADV_WIPEONFORK` MUST require an anonymous, private mapping. The flag is inherited; child sees zero-filled pages on first access.

REQ-21: `MADV_GUARD_INSTALL` MUST install a special guard PTE (population-time guard); any access to a guarded page MUST deliver `SIGSEGV/SEGV_ACCERR` synchronously. `MADV_GUARD_REMOVE` undoes it.

REQ-22: `MADV_COLLAPSE` is synchronous: returns only after THP collapse is attempted on the range.

REQ-23: `vm.overcommit_memory` policy values MUST be `{0,1,2}`. Other values MUST be rejected by sysctl write.

REQ-24: `pkey_mprotect(2)` MUST accept `pkey == -1` (use existing pkey of VMA) or a valid allocated pkey. PKRU bits for that pkey: `PKEY_DISABLE_ACCESS` blocks read+write, `PKEY_DISABLE_WRITE` blocks write only.

REQ-25: `MAP_HUGE_*` encoding: bits `[31:26]` carry the log2-page-size; values must be in `{14, 16, 19, 20, 21, 23, 24, 25, 28, 29, 30, 31, 34}`. Reserved bits `[26:31]` MUST be zero unless `MAP_HUGETLB` is set.

REQ-26: `struct cachestat` and `struct cachestat_range` MUST be 40 bytes and 16 bytes respectively, naturally aligned to 8 bytes.

REQ-27: `cachestat(2)` MUST require `CAP_SYS_ADMIN` for `mlocked` pages or anonymous mappings (per upstream policy); for ordinary page-cache it is unprivileged.

REQ-28: Shadow-stack flags (`map_shadow_stack(2)`) `SHADOW_STACK_SET_TOKEN`, `SHADOW_STACK_SET_MARKER` MUST be the only valid flag bits for that syscall.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prot_unknown_bits_rejected` | INVARIANT | per-mmap: `prot & !(KNOWN_PROT) != 0` ⟹ EINVAL. |
| `prot_growsdown_growsup_exclusive` | INVARIANT | per-mprotect: both flags set ⟹ EINVAL. |
| `map_type_exactly_one` | INVARIANT | per-mmap: `flags & MAP_TYPE ∈ {1, 2, 3}` else EINVAL. |
| `map_fixed_noreplace_returns_eexist` | INVARIANT | per-mmap: overlap with MAP_FIXED_NOREPLACE ⟹ EEXIST. |
| `map_sync_requires_validate_and_dax` | INVARIANT | per-mmap: MAP_SYNC without SHARED_VALIDATE/DAX ⟹ EOPNOTSUPP. |
| `huge_page_log_in_set` | INVARIANT | per-mmap: encoded log ∈ {0,14,16,19,20,21,23,24,25,28,29,30,31,34}. |
| `mremap_fixed_requires_maymove` | INVARIANT | per-mremap: FIXED without MAYMOVE ⟹ EINVAL. |
| `mremap_dontunmap_private_anon_only` | INVARIANT | per-mremap: DONTUNMAP on non-private-anon ⟹ EINVAL. |
| `msync_sync_async_exclusive` | INVARIANT | per-msync: MS_ASYNC|MS_SYNC ⟹ EINVAL. |
| `mlockall_onfault_needs_scope` | INVARIANT | per-mlockall: MCL_ONFAULT alone ⟹ EINVAL. |
| `madvise_value_in_set` | INVARIANT | per-madvise: advice ∈ defined-set else EINVAL. |
| `madvise_hwpoison_needs_cap` | INVARIANT | per-madvise: MADV_HWPOISON without CAP_SYS_ADMIN ⟹ EPERM. |
| `cachestat_struct_sizes` | INVARIANT | `sizeof(Cachestat)==40 && sizeof(CachestatRange)==16`. |

### Layer 2: TLA+

`uapi/headers/mman.tla`:
- Per-`mmap(2)` allocate → `mprotect(2)` permission-change → `madvise(2)` policy → `munmap(2)` lifecycle.
- Per-`MAP_FIXED` vs `MAP_FIXED_NOREPLACE` race against concurrent mmap.
- Per-`MADV_GUARD_INSTALL` → access → SIGSEGV delivery.
- Properties:
  - `safety_mmap_no_overlap_replace` — per-MAP_FIXED_NOREPLACE: never unmaps existing.
  - `safety_pkey_disable_blocks_access` — per-PKEY_DISABLE_ACCESS: faults SEGV_PKUERR.
  - `safety_madv_wipeonfork_zeroes_child` — per-fork: child reads zeros.
  - `safety_madv_guard_segfaults_on_touch` — per-access: synchronous SIGSEGV/SEGV_ACCERR.
  - `liveness_mremap_eventually_succeeds_with_maymove` — per-mremap: with MAYMOVE, either succeed or definite error.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MapFlags` bit layout matches header values | `MapFlags` |
| `Prot` bit layout matches header values | `Prot` |
| `Cachestat` / `CachestatRange` layout exact | structs |
| `SysMmap::sys_mmap` post: returns aligned addr ∈ user-VA or Err | `sys_mmap` |
| `SysMmap::sys_mremap` post: MREMAP_FIXED ⟹ ret == new_address | `sys_mremap` |
| `SysMmap::sys_madvise` post: range fully covered ∨ partial-Err | `sys_madvise` |
| `SysMmap::sys_msync` post: at-most-one-of {MS_SYNC, MS_ASYNC} | `sys_msync` |

### Layer 4: Verus/Creusot functional

`Per mmap(2) → validate(prot, flags) → reserve VA range → install VMA → (optional) populate/lock` semantic equivalence: per-`mmap(2)` man page, per-POSIX 1003.1. `Per madvise(2) → advice dispatch → VMA-level policy` semantic equivalence: per-`madvise(2)` man page and `mm/madvise.c` upstream.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

mman UAPI reinforcement:

- **Unknown `prot` bits rejected with EINVAL** — defense against per-future-bit accidental privilege.
- **`PROT_GROWSDOWN` / `PROT_GROWSUP` rejected on `mmap(2)` path** — defense against per-stack-growth abuse.
- **`MAP_SHARED_VALIDATE` strict unknown-flag rejection** — defense against per-future-flag silent activation.
- **`MAP_FIXED_NOREPLACE` preferred over `MAP_FIXED`** — defense against per-silent-unmap-of-victim-mapping.
- **`MAP_SYNC` ⟹ require SHARED_VALIDATE + DAX-backed file** — defense against per-non-persistent silent corruption.
- **`MAP_HUGETLB` ⟹ require arch-supported encoded size** — defense against per-invalid-hugepage allocation.
- **`MAP_LOCKED` ⟹ check `RLIMIT_MEMLOCK` and `CAP_IPC_LOCK`** — defense against per-uid memory-pinning DoS.
- **`mremap` flags strict combination check** — defense against per-flag-misuse semantic surprise.
- **`madvise` advice value strict-set check** — defense against per-future-advice silent activation.
- **`MADV_HWPOISON` / `MADV_SOFT_OFFLINE` ⟹ CAP_SYS_ADMIN** — defense against per-unprivileged page-poisoning DoS.
- **`process_madvise` ⟹ PTRACE_MODE_READ + CAP_SYS_NICE** — defense against per-cross-process page-stealing.

### grsecurity/pax-style reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user(addr, len, prot, flags, ...)` argument decode — defense against per-syscall-arg user-pointer abuse and per-kernel-pointer dereference of user-supplied addresses.
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset per `mmap`, `mremap`, `madvise` entry so any disclosure of stack-resident pointers (e.g., a VMA pointer leaked via cache-stat) is statistically useless.
- **PAX_PAGEEXEC + PAX_NOEXEC** — `mmap`/`mprotect` requesting `PROT_EXEC` is mediated by per-binary policy (e.g., `XATTR_PAX_FLAGS` on the executable). Non-`xattr`-blessed processes get `PROT_EXEC` silently dropped to non-executable; data pages stay non-executable enforced by NX page bit AND page-table pruning even on processors without NX.
- **GRKERNSEC_MAP_NOEXEC** — `MAP_PRIVATE | MAP_ANONYMOUS` defaults to non-executable; userspace must explicitly `mprotect(PROT_EXEC)` (and that mprotect itself is gated by PAX MPROTECT).
- **PaX MPROTECT** — after a process passes its "init phase" (signaled by the binary's `XATTR_PAX_FLAGS` or first call to a sentinel), all subsequent `mprotect(PROT_EXEC)` calls return `-EACCES`. JIT engines must declare their need at build time and opt out via xattr; this kills the W^X-violation class of bug for "normal" code.
- **MAP_FIXED rejection unless previously-mapped** — kernel refuses `MAP_FIXED` over an unmapped hole if it would silently grow the mapping; only an explicit overlap with existing VMA is honored, otherwise return `-ENOMEM`. This is stricter than upstream and roughly matches `MAP_FIXED_NOREPLACE` semantics as the default.
- **`MAP_DENYWRITE` honored** for `text_relocations`-free ELFs — defense against per-shared-text overwrite.
- **GRKERNSEC_BRUTE on suspicious failures** — repeated `mprotect` `-EACCES` from the same uid inside a short window triggers `SIGKILL` of the offender and a cooldown, defeating brute-force ASLR/PaX-flag scans.
- **PAX_SEGMEXEC / PAX_ASLR coordination** — VA-space layout randomization is per-execve, with the data/code/heap/stack/mmap segments randomized independently; `MAP_GROWSDOWN` stack regions get an additional per-thread randomization slot.
- **`MADV_DONTFORK` strongly recommended for sensitive pages** — Rookery emits a kernel-log warning (rate-limited) when a process performs `fork(2)` while holding mmaps with `PROT_WRITE | PROT_EXEC` and no `MADV_DONTFORK`, surfacing W^X-violations early.
- **`MAP_UNINITIALIZED` always returns `-EINVAL` on MMU systems** — Rookery refuses the no-MMU escape hatch; even with `CONFIG_MMAP_ALLOW_UNINITIALIZED`, MMU kernels zero-fill unconditionally.
- **PaX KERNEXEC-coordinated `PROT_EXEC` mapping**: requesting executable pages from a non-blessed binary forces a `userspace-stage-2` re-mmap path so the kernel image cannot be tricked into reading user code as kernel.

