# Tier-3: fs/binfmt_elf.c — ELF executable loader

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/binfmt_elf.c (~2147 lines)
  - include/linux/binfmts.h (struct linux_binprm, struct linux_binfmt)
  - include/linux/elf.h
  - include/uapi/linux/elf.h (Elf64_Ehdr/Phdr, PT_*, PF_*, AT_*)
  - include/uapi/linux/auxvec.h (AT_NULL, AT_HWCAP, AT_PAGESZ, AT_PHDR, AT_BASE, AT_ENTRY, ...)
  - fs/exec.c (begin_new_exec, setup_new_exec, setup_arg_pages, finalize_exec)
  - arch/*/include/asm/elf.h (ELF_PLATFORM, ELF_HWCAP, ELF_ET_DYN_BASE, START_THREAD, SET_PERSONALITY2)
-->

## Summary

The **ELF binary-format loader** turns an opened ELF executable into a running process: it parses the ELF header and program headers from `bprm->buf`, optionally loads a `PT_INTERP` dynamic linker (`/lib64/ld-linux-x86-64.so.2`), mmaps every `PT_LOAD` segment with per-segment `PROT_*` (`MAP_PRIVATE | MAP_EXECUTABLE` with `MAP_FIXED_NOREPLACE` for the first segment, `MAP_FIXED` thereafter), zero-fills the `.bss` tail page and `vm_brk_flags`-extends past `p_filesz` up to `p_memsz`, builds the user stack with `argc`, `argv[]`, `envp[]`, and the `AT_*` auxiliary vector, randomizes `load_bias` (PIE ASLR via `ELF_ET_DYN_BASE + arch_mmap_rnd()`), initializes `mm->{start,end}_{code,data}`, `mm->start_stack`, and `mm->{start_brk,brk}`, and finally branches to user-space at the chosen `elf_entry` via `START_THREAD`. Per-`load_elf_binary`: the heart. Per-`elf_format`: the registered `struct linux_binfmt`. Per-`PT_GNU_STACK`: switches the stack between `EXSTACK_ENABLE_X` (legacy trampolines) and `EXSTACK_DISABLE_X` (W^X). Per-`elf_read_implies_exec`: turns `PROT_READ` into `PROT_READ | PROT_EXEC` for legacy. Per-`PT_GNU_PROPERTY` + `parse_elf_properties`: enables CET/IBT/SHSTK / `prctl(PR_PAC_*)`. Critical for: every userspace process on Linux except #! scripts and `binfmt_misc` interpreters.

This Tier-3 covers `fs/binfmt_elf.c` (~2147 lines). The non-ELF higher-level binfmt-dispatch flow lives in `fs/exec-binfmt.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct linux_binfmt elf_format` | per-`register_binfmt` entry | `BinfmtElf::FORMAT` |
| `struct linux_binprm` | per-exec params (file, buf, argc, envc, p, exec, secureexec, …) | shared with `exec-binfmt.md` |
| `load_elf_binary()` | per-exec entry | `BinfmtElf::load_elf_binary` |
| `load_elf_interp()` | per-`PT_INTERP` load | `BinfmtElf::load_elf_interp` |
| `load_elf_phdrs()` | per-`e_phoff` read | `BinfmtElf::load_elf_phdrs` |
| `elf_load()` | per-`PT_LOAD` map + bss-pad | `BinfmtElf::elf_load` |
| `elf_map()` | per-`vm_mmap` wrapper | `BinfmtElf::elf_map` |
| `elf_read()` | per-`kernel_read` strict | `BinfmtElf::elf_read` |
| `padzero()` | per-`.bss`-tail zero | `BinfmtElf::padzero` |
| `total_mapping_size()` | per-ET_DYN reservation extent | `BinfmtElf::total_mapping_size` |
| `maximum_alignment()` | per-PT_LOAD max p_align | `BinfmtElf::maximum_alignment` |
| `make_prot()` | per-`p_flags`→`PROT_*` | `BinfmtElf::make_prot` |
| `create_elf_tables()` | per-stack: argv/envp/auxv populate | `BinfmtElf::create_elf_tables` |
| `parse_elf_properties()` / `parse_elf_property()` | per-`PT_GNU_PROPERTY` (CET/BTI) | `BinfmtElf::parse_elf_properties` |
| `arch_elf_pt_proc()` / `arch_check_elf()` | per-arch hook | per-arch trait |
| `elf_core_dump()` | per-coredump writer | `BinfmtElf::core_dump` (separate Tier-3) |
| `ELF_ET_DYN_BASE` | per-arch PIE base | per-arch const |
| `START_THREAD` | per-arch entry into userspace | per-arch macro |
| `SET_PERSONALITY2` | per-arch personality bits | per-arch macro |
| `setup_arg_pages` | per-stack VMA setup | from `fs/exec.c` |
| `begin_new_exec` / `setup_new_exec` / `finalize_exec` | per-exec lifecycle | from `fs/exec.c` |

## Compatibility contract

REQ-1: struct linux_binfmt elf_format:
- .module = THIS_MODULE.
- .load_binary = load_elf_binary.
- .load_shlib = NULL (a.out shlib obsolete).
- .core_dump = elf_core_dump (CONFIG_ELF_CORE).
- .min_coredump = ELF_EXEC_PAGESIZE.
- Registered via `register_binfmt` from `init_elf_binfmt`; deregistered in `exit_elf_binfmt`.

REQ-2: load_elf_binary(bprm) — top-level dispatch:
- elf_ex = (struct elfhdr *)bprm.buf (BINPRM_BUF_SIZE = 256 bytes pre-read by exec).
- /* sanity */
- if memcmp(elf_ex.e_ident, ELFMAG, SELFMAG) != 0: return -ENOEXEC.
- if e_type ∉ {ET_EXEC, ET_DYN}: return -ENOEXEC.
- if !elf_check_arch(elf_ex): return -ENOEXEC.    /* per-EM_X86_64, EM_AARCH64, … */
- if elf_check_fdpic(elf_ex): return -ENOEXEC.    /* fdpic handled by binfmt_elf_fdpic */
- if !can_mmap_file(bprm.file): return -ENOEXEC.
- elf_phdata = load_elf_phdrs(elf_ex, bprm.file).
- if !elf_phdata: return -ENOEXEC.

