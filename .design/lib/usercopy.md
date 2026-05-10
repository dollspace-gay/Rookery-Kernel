# Tier-3: lib/usercopy — userspace-copy primitives + iov_iter

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - lib/usercopy.c
  - lib/strncpy_from_user.c
  - lib/strnlen_user.c
  - lib/iov_iter.c
  - lib/fault-inject-usercopy.c
  - arch/x86/lib/usercopy.c
  - arch/x86/lib/usercopy_64.c
  - arch/x86/lib/copy_user_64.S
  - arch/x86/lib/copy_user_uncached_64.S
  - include/linux/uaccess.h
  - include/linux/iov_iter.h
  - include/asm-generic/uaccess.h
-->

## Summary
Tier-3 design for the kernel↔userspace data-transfer primitives. Owns `copy_to_user`, `copy_from_user`, `clear_user`, `strncpy_from_user`, `strnlen_user`, `get_user` / `put_user`, `iov_iter_*` (the cross-cutting I/O-vector iterator), the userspace-pointer fault-injection harness, and the architecture-side fast-paths in `arch/x86/lib/usercopy*`.

**This Tier-3 owns the implementation of UDEREF, USERCOPY, and PAX_USERCOPY-equivalent hardening per `00-security-principles.md`'s Mandatory category.** Every kernel↔user transfer routes through here; getting it wrong creates the most prolific class of kernel CVE (info-leak + arbitrary-write).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Cross-arch usercopy core | `lib/usercopy.c` |
| User-string copy | `lib/strncpy_from_user.c`, `lib/strnlen_user.c` |
| iov_iter (I/O-vector iteration) | `lib/iov_iter.c`, `include/linux/iov_iter.h` |
| Fault injection for usercopy testing | `lib/fault-inject-usercopy.c` |
| x86-specific assembly fast-paths | `arch/x86/lib/usercopy.c`, `arch/x86/lib/usercopy_64.c`, `arch/x86/lib/copy_user_64.S`, `arch/x86/lib/copy_user_uncached_64.S` |
| Public uaccess API | `include/linux/uaccess.h`, `include/asm-generic/uaccess.h` |

## Compatibility contract

### copy_to/from_user ABI

```rust
fn copy_to_user<T>(dst: UserPtr<T>, src: &T) -> Result<(), Error>;
fn copy_from_user<T>(dst: &mut T, src: UserPtr<T>) -> Result<(), Error>;
```

(Underlying upstream C signature: `unsigned long copy_to_user(void __user *to, const void *from, unsigned long n)` — returns 0 on success, OR the number of bytes that could not be copied on partial failure.)

The Rust wrapper returns `Err(EFAULT)` if any byte couldn't be copied; the partial-byte-count is exposed via a separate `copy_to_user_partial` family for callers that need it.

### get_user / put_user

Single-word transfer wrappers; preserved per upstream's user-handle macro family.

### Userspace-pointer typing (UDEREF mandate)

Per `00-security-principles.md` Mandatory category, every userspace pointer is wrapped in `UserPtr<T>`:

```rust
#[repr(transparent)]
pub struct UserPtr<T> {
    addr: usize,
    _phantom: PhantomData<T>,
}

impl<T> UserPtr<T> {
    /// SAFELY ENFORCED: only copy_to/from_user can dereference.
    /// Compile error to call `as_ref` / `as_mut` / raw `*` on UserPtr.
}
```

Direct dereference of a `UserPtr<T>` is a compile-time error. The only paths that can read/write a `UserPtr<T>` are:
- `copy_to/from_user`
- `iov_iter_copy_*` family
- `strncpy_from_user`, `strnlen_user`
- `get_user`, `put_user`

This compile-time guarantee is the row-1 UDEREF feature per `00-security-principles.md`.

### USERCOPY whitelist (PAX_USERCOPY equivalent)

Per `00-security-principles.md` Mandatory category, slab caches that participate in `copy_*_user` MUST be flagged at creation time:

```rust
let cache = KmemCache::<MyType>::new_user_copyable(name)?;
```

Where `MyType: UserCopyOk` is a marker trait that the implementing crate derives only on POD types. Attempting `copy_to_user(buf, my_struct)` when `MyType: !UserCopyOk` is a compile error.

This compile-time guarantee is the USERCOPY feature per `00-security-principles.md`.

### iov_iter (I/O-vector iterator)

`include/linux/iov_iter.h` defines a polymorphic iterator type that abstracts: kernel-buffer, userspace-buffer, kvec-list, bio_vec-list, xarray-folio, pipe-buffer, ITER_DISCARD. Used by every fs/net read/write path. Compat: every `iov_iter_*` API function preserved with byte-identical behavior.

