---
title: "Foundation: Master design-document index"
tags: ["design-doc", "foundation", "index"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---

# Foundation: Master design-document index

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
upstream-paths:
  - (project-wide registry)
status: draft
-->

## Summary
Single source of truth listing every Rookery design document, its tier, its status, and the upstream paths it covers. Updated whenever a document is added, status changes, or scope shifts. A reader scanning this file can answer: "what design coverage does Rookery have?" and "what's still missing?".

This file complements the crosslink knowledge base — every design doc is also stored as a knowledge page tagged `design-doc` plus tier-specific tags. The knowledge-page slug equals the doc's relative path with `/` replaced by `-` (e.g. `kernel/sched/cfs.md` → knowledge slug `kernel-sched-cfs`).

## Requirements

- REQ-1: Every design document under `.design/` (including this one) appears as a row in the Registry section.
- REQ-2: Every row records: relative path, tier (1–5 per `00-overview.md`), status (`draft` / `reviewed` / `approved`), baseline-commit (matching the doc's frontmatter), and a one-line summary.
- REQ-3: A "Coverage map" section enumerates each top-level upstream Linux subsystem directory and reports whether Rookery has at least a Tier-2 overview doc for it.
- REQ-4: A "Planned next" section lists the Phase B / Phase C / Phase D / Phase E docs we know we'll need but haven't written yet, with priority hints.
- REQ-5: Whenever a design doc's status advances (`draft` → `reviewed` → `approved`), this index is updated in the same commit.

## Acceptance Criteria

- [ ] AC-1: A walk over `.design/**/*.md` yields the same set of paths as the Registry section's path column. (covers REQ-1)
- [ ] AC-2: Every Registry row's `Source paths` field cites at least one upstream path verifiable under `/home/doll/linux-src/`. (covers REQ-2 partially — the row's accuracy)
- [ ] AC-3: The Coverage map names every directory listed by `ls /home/doll/linux-src/ | grep -v '\.'` (i.e. every top-level upstream subsystem) and marks each as covered / planned / out-of-scope. (covers REQ-3)
- [ ] AC-4: A grep for "Phase " in this file finds at least Phase B, C, D, E sections under "Planned next". (covers REQ-4)
- [ ] AC-5: Each registry row's tier matches the doc's filename convention (Tier 1: `.design/00-*.md`; Tier 2: `.design/<area>/00-*.md`; Tier 3: `.design/<area>/<comp>.md`; Tier 4: `.design/drivers/<class>/...md`; Tier 5: `.design/uapi/...md`). (covers REQ-2 partially — the row's tier accuracy)

## Architecture

The registry table format:

| Path | Tier | Status | Baseline | Source paths covered | Summary |
|---|---|---|---|---|---|

Tier values: `1-foundation` / `2-subsystem` / `3-component` / `4-driver` / `5-uapi`. A Tier 0 row exists for project-level references that aren't binding design docs (e.g. `references/grsec-pax-notes.md`).

Status values: `draft` / `reviewed` / `approved`. A `reviewed` doc has had a `--kind decision` sign-off comment on its tracking issue. An `approved` doc additionally has all linked subsystem docs at least `reviewed` and a baseline-commit matching `00-baseline.md`.

Baseline column carries the short SHA of the doc's frontmatter `baseline-commit` (e.g. `27a26ccfd5`). When the project-wide pin advances and a doc hasn't been re-reviewed, this column shows the *old* short SHA in red — making staleness visually obvious.

## Registry

Last updated: 2026-05-09. Project-wide baseline pin: `27a26ccfd528` (Linux 7.1.0-rc2).

| Path | Tier | Status | Baseline | Source paths covered | Summary |
|---|---|---|---|---|---|
| `00-overview.md`              | 1-foundation | draft | `27a26ccfd5` | (project-wide) | Project keystone. 15 REQs, 15 ACs, 6 resolved decisions (D1–D6), tiered design-doc layout, full-ABI compat-contract enumeration. |
| `00-baseline.md`              | 1-foundation | draft | `27a26ccfd5` | (project-wide) | Pins upstream commit + version, declares the per-doc frontmatter convention, documents the rebaseline procedure. |
| `00-rust-conventions.md`      | 1-foundation | draft | `27a26ccfd5` | `rust/`, `rust/kernel/`, `Documentation/rust/` | Rust idioms, no_std/alloc/panic/error model, no-async, unsafe + verification stack (Kani / TLA+ / Creusot-Verus-Prusti), kbuild integration. 17 REQs, 18 ACs, 2 resolved decisions. |
| `00-glossary.md`              | 1-foundation | draft | `27a26ccfd5` | `include/linux/`, `include/uapi/linux/`, `arch/x86/include/` | 91 kernel terms across process/scheduler, memory, locking, time, VFS, network, block, IPC, boot, IRQ, security, syscalls, observability, x86, and Rookery verification artifacts. |
| `00-index.md`                 | 1-foundation | draft | `27a26ccfd5` | (project-wide registry) | This document. Master registry of all design docs + coverage map + planned-next backlog. |
| `references/grsec-pax-notes.md` | 0-reference | draft | `linux-6.6.102` | `/home/doll/grsec-6.6.102.patch` | Reading list — grsecurity / PaX patchset feature catalog mapped to "free in Rust" / "type-system encodable" / "explicit per-subsystem policy" buckets. Source for the deferred `00-security-principles.md`. |
| `arch/00-overview.md`         | 2-subsystem | draft | `27a26ccfd5` | `arch/`, `arch/Kconfig`, `include/asm-generic/` | arch/ tier meta-doc: states v0 is x86_64-only, deferrals for 21 other arches, per-arch design-doc convention (10 required topics), arch-abstraction extraction policy. 4 REQs, 4 ACs, 2 open. |
| `arch/x86/00-overview.md`     | 2-subsystem | draft | `27a26ccfd5` | `arch/x86/` (full subtree) | Substantive x86_64 design: enumerates 19+ Tier-3 component docs (boot, entry, paging, kernel-platform, signal, vDSO, cpuinfo, ptrace-abi, ELF, mitigations, COCO, hyperv-guest, xen-guest, KVM, PCI, PMU, crypto-accel, BPF JIT, power, RAS), declares the x86 slice of the compat contract, sets verification-stack expectations (Layer 1 unsafe blocks, Layer 2 TLA+ for vDSO seqlock + SMP boot + IDT install + kexec hand-off, Layer 3 Kani for page-table walker + IDT/TSS/per-CPU invariants, Layer 4 opt-ins for AES-NI). 13 REQs, 13 ACs, 6 resolved decisions (D1–D6), 0 open. |
| `lib/00-overview.md`          | 2-subsystem | draft | `27a26ccfd5` | `lib/` (full subtree) | Kernel utility library: data structures, strings + vsprintf, compression codecs, checksums + hashes, lib/crypto/ lightweight primitives, math helpers, usercopy + iov_iter, debug tooling, KUnit, vdso-core. Spawns 12 Tier-3 docs. Compat surface: vsprintf %p extensions, dynamic_debug control ABI, KUnit TAP output, vdso_data layout, codec/checksum bit-identity. Heavy on Layer 3 (data-structure invariants) Kani harnesses; Layer 4 opt-ins for lightweight crypto + math. 10 REQs, 11 ACs, 0 open. |
| `mm/00-overview.md`           | 2-subsystem | draft | `27a26ccfd5` | `mm/` (full subtree) + key UAPI headers | Memory management — heart of the kernel. Spawns 24 Tier-3 docs covering page allocator (buddy + per-CPU), SLUB, virtual memory, mmap-family syscalls, page cache (folio), reclaim + OOM, swap + zswap + zsmalloc, THP + hugetlb, migration + compaction, KSM, userfaultfd, memcg, NUMA + mempolicy, memory hotplug, hwpoison, vmalloc, bootmem, percpu, HMM, sanitizers (KASAN/KMSAN/KFENCE), DAMON, debug. Compat surface: every mm syscall, /proc/<pid>/{maps,smaps,pagemap,numa_maps,status,oom_*}, /proc/{meminfo,buddyinfo,zoneinfo,vmstat,slabinfo,vmallocinfo,swaps}, /sys/kernel/mm/*, /sys/devices/system/{memory,node}/*, page-flag bit positions. One of four MANDATORY Layer-3 subsystems per D4 — Kani invariant harnesses required for buddy + slab freelist + VMA tree + LRU + page tables + folio refcount + swap slot + UFFD wait queue. Layer 2 TLA+ models for buddy, SLUB per-CPU, LRU, mmap_lock + per-VMA-lock, userfaultfd, swap_atomic, memcg hierarchy. 16 REQs, 15 ACs, 0 open. |
| `fs/00-overview.md`           | 2-subsystem | draft | `27a26ccfd5` | `fs/` (full subtree) + key UAPI headers | Virtual filesystem + 78 filesystem implementations. Tier-3 layout: vfs/ (core: inode, dcache, super, file-table, mount, path-resolution, fs-context), syscalls (140 syscalls), exec-binfmt (ELF loader), iomap, netfs, jbd2, fs-crypto, fs-verity, fsnotify, quota, buffer-cache, direct-io+dax, pipe-splice, event-fds, aio, charset, pseudo-fs/ (proc, sysfs, devpts, ramfs-tmpfs, hugetlbfs, debugfs, configfs, tracefs, efivarfs, pstore, kernfs), disk-fs/ (ext4 + btrfs + xfs + tmpfs + vfat + iso9660 + squashfs in v0; jbd2 + nfs + cifs + f2fs + exfat + ntfs3 + ecryptfs + autofs + ceph as upstream-C-via-FFI; rest CONFIG=n), network-fs/ (NFS client + 9p in v0; rest as upstream C or out), stacked-fs/ (overlayfs + fuse REQUIRED, ecryptfs + autofs as C). Compat: 140+ fs syscalls, /proc /sys /devpts paths, mount + namespace ABI, statx struct, fanotify/inotify event format, POSIX/OFD/lease locks, fs-crypto + fs-verity ABI, ELF loading, mountinfo format, on-disk format byte-identity for in-port-set FSes. fs/dcache is the fourth of four MANDATORY Layer-3 subsystems per D4 — Kani harnesses required for dcache tree + LRU + inode SB list + fdtable + mount tree + posix-locks. Layer 2 TLA+ models for dcache RCU-walk, mount propagation, concurrent inode link, file-lock compat, fsnotify event order, jbd2 journal replay. 17 REQs, 15 ACs (some cover multiple REQs), 4 open. |
| `block/00-overview.md`        | 2-subsystem | draft | `27a26ccfd5` | `block/` + key UAPI/kernel headers | Block I/O layer. 11 Tier-3 docs: blk-mq, request-queue, bio, block-ioctl, partitions (MBR/GPT/Mac/Amiga/IBM-DASD/LDM/...), io-schedulers (mq-deadline/kyber/bfq), cgroup-throttle (blk-cgroup/throttle/iolatency/iocost/ioprio), blk-crypto (inline encryption + fallback), bio-integrity (T10-PI), zoned, sed-opal. Compat: BLK* ioctl numbers + semantics (size, RO, discard, zeroout, secdiscard, partition mgmt, zoned-zone-mgmt, blktrace setup, SED Opal), /sys/block/<dev>/queue/* knobs, /proc/{diskstats,partitions}, partition-table byte-identical parses, T10-PI generation/verify. Layer 2 TLA+ models for tag allocation, request state machine, blkcg accounting, bio-integrity chain, elevator scheduling. Layer 4 opt-ins: partition parsers (Creusot), T10-PI CRC (Verus), blk-crypto key descriptors (Verus). 13 REQs, 13 ACs, 0 open. |
| `kernel/00-overview.md`       | 2-subsystem | draft | `27a26ccfd5` | `kernel/` (full subtree) + key UAPI headers | Kernel core — largest single subsystem. Spawns 30+ Tier-3 docs covering scheduler family (CFS/EEVDF, RT, deadline, ext, idle, pelt, topology, clock, cpufreq), locking (mutex, spinlock, rwsem, seqlock, RCU, lockdep), time (hrtimer, clocksource, tick, posix-timers, namespace), task lifecycle (fork/exit/exec/signal/kthread/cred/cap/pid/ns), futex, IRQ infra, workqueue + irq_work + stop_machine, SMP+hotplug, cgroup framework (core/cpuset/pids/freezer), BPF (verifier, programs, maps, BTF, LSM, iter, helpers, arena, struct_ops), perf events, tracing (ftrace, tracepoints, kprobes, uprobes, blktrace, eventfs), module loading, printk, power-mgmt, syscall-entry helpers, DMA-mapping, livepatch, KGDB, unwinder, coverage, audit, crash-kexec, panic-reboot, sysctl-ksysfs, kallsyms, runtime-codepatching, accounting. Compat: every kernel-side syscall (process lifecycle, signals, creds, sched, time, futex, BPF, perf, ftrace, modules, kexec), /proc/<pid>/{stat,status,sched,wchan,stack,...}, /proc/{interrupts,softirqs,sched_debug,timer_list,kallsyms,locks,...}, /sys/kernel/{tracing,debug,livepatch,btf}/*, /sys/devices/system/cpu/*, /sys/fs/{cgroup,bpf}/*, BTF binary, dmesg format. `kernel/sched/` is the third of four MANDATORY Layer-3 subsystems per D4 — Kani harnesses required for CFS rbtree ordering + RT priority arrays + DL deadline ordering + PI-graph acyclicity + cgroup hierarchical accounting. Layer 2 TLA+ models for runqueue, load-balance, preempt counter, qspinlock, qrwlock, RCU grace periods, percpu-rwsem, futex wait/wake, futex PI, cgroup hierarchy, workqueue pool, BPF verifier soundness, printk ringbuf, ftrace ringbuf. 19 REQs, 18 ACs, 5 open. |

## Coverage map

Top-level upstream Linux directories at baseline `27a26ccfd528`, listed via `ls /home/doll/linux-src | grep -v '\.'`:

| Upstream dir | Tier-2 design doc | Status |
|---|---|---|
| `arch/`         | `arch/00-overview.md`           | DRAFT (Phase B) |
| `arch/x86/`     | `arch/x86/00-overview.md`       | DRAFT (Phase B) |
| `block/`        | `block/00-overview.md`         | DRAFT (Phase B) |
| `certs/`        | (out-of-scope for v0 — module-signing certs storage; not a kernel subsystem in design sense) | OUT OF SCOPE v0 |
| `crypto/`       | `crypto/00-overview.md` (Phase B) | NOT WRITTEN |
| `drivers/`      | `drivers/00-overview.md` (Phase B; per-class docs in Phase E) | NOT WRITTEN |
| `Documentation/`| (NOT a subsystem; the Linux docs themselves) | OUT OF SCOPE |
| `fs/`           | `fs/00-overview.md`            | DRAFT (Phase B) |
| `include/`      | (NOT a subsystem; covered by `uapi/` Tier-5 docs in Phase D and per-subsystem docs) | (covered indirectly) |
| `init/`         | `init/00-overview.md` (Phase B) | NOT WRITTEN |
| `io_uring/`     | `io_uring/00-overview.md` (Phase B) | NOT WRITTEN |
| `ipc/`          | `ipc/00-overview.md` (Phase B) | NOT WRITTEN |
| `kernel/`       | `kernel/00-overview.md`        | DRAFT (Phase B) |
| `lib/`          | `lib/00-overview.md`           | DRAFT (Phase B) |
| `LICENSES/`     | (license texts — not a design subject) | OUT OF SCOPE |
| `mm/`           | `mm/00-overview.md`            | DRAFT (Phase B) |
| `net/`          | `net/00-overview.md` (Phase B) | NOT WRITTEN |
| `rust/`         | (existing rust-for-linux abstractions — referenced from `00-rust-conventions.md`; no separate Tier-2 doc needed) | covered by Tier 1 |
| `samples/`      | (sample code — not a design subject) | OUT OF SCOPE |
| `scripts/`      | (build scripts — not a design subject; kbuild integration covered in `00-rust-conventions.md`) | covered by Tier 1 |
| `security/`     | `security/00-overview.md` (Phase B; `00-security-principles.md` is the Tier-1 hardening doc tracked separately as issue #2) | NOT WRITTEN |
| `sound/`        | `sound/00-overview.md` (Phase B) | NOT WRITTEN |
| `tools/`        | (userspace tools shipping with kernel — out of scope for v0) | OUT OF SCOPE v0 |
| `usr/`          | (initramfs-cpio packing utility for build — minor; covered under `init/`) | (folded into init) |
| `virt/`         | `virt/00-overview.md` (Phase B; covers KVM core, x86 KVM in `arch/x86/kvm/`) | NOT WRITTEN |

**Coverage status**: 6 / 14 in-scope subsystems have Tier-2 overviews (arch + arch/x86/, lib, mm, kernel, fs, block). Phase B in progress. With mm/ + arch/x86/ both at draft, issue #2 (00-security-principles.md) becomes authorable once they reach `reviewed`.

## Planned next

### Phase B — subsystem overviews (Tier 2)

One `<subsystem>/00-overview.md` per in-scope upstream directory above. Each must:
- enumerate the subsystem's components (which become Tier-3 docs)
- declare the subsystem's compat-contract slice (which userspace-visible interfaces it owns)
- include a "Verification" section per `00-overview.md` REQ-13 / `00-rust-conventions.md` REQ-9
- list its Hardening dependencies (deferred until `00-security-principles.md` lands per D6, but stub a placeholder)

Suggested order (driven by dependency: lower-level subsystems first, so higher-level ones can reference them):

1. ~~`arch/00-overview.md`~~ — DRAFT 2026-05-09
2. ~~`arch/x86/00-overview.md`~~ — DRAFT 2026-05-09
3. ~~`lib/00-overview.md`~~ — DRAFT 2026-05-09
4. ~~`mm/00-overview.md`~~ — DRAFT 2026-05-09
5. ~~`kernel/00-overview.md`~~ — DRAFT 2026-05-09
6. ~~`fs/00-overview.md`~~ — DRAFT 2026-05-09
7. ~~`block/00-overview.md`~~ — DRAFT 2026-05-09
8. `net/00-overview.md` (core, ipv4, ipv6, sockets, NAPI, netfilter)
9. `ipc/00-overview.md` (msg/sem/shm + futex)
10. `crypto/00-overview.md` (crypto_alg, shash/aead/skcipher families)
11. `security/00-overview.md` (LSM hooks, credentials, capabilities)
12. `init/00-overview.md` (start_kernel, initramfs, kernel command line)
13. `io_uring/00-overview.md`
14. `virt/00-overview.md` (KVM core)
15. `drivers/00-overview.md` (driver model, base, per-bus skeletons)
16. `sound/00-overview.md` (ALSA core; lower priority for v0 — desktop audio is non-headless workload)

Once `mm/00-overview.md` and `arch/x86/00-overview.md` exist, unblock issue #2 to author `00-security-principles.md`.

### Phase C — component-level designs (Tier 3)

Per the registry conventions, each Tier-2 overview enumerates the Tier-3 docs it spawns. Confirmed Tier-3 docs we know we'll need (incomplete; will grow as Phase B proceeds):

- `arch/x86/boot.md` (incl. kexec hand-off ABI per REQ-15)
- `arch/x86/entry.md` (syscall + IRQ entry, signal frames, vDSO entry-points)
- `arch/x86/paging.md`
- `arch/x86/kernel-platform.md` (CPU bringup, TSS, IDT, FPU, MSRs)
- `kernel/sched/00-overview.md` + `cfs.md` (covers EEVDF), `rt.md`, `deadline.md`, `idle.md`, `pelt.md`, `topology.md`
- `kernel/locking/00-overview.md` + `spinlock.md`, `mutex.md`, `rwsem.md`, `seqlock.md`, `rcu.md`, `refcount.md`
- `kernel/time/00-overview.md` + `hrtimer.md`, `clocksource.md`, `tick.md`
- `kernel/task-lifecycle.md` (fork/exit/exec/signal)
- `kernel/cgroup/00-overview.md`
- `kernel/bpf/00-overview.md`
- `mm/page-allocator.md`, `mm/slab.md`, `mm/virtual-memory.md`, `mm/reclaim.md`, `mm/swap.md`, `mm/thp.md`
- `fs/vfs/00-overview.md` + `dcache.md`, `pagecache.md`, `mount-namespace.md`
- `fs/proc.md`, `fs/sysfs.md`, `drivers/base/devtmpfs.md` (compat-critical pseudo-FSes)
- `net/core/00-overview.md` + `skbuff.md`, `netdev.md`, `napi.md`, `netfilter.md`
- `net/ipv4/00-overview.md`, `net/ipv6/00-overview.md`
- `block/blk-mq.md`
- `ipc/futex.md` (because futex is its own substantial subsystem)
- `crypto/api.md`
- `security/lsm.md`
- `init/start-kernel.md`

Plus the missing-rust-for-linux abstractions tracked on issue #4 each become Tier-3 docs:
- `kernel/sync/rwsemaphore.md` (REQ-1 of issue #4)
- `kernel/task/kthread-spawn.md`
- `kernel/cpu/percpu.md`

### Phase D — UAPI + per-syscall specs (Tier 5)

Per-header (~30) and per-syscall (~400) specs covering the userspace-visible ABI. Volume estimate: ~430 docs. Examples:

- `uapi/socket.md`, `uapi/stat.md`, `uapi/signalfd.md`, `uapi/io_uring.md`, ...
- `uapi/syscalls/openat2.md`, `uapi/syscalls/clone3.md`, `uapi/syscalls/io_uring_setup.md`, ...

Authoring strategy: generate skeleton docs from `arch/x86/entry/syscalls/syscall_64.tbl` plus `include/uapi/`, then fill semantically per-syscall. The skeleton-generation script itself is a deliverable in this phase.

### Phase E — driver-class designs + v0 port set (Tier 4)

Per `00-overview.md` D3:
- Per-class overviews: `drivers/{usb,net,block,char,gpu,input,scsi,mmc,...}/00-overview.md`
- Per-bus overviews: `drivers/base/00-overview.md`, `drivers/pci/00-overview.md`, `drivers/usb/core/00-overview.md`
- Per-driver designs for the v0 port set:
  - `drivers/block/virtio-blk.md`
  - `drivers/net/virtio-net.md`
  - `drivers/tty/virtio-console.md`
  - `drivers/net/e1000e.md`
  - `drivers/ata/ahci.md`
  - `drivers/usb/host/ehci.md`
  - `drivers/usb/host/xhci.md`
  - `drivers/input/keyboard/atkbd.md` (PS/2)
  - `drivers/hid/hid-input.md`
  - `drivers/tty/vt/00-overview.md` + `vt-console.md`

## Out of Scope

- Documents that aren't binding design specs (release notes, READMEs, CHANGELOGs) — those belong in repository-root markdown files, not under `.design/`.
- Implementation-side documentation (rustdoc, in-source comments) — those live in the implementation tree, not here.
- Non-x86_64 arch docs in v0 — placeholders only when an arch is named in a Tier-2 overview.
