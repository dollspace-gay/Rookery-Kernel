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
| `00-security-principles.md`   | 1-foundation | draft | `27a26ccfd5` | `Documentation/security/`, `security/Kconfig`, `include/uapi/linux/{elf,prctl}.h`, `fs/binfmt_elf.c`, `mm/{mmap,mprotect}.c`, `arch/x86/include/asm/nospec-branch.h` | Binding security policy. Distills `security/00-overview.md` § Locked-in row-1 absorption list into a Tier-1 reference cited by every Tier-3 Hardening section. Locks: 4 axioms (memory-protection ON, policy-enforcement via GR-RBAC, LSM-framework intact, opt-in/opt-out only); per-feature default-on policy; new exec-gain ELF note (`NT_ROOKERY_SECURITY_FLAGS=0x52455253` in `.note.rookery.security`); new prctl `PR_REQUEST_EXEC_GAIN=80`; GR-RBAC stackable LSM positioning (loaded LATE in stack, empty default policy); consolidated knob inventory (Kconfig + sysctl + cmdline + prctl + ELF notes); Hardening-section template Tier-3 docs follow. 10 REQs, 10 ACs, 0 open. |
| `references/grsec-pax-notes.md` | 0-reference | draft | `linux-6.6.102` | `/home/doll/grsec-6.6.102.patch` | Reading list — grsecurity / PaX patchset feature catalog mapped to "free in Rust" / "type-system encodable" / "explicit per-subsystem policy" buckets. Source for the deferred `00-security-principles.md`. |
| `arch/00-overview.md`         | 2-subsystem | draft | `27a26ccfd5` | `arch/`, `arch/Kconfig`, `include/asm-generic/` | arch/ tier meta-doc: states v0 is x86_64-only, deferrals for 21 other arches, per-arch design-doc convention (10 required topics), arch-abstraction extraction policy. 4 REQs, 4 ACs, 2 open. |
| `arch/x86/00-overview.md`     | 2-subsystem | draft | `27a26ccfd5` | `arch/x86/` (full subtree) | Substantive x86_64 design: enumerates 19+ Tier-3 component docs (boot, entry, paging, kernel-platform, signal, vDSO, cpuinfo, ptrace-abi, ELF, mitigations, COCO, hyperv-guest, xen-guest, KVM, PCI, PMU, crypto-accel, BPF JIT, power, RAS), declares the x86 slice of the compat contract, sets verification-stack expectations (Layer 1 unsafe blocks, Layer 2 TLA+ for vDSO seqlock + SMP boot + IDT install + kexec hand-off, Layer 3 Kani for page-table walker + IDT/TSS/per-CPU invariants, Layer 4 opt-ins for AES-NI). 13 REQs, 13 ACs, 6 resolved decisions (D1–D6), 0 open. |
| `lib/00-overview.md`          | 2-subsystem | draft | `27a26ccfd5` | `lib/` (full subtree) | Kernel utility library: data structures, strings + vsprintf, compression codecs, checksums + hashes, lib/crypto/ lightweight primitives, math helpers, usercopy + iov_iter, debug tooling, KUnit, vdso-core. Spawns 12 Tier-3 docs. Compat surface: vsprintf %p extensions, dynamic_debug control ABI, KUnit TAP output, vdso_data layout, codec/checksum bit-identity. Heavy on Layer 3 (data-structure invariants) Kani harnesses; Layer 4 opt-ins for lightweight crypto + math. 10 REQs, 11 ACs, 0 open. |
| `mm/00-overview.md`           | 2-subsystem | draft | `27a26ccfd5` | `mm/` (full subtree) + key UAPI headers | Memory management — heart of the kernel. Spawns 24 Tier-3 docs covering page allocator (buddy + per-CPU), SLUB, virtual memory, mmap-family syscalls, page cache (folio), reclaim + OOM, swap + zswap + zsmalloc, THP + hugetlb, migration + compaction, KSM, userfaultfd, memcg, NUMA + mempolicy, memory hotplug, hwpoison, vmalloc, bootmem, percpu, HMM, sanitizers (KASAN/KMSAN/KFENCE), DAMON, debug. Compat surface: every mm syscall, /proc/<pid>/{maps,smaps,pagemap,numa_maps,status,oom_*}, /proc/{meminfo,buddyinfo,zoneinfo,vmstat,slabinfo,vmallocinfo,swaps}, /sys/kernel/mm/*, /sys/devices/system/{memory,node}/*, page-flag bit positions. One of four MANDATORY Layer-3 subsystems per D4 — Kani invariant harnesses required for buddy + slab freelist + VMA tree + LRU + page tables + folio refcount + swap slot + UFFD wait queue. Layer 2 TLA+ models for buddy, SLUB per-CPU, LRU, mmap_lock + per-VMA-lock, userfaultfd, swap_atomic, memcg hierarchy. 16 REQs, 15 ACs, 0 open. |
| `fs/00-overview.md`           | 2-subsystem | draft | `27a26ccfd5` | `fs/` (full subtree) + key UAPI headers | Virtual filesystem + 78 filesystem implementations. Tier-3 layout: vfs/ (core: inode, dcache, super, file-table, mount, path-resolution, fs-context), syscalls (140 syscalls), exec-binfmt (ELF loader), iomap, netfs, jbd2, fs-crypto, fs-verity, fsnotify, quota, buffer-cache, direct-io+dax, pipe-splice, event-fds, aio, charset, pseudo-fs/ (proc, sysfs, devpts, ramfs-tmpfs, hugetlbfs, debugfs, configfs, tracefs, efivarfs, pstore, kernfs), disk-fs/ (ext4 + btrfs + xfs + tmpfs + vfat + iso9660 + squashfs in v0; jbd2 + nfs + cifs + f2fs + exfat + ntfs3 + ecryptfs + autofs + ceph as upstream-C-via-FFI; rest CONFIG=n), network-fs/ (NFS client + 9p in v0; rest as upstream C or out), stacked-fs/ (overlayfs + fuse REQUIRED, ecryptfs + autofs as C). Compat: 140+ fs syscalls, /proc /sys /devpts paths, mount + namespace ABI, statx struct, fanotify/inotify event format, POSIX/OFD/lease locks, fs-crypto + fs-verity ABI, ELF loading, mountinfo format, on-disk format byte-identity for in-port-set FSes. fs/dcache is the fourth of four MANDATORY Layer-3 subsystems per D4 — Kani harnesses required for dcache tree + LRU + inode SB list + fdtable + mount tree + posix-locks. Layer 2 TLA+ models for dcache RCU-walk, mount propagation, concurrent inode link, file-lock compat, fsnotify event order, jbd2 journal replay. 17 REQs, 15 ACs (some cover multiple REQs), 4 open. |
| `net/00-overview.md`          | 2-subsystem | draft | `27a26ccfd5` | `net/` (full subtree) + key UAPI headers | Networking stack — second-largest subsystem after fs/. 30+ Tier-3 docs in nested subtrees: socket-api, skbuff, netdev, struct-sock, bpf-net, fib, netlink, packet, unix-socket, ipv4/ (00-overview + tcp, udp, icmp, igmp, fib4, fragment, multicast, raw, arp, gre-tunnels, mroute), ipv6/ (analogous), netfilter, traffic-control, xfrm, l2/ (bridge, vlan, dsa, switchdev), ethtool, devlink, tls, mptcp, sctp, wireless/ (cfg80211, mac80211, ieee802154 — FFI'd in v0), bluetooth (FFI'd v0), sunrpc, vsock, 9p-transport, niche-transports. Compat: every socket syscall, /proc/net/*, /sys/class/net/*, TCP/UDP/IP wire formats, netlink + nfnl + tc + xfrm + nl80211 + bt-mgmt protocols, every SO_*/TCP_*/IP_* sockopt, ethtool ABI, devlink ABI, kTLS sockopts, NF hook chains. Layer 2 TLA+ models for skb refcount, NAPI state, conntrack lifecycle, TCP RFC 9293 state, FIB RCU, rtnl big-mutex. Layer 4 opt-ins: TCP state machine (Verus — RFC 9293 conformance), netlink parser (Creusot), xfrm replay window, XDP verifier (cross-ref kernel/bpf). 18 REQs, 17 ACs (some span multiple REQs), 3 open. |
| `block/00-overview.md`        | 2-subsystem | draft | `27a26ccfd5` | `block/` + key UAPI/kernel headers | Block I/O layer. 11 Tier-3 docs: blk-mq, request-queue, bio, block-ioctl, partitions (MBR/GPT/Mac/Amiga/IBM-DASD/LDM/...), io-schedulers (mq-deadline/kyber/bfq), cgroup-throttle (blk-cgroup/throttle/iolatency/iocost/ioprio), blk-crypto (inline encryption + fallback), bio-integrity (T10-PI), zoned, sed-opal. Compat: BLK* ioctl numbers + semantics (size, RO, discard, zeroout, secdiscard, partition mgmt, zoned-zone-mgmt, blktrace setup, SED Opal), /sys/block/<dev>/queue/* knobs, /proc/{diskstats,partitions}, partition-table byte-identical parses, T10-PI generation/verify. Layer 2 TLA+ models for tag allocation, request state machine, blkcg accounting, bio-integrity chain, elevator scheduling. Layer 4 opt-ins: partition parsers (Creusot), T10-PI CRC (Verus), blk-crypto key descriptors (Verus). 13 REQs, 13 ACs, 0 open. |
| `ipc/00-overview.md`          | 2-subsystem | draft | `27a26ccfd5` | `ipc/` + key UAPI headers | System V IPC + POSIX mqueue. 5 Tier-3 docs: sysv-msg, sysv-sem, sysv-shm, posix-mqueue, ipc-namespace. Compat: msgget/semget/shmget/mq_* syscalls + struct ipc_perm/msqid_ds/semid_ds/shmid_ds + IPC namespaces + /proc/sysvipc/* + /proc/sys/kernel/{msgmax,msgmni,sem,shmmax,shmmni}. 6 REQs, 6 ACs, 0 open. |
| `init/00-overview.md`         | 2-subsystem | draft | `27a26ccfd5` | `init/` + `usr/` | Kernel initialization. 3 Tier-3 docs: start-kernel, initramfs, rootfs-mount. Compat: kernel cmdline parameter parsing dispatch, dmesg ordering, /proc/{cmdline,version,version_signature,config.gz}, BogoMIPS, initramfs cpio + decompressors. 8 REQs, 8 ACs, 0 open. |
| `io_uring/00-overview.md`     | 2-subsystem | draft | `27a26ccfd5` | `io_uring/` + key UAPI headers | High-performance async I/O via shared SQ/CQ rings. 14 Tier-3 docs: core, op-dispatch, filetable-rsrc, sqpoll-iowq, fs-ops, net-ops, poll-ops, wait-cancel, buffers, task-context, register, uring-cmd, bpf-integration. Compat: io_uring_setup/enter/register syscalls, mmap'd ring layout (sqe/cqe + offsets), every IORING_OP_* + IORING_REGISTER_* command. Layer 2 TLA+ models for sq_cq_ring (lock-free producer/consumer ordering), iowq, task_work. Layer 4 opt-in: SQ/CQ memory-ordering correctness via Verus. 10 REQs, 8 ACs, 1 open. |
| `crypto/00-overview.md`       | 2-subsystem | draft | `27a26ccfd5` | `crypto/` + key headers | Kernel cryptographic API. 10+ Tier-3 docs: api, af-alg, skcipher/ (AES, ChaCha, etc.), aead, hash, asymmetric, compression, rng, async-tx, fips. Compat: /proc/crypto + AF_ALG socket protocol + every algorithm name + bit-identical algorithm output. **Layer 4 verification MANDATORY for v0 critical-path crypto** (chacha20, poly1305, blake2s, libsha256, hmac(sha256), aes, ecdsa P-256, curve25519). 10 REQs, 10 ACs, 1 open. |
| `security/00-overview.md`     | 2-subsystem | draft | `27a26ccfd5` | `security/` + key headers | LSM framework + 9 in-tree LSMs. 15+ Tier-3 docs: lsm-framework, capabilities, selinux (FFI), apparmor (FFI), smack (FFI), tomoyo (FFI), yama, loadpin, landlock, lockdown, ipe (FFI), integrity (FFI - IMA+EVM), keys, safesetid, bpf-lsm. Compat: every LSM's userspace ABI (selinuxfs, apparmor profiles, landlock syscalls, IMA measurement list, keyctl), capability bits, stackable-LSM dispatch. 16 REQs, 12 ACs (some span multiple REQs), 2 open (per-LSM port set + 00-security-principles authoring timing). |
| `virt/00-overview.md`         | 2-subsystem | draft | `27a26ccfd5` | `virt/kvm/` + `virt/lib/` + key headers | KVM cross-arch virtualization core. 2 Tier-3 docs: kvm-core, virt-lib. (x86-specific KVM in arch/x86/kvm.md.) Compat: every /dev/kvm ioctl + struct kvm_run mmap layout + dirty-ring + irqfd + ioeventfd + KVM_GET_STATS_FD + MMU-notifier callback. v0 graduation gate: kvm-unit-tests passes. 10 REQs, 9 ACs, 0 open. |
| `drivers/00-overview.md`      | 2-subsystem | draft | `27a26ccfd5` | `drivers/` (full subtree) + key headers | Device drivers — class-tiered with named v0 port set. ~30 Tier-3 docs (per-class + per-bus): base, pci, usb, virtio, acpi, block, net, tty, ata, input, hid, scsi (FFI), nvme, mmc (FFI), md (FFI), char, iommu, dma-buf, firmware, clocksource, clk, cpufreq-cpuidle, regulator, power-thermal-hwmon, i2c-spi-gpio, rtc, watchdog, gpu (STUB — DRM deferred to v1+). Per-driver Tier-4 docs ONLY for v0 port set: virtio-{blk,net,console}, e1000e, ahci, ehci/xhci, ps2-keyboard, hid-input, vt-console. Compat: /sys/{bus,class,devices}/* layout + udev events + /dev/* major:minor + per-bus ABI (lspci/lsusb/...). 15 REQs, 14 ACs, 2 open (staging, GPU/DRM). |
| `sound/00-overview.md`        | 2-subsystem | draft | `27a26ccfd5` | `sound/` + key headers | ALSA audio framework. **FFI'd in v0** — Rookery's primary deployment context (server/containers/VMs) doesn't need audio in Rust. 1 Tier-3 doc: core (FFI-shim spec). Compat: /dev/snd/* nodes + /proc/asound/* + every SNDRV_*_IOCTL + alsa-utils + PulseAudio + PipeWire transparent. 8 REQs, 8 ACs, 1 open (timeline for eventual Rust port). |
| `kernel/00-overview.md`       | 2-subsystem | draft | `27a26ccfd5` | `kernel/` (full subtree) + key UAPI headers | Kernel core — largest single subsystem. Spawns 30+ Tier-3 docs covering scheduler family (CFS/EEVDF, RT, deadline, ext, idle, pelt, topology, clock, cpufreq), locking (mutex, spinlock, rwsem, seqlock, RCU, lockdep), time (hrtimer, clocksource, tick, posix-timers, namespace), task lifecycle (fork/exit/exec/signal/kthread/cred/cap/pid/ns), futex, IRQ infra, workqueue + irq_work + stop_machine, SMP+hotplug, cgroup framework (core/cpuset/pids/freezer), BPF (verifier, programs, maps, BTF, LSM, iter, helpers, arena, struct_ops), perf events, tracing (ftrace, tracepoints, kprobes, uprobes, blktrace, eventfs), module loading, printk, power-mgmt, syscall-entry helpers, DMA-mapping, livepatch, KGDB, unwinder, coverage, audit, crash-kexec, panic-reboot, sysctl-ksysfs, kallsyms, runtime-codepatching, accounting. Compat: every kernel-side syscall (process lifecycle, signals, creds, sched, time, futex, BPF, perf, ftrace, modules, kexec), /proc/<pid>/{stat,status,sched,wchan,stack,...}, /proc/{interrupts,softirqs,sched_debug,timer_list,kallsyms,locks,...}, /sys/kernel/{tracing,debug,livepatch,btf}/*, /sys/devices/system/cpu/*, /sys/fs/{cgroup,bpf}/*, BTF binary, dmesg format. `kernel/sched/` is the third of four MANDATORY Layer-3 subsystems per D4 — Kani harnesses required for CFS rbtree ordering + RT priority arrays + DL deadline ordering + PI-graph acyclicity + cgroup hierarchical accounting. Layer 2 TLA+ models for runqueue, load-balance, preempt counter, qspinlock, qrwlock, RCU grace periods, percpu-rwsem, futex wait/wake, futex PI, cgroup hierarchy, workqueue pool, BPF verifier soundness, printk ringbuf, ftrace ringbuf. 19 REQs, 18 ACs, 5 open. |

## Coverage map

Top-level upstream Linux directories at baseline `27a26ccfd528`, listed via `ls /home/doll/linux-src | grep -v '\.'`:

| Upstream dir | Tier-2 design doc | Status |
|---|---|---|
| `arch/`         | `arch/00-overview.md`           | DRAFT (Phase B) |
| `arch/x86/`     | `arch/x86/00-overview.md`       | DRAFT (Phase B) |
| `block/`        | `block/00-overview.md`         | DRAFT (Phase B) |
| `certs/`        | (out-of-scope for v0 — module-signing certs storage; not a kernel subsystem in design sense) | OUT OF SCOPE v0 |
| `crypto/`       | `crypto/00-overview.md`        | DRAFT (Phase B) |
| `drivers/`      | `drivers/00-overview.md`       | DRAFT (Phase B; per-class Tier-3 docs + per-driver Tier-4 v0 port-set docs come in Phase C+) |
| `Documentation/`| (NOT a subsystem; the Linux docs themselves) | OUT OF SCOPE |
| `fs/`           | `fs/00-overview.md`            | DRAFT (Phase B) |
| `include/`      | (NOT a subsystem; covered by `uapi/` Tier-5 docs in Phase D and per-subsystem docs) | (covered indirectly) |
| `init/`         | `init/00-overview.md`          | DRAFT (Phase B) |
| `io_uring/`     | `io_uring/00-overview.md`      | DRAFT (Phase B) |
| `ipc/`          | `ipc/00-overview.md`           | DRAFT (Phase B) |
| `kernel/`       | `kernel/00-overview.md`        | DRAFT (Phase B) |
| `lib/`          | `lib/00-overview.md`           | DRAFT (Phase B) |
| `LICENSES/`     | (license texts — not a design subject) | OUT OF SCOPE |
| `mm/`           | `mm/00-overview.md`            | DRAFT (Phase B) |
| `net/`          | `net/00-overview.md`           | DRAFT (Phase B) |
| `rust/`         | (existing rust-for-linux abstractions — referenced from `00-rust-conventions.md`; no separate Tier-2 doc needed) | covered by Tier 1 |
| `samples/`      | (sample code — not a design subject) | OUT OF SCOPE |
| `scripts/`      | (build scripts — not a design subject; kbuild integration covered in `00-rust-conventions.md`) | covered by Tier 1 |
| `security/`     | `security/00-overview.md`      | DRAFT (Phase B; `00-security-principles.md` is the Tier-1 hardening doc tracked separately as issue #2) |
| `sound/`        | `sound/00-overview.md`         | DRAFT (Phase B; FFI'd in v0) |
| `tools/`        | (userspace tools shipping with kernel — out of scope for v0) | OUT OF SCOPE v0 |
| `usr/`          | (initramfs-cpio packing utility for build — minor; covered under `init/`) | (folded into init) |
| `virt/`         | `virt/00-overview.md`          | DRAFT (Phase B; covers KVM core; x86 KVM in arch/x86/kvm.md) |

**Coverage status**: **14 / 14 in-scope subsystems have Tier-2 overviews. PHASE B COMPLETE 2026-05-09.** Phase C (component-level Tier-3 designs) is the next stage. With mm/, arch/x86/, and security/ now drafted, issue #2 (`00-security-principles.md`) becomes authorable once those reach `reviewed`.

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
8. ~~`net/00-overview.md`~~ — DRAFT 2026-05-09
9. ~~`ipc/00-overview.md`~~ — DRAFT 2026-05-09
10. ~~`crypto/00-overview.md`~~ — DRAFT 2026-05-09
11. ~~`security/00-overview.md`~~ — DRAFT 2026-05-09
12. ~~`init/00-overview.md`~~ — DRAFT 2026-05-09
13. ~~`io_uring/00-overview.md`~~ — DRAFT 2026-05-09
14. ~~`virt/00-overview.md`~~ — DRAFT 2026-05-09
15. ~~`drivers/00-overview.md`~~ — DRAFT 2026-05-09
16. ~~`sound/00-overview.md`~~ — DRAFT 2026-05-09 (FFI'd v0 — minimal Rookery work)

**Phase B complete.** All 14 in-scope subsystems have Tier-2 overviews. ~225 Tier-3 component docs are spawned by these overviews (each names its planned children); Phase C authors them in dependency order.

**Phase C in progress (started 2026-05-09).** 16 Tier-3 docs drafted so far:
- `lib/data-structures.md` (foundational; used by every subsystem)
- `lib/usercopy.md` (UDEREF + USERCOPY enforcement)
- `arch/x86/boot.md` (substrate)
- `arch/x86/entry.md` (syscall, IRQ, exception entry)
- `arch/x86/paging.md` (page tables + fault handling + ioremap)
- `arch/x86/kernel-platform.md` (CPU bringup + IDT + TSS + FPU + time)
- `arch/x86/cpu-mitigations.md` (Spectre, kCFI, CET, IBT, retpolines)
- `arch/x86/signal.md` (signal frames + sigreturn)
- `arch/x86/vdso.md` (vDSO + vsyscall page)
- `mm/page-allocator.md` (buddy allocator — MANDATORY L3)
- `mm/slab.md` (SLUB — MANDATORY L3)
- `mm/virtual-memory.md` (mm_struct + VMAs — MANDATORY L3)
- `mm/mmap.md` (mmap-family syscalls + MPROTECT enforcement)
- `kernel/task-lifecycle.md` (fork/exit/exec/signal/kthread/cred/cap/pid/ns + PR_REQUEST_EXEC_GAIN)
- `fs/exec-binfmt.md` (exec + binfmt_elf + ELF note recognition for exec-gain)
- `fs/vfs/dcache.md` (directory entry cache — MANDATORY L3, the fourth)

All four MANDATORY Layer-3 subsystems per `00-overview.md` D4 now have Tier-3 design docs (mm: page-allocator, slab, virtual-memory; arch/x86: paging; fs: vfs/dcache). The verification mandate is fully scoped.

Phase C totals: 16/~225 Tier-3 docs (~7%). The substrate (boot/entry/paging/kernel-platform/mitigations + mm core + task lifecycle + exec-binfmt + dcache + lib core) is largely covered.

Once `mm/00-overview.md` and `arch/x86/00-overview.md` exist, unblock issue #2 to author `00-security-principles.md`.

### Phase C — component-level designs (Tier 3)

Per the registry conventions, each Tier-2 overview enumerates the Tier-3 docs it spawns. Confirmed Tier-3 docs we know we'll need (incomplete; will grow as Phase B proceeds):

- ~~`arch/x86/boot.md`~~ — DRAFT 2026-05-09 (incl. kexec hand-off ABI per REQ-15)
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