REQ-3: PT_INTERP discovery:
- For each phdr i:
  - if p_type == PT_GNU_PROPERTY: stash as elf_property_phdata.
  - if p_type != PT_INTERP: continue.
  - if p_filesz > PATH_MAX ∨ p_filesz < 2: -ENOEXEC.
  - elf_interpreter = kmalloc(p_filesz).
  - elf_read(bprm.file, elf_interpreter, p_filesz, p_offset).
  - require trailing NUL.
  - interpreter = open_exec(elf_interpreter).      /* exec-safe open */
  - would_dump(bprm, interpreter).
  - interp_elf_ex = kmalloc(struct elfhdr).
  - elf_read(interpreter, interp_elf_ex, sizeof(*interp_elf_ex), 0).
  - break (only first PT_INTERP honored).

REQ-4: PT_GNU_STACK + PT_LOPROC..PT_HIPROC:
- For each phdr:
  - PT_GNU_STACK: executable_stack = (p_flags & PF_X) ? EXSTACK_ENABLE_X : EXSTACK_DISABLE_X. /* default = EXSTACK_DEFAULT */
  - PT_LOPROC..PT_HIPROC: arch_elf_pt_proc(elf_ex, ppnt, bprm.file, false, &arch_state).

REQ-5: Interpreter validation:
- if interpreter:
  - require ELFMAG match on interp_elf_ex.
  - elf_check_arch ∧ !elf_check_fdpic on interp_elf_ex.
  - interp_elf_phdata = load_elf_phdrs(interp_elf_ex, interpreter).
  - For each interp phdr: PT_GNU_PROPERTY override; PT_LOPROC..PT_HIPROC: arch_elf_pt_proc(... true, ...).

REQ-6: parse_elf_properties:
- Caller-file = interpreter ?: bprm.file.
- Walk PT_GNU_PROPERTY notes: NT_GNU_PROPERTY_TYPE_0 entries.
- Per-arch: enable IBT/SHSTK on x86 (`GNU_PROPERTY_X86_FEATURE_1_IBT`, `_SHSTK`); enable BTI/PAC on arm64.

REQ-7: arch_check_elf:
- Final arch reject window (e.g. EM_x86_64 vs CPU-feature mismatch).
- Must be after PT_LOPROC walks; before begin_new_exec.

