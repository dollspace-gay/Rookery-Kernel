---
title: "Tier-5 UAPI: include/uapi/linux/elf.h — ELF binary format ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The ELF UAPI is the binary-format contract for every executable, shared object, and core dump the kernel produces or consumes. It backs `execve(2)` for `ET_EXEC` and `ET_DYN` (PIE) binaries via `fs/binfmt_elf.c`, the dynamic-linker handoff via the auxiliary vector pushed on the new process stack, the `/proc/<pid>/coredump` and `coredump(5)` pipeline that emits `ET_CORE` files, the `ptrace(PTRACE_GETREGSET, NT_*, ...)` register-set transfer, the kernel's own livepatch and module loader (`kernel/livepatch`, `kernel/module/main.c`), kexec image loading (`kernel/kexec_file.c`), and the BPF object loader (libbpf reads ELF via the kernel-defined symbol table layouts). It defines:

- **ELF base typedefs** `Elf32_Addr/Half/Off/Sword/Word/Versym`, `Elf64_Addr/Half/SHalf/Off/Sword/Word/Xword/Sxword/Versym`.
- **ELF header** `Elf32_Ehdr` / `Elf64_Ehdr` with `e_ident[EI_NIDENT=16]` magic + class + data + version + osabi, plus `e_type`, `e_machine`, `e_version`, `e_entry`, `e_phoff`, `e_shoff`, `e_flags`, `e_ehsize`, `e_phentsize`, `e_phnum`, `e_shentsize`, `e_shnum`, `e_shstrndx`.
- **`e_ident` indices** `EI_MAG0..3` (magic `\x7fELF`), `EI_CLASS` (`ELFCLASSNONE/32/64`), `EI_DATA` (`ELFDATANONE/2LSB/2MSB`), `EI_VERSION`, `EI_OSABI` (`ELFOSABI_NONE`, `ELFOSABI_LINUX`), `EI_PAD`.
- **`e_type`** `ET_NONE`, `ET_REL`, `ET_EXEC`, `ET_DYN`, `ET_CORE`, `ET_LOPROC`, `ET_HIPROC`.
- **`e_machine`** the full `EM_*` table from `elf-em.h`: `EM_386`, `EM_PPC`, `EM_PPC64`, `EM_ARM`, `EM_X86_64`, `EM_IA_64`, `EM_AARCH64`, `EM_RISCV`, `EM_LOONGARCH`, `EM_S390`, `EM_BPF`, etc.
- **Program header** `Elf32_Phdr` / `Elf64_Phdr` with `p_type`, `p_offset`, `p_vaddr`, `p_paddr`, `p_filesz`, `p_memsz`, `p_flags` (`PF_R=4`, `PF_W=2`, `PF_X=1`), `p_align`.
- **`p_type` values** `PT_NULL`, `PT_LOAD`, `PT_DYNAMIC`, `PT_INTERP`, `PT_NOTE`, `PT_SHLIB`, `PT_PHDR`, `PT_TLS`, `PT_LOOS`, `PT_HIOS`, `PT_LOPROC`, `PT_HIPROC`, GNU extensions `PT_GNU_EH_FRAME`, `PT_GNU_STACK`, `PT_GNU_RELRO`, `PT_GNU_PROPERTY`, plus arch-specific `PT_AARCH64_MEMTAG_MTE`. `PN_XNUM = 0xffff` is the extended-numbering sentinel.
- **Section header** `Elf32_Shdr` / `Elf64_Shdr` with `sh_name`, `sh_type`, `sh_flags`, `sh_addr`, `sh_offset`, `sh_size`, `sh_link`, `sh_info`, `sh_addralign`, `sh_entsize`.
- **`sh_type`** `SHT_NULL`, `SHT_PROGBITS`, `SHT_SYMTAB`, `SHT_STRTAB`, `SHT_RELA`, `SHT_HASH`, `SHT_DYNAMIC`, `SHT_NOTE`, `SHT_NOBITS`, `SHT_REL`, `SHT_SHLIB`, `SHT_DYNSYM`, plus `SHT_LOPROC/HIPROC/LOUSER/HIUSER`. `sh_flags` bits `SHF_WRITE`, `SHF_ALLOC`, `SHF_EXECINSTR`, `SHF_MERGE`, `SHF_STRINGS`, `SHF_INFO_LINK`, `SHF_LINK_ORDER`, `SHF_OS_NONCONFORMING`, `SHF_GROUP`, `SHF_TLS`, `SHF_RELA_LIVEPATCH`, `SHF_RO_AFTER_INIT`, `SHF_ORDERED`, `SHF_EXCLUDE`.
- **Special section indices** `SHN_UNDEF`, `SHN_LORESERVE/LOPROC/HIPROC`, `SHN_LIVEPATCH`, `SHN_ABS`, `SHN_COMMON`, `SHN_HIRESERVE`.
- **Symbol table** `Elf32_Sym` / `Elf64_Sym` with `st_name`, `st_info` (via `ELF{32,64}_ST_BIND` / `_ST_TYPE`), `st_other`, `st_shndx`, `st_value`, `st_size`. Bindings `STB_LOCAL/GLOBAL/WEAK`. Types `STT_NOTYPE/OBJECT/FUNC/SECTION/FILE/COMMON/TLS`.
- **Relocations** `Elf32_Rel` / `Elf64_Rel` / `Elf32_Rela` / `Elf64_Rela` + the `ELF{32,64}_R_SYM` / `_R_TYPE` extractors.
- **Dynamic table** `Elf32_Dyn` / `Elf64_Dyn` with the `DT_*` tag space (`DT_NULL`, `DT_NEEDED`, `DT_PLTRELSZ`, `DT_PLTGOT`, `DT_HASH`, `DT_STRTAB`, `DT_SYMTAB`, `DT_RELA`, `DT_RELASZ`, `DT_RELAENT`, `DT_STRSZ`, `DT_SYMENT`, `DT_INIT`, `DT_FINI`, `DT_SONAME`, `DT_RPATH`, `DT_SYMBOLIC`, `DT_REL`, `DT_RELSZ`, `DT_RELENT`, `DT_PLTREL`, `DT_DEBUG`, `DT_TEXTREL`, `DT_JMPREL`, `DT_ENCODING`, `DT_GNU_HASH`, `DT_VERSYM`, `DT_VERDEF`, `DT_VERNEED`, `DT_FLAGS_1`, `DT_RELACOUNT`, `DT_RELCOUNT`, `DT_LOPROC`, `DT_HIPROC`).
- **Notes** `Elf32_Nhdr` / `Elf64_Nhdr` (`n_namesz`, `n_descsz`, `n_type`) + the full `NT_*` enumeration with `NN_*` name strings: `NT_PRSTATUS` (1), `NT_PRFPREG` (2), `NT_PRPSINFO` (3), `NT_TASKSTRUCT` (4), `NT_AUXV` (6), `NT_SIGINFO` (0x53494749), `NT_FILE` (0x46494c45), `NT_PRXFPREG` (0x46e62b7f), `NT_X86_XSTATE` (0x202), `NT_X86_SHSTK` (0x204), `NT_GNU_PROPERTY_TYPE_0` (5), plus arch-specific NTs for PPC, S390, ARM (`NT_ARM_VFP`, `NT_ARM_SVE`, `NT_ARM_GCS`, …), MIPS, RISC-V, LoongArch.
- **Auxiliary vector `AT_*`** from `auxvec.h`: `AT_NULL` (0), `AT_IGNORE`, `AT_EXECFD`, `AT_PHDR`, `AT_PHENT`, `AT_PHNUM`, `AT_PAGESZ`, `AT_BASE`, `AT_FLAGS`, `AT_ENTRY`, `AT_NOTELF`, `AT_UID`, `AT_EUID`, `AT_GID`, `AT_EGID`, `AT_PLATFORM`, `AT_HWCAP`, `AT_CLKTCK`, `AT_SECURE` (23), `AT_BASE_PLATFORM`, `AT_RANDOM`, `AT_HWCAP2`, `AT_RSEQ_FEATURE_SIZE`, `AT_RSEQ_ALIGN`, `AT_HWCAP3`, `AT_HWCAP4`, `AT_EXECFN`, `AT_MINSIGSTKSZ` (51).
- **Versioning** `Elf{32,64}_Verdef`, `Elf{32,64}_Verdaux`, `VER_FLG_BASE`, `VER_FLG_WEAK`.

Critical for: every userspace program loader, every dynamic linker (glibc ld-linux, musl ld-musl, bionic linker), `gdb`/`lldb` core-dump readers, `readelf`/`objcopy`/`objdump`/`strip`, `coredumpctl`, livepatch (`kpatch`, `kgraft`), kexec image loaders, libbpf, every JIT that emits ELF objects (V8 perf-map, jvm-perf-agent), every kernel module load path.

