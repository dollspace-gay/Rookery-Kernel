# Tier-5 syscall: modify_ldt(2) — syscall 154

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/kernel/ldt.c
  - arch/x86/include/asm/ldt.h
  - arch/x86/include/uapi/asm/ldt.h (struct user_desc)
  - arch/x86/entry/syscalls/syscall_64.tbl (154  common  modify_ldt)
  - arch/x86/entry/syscalls/syscall_32.tbl (123  i386    modify_ldt)
-->

## Summary

`modify_ldt(2)` reads, writes, or resets the calling process's Local Descriptor Table — the per-mm table of x86 segment descriptors used for legacy 16-bit / 32-bit segmented code, wine's Win32 TLS slots, and DOSEMU-style real-mode emulation. The LDT is a per-`mm_struct` object: forked children share it COW until one of them mutates, then a new LDT page is allocated. Each entry is an 8-byte (`USER_DESC`) descriptor selecting a base, limit, type (code/data), and DPL. Userland references LDT entries by selector `(index << 3) | 4 | 3` (table-indicator = 1, RPL = 3). Critical for: wine 32-bit, DOSEMU2, native 16-bit segment-using legacy binaries, and any program that uses `set_thread_area`-style TLS via the LDT.

Modern x86_64 toolchains never touch the LDT — `set_thread_area` / FS-base via `arch_prctl` replaced it. The LDT remains for ABI compatibility and is the most CVE-prone segmentation surface (Meltdown-era / LDT-mapping bugs, CVE-2015-5157, CVE-2017-17807). PaX `GRKERNSEC_LDT` blocks the syscall entirely by default.

This Tier-5 covers `modify_ldt(2)` on x86_64 and i386. There is no compat shim — the struct layout is identical.

## Signature

```c
int modify_ldt(int func, void *ptr, unsigned long bytecount);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `func` | `int` | in | Operation: 0=read, 1=write, 2=read_default, 0x11=write_default. |
| `ptr` | `void *` | in/out | Per-`func` 0/2: buffer to receive bytes; per-`func` 1/0x11: pointer to one `struct user_desc`. |
| `bytecount` | `unsigned long` | in | Per-`func` 0/2: bytes to read (≤ LDT size); per-`func` 1/0x11: `sizeof(struct user_desc)` (16). |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Per-read: bytes copied. Per-write: 0. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL`   | Unknown `func`; `bytecount` not a multiple of 8 on read; `bytecount != sizeof(struct user_desc)` on write; reserved descriptor bits set; descriptor type invalid (sys / call-gate / TSS forbidden); base > 0xFFFFFFFF on 32-bit. |
| `EFAULT`   | `ptr` is not a valid user pointer. |
| `ENOSYS`   | Kernel compiled with `CONFIG_MODIFY_LDT_SYSCALL=n` (grsec / hardened default). |
| `ENOMEM`   | Per-LDT-grow allocation failed. |
| `EPERM`    | Per-grsec policy: `GRKERNSEC_LDT` blocks all callers. |

## ABI surface

```text
__NR_modify_ldt (x86_64) = 154
__NR_modify_ldt (i386)   = 123

func values:
  0  = read     (copies up to bytecount bytes of LDT into *ptr)
  1  = write    (installs one descriptor at user_desc.entry_number)
  2  = read_default  (copies LDT_ENTRIES * 8 zeroes — historical)
  0x11 = write_default  (write with stricter validation; man-page deprecated alias)

struct user_desc {                /* 16 bytes — uapi/asm/ldt.h */
    unsigned int  entry_number;   /* 0 .. LDT_ENTRIES-1 (=8191) */
    unsigned int  base_addr;      /* 32-bit base */
    unsigned int  limit;          /* 20-bit limit; bit-granularity per flag */
    unsigned int  seg_32bit:1;
    unsigned int  contents:2;     /* 0=data/up, 1=data/down, 2=code/!conf, 3=code/conf */
    unsigned int  read_exec_only:1;
    unsigned int  limit_in_pages:1;
    unsigned int  seg_not_present:1;
    unsigned int  useable:1;      /* AVL bit */
    unsigned int  lm:1;           /* x86_64 only: long-mode segment */
};

LDT_ENTRIES   = 8192    /* maximum entries */
LDT_ENTRY_SIZE= 8       /* bytes per descriptor */
LDT_size_max  = LDT_ENTRIES * 8 = 65 536 bytes
```

