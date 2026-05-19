---
title: "Tier-5 syscall: acct(2) — syscall 163"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`acct(2)` enables or disables BSD-style process accounting. When enabled with a pathname, the kernel appends a fixed-size `struct acct_v3` record to the named file every time a process exits, containing CPU time, memory footprint, IO counters, exit status, command name, and credentials. Passing `NULL` disables accounting. The file must be on a writable, non-read-only filesystem and reside in caller's mount namespace. Critical for: legacy `psacct`/`acct` tools, regulatory audit baselines, batch-job accounting on HPC clusters.

Process accounting has fallen out of common use (superseded by `audit(7)`, `bpf` exit hooks, cgroup-v2 stats) but remains in POSIX and is occasionally required for compliance regimes (e.g. orange-book C2-class).

### Acceptance Criteria

- [ ] AC-1: Root: `acct("/var/log/acct")` returns 0; subsequent process exit appends an `acct_v3` record.
- [ ] AC-2: `acct(NULL)` after enabling returns 0; further exits do NOT append.
- [ ] AC-3: Non-root: -EPERM.
- [ ] AC-4: `acct("/tmp/acct")` on RO fs: -EROFS.
- [ ] AC-5: `acct("/tmp/dir")` directory: -EISDIR or -EACCES.
- [ ] AC-6: `acct("/nonexistent")`: -ENOENT.
- [ ] AC-7: Filesystem space below ACCT_PARM_LOW: acct auto-pauses; rises above HIGH: resumes.
- [ ] AC-8: pidns child: setting acct in child does not affect parent's pidns acct file.
- [ ] AC-9: Build without CONFIG_BSD_PROCESS_ACCT: -ENOSYS.
- [ ] AC-10: Each record contains valid `ac_comm`, `ac_uid`, `ac_pid`, monotonic `ac_etime`.

### Architecture

```rust
#[syscall(nr = 163, abi = "sysv")]
pub fn sys_acct(filename: UserPtr<u8>) -> isize {
    Acct::do_acct(filename)
}
```

`Acct::do_acct(filename) -> isize`:
1. if !cfg!(CONFIG_BSD_PROCESS_ACCT) { return -ENOSYS; }
2. if !ns_capable(current_user_ns(), CAP_SYS_PACCT) { return -EPERM; }
3. if filename.is_null() {
4.   Acct::close_for_pidns(current_pidns());
5.   return 0;
6. }
7. let name = getname(filename)?;
8. let f = filp_open(&name, O_WRONLY | O_APPEND | O_LARGEFILE, 0)?;
9. Acct::install(current_pidns(), f)

`Acct::install(pidns, file) -> isize`:
1. let inode = file.inode();
2. if !S_ISREG(inode.mode) { filp_close(file); return -EACCES; }
3. if inode.sb().is_read_only() { filp_close(file); return -EROFS; }
4. if file.f_op.write_iter.is_none() { filp_close(file); return -EIO; }
5. let _g = pidns.acct_mutex.lock();
6. let old = pidns.bacct.replace(file);
7. if let Some(prev) = old { filp_close(prev); }
8. return 0;

`Acct::process(task)` (invoked from do_exit):
1. let pidns = task.pid_namespace();
2. let _g = pidns.acct_mutex.lock();
3. let Some(file) = pidns.bacct.clone() else { return; };
4. if !Acct::check_free_space(&file) { return; }
5. let rec = Acct::build_acct_v3(task);
6. let _ = __kernel_write(&file, rec.as_bytes(), &mut file.f_pos);

`Acct::build_acct_v3(task) -> AcctV3`:
1. AcctV3 {
2.   ac_flag: (if task.flags & PF_FORKNOEXEC { AFORK } else { 0 }) | (if task.signal.flags & SIGNAL_GROUP_COREDUMP { ACORE } else { 0 }) | ...,
3.   ac_version: 3,
4.   ac_uid: from_kuid_munged(init_user_ns, task.real_cred.uid),
5.   ac_gid: from_kgid_munged(init_user_ns, task.real_cred.gid),
6.   ac_pid: task_pid_vnr(task),
7.   ac_ppid: task_pid_vnr(rcu_dereference(task.real_parent)),
8.   ac_btime: get_seconds() - task.start_time_sec(),
9.   ac_etime: encode_comp_t(task.elapsed_jiffies()),
10.  ac_utime: encode_comp_t(task.utime_jiffies()),
11.  ac_stime: encode_comp_t(task.stime_jiffies()),
12.  ac_mem: encode_comp_t(task.rss_high_water()),
13.  ac_io: encode_comp_t(task.ioac.rchar + task.ioac.wchar),
14.  ac_rw: encode_comp_t(task.ioac.read_bytes + task.ioac.write_bytes),
15.  ac_minflt: encode_comp_t(task.min_flt),
16.  ac_majflt: encode_comp_t(task.maj_flt),
17.  ac_swaps: 0,
18.  ac_exitcode: task.exit_code,
19.  ac_comm: task.comm,
20.  ac_tty: task.signal.tty.dev() as u32,
21. }