This Tier-5 covers `include/uapi/linux/elf.h` (~615 lines) plus the tightly-coupled `elf-em.h` and `auxvec.h`. The corresponding implementations (`fs/binfmt_elf.c`, `fs/binfmt_elf_fdpic.c`, `kernel/module/main.c`, `kernel/kexec_file.c`, `fs/coredump.c`) are Tier-3 docs separately.

### Acceptance Criteria

- [ ] AC-1: `e_ident[0..4] != "\x7fELF"`: `execve` returns `-ENOEXEC`.
- [ ] AC-2: `EI_CLASS == ELFCLASS64` for x86_64 binaries; `ELFCLASS32` for i386.
- [ ] AC-3: `EI_DATA == ELFDATA2LSB` on x86_64; field byte-order matches loader's parse.
- [ ] AC-4: `e_phentsize != sizeof(Elf64_Phdr) == 56`: `-ENOEXEC`.
- [ ] AC-5: `ET_DYN` PIE: load bias randomized when ASLR enabled.
- [ ] AC-6: `PT_LOAD` with `PF_W | PF_X`: W^X policy refuses or downgrades.
- [ ] AC-7: `PT_GNU_STACK` absent on x86_64: stack VMA gets `PROT_READ | PROT_WRITE` (NX).
- [ ] AC-8: `PT_GNU_STACK.p_flags & PF_X == 0`: stack VMA never gets PROT_EXEC.
- [ ] AC-9: `PT_GNU_RELRO`: dynamic linker mprotects the region to `PROT_READ` post-relocation.
- [ ] AC-10: `PT_INTERP` chain depth > 4: `-ELOOP`.
- [ ] AC-11: `e_phnum == PN_XNUM`: kernel reads true count from `Shdr[0].sh_info`.
- [ ] AC-12: ptrace(PTRACE_GETREGSET, NT_PRSTATUS) returns `Elf{32,64}_Prstatus`-shaped buffer.
- [ ] AC-13: core dump contains `NT_PRSTATUS`, `NT_PRFPREG`, `NT_PRPSINFO`, `NT_SIGINFO`, `NT_AUXV`, `NT_FILE`.
- [ ] AC-14: `AT_RANDOM` points to 16 bytes of kernel randomness; reading is page-aligned and per-exec unique.
- [ ] AC-15: setuid exec: `AT_SECURE != 0`; libc skips `LD_LIBRARY_PATH` / `LD_PRELOAD` expansions.
- [ ] AC-16: `AT_PHDR + AT_PHENT * AT_PHNUM` covers exactly the executable's program header table.
- [ ] AC-17: `AT_ENTRY == e_entry + load_bias` for ET_DYN.
- [ ] AC-18: `AT_PAGESZ == 4096` on x86_64 default.
- [ ] AC-19: `AT_HWCAP` bit for `SSE2` set on every x86_64 (mandatory baseline).
- [ ] AC-20: `AT_NULL` terminates the auxv blob.
- [ ] AC-21: `EM_X86_64 == 62`, `EM_AARCH64 == 183`, `EM_RISCV == 243`, `EM_LOONGARCH == 258`, `EM_BPF == 247`.
- [ ] AC-22: `GNU_PROPERTY_AARCH64_FEATURE_1_BTI`: when present in `PT_GNU_PROPERTY`, kernel sets BTI-enforced VMA flag on PT_LOAD.

### Architecture

```
// Base types
pub type Elf32Addr   = u32;
pub type Elf32Half   = u16;
pub type Elf32Off    = u32;
pub type Elf32Sword  = i32;
pub type Elf32Word   = u32;
pub type Elf32Versym = u16;

pub type Elf64Addr   = u64;
pub type Elf64Half   = u16;
pub type Elf64SHalf  = i16;
pub type Elf64Off    = u64;
pub type Elf64Sword  = i32;
pub type Elf64Word   = u32;
pub type Elf64Xword  = u64;
pub type Elf64Sxword = i64;
pub type Elf64Versym = u16;

pub const EI_NIDENT: usize = 16;
pub const SELFMAG: usize = 4;
pub const ELFMAG: [u8; 4] = [0x7f, b'E', b'L', b'F'];

pub const ELFCLASSNONE: u8 = 0;
pub const ELFCLASS32:   u8 = 1;
pub const ELFCLASS64:   u8 = 2;
pub const ELFCLASSNUM:  u8 = 3;

pub const ELFDATANONE: u8 = 0;
pub const ELFDATA2LSB: u8 = 1;
pub const ELFDATA2MSB: u8 = 2;

pub const EV_NONE:    u32 = 0;
pub const EV_CURRENT: u32 = 1;

pub const ELFOSABI_NONE:  u8 = 0;
pub const ELFOSABI_LINUX: u8 = 3;

// e_type
pub const ET_NONE: u16   = 0;
pub const ET_REL:  u16   = 1;
pub const ET_EXEC: u16   = 2;
pub const ET_DYN:  u16   = 3;
pub const ET_CORE: u16   = 4;

#[repr(C)]
pub struct Elf64Hdr {
    pub e_ident:     [u8; EI_NIDENT],
    pub e_type:      Elf64Half,
    pub e_machine:   Elf64Half,
    pub e_version:   Elf64Word,
    pub e_entry:     Elf64Addr,
    pub e_phoff:     Elf64Off,
    pub e_shoff:     Elf64Off,
    pub e_flags:     Elf64Word,
    pub e_ehsize:    Elf64Half,
    pub e_phentsize: Elf64Half,
    pub e_phnum:     Elf64Half,
    pub e_shentsize: Elf64Half,
    pub e_shnum:     Elf64Half,
    pub e_shstrndx:  Elf64Half,
}
const _ASSERT_EHDR64: () = assert!(core::mem::size_of::<Elf64Hdr>() == 64);

#[repr(C)]
pub struct Elf64Phdr {
    pub p_type:   Elf64Word,
    pub p_flags:  Elf64Word,
    pub p_offset: Elf64Off,
    pub p_vaddr:  Elf64Addr,
    pub p_paddr:  Elf64Addr,
    pub p_filesz: Elf64Xword,
    pub p_memsz:  Elf64Xword,
    pub p_align:  Elf64Xword,
}
const _ASSERT_PHDR64: () = assert!(core::mem::size_of::<Elf64Phdr>() == 56);

#[repr(C)]
pub struct Elf64Shdr {
    pub sh_name:      Elf64Word,
    pub sh_type:      Elf64Word,
    pub sh_flags:     Elf64Xword,
    pub sh_addr:      Elf64Addr,
    pub sh_offset:    Elf64Off,
    pub sh_size:      Elf64Xword,
    pub sh_link:      Elf64Word,
    pub sh_info:      Elf64Word,
    pub sh_addralign: Elf64Xword,
    pub sh_entsize:   Elf64Xword,
}
const _ASSERT_SHDR64: () = assert!(core::mem::size_of::<Elf64Shdr>() == 64);

#[repr(C)]
pub struct Elf64Sym {
    pub st_name:  Elf64Word,
    pub st_info:  u8,
    pub st_other: u8,
    pub st_shndx: Elf64Half,
    pub st_value: Elf64Addr,
    pub st_size:  Elf64Xword,
}

#[repr(C)]
pub struct Elf64Nhdr {
    pub n_namesz: Elf64Word,
    pub n_descsz: Elf64Word,
    pub n_type:   Elf64Word,
}

// PT_* selected (full table per § ABI surface)
pub const PT_NULL: u32          = 0;
pub const PT_LOAD: u32          = 1;
pub const PT_DYNAMIC: u32       = 2;
pub const PT_INTERP: u32        = 3;
pub const PT_NOTE: u32          = 4;
pub const PT_PHDR: u32          = 6;
pub const PT_TLS: u32           = 7;
pub const PT_GNU_EH_FRAME: u32  = 0x6474e550;
pub const PT_GNU_STACK: u32     = 0x6474e551;
pub const PT_GNU_RELRO: u32     = 0x6474e552;
pub const PT_GNU_PROPERTY: u32  = 0x6474e553;
pub const PN_XNUM: u16          = 0xffff;

// PF_*
pub const PF_X: u32 = 0x1;
pub const PF_W: u32 = 0x2;
pub const PF_R: u32 = 0x4;
```

