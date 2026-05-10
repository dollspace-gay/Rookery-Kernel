# Tier-3: fs/coredump.c — Core-dump generation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/coredump.c (~1788 lines)
  - include/linux/coredump.h (struct coredump_params, dump_emit/dump_skip/dump_align prototypes)
  - include/linux/sched/coredump.h (MMF_DUMPABLE / MMF_DUMP_* / __mm_flags_get_dumpable)
  - include/uapi/linux/elfcore.h (NT_PRSTATUS / NT_PRPSINFO / NT_AUXV / NT_FILE / NT_X86_XSTATE / ...)
  - fs/binfmt_elf.c (elf_core_dump — Tier-3 covers caller-side; payload format in `binfmt-elf.md`)
  - Documentation/admin-guide/sysctl/kernel.rst (kernel.core_pattern / core_uses_pid / core_pipe_limit)
-->

## Summary

When a process receives a fatal signal with `dump_able` set, the kernel synthesizes a **core image** of its address space + thread state and routes it to one of four sinks: a regular file (default), a usermode-helper pipe, a Unix-domain socket, or a Unix-domain socket-request (one-shot connection per dump). Per-`vfs_coredump(siginfo)` snapshots `mm.flags` (MMF_DUMP* mask), prepares a fresh `struct cred`, zaps sibling threads and waits for them to become inactive (`coredump_wait` → `zap_threads`), parses `core_pattern` into a destination filename or argv (`coredump_parse` — formerly `format_corename` — expanding `%p/%P/%i/%I/%u/%g/%d/%s/%t/%h/%e/%f/%E/%c/%C/%F/%%`), opens the sink, hands off to the binfmt's `core_dump()` op (typically `elf_core_dump`), then waits for the helper pipe/socket to consume the bytes. Per-`coredump_filter` mask selects which VMA classes are written: anonymous-private, anonymous-shared, file-backed-private, file-backed-shared, ELF-headers, hugetlb-private, hugetlb-shared, dax-private, dax-shared. RLIMIT_CORE caps output size. Critical for: post-mortem debugging (gdb / lldb / ABRT), crash-reporting daemons (systemd-coredump, apport), production-incident response.