### Out of Scope

- `audit(7)` modern auditing (covered in `audit.md`).
- cgroup-v2 stats (covered in cgroup Tier-3).
- bpf exit hooks (covered in `bpf.md`).
- Implementation code.

### signature

```c
int acct(const char *filename);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `filename` | `const char *` | in | Path to accounting file. `NULL` disables accounting. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success (enabled or disabled). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Path component permission denial, OR target not a regular file, OR target opened on read-only fs. |
| `EFAULT` | `filename` not readable. |
| `EIO` | Write error on accounting file. |
| `EISDIR` | Target is a directory. |
| `ELOOP` | Symlink loop. |
| `ENAMETOOLONG` | Path > PATH_MAX. |
| `ENFILE` / `EMFILE` | System-wide / per-process fd table full. |
| `ENOENT` | Path component missing. |
| `ENOMEM` | Out of memory. |
| `ENOSYS` | Kernel built without CONFIG_BSD_PROCESS_ACCT. |
| `ENOTDIR` | Non-directory in path. |
| `EPERM` | Caller lacks `CAP_SYS_PACCT` in current userns. |
| `EROFS` | Read-only filesystem. |
| `EUSERS` | No free accounting slots (single-pernet-acct limit). |

### abi surface

```text
__NR_acct  (x86_64) = 163
__NR_acct  (arm64)  = 89
__NR_acct  (riscv)  = 89
__NR_acct  (i386)   = 51

/* struct acct_v3 is fixed-layout (64 bytes); kernel writes one per exit. */
/* Per-pidns: each pid_namespace has its own active acct file pointer. */
```

### compatibility contract

REQ-1: Syscall number is **163** on x86_64. ABI-stable.

REQ-2: Capability gating: `ns_capable(current_user_ns(), CAP_SYS_PACCT)`. Else `-EPERM`.

REQ-3: `filename == NULL`: disable accounting for current pid namespace; close the previously-installed accounting file. Return 0.

REQ-4: `filename != NULL`:
- `getname(filename)` (copy + path-length check).
- `filp_open(name, O_WRONLY | O_APPEND | O_LARGEFILE, 0)`.
- Validate target:
  - Regular file (`S_ISREG`); else `-EACCES`.
  - Filesystem writable (`!sb->s_readonly_remount`); else `-EROFS`.
  - File `f_op->write_iter` non-NULL; else `-EIO`.
- Lock the per-pidns acct mutex; replace any previous file pointer (close it).

REQ-5: Each task exit invokes `acct_process(task)`:
- If pidns acct file is set, allocate a stack/percpu `struct acct_v3`.
- Fill: `ac_uid`, `ac_gid`, `ac_pid`, `ac_ppid`, `ac_btime` (boot time), `ac_etime`, `ac_utime`, `ac_stime`, `ac_mem`, `ac_io`, `ac_rw`, `ac_minflt`, `ac_majflt`, `ac_swaps`, `ac_exitcode`, `ac_comm[16]`, `ac_tty`, `ac_flag` (AFORK/ASU/ACORE/AXSIG).
- Use `__kernel_write` to append; failures logged but do not fail the exit.

REQ-6: Per-pidns isolation: `pid_namespace->bacct` holds the active file; child pidns may have its own; on pidns destruction, the file is closed.

REQ-7: Free-disk gating: `check_free_space()` runs periodically; if filesystem `bavail / btotal < acct_parm[ACCT_PARM_LOW]`, acct is paused. Resumes when `bavail / btotal > acct_parm[ACCT_PARM_HIGH]`.

REQ-8: `/proc/sys/kernel/acct` (sysctl): three values `high low frequency` — pause/resume thresholds (defaults 4 2 30).

REQ-9: When CONFIG_BSD_PROCESS_ACCT=n: syscall returns `-ENOSYS`.

REQ-10: Filename is a path in caller's mount namespace; subsequent mount-ns changes do not change which file is written (the open file handle is held).

REQ-11: 32-bit compat: same syscall, no struct compatibility issue (path string).

REQ-12: SECCOMP / unprivileged-container default profiles: typically block acct entirely (CAP_SYS_PACCT not granted to unprivileged containers).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_pacct_required` | INVARIANT | missing CAP_SYS_PACCT ⟹ EPERM, no file opened. |
| `null_disables` | INVARIANT | filename == NULL ⟹ close previous, no new install. |
| `regular_file_only` | INVARIANT | target not S_ISREG ⟹ EACCES; file closed. |
| `ro_fs_rejected` | INVARIANT | RO sb ⟹ EROFS; file closed. |
| `pidns_isolation` | INVARIANT | acct file scoped to current pidns; siblings unaffected. |
| `exit_record_size_fixed` | INVARIANT | each record is exactly sizeof(acct_v3) bytes. |