`BinfmtElf::load_binary(bprm, hdr_user) -> Result<()>`:
1. let ehdr: Elf64Hdr = read_ehdr(bprm)?;
2. /* Magic */
3. if ehdr.e_ident[..4] != ELFMAG: return Err(ENOEXEC).
4. /* Class / data / version */
5. if ehdr.e_ident[EI_CLASS] != ELFCLASS64: return Err(ENOEXEC).
6. if ehdr.e_ident[EI_DATA] != ELFDATA2LSB: return Err(ENOEXEC).
7. if ehdr.e_ident[EI_VERSION] != EV_CURRENT as u8: return Err(ENOEXEC).
8. /* e_type */
9. if !matches!(ehdr.e_type, ET_EXEC | ET_DYN): return Err(ENOEXEC).
10. /* e_machine matches kernel arch */
11. if ehdr.e_machine != EM_X86_64: return Err(ENOEXEC).
12. /* Size sanity */
13. if ehdr.e_ehsize != 64: return Err(ENOEXEC).
14. if ehdr.e_phentsize != 56: return Err(ENOEXEC).
15. let phnum = if ehdr.e_phnum == PN_XNUM { read_shdr0_sh_info(bprm)? } else { ehdr.e_phnum as u32 };
16. if phnum > MAX_PHDRS: return Err(ENOEXEC).
17. let phdrs = read_phdrs(bprm, ehdr.e_phoff, phnum)?;
18. /* Process PT_INTERP first */
19. let interp = phdrs.iter().find(|p| p.p_type == PT_INTERP);
20. if let Some(i) = interp {
        if recursion_depth(bprm) >= 4: return Err(ELOOP).
        let interp_path = read_string(bprm, i.p_offset, i.p_filesz)?;
        bprm.interp = Some(open_path(&interp_path)?);
    }
21. /* Compute load bias */
22. let load_bias = match ehdr.e_type {
        ET_EXEC => 0,
        ET_DYN  => arch_randomize_brk(current),
        _       => return Err(ENOEXEC),
    };
23. /* Map PT_LOAD segments */
24. for p in &phdrs {
        if p.p_type != PT_LOAD: continue;
        if p.p_filesz > p.p_memsz: return Err(ENOEXEC).
        let prot = phflags_to_prot(p.p_flags);
        elf_map(bprm.file, load_bias + p.p_vaddr, p, prot, MAP_FIXED | MAP_PRIVATE)?;
        if p.p_memsz > p.p_filesz {
            elf_zero_tail(load_bias + p.p_vaddr + p.p_filesz, p.p_memsz - p.p_filesz)?;
        }
    }
25. /* Apply PT_GNU_STACK policy */
26. let exec_stack = phdrs.iter().find(|p| p.p_type == PT_GNU_STACK)
        .map(|p| p.p_flags & PF_X != 0)
        .unwrap_or(arch_default_exec_stack());
27. bprm.exec_stack = exec_stack;
28. /* PT_GNU_PROPERTY -> CET/BTI */
29. if let Some(p) = phdrs.iter().find(|p| p.p_type == PT_GNU_PROPERTY) {
        apply_gnu_property(bprm, p)?;
    }
30. /* Mark PT_GNU_RELRO region */
31. if let Some(p) = phdrs.iter().find(|p| p.p_type == PT_GNU_RELRO) {
        bprm.relro = Some((load_bias + p.p_vaddr, p.p_memsz));
    }
32. /* Build auxv */
33. push_auxv(bprm, ehdr, load_bias, &phdrs)?;
34. /* Transfer control: load interp if present, else jump to ehdr.e_entry + load_bias */
35. start_thread(...).

`push_auxv(bprm, ehdr, load_bias, phdrs)`:
1. let mut auxv = Vec::<AuxEntry>::with_capacity(40);
2. auxv.push(AT_PHDR,   load_bias + ehdr.e_phoff);
3. auxv.push(AT_PHENT,  ehdr.e_phentsize as u64);
4. auxv.push(AT_PHNUM,  phdrs.len() as u64);
5. auxv.push(AT_PAGESZ, PAGE_SIZE as u64);
6. auxv.push(AT_BASE,   bprm.interp_load_bias.unwrap_or(0));
7. auxv.push(AT_FLAGS,  0);
8. auxv.push(AT_ENTRY,  load_bias + ehdr.e_entry);
9. auxv.push(AT_UID,    current.ruid().raw_uid());
10. auxv.push(AT_EUID,  current.euid().raw_uid());
11. auxv.push(AT_GID,   current.rgid().raw_gid());
12. auxv.push(AT_EGID,  current.egid().raw_gid());
13. auxv.push(AT_SECURE, bprm.secureexec as u64);
14. auxv.push(AT_PLATFORM, push_string(bprm, arch_platform_string()));
15. auxv.push(AT_BASE_PLATFORM, push_string(bprm, arch_base_platform_string()));
16. auxv.push(AT_HWCAP,  arch_hwcap());
17. auxv.push(AT_HWCAP2, arch_hwcap2());
18. auxv.push(AT_HWCAP3, arch_hwcap3());
19. auxv.push(AT_HWCAP4, arch_hwcap4());
20. auxv.push(AT_CLKTCK, USER_HZ as u64);
21. auxv.push(AT_RANDOM, push_random_16(bprm));
22. auxv.push(AT_EXECFN, push_string(bprm, &bprm.filename));
23. auxv.push(AT_RSEQ_FEATURE_SIZE, RSEQ_FEATURE_SIZE);
24. auxv.push(AT_RSEQ_ALIGN, RSEQ_ALIGN);
25. #[cfg(target_arch = "x86_64")] auxv.push(AT_SYSINFO_EHDR, vdso_base());
26. auxv.push(AT_NULL, 0);
27. copy_to_user_stack(bprm.stack_ptr, &auxv)

### Out of Scope

- `fs/binfmt_elf.c` ELF loader implementation — covered in `fs/binfmt_elf.md` Tier-3
- `fs/binfmt_elf_fdpic.c` FDPIC variant — covered in `fs/binfmt_elf_fdpic.md` Tier-3
- `fs/coredump.c` coredump pipeline — covered in `fs/coredump.md` Tier-3
- `kernel/module/main.c` module loader's ELF parser — covered in `kernel/module.md` Tier-3
- `kernel/kexec_file.c` kexec ELF image loading — covered in `kernel/kexec.md` Tier-3
- `kernel/livepatch/` SHN_LIVEPATCH handling — covered in `kernel/livepatch.md` Tier-3
- Per-arch `arch/<arch>/include/uapi/asm/auxvec.h` (AT_SYSINFO_EHDR, AT_L1I_CACHESIZE, AT_FPUCW, AT_DCACHEBSIZE, etc.) — covered per-arch
- glibc / musl dynamic linker — out of UAPI scope
- BPF object loading via libbpf — covered in `uapi/headers/bpf.md` Tier-5
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `Elf32_Addr` / `Elf64_Addr` / `_Half` / `_Off` / `_Word` / `_Xword` / … | per-base typedef | `Elf32Addr` / `Elf64Addr` / … |
| `struct elf32_hdr` / `struct elf64_hdr` (`Elf{32,64}_Ehdr`) | per-ELF file header | `Elf32Hdr` / `Elf64Hdr` |
| `struct elf32_phdr` / `struct elf64_phdr` (`Elf{32,64}_Phdr`) | per-program header | `Elf32Phdr` / `Elf64Phdr` |
| `struct elf32_shdr` / `struct elf64_shdr` (`Elf{32,64}_Shdr`) | per-section header | `Elf32Shdr` / `Elf64Shdr` |
| `struct elf32_sym` / `struct elf64_sym` (`Elf{32,64}_Sym`) | per-symbol table entry | `Elf32Sym` / `Elf64Sym` |
| `struct elf32_rel/_rela` / `struct elf64_rel/_rela` | per-relocation | `Elf{32,64}Rel` / `Elf{32,64}Rela` |
| `Elf{32,64}_Dyn` | per-dynamic-table entry | `Elf{32,64}Dyn` |
| `struct elf32_note` / `struct elf64_note` (`Elf{32,64}_Nhdr`) | per-PT_NOTE entry | `Elf{32,64}Note` |
| `EI_NIDENT` / `EI_MAG0..3` / `EI_CLASS` / `EI_DATA` / `EI_VERSION` / `EI_OSABI` / `EI_PAD` | per-e_ident index | shared |
| `ELFMAG0..3` / `ELFMAG` / `SELFMAG` | per-magic | shared |
| `ELFCLASS{NONE,32,64,NUM}` | per-class | shared |
| `ELFDATA{NONE,2LSB,2MSB}` | per-endian | shared |
| `EV_{NONE,CURRENT,NUM}` | per-version | shared |
| `ELFOSABI_{NONE,LINUX}` | per-osabi | shared |
| `ET_{NONE,REL,EXEC,DYN,CORE,LOPROC,HIPROC}` | per-e_type | shared |
| `EM_*` table | per-e_machine | shared |
| `PT_{NULL,LOAD,DYNAMIC,INTERP,NOTE,SHLIB,PHDR,TLS}` / `PT_{LO,HI}{OS,PROC}` | per-p_type | shared |
| `PT_GNU_{EH_FRAME,STACK,RELRO,PROPERTY}` | per-GNU p_type | shared |
| `PN_XNUM` | per-extended-phnum sentinel | shared |
| `PF_{R,W,X}` | per-segment flag | shared |
| `SHT_*` / `SHF_*` / `SHN_*` | per-section | shared |
| `STB_*` / `STT_*` / `ELF_ST_BIND` / `ELF_ST_TYPE` | per-symbol bind/type | shared |
| `DT_*` | per-dyn-tag | shared |
| `NT_*` | per-core-note type | shared |
| `NN_*` | per-core-note name | shared |
| `AT_*` | per-auxv entry | shared |
| `VER_FLG_BASE` / `VER_FLG_WEAK` | per-version flag | shared |
| `GNU_PROPERTY_AARCH64_FEATURE_1_BTI` | per-arm64 BTI feature bit | shared |