REQ-8: begin_new_exec(bprm):
- Per-`fs/exec.c`: flush current mm, signals, fd-table (CLOEXEC), comm; set credentials; mark TIF_NEW_EXEC.
- After this point, failure ⟹ SIGKILL to the calling task (we can't return to old image).

REQ-9: Personality + ASLR:
- SET_PERSONALITY2(*elf_ex, &arch_state).   /* sets PER_LINUX32 / READ_IMPLIES_EXEC / FDPIC */
- if elf_read_implies_exec(*elf_ex, executable_stack): current.personality |= READ_IMPLIES_EXEC.
- snapshot_randomize_va_space = READ_ONCE(randomize_va_space).
- if !(personality & ADDR_NO_RANDOMIZE) ∧ snapshot_randomize_va_space: current.flags |= PF_RANDOMIZE.

REQ-10: setup_new_exec + setup_arg_pages:
- setup_new_exec(bprm).
- setup_arg_pages(bprm, randomize_stack_top(STACK_TOP), executable_stack):
  - Allocate a stack VMA at STACK_TOP minus a randomized delta.
  - Set its VM_GROWSDOWN, VM_STACK; per-`executable_stack` set VM_EXEC.

REQ-11: PT_LOAD segment mapping (per-load loop):
- elf_prot = make_prot(p_flags, &arch_state, !!interpreter, false).
- elf_flags = MAP_PRIVATE.
- vaddr = ppnt.p_vaddr.
- if !first_pt_load: elf_flags |= MAP_FIXED.
- else if ET_EXEC: elf_flags |= MAP_FIXED_NOREPLACE.
- else if ET_DYN:
  - total_size = total_mapping_size(elf_phdata, e_phnum).
  - alignment = maximum_alignment(elf_phdata, e_phnum).
  - if interpreter:
    - load_bias = ELF_ET_DYN_BASE.
    - if PF_RANDOMIZE: load_bias += arch_mmap_rnd().
    - if alignment: load_bias &= ~(alignment - 1).
    - elf_flags |= MAP_FIXED_NOREPLACE.
  - else: /* static PIE (loader run directly) */
    - if alignment > ELF_MIN_ALIGN:
      - probe-map at 0 to learn mmap-base, vm_munmap, re-align, MAP_FIXED_NOREPLACE.
    - else: load_bias = 0.
  - load_bias = ELF_PAGESTART(load_bias - vaddr).
- error = elf_load(bprm.file, load_bias + vaddr, ppnt, elf_prot, elf_flags, total_size).
- if first_pt_load ∧ ET_DYN: load_bias += error - ELF_PAGESTART(load_bias + vaddr); reloc_func_desc = load_bias.
- /* phdr_addr discovery */
- if ppnt.p_offset ≤ e_phoff < ppnt.p_offset + p_filesz: phdr_addr = e_phoff - p_offset + p_vaddr.
- /* mm code/data extents */
- if (p_flags & PF_X) ∧ p_vaddr < start_code: start_code = p_vaddr.
- if p_vaddr > start_data: start_data = p_vaddr.
- /* overflow checks */
- require p_filesz ≤ p_memsz; p_memsz ≤ TASK_SIZE; TASK_SIZE - p_memsz ≥ p_vaddr.
- end_code = max(end_code, p_vaddr + p_filesz) when PF_X.
- end_data = max(end_data, p_vaddr + p_filesz).
- elf_brk = max(elf_brk, p_vaddr + p_memsz).

REQ-12: elf_load (per-`PT_LOAD` with .bss):
- if p_filesz:
  - map_addr = elf_map(file, addr, ppnt, prot, type, total_size).
  - if p_memsz > p_filesz:
    - zero_start = map_addr + ELF_PAGEOFFSET(p_vaddr) + p_filesz.
    - zero_end   = map_addr + ELF_PAGEOFFSET(p_vaddr) + p_memsz.
    - padzero(zero_start); error tolerated unless PROT_WRITE.
- else: map_addr = zero_start = ELF_PAGESTART(addr); zero_end = ... + p_memsz.
- if p_memsz > p_filesz:
  - zero_start = ELF_PAGEALIGN(zero_start); zero_end = ELF_PAGEALIGN(zero_end).
  - vm_brk_flags(zero_start, zero_end - zero_start, prot & PROT_EXEC).

REQ-13: elf_map:
- size = p_filesz + ELF_PAGEOFFSET(p_vaddr).
- off  = p_offset - ELF_PAGEOFFSET(p_vaddr).
- addr = ELF_PAGESTART(addr).
- size = ELF_PAGEALIGN(size).
- if total_size:
  - map_addr = vm_mmap(file, addr, total_size, prot, type, off).
  - if !BAD_ADDR(map_addr): vm_munmap(map_addr + size, total_size - size). /* trim tail */
- else: map_addr = vm_mmap(file, addr, size, prot, type, off).
- return map_addr.

REQ-14: Post-segments fixups:
- e_entry = elf_ex.e_entry + load_bias.
- phdr_addr += load_bias; elf_brk += load_bias.
- start_code += load_bias; end_code += load_bias; start_data += load_bias; end_data += load_bias.

REQ-15: load_elf_interp(interp_elf_ex, interpreter, no_base, interp_phdata, &arch_state):
- Validate ET_{EXEC,DYN}, elf_check_arch, !elf_check_fdpic.
- total_size = total_mapping_size(interp_phdata, e_phnum).
- For each PT_LOAD eppnt:
  - elf_type = MAP_PRIVATE; elf_prot = make_prot(p_flags, arch, true, true).
  - vaddr = eppnt.p_vaddr.
  - if !load_addr_set ∧ ET_DYN: elf_type |= MAP_FIXED_NOREPLACE.
  - else if load_addr_set: elf_type |= MAP_FIXED.
  - map_addr = elf_load(interpreter, load_addr + vaddr, eppnt, elf_prot, elf_type, total_size).
  - if !load_addr_set: load_addr = map_addr - ELF_PAGESTART(vaddr); load_addr_set = 1.
- error = load_addr.   /* returned as relocation adjustment to caller */

REQ-16: elf_entry selection:
- if interpreter:
  - elf_entry = load_elf_interp(...).
  - interp_load_addr = elf_entry.
  - elf_entry += interp_elf_ex.e_entry.
  - reloc_func_desc = interp_load_addr.
  - exe_file_allow_write_access(interpreter); fput(interpreter).
- else:
  - elf_entry = e_entry.
- require !BAD_ADDR(elf_entry); else -EINVAL.

REQ-17: ARCH_SETUP_ADDITIONAL_PAGES (vDSO):
- ARCH_SETUP_ADDITIONAL_PAGES(bprm, elf_ex, !!interpreter):
  - Maps the vDSO and signal-trampoline pages into the new mm.

REQ-18: create_elf_tables — stack layout (top-to-bottom):
- Per-strings stored at the top of the stack by `copy_strings_kernel` / `copy_strings` (in exec.c) before binfmt; p = bprm.p points just below them.
- u_platform = STACK_ALLOC(p, strlen(ELF_PLATFORM)+1); copy_to_user.
- u_base_platform = STACK_ALLOC(p, strlen(ELF_BASE_PLATFORM)+1) if defined.
- get_random_bytes(k_rand_bytes, 16); STACK_ALLOC + copy_to_user → u_rand_bytes (for AT_RANDOM).
- elf_info populated via NEW_AUX_ENT:
  - ARCH_DLINFO first (PPC alignment).
  - AT_HWCAP, AT_PAGESZ, AT_CLKTCK, AT_PHDR, AT_PHENT, AT_PHNUM, AT_BASE (= interp_load_addr), AT_FLAGS, AT_ENTRY (= e_entry), AT_UID, AT_EUID, AT_GID, AT_EGID, AT_SECURE, AT_RANDOM.
  - AT_HWCAP2 / AT_HWCAP3 / AT_HWCAP4 if defined.
  - AT_EXECFN = bprm.exec.
  - AT_PLATFORM = u_platform (if ELF_PLATFORM).
  - AT_BASE_PLATFORM = u_base_platform (if ELF_BASE_PLATFORM).
  - AT_EXECFD = bprm.execfd if have_execfd.
  - AT_RSEQ_FEATURE_SIZE = offsetof(rseq, end); AT_RSEQ_ALIGN = rseq_alloc_align() (CONFIG_RSEQ).
  - AT_NULL = 0 terminator + clear remainder of mm.saved_auxv.
- ei_index = auxv-entry-count.
- sp = STACK_ADD(p, ei_index).
- items = (argc+1) + (envc+1) + 1.
- bprm.p = STACK_ROUND(sp, items).      /* 16-byte align */
- Grow stack: mmap_write_lock_killable; find_extend_vma_locked(mm, bprm.p); mmap_write_unlock.
- put_user(argc, sp++).
- For arg in 0..argc: put_user((elf_addr_t)p, sp++); p += strnlen_user(p, MAX_ARG_STRLEN) (range-check); mm.arg_end = p.
- put_user(0, sp++).
- mm.env_end = mm.env_start = p.
- For env in 0..envc: put_user((elf_addr_t)p, sp++); p += strnlen_user(...); mm.env_end = p.
- put_user(0, sp++).
- copy_to_user(sp, mm.saved_auxv, ei_index * sizeof(elf_addr_t)).   /* AT_* block */

REQ-19: brk / start_brk init + brk-ASLR:
- mm.end_code = end_code; mm.start_code = start_code.
- mm.start_data = start_data; mm.end_data = end_data.
- mm.start_stack = bprm.p.
- /* static PIE: move brk out of mmap region */
- if !CONFIG_COMPAT_BRK ∧ CONFIG_ARCH_HAS_ELF_RANDOMIZE ∧ ET_DYN ∧ !interpreter:
  - elf_brk = ELF_ET_DYN_BASE; brk_moved = true.
- mm.start_brk = mm.brk = ELF_PAGEALIGN(elf_brk).
- if PF_RANDOMIZE ∧ randomize_va_space > 1:
  - if !brk_moved: mm.brk = mm.start_brk = mm.brk + PAGE_SIZE.   /* gap between .bss and brk */
  - mm.brk = mm.start_brk = arch_randomize_brk(mm). brk_moved = true.
- if `compat_brk_randomized` ∧ brk_moved: current.brk_randomized = 1.

REQ-20: MMAP_PAGE_ZERO (SVR4 emulation):
- if personality & MMAP_PAGE_ZERO:
  - vm_mmap(NULL, 0, PAGE_SIZE, PROT_READ | PROT_EXEC, MAP_FIXED | MAP_PRIVATE, 0).
  - do_mseal(0, PAGE_SIZE, 0) (best-effort).

REQ-21: ELF_PLAT_INIT + finalize:
- regs = current_pt_regs().
- ELF_PLAT_INIT(regs, reloc_func_desc).   /* per-arch register init: i386 %edx = DT_FINI; ppc64 function-descriptor */
- finalize_exec(bprm).
- START_THREAD(elf_ex, regs, elf_entry, bprm.p).   /* writes IP/SP, marks ready to return-to-user */
- return 0.

REQ-22: fdpic distinction:
- elf_check_fdpic(elf_ex): EM-specific match for FDPIC binaries. `binfmt_elf.c` *rejects* them (returns -ENOEXEC) so `binfmt_elf_fdpic.c` (separate Tier-3) can claim them. Pointer to the fdpic variant is therefore via fallthrough in binfmt-dispatch.

REQ-23: Error cleanup (out_free_*):
- out_free_dentry: kfree(interp_elf_ex); kfree(interp_elf_phdata).
- out_free_file: exe_file_allow_write_access(interpreter); fput(interpreter) if non-NULL.
- out_free_ph: kfree(elf_phdata); jump to out.
- out: return retval (errors before begin_new_exec; after, exec.c will SIGKILL).

REQ-24: Constants & policy hooks:
- ELF_MIN_ALIGN = page-size (arch-defined).
- ELF_PAGESTART / ELF_PAGEOFFSET / ELF_PAGEALIGN: page-rounding macros.
- ELF_ET_DYN_BASE: per-arch PIE base (2/3 of TASK_SIZE on x86_64; arch_mmap_rnd-randomized).
- BAD_ADDR(x): `(unsigned long)(x) >= TASK_SIZE`.
- can_mmap_file(file): refuses pseudo-filesystems lacking mmap.
- elf_read_implies_exec(ex, stk): legacy-i386 EXSTACK_DEFAULT ⟹ R⟹X.
- AT_VECTOR_SIZE_BASE / AT_VECTOR_SIZE_ARCH: sized to fit all NEW_AUX_ENT entries plus ARCH_DLINFO.

## Acceptance Criteria

- [ ] AC-1: `execve("/bin/ls", argv, envp)`: load_elf_binary returns 0; ls runs; argc / argv visible via /proc/self/cmdline.
- [ ] AC-2: ET_EXEC static binary: first PT_LOAD mapped with MAP_FIXED_NOREPLACE at e_phoff vaddr; subsequent with MAP_FIXED.
- [ ] AC-3: ET_DYN PIE with PT_INTERP: load_bias = ELF_ET_DYN_BASE + random; differs across exec invocations when `kernel.randomize_va_space ≥ 1`.
- [ ] AC-4: ET_DYN static (no PT_INTERP): load_bias = arch_mmap_rnd-controlled mmap base; brk relocated to ELF_ET_DYN_BASE.
- [ ] AC-5: PT_INTERP `/lib64/ld-linux-x86-64.so.2`: opened, headers read, mapped via load_elf_interp; elf_entry = interp_load_addr + interp_elf_ex.e_entry.
- [ ] AC-6: PT_GNU_STACK with PF_X: stack VMA gets VM_EXEC; without: cleared.
- [ ] AC-7: Stack auxv contains AT_PHDR / AT_PHENT / AT_PHNUM / AT_BASE / AT_ENTRY / AT_RANDOM / AT_EXECFN / AT_PAGESZ / AT_HWCAP / AT_PLATFORM / AT_NULL.
- [ ] AC-8: AT_SECURE = bprm.secureexec (set when setuid/setgid/setcaps elevated privileges).
- [ ] AC-9: AT_RANDOM points to 16 bytes of `get_random_bytes` on the stack.
- [ ] AC-10: .bss is zero-filled: pages between `p_filesz` and `p_memsz` allocated via vm_brk_flags; readable as zeros.
- [ ] AC-11: mm.start_code / end_code / start_data / end_data / start_stack / start_brk / brk all set; /proc/self/maps reflects the segments.
- [ ] AC-12: Malformed ELF (bad ELFMAG / bad e_type / e_phnum * e_phentsize > 65536): returns -ENOEXEC and leaves bprm unchanged.
- [ ] AC-13: Interpreter path > PATH_MAX or not NUL-terminated: returns -ENOEXEC.
- [ ] AC-14: parse_elf_properties with `GNU_PROPERTY_X86_FEATURE_1_IBT`: arch_state records IBT; later mprotect of code pages keeps IBT bit.
- [ ] AC-15: MMAP_PAGE_ZERO personality: page 0 mapped PROT_READ|PROT_EXEC and sealed (best-effort).

## Architecture

```
struct LinuxBinprm {       // shared with exec-binfmt.md
  vma: *VmAreaStruct,      // top-of-stack VMA
  mm: *MmStruct,           // new mm-to-be
  p: usize,                // current stack-pointer (kernel view, top)
  argmin: usize,
  have_execfd: bool,
  execfd: i32,
  argc: i32,
  envc: i32,
  filename: KString,
  interp: KString,
  fdpath: KString,
  buf: [u8; BINPRM_BUF_SIZE], // first 256 bytes of file
  file: *File,
  exec: u64,
  secureexec: bool,
  interp_flags: u32,
  ...
}

struct ArchElfState {       // arch-overridable
  // x86: GNU_PROPERTY_X86_FEATURE_1_{IBT,SHSTK}
  // arm64: GNU_PROPERTY_AARCH64_FEATURE_1_{BTI,PAC}
}
```

`BinfmtElf::load_elf_binary(bprm) -> Result<()>`:
1. elf_ex = &bprm.buf as &Elfhdr.
2. /* sanity */
3. require elf_ex.e_ident starts with ELFMAG && e_type ∈ {ET_EXEC, ET_DYN} && elf_check_arch(elf_ex) && !elf_check_fdpic(elf_ex) && can_mmap_file(bprm.file). else -ENOEXEC.
4. elf_phdata = BinfmtElf::load_elf_phdrs(elf_ex, bprm.file)?.
5. /* PT_INTERP discovery */
6. let (interpreter, interp_elf_ex) = BinfmtElf::open_interp(bprm, elf_ex, &elf_phdata)?;
7. /* per-phdr scan: PT_GNU_STACK + PT_LOPROC..HIPROC */
8. let (executable_stack, arch_state) = BinfmtElf::scan_phdrs(elf_ex, &elf_phdata)?;
9. /* interpreter phdrs */
10. let interp_elf_phdata = if interpreter.is_some() { Some(BinfmtElf::load_elf_phdrs(&interp_elf_ex, &interpreter)?) } else { None };
11. /* PT_GNU_PROPERTY notes */
12. BinfmtElf::parse_elf_properties(interpreter.as_ref().unwrap_or(&bprm.file), property_phdata, &mut arch_state)?.
13. /* arch reject window */
14. arch_check_elf(elf_ex, interpreter.is_some(), &interp_elf_ex, &arch_state)?.
15. /* point of no return: flush old image */
16. begin_new_exec(bprm)?.
17. SET_PERSONALITY2(*elf_ex, &arch_state).
18. if elf_read_implies_exec(*elf_ex, executable_stack): current.personality |= READ_IMPLIES_EXEC.
19. /* ASLR */
20. let randomize = READ_ONCE(randomize_va_space).
21. if !(current.personality & ADDR_NO_RANDOMIZE) && randomize: current.flags |= PF_RANDOMIZE.
22. setup_new_exec(bprm).
23. setup_arg_pages(bprm, randomize_stack_top(STACK_TOP), executable_stack)?.
24. /* PT_LOAD loop */
25. let (load_bias, e_entry, phdr_addr, extents) = BinfmtElf::map_segments(bprm, elf_ex, &elf_phdata, interpreter.is_some(), &arch_state)?;
26. /* interpreter */
27. let elf_entry = if let Some(interp) = interpreter {
       let interp_load_addr = BinfmtElf::load_elf_interp(&interp_elf_ex, interp, load_bias, &interp_elf_phdata, &arch_state)?;
       interp_load_addr + interp_elf_ex.e_entry
     } else { e_entry };
28. /* vDSO + auxv */
29. ARCH_SETUP_ADDITIONAL_PAGES(bprm, elf_ex, interpreter.is_some())?.
30. BinfmtElf::create_elf_tables(bprm, elf_ex, interp_load_addr, e_entry, phdr_addr)?.
31. /* mm extents */
32. let mm = current.mm;
33. mm.{start,end}_code = extents.code; mm.{start,end}_data = extents.data; mm.start_stack = bprm.p.
34. /* brk + ASLR */
35. BinfmtElf::init_brk(mm, elf_ex, interpreter.is_some(), extents.brk, randomize).
36. /* MMAP_PAGE_ZERO */
37. if current.personality & MMAP_PAGE_ZERO: vm_mmap(None, 0, PAGE_SIZE, PROT_READ|PROT_EXEC, MAP_FIXED|MAP_PRIVATE, 0); do_mseal(0, PAGE_SIZE, 0).
38. /* enter user */
39. let regs = current_pt_regs();
40. ELF_PLAT_INIT(regs, reloc_func_desc);
41. finalize_exec(bprm);
42. START_THREAD(elf_ex, regs, elf_entry, bprm.p);
43. Ok(()).

`BinfmtElf::map_segments(...)` (the heart of PT_LOAD loop):
1. let mut first = true; load_bias = 0; brk = 0; phdr_addr = 0.
2. let mut extents = Extents::default().
3. for ppnt in elf_phdata where p_type == PT_LOAD:
   - elf_prot = BinfmtElf::make_prot(p_flags, &arch_state, has_interp, false).
   - elf_flags = MAP_PRIVATE.
   - if !first: elf_flags |= MAP_FIXED.
   - else if ET_EXEC: elf_flags |= MAP_FIXED_NOREPLACE.
   - else: /* ET_DYN */
     - total_size = BinfmtElf::total_mapping_size(elf_phdata).
     - alignment = BinfmtElf::maximum_alignment(elf_phdata).
     - if has_interp:
       - load_bias = ELF_ET_DYN_BASE.
       - if PF_RANDOMIZE: load_bias += arch_mmap_rnd().
       - if alignment: load_bias &= !(alignment - 1).
       - elf_flags |= MAP_FIXED_NOREPLACE.
     - else:
       - if alignment > ELF_MIN_ALIGN: probe-map@0 + munmap → align → MAP_FIXED_NOREPLACE.
       - else: load_bias = 0.
     - load_bias = ELF_PAGESTART(load_bias - p_vaddr).
   - addr = BinfmtElf::elf_load(bprm.file, load_bias + p_vaddr, ppnt, elf_prot, elf_flags, total_size)?.
   - if first && ET_DYN: load_bias += addr - ELF_PAGESTART(load_bias + p_vaddr); reloc_func_desc = load_bias.
   - first = false.
   - if p_offset ≤ e_phoff < p_offset + p_filesz: phdr_addr = e_phoff - p_offset + p_vaddr.
   - extents.update(p_vaddr, p_filesz, p_memsz, p_flags & PF_X).
   - require !BAD_ADDR(p_vaddr) && p_filesz ≤ p_memsz && p_memsz ≤ TASK_SIZE && TASK_SIZE - p_memsz ≥ p_vaddr.
4. /* apply load_bias to all extents */
5. extents += load_bias.
6. return (load_bias, elf_ex.e_entry + load_bias, phdr_addr + load_bias, extents).

`BinfmtElf::elf_load(file, addr, eppnt, prot, type, total_size) -> Result<usize>`:
1. if p_filesz > 0:
   - map_addr = BinfmtElf::elf_map(file, addr, eppnt, prot, type, total_size)?.
   - if p_memsz > p_filesz:
     - zero_start = map_addr + ELF_PAGEOFFSET(p_vaddr) + p_filesz.
     - zero_end = map_addr + ELF_PAGEOFFSET(p_vaddr) + p_memsz.
     - BinfmtElf::padzero(zero_start)?;   /* may fault on RO segment — tolerated */
2. else:
   - map_addr = zero_start = ELF_PAGESTART(addr).
   - zero_end = zero_start + ELF_PAGEOFFSET(p_vaddr) + p_memsz.
3. if p_memsz > p_filesz:
   - zero_start = ELF_PAGEALIGN(zero_start). zero_end = ELF_PAGEALIGN(zero_end).
   - vm_brk_flags(zero_start, zero_end - zero_start, prot & PROT_EXEC)?.
4. Ok(map_addr).

`BinfmtElf::create_elf_tables(bprm, elf_ex, interp_load_addr, e_entry, phdr_addr) -> Result<()>`:
1. p = arch_align_stack(bprm.p).
2. /* AT_PLATFORM / AT_BASE_PLATFORM / AT_RANDOM bytes laid above auxv */
3. let u_platform = if ELF_PLATFORM.is_some() { Some(STACK_ALLOC(p, k_platform.len()+1)) } else { None };
   copy_to_user.
4. analogous for u_base_platform.
5. get_random_bytes(&mut k_rand_bytes); STACK_ALLOC → u_rand_bytes; copy_to_user.
6. /* fill mm.saved_auxv */
7. let elf_info = mm.saved_auxv.as_mut_ptr() as *mut elf_addr_t;
8. macro NEW_AUX_ENT(id, val) writes 2 words.
9. ARCH_DLINFO first.
10. AT_HWCAP, AT_PAGESZ, AT_CLKTCK, AT_PHDR, AT_PHENT, AT_PHNUM, AT_BASE, AT_FLAGS, AT_ENTRY, AT_UID, AT_EUID, AT_GID, AT_EGID, AT_SECURE, AT_RANDOM.
11. Optional AT_HWCAP2/3/4.
12. AT_EXECFN, AT_PLATFORM, AT_BASE_PLATFORM, AT_EXECFD.
13. AT_RSEQ_FEATURE_SIZE, AT_RSEQ_ALIGN.
14. AT_NULL + memset the tail.
15. ei_index = (elf_info - mm.saved_auxv) / 2.
16. sp = STACK_ADD(p, ei_index).
17. items = (argc+1) + (envc+1) + 1.
18. bprm.p = STACK_ROUND(sp, items).
19. mmap_write_lock_killable; vma = find_extend_vma_locked(mm, bprm.p); mmap_write_unlock.
20. put_user(argc).
21. for _ in 0..argc: put_user(p); p += strnlen_user(p, MAX_ARG_STRLEN). 0-terminator.
22. mm.arg_end = p. mm.env_end = mm.env_start = p.
23. for _ in 0..envc: same. 0-terminator. mm.env_end = p.
24. copy_to_user(sp, mm.saved_auxv, ei_index * sizeof(elf_addr_t)).
25. Ok(()).

`BinfmtElf::init_brk(mm, elf_ex, has_interp, elf_brk, randomize)`:
1. if !COMPAT_BRK ∧ ARCH_HAS_ELF_RANDOMIZE ∧ e_type==ET_DYN ∧ !has_interp:
   - elf_brk = ELF_ET_DYN_BASE; brk_moved = true.
2. mm.start_brk = mm.brk = ELF_PAGEALIGN(elf_brk).
3. if (current.flags & PF_RANDOMIZE) ∧ randomize > 1:
   - if !brk_moved: mm.brk = mm.start_brk = mm.brk + PAGE_SIZE.
   - mm.brk = mm.start_brk = arch_randomize_brk(mm).
   - brk_moved = true.
4. if compat_brk_randomized && brk_moved: current.brk_randomized = 1.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `phdr_size_bounded` | INVARIANT | per-load_elf_phdrs: e_phentsize == sizeof(elf_phdr) ∧ size ≤ 65536. |
| `interp_path_bounded` | INVARIANT | per-PT_INTERP: 2 ≤ p_filesz ≤ PATH_MAX ∧ NUL-terminated. |
| `segment_extent_in_task_size` | INVARIANT | per-PT_LOAD: p_filesz ≤ p_memsz ∧ p_memsz ≤ TASK_SIZE ∧ TASK_SIZE - p_memsz ≥ p_vaddr. |
| `first_load_uses_FIXED_NOREPLACE_or_addr_probe` | INVARIANT | per-first PT_LOAD: MAP_FIXED_NOREPLACE for ET_EXEC and ET_DYN-with-interp; for static PIE either MAP_FIXED_NOREPLACE or load_bias==0. |
| `subsequent_loads_use_MAP_FIXED` | INVARIANT | per-non-first PT_LOAD: MAP_FIXED is in elf_flags. |
| `bss_zeroed` | INVARIANT | per-elf_load: pages between p_filesz and p_memsz allocated and zero-filled. |
| `stack_argv_envp_terminated` | INVARIANT | per-create_elf_tables: argv and envp each end with a 0 word in user-space. |
| `auxv_AT_NULL_terminator` | INVARIANT | per-create_elf_tables: final auxv entry is `(AT_NULL, 0)`. |
| `auxv_AT_RANDOM_points_to_user_bytes` | INVARIANT | per-create_elf_tables: AT_RANDOM points to 16 bytes copy_to_user'd from `k_rand_bytes`. |
| `auxv_AT_SECURE_eq_secureexec` | INVARIANT | per-create_elf_tables: AT_SECURE == bprm.secureexec. |
| `pie_load_bias_aligned` | INVARIANT | per-ET_DYN with alignment a: load_bias & (a-1) == 0. |
| `interpreter_fput_balanced` | INVARIANT | per-load_elf_binary: every open_exec(interpreter) matched by fput on success or error path. |
| `phdrs_kfree_balanced` | INVARIANT | per-load_elf_binary: elf_phdata and interp_elf_phdata kfree'd on every exit path. |
| `begin_new_exec_point_of_no_return` | INVARIANT | per-load_elf_binary: any failure after begin_new_exec ⟹ task SIGKILL'd (no -E returned to caller's syscall frame). |

