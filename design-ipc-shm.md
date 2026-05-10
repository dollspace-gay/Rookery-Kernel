---
title: "Tier-3: ipc/shm.c — System V shared memory"
tags: ["tier-3", "ipc", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

System V **shared memory** maps a kernel-backed shmem (or hugetlbfs) file into one or more processes' address spaces, addressable by `key_t` / `shmid`. Per-segment: `struct shmid_kernel` (shm_perm, shm_file = the underlying tmpfs/hugetlbfs file, shm_segsz, shm_nattch, shm_atim/dtim/ctim, shm_cprid/lprid, shm_creator, shm_clist, ns). Per-syscall: `shmget(key, size, shmflg)` creates / opens; `shmat(shmid, shmaddr, shmflg)` maps via `do_mmap` of an `alloc_file_clone` wrapping `struct shm_file_data {id, ns, file, vm_ops}` so that VMAs use `shm_vm_ops` (intercepts open/close/fault/may_split/pagesize/{set,get}_policy); `shmdt(shmaddr)` walks current.mm vmas, calls do_vmi_align_munmap on contiguous shm_vm_ops vmas of same file; `shmctl(shmid, cmd, buf)` IPC_STAT / IPC_SET / IPC_RMID / IPC_INFO / SHM_INFO / SHM_STAT / SHM_STAT_ANY / SHM_LOCK / SHM_UNLOCK. Per-namespace `ns->shm_ctlmax` (SHMMAX), `ns->shm_ctlall` (SHMALL), `ns->shm_ctlmni` (SHMMNI), `ns->shm_tot` (current pages), `ns->shm_rmid_forced`. SHM_HUGETLB uses hugetlb_file_setup with `(shmflg >> SHM_HUGE_SHIFT) & SHM_HUGE_MASK`. SHM_RDONLY → PROT_READ + O_RDONLY; SHM_RND → addr rounded down to SHMLBA boundary; SHM_REMAP → MAP_FIXED replaces existing mapping. IPC_RMID after `shm_nattch > 0` sets SHM_DEST + private key; actual destroy on last detach via shm_close → shm_destroy. Critical for: legacy IPC ABI, X11 MIT-SHM, PostgreSQL, hugepage shm.

This Tier-3 covers `ipc/shm.c` (~1880 lines).

### Acceptance Criteria

- [ ] AC-1: shmget(IPC_PRIVATE, 4096, 0666) creates segment with shm_segsz == 4096; shm_tot += 1 page.
- [ ] AC-2: shmget with size > ns.shm_ctlmax → -EINVAL.
- [ ] AC-3: shmget pushing shm_tot above ns.shm_ctlall → -ENOSPC.
- [ ] AC-4: shmget with size < SHMMIN (==0) → -EINVAL.
- [ ] AC-5: shmget(K, IPC_CREAT|IPC_EXCL) twice → second returns -EEXIST.
- [ ] AC-6: shmat(shmid, NULL, 0) returns a valid mapped address; nattch increments.
- [ ] AC-7: shmat(shmid, NULL, SHM_RDONLY) maps PROT_READ only; write fault SIGSEGV.
- [ ] AC-8: shmat with misaligned addr without SHM_RND → -EINVAL.
- [ ] AC-9: shmat with misaligned addr + SHM_RND rounds down to shmlba boundary.
- [ ] AC-10: shmat with addr=0 + SHM_REMAP → -EINVAL.
- [ ] AC-11: shmat with addr+size overlapping existing VMA without SHM_REMAP → -EINVAL.
- [ ] AC-12: shmdt(addr) unmaps the shm VMA + contiguous shm fragments of same file; nattch decrements.
- [ ] AC-13: shmctl(IPC_RMID) with nattch>0 sets SHM_DEST + private key; segment vanishes from shmget; persists until last detach.
- [ ] AC-14: shmctl(IPC_RMID) with nattch==0 destroys immediately.
- [ ] AC-15: shmctl(SHM_LOCK) without CAP_IPC_LOCK and non-owner euid → -EPERM.
- [ ] AC-16: SHM_HUGETLB with valid hstate creates hugetlb-backed segment; mismatched hstate → -EINVAL.
- [ ] AC-17: ipc_namespace isolation: segment created in ns A invisible in ns B.
- [ ] AC-18: shm_rmid_forced=1: orphaned segments destroyed on creator exit.
- [ ] AC-19: shmat / shmdt update shm_atim / shm_dtim and shm_lprid.

### Architecture

```
struct ShmidKernel {
  shm_perm: KernIpcPerm,
  shm_file: *File,                     // underlying shmem/hugetlbfs file
  shm_nattch: u64,                     // attach count
  shm_segsz: u64,                      // bytes
  shm_atim: i64,
  shm_dtim: i64,
  shm_ctim: i64,
  shm_cprid: *Pid,
  shm_lprid: Option<*Pid>,
  mlock_ucounts: Option<*Ucounts>,
  shm_creator: *TaskStruct,
  shm_clist: ListLink,                 // on creator.sysvshm.shm_clist
  ns: *IpcNamespace,
}

struct ShmFileData {                   // file->private_data on attach
  id: i32,
  ns: *IpcNamespace,
  file: *File,                         // ref to ShmidKernel.shm_file
  vm_ops: Option<*VmOperationsStruct>, // saved underlying ops
}
```

`Shm::sys_shmget(key, size, shmflg) -> Result<i32>`:
1. ns = current.nsproxy.ipc_ns.
2. shm_ops = {.getnew = Shm::newseg, .associate, .more_checks}.
3. return ipcget(ns, &shm_ids(ns), &shm_ops, ¶ms).

`Shm::newseg(ns, params) -> Result<i32>`:
1. numpages = (size + PAGE_SIZE - 1) >> PAGE_SHIFT.
2. Bounds: SHMMIN ≤ size ≤ ns.shm_ctlmax; overflow check; ns.shm_tot + numpages ≤ ns.shm_ctlall.
3. shp = kmalloc_obj.
4. security_shm_alloc.
5. if shmflg & SHM_HUGETLB:
   - hs = hstate_sizelog((shmflg >> SHM_HUGE_SHIFT) & SHM_HUGE_MASK).
   - file = hugetlb_file_setup(name, ALIGN(size, huge_page_size(hs)), acctflag, HUGETLB_SHMFS_INODE, log).
6. else: file = shmem_kernel_file_setup(name, size, acctflag).
7. shp.shm_file = file; shp.shm_segsz = size; shp.shm_creator = current.
8. ipc_addid(&shm_ids(ns), &shp.shm_perm, ns.shm_ctlmni).
9. task_lock(current); list_add(&shp.shm_clist, &current.sysvshm.shm_clist); task_unlock.
10. file_inode(file).i_ino = shp.shm_perm.id.
11. ns.shm_tot += numpages.

`Shm::do_shmat(shmid, shmaddr, shmflg, raddr, shmlba) -> Result<()>`:
1. Validate addr alignment (SHM_RND round-down or strict).
2. Compute prot/acc_mode/f_flags from SHM_RDONLY / SHM_EXEC.
3. rcu_read_lock; shp = shm_obtain_object_check.
4. ipcperms; security_shm_shmat.
5. ipc_lock; valid_object; base = get_file(shp.shm_file); shp.shm_nattch++; size = i_size_read.
6. sfd = ShmFileData{id, ns: get_ipc_ns(ns), file: base, vm_ops: None}.
7. file = alloc_file_clone(base, f_flags, is_file_hugepages(base) ? &file_ops_huge : &file_ops).
8. file.private_data = sfd.
9. mmap_write_lock_killable.
10. if addr ∧ !SHM_REMAP: check overflow + find_vma_intersection ⟹ -EINVAL.
11. addr = do_mmap(file, addr, size, prot, MAP_SHARED|MAP_FIXED?, 0, 0, &populate, NULL).
12. mm_populate(addr, populate).
13. fput(file). /* alloc_file_clone ref balanced by do_mmap */
14. down_write(&shm_ids(ns).rwsem); shm_lock; shp.shm_nattch-- (adjust counter); destroy-if-may; up_write.

`Shm::mmap(file, vma) -> Result<()>`:
1. sfd = shm_file_data(file).
2. Shm::open_inner(sfd). /* +1 nattch; atim, lprid */
3. vfs_mmap(sfd.file, vma).
4. sfd.vm_ops = vma.vm_ops.
5. vma.vm_ops = &shm_vm_ops.

`Shm::sys_shmdt(shmaddr) -> Result<()>`:
1. if addr & ~PAGE_MASK: -EINVAL.
2. mmap_write_lock_killable.
3. Find first matching shm_vm_ops vma starting at addr (pgoff == (vm_start - addr) / PAGE_SIZE).
4. Record file + size = i_size_read; do_vmi_align_munmap(vma).
5. Continue iterating: unmap contiguous fragments of same file matching pgoff arithmetic.
6. mmap_write_unlock.

`Shm::sys_shmctl(shmid, cmd, buf) -> Result<i64>`:
1. Dispatch cmd → ctl_info / ctl_stat / ctl_down (IPC_RMID/IPC_SET) / ctl_do_lock (SHM_LOCK/UNLOCK).

`Shm::destroy(ns, shp)`:
1. shm_file = shp.shm_file; shp.shm_file = NULL.
2. ns.shm_tot -= numpages.
3. Shm::rmid(shp). /* clist_rm + ipc_rmid */
4. shm_unlock.
5. if !is_file_hugepages(shm_file): shmem_lock(shm_file, 0, shp.mlock_ucounts).
6. fput(shm_file).
7. ipc_update_pid(&shp.shm_cprid, NULL); ipc_update_pid(&shp.shm_lprid, NULL).
8. ipc_rcu_putref(&shp.shm_perm, shm_rcu_free).

`Shm::exit_shm(task)`:
1. Loop until task.sysvshm.shm_clist empty.
2. For each shp ∈ shm_clist: branch on ns.shm_rmid_forced.
3. If !shm_rmid_forced: list_del_init(&shp.shm_clist); continue.
4. Else: ipc_rcu_getref; list_del_init; down_write(&shm_ids(ns).rwsem); shm_lock_by_ptr.
5. ipc_rcu_putref; if valid + may_destroy: Shm::destroy.

`Shm::init_ns(ns)`:
1. ns.shm_ctlmax = SHMMAX; ns.shm_ctlall = SHMALL; ns.shm_ctlmni = SHMMNI.
2. ns.shm_rmid_forced = 0; ns.shm_tot = 0.
3. ipc_init_ids(&shm_ids(ns)).

### Out of Scope

- `mm/shmem.c` tmpfs backing — covered in `mm/shmem.md` Tier-3.
- `fs/hugetlbfs/inode.c` hugetlb_file_setup — covered in `fs/hugetlbfs.md` Tier-3.
- `ipc/util.c` ipc_addid / ipcget / ipcctl_obtain_check / ipcperms — covered in `ipc/util.md`.
- `ipc/namespace.c` ipc_namespace lifecycle — covered in `ipc/namespace.md`.
- `mm/mmap.c` do_mmap / do_vmi_align_munmap — covered in `mm/mmap.md` Tier-3.
- POSIX shm_open (`fs/tmpfs/`) — separate, not in ipc/shm.c.
- LSM security_shm_* implementations.
- compat (32-bit) ABI marshaling beyond what compat_ksys_shmctl does inline.
- /proc/sys/kernel/{shmmax,shmall,shmmni,shm_rmid_forced} sysctl write path.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct shmid_kernel` | per-segment kernel object | `ShmidKernel` |
| `struct shm_file_data` | per-attached-file shim | `ShmFileData` |
| `shm_file_operations` / `shm_file_operations_huge` | per-shim-file fops (mmap, fsync, release, get_unmapped_area, llseek, fallocate) | `Shm::file_ops` / `file_ops_huge` |
| `shm_vm_ops` | per-VMA ops (open, close, fault, may_split, pagesize, set/get_policy) | `Shm::vm_ops` |
| `ksys_shmget()` / `SYSCALL_DEFINE3(shmget, ...)` | per-syscall create/open | `Shm::sys_shmget` |
| `do_shmat()` / `SYSCALL_DEFINE3(shmat, ...)` | per-syscall attach | `Shm::sys_shmat` / `do_shmat` |
| `ksys_shmdt()` / `SYSCALL_DEFINE1(shmdt, ...)` | per-syscall detach | `Shm::sys_shmdt` |
| `ksys_shmctl()` / `SYSCALL_DEFINE3(shmctl, ...)` | per-syscall control | `Shm::sys_shmctl` |
| `newseg()` | per-segment constructor | `Shm::newseg` |
| `shm_destroy()` | per-zero-attach destructor | `Shm::destroy` |
| `shm_may_destroy()` | per-destroy-eligibility | `Shm::may_destroy` |
| `do_shm_rmid()` / `shm_rmid()` / `shm_clist_rm()` | per-RMID cleanup | `Shm::do_rmid` / `rmid` / `clist_rm` |
| `__shm_open()` / `shm_open()` / `__shm_close()` / `shm_close()` | per-attach/detach VMA ops | `Shm::open_inner` / `open` / `close_inner` / `close` |
| `shm_mmap()` / `shm_fault()` | per-shim-file mmap + fault delegation | `Shm::mmap` / `fault` |
| `shm_release()` | per-fput release of shim file | `Shm::release` |
| `shmctl_down()` / `shmctl_stat()` / `shmctl_ipc_info()` / `shmctl_shm_info()` / `shmctl_do_lock()` | per-cmd dispatch | `Shm::ctl_down` / `ctl_stat` / `ctl_ipc_info` / `ctl_shm_info` / `ctl_do_lock` |
| `shm_get_stat()` / `shm_add_rss_swap()` | per-RSS/swap accounting | `Shm::get_stat` / `add_rss_swap` |
| `shm_destroy_orphaned()` / `exit_shm()` | per-task-exit orphan cleanup | `Shm::destroy_orphaned` / `exit_shm` |
| `shm_init_ns()` / `shm_exit_ns()` / `shm_init()` | per-namespace lifecycle | `Shm::init_ns` / `exit_ns` / `init` |
| `shm_try_destroy_orphaned()` | per-sysctl shm_rmid_forced helper | `Shm::try_destroy_orphaned` |

### compatibility contract

REQ-1: struct shmid_kernel:
- shm_perm: per-`struct kern_ipc_perm`.
- shm_file: per-`struct file *` to underlying tmpfs/hugetlbfs file (created in newseg via shmem_kernel_file_setup or hugetlb_file_setup).
- shm_nattch: per-current attach count.
- shm_segsz: per-segment size in bytes (1..ns.shm_ctlmax).
- shm_atim / shm_dtim / shm_ctim: per-time stamps.
- shm_cprid / shm_lprid: per-creator / last attaching pid.
- mlock_ucounts: per-`struct ucounts *` for memlock accounting (SHM_LOCK).
- shm_creator: per-`struct task_struct *` (task_lock for read/write on shm_clist).
- shm_clist: per-list-link on creator's task->sysvshm.shm_clist.
- ns: per-`struct ipc_namespace *` back-pointer.
- __randomize_layout.

REQ-2: shm_mode upper-byte flags (private):
- SHM_DEST (01000): segment scheduled for destruction on last detach (set by IPC_RMID when nattch > 0).
- SHM_LOCKED (02000): segment mlocked.

REQ-3: struct shm_file_data (file->private_data on attach):
- id: per-shmid (must match shp->shm_perm.id; reuse detected by file pointer compare).
- ns: per-`struct ipc_namespace *` (held via get_ipc_ns).
- file: per-real shm_file (held via get_file in do_shmat).
- vm_ops: per-underlying-vma's vm_ops (saved by shm_mmap before overwriting with shm_vm_ops).

REQ-4: shmget(key, size, shmflg) → ksys_shmget:
- ns = current.nsproxy.ipc_ns.
- shm_ops = {.getnew = newseg, .associate = security_shm_associate, .more_checks = shm_more_checks}.
- params = {.key, .flg = shmflg, .u.size = size}.
- return ipcget(ns, &shm_ids(ns), &shm_ops, ¶ms).

REQ-5: newseg(ns, params):
- numpages = (size + PAGE_SIZE - 1) >> PAGE_SHIFT.
- if size < SHMMIN ∨ size > ns.shm_ctlmax: return -EINVAL.
- if numpages << PAGE_SHIFT < size: return -ENOSPC. /* overflow */
- if ns.shm_tot + numpages > ns.shm_ctlall: return -ENOSPC.
- shp = kmalloc_obj(*shp, GFP_KERNEL_ACCOUNT).
- shp.shm_perm.mode = shmflg & S_IRWXUGO.
- shp.mlock_ucounts = NULL.
- security_shm_alloc(&shp.shm_perm).
- name = "SYSV%08x" % key.
- if shmflg & SHM_HUGETLB:
  - hs = hstate_sizelog((shmflg >> SHM_HUGE_SHIFT) & SHM_HUGE_MASK).
  - hugesize = ALIGN(size, huge_page_size(hs)).
  - if has_no_reserve (SHM_NORESERVE): vma_flags_set(&acctflag, VMA_NORESERVE_BIT).
  - file = hugetlb_file_setup(name, hugesize, acctflag, HUGETLB_SHMFS_INODE, log2_page_size).
- else:
  - if has_no_reserve ∧ sysctl_overcommit_memory != OVERCOMMIT_NEVER: vma_flags_set(&acctflag, VMA_NORESERVE_BIT).
  - file = shmem_kernel_file_setup(name, size, acctflag).
- shp.shm_cprid = get_pid(task_tgid(current)).
- shp.shm_lprid = NULL.
- shp.shm_atim = shp.shm_dtim = 0.
- shp.shm_ctim = ktime_get_real_seconds.
- shp.shm_segsz = size; shp.shm_nattch = 0.
- shp.shm_file = file; shp.shm_creator = current.
- ipc_addid(&shm_ids(ns), &shp.shm_perm, ns.shm_ctlmni).
- shp.ns = ns.
- task_lock(current); list_add(&shp.shm_clist, &current.sysvshm.shm_clist); task_unlock.
- file_inode(file).i_ino = shp.shm_perm.id. /* shown as "inode#" in /proc/pid/maps */
- ns.shm_tot += numpages.
- ipc_unlock_object; rcu_read_unlock.
- return shp.shm_perm.id.

REQ-6: do_shmat(shmid, shmaddr, shmflg, *raddr, shmlba):
- /* Address handling */
- addr = (unsigned long)shmaddr.
- if addr:
  - if addr & (shmlba - 1):
    - if shmflg & SHM_RND: addr &= ~(shmlba - 1). /* round down */
    - else: return -EINVAL.
  - flags |= MAP_FIXED.
- else if shmflg & SHM_REMAP: return -EINVAL.
- /* Prot + flags from SHM_RDONLY / SHM_EXEC */
- if shmflg & SHM_RDONLY: prot = PROT_READ; acc_mode = S_IRUGO; f_flags = O_RDONLY.
- else: prot = PROT_READ | PROT_WRITE; acc_mode = S_IRUGO | S_IWUGO; f_flags = O_RDWR.
- if shmflg & SHM_EXEC: prot |= PROT_EXEC; acc_mode |= S_IXUGO.
- /* Lookup + perm */
- ns = current.nsproxy.ipc_ns.
- rcu_read_lock; shp = shm_obtain_object_check(ns, shmid).
- ipcperms(ns, &shp.shm_perm, acc_mode).
- security_shm_shmat(&shp.shm_perm, shmaddr, shmflg).
- ipc_lock_object; ipc_valid_object check.
- /* Ref + nattch */
- base = get_file(shp.shm_file). /* hold real file */
- shp.shm_nattch++. /* may go to 0 again in out_nattch path */
- size = i_size_read(file_inode(base)).
- ipc_unlock_object; rcu_read_unlock.
- /* Build shim file */
- sfd = kzalloc_obj(*sfd).
- file = alloc_file_clone(base, f_flags, is_file_hugepages(base) ? &shm_file_operations_huge : &shm_file_operations).
- sfd = {.id = shp.shm_perm.id, .ns = get_ipc_ns(ns), .file = base, .vm_ops = NULL}.
- file.private_data = sfd.
- security_mmap_file(file, prot, flags).
- /* Map */
- mmap_write_lock_killable(current.mm).
- if addr ∧ !(shmflg & SHM_REMAP):
  - if addr + size < addr: -EINVAL.
  - if find_vma_intersection(current.mm, addr, addr + size): -EINVAL.
- addr = do_mmap(file, addr, size, prot, flags, 0, 0, &populate, NULL).
- *raddr = addr.
- mmap_write_unlock; mm_populate(addr, populate).
- fput(file). /* clone done; do_mmap took a ref */
- /* Adjust nattch */
- down_write(&shm_ids(ns).rwsem); shp = shm_lock(ns, shmid); shp.shm_nattch--.
- if shm_may_destroy(shp): shm_destroy(ns, shp). else: shm_unlock.
- up_write.

REQ-7: shm_mmap(file, vma):
- sfd = shm_file_data(file).
- ret = __shm_open(sfd). /* +1 nattch; sets shm_atim, shm_lprid */
- if ret: return ret.
- vfs_mmap(sfd.file, vma). /* delegate to underlying shmem/hugetlbfs mmap */
- if err: __shm_close(sfd); return err.
- sfd.vm_ops = vma.vm_ops. /* save underlying ops */
- vma.vm_ops = &shm_vm_ops. /* override; intercepts fault → sfd->vm_ops->fault */
- return 0.

REQ-8: shm_vm_ops (overrides):
- open = shm_open → __shm_open (sfd->vm_ops->open + nattch++, atim, lprid).
- close = shm_close → __shm_close (sfd->vm_ops->close + nattch--, dtim, lprid; destroy if may_destroy).
- fault = shm_fault → sfd.vm_ops->fault(vmf).
- may_split = shm_may_split → sfd.vm_ops->may_split.
- pagesize = shm_pagesize → sfd.vm_ops->pagesize (HugeTLB).
- set_policy / get_policy: NUMA mempolicy passthrough.

REQ-9: __shm_open(sfd):
- shp = shm_lock(sfd.ns, sfd.id).
- if shp.shm_file != sfd.file: shm_unlock; return -EINVAL. /* ID reused */
- shp.shm_atim = ktime_get_real_seconds.
- ipc_update_pid(&shp.shm_lprid, task_tgid(current)).
- shp.shm_nattch++.
- shm_unlock.

REQ-10: __shm_close(sfd):
- down_write(&shm_ids(ns).rwsem).
- shp = shm_lock(ns, sfd.id).
- ipc_update_pid(&shp.shm_lprid, task_tgid(current)).
- shp.shm_dtim = ktime_get_real_seconds.
- shp.shm_nattch--.
- if shm_may_destroy(shp): shm_destroy(ns, shp). /* unlocks shp */
- else: shm_unlock(shp).
- up_write.

REQ-11: shm_may_destroy(shp):
- return shp.shm_nattch == 0 ∧ (shp.ns.shm_rmid_forced ∨ (shp.shm_perm.mode & SHM_DEST)).

REQ-12: shm_destroy(ns, shp):
- shm_file = shp.shm_file; shp.shm_file = NULL.
- ns.shm_tot -= (shp.shm_segsz + PAGE_SIZE - 1) >> PAGE_SHIFT.
- shm_rmid(shp). /* shm_clist_rm + ipc_rmid */
- shm_unlock(shp).
- if !is_file_hugepages(shm_file): shmem_lock(shm_file, 0, shp.mlock_ucounts). /* drop mlock */
- fput(shm_file).
- ipc_update_pid(&shp.shm_cprid, NULL); ipc_update_pid(&shp.shm_lprid, NULL).
- ipc_rcu_putref(&shp.shm_perm, shm_rcu_free).

REQ-13: shmdt(shmaddr) — ksys_shmdt:
- if addr & ~PAGE_MASK: return -EINVAL.
- mmap_write_lock_killable(current.mm).
- /* Find first shm_vm_ops vma starting at addr */
- VMA_ITERATOR(vmi, mm, addr).
- for_each_vma(vmi, vma):
  - if vma.vm_ops == &shm_vm_ops ∧ (vma.vm_start - addr)/PAGE_SIZE == vma.vm_pgoff:
    - file = vma.vm_file; size = i_size_read(file_inode(vma.vm_file)).
    - do_vmi_align_munmap(&vmi, vma, mm, vma.vm_start, vma.vm_end, NULL, false).
    - retval = 0; vma = vma_next(&vmi); break.
- /* Continue unmapping contiguous fragments of same shm file */
- size = PAGE_ALIGN(size).
- while vma ∧ (vma.vm_end - addr) ≤ size:
  - if vma.vm_ops == &shm_vm_ops ∧ same-file ∧ matching-pgoff:
    - do_vmi_align_munmap(...).
  - vma = vma_next(&vmi).
- mmap_write_unlock.

REQ-14: shmctl(shmid, cmd, buf):
- IPC_INFO: shmctl_ipc_info → struct shminfo64 (shmmni = shmseg = ns.shm_ctlmni; shmmax = ns.shm_ctlmax; shmall = ns.shm_ctlall; shmmin = SHMMIN).
- SHM_INFO: shmctl_shm_info → struct shm_info (used_ids, shm_rss, shm_swp, shm_tot; swap_attempts/successes always 0).
- SHM_STAT / SHM_STAT_ANY / IPC_STAT: shmctl_stat → copy_shmid_to_user.
- IPC_SET: shmctl_down(IPC_SET) → ipc_update_perm; shm_ctim updated.
- IPC_RMID: shmctl_down(IPC_RMID) → do_shm_rmid (set SHM_DEST + private key if nattch>0; else destroy).
- SHM_LOCK / SHM_UNLOCK: shmctl_do_lock → shmem_lock(shp.shm_file, lock, &shp.mlock_ucounts); requires CAP_IPC_LOCK or matching euid.

REQ-15: do_shm_rmid(ns, ipcp):
- shp = container_of(ipcp, struct shmid_kernel, shm_perm).
- if shp.shm_nattch:
  - shp.shm_perm.mode |= SHM_DEST.
  - ipc_set_key_private(&shm_ids(ns), &shp.shm_perm). /* invisible to future shmget */
  - shm_unlock(shp).
- else:
  - shm_destroy(ns, shp). /* zero attach: destroy now */

REQ-16: exit_shm(task) — task-exit cleanup:
- Loop while task.sysvshm.shm_clist not empty:
  - shp = list_first_entry(&task.sysvshm.shm_clist, struct shmid_kernel, shm_clist).
  - ns = shp.ns.
  - if !ns.shm_rmid_forced: list_del_init(&shp.shm_clist); continue. /* only book-keeping */
  - ns = get_ipc_ns_not_zero(ns). /* hold ns alive */
  - ipc_rcu_getref(&shp.shm_perm).
  - list_del_init(&shp.shm_clist); task_unlock.
  - down_write(&shm_ids(ns).rwsem); shm_lock_by_ptr(shp).
  - ipc_rcu_putref(&shp.shm_perm, shm_rcu_free).
  - if ipc_valid_object(&shp.shm_perm):
    - if shm_may_destroy(shp): shm_destroy(ns, shp).
    - else: shm_unlock(shp).
  - else: shm_unlock(shp).
  - up_write; put_ipc_ns(ns).

REQ-17: shm_destroy_orphaned(ns):
- /* Called when shm_rmid_forced is set; reaps orphans of dead creators */
- down_write(&shm_ids(ns).rwsem).
- if shm_ids(ns).in_use: idr_for_each(&shm_ids(ns).ipcs_idr, shm_try_destroy_orphaned, ns).
- up_write.

REQ-18: Limits + ns-isolation:
- SHMMIN = 1 byte (min seg size — newseg rejects 0 with -EINVAL).
- SHMMAX = ULONG_MAX - (1UL << 24) (effectively unbounded; ns.shm_ctlmax overrides).
- SHMALL = ULONG_MAX - (1UL << 24) (system-wide total in pages; ns.shm_ctlall).
- SHMMNI = 4096 segments (ns.shm_ctlmni).
- SHMSEG = SHMMNI (per-process attach limit).
- All limits live on `struct ipc_namespace`.

REQ-19: shm_init_ns / shm_exit_ns:
- init: ns.shm_ctlmax = SHMMAX; ns.shm_ctlall = SHMALL; ns.shm_ctlmni = SHMMNI; ns.shm_rmid_forced = 0; ns.shm_tot = 0; ipc_init_ids.
- exit: free_ipcs(ns, &shm_ids(ns), do_shm_rmid); idr_destroy; rhashtable_destroy.

REQ-20: SHM_LOCK / SHM_UNLOCK (shmctl_do_lock):
- if !ns_capable(ns.user_ns, CAP_IPC_LOCK):
  - euid = current_euid().
  - if !uid_eq(euid, shp.shm_perm.uid) ∧ !uid_eq(euid, shp.shm_perm.cuid): -EPERM.
- shmem_lock(shp.shm_file, lock(1)/unlock(0), &shp.mlock_ucounts). /* manages ucount and SHM_LOCKED bit */

REQ-21: SHMLBA address alignment:
- SHM_RND clears low (shmlba - 1) bits of shmaddr; must be non-zero after round-down (otherwise -EINVAL).
- Without SHM_RND: addr must be SHMLBA-aligned else -EINVAL (arch-specific exception: __ARCH_FORCE_SHMLBA only requires PAGE-alignment).

REQ-22: SHM_REMAP semantics:
- Requires non-zero shmaddr; forces MAP_FIXED via do_mmap; replaces overlapping existing mappings.
- Without SHM_REMAP, MAP_FIXED is set only when addr != 0, and existing intersecting VMA causes -EINVAL.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_bounded` | INVARIANT | per-newseg: SHMMIN ≤ size ≤ ns.shm_ctlmax. |
| `numpages_overflow_checked` | INVARIANT | per-newseg: numpages << PAGE_SHIFT ≥ size. |
| `shm_tot_bounded` | INVARIANT | per-newseg: ns.shm_tot + numpages ≤ ns.shm_ctlall. |
| `shmlba_alignment` | INVARIANT | per-do_shmat: addr & (shmlba - 1) == 0 (unless SHM_RND with non-zero round-down). |
| `shm_rdonly_prot` | INVARIANT | per-do_shmat: shmflg & SHM_RDONLY ⟹ prot == PROT_READ; f_flags == O_RDONLY. |
| `shm_remap_requires_addr` | INVARIANT | per-do_shmat: SHM_REMAP ⟹ addr != 0. |
| `shm_nattch_balanced` | INVARIANT | per-shmat / shmdt: each open + each close balanced; shm_destroy iff nattch == 0. |
| `shm_dest_after_rmid_nattch_pos` | INVARIANT | per-do_shm_rmid: shm_nattch > 0 ⟹ SHM_DEST set + private key. |
| `shm_destroy_only_zero_nattch` | INVARIANT | per-shm_may_destroy: nattch == 0 enforced. |
| `shm_rmid_forced_orphan_reap` | INVARIANT | per-exit_shm: shm_rmid_forced ⟹ orphan reaped on creator exit. |

### Layer 2: TLA+

`ipc/shm.tla`:
- States: ShmGet, ShmAt{Lookup, MapFile, BumpNattch, Mmap}, ShmDt{FindVma, Munmap}, ShmCtlRmid, ExitShm.
- Properties:
  - `safety_nattch_consistent` — shp.shm_nattch == count of live VMAs referencing shp via sfd.
  - `safety_shm_tot_consistent` — ns.shm_tot == sum of numpages of all shp in ns.
  - `safety_rmid_after_attach` — IPC_RMID + nattch > 0 ⟹ SHM_DEST + invisible to shmget (private key); detach → destroy.
  - `safety_destroy_serialized` — shm_ids.rwsem held writer during shm_destroy.
  - `safety_huge_aligned` — SHM_HUGETLB ⟹ underlying file size aligned to huge_page_size.
  - `liveness_destroy_progress` — after IPC_RMID + final shmdt: shm_destroy eventually runs.
  - `liveness_exit_shm_drains` — task exit eventually empties task.sysvshm.shm_clist.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Shm::newseg` post: shp.shm_segsz == size; shp on creator's shm_clist; ns.shm_tot += numpages | `Shm::newseg` |
| `Shm::do_shmat` post: on success current.mm has VMA(start..start+size) with vm_ops == &shm_vm_ops | `Shm::do_shmat` |
| `Shm::mmap` post: vma.vm_ops == &shm_vm_ops ∧ sfd.vm_ops == saved-underlying | `Shm::mmap` |
| `Shm::open_inner` post: shp.shm_nattch += 1; shp.shm_atim updated | `Shm::open_inner` |
| `Shm::close_inner` post: shp.shm_nattch -= 1; shp.shm_dtim updated; destroy iff may_destroy | `Shm::close_inner` |
| `Shm::destroy` post: shp.shm_file = NULL; ns.shm_tot -= numpages; perm freed via RCU | `Shm::destroy` |
| `Shm::do_rmid` post: nattch>0 ⟹ SHM_DEST + private key; nattch==0 ⟹ destroy now | `Shm::do_rmid` |
| `Shm::sys_shmdt` post: all contiguous matching shm_vm_ops vmas from addr unmapped | `Shm::sys_shmdt` |

### Layer 4: Verus/Creusot functional

`shmget → shmem_kernel_file_setup (or hugetlb_file_setup) → shmid_kernel → ipc_addid → return id` equivalence: per-SUS shmget(2). `shmat → alloc_file_clone(shm_file_operations) → do_mmap(file)` equivalence: per-SUS shmat(2). `shm_close → shm_destroy on may_destroy` equivalence: per-SUS shmctl(2) IPC_RMID + Linux shm_rmid_forced semantics.

### hardening

(Inherits row-1 features from `ipc/00-overview.md` § Hardening.)

Shm-segment reinforcement:

- **Per-size > ns.shm_ctlmax rejected** — defense against per-userland huge-segment OOM.
- **Per-numpages overflow check** — defense against per-(size + PAGE_SIZE - 1) integer wrap.
- **Per-ns.shm_tot + numpages ≤ ns.shm_ctlall** — defense against per-namespace global-page exhaustion.
- **Per-SHM_HUGETLB hstate validation** — defense against per-invalid-hpage shmflg crash.
- **Per-SHM_NORESERVE rejected when OVERCOMMIT_NEVER** — defense against per-no-accounting trap.
- **Per-SHM_RND must yield non-zero addr** — defense against per-round-down-to-NULL mapping.
- **Per-SHM_REMAP requires non-zero shmaddr** — defense against per-implicit-MAP_FIXED ambiguity.
- **Per-addr+size overflow check** — defense against per-VMA-wrap integer overflow.
- **Per-find_vma_intersection without SHM_REMAP** — defense against per-silent-overlap.
- **Per-shp.shm_file != sfd.file on __shm_open** — defense against per-ID-reuse misattach.
- **Per-CAP_IPC_LOCK or matching euid for SHM_LOCK** — defense against per-unprivileged mlock DoS.
- **Per-CAP_IPC_OWNER for IPC_RMID / IPC_SET** — defense against per-cross-user destruction.
- **Per-LSM hooks security_shm_{alloc,shmat,shmctl,associate}** — defense via SELinux/AppArmor.
- **Per-RCU rcu_free + ipc_rcu_putref** — defense against per-UAF on concurrent RMID.
- **Per-ipc_valid_object re-check after lock** — defense against per-RMID race.
- **Per-shm_rmid_forced orphan reap** — defense against per-leak from dead creator.
- **Per-mmap_write_lock_killable** — defense against per-uninterruptible-mmap hang.
- **Per-`__randomize_layout`** — defense against per-known-offset attacks.
- **Per-ipc_namespace isolation** — defense against per-cross-ns segment access.