### abi surface (constants + structs)

### `e_ident` layout (`uapi/linux/elf.h:352-367, 215`)

```text
EI_NIDENT = 16     /* e_ident[] length */

/* indices into e_ident[] */
EI_MAG0    = 0   /* 0x7F */
EI_MAG1    = 1   /* 'E' */
EI_MAG2    = 2   /* 'L' */
EI_MAG3    = 3   /* 'F' */
EI_CLASS   = 4   /* ELFCLASS{NONE,32,64} */
EI_DATA    = 5   /* ELFDATA{NONE,2LSB,2MSB} */
EI_VERSION = 6   /* EV_CURRENT */
EI_OSABI   = 7   /* ELFOSABI_{NONE,LINUX,...} */
EI_PAD     = 8   /* start of padding (e_ident[8..16] reserved 0) */

ELFMAG0    = 0x7F
ELFMAG1    = 'E'
ELFMAG2    = 'L'
ELFMAG3    = 'F'
ELFMAG     = "\177ELF"
SELFMAG    = 4
```

### Class / data / version / osabi (`uapi/linux/elf.h:369-387`)

```text
ELFCLASSNONE = 0
ELFCLASS32   = 1
ELFCLASS64   = 2
ELFCLASSNUM  = 3

ELFDATANONE  = 0
ELFDATA2LSB  = 1    /* little-endian */
ELFDATA2MSB  = 2    /* big-endian */

EV_NONE      = 0
EV_CURRENT   = 1
EV_NUM       = 2

ELFOSABI_NONE  = 0
ELFOSABI_LINUX = 3
```

### `e_type` (`uapi/linux/elf.h:72-78`)

```text
ET_NONE   = 0
ET_REL    = 1       /* relocatable object (.o) */
ET_EXEC   = 2       /* fixed-load-address executable */
ET_DYN    = 3       /* shared object OR PIE */
ET_CORE   = 4       /* core dump */
ET_LOPROC = 0xff00
ET_HIPROC = 0xffff
```

`ET_DYN` is the modern default for executables under ASLR — kernel allocates a randomized base for `PT_LOAD` segments. `ET_EXEC` is pinned at file-specified `p_vaddr`.

### `e_machine` (selected — `uapi/linux/elf-em.h`)

```text
EM_NONE         =   0
EM_M32          =   1     /* WE 32100 */
EM_SPARC        =   2
EM_386          =   3     /* Intel 80386 */
EM_68K          =   4
EM_88K          =   5
EM_486          =   6
EM_860          =   7
EM_MIPS         =   8
EM_MIPS_RS3_LE  =  10
EM_MIPS_RS4_BE  =  10
EM_PARISC       =  15
EM_SPARC32PLUS  =  18
EM_PPC          =  20
EM_PPC64        =  21
EM_S390         =  22
EM_SPU          =  23
EM_ARM          =  40
EM_SH           =  42
EM_SPARCV9      =  43
EM_H8_300       =  46
EM_IA_64        =  50
EM_X86_64       =  62
EM_CRIS         =  76
EM_M32R         =  88
EM_MN10300      =  89
EM_OPENRISC     =  92
EM_ARCOMPACT    =  93
EM_XTENSA       =  94
EM_BLACKFIN     = 106
EM_UNICORE      = 110
EM_ALTERA_NIOS2 = 113
EM_TI_C6000     = 140
EM_HEXAGON      = 164
EM_NDS32        = 167
EM_AARCH64      = 183
EM_TILEPRO      = 188
EM_MICROBLAZE   = 189
EM_TILEGX       = 191
EM_ARCV2        = 195
EM_RISCV        = 243
EM_BPF          = 247
EM_CSKY         = 252
EM_LOONGARCH    = 258
EM_FRV          = 0x5441
EM_ALPHA        = 0x9026
```

### `struct elf{32,64}_hdr` (`uapi/linux/elf.h:217-249`)

```text
typedef struct elf64_hdr {
    unsigned char e_ident[EI_NIDENT];   /* 16 bytes magic + class + data + ver + osabi */
    Elf64_Half  e_type;                  /* ET_* */
    Elf64_Half  e_machine;               /* EM_* */
    Elf64_Word  e_version;               /* EV_CURRENT */
    Elf64_Addr  e_entry;                 /* entry VA */
    Elf64_Off   e_phoff;                 /* program header table file offset */
    Elf64_Off   e_shoff;                 /* section header table file offset */
    Elf64_Word  e_flags;                 /* arch-specific */
    Elf64_Half  e_ehsize;                /* this header's size = sizeof(Elf64_Ehdr) = 64 */
    Elf64_Half  e_phentsize;             /* sizeof(Elf64_Phdr) = 56 */
    Elf64_Half  e_phnum;                 /* count of program headers (or PN_XNUM) */
    Elf64_Half  e_shentsize;             /* sizeof(Elf64_Shdr) = 64 */
    Elf64_Half  e_shnum;                 /* count of section headers */
    Elf64_Half  e_shstrndx;              /* shstrtab section index */
} Elf64_Ehdr;
```

`sizeof(Elf32_Ehdr) == 52`, `sizeof(Elf64_Ehdr) == 64`. `e_phnum == PN_XNUM (0xffff)` signals that the real count is in `Shdr[0].sh_info`.

### `p_type` (`uapi/linux/elf.h:28-47`)

```text
PT_NULL    = 0
PT_LOAD    = 1
PT_DYNAMIC = 2
PT_INTERP  = 3       /* path to dynamic linker */
PT_NOTE    = 4
PT_SHLIB   = 5       /* reserved, unused */
PT_PHDR    = 6
PT_TLS     = 7

PT_LOOS    = 0x60000000
PT_HIOS    = 0x6fffffff
PT_LOPROC  = 0x70000000
PT_HIPROC  = 0x7fffffff

/* GNU extensions */
PT_GNU_EH_FRAME = 0x6474e550   /* .eh_frame_hdr — unwind table */
PT_GNU_STACK    = 0x6474e551   /* p_flags determines stack X (W^X policy) */
PT_GNU_RELRO    = 0x6474e552   /* read-only-after-relocation region */
PT_GNU_PROPERTY = 0x6474e553   /* CET / BTI / shadow-stack property notes */

/* arch-specific */
PT_AARCH64_MEMTAG_MTE = 0x70000002   /* PROT_MTE region (ARM MTE) */

PN_XNUM    = 0xffff               /* sentinel for extended-phnum */
```

### Program header (`uapi/linux/elf.h:253-277`)

```text
PF_X = 0x1   /* execute */
PF_W = 0x2   /* write */
PF_R = 0x4   /* read */

typedef struct elf64_phdr {
    Elf64_Word  p_type;     /* PT_* */
    Elf64_Word  p_flags;    /* PF_R | PF_W | PF_X */
    Elf64_Off   p_offset;   /* file offset */
    Elf64_Addr  p_vaddr;    /* virtual address */
    Elf64_Addr  p_paddr;    /* physical address (unused on most targets) */
    Elf64_Xword p_filesz;   /* file size */
    Elf64_Xword p_memsz;    /* in-memory size (>= p_filesz; tail is .bss) */
    Elf64_Xword p_align;    /* alignment (p_offset mod p_align == p_vaddr mod p_align) */
} Elf64_Phdr;

typedef struct elf32_phdr {
    Elf32_Word  p_type;
    Elf32_Off   p_offset;
    Elf32_Addr  p_vaddr;
    Elf32_Addr  p_paddr;
    Elf32_Word  p_filesz;
    Elf32_Word  p_memsz;
    Elf32_Word  p_flags;
    Elf32_Word  p_align;
} Elf32_Phdr;
```

