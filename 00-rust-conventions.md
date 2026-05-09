---
title: "Foundation: Rust conventions for Rookery"
tags: ["design-doc", "foundation"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---

# Foundation: Rust conventions for Rookery

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
upstream-paths:
  - rust/
  - rust/kernel/
  - rust/Makefile
  - Documentation/rust/
  - scripts/min-tool-version.sh
status: draft
-->

## Summary
Defines the project-wide Rust conventions that bind every implementation crate, every subsystem design doc, and every verification artifact in Rookery. These conventions extend the existing rust-for-linux conventions in upstream `rust/` rather than replacing them — when in doubt, match upstream. The major Rookery-specific addition is the formal-verification baseline (`00-overview.md` REQ-13 / D4): every `unsafe` block carries a Kani-checked SAFETY proof, concurrency primitives ship TLA+ models, and critical data-structure invariants are Kani-checked.

## Requirements

- REQ-1: **Compatibility with rust-for-linux** — every existing abstraction in upstream `rust/kernel/` (Arc, Atomic, Lock<T, B: Backend>, Rcu, Refcount, error::Error, etc.) is the canonical implementation. Rookery extends; it does not fork. New abstractions live alongside upstream's, not in place of them.
- REQ-2: **Toolchain** — Rookery uses the same Rust toolchain rust-for-linux specifies via `scripts/min-tool-version.sh`. At baseline `27a26ccfd528`: rustc ≥ 1.85.0, bindgen ≥ 0.71.1, LLVM ≥ 15.0.0, edition 2021. Rookery does not advance MSRV unilaterally — it tracks upstream.
- REQ-3: **`no_std`** — every Rookery crate is `#![no_std]`. No `std`, ever. Imports come from `core`, `alloc` (when allocation is allowed in the context), and the project's `kernel` crate.
- REQ-4: **Allocation** — `alloc` is permitted but every allocation site MUST specify a context (sleepable / atomic / NMI) and a fallibility mode (fallible by default; infallible only in init paths). The `alloc` crate's default infallible API (`Box::new`, `Vec::push`) is forbidden in driver and core paths; use upstream's `KBox::new(value, GFP_KERNEL)` and friends.
- REQ-5: **Panic discipline** — `panic!()`, `unwrap()`, `expect()`, `unreachable!()`, `todo!()`, `unimplemented!()`, and arithmetic that can panic (integer overflow in debug, divide-by-zero, slice out-of-bounds) are forbidden in production code paths. The only legal panic is via the project's `kernel::panic!` macro which routes to the kernel BUG/panic handler. Slice indexing uses `.get()`, arithmetic uses `checked_*` / `saturating_*` / `wrapping_*` explicitly.
- REQ-6: **Error model** — fallible operations return `Result<T, kernel::error::Error>`. The error type is the existing rust-for-linux newtype around `NonZeroI32` representing kernel errnos (`-EXXX` from `include/uapi/asm-generic/errno-base.h` and `errno.h`). Subsystem-specific error types are forbidden; map to the errno table.
- REQ-7: **`async`/`await` forbidden** — `async fn`, `async {}` blocks, `.await`, `Future`, and any executor (tokio, async-std, smol, custom) are FORBIDDEN in v0. Concurrency is expressed as kthreads, workqueues, tasklets, softirqs, and explicit state machines. Revisitable per design doc with project-level approval.
- REQ-8: **Unsafe discipline** — `unsafe` is permitted only behind a documented abstraction boundary. Every `unsafe { ... }` block MUST carry: (a) a `// SAFETY:` comment naming the invariants the caller relies on, AND (b) a `// PROOF:` comment naming a Kani harness that mechanically checks the block under those invariants. Code without both does not compile (enforced by clippy lint `kernel::unsafe_without_proof`, to be authored).
- REQ-9: **Verification artifacts coexist with code** — every Rookery crate has a `proofs/` subdirectory (Kani harnesses), a `models/` subdirectory (TLA+ specs for any crate that exposes a concurrency primitive), and an optional `contracts/` subdirectory (Creusot / Verus / Prusti annotations). The crate's `Kbuild` declares which verification levels apply and how to invoke the verifier.
- REQ-10: **Build via kbuild** — Rookery crates compile via the same kbuild rules upstream uses for `rust/`. No standalone `cargo build` for kernel artifacts. The kbuild target `make rookery-vmlinux` produces a Rookery `bzImage` from a Rookery-instrumented Linux source tree.
- REQ-11: **Lint baseline** — clippy is mandatory (`make CLIPPY=1`). Rookery adds project-specific lints in a clippy-driver plugin: `unsafe_without_proof`, `panic_in_kernel_path`, `async_in_kernel`, `bare_arithmetic_on_size`. All lints are deny-level unless the file has an `#![allow(...)]` justified by an inline comment that names a tracking issue.
- REQ-12: **Format via rustfmt** — default `rustfmt` settings (4-space indent, the rust-for-linux convention). `make rustfmtcheck` is a CI gate.
- REQ-13: **Module documentation** — every `pub` item has rustdoc. Every module file (`mod.rs` or `<name>.rs` at module root) opens with a `//!` block summarizing purpose and naming the upstream C counterpart via `[`include/...`](srctree/include/...)` links (matching upstream rust-for-linux convention).
- REQ-14: **Naming** — `snake_case` for fns/vars, `PascalCase` for types, `SCREAMING_SNAKE_CASE` for consts and statics, matching rust-for-linux. Module names mirror upstream component names where reasonable (e.g. `kernel::sched::cfs` corresponds to `kernel/sched/fair.c`).
- REQ-15: **Lifetimes don't span syscall boundaries** — references handed to or from userspace are `'static` or owned. No lifetime parameter in any function signature reachable from a syscall entry point unless it's bound to a structurally-owning type (`&self` on a `Pin<&'a SyscallContext>` is OK; `&'a [u8]` returned from a syscall is not).
- REQ-16: **FFI discipline** — every `extern "C"` declaration lives in `bindings/` (auto-generated by bindgen) or a `ffi.rs` module. Hand-written `extern "C"` outside these locations is forbidden.
- REQ-17: **Forbidden unstable features** — `#![feature(...)]` is permitted only for the set rust-for-linux already uses (currently `arbitrary_self_types`, `derive_coerce_pointee`, `used_with_arg`, etc., per `rust/kernel/lib.rs`). Adding a new unstable feature requires a project-level decision recorded as a `--kind decision` comment on a tracking issue.

## Acceptance Criteria

- [ ] AC-1: A reader can grep `git -C ~/linux-src grep -l '#!\[no_std\]' rust/` and find every existing rust-for-linux crate; Rookery's crates appear in the same listing. (covers REQ-3)
- [ ] AC-2: The `kernel::error::Error` type is referenced by every Rookery `Result` return; no subsystem-defined error enums exist outside what upstream already offers. (covers REQ-6)
- [ ] AC-3: `make CLIPPY=1` on a Rookery checkout completes without warnings; `unsafe_without_proof`, `panic_in_kernel_path`, `async_in_kernel`, `bare_arithmetic_on_size` lints are wired and trigger on test fixtures. (covers REQ-8, REQ-11)
- [ ] AC-4: `make rookery-vmlinux` produces a bootable bzImage via kbuild without invoking standalone `cargo build`. (covers REQ-10)
- [ ] AC-5: `make verify` runs every Kani harness in every `proofs/` directory and reports per-crate pass/fail. `make tla` runs TLC over every `.tla` model in every `models/` directory and reports per-model pass/fail. `make verify-strict` additionally runs Creusot/Verus/Prusti targets per subsystem `Kbuild` declaration. (covers REQ-9)
- [ ] AC-6: A grep for `unsafe {` in Rookery code (excluding `bindings/`) returns no matches lacking either a `// SAFETY:` or a `// PROOF:` comment. CI fails the build if such a match exists. (covers REQ-8)
- [ ] AC-7: A grep for `\.await\b\|\basync fn\|async {` in Rookery code returns zero matches outside test fixtures. (covers REQ-7)
- [ ] AC-8: Every `pub` item has rustdoc; `make rustdoc` succeeds; broken intra-doc links fail the build. (covers REQ-13)
- [ ] AC-9: Every Rookery crate's `Cargo.toml` (or kbuild equivalent) declares `edition = "2021"` and the workspace-level toolchain pin matches `scripts/min-tool-version.sh rustc`. (covers REQ-2)
- [ ] AC-10: Every Rookery crate has at least an empty `proofs/`, `models/`, and `contracts/` directory committed (so the convention is mechanically checkable); the directories are populated as the subsystem matures. (covers REQ-9)
- [ ] AC-11: A grep over Rookery code for `Box::new\|Vec::new\|Vec::with_capacity\|Vec::push\|Arc::new` (excluding test fixtures) returns zero matches; the project lint `alloc_without_gfp` triggers on a test fixture. (covers REQ-4)
- [ ] AC-12: A grep over Rookery code for `panic!\|\.unwrap()\|\.expect(\|unreachable!\|todo!\|unimplemented!` (excluding test fixtures) returns zero matches; the project lint `panic_in_kernel_path` triggers on a test fixture. (covers REQ-5)
- [ ] AC-13: `make rustfmtcheck` is wired as a CI gate and reports diff-clean on every Rookery commit. (covers REQ-12)
- [ ] AC-14: A linter or convention-check script verifies every Rookery module's path mirrors an upstream Linux directory or file (per the Module organization section); deviations require a `// MAPPING:` comment naming the upstream counterpart. (covers REQ-14)
- [ ] AC-15: A static-analysis pass over every syscall entry function (in `arch/x86/entry/` Rust counterparts) verifies no transitive callee returns or accepts a borrowed reference whose lifetime is bound to a non-`'static` parameter on the syscall's signature. (covers REQ-15)
- [ ] AC-16: A grep for `extern "C"` in Rookery code returns matches only inside `bindings/` directories or files named `ffi.rs`. (covers REQ-16)
- [ ] AC-17: The set of `#![feature(...)]` attributes used in Rookery code is a subset of the set used in upstream `rust/kernel/lib.rs` at the project's baseline-commit pin; a CI check diffs the two sets and fails on a Rookery-specific addition without a corresponding `--kind decision` comment on a tracking issue. (covers REQ-17)
- [ ] AC-18: A subsystem implementation crate cannot land without first landing in upstream rust-for-linux's `rust/kernel/` either: (a) the abstractions it depends on, OR (b) Rookery-authored Tier 3 design docs for the missing abstractions enumerated in this doc's concurrency-primitives table. (covers REQ-1)

## Architecture

### Toolchain pin

Rust toolchain is managed identically to rust-for-linux:
- Read minimum version: `scripts/min-tool-version.sh rustc` → `1.85.0` at baseline. Bindgen: `scripts/min-tool-version.sh bindgen` → `0.71.1`.
- Distributions and `rustup` setup: `Documentation/rust/quick-start.rst` is authoritative.
- `rust-toolchain.toml` (if present) is allowed but discouraged; track upstream's policy.
- Cross-compilation for x86_64-pc-linux-gnu builds the kernel; for verification (Kani / cargo-kani), a separate cargo invocation may run on the host. Verification builds are NOT required to share the kernel target triple.

### no_std + alloc model

Every Rookery crate is `#![no_std]`. The standard library is unavailable; only:
- `core` — always available
- `alloc` — available where allocation is permitted
- `kernel` — the upstream rust-for-linux project crate, which exposes safe wrappers over kernel C APIs (Arc, KBox, KVec, Lock, Rcu, etc.)

The `alloc` crate's default infallible API (`Box::new`, `Vec::push`, `String::push`, etc.) is **forbidden** in driver and core paths. Use the upstream `kernel::alloc` family:

| `alloc` API (forbidden in driver/core) | `kernel` API (use this) |
|---|---|
| `Box::new(x)` | `KBox::new(x, GFP_KERNEL)` (sleepable) or `KBox::new(x, GFP_ATOMIC)` (atomic context) |
| `Vec::new()` | `KVec::new()` (the upstream alias) |
| `Vec::with_capacity(n)` | `KVec::with_capacity(n, GFP_KERNEL)` |
| `String::new()` | `KString` (or stay with `&'static CStr`) |
| `Arc::new(x)` | `kernel::sync::Arc::new(x, GFP_KERNEL)` |

GFP flag selection is a contextual decision: see `Documentation/core-api/memory-allocation.rst` upstream. Allocation must be fallible (returns `Result`). Infallible allocation is permitted only in `__init` paths (kernel boot) under explicit annotation.

### Panic discipline

The kernel cannot generally unwind. `panic!` from Rust code must route to the kernel `panic()` C function (in `kernel/panic.c`). Upstream provides `kernel::panic!` for this; **the bare `core::panic!` macro is forbidden** in Rookery code.

The forbidden-panic-source list:
- `panic!()`, `core::panic!()`
- `Option::unwrap`, `Result::unwrap`, `expect`
- `unreachable!()`, `todo!()`, `unimplemented!()`
- `[]` slice indexing where the index is not a compile-time-known in-bounds value (use `.get(i)` returning `Option`)
- Bare arithmetic on user-controlled values where overflow can occur — use `checked_*` / `saturating_*` / `wrapping_*` explicitly
- `assert!` / `assert_eq!` — use `kernel::build_assert!` for compile-time checks; for runtime checks that should never fire, use `WARN_ON`/`BUG_ON`-equivalent macros routed via `kernel::pr_warn` and panic-or-not policy from upstream

The clippy lint `panic_in_kernel_path` enforces this in production code paths (allowed in test fixtures and harnesses).

### Error model

Every fallible operation returns `Result<T, kernel::error::Error>`. The `Error` type is upstream's:

```rust
// rust/kernel/error.rs (upstream)
pub struct Error(NonZeroI32);

pub mod code {
    pub const EPERM: Error = ...;   // -1
    pub const ENOENT: Error = ...;  // -2
    pub const ESRCH: Error = ...;   // -3
    // ... full table mirroring uapi errno headers
}
```

Project conventions:

1. **No subsystem-specific `enum FooError`** — map to the errno table. If a syscall semantically lacks an errno, file a tracking issue and use `EINVAL` plus a structured log line (`pr_warn`).
2. **`?` operator everywhere** — never `match err { Err(e) => return Err(e), ... }`. Conversion via `From<E> for Error` (already provided by upstream for `LayoutError`, `AllocError`, `TryFromIntError`, `Utf8Error`).
3. **Operator-recoverable errors only** — return errors the caller (kernel core OR userspace via the syscall return) can act on. Internal invariant violations are `BUG_ON` / `WARN_ON`, not `Err`.

### Concurrency model — sync only

`async`/`await` is forbidden (REQ-7). Concurrency primitives — the third column distinguishes what already exists in upstream rust-for-linux at baseline `27a26ccfd528` from what Rookery must author. Adding a new primitive means adding a Tier 3 design doc that specs both the Rust API and the TLA+ model (Layer 2 verification).

| Need | Use | Upstream status (baseline `27a26ccfd528`) |
|---|---|---|
| Sleepable mutual exclusion | `kernel::sync::Mutex<T>` (`Lock<T, MutexBackend>`) | Exists — `rust/kernel/sync/lock/mutex.rs` |
| Spin (interrupt-safe) mutual exclusion | `kernel::sync::SpinLock<T>` (`Lock<T, SpinLockBackend>`) | Exists — `rust/kernel/sync/lock/spinlock.rs` |
| Reader-writer (sleepable) | `kernel::sync::RwSemaphore<T>` | **Rookery to author** — no upstream rust binding for `rwsem` yet. New Tier 3 doc + TLA+ model required. |
| Reference count | `kernel::sync::Refcount` (overflow-checked) | Exists — `rust/kernel/sync/refcount.rs` |
| Atomic load/store/CAS | `kernel::sync::atomic::Atomic*` | Exists — `rust/kernel/sync/atomic/` |
| Memory barriers | `kernel::sync::barrier::*` | Exists — `rust/kernel/sync/barrier.rs` |
| RCU read-side | `kernel::sync::rcu::*` (Guard + Token) | Exists — `rust/kernel/sync/rcu.rs` |
| Completion | `kernel::sync::Completion` | Exists — `rust/kernel/sync/completion.rs` |
| Condition variable | `kernel::sync::CondVar` | Exists — `rust/kernel/sync/condvar.rs` |
| One-shot init | `kernel::sync::SetOnce<T>` | Exists — `rust/kernel/sync/set_once.rs` |
| Process-context task handle | `kernel::task::Task` | Exists — `rust/kernel/task.rs` (handle type; spawning via existing C-side `kthread_create`+wrapper) |
| Long-running kernel thread (kthread) spawn API | `kernel::task::Kthread` (proposed name) | **Rookery to author** — current upstream wraps the C `task_struct` handle but does not provide a safe Rust kthread-spawning API. New Tier 3 doc + harness. |
| Deferred work in process context | `kernel::workqueue::Work<T>` / `DelayedWork<T>` plus `Queue` | Exists — `rust/kernel/workqueue.rs` (incl. `system_bh()` for softirq-context queues) |
| Deferred work in softirq context | `kernel::workqueue::Queue::system_bh()` (BH-class workqueue) | Exists upstream as the kbuild-bh extension to workqueues. **Note**: classic Linux `tasklet_struct` is being phased out upstream in favour of BH workqueues; Rookery follows this direction. A separate `Tasklet` abstraction is NOT planned. |
| Per-CPU data | `kernel::cpu::PerCpu<T>` (proposed name) | **Rookery to author** — `rust/kernel/cpu.rs` exposes CPU-id helpers but no safe per-CPU container. New Tier 3 doc + harness + TLA+ model (because per-CPU access interacts with preemption). |
| Mutable allocation containers | `kernel::alloc::KBox<T>`, `kernel::alloc::KVec<T>` | Exists — `rust/kernel/alloc/{kbox.rs,kvec.rs}` |
| Owning C-style string | `kernel::str::KString` (proposed name) | **Rookery to author** if needed — upstream uses `&'static CStr` and `CString` (capital S, in `rust/kernel/str.rs`). Most Rookery code should prefer `CString` over a new `KString`; the row is listed in case a heap-allocated owned `str` becomes necessary. |

Locking discipline:
- A `Lock<T, B>` is held only across short critical sections; sleeping while holding a SpinLock is a kernel BUG.
- Lock ordering: documented per-subsystem; lockdep-equivalent compile-time checking is a verification-layer-2 (TLA+) responsibility.
- Lock-free data structures use the `Atomic*` family with explicit ordering (`Ordering::Acquire/Release/SeqCst`); no `Ordering::Relaxed` without a `// SAFETY:` and `// PROOF:` justification.
- Per-CPU access: only via `PerCpu<T>::with_locked` or equivalent; raw `this_cpu_*` operations are forbidden outside `arch/`.

State-machine pattern (replacement for `async`):
- Each long-running operation defines an explicit state enum (`enum FooState { Init, Reading, Writing(usize), Done, Failed(Error) }`).
- The state is owned by the data structure, not by a stack frame.
- Transitions are explicit method calls (`fn step(&mut self) -> StepResult`).
- Wakeups go through the workqueue / tasklet / completion API.

### Unsafe discipline

`unsafe` is allowed; the project requires that every `unsafe` block carry both a SAFETY justification AND a machine-checkable PROOF. Example:

```rust
// SAFETY: `ptr` is non-null and 16-byte-aligned by Caller.preconditions[1];
// `ptr` points to at least `len * size_of::<T>()` bytes of valid memory by Caller.preconditions[2];
// The pointed-to memory is not aliased by any concurrent &mut by the lock guard at line 142.
// PROOF: kani::proofs::mm::page_alloc::buddy_split_safety
unsafe { core::ptr::write(ptr.add(i), value) }
```

The clippy lint `unsafe_without_proof` (Rookery-specific, authored as part of REQ-11) rejects:
- `unsafe { ... }` blocks lacking either a `// SAFETY:` or `// PROOF:` comment in the preceding 8 lines
- `unsafe fn` definitions lacking a `# Safety` rustdoc section AND a `# Proof` rustdoc section naming the harness
- `unsafe impl Trait for Type` lacking the same documentation

`unsafe` is permitted in:
- `bindings/` — auto-generated; SAFETY annotations on the safe wrappers, not on bindings themselves
- Architecture / hardware code (`arch/x86/`) — register manipulation, MSR/MMIO access
- Allocator implementations (`mm/`)
- Locking primitive backends (`kernel::sync::lock::*`)
- Per-CPU and atomic primitive implementations

`unsafe` is forbidden in:
- Driver business logic (drivers must use safe abstractions)
- Filesystem business logic
- Networking protocol parsers (use safe slicing and `byteorder`-style conversions)
- Anything in `lib/` outside the explicit unsafe-primitives modules

### Verification stack

Per `00-overview.md` D4, formal verification is the baseline. Four layers:

#### Layer 1 — Kani SAFETY proofs (mandatory, all unsafe blocks)

[Kani](https://model-checking.github.io/kani/) is a bit-precise model checker for Rust. Every `unsafe` block names a Kani harness:

```rust
// In src/foo.rs
pub fn write_at(buf: &mut [u8; 4096], offset: usize, value: u8) {
    if offset >= 4096 { return; }
    // SAFETY: bounds check above guarantees offset < 4096; buf is uniquely &mut.
    // PROOF: kani::proofs::lib::write_at_safety
    unsafe { core::ptr::write(buf.as_mut_ptr().add(offset), value) }
}
```

```rust
// In proofs/lib_proofs.rs (or wherever the crate organizes harnesses)
#[cfg(kani)]
mod write_at_safety {
    use super::*;

    #[kani::proof]
    fn check_write_at_safety() {
        let mut buf = [0u8; 4096];
        let offset: usize = kani::any();
        let value: u8 = kani::any();
        // Kani explores offset symbolically; the unsafe block must not trip a panic
        // (out-of-bounds, data race, UB) under any value of offset that the function lets through.
        write_at(&mut buf, offset, value);
    }
}
```

Conventions:
- Harness names match `kani::proofs::<crate-path>::<function>_safety` (or `_invariant` for data-structure invariants).
- Harnesses live in the crate's `proofs/` subdirectory, gated by `#[cfg(kani)]`.
- `make verify` runs `cargo kani --workspace`.
- A subsystem-level CI gate fails if any harness times out, fails, or is unreachable.

Kani limitations:
- Bounded model checking: complexity grows with loop bound and input size. Large data structures need bounded harnesses; we accept this and document the bounds in the harness comment.
- No support for global allocator or panicking macros within proof scope — write proofs over pure-data fragments.
- Concurrency reasoning is weak; that's Layer 2's job.

#### Layer 2 — TLA+ concurrency models (mandatory for concurrency primitives)

[TLA+](https://lamport.azurewebsites.net/tla/tla.html) (with the TLC model checker) is the project standard for concurrency reasoning. Every component that:
- Implements a concurrency primitive (Spinlock, Mutex, Refcount, Rcu, Seqlock, lock-free queue, etc.)
- Implements a scheduler invariant (runqueue order, preemption rules, priority inheritance)
- Coordinates more than one CPU through atomic ops without a higher-level lock

…ships a `.tla` model in its `models/` subdirectory.

Each model proves:
- **Safety**: no two CPUs ever observe a state forbidden by the primitive's contract (e.g. for Mutex: no two threads hold the lock simultaneously; for Refcount: no count drops below zero; for RCU: a reader never sees torn writes).
- **Liveness** (where meaningful): under fair scheduling, the operation terminates (e.g. for Mutex: every bounded-wait eventually obtains the lock; for RCU: every read-side critical section eventually completes).

Conventions:
- Filename matches the Rust module (`lock/mutex.tla` ↔ `lock/mutex.rs`).
- The TLA+ model declares the abstract state in upper-case (CONSTANTS, VARIABLES) and an abstract operation per Rust API call.
- A `MODELED:` comment block at the top of the Rust module names the `.tla` file and lists the safety/liveness theorems proven there.
- `make tla` runs TLC on every `.tla` model.
- A linker-equivalent step (planned, not yet authored) verifies that every Rust concurrency-primitive module names a corresponding `.tla` file that exists.

TLA+ limitations:
- Models are *abstract* — they prove invariants of an idealized algorithm, not bit-precise Rust code. The translation gap is part of human review.
- TLC explores a finite state space; some properties require Apalache (TLA+ symbolic model checker). We allow either.

#### Layer 3 — Kani harnesses for data-structure invariants (mandatory for the four named subsystems)

Per REQ-13, the page allocator (`mm/page_alloc`), slab caches (`mm/slab`), VFS dcache (`fs/dcache`), and scheduler runqueues (`kernel/sched`) ship Kani-checkable invariant harnesses:

| Subsystem | Sample invariant |
|---|---|
| Page allocator | "No physical page address appears in two free lists simultaneously" |
| Slab | "Every cached object's freelist next-pointer points within the same slab page" |
| dcache | "Every dentry's parent is reachable from the root, and the parent's child list contains the dentry" |
| Scheduler runqueue | "The CFS rbtree's leftmost node has the smallest vruntime among RUNNABLE tasks on this CPU" |

These are bounded checks (small N) but they exercise the actual Rust implementation, not an abstract model.

#### Layer 4 — Functional correctness via Creusot / Verus / Prusti (per-subsystem opt-in)

For modules where bit-precise functional correctness pays for itself:

- **Creusot** ([github.com/creusot-rs/creusot](https://github.com/creusot-rs/creusot)) — translates annotated Rust to Why3, dispatches to Z3 / Alt-Ergo / CVC4. Best for pure-data algorithms.
- **Verus** ([github.com/verus-lang/verus](https://github.com/verus-lang/verus)) — separation logic for Rust; more powerful but requires a Rust dialect.
- **Prusti** ([github.com/viperproject/prusti-dev](https://github.com/viperproject/prusti-dev)) — verifies safety + user-supplied contracts via Viper.

Subsystem opt-in candidates (declared in the subsystem doc, not enforced project-wide):
- Cryptographic primitives (constant-time properties via Creusot or hand-written Coq linkage)
- ELF loader (Creusot for parser correctness)
- Network protocol state machines (Verus for state-machine correctness against TLA+ model)

Annotations live in `contracts/` subdirectory; tooling integrates into `make verify-strict`.

### kbuild integration

Rookery crates compile via the same kbuild rules as rust-for-linux:
- Every Rust source dir has a `Kbuild` (or `Makefile`) with `obj-$(CONFIG_<feature>) += foo.o`.
- Cross-crate dependencies declared via `rustc_target_spec_json` and `RUSTFLAGS_<file>.o`.
- `Cargo.toml` is permitted at the workspace root for `cargo kani` / `cargo creusot` / IDE integration but is NOT the kernel build's source of truth — kbuild is.
- `make rookery-vmlinux` is a phony target that produces the bootable image; defined in the project's top-level Makefile (lives at the kernel-source-tree root).

### Lints, formatting, documentation

| Tool | Invocation | CI gate? |
|---|---|---|
| rustfmt | `make rustfmt` (auto-format), `make rustfmtcheck` (verify) | Yes (rustfmtcheck) |
| clippy (upstream lints) | `make CLIPPY=1` | Yes |
| clippy (Rookery lints: `unsafe_without_proof` etc.) | `make CLIPPY=1 ROOKERY_LINTS=1` | Yes |
| rustdoc | `make rustdoc` | Yes (broken intra-doc links fail) |
| Kani | `make verify` | Yes |
| TLA+ TLC | `make tla` | Yes |
| Creusot/Verus/Prusti | `make verify-strict` | Yes (only for subsystems opted in) |

### FFI and bindings

- Auto-generated bindings live in `rust/bindings/`. Bindgen is invoked from kbuild.
- Hand-written `extern "C"` declarations are confined to `ffi.rs` modules; one such module per crate that exports symbols to C.
- Rust functions exported to C are `#[no_mangle] pub extern "C" fn`; they must have `// SAFETY:` documentation describing the invariants C callers must uphold AND `// PROOF:` referencing a Kani harness over the safety boundary.
- Layout-compatible structs use `#[repr(C)]`; layout-randomized internal-only structs use the default `#[repr(Rust)]` plus an explicit `#[repr(...)]` if a future `RANDSTRUCT`-equivalent is desired (per `00-security-principles.md`).

### Forbidden patterns

- `async fn`, `async {}`, `.await`, `Future`, any executor — `clippy: async_in_kernel`
- Bare `panic!`, `unwrap`, `expect`, `unreachable!`, `todo!`, `unimplemented!` in production paths — `clippy: panic_in_kernel_path`
- `Box::new`, `Vec::new`, `Vec::push` (use `KBox` / `KVec`) — `clippy: alloc_without_gfp`
- Bare arithmetic on `usize` / `u64` / `u32` size types — `clippy: bare_arithmetic_on_size`
- `unsafe { ... }` lacking `// SAFETY:` AND `// PROOF:` — `clippy: unsafe_without_proof`
- New `#![feature(...)]` not in upstream's allowed set — `clippy: unstable_feature_without_decision`
- Lifetime parameters on functions reachable from syscall entry that aren't tied to an owning type — manual review
- Tabs in Rust source (rust-for-linux uses 4 spaces; rustfmt enforces)

### Naming and module organization

- Files: `snake_case.rs`. Module roots in subsystem directories use `mod.rs` or the subsystem name.
- Public types: `PascalCase`. Trait names: `PascalCase` matching upstream patterns (`Lock`, `LockBackend`, `Backend`).
- Functions / methods / variables: `snake_case`.
- Constants and statics: `SCREAMING_SNAKE_CASE`.
- Module names mirror the upstream Linux directory or file:
  - `kernel::sched::cfs` ↔ `kernel/sched/fair.c`
  - `kernel::mm::page_alloc` ↔ `mm/page_alloc.c`
  - `kernel::fs::vfs::dcache` ↔ `fs/dcache.c`

When the upstream filename is opaque (e.g. `fair.c` for the CFS scheduler), the Rust module uses the conceptual name (`cfs`) and the upstream-paths frontmatter records the actual filename.

### Workspace layout (informative; binding decision is in `00-overview.md` "Rust crate organisation" section, delegated to implementing instance)

The implementing instance decides crate boundaries within these guardrails:
- Crates align with upstream subsystem directories where possible (`mm/` → `kernel-mm` crate, etc.).
- A crate's `proofs/`, `models/`, `contracts/` are sibling to `src/`.
- Workspace-level `Cargo.toml` lives at the kernel-source-tree root alongside `rust/Cargo.toml`.

## Out of Scope

- **Specific tooling versions for Kani / TLA+ / Creusot / Verus / Prusti** — track upstream tooling. Pin in a separate `00-toolchain.md` only if version drift becomes a problem.
- **Workspace shape** — implementing instance's call.
- **Per-subsystem Verification section content** — those are subsystem doc concerns; this doc only mandates the section exists.
- **Documentation language** — English only, matching upstream Linux convention.
- **Rust 2024 edition** — deferred until rust-for-linux moves; currently 2021.

## Open Questions

<!-- OPEN: Q1 -->
### Q1: Project-specific clippy lint authoring schedule
The lints `unsafe_without_proof`, `panic_in_kernel_path`, `async_in_kernel`, `bare_arithmetic_on_size`, `alloc_without_gfp`, `unstable_feature_without_decision` are referenced as enforcement mechanisms but don't exist yet. They are independent crate-of-clippy plugins, each requiring AST traversal logic.

**Recommendation**: Track each lint as its own crosslink subissue under issue #1. Author the lints as one of the first implementation tasks (before any subsystem code) so the rest of the codebase grows under enforcement.

**To resolve**: User confirms the priority — block any non-foundation implementation work on these lints landing first.
<!-- /OPEN -->

<!-- OPEN: Q2 -->
### Q2: TLA+ tooling — TLC vs Apalache
Two TLA+ model checkers exist: TLC (the original) and Apalache (symbolic, supports unbounded properties via SMT). Many concurrency proofs are tractable in TLC; some (large state spaces, parametric proofs) need Apalache.

**Recommendation**: TLC is the default. A model may declare `(* APALACHE *)` in its preamble to opt into Apalache instead. CI runs both checkers as appropriate.

**To resolve**: User confirms the default.
<!-- /OPEN -->
