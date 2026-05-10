# Foundation: Linux kernel glossary

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
upstream-paths:
  - include/linux/
  - include/uapi/linux/
  - arch/x86/include/
  - Documentation/
status: draft
-->

## Summary
Defines the kernel-specific terminology every Rookery design doc relies on. Each entry names the upstream Linux source location where the concept is introduced, so a reader can verify the definition. Terms are organized by subsystem; cross-references link related concepts.

This is a *terminology* document, not an API reference. The exhaustive API surface is the union of `rust/kernel/` (already-wrapped) and the `.design/uapi/` Tier-5 docs (TBD). When this glossary and a subsystem doc disagree on a definition, the subsystem doc wins and this doc is updated.

## Requirements

- REQ-1: At least 40 kernel-specific terms are defined (per `00-overview.md` AC-5).
- REQ-2: Every entry names an upstream source path verifiable via `ls /home/doll/linux-src/<path>`.
- REQ-3: Cross-references between related terms use the format `(see [Term])`.
- REQ-4: Terms are organized by subsystem so a reader can browse the kernel's structure topically.
- REQ-5: When the upstream definition spans multiple files, the entry names the canonical primary header (most often `include/linux/<concept>.h`).

## Acceptance Criteria

- [ ] AC-1: A grep for `^### ` (entry headers) returns ≥ 40 matches. (covers REQ-1)
- [ ] AC-2: Every `Source:` line names a path that resolves under `/home/doll/linux-src/` (verified via the script in the appendix). (covers REQ-2)
- [ ] AC-3: Every `(see [...])` cross-reference resolves to another entry in this document. (covers REQ-3)
- [ ] AC-4: Each major subsystem (process / memory / locking / time / VFS / network / block / IRQ / boot / security / IPC / observability) has at least three entries. (covers REQ-4)

## Architecture

The glossary entry format:

```markdown
### TermName
**Kind**: data type | macro | concept | subsystem | function family | file format
**One-liner**: A single sentence definition.
**Source**: `<primary upstream path>` (e.g. `include/linux/sched.h`)
**Notes**: optional aliases, units, cross-refs to other entries.
```

Cross-references format: `(see [TermName])`. When a term is a Linux-ism that has a precise Rookery counterpart already in `rust/kernel/`, the entry names it.

---

## Process and scheduling

### task_struct
**Kind**: data type
**One-liner**: The kernel's per-task control block — every kernel-visible execution context (kthread or userspace thread) is a `task_struct`.
**Source**: `include/linux/sched.h`
**Notes**: Linux uses "task" to mean either a process or a thread; threads share `mm_struct` and `files_struct` but have distinct `task_struct`s. (see [thread_info], [pid], [tgid])

### thread_info
**Kind**: data type
**One-liner**: Per-task auxiliary state historically placed at the bottom of the kernel stack; on modern x86_64 it lives inside `task_struct`.
**Source**: `arch/x86/include/asm/thread_info.h`
**Notes**: Holds flags such as `TIF_NEED_RESCHED`, `TIF_SIGPENDING`, `TIF_NOTIFY_RESUME`. (see [task_struct])

### pid
**Kind**: scalar identifier
**One-liner**: Process IDentifier — uniquely identifies a `task_struct` within a PID namespace.
**Source**: `include/linux/pid.h`
**Notes**: Inside the kernel `pid_t` is the integer; `struct pid` is the refcounted handle. (see [tgid], [namespace])

### tgid
**Kind**: scalar identifier
**One-liner**: Thread Group ID — the PID of the thread-group leader, equal across all threads of a process.
**Source**: `include/linux/sched.h` (field `tgid` of `task_struct`)
**Notes**: The userspace `getpid(2)` returns the tgid; `gettid(2)` returns the kernel's pid. (see [pid])