Note the 32-bit and 64-bit `Phdr` orderings differ: 64-bit has `p_flags` immediately after `p_type` (for natural alignment of the 8-byte offset/addr fields); 32-bit has `p_flags` near the end.

### Section header (`uapi/linux/elf.h:326-350`)

```text
typedef struct elf64_shdr {
    Elf64_Word  sh_name;        /* index into shstrtab */
    Elf64_Word  sh_type;        /* SHT_* */
    Elf64_Xword sh_flags;       /* SHF_* */
    Elf64_Addr  sh_addr;        /* address at execution */
    Elf64_Off   sh_offset;      /* file offset */
    Elf64_Xword sh_size;        /* section size */
    Elf64_Word  sh_link;        /* index of another section */
    Elf64_Word  sh_info;        /* additional info */
    Elf64_Xword sh_addralign;   /* alignment */
    Elf64_Xword sh_entsize;     /* table entry size if applicable */
} Elf64_Shdr;
```

### `sh_type` and `sh_flags` (`uapi/linux/elf.h:279-314`)

```text
SHT_NULL    =  0
SHT_PROGBITS=  1
SHT_SYMTAB  =  2
SHT_STRTAB  =  3
SHT_RELA    =  4
SHT_HASH    =  5
SHT_DYNAMIC =  6
SHT_NOTE    =  7
SHT_NOBITS  =  8       /* .bss-style */
SHT_REL     =  9
SHT_SHLIB   = 10       /* reserved */
SHT_DYNSYM  = 11
SHT_NUM     = 12
SHT_LOPROC  = 0x70000000
SHT_HIPROC  = 0x7fffffff
SHT_LOUSER  = 0x80000000
SHT_HIUSER  = 0xffffffff

SHF_WRITE             = 0x1
SHF_ALLOC             = 0x2
SHF_EXECINSTR         = 0x4
SHF_MERGE             = 0x10
SHF_STRINGS           = 0x20
SHF_INFO_LINK         = 0x40
SHF_LINK_ORDER        = 0x80
SHF_OS_NONCONFORMING  = 0x100
SHF_GROUP             = 0x200
SHF_TLS               = 0x400
SHF_RELA_LIVEPATCH    = 0x00100000
SHF_RO_AFTER_INIT     = 0x00200000
SHF_ORDERED           = 0x04000000
SHF_EXCLUDE           = 0x08000000
SHF_MASKOS            = 0x0ff00000
SHF_MASKPROC          = 0xf0000000
```

### Special section indices (`uapi/linux/elf.h:317-324`)

```text
SHN_UNDEF      = 0
SHN_LORESERVE  = 0xff00
SHN_LOPROC     = 0xff00
SHN_HIPROC     = 0xff1f
SHN_LIVEPATCH  = 0xff20
SHN_ABS        = 0xfff1
SHN_COMMON     = 0xfff2
SHN_HIRESERVE  = 0xffff
```

### Symbol table (`uapi/linux/elf.h:127-149, 196-212`)

```text
STB_LOCAL  = 0
STB_GLOBAL = 1
STB_WEAK   = 2

STN_UNDEF = 0

STT_NOTYPE  = 0
STT_OBJECT  = 1
STT_FUNC    = 2
STT_SECTION = 3
STT_FILE    = 4
STT_COMMON  = 5
STT_TLS     = 6

#define ELF_ST_BIND(x)   ((x) >> 4)
#define ELF_ST_TYPE(x)   ((x) & 0xf)
#define ELF32_ST_BIND(x) ELF_ST_BIND(x)
#define ELF32_ST_TYPE(x) ELF_ST_TYPE(x)
#define ELF64_ST_BIND(x) ELF_ST_BIND(x)
#define ELF64_ST_TYPE(x) ELF_ST_TYPE(x)

typedef struct elf64_sym {
    Elf64_Word    st_name;     /* string-table index */
    unsigned char st_info;     /* bind << 4 | type */
    unsigned char st_other;    /* visibility */
    Elf64_Half    st_shndx;    /* section index or SHN_* */
    Elf64_Addr    st_value;
    Elf64_Xword   st_size;
} Elf64_Sym;
```

32-bit `Elf32_Sym` has the same fields but a different order (`st_name, st_value, st_size, st_info, st_other, st_shndx`).

### Relocations (`uapi/linux/elf.h:168-194`)

```text
#define ELF32_R_SYM(x)   ((x) >> 8)
#define ELF32_R_TYPE(x)  ((x) & 0xff)

#define ELF64_R_SYM(i)   ((i) >> 32)
#define ELF64_R_TYPE(i)  ((i) & 0xffffffff)

typedef struct elf64_rel  { Elf64_Addr r_offset; Elf64_Xword r_info; }                  Elf64_Rel;
typedef struct elf64_rela { Elf64_Addr r_offset; Elf64_Xword r_info; Elf64_Sxword r_addend; } Elf64_Rela;
```

### Dynamic table (`uapi/linux/elf.h:81-125, 151-166`)

```text
DT_NULL=0  DT_NEEDED=1  DT_PLTRELSZ=2  DT_PLTGOT=3
DT_HASH=4  DT_STRTAB=5  DT_SYMTAB=6   DT_RELA=7
DT_RELASZ=8 DT_RELAENT=9 DT_STRSZ=10  DT_SYMENT=11
DT_INIT=12 DT_FINI=13   DT_SONAME=14  DT_RPATH=15
DT_SYMBOLIC=16  DT_REL=17  DT_RELSZ=18  DT_RELENT=19
DT_PLTREL=20    DT_DEBUG=21  DT_TEXTREL=22 DT_JMPREL=23
DT_ENCODING=32

OLD_DT_LOOS=0x60000000
DT_LOOS=0x6000000d         DT_HIOS=0x6ffff000
DT_VALRNGLO=0x6ffffd00     DT_VALRNGHI=0x6ffffdff
DT_ADDRRNGLO=0x6ffffe00    DT_GNU_HASH=0x6ffffef5
DT_ADDRRNGHI=0x6ffffeff
DT_VERSYM=0x6ffffff0       DT_RELACOUNT=0x6ffffff9
DT_RELCOUNT=0x6ffffffa     DT_FLAGS_1=0x6ffffffb
DT_VERDEF=0x6ffffffc       DT_VERDEFNUM=0x6ffffffd
DT_VERNEED=0x6ffffffe      DT_VERNEEDNUM=0x6fffffff
OLD_DT_HIOS=0x6fffffff

DT_LOPROC=0x70000000  DT_HIPROC=0x7fffffff

typedef struct {
    Elf64_Sxword d_tag;
    union {
        Elf64_Xword d_val;
        Elf64_Addr  d_ptr;
    } d_un;
} Elf64_Dyn;
```

### Notes (`uapi/linux/elf.h:389-577`)

```text
typedef struct elf64_note {
    Elf64_Word n_namesz;     /* including trailing NUL */
    Elf64_Word n_descsz;
    Elf64_Word n_type;
} Elf64_Nhdr;

/* PT_NOTE/SHT_NOTE descriptor layout:
 *   Elf64_Nhdr | name (padded to 4) | desc (padded to 4)
 *
 * NN_<TYPE> is the canonical name string for note type NT_<TYPE>.
 */

/* CORE notes (process / core dumps) */
NN_PRSTATUS    = "CORE"   NT_PRSTATUS   = 1     /* struct elf_prstatus_common (regs + signal) */
NN_PRFPREG     = "CORE"   NT_PRFPREG    = 2     /* fpregs */
NN_PRPSINFO    = "CORE"   NT_PRPSINFO   = 3     /* struct elf_prpsinfo (cmdline + state) */
NN_TASKSTRUCT  = "CORE"   NT_TASKSTRUCT = 4
NN_AUXV        = "CORE"   NT_AUXV       = 6     /* auxiliary vector blob */
NN_SIGINFO     = "CORE"   NT_SIGINFO    = 0x53494749   /* "SIGI" */
NN_FILE        = "CORE"   NT_FILE       = 0x46494c45   /* "FILE" */

/* LINUX notes (arch register sets — ptrace(PTRACE_GETREGSET, NT_*, ...)) */
NN_PRXFPREG    = "LINUX"  NT_PRXFPREG     = 0x46e62b7f   /* i387 extended */
NN_X86_XSTATE  = "LINUX"  NT_X86_XSTATE   = 0x202        /* XSAVE area */
NN_X86_SHSTK   = "LINUX"  NT_X86_SHSTK    = 0x204        /* CET shadow stack */
NN_X86_XSAVE_LAYOUT = "LINUX" NT_X86_XSAVE_LAYOUT = 0x205

NN_ARM_VFP             = "LINUX"  NT_ARM_VFP             = 0x400
NN_ARM_TLS             = "LINUX"  NT_ARM_TLS             = 0x401
NN_ARM_SVE             = "LINUX"  NT_ARM_SVE             = 0x405
NN_ARM_PAC_MASK        = "LINUX"  NT_ARM_PAC_MASK        = 0x406
NN_ARM_TAGGED_ADDR_CTRL= "LINUX"  NT_ARM_TAGGED_ADDR_CTRL= 0x409
NN_ARM_SSVE            = "LINUX"  NT_ARM_SSVE            = 0x40b
NN_ARM_ZA              = "LINUX"  NT_ARM_ZA              = 0x40c
NN_ARM_GCS             = "LINUX"  NT_ARM_GCS             = 0x410

NN_RISCV_CSR           = "LINUX"  NT_RISCV_CSR           = 0x900
NN_RISCV_VECTOR        = "LINUX"  NT_RISCV_VECTOR        = 0x901
NN_RISCV_USER_CFI      = "LINUX"  NT_RISCV_USER_CFI      = 0x903

/* GNU property note (CET, BTI, shadow stack, etc.) */
NN_GNU_PROPERTY_TYPE_0 = "GNU"
NT_GNU_PROPERTY_TYPE_0 = 5
```

