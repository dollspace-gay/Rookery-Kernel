---
title: "Rookery-Kernel — project overview"
tags: ["design-doc", "foundation"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Rookery-Kernel is a from-scratch Rust rewrite of the Linux kernel, targeting full-ABI drop-in compatibility with upstream Linux on x86_64 hardware. Existing GNU/Linux distributions, kernel modules built against upstream, and userspace must boot and run unchanged when the upstream `bzImage` is replaced by Rookery's. The project follows a **GPL-clean reference** methodology: we read upstream Linux freely as a behavioral and ABI reference, but write all code from scratch in idiomatic Rust under GPL-2.0.

This document is the project's keystone contract. Every other design document in `.design/` is constrained by the requirements, conventions, and out-of-scope clauses recorded here.

### Requirements

- REQ-1: **Drop-in substitution** — the produced `rookery-bzImage` MUST be substitutable for an upstream Linux 7.1.0-rc2 `bzImage` on x86_64 hardware, with no modification to userspace, initramfs, or bootloader configuration. The system boots to a usable userspace identically.
- REQ-2: **Syscall ABI parity** — every syscall enumerated in `arch/x86/entry/syscalls/syscall_64.tbl` (and its 32-bit-compat counterpart) MUST be implemented with byte-identical entry/exit conventions, register usage, error-return semantics, and argument/result struct layouts (per `include/uapi/`).
- REQ-3: **Pseudo-FS parity** — `/proc`, `/sys`, `/dev` (devtmpfs), `/sys/kernel/debug` (debugfs) MUST present the same paths and the same content formats (where format is documented or de-facto standardized) as a stock Linux 7.1 kernel running on the same hardware.
- REQ-4: **Module loading conventions** — kernel modules built from the same source against Rookery's `vmlinux` MUST load and behave identically to those built against upstream's `vmlinux`. (Strict CONFIG_MODVERSIONS CRC parity for binaries built against upstream is an open question — see Q2 below.)
- REQ-5: **VDSO parity** — the vDSO image exposed to userspace (`linux-vdso.so.1`) MUST export the same symbols (`__vdso_clock_gettime`, `__vdso_gettimeofday`, `__vdso_time`, `__vdso_getcpu`) with the same calling conventions.
- REQ-6: **x86_64 only for v0** — design coverage in v0 is limited to `arch/x86/`. Other architectures get placeholder design docs only.
- REQ-7: **GPL-clean reference methodology** — implementation reads upstream source as a behavioral reference; copies no source code, including comments and identifier names where doing so would amount to literal copying. Output is GPL-2.0 to match upstream.
- REQ-8: **kbuild integration** — Rust crates compile via the same kbuild rules used by `rust/` in upstream; Cargo workspaces live inside the kernel source tree; the build invocation `make olddefconfig && make -j$(nproc)` produces a Rookery `bzImage` from a Rookery checkout exactly as it does for upstream.
- REQ-9: **Sync-only concurrency** — kernel control flow uses kthreads, workqueues, tasklets, softirqs, hardirqs, and explicit state machines. Rust `async/await` is forbidden in v0; revisitable per design doc with a written justification reviewed by a project-level decision.
- REQ-10: **No new features beyond baseline** — Rookery does not add functionality not present in Linux 7.1.0-rc2. Net-new features become a v1+ activity.
- REQ-11: **Document hierarchy mirrors upstream** — design docs are organized in nested subdirectories under `.design/` that mirror upstream's source-tree layout (`.design/kernel/sched/cfs.md` describes what `kernel/sched/fair.c` does upstream, etc.). Foundational docs prefix `00-` and live at `.design/` root or at the top of each subsystem subdirectory.
- REQ-12: **Spec-first, code-later** — this design corpus is consumed by a separate implementation instance. No design doc instructs the reader to write code; specs describe behavior, structures, and Rust module shape. Implementation choices below the spec level are the implementor's prerogative.

### Acceptance Criteria

- [ ] AC-1: The five Phase-A foundational documents (`00-overview.md`, `00-baseline.md`, `00-rust-conventions.md`, `00-glossary.md`, `00-index.md`) are committed to `.design/`.
- [ ] AC-2: `00-baseline.md` pins the exact upstream commit (`27a26ccfd528da725a999ea1e3102503c61eb655`) and version (`7.1.0-rc2`), with the verification commands present and runnable. (covers REQ-1)
- [ ] AC-3: Every Phase-A document's Architecture section references at least one real upstream path verifiable with `ls /home/doll/linux-src/<path>`. (covers REQ-1, REQ-11)
- [ ] AC-4: `00-rust-conventions.md` explicitly states "async forbidden in v0", documents the `no_std` / `alloc` / `panic_handler` / error-handling stance, and references `rust/kernel/` as the reference for existing rust-for-linux abstractions. (covers REQ-9)
- [ ] AC-5: `00-glossary.md` defines at least 40 kernel-specific terms with explicit references to where each is introduced in upstream (e.g. `task_struct` → `include/linux/sched.h`).
- [ ] AC-6: `00-index.md` enumerates every design doc in the project with: relative path, tier (foundation/subsystem/component/driver/uapi), status (draft/reviewed/approved), and a one-line summary. The index is updated each time a new doc is added or a status changes.
- [ ] AC-7: For each top-level Linux subsystem directory (`arch`, `block`, `crypto`, `drivers`, `fs`, `init`, `io_uring`, `ipc`, `kernel`, `lib`, `mm`, `net`, `security`, `sound`, `virt`), Phase B produces an `00-overview.md` enumerating its components and mapping them to planned design-doc paths. (covers REQ-11)
- [ ] AC-8: Each subsystem overview includes a "Compatibility contract" section that names the userspace-visible structs, paths, files, and constants it must preserve. (covers REQ-2, REQ-3, REQ-4, REQ-5)
- [ ] AC-9: A subsystem design doc cannot graduate to status `approved` while any `<!-- OPEN -->` block remains.
- [ ] AC-10: A subsystem design doc cannot graduate to status `reviewed` until a user has signed off via a `crosslink issue comment --kind decision` on the design's tracking issue.

### Architecture

(All upstream paths are relative to `/home/doll/linux-src` at baseline `27a26ccfd528`.)

### Tiered design-doc layout

We organize design documents in five tiers that mirror the upstream Linux directory tree. The tier of a doc determines its required template fields and review depth.

| Tier | Path pattern | Purpose | Examples |
|---|---|---|---|
| 1: Foundation | `.design/00-*.md` | Project-wide contracts: this doc, baseline, conventions, glossary, index | `00-overview.md`, `00-rust-conventions.md` |
| 2: Subsystem | `.design/<subsys>/00-overview.md` | One per top-level Linux dir; enumerates components | `kernel/00-overview.md`, `mm/00-overview.md` |
| 3: Component | `.design/<subsys>/<component>.md` | One per logical component within a subsystem | `kernel/sched/cfs.md`, `mm/page-allocator.md` |
| 4: Driver | `.design/drivers/<class>/00-overview.md` plus per-driver `.md` | Driver-class designs and individual driver designs for the v0 port set | `drivers/net/00-overview.md`, `drivers/net/e1000.md` |
| 5: UAPI | `.design/uapi/<header>.md` and `.design/uapi/syscalls/<name>.md` | Per-header struct/constant specs and per-syscall semantic specs | `uapi/socket.md`, `uapi/syscalls/openat2.md` |

Foundation docs may be referenced from any tier. Tier-2+ docs reference all higher tiers (e.g. a Tier 4 driver doc may reference a Tier 3 driver-bus doc).

### Key upstream references and their planned design-doc homes

| Upstream path | Planned design doc | Tier |
|---|---|---|
| `arch/x86/boot/` | `arch/x86/boot.md` | 3 |
| `arch/x86/entry/` | `arch/x86/entry.md` | 3 |
| `arch/x86/kernel/` (CPU bringup, traps, FPU, signal frames) | `arch/x86/kernel-platform.md` | 3 |
| `arch/x86/mm/` | `arch/x86/paging.md` | 3 |
| `arch/x86/kvm/` | `arch/x86/kvm.md` (under `virt/00-overview.md`) | 3 |
| `kernel/sched/` | `kernel/sched/00-overview.md` + `cfs.md`, `rt.md`, `deadline.md`, `idle.md`, `pelt.md`, `topology.md` | 3 |
| `kernel/locking/` | `kernel/locking/00-overview.md` + per-primitive docs | 3 |
| `kernel/time/` | `kernel/time/00-overview.md` + `hrtimer.md`, `clocksource.md`, `tick.md` | 3 |
| `kernel/fork.c`, `kernel/exit.c`, `kernel/signal.c` | `kernel/task-lifecycle.md` | 3 |
| `mm/page_alloc.c`, `mm/slub.c`, `mm/memory.c`, `mm/vmscan.c` | `mm/page-allocator.md`, `mm/slab.md`, `mm/virtual-memory.md`, `mm/reclaim.md` | 3 |
| `fs/namespace.c`, `fs/super.c`, `fs/inode.c`, `fs/dcache.c`, `fs/file.c` | `fs/vfs/00-overview.md` + per-component | 3 |
| `fs/proc/`, `fs/sysfs/`, `drivers/base/devtmpfs.c` | `fs/proc.md`, `fs/sysfs.md`, `drivers/base/devtmpfs.md` | 3 (compat-critical) |
| `net/core/`, `net/ipv4/`, `net/ipv6/`, `net/socket.c` | `net/00-overview.md` + per-component | 3 |
| `block/blk-core.c`, `block/blk-mq.c` | `block/00-overview.md` + `blk-mq.md` | 3 |
| `ipc/msg.c`, `ipc/sem.c`, `ipc/shm.c` | `ipc/00-overview.md` + per-API | 3 |
| `crypto/api.c`, `crypto/algapi.c` | `crypto/00-overview.md` + `api.md` | 3 |
| `security/security.c`, `security/<lsm>/` | `security/00-overview.md` + per-LSM | 3 |
| `io_uring/io_uring.c` | `io_uring/00-overview.md` | 3 |
| `rust/kernel/` (existing rust-for-linux abstractions) | `rust-bindings.md` (Tier 1; conventions doc) | 1 |
| `include/uapi/` headers | `.design/uapi/*.md` | 5 |

### Compatibility contract enumeration

The drop-in compat target (REQ-1) is the union of the following preserved interfaces. Each subsystem doc enumerates the slice of this union it owns.

| Interface class | Examples | Preservation level |
|---|---|---|
| Syscall numbers + register conventions | `read=0`, `write=1`, ... | Byte-identical (REQ-2) |
| UAPI struct layouts | `struct stat`, `struct iovec`, `struct sigaction`, `struct epoll_event` | Byte-identical layout, same field semantics |
| UAPI constants | `O_*`, `MAP_*`, `PROT_*`, `SIG*`, `EPOLL*` | Numeric value identity |
| /proc paths | `/proc/cpuinfo`, `/proc/meminfo`, `/proc/<pid>/maps`, `/proc/sys/...` | Path + content format identity (where format documented) |
| /sys paths | `/sys/devices/...`, `/sys/class/...`, `/sys/block/...`, `/sys/kernel/...` | Path + attribute file identity |
| /dev nodes (devtmpfs) | `/dev/null`, `/dev/zero`, `/dev/tty`, `/dev/random`, char/block major-minor numbers | Identity per `Documentation/admin-guide/devices.txt` |
| vDSO symbols | `__vdso_clock_gettime` etc. | ABI-identical (REQ-5) |
| Kernel command line | `Documentation/admin-guide/kernel-parameters.txt` | Same parameters, same parse semantics |
| ioctl numbers | `_IO/_IOR/_IOW/_IOWR` macros and per-driver assignments | Numeric identity per upstream `include/uapi/` |
| sysfs/procfs ABI promises | per `Documentation/ABI/` | All `stable/`, all `testing/` (best effort), `obsolete/` (declined) |

A subsystem design doc's "Compatibility contract" section enumerates the row of this table it owns.

### Rust crate organisation (delegated, not designed here)

This document deliberately does not specify the Rust workspace layout (crate names, dependency graph, where `unsafe` boundaries live in module form). That is the implementing instance's call, constrained by the Rust conventions doc and rust-for-linux's existing layout under `rust/kernel/`. The conventions doc establishes guardrails; the workspace shape is a downstream consequence.

### Quality gates

A subsystem design doc graduates through three statuses:

1. **draft** — committed to `.design/` but not yet reviewed.
2. **reviewed** — a user has signed off via a `crosslink issue comment --kind decision`. All `<!-- OPEN -->` blocks resolved.
3. **approved** — `reviewed` plus: all linked subsystem docs are also at least `reviewed`; baseline-commit in frontmatter matches `00-baseline.md`'s pinned commit.

A doc cannot graduate while:
- Any `<!-- OPEN -->` block remains
- Any REQ-N has no matching AC-N
- Any architecture path doesn't resolve under `/home/doll/linux-src`
- The frontmatter's `baseline-commit` is older than `00-baseline.md`'s pinned commit

### Out of Scope

- **Architectures other than x86_64 in v0** — `arch/{arm,arm64,riscv,powerpc,...}/` design docs deferred. Placeholder docs may exist with `status: deferred-v1`.
- **Userspace** — busybox, glibc, systemd, GRUB, util-linux are upstream-of-the-kernel concerns. We accept them as-is.
- **Bootloader code** — we accept the x86 boot protocol contract; no bootloader is rewritten.
- **New features beyond Linux 7.1.0-rc2** — no innovation past upstream functionality. Innovation is a v1+ activity, gated behind a separate project decision.
- **Rust workspace shape (crate layout, dep graph)** — implementing instance's prerogative, constrained by `00-rust-conventions.md`.
- **CI/CD pipelines** — operator concern; not modeled in design docs.
- **Implementation code** — this design corpus produces specs only. The implementing instance handles all `.rs` and Makefile authorship in a separate session.
- **Strict CONFIG_MODVERSIONS binary-module compat** — explicitly deferred to v1+ pending Q2 resolution.
- **Subsystems already deprecated upstream** — anything in `Documentation/process/deprecated.rst` or marked `obsolete` in `Documentation/ABI/` is skipped unless a design doc explicitly opts in.

### project vision

The Linux kernel is the most successful systems-software artifact in history, but its 25M-line C codebase carries the cumulative weight of three decades of memory-unsafe practice. Modern Rust offers the first realistic chance to express the same ABI in a memory-safe language without rewriting the surrounding ecosystem. Rookery-Kernel is the bet that we can.

We are not innovating on the kernel/userspace boundary. We are not adding new features. We are not changing the syscall surface, the procfs layout, or the module-loading conventions. Every visible behavior of upstream Linux at the same baseline version is something we strive to reproduce exactly. Innovation, where it occurs, is purely internal: better data structures, safer locking primitives, sharper compile-time checks, more aggressive elimination of `unsafe`. Userspace cannot tell the difference at runtime.

### foundational decisions

These four decisions, recorded as `--kind decision` comments on the project's tracking issue, govern everything downstream. They are not revisited inside individual subsystem docs.

| Axis | Decision |
|---|---|
| RE methodology | GPL-clean reference: read upstream freely; copy no code; output GPL-2.0 |
| Architecture priority | x86_64 only for v0; other arches deferred to v1+ |
| Build system | Integrate with kbuild (Make-driven); follow rust-for-linux conventions |
| Concurrency model | Sync only; kthreads + workqueues + tasklets + explicit state machines; `async/await` forbidden in v0 |