### Limits

```text
entry_number   : 0 .. 8191
bytecount on read   : 0 .. 65 536, must be 8-aligned
bytecount on write  : exactly 16 (sizeof(struct user_desc))
base_addr (i386)    : 0 .. 0xFFFFFFFF
base_addr (x86_64)  : 0 .. 0xFFFFFFFF  (the LDT cannot describe high-half kernel addrs)
descriptor type     : data or code segments only; call-gates / TSS / LDT descriptors forbidden
```

## Compatibility contract

REQ-1: Syscall number is **154** on x86_64, **123** on i386. ABI-stable since 1.0.

REQ-2: `func == 0` (read): copies `min(bytecount, mm.context.ldt.nr_entries * 8)` bytes from the in-kernel LDT image into `*ptr`. Returns bytes copied. If no LDT installed, returns 0 (NOT `-EINVAL`).

REQ-3: `func == 2` (read_default): writes `bytecount` zero bytes; historical, equivalent to "read system default LDT" which on Linux is empty.

REQ-4: `func == 1 || func == 0x11` (write): `bytecount` MUST equal `sizeof(struct user_desc)` (16). The kernel copies the descriptor in, validates, encodes it into the 8-byte hardware descriptor format, and installs it at `entry_number`.

REQ-5: Write validation: `entry_number < LDT_ENTRIES`. `contents ∈ {0,1,2,3}` — only data/code permitted; system descriptors (gates, TSS, LDT, task-gate) are silently coerced to "not present" and rejected via `-EINVAL` on certain bits.

REQ-6: Per-write: `seg_not_present == 1 && useable == 0 && contents == 0 && read_exec_only == 1 && base == 0 && limit == 0` is the "empty descriptor" — installs a null entry (clears the slot).

REQ-7: Per-write: DPL is hard-coded to 3 (user). The kernel never installs DPL 0/1/2 from userspace, regardless of input bits. Bits 5-6 of the access byte are forced to `0b11`.

REQ-8: Per-write: the `lm` bit (long mode) is honored only on x86_64. When set, the descriptor is a 64-bit code segment; `seg_32bit` is forced consistent. The kernel rejects `lm == 1 && contents != 2` (long mode is code-only).

REQ-9: Per-LDT lifecycle: the LDT is a per-`mm_struct` resource. `fork()` shares it copy-on-write (the LDT physical pages are reused). First `modify_ldt(write)` after fork allocates a new LDT (per-mm), copies the parent's entries, installs the new descriptor.

REQ-10: Per-LDT pages: kernel allocates LDT pages from a dedicated slab (`ldt_struct`) and maps them in the per-cpu `cpu_entry_area` for the LDTR-load path. PTI maps these read-only to user-mode-cpl-side mapping.

REQ-11: Per-context-switch: when the task is scheduled in, the kernel does `lldt(mm.context.ldt.selector)` only if `mm` changed. The mapping in `cpu_entry_area` is updated atomically against concurrent `modify_ldt`.

REQ-12: Per-execve: the LDT is freed; the new process starts with an empty LDT. Threads in the same `mm` continue to share the new LDT.

REQ-13: Per-CONFIG_MODIFY_LDT_SYSCALL=n: the entire syscall returns `-ENOSYS`. This is the default on `CONFIG_HARDENED` builds and unconditional under PaX `GRKERNSEC_LDT`.

REQ-14: `func == 0x11` ("write_default", a.k.a. `MODIFY_LDT_CONTENTS_DATA` historical alias): identical to `func == 1` except that descriptors with `seg_not_present == 1` are still installed (legacy DOSEMU expects to install "not-present" guard descriptors).

REQ-15: Unknown `func` returns `-EINVAL`; no state mutation.

REQ-16: Per-LDT-vs-GDT selectors: descriptors installed live in LDT, accessed via selector `(idx << 3) | 0x07` (TI=1, RPL=3). The GDT is untouched.

REQ-17: Per-x86_64 specific: a properly formed 32-bit code descriptor in the LDT allows a 64-bit process to `lcall` into 32-bit code; the kernel honors this for wine.

REQ-18: Per-thread visibility: writes to the LDT are visible across all threads of the `mm` immediately after `modify_ldt` returns (memory barrier + IPI broadcast TLB-style invalidation on the LDTR mapping).

## Acceptance Criteria