(PPC, S390, MIPS, LoongArch and ARM-extension NTs follow the same `NN_/NT_` convention — see `uapi/linux/elf.h:419-563`.)

GNU property bits for AArch64:

```text
GNU_PROPERTY_AARCH64_FEATURE_1_AND = 0xc0000000   /* note type within GNU property */
GNU_PROPERTY_AARCH64_FEATURE_1_BTI = (1U << 0)    /* program uses BTI */
```

### Auxiliary vector (`uapi/linux/auxvec.h`)

```text
AT_NULL              =  0    /* terminator */
AT_IGNORE            =  1
AT_EXECFD            =  2    /* fd for kernel-script interpreter loading */
AT_PHDR              =  3    /* program headers address */
AT_PHENT             =  4    /* phdr entry size */
AT_PHNUM             =  5    /* phdr count */
AT_PAGESZ            =  6    /* system page size */
AT_BASE              =  7    /* interpreter (ld.so) base address */
AT_FLAGS             =  8
AT_ENTRY             =  9    /* program entry point */
AT_NOTELF            = 10
AT_UID               = 11
AT_EUID              = 12
AT_GID               = 13
AT_EGID              = 14
AT_PLATFORM          = 15    /* arch string (e.g., "x86_64") */
AT_HWCAP             = 16    /* arch CPU feature bits */
AT_CLKTCK            = 17    /* sysconf(_SC_CLK_TCK) */
/* 18-22 reserved */
AT_SECURE            = 23    /* AT_SECURE != 0 ⟺ setuid / setgid / capability transition */
AT_BASE_PLATFORM     = 24    /* may differ from AT_PLATFORM (e.g., compatibility) */
AT_RANDOM            = 25    /* 16-byte random pool for stack-canary / SSP */
AT_HWCAP2            = 26
AT_RSEQ_FEATURE_SIZE = 27
AT_RSEQ_ALIGN        = 28
AT_HWCAP3            = 29
AT_HWCAP4            = 30
AT_EXECFN            = 31    /* string: original argv[0] path */
AT_MINSIGSTKSZ       = 51    /* generic, per-arch may override */
```

`Documentation/admin-guide/cpu.rst` arch-specific aux entries (`AT_L1I_*_GEOMETRY`, `AT_L1D_*`, `AT_L2_*`, `AT_L3_*`, `AT_DCACHEBSIZE`, `AT_UCACHEBSIZE`, `AT_ICACHEBSIZE`, `AT_FPUCW`, `AT_IGNOREPPC`, `AT_SYSINFO`, `AT_SYSINFO_EHDR`) are defined per arch in `arch/<arch>/include/uapi/asm/auxvec.h` and are pulled in by the `#include <asm/auxvec.h>` at the top of the generic header.

### Versioning (`uapi/linux/elf.h:141-142, 585-613`)

```text
VER_FLG_BASE = 0x1
VER_FLG_WEAK = 0x2

typedef struct {
    Elf64_Half  vd_version;
    Elf64_Half  vd_flags;
    Elf64_Half  vd_ndx;
    Elf64_Half  vd_cnt;
    Elf64_Word  vd_hash;
    Elf64_Word  vd_aux;
    Elf64_Word  vd_next;
} Elf64_Verdef;

typedef struct {
    Elf64_Word vda_name;
    Elf64_Word vda_next;
} Elf64_Verdaux;
```

### compatibility contract

REQ-1: Every ELF file begins with `e_ident[EI_MAG0..3] == {0x7f, 'E', 'L', 'F'}`. Loader MUST reject anything else with `-ENOEXEC`.

REQ-2: `e_ident[EI_CLASS]` MUST be `ELFCLASS32` or `ELFCLASS64`. `ELFCLASSNONE` MUST be rejected with `-ENOEXEC`.

REQ-3: `e_ident[EI_DATA]` MUST be `ELFDATA2LSB` (little-endian) or `ELFDATA2MSB`. The loader MUST honor the encoded endianness when parsing fields; a host-endian/file-endian mismatch is acceptable only if the kernel performs runtime byte-swap for ELF parsing on cross-endian boots (rare; usually rejected).

REQ-4: `e_ident[EI_VERSION]` MUST equal `EV_CURRENT == 1`. `e_version` field MUST also equal `EV_CURRENT`.

REQ-5: `e_ident[EI_OSABI]` MUST be `ELFOSABI_NONE` (0) or `ELFOSABI_LINUX` (3). Other values MAY load but lose Linux-specific features (e.g., GNU_STACK).

REQ-6: `e_type` MUST be `ET_EXEC`, `ET_DYN`, or `ET_CORE`. `ET_REL` is loaded only by the module loader / livepatch; not by `execve(2)`.

REQ-7: `sizeof(Elf32_Ehdr) == 52`, `sizeof(Elf64_Ehdr) == 64`. `e_ehsize` MUST equal one of those.

REQ-8: `e_phentsize` MUST equal `sizeof(Elf{32,64}_Phdr)` (32 or 56). `e_shentsize` MUST equal `sizeof(Elf{32,64}_Shdr)` (40 or 64). Mismatch ⟹ `-ENOEXEC`.

REQ-9: `e_phnum == PN_XNUM (0xffff)` signals that the true count is in section header 0's `sh_info`.

REQ-10: For `ET_DYN` (PIE), the kernel MUST allocate a randomized base address (subject to ASLR policy and `personality(ADDR_NO_RANDOMIZE)` opt-out) and add it to every `PT_LOAD.p_vaddr`.

REQ-11: For `ET_EXEC`, the kernel MUST map `PT_LOAD` segments at their literal `p_vaddr` (or refuse if a conflict exists).

REQ-12: `PT_LOAD` is the only program header that consumes memory. `p_filesz <= p_memsz`; the tail `p_memsz - p_filesz` MUST be zero-filled (.bss).

REQ-13: `PT_INTERP` (if present) names the dynamic linker. Kernel MUST execve-recursively load the interp binary; max nesting depth ≤ 4.

REQ-14: `PT_TLS` describes the thread-local-storage initialization image. The dynamic linker (not the kernel) processes it.

REQ-15: `PT_GNU_STACK.p_flags` controls whether the stack VMA gets `PROT_EXEC`. If `PT_GNU_STACK` is absent, the kernel uses the per-arch default (modern arches: NX stack). If present with `PF_X` cleared, stack MUST NOT be executable.

REQ-16: `PT_GNU_RELRO` covers the region the dynamic linker will mprotect to `PROT_READ` after relocation processing. Required for full RELRO.

REQ-17: `PT_GNU_PROPERTY` carries `NT_GNU_PROPERTY_TYPE_0` notes that may enable arch-specific protections (CET IBT/SHSTK on x86, BTI/PAC on AArch64, branch-protection on RISC-V).

REQ-18: `PT_GNU_EH_FRAME` is a pointer to the `.eh_frame_hdr` lookup table for unwind.

REQ-19: `PF_R = 4`, `PF_W = 2`, `PF_X = 1`. A `PT_LOAD` segment with both `PF_W` and `PF_X` set is `W^X` violation; modern kernels MAY refuse to load such segments (depending on PaX/policy).

REQ-20: `Elf64_Phdr` field order: `p_type, p_flags, p_offset, p_vaddr, p_paddr, p_filesz, p_memsz, p_align` (56 bytes). `Elf32_Phdr` field order: `p_type, p_offset, p_vaddr, p_paddr, p_filesz, p_memsz, p_flags, p_align` (32 bytes).