### Layer 2: TLA+

`fs/binfmt-elf.tla`:
- States: parsing → interp_open → arch_check → begin_new_exec → arg_pages → segment_map* → interp_map? → vdso → auxv → brk → start_thread.
- Properties:
  - `safety_no_user_mapping_before_begin_new_exec` — per-load_elf_binary: no PT_LOAD reaches vm_mmap before begin_new_exec.
  - `safety_interpreter_fput_paired` — per-open_exec(interpreter): fput eventually called.
  - `safety_phdr_buffers_freed` — per-kmalloc of elf_phdata / interp_elf_phdata: kfree on every exit.
  - `safety_argv_envp_strnlen_bounded` — per-create_elf_tables: each argv/envp entry ≤ MAX_ARG_STRLEN.
  - `safety_aslr_when_randomize_enabled` — per-PF_RANDOMIZE ∧ randomize_va_space > 0: load_bias depends on arch_mmap_rnd output.
  - `liveness_per_load_eventually_starts_user` — per-load_elf_binary: terminates with START_THREAD or an error before begin_new_exec.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `load_elf_binary` post: ok ⟹ START_THREAD invoked with elf_entry in [ELF_MIN_ADDR, TASK_SIZE) | `BinfmtElf::load_elf_binary` |
| `map_segments` post: returned extents cover every PT_LOAD; phdr_addr lies inside a PT_LOAD if e_phoff lay inside one in-file | `BinfmtElf::map_segments` |
| `elf_load` post: addr..addr+ELF_PAGEALIGN(p_memsz) is mapped (FILE for [0,p_filesz), ANON for [p_filesz, p_memsz)) | `BinfmtElf::elf_load` |
| `load_elf_interp` post: returns load_addr such that load_addr + interp.e_entry is a valid PC | `BinfmtElf::load_elf_interp` |
| `create_elf_tables` post: bprm.p is 16-byte aligned; sp[0]==argc; last auxv pair is (AT_NULL,0) | `BinfmtElf::create_elf_tables` |
| `init_brk` post: mm.brk ≥ mm.start_brk ≥ ELF_PAGEALIGN(elf_brk); if PF_RANDOMIZE: != original | `BinfmtElf::init_brk` |
| `make_prot` post: PROT_READ ⟺ PF_R; PROT_WRITE ⟺ PF_W; PROT_EXEC ⟺ PF_X (modulo arch_elf_adjust_prot) | `BinfmtElf::make_prot` |