- [ ] AC-1: `modify_ldt(0, buf, 64)` on a process with no LDT returns 0 and does not touch `buf`.
- [ ] AC-2: `modify_ldt(1, &desc, 16)` with valid 32-bit data descriptor at `entry_number=0` returns 0; subsequent read returns the encoded 8-byte form.
- [ ] AC-3: `modify_ldt(1, &desc, 8)` returns `-EINVAL` (wrong bytecount).
- [ ] AC-4: `modify_ldt(1, &desc, 16)` with `entry_number=8192` returns `-EINVAL`.
- [ ] AC-5: `modify_ldt(1, NULL, 16)` returns `-EFAULT`.
- [ ] AC-6: `modify_ldt(3, ..., ...)` (unknown func) returns `-EINVAL`.
- [ ] AC-7: Per-fork: child sees parent's LDT entries on read; child's subsequent write does not modify parent's LDT.
- [ ] AC-8: After execve, LDT is empty: `modify_ldt(0, buf, 65536)` returns 0.
- [ ] AC-9: Per-write of empty descriptor (all-zero `user_desc` with `seg_not_present=1`) at occupied slot clears that slot.
- [ ] AC-10: `modify_ldt(1, &desc, 16)` with `contents=3` and `lm=1` installs a 64-bit conforming code segment.
- [ ] AC-11: Concurrent thread reading LDT during another thread's write never observes a torn descriptor (atomic 8-byte install).
- [ ] AC-12: Kernel compiled `CONFIG_MODIFY_LDT_SYSCALL=n`: syscall returns `-ENOSYS`.
- [ ] AC-13: Grsec `GRKERNSEC_LDT=y`: syscall returns `-EPERM` for all callers (logged).
- [ ] AC-14: Per-LTP `modify_ldt01..03` pass on a CONFIG_MODIFY_LDT_SYSCALL=y kernel.

## Architecture

```rust
#[syscall(nr = 154, abi = "sysv")]
pub fn sys_modify_ldt(func: i32, ptr: UserPtr<u8>, bytecount: usize) -> isize {
    if !cfg!(feature = "modify_ldt_syscall") { return -ENOSYS; }
    if grsec::ldt_blocked() { return -EPERM; }
    ModifyLdt::dispatch(func, ptr, bytecount)
}
```

`ModifyLdt::dispatch(func, ptr, bytecount) -> isize`:
1. match func {
2.   0    => ModifyLdt::read(ptr, bytecount, /*default=*/false),
3.   2    => ModifyLdt::read(ptr, bytecount, /*default=*/true),
4.   1    => ModifyLdt::write(ptr, bytecount, /*allow_not_present=*/false),
5.   0x11 => ModifyLdt::write(ptr, bytecount, /*allow_not_present=*/true),
6.   _    => Err(EINVAL),
7. }

`ModifyLdt::read(ptr, bytecount, default) -> isize`:
1. if default { return UserPtr::write_zero(ptr, bytecount).map(|_| bytecount as isize); }
2. let mm = current.mm();
3. let ldt = mm.context.ldt.read();
4. let avail = ldt.as_ref().map_or(0, |l| l.nr_entries * 8);
5. let n = min(bytecount, avail);
6. if n > 0 { ptr.write_from(&ldt.as_ref().unwrap().entries[..n])?; }
7. Ok(n as isize)

`ModifyLdt::write(ptr, bytecount, allow_not_present) -> isize`:
1. if bytecount != size_of::<UserDesc>() { return Err(EINVAL); }
2. let desc: UserDesc = ptr.read()?;
3. ModifyLdt::validate_user_desc(&desc, allow_not_present)?;
4. let enc: u64 = encode_descriptor(&desc);   // DPL forced to 3
5. let mm = current.mm();
6. let mut ldt = mm.context.ldt.write_cow();  // COW from parent if shared
7. ldt.install(desc.entry_number, enc);
8. cpu_entry_area::refresh_ldt(&ldt);
9. Ok(0)