REQ-21: `Elf64_Shdr` is 64 bytes; `Elf32_Shdr` is 40 bytes. Order: `sh_name, sh_type, sh_flags, sh_addr, sh_offset, sh_size, sh_link, sh_info, sh_addralign, sh_entsize`.

REQ-22: `SHN_LIVEPATCH (0xff20)` references a symbol fixed-up by the livepatch loader at module-attach time.

REQ-23: `SHF_RO_AFTER_INIT` instructs the kernel module loader to make the section `PROT_READ` after the module's `init` function returns.

REQ-24: `Elf{32,64}_Sym.st_info` packs binding (high 4 bits) and type (low 4 bits): `ELF_ST_BIND(x) = x >> 4`, `ELF_ST_TYPE(x) = x & 0xf`.

REQ-25: `Elf64_Rel.r_info` = `(sym_index << 32) | reloc_type`. `Elf32_Rel.r_info` = `(sym_index << 8) | reloc_type`.

REQ-26: `Elf{32,64}_Nhdr` is `{ n_namesz, n_descsz, n_type }`. Following the header: name padded to 4-byte boundary, then desc padded to 4-byte boundary.

REQ-27: For `NT_PRSTATUS` (1), the kernel emits per-thread register set in a core dump. For ptrace, `PTRACE_GETREGSET` with `NT_PRSTATUS` returns the same register-set encoding.

REQ-28: `NT_PRPSINFO` (3) carries process metadata (`pr_state`, `pr_sname`, `pr_zomb`, `pr_nice`, `pr_flag`, `pr_uid`, `pr_gid`, `pr_pid`, `pr_ppid`, `pr_pgrp`, `pr_sid`, `pr_fname[16]`, `pr_psargs[80]`).

REQ-29: `NT_AUXV` (6) embeds the original `auxv` blob in core dumps so debuggers can recover `AT_RANDOM`, `AT_ENTRY`, `AT_BASE`, etc.

REQ-30: `NT_SIGINFO` (0x53494749 = "SIGI") carries the `siginfo_t` of the signal that caused the core dump. Size may grow in future kernels.

REQ-31: `NT_FILE` (0x46494c45 = "FILE") describes mapped files in the core dump.

REQ-32: `NT_X86_XSTATE` (0x202) is the XSAVE area; size depends on enabled feature bits in XCR0.

REQ-33: `NT_X86_SHSTK` (0x204) is the CET shadow stack pointer state.

REQ-34: `AT_NULL == 0` terminates the auxv on stack. `AT_IGNORE == 1` entries MUST be skipped.

REQ-35: `AT_PHDR` / `AT_PHENT` / `AT_PHNUM` MUST give the program-header table location, entry size, and count of the *executable* (not the interpreter).

REQ-36: `AT_PAGESZ` MUST equal `PAGE_SIZE` of the running kernel (4096 on x86_64 default; per arch).

REQ-37: `AT_BASE` MUST give the load address of `PT_INTERP` (the dynamic linker base). For static executables, this is 0.

REQ-38: `AT_ENTRY` MUST give the program's entry-point VA (`e_entry + load_bias` for ET_DYN).

REQ-39: `AT_UID` / `AT_EUID` / `AT_GID` / `AT_EGID` carry the pre-exec credentials. `AT_SECURE != 0` ⟺ the exec performed a setuid/setgid/capability transition; libc MUST ignore environment-driven library-search-path expansions in this case.

REQ-40: `AT_PLATFORM` / `AT_BASE_PLATFORM` are pointers to NUL-terminated strings naming the CPU subtype.

REQ-41: `AT_HWCAP` / `AT_HWCAP2` / `AT_HWCAP3` / `AT_HWCAP4` are arch-specific bitmasks of CPU feature presence (e.g., on x86 `AT_HWCAP` mirrors CPUID bits; on arm64, `AT_HWCAP` covers `ID_AA64*` registers).

REQ-42: `AT_CLKTCK` MUST equal `USER_HZ` (typically 100).

REQ-43: `AT_RANDOM` MUST point to a 16-byte buffer of kernel-supplied randomness. libc uses bytes 8-15 for `__stack_chk_guard` (SSP canary) and 0-7 for `__pointer_chk_guard`.