This Tier-3 covers `fs/coredump.c` (~1788 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct coredump_params` | per-dump context (siginfo, limit, mm_flags, file, vma_meta, cpu, pos, ...) | `CoredumpParams` |
| `struct core_name` | per-pattern expansion buffer + sink type | `CoreName` |
| `struct core_state` / `core_thread` / `dumper` | per-zap thread-rendezvous | `CoreState` |
| `struct core_vma_metadata` | per-VMA snapshot record | `CoreVmaMetadata` |
| `vfs_coredump()` | per-fatal-signal entry | `Coredump::vfs_entry` |
| `do_coredump()` | per-sink dispatch | `Coredump::do_dump` |
| `coredump_parse()` | per-`core_pattern` → corename / argv | `Coredump::parse_pattern` |
| `cn_printf()` / `cn_vprintf()` / `cn_esc_printf()` | per-corename printf helpers | `Coredump::cn_printf` / `cn_vprintf` / `cn_esc_printf` |
| `cn_print_exe_file()` | per-`%f`/`%E` exe path | `Coredump::print_exe_file` |
| `expand_corename()` | per-buffer grow | `Coredump::expand_corename` |
| `coredump_file()` | per-FILE sink open | `Coredump::open_file_sink` |
| `coredump_pipe()` | per-PIPE sink (umh) | `Coredump::open_pipe_sink` |
| `coredump_socket()` | per-SOCK / SOCK_REQ sink | `Coredump::open_socket_sink` |
| `coredump_sock_connect()` / `_send()` / `_recv()` / `_mark()` / `_wait()` / `_shutdown()` | per-AF_UNIX coredump protocol | `Coredump::sock_*` |
| `coredump_write()` | per-binfmt invocation | `Coredump::write` |
| `coredump_wait()` | per-thread-zap wait | `Coredump::wait_threads` |
| `zap_threads()` / `zap_process()` | per-SIGKILL siblings | `Coredump::zap_threads` / `zap_process` |
| `coredump_finish()` | per-post-dump signal-state restore | `Coredump::finish` |
| `coredump_cleanup()` | per-buffer free | `Coredump::cleanup` |
| `wait_for_dump_helpers()` | per-PIPE drain wait | `Coredump::wait_for_helpers` |
| `umh_coredump_setup()` | per-PIPE usermode-helper env setup | `Coredump::umh_setup` |
| `dump_interrupted()` | per-write-interrupt check (fatal-signal/freeze) | `Coredump::dump_interrupted` |
| `__dump_emit()` / `dump_emit()` | per-byte-range emit | `Coredump::dump_emit_inner` / `dump_emit` |
| `__dump_skip()` / `dump_skip()` / `dump_skip_to()` | per-hole skip (lseek or zero-fill) | `Coredump::dump_skip` / `dump_skip_to` |
| `dump_emit_page()` | per-page emit (handles HW poison) | `Coredump::dump_emit_page` |
| `dump_page_copy()` | per-page copy (with poison cleanup) | `Coredump::dump_page_copy` |
| `dump_user_range()` | per-user-VA emit | `Coredump::dump_user_range` |
| `dump_align()` | per-padding to alignment | `Coredump::dump_align` |
| `validate_coredump_safety()` | per-suid_dumpable=2 / core_pattern check | `Coredump::validate_safety` |
| `always_dump_vma()` | per-VMA force-dump (vDSO / gate_vma / named) | `Coredump::always_dump_vma` |
| `vma_dump_size()` | per-VMA byte count from MMF_DUMP* | `Coredump::vma_dump_size` |
| `dump_vma_snapshot()` / `free_vma_snapshot()` | per-mm snapshot/release | `Coredump::vma_snapshot` / `free_vma_snapshot` |
| `coredump_next_vma()` | per-iter VMA + gate_vma tail | `Coredump::next_vma` |
| `cmp_vma_size()` | per-`core_sort_vma` ordering | `Coredump::cmp_vma_size` |
| `proc_dostring_coredump()` | per-sysctl write hook | `Coredump::proc_dostring` |
| `validate_coredump_safety()` | per-sysctl post-write validation | `Coredump::validate_safety` |
| `core_pattern` / `core_uses_pid` / `core_pipe_limit` / `core_sort_vma` / `core_file_note_size_limit` | per-system sysctls | shared |
| `MMF_DUMPABLE` / `MMF_DUMP_*` flags | per-mm dump policy | shared |
| `core_pipe_count` (atomic_t) | per-system pipe-pressure counter | shared |

## Compatibility contract

REQ-1: struct coredump_params:
- siginfo: per-fatal kernel_siginfo_t (signo, code, sender pid/uid).
- limit: per-RLIMIT_CORE (max bytes; 0 disables).
- mm_flags: per-mm flags snapshot at entry (MMF_DUMP* mask) — captured under no-lock and treated as constant for the dump.
- file: per-sink struct file* (regular file OR pipe OR socket).
- pid: per-pidfd-install task_pid_struct.
- vma_meta: per-VMA snapshot array.
- vma_count: count.
- vma_data_size: total dumpable bytes.
- written: bytes written so far (vs. limit).
- pos: per-dump-file position (post-skip).
- to_skip: per-pending hole to skip on next emit.
- cpu: per-raw_smp_processor_id at entry (for `%C`).

REQ-2: struct core_name:
- corename: per-malloc'd output buffer.
- used: per-current length.
- size: per-allocated capacity.
- mask: bitmask COREDUMP_KERNEL | COREDUMP_USERSPACE | COREDUMP_WAIT | COREDUMP_REJECT.
- core_type: COREDUMP_FILE | COREDUMP_PIPE | COREDUMP_SOCK | COREDUMP_SOCK_REQ.
- core_pipe_limit: per-system reservation count for pipe-rate-limit.
- core_dumped: per-cleanup completion flag.

REQ-3: vfs_coredump(siginfo) — entry:
- argv = NULL (kfree-on-scope-exit).
- cprm.siginfo = siginfo; cprm.limit = rlimit(RLIMIT_CORE).
- cprm.mm_flags = __mm_flags_get_dumpable(mm).
- cprm.vma_meta = NULL.
- cprm.cpu = raw_smp_processor_id().
- audit_core_dumps(signo).
- if coredump_skip(&cprm, binfmt): return.
- prepare_creds(cred) — alloc transient creds; if !cred: return.
- if coredump_force_suid_safe(&cprm): cred.fsuid = GLOBAL_ROOT_UID.
- if coredump_wait(signo, &core_state) < 0: return.
- scoped_with_creds(cred) do_coredump(&cn, &cprm, &argv, &argc, binfmt).
- coredump_cleanup(&cn, &cprm).

REQ-4: coredump_skip predicate:
- binfmt == NULL ∨ binfmt.core_dump == NULL: skip.
- !__get_dumpable(cprm.mm_flags) ∧ !is_setuid_dumpable: skip.
- !signal-default-coredump-action(siginfo.si_signo): skip (only SIGQUIT/ABRT/BUS/SEGV/FPE/ILL/SYS/TRAP/XCPU/XFSZ).

REQ-5: zap_process(signal, exit_code):
- signal.flags = SIGNAL_GROUP_EXIT.
- signal.group_exit_code = exit_code.
- signal.group_stop_count = 0.
- for_each_thread(signal, t):
  - task_clear_jobctl_pending(t, JOBCTL_PENDING_MASK).
  - if t != current ∧ !(t.flags & PF_POSTCOREDUMP):
    - sigaddset(&t.pending.signal, SIGKILL).
    - signal_wake_up(t, 1).
    - nr++.
- return nr (sibling count).

REQ-6: zap_threads(tsk, core_state, exit_code):
- spin_lock_irq(&tsk.sighand.siglock).
- if !(SIGNAL_GROUP_EXIT) ∧ !group_exec_task:
  - signal.core_state = core_state.
  - nr = zap_process(signal, exit_code).
  - clear_tsk_thread_flag(tsk, TIF_SIGPENDING).
  - tsk.flags |= PF_DUMPCORE.
  - atomic_set(&core_state.nr_threads, nr).
- spin_unlock_irq.
- return nr ≥ 0 OR -EAGAIN.

REQ-7: coredump_wait(exit_code, core_state):
- init_completion(&core_state.startup).
- core_state.dumper.task = current; dumper.next = NULL.
- core_waiters = zap_threads(current, core_state, exit_code).
- if core_waiters > 0:
  - wait_for_completion_state(&core_state.startup, TASK_UNINTERRUPTIBLE | TASK_FREEZABLE).
  - for ptr = dumper.next; ptr != NULL; ptr = ptr.next: wait_task_inactive(ptr.task, TASK_ANY).
- return core_waiters.

REQ-8: coredump_parse(cn, cprm, &argv, &argc) — `core_pattern` template expansion:
- mask = COREDUMP_KERNEL.
- if core_pipe_limit: mask |= COREDUMP_WAIT.
- First char of core_pattern:
  - '|' → core_type = COREDUMP_PIPE; allocate argv; advance pat_ptr; require non-empty remainder.
  - '@' → core_type = COREDUMP_SOCK (or COREDUMP_SOCK_REQ if "@@"); cn_printf path; require absolute, no spaces, no "..", strlen < UNIX_PATH_MAX; return after.
  - otherwise → core_type = COREDUMP_FILE.
- Walk `pat_ptr` char-by-char:
  - For PIPE: split on whitespace → record argv[argc++] = used; emit '\0'.
  - On '%X':
    - `%%` → '%'.
    - `%p` → task_tgid_vnr(current) (pid-namespace pid); set pid_in_pattern.
    - `%P` → task_tgid_nr(current) (init-ns pid).
    - `%i` → task_pid_vnr (namespace tid).
    - `%I` → task_pid_nr (init-ns tid).
    - `%u` → from_kuid(init_user_ns, cred.uid).
    - `%g` → from_kgid(init_user_ns, cred.gid).
    - `%d` → __get_dumpable(cprm.mm_flags).
    - `%s` → siginfo.si_signo.
    - `%t` → ktime_get_real_seconds().
    - `%h` → utsname().nodename (under uts_sem read).
    - `%e` → current.comm (escaped).
    - `%f` → exe-file basename (escaped).
    - `%E` → exe-file full path (escaped).
    - `%c` → rlimit(RLIMIT_CORE).
    - `%C` → cprm.cpu.
    - `%F` → COREDUMP_PIDFD_NUMBER (only for PIPE; installs pidfd into helper).
    - default → drop char.
  - Plain char → cn_printf("%c").
- Tail: if FILE ∧ !pid_in_pattern ∧ core_uses_pid: append ".<tgid>".
- Return false on any allocation/format failure.

REQ-9: do_coredump(cn, cprm, &argv, &argc, binfmt):
- trace_coredump(signo).
- if !coredump_parse(...): coredump_report_failure("format_corename failed"); return.
- switch cn.core_type:
  - COREDUMP_FILE: if !coredump_file(cn, cprm, binfmt): return.
  - COREDUMP_PIPE: if !coredump_pipe(cn, cprm, argv, argc): return.
  - COREDUMP_SOCK_REQ | COREDUMP_SOCK: if !coredump_socket(cn, cprm): return.
- if cn.mask & COREDUMP_REJECT: return.
- unshare_files() — if fails: return.
- if cn.mask & COREDUMP_KERNEL ∧ !coredump_write(cn, cprm, binfmt): return.
- coredump_sock_shutdown(cprm.file).
- if cn.mask & COREDUMP_USERSPACE: cn.core_dumped = true.
- if cn.mask & COREDUMP_WAIT:
  - PIPE → wait_for_dump_helpers(cprm.file).
  - SOCK / SOCK_REQ → coredump_sock_wait(cprm.file).

REQ-10: coredump_file (FILE sink):
- if cprm.limit < binfmt.min_coredump: return false.
- if coredump_force_suid_safe ∧ corename[0] != '/': return false.
- if !coredump_force_suid_safe: filename_unlinkat(AT_FDCWD, corename) — best-effort.
- open_flags = O_CREAT | O_WRONLY | O_NOFOLLOW | O_LARGEFILE | O_EXCL.
- if coredump_force_suid_safe: file_open_root(init_task.fs.root, corename, open_flags, 0600).
- else: filp_open(corename, open_flags, 0600).
- Validate: inode.i_nlink ≤ 1, !d_unhashed, S_ISREG, uid preserved (vfsuid==fsuid), mode==0600, FMODE_CAN_WRITE.
- do_truncate(...) to 0.
- cprm.file = file.

REQ-11: coredump_pipe (PIPE sink):
- if cprm.limit == 1: report failure ("RLIMIT_CORE is set to 1, aborting core"); return false. (Set by umh helper to break recursion.)
- if atomic_inc_return(&core_pipe_count) > core_pipe_limit ∧ core_pipe_limit > 0: report; return false.
- Build helper_argv from argv[] offsets into cn.corename buffer.
- call_usermodehelper_setup with umh_coredump_setup (sets RLIMIT_CORE=1, fsuid, optional pidfd).
- Spawn helper; cprm.file = pipe.

REQ-12: coredump_socket (SOCK / SOCK_REQ sink):
- Validate `core_pattern` socket-mode safety (validate_coredump_safety).
- coredump_sock_connect(cn, cprm) → AF_UNIX SOCK_STREAM connect to corename (or coredump_sock_request for SOCK_REQ).
- For SOCK_REQ: coredump_sock_send (struct coredump_req: siginfo signo, pid, mark) → coredump_sock_recv (struct coredump_ack) → on ACK, proceed; otherwise abort.
- cprm.file = sock_file.

REQ-13: coredump_write — binfmt invocation:
- Call binfmt.core_dump(cprm) — for ELF/ELF-FDPIC produces:
  - ELF header + program headers (PT_NOTE + PT_LOAD per dumpable VMA).
  - Notes segment (PT_NOTE) with:
    - NT_PRSTATUS (per-thread registers, signal, sigmask) — `struct elf_prstatus`.
    - NT_PRPSINFO (process info: state, pid, ppid, comm, args).
    - NT_SIGINFO (siginfo of fatal signal).
    - NT_AUXV (auxv array from mm.saved_auxv).
    - NT_FILE (file-backed VMA table: addr ranges + offsets + path strings).
    - NT_X86_XSTATE (FPU + AVX/AVX-512/MPX/PKRU state).
    - Optionally NT_PRFPREG, NT_X86_TLS, NT_X86_IOPERM, NT_ARM_VFP, NT_ARM_TLS, NT_ARM_HW_BREAK, NT_ARM_HW_WATCH, NT_ARM_SVE, NT_ARM_SSVE, NT_ARM_ZA, ...
    - File-backed module note from binfmt_elf (per-arch-specific).
  - PT_LOAD bodies: per-VMA `dump_user_range(cprm, vma.vm_start, dump_size)`.
- Return true on success; false if dump_interrupted or write failed.

REQ-14: vma_dump_size(vma, mm_flags) — MMF_DUMP_* mask decision:
- if always_dump_vma(vma): whole.
- if vma.vm_flags & VM_DONTDUMP: 0.
- if vma_is_dax(vma):
  - VM_SHARED ∧ FILTER(DAX_SHARED) → whole; !VM_SHARED ∧ FILTER(DAX_PRIVATE) → whole; else 0.
- if is_vm_hugetlb_page(vma):
  - VM_SHARED ∧ FILTER(HUGETLB_SHARED) → whole; !VM_SHARED ∧ FILTER(HUGETLB_PRIVATE) → whole; else 0.
- if VM_IO: 0.
- if VM_SHARED:
  - i_nlink == 0 (anonymous shared) ? FILTER(ANON_SHARED) : FILTER(MAPPED_SHARED) → whole; else 0.
- if (!MMU ∨ anon_vma) ∧ FILTER(ANON_PRIVATE): whole.
- if vm_file == NULL: 0.
- if FILTER(MAPPED_PRIVATE): whole.
- if FILTER(ELF_HEADERS) ∧ vm_pgoff == 0 ∧ VM_READ:
  - if executable bits in i_mode: PAGE_SIZE.
  - else: DUMP_SIZE_MAYBE_ELFHDR_PLACEHOLDER (1, fixed up later by dump_vma_snapshot reading the actual first bytes).
- else: 0.

REQ-15: dump_emit / dump_skip / dump_align — primitives used by binfmt:
- dump_emit(cprm, addr, nr) → __kernel_write to cprm.file; honors cprm.to_skip pending; cprm.written += nr; check cprm.written ≤ cprm.limit; honors dump_interrupted (fatal_signal_pending OR freezing).
- dump_skip / dump_skip_to → if FMODE_LSEEK: vfs_llseek; else zero-fill in PAGE_SIZE chunks.
- dump_align(cprm, align) → cprm.to_skip += round_up(cprm.pos, align) - cprm.pos.
- dump_user_range(cprm, start, len) → copy user-space pages out (handles HW-poisoned pages via dump_page_copy zero-fill).

REQ-16: validate_coredump_safety (sysctl gate):
- If fs.suid_dumpable == 2 ∧ core_pattern[0] != '/' ∧ '|' ∧ '@': warn ("Unsafe core_pattern used with fs.suid_dumpable=2").
- For SOCK pattern: must be absolute path under /; second char must not be '@' (reserved); no name_contains_dotdot.

REQ-17: Sysctls (kernel.*):
- core_pattern (string, max CORENAME_MAX_SIZE; proc_dostring_coredump → validate_coredump_safety post-write).
- core_uses_pid (int): if FILE pattern lacks %p, append ".<tgid>".
- core_pipe_limit (uint): rate-limit concurrent pipe dumps; 0 = unlimited; ENABLES coredump_wait.
- core_sort_vma (uint): if non-zero, sort VMAs by size descending (cmp_vma_size) so the most useful pages are written first within the limit.
- core_file_note_size_limit (uint): per-note size cap (default CORE_FILE_NOTE_SIZE_DEFAULT=4MB; max CORE_FILE_NOTE_SIZE_MAX=16MB).

REQ-18: MMF_DUMP_* (coredump_filter) bits — selectable via `/proc/self/coredump_filter`:
- ANON_PRIVATE  (0): anonymous private memory.
- ANON_SHARED   (1): anonymous shared memory.
- MAPPED_PRIVATE (2): file-backed private mappings.
- MAPPED_SHARED (3): file-backed shared mappings.
- ELF_HEADERS   (4): ELF header pages of file-backed executables (1 page).
- HUGETLB_PRIVATE (5): hugetlbfs private.
- HUGETLB_SHARED  (6): hugetlbfs shared.
- DAX_PRIVATE    (7): DAX private.
- DAX_SHARED     (8): DAX shared.

## Acceptance Criteria

- [ ] AC-1: Fatal SIGSEGV on dumpable task: vfs_coredump invoked; SIGKILL fans out to siblings; siblings parked inactive before binfmt.core_dump runs.
- [ ] AC-2: core_pattern == "core.%p.%t" + dumpable → file `core.<pid>.<unix-time>` mode 0600 created with content matching binfmt expectations.
- [ ] AC-3: core_pattern starting with '|' invokes usermode helper; helper receives core image on stdin; helper exits → wait_for_dump_helpers returns.
- [ ] AC-4: core_pattern starting with '@' connects AF_UNIX socket; SOCK_REQ ("@@/path") performs request/ack handshake.
- [ ] AC-5: RLIMIT_CORE = 0: coredump_skip OR coredump_file refuses (limit < min_coredump).
- [ ] AC-6: RLIMIT_CORE truncates output exactly at the byte cap; subsequent dump_emit returns 0 without filesystem error.
- [ ] AC-7: coredump_filter cleared except for ANON_PRIVATE: only anonymous private VMAs included in PT_LOADs.
- [ ] AC-8: fs.suid_dumpable=2 + relative core_pattern: warning logged + dump refused (force_suid_safe path).
- [ ] AC-9: core_pipe_limit=N: (N+1)-th concurrent pipe dump declined (core_pipe_count over limit).
- [ ] AC-10: core_uses_pid=1 + pattern without %p (FILE): ".<tgid>" appended.
- [ ] AC-11: Process tree with shared mm: zap_threads delivers SIGKILL to all PF_POSTCOREDUMP==0 threads; PF_DUMPCORE set on the dumping task.
- [ ] AC-12: Signal during dump (dump_interrupted): __dump_emit returns 0; binfmt aborts cleanly; partial file remains.
- [ ] AC-13: core_sort_vma=1: VMAs written in size-descending order so the most data fits under RLIMIT_CORE.
- [ ] AC-14: NT_FILE note enumerates every file-backed VMA with (start, end, pgoff, path).
- [ ] AC-15: Coredump from setuid binary with suid_dumpable=0: skipped silently.

## Architecture

```
struct CoredumpParams {
  siginfo: *KernelSiginfo,
  limit: u64,
  mm_flags: u64,
  file: Option<*File>,
  pid: Option<*PidStruct>,
  vma_meta: Option<*[CoreVmaMetadata]>,
  vma_count: usize,
  vma_data_size: u64,
  written: u64,
  pos: u64,
  to_skip: u64,
  cpu: i32,
}

struct CoreName {
  corename: *mut u8,
  used: u32,
  size: u32,
  mask: u32,                     // COREDUMP_KERNEL | _USERSPACE | _WAIT | _REJECT
  core_type: CoreType,           // File | Pipe | Sock | SockReq
  core_pipe_limit: u32,
  core_dumped: bool,
}

enum CoreType { File, Pipe, Sock, SockReq }

struct CoreVmaMetadata {
  start: u64,
  end: u64,
  flags: u64,                    // VMA flags snapshot
  pgoff: u64,
  file: Option<*File>,
  dump_size: u64,                // from vma_dump_size
}

struct CoreState {
  startup: Completion,
  dumper: CoreThread,
  nr_threads: AtomicI32,
}

struct CoreThread {
  task: *TaskStruct,
  next: Option<*CoreThread>,
}

const MMF_DUMP_ANON_PRIVATE: u32 = 0;
const MMF_DUMP_ANON_SHARED: u32 = 1;
const MMF_DUMP_MAPPED_PRIVATE: u32 = 2;
const MMF_DUMP_MAPPED_SHARED: u32 = 3;
const MMF_DUMP_ELF_HEADERS: u32 = 4;
const MMF_DUMP_HUGETLB_PRIVATE: u32 = 5;
const MMF_DUMP_HUGETLB_SHARED: u32 = 6;
const MMF_DUMP_DAX_PRIVATE: u32 = 7;
const MMF_DUMP_DAX_SHARED: u32 = 8;
```

`Coredump::vfs_entry(siginfo)`:
1. argv = NULL (drop-on-scope-exit).
2. cprm = CoredumpParams { siginfo, limit: rlimit(RLIMIT_CORE), mm_flags: __mm_flags_get_dumpable(current.mm), vma_meta: None, cpu: raw_smp_processor_id(), ... }.
3. audit_core_dumps(siginfo.si_signo).
4. if Coredump::skip(&cprm, current.mm.binfmt): return.
5. cred = prepare_creds(); if !cred: return.
6. if Coredump::force_suid_safe(&cprm): cred.fsuid = GLOBAL_ROOT_UID.
7. if Coredump::wait_threads(siginfo.si_signo, &core_state) < 0: return.
8. with_creds(cred): Coredump::do_dump(&cn, &cprm, &argv, &argc, current.mm.binfmt).
9. Coredump::cleanup(&cn, &cprm).

`Coredump::do_dump(cn, cprm, &argv, &argc, binfmt)`:
1. trace_coredump(cprm.siginfo.si_signo).
2. if !Coredump::parse_pattern(cn, cprm, &argv, &argc): report-failure; return.
3. match cn.core_type:
   - File => if !Coredump::open_file_sink(cn, cprm, binfmt): return,
   - Pipe => if !Coredump::open_pipe_sink(cn, cprm, argv, argc): return,
   - Sock | SockReq => if !Coredump::open_socket_sink(cn, cprm): return.
4. if cn.mask & COREDUMP_REJECT: return.
5. unshare_files().or_return().
6. if cn.mask & COREDUMP_KERNEL ∧ !Coredump::write(cn, cprm, binfmt): return.
7. Coredump::sock_shutdown(cprm.file).
8. if cn.mask & COREDUMP_USERSPACE: cn.core_dumped = true.
9. if cn.mask & COREDUMP_WAIT:
   - Pipe => Coredump::wait_for_helpers(cprm.file),
   - Sock | SockReq => Coredump::sock_wait(cprm.file),
   - File => skip.

`Coredump::parse_pattern(cn, cprm, &argv, &argc) -> bool`:
1. cn.mask = COREDUMP_KERNEL; if core_pipe_limit > 0: cn.mask |= COREDUMP_WAIT.
2. cn.used = 0; cn.corename = NULL; cn.core_pipe_limit = 0; cn.core_dumped = false.
3. First char of core_pattern:
   - '|' → core_type = Pipe; advance.
   - '@' → core_type = Sock (advance once); if next is '@': core_type = SockReq (advance again).
   - else → core_type = File.
4. Coredump::expand_corename(cn, core_name_size).
5. cn.corename[0] = '\0'.
6. match core_type:
   - Pipe: alloc argv[sizeof core_pattern / 2]; argv[argc++] = 0; require non-empty remainder.
   - Sock | SockReq: cn_printf("%s", remainder); require corename[0] == '/' ∧ no spaces ∧ no ".." ∧ strlen < UNIX_PATH_MAX; return true.
   - File: nothing extra.
7. While *pat_ptr:
   - If Pipe ∧ whitespace: if used > 0: was_space = true; advance; continue.
   - If was_space: was_space = false; cn_printf("\\0"); argv[argc++] = cn.used.
   - If *pat_ptr != '%': cn_printf("%c", *pat_ptr++).
   - Else: pat_ptr += 1; match *pat_ptr:
     - 0 (lone trailing %): break.
     - '%' → cn_printf("%").
     - 'p' → cn_printf("%d", task_tgid_vnr(current)); pid_in_pattern = true.
     - 'P' → task_tgid_nr.
     - 'i' → task_pid_vnr.
     - 'I' → task_pid_nr.
     - 'u' → from_kuid(init_user_ns, cred.uid).
     - 'g' → from_kgid(init_user_ns, cred.gid).
     - 'd' → __get_dumpable(cprm.mm_flags).
     - 's' → cprm.siginfo.si_signo.
     - 't' → ktime_get_real_seconds.
     - 'h' → utsname().nodename (under uts_sem).
     - 'e' → current.comm (escaped).
     - 'f' → Coredump::print_exe_file(cn, name_only=true).
     - 'E' → Coredump::print_exe_file(cn, name_only=false).
     - 'c' → rlimit(RLIMIT_CORE).
     - 'C' → cprm.cpu.
     - 'F' → if core_type == Pipe: cprm.pid = task_tgid(current); cn_printf("%d", COREDUMP_PIDFD_NUMBER).
     - default → drop.
8. /* out: */
9. if core_type == File ∧ !pid_in_pattern ∧ core_uses_pid: cn_printf(".%d", task_tgid_vnr(current)).
10. return true.

`Coredump::zap_threads(tsk, core_state, exit_code) -> i32`:
1. spin_lock_irq(&tsk.sighand.siglock).
2. if !(tsk.signal.flags & SIGNAL_GROUP_EXIT) ∧ !tsk.signal.group_exec_task:
   - tsk.signal.core_state = core_state.
   - nr = Coredump::zap_process(tsk.signal, exit_code).
   - clear_tsk_thread_flag(tsk, TIF_SIGPENDING).
   - tsk.flags |= PF_DUMPCORE.
   - atomic_set(&core_state.nr_threads, nr).
3. else: nr = -EAGAIN.
4. spin_unlock_irq.
5. return nr.

`Coredump::vma_snapshot(cprm) -> bool`:
1. /* Walk mm under mmap_read_lock; snapshot all VMAs */
2. vma_count = 0; vma_data_size = 0.
3. for vma in mm.mm_vmas() + gate_vma:
   - dump_size = Coredump::vma_dump_size(vma, cprm.mm_flags).
   - if dump_size > 0:
     - Record CoreVmaMetadata { start, end, flags, pgoff, file (refcounted), dump_size }.
     - vma_count += 1; vma_data_size += dump_size.
4. /* Fix up ELF_HEADERS placeholder: read first bytes; verify ELF magic; convert PLACEHOLDER → PAGE_SIZE OR 0 */.
5. if core_sort_vma: sort cprm.vma_meta by dump_size descending.
6. mmap_read_unlock.
7. cprm.vma_meta = Some(vma_meta); cprm.vma_count = vma_count.
8. return true.

`Coredump::dump_emit_inner(cprm, addr, nr) -> bool`:
1. if cprm.written + nr > cprm.limit: return false (silent stop).
2. if Coredump::dump_interrupted(): return false.
3. pos = cprm.file.f_pos.
4. n = __kernel_write(cprm.file, addr, nr, &pos).
5. if n != nr: return false.
6. cprm.file.f_pos = pos.
7. cprm.written += n; cprm.pos += n.
8. return true.

`Coredump::dump_emit(cprm, addr, nr)`:
1. if cprm.to_skip > 0:
   - if !Coredump::dump_skip(cprm, cprm.to_skip): return false.
   - cprm.to_skip = 0.
2. return Coredump::dump_emit_inner(cprm, addr, nr).

`Coredump::vma_dump_size(vma, mm_flags) -> u64` (see REQ-14 for full predicate).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pattern_buffer_bounded` | INVARIANT | per-parse_pattern: cn.used ≤ cn.size after every cn_printf; no buffer-overflow. |
| `socket_path_absolute` | INVARIANT | per-Sock/SockReq parse: corename[0] == '/' ∧ strlen < UNIX_PATH_MAX ∧ no "..". |
| `pipe_pidfd_only_in_pipe` | INVARIANT | per-%F: cprm.pid installed only when core_type == Pipe. |
| `dump_emit_respects_limit` | INVARIANT | per-dump_emit: cprm.written + nr ≤ cprm.limit OR returns 0. |
| `zap_threads_excludes_current` | INVARIANT | per-zap_process: SIGKILL not posted to current; PF_POSTCOREDUMP threads skipped. |
| `vma_dump_size_zero_for_dontdump` | INVARIANT | per-vma_dump_size: VM_DONTDUMP ⟹ 0; VM_IO ⟹ 0. |
| `mm_flags_constant_during_dump` | INVARIANT | per-vfs_coredump: cprm.mm_flags captured once at entry; never re-read. |

### Layer 2: TLA+

`fs/coredump.tla`:
- Per-vfs_coredump entry, per-zap_threads, per-coredump_wait rendezvous, per-do_coredump dispatch, per-binfmt write, per-coredump_finish.
- Properties:
  - `safety_no_double_dump` — per-tsk.signal.flags & SIGNAL_GROUP_EXIT prevents two concurrent vfs_coredumps in the same group.
  - `safety_siblings_inactive_before_write` — per-coredump_wait: all sibling threads with PF_POSTCOREDUMP==0 are TASK_INACTIVE before binfmt.core_dump runs.
  - `safety_no_write_after_signal` — per-dump_interrupted: fatal_signal_pending ⟹ subsequent dump_emit returns 0.
  - `safety_pipe_quota_bounded` — per-coredump_pipe: core_pipe_count ≤ core_pipe_limit (when limit > 0).
  - `safety_corename_size_monotone` — per-cn_printf: cn.used only increases; expand_corename only grows.
  - `liveness_helper_wait_terminates` — per-PIPE: wait_for_dump_helpers eventually returns (helper exit OR pipe-close).
  - `liveness_dump_terminates` — per-vfs_coredump: do_coredump returns within bounded steps modulo dump_interrupted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Coredump::vfs_entry` post: SIGNAL_GROUP_EXIT set; siblings either SIGKILL'd or PF_POSTCOREDUMP | `Coredump::vfs_entry` |
| `Coredump::parse_pattern` post: cn.corename NUL-terminated; argv[i] are valid offsets into cn.corename | `Coredump::parse_pattern` |
| `Coredump::open_file_sink` post: cprm.file is regular S_ISREG, mode 0600, owner==current_fsuid | `Coredump::open_file_sink` |
| `Coredump::open_pipe_sink` post: cprm.file is pipe; helper spawned with RLIMIT_CORE=1 | `Coredump::open_pipe_sink` |
| `Coredump::dump_emit` post: ret==true ⟹ written += nr ≤ limit; ret==false ⟹ no partial bytes counted | `Coredump::dump_emit` |
| `Coredump::vma_dump_size` post: VM_DONTDUMP/VM_IO returns 0; mask-disabled class returns 0 | `Coredump::vma_dump_size` |
| `Coredump::wait_threads` post: nr_threads == count of zapped siblings; current.flags has PF_DUMPCORE | `Coredump::wait_threads` |

### Layer 4: Verus/Creusot functional

`Per-fatal-signal sequence: signal-default-action(core) → vfs_coredump → coredump_wait → parse_pattern → open_sink → binfmt.core_dump (ELF: header + PT_NOTEs (NT_PRSTATUS/PRPSINFO/AUXV/FILE/X86_XSTATE) + PT_LOADs) → coredump_finish` semantic equivalence: per-`core(5)` man-page, `Documentation/admin-guide/sysctl/kernel.rst` core_pattern semantics, and gdb's core-file reader expectations.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

Coredump reinforcement:

- **Per-O_NOFOLLOW | O_EXCL on FILE sink** — defense against per-symlink-attack writing core into a privileged location.
- **Per-S_ISREG + mode==0600 + owner==fsuid post-open validation** — defense against per-race-substitution between unlink and create.
- **Per-fs.suid_dumpable=2 forces absolute path OR pipe/socket** — defense against per-suid-binary dumping to user-controlled relative path.
- **Per-RLIMIT_CORE strict (dump_emit refuses past limit)** — defense against per-coredump filling filesystem.
- **Per-core_pipe_limit caps concurrent helper invocations** — defense against per-fork-bomb crash storm spawning unlimited helpers.
- **Per-RLIMIT_CORE=1 sentinel from umh_coredump_setup** — defense against per-recursive-coredump (helper crashes → helper handles it).
- **Per-name_contains_dotdot rejected in core_pattern (FILE/SOCK)** — defense against per-path-traversal.
- **Per-pidfd installed only for PIPE (`%F`)** — defense against per-pidfd leaked into unrelated process.
- **Per-cprm.mm_flags snapshot at entry; never re-read** — defense against per-MMF_DUMP-toggle-mid-dump producing inconsistent VMA selection.
- **Per-cn_esc_printf escapes shell-meta in %e/%f/%E/%h** — defense against per-comm-injection into pipe helper argv.
- **Per-COREDUMP_KERNEL/USERSPACE/WAIT/REJECT mask honored** — defense against per-LSM-veto bypass.
- **Per-zap_threads atomic under siglock** — defense against per-thread-spawn-race during zap.
- **Per-wait_task_inactive(TASK_ANY) on every dumper-next** — defense against per-live-thread mutating mm during dump_user_range.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- ELF / ELF-FDPIC payload format (covered in `binfmt-elf.md` Tier-3)
- Per-arch register-set notes (covered in `arch/*/coredump.md` Tier-3s if expanded)
- usermode-helper framework (covered in `kernel/umh.md` Tier-3)
- AF_UNIX socket plumbing (covered in `net/af-unix.md` Tier-3)
- systemd-coredump / apport user-space handlers (out of kernel scope)
- /proc/PID/coredump_filter implementation surface (covered in `proc/pid.md` Tier-3)
- Signal delivery / get_signal core-action selection (covered in `kernel/signal.md` Tier-3)
- Implementation code