`ModifyLdt::validate_user_desc(d, allow_np) -> Result<(),i32>`:
1. if d.entry_number >= LDT_ENTRIES { return Err(EINVAL); }
2. if d.contents > 3 { return Err(EINVAL); }
3. if d.lm == 1 && d.contents != 2 { return Err(EINVAL); }  // long mode = code only
4. if !allow_np && d.seg_not_present == 1 && !is_empty_desc(d) { return Err(EINVAL); }
5. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bytecount_validated` | INVARIANT | per-write: bytecount == 16. |
| `entry_in_range` | INVARIANT | per-write: entry_number < LDT_ENTRIES. |
| `dpl_forced_to_3` | INVARIANT | per-encode: DPL bits == 0b11 regardless of input. |
| `system_desc_rejected` | INVARIANT | per-encode: type-bits never describe gate/TSS/LDT. |
| `read_clamps_bytecount` | INVARIANT | per-read: bytes_copied ≤ min(bytecount, ldt_size). |
| `cow_on_write` | INVARIANT | per-write: if ldt shared, allocate new before mutate. |
| `unknown_func_einval` | INVARIANT | unknown func ⟹ -EINVAL, no state change. |

### Layer 2: TLA+

`arch/x86/modify_ldt.tla`:
- States: per-mm LDT array; refcount on shared LDT page; per-cpu LDTR mapping.
- Properties:
  - `safety_dpl3` — every installed descriptor has DPL == 3.
  - `safety_no_system_desc` — type-byte never matches system descriptors.
  - `safety_cow` — write to shared LDT allocates new page before mutating.
  - `safety_read_no_tear` — reader sees full 8-byte descriptor; writer's mid-write state invisible.
  - `liveness_dispatch_returns` — every call terminates with errno or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `write` post: ldt[entry] == encode(desc), DPL=3 | `ModifyLdt::write` |
| `read` post: bytes_copied == min(bytecount, ldt_size) | `ModifyLdt::read` |
| `validate_user_desc` post: ok ⟹ desc is data/code, in-range | `ModifyLdt::validate_user_desc` |
| `cow_on_write` post: refcount(ldt) == 1 after write | `Mm::ldt.write_cow` |

### Layer 4: Verus/Creusot functional

Per-`modify_ldt(2)` man-page semantic equivalence. LTP `modify_ldt01..03` and DOSEMU2 / wine 32-bit smoke pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`modify_ldt(2)` reinforcement:

- **Per-DPL-3 forced** — defense against per-ring-0 descriptor injection (historical CVE class).
- **Per-system-descriptor reject** — defense against per-call-gate / per-TSS-pointer escalation.
- **Per-COW LDT page** — defense against per-cross-mm corruption.
- **Per-PTI LDT mapping read-only on user side** — defense against per-Meltdown LDT read.
- **Per-`cpu_entry_area` LDT mapping bounded** — defense against per-LDT-overflow into adjacent kernel data.
- **Per-`lm` bit gated on x86_64** — defense against per-32-bit-process forging 64-bit code segment to escape sandbox.
- **Per-LDT entries 8 KiB max** — defense against per-large-LDT DoS.

## Grsecurity / PaX surface

- **GRKERNSEC_LDT** — `CONFIG_MODIFY_LDT_SYSCALL=n` is the grsec default; if enabled, syscall returns `-EPERM` and logs the attempt with task creds + RIP.
- **PaX UDEREF on `ptr`** — defense against per-malicious-user-pointer kernel deref bug.
- **PAX_RANDKSTACK at modify_ldt entry** — randomizes kernel stack offset; defeats LDT-related stack-info leaks.
- **GRKERNSEC_KMEM** — kernel LDT image never exposed via `/dev/kmem` or `/proc/<pid>/mem`.
- **PaX SEGMEXEC integration** — under SEGMEXEC, the kernel reserves segments 0x33/0x3B and refuses any LDT entry that overlaps; `-EINVAL`.
- **Per-grsec audit** — every successful write logged with `entry_number, base, limit, contents, flags`.
- **GRKERNSEC_CHROOT_NO_LDT** — inside a grsec chroot, `modify_ldt(1/0x11)` returns `-EPERM` even if syscall is otherwise allowed.
- **PaX MPROTECT integration** — `modify_ldt` cannot install a code descriptor whose base/limit overlaps a non-executable PaX vma.
- **Per-grsec RBAC** — the `+L` flag (LDT-allow) gates per-subject access; default policy denies.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- x86 segmentation hardware semantics (Tier-3 in `arch/x86/segments.md`).
- `set_thread_area` / `get_thread_area` (separate Tier-5 docs).
- LDT-via-`cpu_entry_area` page-table machinery (Tier-3 in `arch/x86/cpu_entry_area.md`).
- wine / DOSEMU2 userspace integration.
- Implementation code.