REQ-44: `AT_EXECFN` MUST point to the NUL-terminated original `argv[0]` (the executable's path as passed to `execve`).

REQ-45: `AT_RSEQ_FEATURE_SIZE` / `AT_RSEQ_ALIGN` indicate the kernel-supported size and required alignment of the `rseq` ABI struct.

REQ-46: `AT_SYSINFO` (per arch) / `AT_SYSINFO_EHDR` (per arch) point to the vDSO ELF base; userspace dynamic linker MUST load it.

REQ-47: `AT_MINSIGSTKSZ` (51) supersedes the static `MINSIGSTKSZ` constant on SVE-capable arm64 systems where vector length dynamically grows the signal frame.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ehdr_magic_required` | INVARIANT | per-load: `e_ident[0..4] == "\x7fELF"` else `ENOEXEC`. |
| `eclass_supported` | INVARIANT | per-load: `EI_CLASS ∈ {ELFCLASS32, ELFCLASS64}`. |
| `edata_supported` | INVARIANT | per-load: `EI_DATA ∈ {ELFDATA2LSB, ELFDATA2MSB}`. |
| `e_type_supported_for_execve` | INVARIANT | per-execve: `e_type ∈ {ET_EXEC, ET_DYN}`. |
| `e_phentsize_exact` | INVARIANT | per-load: `e_phentsize == sizeof(Elf{32,64}_Phdr)`. |
| `e_shentsize_exact` | INVARIANT | per-load: `e_shentsize == sizeof(Elf{32,64}_Shdr)`. |
| `phdr_load_filesz_le_memsz` | INVARIANT | per-PT_LOAD: `p_filesz <= p_memsz`. |
| `phdr_load_no_wx` | INVARIANT | per-PT_LOAD: NOT (PF_W ∧ PF_X) under W^X policy. |
| `pn_xnum_resolved` | INVARIANT | per-load: `e_phnum == PN_XNUM ⟹ true count from Shdr[0].sh_info`. |
| `interp_depth_bounded` | INVARIANT | per-load: PT_INTERP recursion ≤ 4 else `-ELOOP`. |
| `gnu_stack_nx_default` | INVARIANT | per-load: PT_GNU_STACK absent ⟹ stack non-executable on modern arch. |
| `gnu_relro_within_load` | INVARIANT | per-load: PT_GNU_RELRO range ⊆ PT_LOAD range. |
| `auxv_terminator` | INVARIANT | per-auxv: last entry is `(AT_NULL, 0)`. |
| `at_random_16_bytes` | INVARIANT | per-auxv: `AT_RANDOM` points to exactly 16 bytes of kernel-supplied entropy. |
| `at_secure_setuid_implies` | INVARIANT | per-auxv: setuid/setgid/cap-transition exec ⟹ `AT_SECURE != 0`. |
| `note_alignment` | INVARIANT | per-note: name and desc each 4-byte aligned. |
| `gnu_property_aarch64_bti_bit` | INVARIANT | `GNU_PROPERTY_AARCH64_FEATURE_1_BTI == 0x1`. |

### Layer 2: TLA+

`uapi/headers/elf.tla`:
- Per-`execve` ELF parse → validate → map PT_LOAD → apply GNU_STACK/RELRO/PROPERTY → push auxv → start_thread.
- Per-`coredump` walk threads → emit NT_PRSTATUS/PRFPREG/PRPSINFO/SIGINFO/AUXV/FILE notes.
- Per-`ptrace(PTRACE_GETREGSET, NT_*, ...)` per-thread regset read.
- Properties:
  - `safety_magic_required_before_map` — no PT_LOAD is mapped if `e_ident` is invalid.
  - `safety_pie_load_bias_random` — per-`ET_DYN` exec under ASLR: two runs produce distinct load biases.
  - `safety_nx_stack_default` — per-modern-arch: stack VMA is non-executable absent explicit PF_X PT_GNU_STACK.
  - `safety_relro_post_init` — per-PT_GNU_RELRO: region reaches `PROT_READ` before any user instruction executes after relocation.
  - `safety_at_secure_excludes_path_envs` — per-setuid exec: libc honors `AT_SECURE != 0` by suppressing `LD_*` env.
  - `safety_at_random_per_exec_unique` — per-exec: `AT_RANDOM` blob differs.
  - `liveness_load_terminates` — per-execve: PT_LOAD count is finite (≤ MAX_PHDRS) and termination guaranteed.
  - `liveness_interp_chain_terminates` — per-PT_INTERP chain: bounded by depth 4.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Elf64Hdr` layout: 64 bytes; field order matches upstream | `Elf64Hdr` |
| `Elf64Phdr` layout: 56 bytes; field order distinct from `Elf32Phdr` | `Elf64Phdr` |
| `Elf64Shdr` layout: 64 bytes | `Elf64Shdr` |
| `Elf64Sym` layout: 24 bytes; bind/type packed in `st_info` | `Elf64Sym` |
| `Elf64Nhdr` layout: 12 bytes; descsz follows namesz | `Elf64Nhdr` |
| `load_binary` post: every PT_LOAD mapped at `load_bias + p_vaddr` with correct prot | `BinfmtElf::load_binary` |
| `push_auxv` post: auxv terminated by AT_NULL; AT_RANDOM is 16B page-aligned blob | `BinfmtElf::push_auxv` |
| ptrace regset NT_PRSTATUS/NT_PRFPREG/NT_X86_XSTATE shape matches per-arch | `Ptrace::get_regset` |
| `Elf64Rel` / `Elf64Rela` `r_info` packing: SYM >> 32, TYPE & 0xFFFFFFFF | `Elf64Rel*` |

### Layer 4: Verus/Creusot functional

`Per execve(2) → binfmt_elf::load_binary → map PT_LOAD → apply GNU_* policies → push auxv → start_thread` semantic equivalence: per-`fs/binfmt_elf.c` upstream, per-`Documentation/admin-guide/abi-stable.rst`, per-`/proc/<pid>/auxv` man page. `Per coredump emit NT_PRSTATUS / NT_PRPSINFO / NT_AUXV / NT_SIGINFO / NT_FILE` semantic equivalence: per-`fs/coredump.c` + per-`gdb`-readable layout. `Per ptrace(PTRACE_GETREGSET, NT_*, &iov)` semantic equivalence: per-`arch/<arch>/kernel/ptrace.c` regset table.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

ELF UAPI reinforcement:

- **Magic + EI_CLASS + EI_DATA + EI_VERSION + EI_OSABI all validated before any PT_LOAD mapping** — defense against per-truncated / per-malformed ELF triggering allocator/parser state.
- **`e_ehsize`, `e_phentsize`, `e_shentsize` cross-checked against `sizeof(struct)`** — defense against per-forged-header-size that would walk past end of buffer.
- **`e_phnum` ≤ MAX_PHDRS (typically 256)** — defense against per-resource-exhaustion via phdr blow-up.
- **PT_INTERP recursion depth ≤ 4** — defense against per-circular-interp DoS.
- **PT_LOAD overlap detection** — defense against per-segment-overlap that could shadow earlier mappings.
- **PT_GNU_STACK absent ⟹ NX stack** — defense against per-implicit-executable-stack on legacy binaries.
- **PT_GNU_RELRO mprotect-to-RO honored by dynamic linker** — defense against per-GOT-overwrite (full RELRO).
- **PT_GNU_PROPERTY parsing strictly bounded** — defense against per-malformed-property-note OOB read.
- **`AT_SECURE` correctly set on setuid/setgid/file-caps exec** — defense against per-LD_PRELOAD privilege escalation.
- **`AT_RANDOM` 16 bytes from CSPRNG** — defense against per-predictable canary (SSP stack-protector).
- **NT_PRSTATUS / NT_PRFPREG / NT_X86_XSTATE size strictly per-arch-known** — defense against per-regset-size mismatch in ptrace.
- **`NT_SIGINFO` size MAY grow; consumers MUST use `n_descsz`** — defense against per-fixed-size-assumption.
- **`SHN_LIVEPATCH` only honored when livepatch module loading active** — defense against per-symbol-table-spoof.

### grsecurity/pax-style reinforcement

- **PAX_ASLR for ET_DYN** — Rookery enforces randomized load base for every `ET_DYN` PIE executable, with entropy = `mmap_base_rnd_bits` (32 bits on x86_64, 28 on i386, 33 on arm64). `personality(ADDR_NO_RANDOMIZE)` requires `CAP_SYS_ADMIN` to enable, unlike vanilla Linux where any user may disable ASLR for the calling process. Defeats per-fixed-load-address bypass and per-libc-relative-offset exploits.
- **PAX_PAGEEXEC + PAX_NOEXEC for PT_GNU_STACK** — Rookery treats the *absence* of PT_GNU_STACK as non-executable stack (matching modern toolchains) AND refuses to load any binary that explicitly requests `PT_GNU_STACK.p_flags & PF_X`. The legacy "trampoline" workaround (executable stack for nested-function trampolines from old gcc) is rejected; binaries built before gcc 4.0 with nested-fn trampolines must be recompiled. Defeats shellcode-on-stack injection categorically.
- **PAX_MPROTECT for PT_GNU_RELRO** — the kernel itself mprotects the PT_GNU_RELRO region to `PROT_READ` immediately after `execve` completes, not waiting for the dynamic linker. (Partial-RELRO only; full-RELRO still requires the linker's post-relocation mprotect, which Rookery additionally enforces via a `PR_SET_NO_NEW_PRIVS`-style ratchet that refuses to grant `PROT_WRITE` back to a once-RELRO'd page.) Defeats per-`__free_hook` / per-`*.got.plt` overwrite exploits.
- **PAX_RANDMMAP for AT_RANDOM source** — the 16 bytes published via `AT_RANDOM` come from `get_random_bytes_user()` with per-exec freshness; the kernel additionally re-seeds the per-task `stack_canary` from independent entropy on every `fork` and `clone`. Defeats per-canary-leak-then-reuse across `fork()` boundaries.
- **AT_SECURE strict-mode setuid bit** — Rookery's `AT_SECURE` is `1` not only for setuid/setgid/file-caps exec but also for any exec that crosses a user-namespace boundary (`CLONE_NEWUSER` enter), drops capabilities, or transitions LSM labels. libc MUST treat `AT_SECURE != 0` as a hard signal to discard `LD_PRELOAD`, `LD_LIBRARY_PATH`, `LD_AUDIT`, `MALLOC_*`, `RES_OPTIONS`, `TMPDIR`, and `IFS`. The kernel additionally hides `/proc/<pid>/environ` of the new exec from non-same-uid readers for the first 100ms (auxv-leak window).
- **PAX_PAGEEXEC PT_LOAD `PF_W | PF_X` rejection** — any `PT_LOAD` segment requesting both `PF_W` and `PF_X` is rejected with `-ENOEXEC` even before mapping, not downgraded silently. Defeats per-RWX-from-disk loaders (legitimate JIT engines must call `mprotect` after the fact, subject to PAX_MPROTECT policy).
- **`AT_PHDR` / `AT_BASE` / `AT_ENTRY` published via auxv use *post-randomization* virtual addresses** — userspace can recover its own load bias from `AT_PHDR - e_phoff`, but the address is bounded to the per-exec randomized region, never leaking the kernel-side mm structure layout.
- **GRKERNSEC_DMESG redacts ELF crash diagnostics** — when a process is killed for an ELF policy violation (W^X violation in PT_LOAD, PT_GNU_STACK refused, interp depth exceeded), the kernel emits the warning to the audit log but redacts faulting addresses, load bias, and binary path from non-root `dmesg` readers.
- **PT_INTERP path validated against per-mount-ns + per-cgroup chroot** — the `PT_INTERP` string is resolved relative to the calling task's mount namespace and cgroup hierarchy, not the host root. Defeats per-chroot-escape via interp-bind-mount tricks.
- **NT_X86_XSTATE / NT_X86_SHSTK ptrace gets require CAP_SYS_PTRACE** — even when the caller is the same uid, reading shadow-stack (CET) state via `ptrace(PTRACE_GETREGSET, NT_X86_SHSTK, ...)` requires `CAP_SYS_PTRACE` in the *target's* user namespace. Defeats per-CET-bypass via shadow-stack-state-disclosure.
- **`AT_RANDOM` page mapped read-only and unmapped on first read** — the 16-byte randomness page is anonymous, read-only, and the dynamic linker is expected to read it once. After 5 seconds OR on first `dlopen()`, the kernel unmaps the page entirely, preventing late attacker code from re-reading the canary.
- **GNU_PROPERTY_AARCH64_FEATURE_1_BTI / GNU_PROPERTY_X86_FEATURE_1_IBT / _SHSTK enforced at load time** — if `PT_GNU_PROPERTY` advertises CET-IBT, CET-SHSTK, or BTI, the kernel sets the corresponding per-VMA enforcement flag and refuses any `mprotect` that would weaken it. Defeats per-mprotect-disable-CET bypass.