### Layer 4: Verus/Creusot functional

`Per-execve: open_exec(file) → bprm-init → load_elf_binary → (phdr-parse → PT_INTERP-open → arch-checks → begin_new_exec → setup_arg_pages → PT_LOAD-map → load_elf_interp → vDSO → create_elf_tables(argv,envp,auxv) → mm.brk-init → START_THREAD)` semantic equivalence to Linux per the System V ABI ELF psABI for x86_64 / aarch64 / riscv64 and `Documentation/admin-guide/sysctl/kernel.rst` (randomize_va_space) / `man execve(2)` / `man elf(5)`.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

ELF-loader reinforcement:

- **Per-`begin_new_exec` is point-of-no-return** — defense against per-info-leak-from-partial-load (failure after this point kills the task instead of returning to old image).
- **Per-`MAP_FIXED_NOREPLACE` on first ET_EXEC PT_LOAD** — defense against per-collision with an attacker-pre-mapped region at the load address (Stack Clash / TOCTOU).
- **Per-`PF_RANDOMIZE` + `ELF_ET_DYN_BASE` + `arch_mmap_rnd`** — defense against per-known-image-layout for ROP/JOP gadget chains.
- **Per-`PT_INTERP` filesz ≤ PATH_MAX ∧ NUL-terminated** — defense against per-overlong / unterminated interp path → kernel buffer-read overrun.
- **Per-`open_exec` on interpreter** — defense against per-arbitrary-file-as-interp; honors noexec / exec-domain restrictions.
- **Per-`would_dump(bprm, interpreter)`** — defense against per-leak of unreadable interpreter contents via coredump.
- **Per-`PT_GNU_STACK` honors PF_X / !PF_X** — defense against per-W^X violation (legacy executable stack quarantined to opt-in).
- **Per-`PT_GNU_PROPERTY` + `parse_elf_properties` enabling IBT/SHSTK/BTI/PAC** — defense against per-indirect-branch & per-return-oriented attacks.
- **Per-`elf_phnum * elf_phentsize ≤ 65536` cap** — defense against per-DoS via attacker-claimed massive phdr count.
- **Per-`p_filesz ≤ p_memsz ≤ TASK_SIZE` checks before map** — defense against per-overflow leading to map at small TASK_SIZE wrap-around.
- **Per-stack-VMA `find_extend_vma_locked` under mmap_write_lock_killable** — defense against per-stack-pivot during exec.
- **Per-`AT_RANDOM` 16-byte ChaCha-seeded** — defense against per-predictable userspace PRNG seeding (glibc / musl init).
- **Per-`AT_SECURE` set on suid/sgid/caps** — defense against per-LD_PRELOAD / LD_LIBRARY_PATH abuse via untrusted env.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `fs/binfmt_elf_fdpic.c` (MMU-less / FDPIC ELF variant — covered separately as `fs/binfmt-elf-fdpic.md` Tier-3 if expanded)
- `fs/binfmt_misc.c` (user-registered interpreters; separate)
- `fs/binfmt_script.c` (#! shebang; separate)
- `fs/exec.c` (do_execveat, copy_strings, search_binary_handler, begin_new_exec/setup_new_exec/finalize_exec — covered in `fs/exec-binfmt.md` Tier-3)
- `fs/coredump.c` and `elf_core_dump()` (covered in `fs/coredump.md` Tier-3)
- vDSO image + signal trampoline (per-arch, covered separately)
- `kernel/sys.c` brk/sbrk syscalls (covered in `mm/page-allocator.md` / a future `mm/brk.md` if expanded)
- `mm/mmap.c` vm_mmap / vm_munmap (covered in `mm/mmap.md` Tier-3)
- Implementation code