### kthread
**Kind**: concept + API family
**One-liner**: Kernel thread — a `task_struct` with no `mm_struct`, executing only in kernel space.
**Source**: `include/linux/kthread.h`, `kernel/kthread.c`
**Notes**: Spawned via `kthread_create`/`kthread_run`; stoppable via `kthread_stop`. The Rookery `kernel::task::Kthread` abstraction (issue #4) wraps this.

### runqueue (rq)
**Kind**: data type
**One-liner**: Per-CPU scheduling state holding the runnable tasks ready to execute on this CPU.
**Source**: `kernel/sched/sched.h` (`struct rq`)
**Notes**: Contains a CFS rbtree, real-time arrays, and deadline tree. (see [vruntime], [CFS])

### CFS
**Kind**: scheduler class
**One-liner**: Completely Fair Scheduler — the default scheduler class for `SCHED_NORMAL` tasks; orders tasks by `vruntime`.
**Source**: `kernel/sched/fair.c`
**Notes**: Linux 7.x has migrated to EEVDF semantics inside the same `fair.c` file. (see [vruntime], [EEVDF])

### EEVDF
**Kind**: scheduler algorithm
**One-liner**: Earliest Eligible Virtual Deadline First — the algorithm that replaced classic CFS scheduling in 2024+, retaining the `fair.c` filename for code-archeology continuity.
**Source**: `kernel/sched/fair.c` (function `pick_eevdf`)
**Notes**: (see [CFS], [vruntime])

### vruntime
**Kind**: scalar field
**One-liner**: Virtual runtime — the CFS/EEVDF accounting field tracking how much CPU time a task has consumed, weighted by nice value.
**Source**: `kernel/sched/fair.c`, field on `struct sched_entity` in `include/linux/sched.h`
**Notes**: The leftmost task in the rbtree (smallest vruntime) is the next picked. (see [runqueue (rq)], [CFS])

### preemption / NEED_RESCHED
**Kind**: concept + flag
**One-liner**: The scheduler's ability to preempt a running task at a kernel preemption point; `TIF_NEED_RESCHED` requests the next preemption point reschedule.
**Source**: `include/linux/preempt.h`, flag in `arch/x86/include/asm/thread_info.h`
**Notes**: Voluntary at `cond_resched()`; involuntary at IRQ-return-to-userspace. (see [thread_info])

### nice / priority
**Kind**: scalar field + concept
**One-liner**: Nice values (-20 to +19) translate to scheduler priorities; lower = more CPU.
**Source**: `include/linux/sched/prio.h`
**Notes**: Real-time tasks have priorities 0–99 (FIFO/RR), CFS tasks 100–139.

### cgroup
**Kind**: subsystem + data type
**One-liner**: Control group — a hierarchy that groups tasks for resource accounting and limits (CPU, memory, I/O, PID, net, etc.).
**Source**: `include/linux/cgroup.h`, `kernel/cgroup/`
**Notes**: cgroup v1 and v2 coexist; v2 is the canonical interface. (see [namespace])

### namespace
**Kind**: subsystem + data type
**One-liner**: A view of a kernel resource (PIDs, filesystems, networks, users, IPC, hostnames, time) scoped to a subset of tasks.
**Source**: `include/linux/nsproxy.h`, `include/linux/pid_namespace.h`, `include/linux/mount.h` (mnt_ns), etc.
**Notes**: Underpins containers (Docker, LXC, systemd-nspawn). (see [pid], [cgroup])

---

## Memory

### page (struct page)
**Kind**: data type
**One-liner**: The kernel's metadata for a single physical 4 KiB page frame.
**Source**: `include/linux/mm_types.h`
**Notes**: Increasingly being phased out in favor of `struct folio` for compound pages. (see [folio])

### folio
**Kind**: data type
**One-liner**: A type-safe wrapper around a contiguous, naturally-aligned set of `struct page`s representing a logical unit of memory (often a page-cache mapping).
**Source**: `include/linux/mm_types.h` (struct folio), `include/linux/page-flags.h`
**Notes**: Enables future variable-page-size support. (see [page (struct page)])

### pgd / p4d / pud / pmd / pte
**Kind**: data types
**One-liner**: Page table levels: Page Global / 4th-level / Upper / Middle / Page Table Entry — the five-level page table tree used on x86_64.
**Source**: `arch/x86/include/asm/pgtable_types.h`, `arch/x86/include/asm/pgtable.h`
**Notes**: x86_64 uses 4 or 5 levels depending on `CONFIG_X86_5LEVEL` and CR4.LA57. (see [vma (vm_area_struct)])

### vma (vm_area_struct)
**Kind**: data type
**One-liner**: Virtual Memory Area — a contiguous range of addresses in a process's `mm_struct` with uniform protections and backing.
**Source**: `include/linux/mm_types.h` (`struct vm_area_struct`)
**Notes**: Stored in a maple tree per `mm_struct`. (see [mm_struct])

### mm_struct
**Kind**: data type
**One-liner**: Per-process memory descriptor: page tables root (PGD), VMA tree, mmap_lock, refcount, etc.
**Source**: `include/linux/mm_types.h`
**Notes**: Shared between threads of one process; absent on kthreads. (see [task_struct], [vma (vm_area_struct)])

### slab / kmem_cache
**Kind**: subsystem + data type
**One-liner**: Object-cache allocator — `kmem_cache` is a pool of fixed-size objects served from a slab page; "slab" is the historical name kept for the API.
**Source**: `include/linux/slab.h`, `mm/slub.c` (the SLUB implementation; SLAB and SLOB are removed in current mainline)
**Notes**: `kmalloc` / `kmem_cache_alloc` route through SLUB. (see [GFP flags])

### GFP flags
**Kind**: bit-flag enum
**One-liner**: Get-Free-Page flags controlling allocator behavior: sleepable, atomic, NMI-safe, NUMA hint, zeroing, etc.
**Source**: `include/linux/gfp_types.h`, `include/linux/gfp.h`
**Notes**: Common values: `GFP_KERNEL` (sleepable), `GFP_ATOMIC` (no-sleep), `GFP_NOWAIT`, `GFP_USER`. Rookery's `KBox::new(value, flags)` mandates explicit flags.

### buddy allocator
**Kind**: algorithm
**One-liner**: The page-frame allocator that maintains free-list pools at power-of-2-page sizes and splits/merges blocks on alloc/free.
**Source**: `mm/page_alloc.c`
**Notes**: Backs SLUB and direct-page allocations; per-NUMA-node, per-zone. (see [zone], [node])

### zone
**Kind**: data type
**One-liner**: A region of physical memory with shared properties (DMA-reachable, normal, highmem on 32-bit, movable).
**Source**: `include/linux/mmzone.h` (`struct zone`)
**Notes**: A NUMA node has one or more zones. (see [node])

### node (NUMA)
**Kind**: data type
**One-liner**: A NUMA node — physical memory + CPU set with uniform access cost.
**Source**: `include/linux/mmzone.h` (`pg_data_t` aka `struct pglist_data`)
**Notes**: Use `for_each_online_node()` to iterate. (see [zone])

### vmalloc
**Kind**: API family
**One-liner**: Allocator for virtually-contiguous, physically-non-contiguous memory.
**Source**: `include/linux/vmalloc.h`, `mm/vmalloc.c`
**Notes**: More expensive than `kmalloc`; used when contiguous physical pages are unavailable.

### swap
**Kind**: subsystem
**One-liner**: Page-out mechanism evicting anonymous and shared pages to swap devices/files when memory pressure rises.
**Source**: `mm/swap.c`, `mm/swap_state.c`, `mm/page_io.c`, `include/linux/swap.h`
**Notes**: Distinct from page-cache writeback (file-backed eviction).

### THP (Transparent Huge Pages)
**Kind**: subsystem
**One-liner**: Mechanism that opportunistically backs anonymous and pagecache mappings with 2 MiB (or 1 GiB) pages.
**Source**: `mm/huge_memory.c`, `include/linux/huge_mm.h`
**Notes**: Tunable via `/sys/kernel/mm/transparent_hugepage/`.

---

## Synchronization and locking

### spinlock_t / raw_spinlock_t
**Kind**: data type
**One-liner**: Spin lock — busy-waits for the lock holder; `raw_spinlock_t` is the low-level unqueued variant; `spinlock_t` is the queued variant on PREEMPT_RT it morphs into a sleeping mutex.
**Source**: `include/linux/spinlock.h`, `include/linux/spinlock_types.h`
**Notes**: Rookery uses `kernel::sync::SpinLock<T>`. (see [mutex])

### mutex
**Kind**: data type
**One-liner**: Sleepable mutual-exclusion lock; the lock holder may sleep.
**Source**: `include/linux/mutex.h`, `kernel/locking/mutex.c`
**Notes**: Distinct from `binary semaphore` (`include/linux/semaphore.h`). Rookery uses `kernel::sync::Mutex<T>`.

### rw_semaphore
**Kind**: data type
**One-liner**: Sleepable reader-writer lock — many readers OR one writer.
**Source**: `include/linux/rwsem.h`, `kernel/locking/rwsem.c`
**Notes**: Used by `mm_struct.mmap_lock` historically. Rookery's `kernel::sync::RwSemaphore<T>` is on the to-author list (issue #4).

### seqlock
**Kind**: data type
**One-liner**: Sequence lock — readers retry on writer interleaving via a sequence-counter; writers serialize via a spinlock.
**Source**: `include/linux/seqlock.h`
**Notes**: Used for `jiffies` and other read-mostly hot paths.

### RCU (Read-Copy-Update)
**Kind**: subsystem + locking discipline
**One-liner**: A lock-free read protocol where readers traverse data unchanged while writers publish new copies; old copies freed after a grace period.
**Source**: `include/linux/rcupdate.h`, `kernel/rcu/`
**Notes**: Variants: classic RCU, SRCU, Tree RCU, Tasks RCU, Tasks Trace RCU. Rookery uses `kernel::sync::rcu::*`.

### atomic_t / atomic64_t
**Kind**: data type
**One-liner**: Atomic integer types with memory-ordered operations (`atomic_inc`, `atomic_cmpxchg`, …).
**Source**: `include/linux/atomic.h`, `include/asm-generic/atomic.h`, `arch/x86/include/asm/atomic.h`
**Notes**: Rookery uses `kernel::sync::atomic::Atomic*`.

### refcount_t
**Kind**: data type
**One-liner**: Overflow-checked saturating reference counter; `refcount_inc` saturates instead of wrapping past zero.
**Source**: `include/linux/refcount.h`
**Notes**: Replacement for unchecked `atomic_t` reference counts. Rookery uses `kernel::sync::Refcount`.

### lockdep
**Kind**: subsystem
**One-liner**: Runtime lock-ordering validator that detects potential deadlocks via lock-acquisition-graph analysis.
**Source**: `include/linux/lockdep.h`, `kernel/locking/lockdep.c`
**Notes**: `CONFIG_LOCKDEP`. Rookery's TLA+ models for concurrency primitives are an additional, static-time complement to lockdep.

---

## Time

### jiffies
**Kind**: scalar counter
**One-liner**: A monotonic counter incrementing at HZ rate (250/300/1000 per second), used for coarse-grained timing.
**Source**: `include/linux/jiffies.h`
**Notes**: 32-bit on some configs, 64-bit on x86_64. (see [HZ])

### HZ
**Kind**: configuration constant
**One-liner**: The system-tick frequency — number of `jiffies` increments per second.
**Source**: `include/asm-generic/param.h`, `include/linux/jiffies.h`
**Notes**: x86_64 default is 1000.

### hrtimer
**Kind**: data type + subsystem
**One-liner**: High-resolution timer — nanosecond-precision timer with a per-CPU rbtree of expirations.
**Source**: `include/linux/hrtimer.h`, `kernel/time/hrtimer.c`
**Notes**: Distinct from the older `timer_list` jiffies-based timer wheel.

### clocksource / clockevents
**Kind**: data types
**One-liner**: A `clocksource` reads a hardware monotonic counter (TSC, HPET); a `clockevents` device fires interrupts at programmed times.
**Source**: `include/linux/clocksource.h`, `include/linux/clockchips.h`
**Notes**: Used by hrtimer to schedule wakeups.

### ktime_t
**Kind**: scalar type
**One-liner**: 64-bit nanoseconds-since-boot timestamp used by hrtimer and high-precision time accounting.
**Source**: `include/linux/ktime.h`
**Notes**: Distinct from `timespec64` (split sec/nsec).

### tick
**Kind**: concept
**One-liner**: A periodic per-CPU timer interrupt that drives `jiffies` and the scheduler; can be stopped under `CONFIG_NO_HZ`.
**Source**: `kernel/time/tick-common.c`, `kernel/time/tick-sched.c`, `include/linux/tick.h`
**Notes**: NO_HZ_FULL eliminates ticks on isolated CPUs running a single task.

---

## VFS and filesystems

### inode
**Kind**: data type
**One-liner**: VFS object representing a file's metadata (size, perms, owner, timestamps, fs-specific ops vtable).
**Source**: `include/linux/fs.h` (`struct inode`)
**Notes**: One per file (across all hardlinks). (see [dentry])

### dentry
**Kind**: data type
**One-liner**: Directory entry — the path-component cache linking a name to an inode and its parent dentry.
**Source**: `include/linux/dcache.h` (`struct dentry`)
**Notes**: The dcache hash table accelerates path lookups. (see [path], [inode])

### file
**Kind**: data type
**One-liner**: A per-open-instance object holding a file position, mode flags, and pointers to the dentry and file_operations vtable.
**Source**: `include/linux/fs.h` (`struct file`)
**Notes**: One per `open(2)` call; multiple `file`s can refer to one inode.

### super_block
**Kind**: data type
**One-liner**: Per-mounted-filesystem-instance descriptor; holds the fs root inode and fs_operations.
**Source**: `include/linux/fs.h` (`struct super_block`)
**Notes**: Distinct from on-disk filesystem superblock; `super_block` is the in-memory mount instance.

### vfsmount / mount
**Kind**: data type
**One-liner**: An instance of a mounted filesystem in the mount-namespace tree.
**Source**: `include/linux/mount.h` (`struct vfsmount`), `fs/mount.h` (`struct mount`)
**Notes**: (see [namespace], [super_block])

### path
**Kind**: data type
**One-liner**: The pair `(vfsmount, dentry)` uniquely identifying a path-resolution result.
**Source**: `include/linux/path.h`
**Notes**: (see [dentry], [vfsmount / mount])

### page cache
**Kind**: subsystem
**One-liner**: The kernel's cache of file-backed pages keyed by `(inode, offset)`.
**Source**: `mm/filemap.c`, `include/linux/pagemap.h`
**Notes**: Now folio-based on current mainline. (see [folio])

---

## Networking

### sk_buff (skb)
**Kind**: data type
**One-liner**: Socket buffer — the canonical packet container used by every network protocol layer.
**Source**: `include/linux/skbuff.h`
**Notes**: Reference-counted; cloned via `skb_clone` for fanout. (see [net_device], [struct sock])

### net_device (netdev)
**Kind**: data type
**One-liner**: A network device — abstracts a physical NIC, virtual interface, or tunnel.
**Source**: `include/linux/netdevice.h` (`struct net_device`)
**Notes**: Iterated via `for_each_netdev`.

### struct sock
**Kind**: data type
**One-liner**: Protocol-agnostic kernel-side socket state (queues, refcount, owner cred, family-specific ops).
**Source**: `include/net/sock.h`
**Notes**: Each protocol (TCP, UDP, ...) has a `sock`-derived struct with extra state.

### NAPI
**Kind**: subsystem
**One-liner**: New API — interrupt-mitigation framework for receive: disable RX IRQ once packets arrive, poll until queue drains.
**Source**: `include/linux/netdevice.h`, `net/core/dev.c`
**Notes**: Reduces per-packet IRQ overhead at high rates.

### netfilter
**Kind**: subsystem
**One-liner**: Generic in-kernel packet-filter / NAT framework underpinning iptables / nftables / conntrack.
**Source**: `include/linux/netfilter.h`, `net/netfilter/`
**Notes**: Hook points: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING.

---

## Block I/O

### bio
**Kind**: data type
**One-liner**: A block I/O request — vector of (page, offset, length) tuples representing one read/write.
**Source**: `include/linux/blk_types.h` (`struct bio`)
**Notes**: Submitted via `submit_bio`; ends in a `bio_endio` callback.

### blk_mq
**Kind**: subsystem
**One-liner**: Multi-queue block layer — per-CPU submission queues feeding per-device hardware queues.
**Source**: `include/linux/blk-mq.h`, `block/blk-mq.c`
**Notes**: Replaces the older single-queue request_fn API.

### request_queue
**Kind**: data type
**One-liner**: Per-block-device queue holding pending requests, scheduler choice, and limits.
**Source**: `include/linux/blkdev.h` (`struct request_queue`)
**Notes**: One per `gendisk`. (see [gendisk])

### gendisk
**Kind**: data type
**One-liner**: A block device's identity (name, partitions, size, ops vtable).
**Source**: `include/linux/blkdev.h` (`struct gendisk`)
**Notes**: As of recent mainline, the historical `include/linux/genhd.h` no longer exists; `gendisk` lives in `blkdev.h`.

---

## IPC and signals

### futex
**Kind**: syscall + concept
**One-liner**: Fast Userspace muTEX — a userspace 32-bit word the kernel can park threads on; basis for pthread_mutex / pthread_cond.
**Source**: `include/linux/futex.h`, `kernel/futex/`
**Notes**: Userspace fast path; kernel slow path. `futex(2)` ops: WAIT, WAKE, REQUEUE, etc.

### signal
**Kind**: subsystem
**One-liner**: Asynchronous notification delivered to a task — terminates, stops, or invokes a handler depending on disposition.
**Source**: `include/linux/signal.h`, `kernel/signal.c`
**Notes**: Set via `sigaction(2)`; per-thread or per-process pending.

### pipe
**Kind**: data type + IPC mechanism
**One-liner**: Anonymous unidirectional FIFO between processes (or named via `mkfifo(2)`).
**Source**: `include/linux/pipe_fs_i.h`, `fs/pipe.c`
**Notes**: Capacity is page-aligned (default 16 pages = 64 KiB).

### eventfd / signalfd / timerfd
**Kind**: file-descriptor APIs
**One-liner**: User-pollable file descriptors that surface kernel events: counter (eventfd), pending signals (signalfd), timer expirations (timerfd).
**Source**: `fs/eventfd.c`, `fs/signalfd.c`, `fs/timerfd.c`, plus `include/linux/eventfd.h` etc.
**Notes**: Useful in `epoll`-driven event loops.

---

## Boot, init, and modules

### bzImage / vmlinuz / vmlinux
**Kind**: file format / artifacts
**One-liner**: `vmlinux` is the uncompressed kernel ELF; `vmlinuz` is the compressed bootable image; `bzImage` is the x86 specifically-formatted bootable image (gzip+setup).
**Source**: `arch/x86/boot/`, `Documentation/arch/x86/boot.rst`
**Notes**: Bootloaders parse the bzImage header per the boot protocol.

### initramfs
**Kind**: archive format
**One-liner**: A cpio archive unpacked into rootfs at boot, providing the initial userspace before the real root mount.
**Source**: `init/initramfs.c`, `Documentation/filesystems/ramfs-rootfs-initramfs.rst`

### start_kernel
**Kind**: function
**One-liner**: The first C function called once early arch setup completes; orchestrates all subsystem initialization.
**Source**: `init/main.c`
**Notes**: Calls `setup_arch`, `mm_init`, `sched_init`, `rest_init`, etc. (see [boot_params])

### boot_params
**Kind**: data type
**One-liner**: The struct the bootloader hands to the kernel via the x86 boot protocol — memory map, command line, framebuffer info, etc.
**Source**: `arch/x86/include/uapi/asm/bootparam.h`
**Notes**: Documented in `Documentation/arch/x86/zero-page.rst`.

### module (struct module)
**Kind**: data type
**One-liner**: A loadable kernel object (.ko) — module init/exit, exported symbols, refcount, dependencies.
**Source**: `include/linux/module.h`, `kernel/module/`
**Notes**: Loaded via `init_module(2)` / `finit_module(2)` / `modprobe(8)`.

### kallsyms
**Kind**: subsystem
**One-liner**: A compile-time-baked symbol table mapping kernel addresses to names; backs `/proc/kallsyms` and oops decoding.
**Source**: `include/linux/kallsyms.h`, `kernel/kallsyms.c`
**Notes**: Visibility controlled by `kptr_restrict`. (see [GRKERNSEC_HIDESYM] in `references/grsec-pax-notes.md`)

---

## Interrupts and deferred work

### IRQ / IDT
**Kind**: concept
**One-liner**: Hardware interrupt request handled via the Interrupt Descriptor Table (x86_64).
**Source**: `include/linux/interrupt.h`, `arch/x86/kernel/idt.c`
**Notes**: `request_irq` registers a handler. (see [hardirq], [softirq])

### hardirq / softirq
**Kind**: concept
**One-liner**: Hardirq is the (top-half) immediate IRQ handler running with IRQs disabled; softirq is the (bottom-half) deferred work running with IRQs enabled.
**Source**: `kernel/softirq.c`, `include/linux/interrupt.h`
**Notes**: Softirq vector enum: `HI`, `TIMER`, `NET_TX`, `NET_RX`, `BLOCK`, `IRQ_POLL`, `TASKLET`, `SCHED`, `HRTIMER`, `RCU`. (see [workqueue])

### workqueue
**Kind**: subsystem + data type
**One-liner**: Process-context deferred-work mechanism; `Work` items submitted to a `Queue` are run by a kernel worker thread.
**Source**: `include/linux/workqueue.h`, `kernel/workqueue.c`
**Notes**: Variants: `system`, `system_highpri`, `system_long`, `system_unbound`, `system_freezable`, `system_bh` (bottom-half class).

### tasklet
**Kind**: legacy data type
**One-liner**: Softirq-context deferred work primitive being phased out in favor of BH-class workqueues.
**Source**: `include/linux/interrupt.h` (`struct tasklet_struct`)
**Notes**: Rookery does not provide a Rust tasklet abstraction (per `00-rust-conventions.md`); use `Queue::system_bh()` instead.

### per-CPU variable
**Kind**: storage class
**One-liner**: A variable replicated per CPU; accessed via `this_cpu_*` macros or `per_cpu(name, cpu)`.
**Source**: `include/linux/percpu.h`, `include/linux/percpu-defs.h`
**Notes**: Defined via `DEFINE_PER_CPU(type, name)`. Rookery's `kernel::cpu::PerCpu<T>` is on the to-author list (issue #4).

---

## Security

### LSM (Linux Security Module)
**Kind**: subsystem
**One-liner**: A hook framework letting security modules (SELinux, AppArmor, Smack, TOMOYO, Yama, Landlock) intercept kernel security-relevant decisions.
**Source**: `include/linux/security.h`, `security/`
**Notes**: Stackable as of LSM v2.

### credentials (struct cred)
**Kind**: data type
**One-liner**: A task's security credentials: UIDs, GIDs, capabilities, security_label.
**Source**: `include/linux/cred.h`
**Notes**: Immutable; replaced atomically via `prepare_creds`/`commit_creds`.

### capability
**Kind**: concept
**One-liner**: Bit-flag privilege, replacing the all-or-nothing root model: e.g. `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_DAC_OVERRIDE`.
**Source**: `include/linux/capability.h`, `include/uapi/linux/capability.h`
**Notes**: Inherited / effective / permitted / bounding / ambient sets.

---

## System call surface

### syscall
**Kind**: concept + ABI
**One-liner**: A request from userspace into the kernel — entry via `syscall` instruction (x86_64) goes through `arch/x86/entry/` dispatching by syscall number to a `SYSCALL_DEFINE*` handler.
**Source**: `include/linux/syscalls.h`, `arch/x86/entry/`
**Notes**: x86_64 syscall numbers in `arch/x86/entry/syscalls/syscall_64.tbl`.

### vDSO
**Kind**: concept + artifact
**One-liner**: Virtual Dynamic Shared Object — a kernel-provided `.so` mapped into every process exposing fast-path syscalls (`clock_gettime`, `gettimeofday`, etc.) without an actual syscall.
**Source**: `arch/x86/entry/vdso/`
**Notes**: ELF-linked into each process at exec time; preserves ABI in REQ-5 of `00-overview.md`.

### ELF
**Kind**: file format
**One-liner**: Executable and Linkable Format — the binary format for executables, shared objects, and core dumps.
**Source**: `include/uapi/linux/elf.h`, `fs/binfmt_elf.c`
**Notes**: Loaded by the binfmt-elf binary formats handler.

---

## Observability

### ftrace
**Kind**: subsystem
**One-liner**: Function tracer — instruments kernel functions at runtime via dynamic patching of mcount/fentry call sites.
**Source**: `include/linux/ftrace.h`, `kernel/trace/`
**Notes**: Drives `/sys/kernel/tracing/`.

### perf_event
**Kind**: subsystem
**One-liner**: Performance-monitoring framework wrapping HW PMUs and SW events; exposed via `perf_event_open(2)`.
**Source**: `include/linux/perf_event.h`, `kernel/events/`
**Notes**: Drives the `perf` userspace tool.

### BPF / eBPF
**Kind**: subsystem
**One-liner**: An in-kernel sandboxed VM running verifier-checked bytecode programs attached to hooks (kprobes, tracepoints, XDP, sockets, LSM, ...).
**Source**: `include/linux/bpf.h`, `kernel/bpf/`, `net/core/filter.c`
**Notes**: Programs JITed on x86_64.

### io_uring
**Kind**: subsystem + ABI
**One-liner**: A two-ring (submission / completion) shared-memory I/O interface between userspace and kernel — alternative to epoll + read/write syscalls.
**Source**: `io_uring/io_uring.c`, `io_uring/io_uring.h`, `include/uapi/linux/io_uring.h`
**Notes**: Top-level kernel directory in current mainline (was `fs/io_uring.c` historically).

### KVM
**Kind**: subsystem
**One-liner**: Kernel-based Virtual Machine — exposes hardware virtualization (Intel VMX, AMD SVM) to userspace as `/dev/kvm` ioctls.
**Source**: `arch/x86/kvm/`, `virt/kvm/`, `include/linux/kvm_host.h`, `include/uapi/linux/kvm.h`
**Notes**: QEMU is the canonical userspace client.

---

## x86_64-specific

### MSR (Model-Specific Register)
**Kind**: hardware concept
**One-liner**: A CPU register identified by a 32-bit address, accessed via `RDMSR` / `WRMSR`.
**Source**: `arch/x86/include/asm/msr.h`, `arch/x86/include/asm/msr-index.h`
**Notes**: Used for IA32_EFER, IA32_LSTAR (syscall entry), IA32_GS_BASE, etc.

### CR0/CR2/CR3/CR4
**Kind**: control registers
**One-liner**: x86_64 control registers: paging enable + WP (CR0), page-fault address (CR2), top-level page-table physical address + PCID (CR3), feature enables (CR4: PAE, PSE, SMEP, SMAP, LA57, …).
**Source**: `arch/x86/include/asm/processor.h`, `arch/x86/include/uapi/asm/processor-flags.h`

### IDT (Interrupt Descriptor Table)
**Kind**: hardware structure
**One-liner**: 256-entry table the CPU consults on interrupt/exception dispatch — each entry names a code-segment + offset to the handler.
**Source**: `arch/x86/include/asm/desc.h`, `arch/x86/kernel/idt.c`
**Notes**: (see [IRQ / IDT])

### TSS (Task State Segment)
**Kind**: hardware structure
**One-liner**: Per-CPU structure holding the kernel stack pointer used on user→kernel transitions and the IO permission bitmap.
**Source**: `arch/x86/include/asm/processor.h`, `arch/x86/kernel/cpu/common.c`
**Notes**: x86_64 mostly uses TSS for stack switching; legacy task-switching is unused.

### kallsyms address-space-randomization (KASLR)
**Kind**: feature
**One-liner**: Kernel Address Space Layout Randomization — the kernel image base is randomized at each boot.
**Source**: `arch/x86/boot/compressed/kaslr.c`
**Notes**: Per-build-config; CONFIG_RANDOMIZE_BASE.

---

## Verification artifacts (Rookery-internal)

### Kani harness
**Kind**: artifact convention
**One-liner**: A `#[cfg(kani)] #[kani::proof] fn check_*()` body that exercises a code path under symbolic inputs to certify a SAFETY or invariant property.
**Source**: this project; per `00-rust-conventions.md`
**Notes**: Lives in each crate's `proofs/` subdirectory.

### TLA+ model
**Kind**: artifact convention
**One-liner**: A `.tla` specification of an abstract algorithm or data structure, accompanied by a `.cfg` and proven via TLC or Apalache.
**Source**: this project; per `00-rust-conventions.md`
**Notes**: Lives in each crate's `models/` subdirectory.

### SAFETY / PROOF comment pair
**Kind**: convention
**One-liner**: Inline annotations on every `unsafe` block — `// SAFETY:` describes invariants assumed; `// PROOF:` names the Kani harness that mechanically discharges them.
**Source**: this project; per `00-rust-conventions.md` REQ-8
**Notes**: Enforced by the Rookery `unsafe_without_proof` clippy lint (issue #3).

---

## Out of Scope

- Userspace-only terminology (libc, dynamic linker, systemd) — refer to upstream userspace docs.
- Architecture-specific terms outside x86_64 (until v1+ — see `00-overview.md` D3 / Out of Scope).
- Driver-class-specific terminology (USB descriptors, PCI BARs, GPU command rings) — those go in the relevant Tier-4 driver-class doc.
- Filesystem-specific on-disk formats (ext4 inode layout, btrfs subvolumes) — those go in per-FS Tier-3 docs.

## Appendix: validation script

```sh
# Verify every Source: line resolves under the linux source tree.
# Skip lines inside fenced code blocks (the format-template example).
awk '/^```/ { in_fence = !in_fence; next } !in_fence' .design/00-glossary.md \
  | grep -oE '\*\*Source\*\*: `[^`]+`' \
  | sed -E 's/.*`([^`]+)`.*/\1/' \
  | while read p; do
      first=$(echo "$p" | cut -d, -f1 | tr -d '` ')
      [ -e "/home/doll/linux-src/$first" ] && echo "OK  $first" || echo "MISS $first"
    done
```