## Requirements

- REQ-1: `copy_to_user`, `copy_from_user`, `clear_user`, `strncpy_from_user`, `strnlen_user`, `get_user`, `put_user`, `__copy_to_user_inatomic`, `__copy_from_user_inatomic`, `nocache_copy_*`, `copy_in_user` semantics byte-identical to upstream — same return-value convention (0 on success, partial-byte-count on failure for the size variants).
- REQ-2: `UserPtr<T>` newtype enforces UDEREF at compile time; raw dereference of a UserPtr is a compile error. (Mandatory per `00-security-principles.md`.)
- REQ-3: `UserCopyOk` marker trait; `copy_to/from_user` on `T` requires `T: UserCopyOk`. (Mandatory per `00-security-principles.md`.)
- REQ-4: x86-specific assembly fast-paths preserved: `arch/x86/lib/copy_user_64.S` (the canonical fast path), `arch/x86/lib/copy_user_uncached_64.S` (cache-bypass for very large transfers). Performance within ±5% of upstream.
- REQ-5: `iov_iter_*` API (full set — `iov_iter_init`, `_iter_advance`, `_iter_revert`, `iov_iter_count`, `iov_iter_get_pages2`, `copy_page_from_iter_atomic`, `copy_page_to_iter`, `import_iovec`, etc.) byte-identical with upstream's behavior.
- REQ-6: Fault injection (`fault-inject-usercopy.c`) preserved: `/sys/kernel/debug/fail_usercopy/*` knobs honored.
- REQ-7: SMAP integration: `stac()` / `clac()` (cross-ref `arch/x86/cpu-mitigations.md`'s SMAP CR4 bit) bracket every userspace access; `pagefault_disable` regions block sleeping.
- REQ-8: Page-fault-during-usercopy semantics: faults during copy → return partial byte count + the kernel handles the fault (faults aren't propagated to caller as kernel oops).
- REQ-9: `nocache_copy_*` (write-combining stores for /dev/{mem,zero,kmsg}-style mass copies) preserves upstream semantics.
- REQ-10: Hardening section per `00-security-principles.md` template.
- REQ-11: A grep over Rookery code for `unsafe { *user_ptr.addr ... }` returns zero matches — compile-time enforcement of UDEREF.
- REQ-12: A grep over Rookery code for `copy_to_user(...)` with a non-UserCopyOk type fails to compile.

## Acceptance Criteria

- [ ] AC-1: A test harness exercises every documented usercopy primitive against a curated input set (zero-byte, one-byte, page-aligned, page-crossing, fault-injection); behavior matches upstream byte-for-byte. (covers REQ-1)
- [ ] AC-2: A grep over Rookery for `unsafe.*user_ptr` outside the lib/usercopy + arch/x86/lib/usercopy modules returns zero matches; the dereferencing primitives are confined. (covers REQ-2, REQ-11)
- [ ] AC-3: A type-tag mismatch test (a struct that lacks `UserCopyOk` derive) fails to compile when passed to `copy_to_user`. (covers REQ-3, REQ-12)
- [ ] AC-4: A microbenchmark (`tools/perf/lib/copy-bench`) reports copy_to/from_user throughput within ±5% of upstream on identical hardware. (covers REQ-4)
- [ ] AC-5: An iov_iter test covering all 7 iter types (UBUF, IOVEC, KVEC, BVEC, FOLIO_QUEUE, XARRAY, DISCARD) round-trips data correctly. (covers REQ-5)
- [ ] AC-6: Fault-injection test: `/sys/kernel/debug/fail_usercopy/probability` set to 100% causes EFAULT on every copy_to_user; pt_regs state remains coherent post-fault. (covers REQ-6, REQ-8)
- [ ] AC-7: SMAP enforcement test: a kernel-side `*p = 0;` where `p` is a raw user-VA (not via `copy_*_user`) triggers #PF + handler-emitted "kernel/userspace SMAP violation" (verifiable via dmesg). (covers REQ-7)
- [ ] AC-8: A `copy_*_user` page-crossing test where the second page faults: the call returns the partial byte count corresponding to the first page; no crash. (covers REQ-8)
- [ ] AC-9: A nocache_copy test against /dev/zero shows write-combining stores via PMU counter. (covers REQ-9)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::user::UserPtr<T>` — Pointer to userspace memory (newtype around `usize`)
- `kernel::user::UserCopyOk` — Marker trait for types safe to copy across the kernel-user boundary
- `kernel::user::copy_to_user`, `copy_from_user` — typed copies
- `kernel::user::clear_user` — zero out user range
- `kernel::user::strncpy_from_user`, `strnlen_user` — bounded user-string ops
- `kernel::user::get_user`, `put_user` — single-word fast path
- `kernel::user::iov_iter::IovIter` — polymorphic I/O-vector iterator (cross-ref `lib/00-overview.md` for top-level positioning)
- `kernel::user::fault_inject` — debug fault injection
- `kernel::arch::x86::user::copy_*` — x86 assembly fast-paths (called from cross-arch wrappers)

### Locking and concurrency

Usercopy operations cross the kernel-user boundary. Concurrency considerations:
- **Page-fault-during-usercopy**: enabled by default; the kernel side releases any held kernel locks if it would sleep (most usercopy callers must NOT hold spinlocks across copy_to_user, per upstream documentation).
- **`__copy_*_inatomic`** variant: no fault-handling; for callers that hold spinlocks. If a fault would occur, returns immediately with non-zero remaining-bytes.
- **SMAP STAC/CLAC**: per-CPU; bracketing every userspace access in the assembly fast-path. Not a lock — a CPU mode switch.

### Error handling

- `Err(EFAULT)` — userspace pointer is not mapped, or write to read-only mapping
- `Err(EINVAL)` — bad iov_iter state
- Partial copies expose a `Result<(), PartialCopyError>` where `PartialCopyError` carries the byte count not copied; standard wrappers convert to `Err(EFAULT)` for callers that don't care about partial counts.

## Verification

### Layer 1: Kani SAFETY proofs (mandatory all unsafe blocks)

| Surface | Harness |
|---|---|
| `__copy_user` raw asm wrapper (the asm primitive's safety) | `kani::proofs::lib::usercopy::raw_copy_safety` |
| iov_iter advance / revert (cursor arithmetic) | `kani::proofs::lib::iov_iter::cursor_safety` |
| `strncpy_from_user` bounded loop | `kani::proofs::lib::usercopy::strncpy_safety` |
| Fault injection trampoline | `kani::proofs::lib::usercopy::fault_inject_safety` |

### Layer 2: TLA+ models

(none mandatory — usercopy is per-CPU; no novel concurrency)

### Layer 3: Kani harnesses for data-structure invariants

- iov_iter cursor invariants (mandatory per `lib/00-overview.md`) — `count + remaining == initial`; segment cursor never beyond segment length: `kani::proofs::lib::iov_iter::cursor_invariants`

### Layer 4: Functional correctness (opt-in)

- **iov_iter** copy-equivalence proof via Creusot — proves: `copy_from_iter(&iter, buf)` produces the same bytes as a serial walk over `iter`'s segments. Tractable; high-leverage (every fs/net read/write depends).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **UDEREF** (compile-time enforcement) | `UserPtr<T>` newtype rejects raw deref at compile time; only `copy_*_user` family can read/write a `UserPtr<T>` | § Mandatory |
| **USERCOPY** (compile-time slab-cache type tagging) | `UserCopyOk` marker trait; `KmemCache::<T>::new_user_copyable` API; `copy_to/from_user(...)` on a non-UserCopyOk T is a compile error | § Mandatory |
| **PAX_USERCOPY-equivalent** (this is the row-1 form of the same) | Same — implemented via type-system, not runtime check | § Mandatory |

### Row-1 features consumed by this component

- **CLOSE_KERNEL/CLOSE_USERLAND**: SMAP enforced by hardware via stac()/clac() in the assembly fast-path (consumed from `arch/x86/cpu-mitigations.md`)
- **SIZE_OVERFLOW**: byte-count + offset arithmetic in iov_iter uses checked operators
- **AUTOSLAB**: kvec / iovec arrays allocated via per-type slab caches when needed
- **MEMORY_SANITIZE**: bouncebuffer-backed iov_iter routes zero on free

### Row-2 / GR-RBAC integration

This component runs below the LSM-hook layer. Its callers (syscall handlers in `mm/mmap.md`, `fs/exec-binfmt.md`, etc.) invoke LSM hooks before/after their copy_*_user operations. The usercopy primitives themselves are infrastructure.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — usercopy semantics are exhaustively specified by Linux's user-handle convention)

## Out of Scope

- Bouncebuffers / DMA-bounce (cross-ref `kernel/00-overview.md` § dma-mapping.md)
- 32-bit-only usercopy paths (mostly auto-handled by `arch/x86/lib/usercopy_32.S` not present in `X86_64=y` builds)
- Implementation code