### Layer 2: TLA+

`kernel/acct.tla`:
- States: per-pidns acct file pointer, per-exit record append, per-fs free-space.
- Properties:
  - `safety_cap_gate` — no install without CAP_SYS_PACCT.
  - `safety_pidns_isolation` — pidns A acct unaffected by pidns B.
  - `safety_no_record_when_paused` — low-space pauses appends.
  - `safety_fixed_record_size` — record size invariant.
  - `liveness_exit_records_when_enabled` — task exit with acct enabled ⟹ one record.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_acct` post: success ⟹ pidns.bacct set or cleared as requested | `Acct::do_acct` |
| `do_acct` post: EPERM without CAP_SYS_PACCT | `Acct::do_acct` |
| `process` post: pidns.bacct set ⟹ append issued (modulo low-space) | `Acct::process` |
| `build_acct_v3` post: every field within type range; version == 3 | `Acct::build_acct_v3` |

### Layer 4: Verus/Creusot functional

Per-`acct(2)` man page + `psacct` tooling read of acct_v3 records semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`acct(2)` reinforcement:

- **Per-CAP_SYS_PACCT gate** — defense against per-unprivileged accounting hijack.
- **Per-S_ISREG check** — defense against per-target-is-device write amplification.
- **Per-RO-fs check** — defense against per-RO-fs futile write loop.
- **Per-pidns scoping** — defense against per-cross-pidns accounting interference.
- **Per-low-space auto-pause** — defense against per-acct-fills-fs DoS.
- **Per-exit append best-effort** — defense against per-write-failure blocking exit path.
- **Per-fixed record size** — defense against per-truncated-record corruption of consumer.

### grsecurity / pax surface

- **PaX UDEREF on getname(filename)** — defense against per-filename kernel-deref bug; SMAP forced.
- **acct CAP_SYS_PACCT** — strict enforcement; grsec disables the legacy CAP_SYS_ADMIN fallback. The only path to enable accounting is CAP_SYS_PACCT in init_user_ns; in non-init userns, accounting is rejected unconditionally regardless of capability.
- **GRKERNSEC_CHROOT_ACCT** — disallow acct(2) when caller is in a chroot, even with CAP_SYS_PACCT; defense against chroot-escape-via-accounting-file (a writable path under chroot that, post-escape, persists as a kernel-held fd).
- **PAX_USERCOPY_HARDEN on record write** — bounded sizeof(acct_v3); whitelisted slab buffer.
- **GRKERNSEC_PROC_HIDE on /proc/sys/kernel/acct** — sysctl gated by GRKERNSEC_PROC_USER.
- **GRKERNSEC_AUDIT_ACCT** — every acct(2) call (enable/disable) logged via grsec_audit with uid + path + cap state; defense against silent install of an accounting tap.
- **PAX_REFCOUNT on accounting file struct file** — defense against per-refcount-overflow UAF; install paths use file_get/file_put strict pairing.
- **Per-cred snapshot at acct install** — defense against per-cred-elevation between cap check and install.
- **GRKERNSEC_HIDESYM** — exposed file pointer through /proc/<pid>/* redacted.
- **Per-pidns destroy hook closes file deterministically** — defense against per-pidns vanish leaving dangling kernel reference.
- **PaX KERNEXEC on file_operations vtable for accounting file** — read-only.

