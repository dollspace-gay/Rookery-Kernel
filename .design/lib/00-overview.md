# Subsystem: lib/ — kernel utility library

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - lib/
  - lib/crypto/
  - lib/crc/
  - lib/math/
  - lib/vdso/
  - lib/kunit/
  - lib/lz4/
  - lib/lzo/
  - lib/zstd/
  - lib/zlib_deflate/
  - lib/zlib_inflate/
  - lib/zlib_dfltcc/
  - lib/xz/
  - lib/842/
  - lib/raid/
  - lib/raid6/
  - lib/reed_solomon/
  - lib/pldmfw/
  - lib/dim/
  - lib/fonts/
  - lib/tests/
-->

## Summary
Tier-2 overview for `lib/` — the kernel's general-purpose utility library. Holds generic data structures (rbtree, list, maple_tree, xarray, radix-tree, kfifo, …), string and number formatting (vsprintf, kstrtox), compression codecs (lz4, lzo, zstd, zlib, xz, 842), checksums and lightweight hashes (CRC family, siphash, xxhash), arithmetic helpers (`lib/math/`), userspace-copy primitives (`usercopy`, `iov_iter`), debugging infrastructure (debugobjects, fault-injection, stackdepot, dynamic_debug), the KUnit test framework, and cross-architecture vDSO data + time logic.

`lib/` is mostly pure utility code: thin wrappers around well-defined data structures and algorithms. A substantial fraction already has rust-for-linux abstractions (`rust/kernel/list.rs`, `rust/kernel/rbtree.rs`, `rust/kernel/str.rs`, `rust/kernel/iov.rs`, `rust/kernel/io.rs`); Rookery extends these rather than replacing them. The remaining surface is the bulk of new Rust authoring in this subsystem.

## Upstream references in scope

`lib/` (267 files at the top level + 22 subdirectories at baseline). Categorized:

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| Generic data structures | `lib/rbtree.c`, `lib/list_sort.c`, `lib/list_debug.c`, `lib/maple_tree.c`, `lib/xarray.c`, `lib/radix-tree.c`, `lib/generic-radix-tree.c`, `lib/btree.c`, `lib/interval_tree.c`, `lib/kfifo.c`, `lib/sort.c`, `lib/min_heap.c`, `lib/bsearch.c`, `lib/assoc_array.c`, `lib/bitmap.c`, `lib/bitmap-str.c`, `lib/find_bit.c`, `lib/hexdump.c`, `lib/stackdepot.c`, `include/linux/{rbtree,list,maple_tree,xarray,radix-tree,btree,interval_tree,kfifo,sort,min_heap,hashtable,bitmap}.h` | `data-structures.md` |
| String + number formatting | `lib/string.c`, `lib/string_helpers.c`, `lib/vsprintf.c`, `lib/kstrtox.c`, `lib/parser.c`, `lib/argv_split.c`, `lib/cmdline.c`, `lib/ucs2_string.c`, `lib/seq_buf.c`, `include/linux/{string,kstrtox,parser,seq_buf}.h` | `strings-fmt.md` |
| Compression codecs | `lib/lz4/`, `lib/lzo/`, `lib/zstd/`, `lib/zlib_deflate/`, `lib/zlib_inflate/`, `lib/zlib_dfltcc/`, `lib/xz/`, `lib/842/`, `include/linux/{lz4,lzo,zstd,zlib,xz}.h` | `compression.md` |
| Checksums + lightweight hashes | `lib/crc/`, `lib/checksum.c`, `lib/siphash.c`, `lib/xxhash.c`, `include/linux/{crc8,crc16,crc32,crc64,crc32c,siphash,xxhash}.h` | `checksums-hashes.md` |
| Lightweight crypto primitives | `lib/crypto/` (chacha20, poly1305, blake2s, libsha256, etc.) | `lightweight-crypto.md` |
| Arithmetic helpers | `lib/math/` (gcd, lcm, int_log, int_pow, int_sqrt, div64, cordic, polynomial, rational, reciprocal_div, prime_numbers) | `math.md` |
| Userspace copy primitives | `lib/usercopy.c`, `lib/strncpy_from_user.c`, `lib/strnlen_user.c`, `lib/iov_iter.c`, `lib/fault-inject-usercopy.c`, `include/linux/uaccess.h`, `include/linux/iov_iter.h` | `usercopy.md` |
| Debug tooling | `lib/debug_locks.c`, `lib/debugobjects.c`, `lib/fault-inject.c`, `lib/dynamic_debug.c` (and the runtime debug infra), `lib/bug.c`, `lib/bust_spinlocks.c`, `lib/dump_stack.c`, `lib/error-inject.c`, `include/linux/{debug_locks,debugobjects,fault-inject,dynamic_debug}.h` | `debug-tooling.md` |
| KUnit framework | `lib/kunit/`, `include/kunit/`, `Documentation/dev-tools/kunit/` | `kunit.md` |
| vDSO core (cross-arch) | `lib/vdso/`, `include/vdso/` | `vdso-core.md` (cross-references `arch/x86/vdso.md`) |
| Specialized hardware-class helpers | `lib/raid/`, `lib/raid6/`, `lib/reed_solomon/`, `lib/dim/` (Dynamic Interrupt Moderation), `lib/pldmfw/` (PLDM firmware), `lib/fonts/` | `specialized-helpers.md` |
| Test fixtures (NOT design surface) | `lib/tests/`, `lib/test_*.c`, `lib/test_fortify/` | (not separately documented; tests track the components they cover) |

The `lib/Kconfig` and `lib/Makefile` describe how each component is selectively compiled.

## Compatibility contract

`lib/` is mostly internal kernel code, but has narrow userspace-visible surfaces:

| Interface | Owner doc | Compat level |
|---|---|---|
| `vsprintf`'s `%p` extension format specifiers (`%pV`, `%pS`, `%pf`, `%pe`, `%pd`, `%pD`, `%ps`, `%pSR`, `%phN`, `%pa`, `%pap`, `%pap`, `%pK`, `%pks`, `%puk`, `%pi4`, `%pi6`, `%pI4`, `%pI6`, `%pI6c`, `%pM`, `%pMR`, `%pmR`, `%pUb`, `%pUl`, `%pUL`, `%pUB`, `%pt`, `%ptR`, `%ptT`, `%ptd`, `%pte`, `%pVe`, `%pX`, `%pNF`, `%pNh`, `%pNn`, `%pNI`) | `strings-fmt.md` | Identical formatting (visible in `dmesg`, ftrace, `/proc/*` output) |
| `/sys/kernel/debug/dynamic_debug/control` ABI | `debug-tooling.md` | Identical query/set syntax |
| KUnit test result format (`/sys/kernel/test/...` and TAP output) | `kunit.md` | Identical (downstream tooling parses TAP) |
| `/proc/<pid>/stackdepot` (where applicable) and `/sys/kernel/debug/.../stack_*_users` | `debug-tooling.md` | Identical content format |
| vDSO data layout (the `vdso_data` struct read by userspace vDSO functions) | `vdso-core.md` + arch-side `arch/x86/vdso.md` | Layout-identical (or callers may break) |
| Compression algorithm output bytes (when surfaced via syscalls like `BTRFS_IOC_COMPR_*`, `f2fs_compress_*`, `zram` block layer) | `compression.md` | Bit-identical compressed output for a given input + algorithm + level (so existing data on disk decompresses correctly under a new kernel — actually a stronger guarantee than upstream gives, but easy to honor by matching algorithm-spec implementations) |
| CRC algorithm outputs visible in headers (network checksums, btrfs metadata, ext4 checksums) | `checksums-hashes.md` | Bit-identical for a given input + algorithm |

The `vsprintf %p` extensions list above is intentionally exhaustive — these formats appear in `dmesg`, `ftrace`, and `/proc` content that downstream tools parse with regex. Adding or removing one is a userspace-visible compat break.

## Requirements

- REQ-1: Every existing rust-for-linux abstraction in upstream `rust/kernel/` for a `lib/` concept is the canonical wrapper. Rookery extends these — Rookery does NOT introduce a parallel `kernel::lib::list::List<T>` when `kernel::list::List<T>` already exists upstream.
- REQ-2: Cross-references are correct: `vsprintf` `%pV`/`%pe`/`%ps`/etc. format specifier handlers MUST produce byte-identical output to upstream's `vsprintf` for the same input. This is mechanically testable.
- REQ-3: Compression codec outputs are bit-identical to upstream's for the same input + algorithm + compression level. (Codecs are spec-defined; "byte-identical" means following the spec exactly.)
- REQ-4: CRC and checksum outputs are bit-identical. (Same algorithm specs apply.)
- REQ-5: KUnit test framework's TAP output is byte-identical (modulo test names / file paths) so downstream test runners that grep TAP continue to work.
- REQ-6: `dynamic_debug` query/set syntax is identical; debug pr_debug() print sites use the same opt-in/opt-out commands at `/sys/kernel/debug/dynamic_debug/control`.
- REQ-7: Userspace copy primitives (`copy_to_user`, `copy_from_user`, `strncpy_from_user`, `strnlen_user`, `iov_iter_*`) preserve the exact fault-handling, exception-recovery behavior of upstream — including the `__must_check` discipline.
- REQ-8: `vdso_data` struct (read by userspace vDSO symbol implementations) is layout-identical, and is updated atomically using the same seqlock protocol upstream uses.
- REQ-9: All Tier-3 docs spawned by this overview enumerate their unsafe-block clusters and verification artifacts (Layer 1 mandatory, Layer 2 when concurrency primitives are introduced, Layer 3 mandatory for data-structure invariants).
- REQ-10: A grep for "Source:" references in this overview's Architecture section must resolve under `/home/doll/linux-src/`.

## Acceptance Criteria

- [ ] AC-1: A grep for `pub use kernel::{list,rbtree,str,iov};` over Rookery code finds usages but `pub use kernel::lib::{list,rbtree,str,iov};` finds zero — i.e., Rookery does not re-export under a `lib::` prefix that competes with upstream's namespace. (covers REQ-1)
- [ ] AC-2: A `vsprintf` golden-output test exercises every `%p` extension format with curated inputs and asserts byte-identical output between Rookery and upstream. (covers REQ-2)
- [ ] AC-3: For each compression codec (lz4, lzo, zstd, zlib, xz, 842), compress a curated corpus and compare bytes against upstream's compressed bytes. (covers REQ-3)
- [ ] AC-4: For each CRC variant (crc8, crc16, crc32, crc32c, crc64) and lightweight hash (siphash, xxhash), assert bit-identical output for curated inputs. (covers REQ-4)
- [ ] AC-5: KUnit's TAP output for an empty-passing test, an empty-failing test, and a parameterized test matches upstream's byte-for-byte. (covers REQ-5)
- [ ] AC-6: `echo "+p" > /sys/kernel/debug/dynamic_debug/control` and `cat /sys/kernel/debug/dynamic_debug/control` parsing succeed identically to upstream. (covers REQ-6)
- [ ] AC-7: A fault-injection harness triggering `EFAULT` mid-`copy_from_user` produces the same return value (non-zero remaining bytes) and same `pt_regs` state on Rookery as on upstream. (covers REQ-7)
- [ ] AC-8: A `pmemcmp(rookery_vdso_data, upstream_vdso_data, sizeof(vdso_data))` comparison after equivalent kernel boot states returns zero. (covers REQ-8)
- [ ] AC-9: Each Tier-3 doc has a non-empty Verification section per the Layer-1/2/3/4 schema. (covers REQ-9)
- [ ] AC-10: `make verify` passes all Kani harnesses under `kernel/lib/proofs/`. (covers REQ-9 mechanical)
- [ ] AC-11: A walk over `**Source**:` lines in this overview reports zero MISS against `/home/doll/linux-src/`. (covers REQ-10)

## Architecture

### Layout map (Tier-3 docs spawned from this overview)

```
.design/lib/
  00-overview.md            ← this document
  data-structures.md        ← rbtree, list, maple_tree, xarray, radix-tree, btree, interval_tree, kfifo, sort, min_heap, bsearch, assoc_array, bitmap, find_bit, hexdump, stackdepot, hashtable, generic-radix-tree
  strings-fmt.md            ← string, string_helpers, vsprintf (incl. all %p extensions), kstrtox, parser, argv_split, cmdline, ucs2_string, seq_buf
  compression.md            ← lz4, lzo, zstd, zlib (deflate + inflate + dfltcc), xz, 842
  checksums-hashes.md       ← CRC family (crc8/16/32/64/crc32c), checksum (IP), siphash, xxhash
  lightweight-crypto.md     ← lib/crypto/ primitives (chacha20, poly1305, blake2s, libsha256) — distinct from full crypto/ API
  math.md                   ← gcd, lcm, int_log, int_pow, int_sqrt, div64, cordic, polynomial, rational, reciprocal_div, prime_numbers
  usercopy.md               ← copy_to/from_user backends, strncpy_from_user, strnlen_user, iov_iter, fault-inject-usercopy
  debug-tooling.md          ← debug_locks, debugobjects, fault-inject, dynamic_debug, bug, bust_spinlocks, dump_stack, error-inject
  kunit.md                  ← KUnit test framework + TAP output spec
  vdso-core.md              ← lib/vdso/ data + time logic (cross-arch); cross-references arch/x86/vdso.md
  specialized-helpers.md    ← raid (RAID device-mapper helper), raid6 (PQ), reed_solomon, dim (Dynamic Interrupt Moderation), pldmfw, fonts
```

### Cross-references

- `00-rust-conventions.md` § Concurrency primitives table — `kernel::list::List<T>`, `kernel::rbtree::RBTree<T>`, `kernel::str::CStr` / `CString`, `kernel::iov::IovIter` already exist in upstream rust/kernel/ and are the reference.
- `arch/x86/00-overview.md` § cross-references — `arch/x86/lib/` (the x86-specific portions of memcpy/memset/copy_to_user implementations) cooperates with this `lib/` doc; the *generic* fallbacks live here, the *x86 assembly* lives there.
- `00-glossary.md` — `seqlock`, `RCU`, `atomic_t`, `refcount_t` definitions are referenced from data-structures.md.
- `crypto/00-overview.md` (Phase B, not yet authored) — `lightweight-crypto.md` here covers `lib/crypto/`; the full kernel crypto API is in `crypto/`. Each Tier-3 doc must declare which side of the boundary a primitive sits on.

### Rust module organization (informative)

- `kernel::list` — exists upstream (`rust/kernel/list.rs`)
- `kernel::rbtree` — exists upstream (`rust/kernel/rbtree.rs`)
- `kernel::str` — exists upstream (`rust/kernel/str.rs` + `rust/kernel/str/`)
- `kernel::iov` — exists upstream (`rust/kernel/iov.rs`)
- `kernel::io` — exists upstream (`rust/kernel/io.rs` + `rust/kernel/io/`)
- `kernel::xarray` ← `rust/kernel/xarray.rs` (already partly in upstream — confirm coverage in `data-structures.md`)
- `kernel::maple_tree` — Rookery to author (Tier 3 in `data-structures.md`)
- `kernel::radix_tree` — Rookery to author (Tier 3 in `data-structures.md`); upstream is migrating most users to xarray, so this may be lower priority
- `kernel::bitmap` — Rookery to author (Tier 3 in `data-structures.md`)
- `kernel::sort` — Rookery to author (Tier 3 in `data-structures.md`)
- `kernel::vsprintf` — Rookery to author (Tier 3 in `strings-fmt.md`)
- `kernel::compress::{lz4,lzo,zstd,zlib,xz,bz842}` — Rookery to author (Tier 3 in `compression.md`)
- `kernel::crc` — Rookery to author (Tier 3 in `checksums-hashes.md`)
- `kernel::math::*` — Rookery to author (Tier 3 in `math.md`)
- `kernel::dbg::{debugobjects, fault_inject, dynamic_debug, …}` — Rookery to author (Tier 3 in `debug-tooling.md`)
- `kernel::kunit` — exists partly upstream (`rust/kernel/kunit.rs`); extended in Tier 3 `kunit.md`

### Locking and concurrency

`lib/` is mostly lock-free utility code. Locking surfaces:
- **xarray, maple_tree** — internal locking (xa_lock RCU + spinlock); their TLA+ models live in their Tier-3 docs.
- **stackdepot** — uses RCU for the depot's hash table.
- **dynamic_debug** — control-file writes serialize via mutex.
- **vdso seqlock** — readers (userspace via vDSO) and writers (timekeeping IRQ) coordinate via seqlock; TLA+ model required.

### Error handling

`lib/` returns `Result<T, kernel::error::Error>` per `00-rust-conventions.md` REQ-6. Specific common returns:
- `Err(EINVAL)` — bad input
- `Err(ENOMEM)` — allocation failure
- `Err(EFAULT)` — userspace pointer fault (in usercopy paths)
- `Err(EOVERFLOW)` — arithmetic overflow in `lib/math/` paths
- `Err(EBADMSG)` — bad input to compression/decompression/checksum (corrupt data)

## Verification

### Layer 1: Kani SAFETY proofs (mandatory all `unsafe` blocks)

Anticipated `unsafe` clusters in `lib/` (logical groupings, not literal counts):
- Pointer arithmetic in maple_tree / xarray walking — `kani::proofs::lib::maple_walk_safety`, `kani::proofs::lib::xarray_walk_safety`.
- `iov_iter` segment advance — `kani::proofs::lib::iov_iter::advance_safety`.
- `vsprintf` buffer-bound writes — `kani::proofs::lib::vsprintf::buffer_safety`.
- `usercopy` raw `copy_*_user_inatomic` calls — `kani::proofs::lib::usercopy::*_safety`.
- bitmap raw bit ops — `kani::proofs::lib::bitmap::set_clear_safety`.

Most of `lib/` should require *less* `unsafe` than rust-for-linux's first-cut wrappers — Rust's slicing types remove many bare-pointer-deref needs. A goal of these Tier-3 docs is to minimize `unsafe` further than even existing rust-for-linux versions where possible (and document any reduction in PROOF: comments).

### Layer 2: TLA+ models (mandatory for concurrency primitives)

- `models/lib/xarray.tla` — proves the xarray's RCU-readers + spinlock-writers concurrency contract: readers always observe a coherent tree state.
- `models/lib/maple_tree.tla` — proves maple_tree concurrency invariants (similar shape to xarray).
- `models/lib/stackdepot.tla` — proves stackdepot's RCU-based hash invariants.
- `models/lib/vdso_seqlock.tla` — proves writer-reader coordination for vDSO data updates. (Cross-referenced by `arch/x86/00-overview.md` Layer-2 list — the model lives here in `lib/`, the x86-specific writer side cites it.)
- `models/lib/dynamic_debug.tla` — proves dynamic_debug control-file writes serialize correctly w.r.t. concurrent debug-print sites.

### Layer 3: Kani harnesses for data-structure invariants (mandatory for `lib/` data structures)

`lib/` is heavily data-structural — Layer 3 is its biggest verification area:

| Data structure | Invariant | Harness |
|---|---|---|
| rbtree | Red-black tree invariants (root is black, no two consecutive red, all paths same black-height) | `kani::proofs::lib::rbtree::invariants` |
| list (klist, ll-list) | Doubly-linked correctness: `next->prev == self`, `prev->next == self`, no cycles in regular lists | `kani::proofs::lib::list::link_invariants` |
| maple_tree | Range-keyed tree invariants: all leaves disjoint, in-order, parent-pivot bounds correct | `kani::proofs::lib::maple_tree::invariants` |
| xarray | Radix tree invariants: every internal node has ≥ 1 valid child, no orphan slots | `kani::proofs::lib::xarray::invariants` |
| kfifo | Power-of-two ring invariants: `head - tail` always within capacity, lock-free producer/consumer order | `kani::proofs::lib::kfifo::ring_invariants` |
| min_heap | Heap invariant: parent ≤ children for every node | `kani::proofs::lib::min_heap::heap_invariant` |
| bitmap | Boundary-bit handling, count_set_bits correctness | `kani::proofs::lib::bitmap::*` |
| stackdepot | Hash chain integrity | `kani::proofs::lib::stackdepot::chain_invariants` |
| iov_iter | Cursor invariants: `count + remaining == initial`, segment cursor never beyond segment length | `kani::proofs::lib::iov_iter::cursor_invariants` |

### Layer 4: Functional correctness via Creusot / Verus / Prusti (opt-in)

Strong opt-in candidates (declared in their Tier-3 docs):
- `lightweight-crypto.md` — chacha20, poly1305, blake2s, libsha256 are textbook algorithms with formal specs; **Creusot** functional-correctness proofs are tractable. Recommended target for v0 if budget allows.
- `checksums-hashes.md` — CRC algorithms have polynomial specs; siphash + xxhash are well-defined; **Creusot** suitable.
- `compression.md` — *decompression* correctness (output bytes match a reference decoder for any valid encoded input) is interesting for security; encoders are less critical. Not v0; track for v1+.
- `math.md` — `int_sqrt`, `gcd`, `lcm`, `cordic` are pure math; **Creusot** works well. Lower priority but easy wins.

## Hardening

Placeholder per `00-overview.md` D6. The `lib/` tier owns implementation of:
- Memory sanitization on free for sensitive allocations (per `00-rust-conventions.md` and the deferred `00-security-principles.md` per D6 of overview).
- Bounds-checked indexing throughout (Rust's slice + `.get()` model).
- Constant-time discipline in `lightweight-crypto.md` (the principles doc will mandate constant-time for crypto).
- Userspace-pointer hygiene in `usercopy.md` (UDEREF-equivalent enforced by `UserPtr<T>` per conventions doc).

## Out of Scope

- Test fixtures themselves (`lib/test_*.c`, `lib/tests/`) — not separate design docs; the components they test cover them.
- Specialized helpers with low compatibility surface (`lib/fonts/` is bitmap fonts for VT console — relevant only if `vt-console.md` Tier-4 driver doc needs them; otherwise minimal).
- 32-bit-specific arithmetic (`lib/ashldi3.c`, `lib/cmpdi2.c`, `lib/ashrdi3.c`, `lib/lshrdi3.c`, `lib/muldi3.c`, `lib/ucmpdi2.c`) — these are libgcc fallbacks for 32-bit-on-32-bit; x86_64 has 64-bit instructions natively. OUT OF SCOPE for v0 per `arch/x86/00-overview.md` D1.
- Full ZFS-style coding-style tooling — out of scope; we use existing kernel coding-style tools.
- The pseudo-random number generator's *cryptographic* side — that lives in `crypto/` (Tier-2 `crypto/00-overview.md` not yet authored). `lib/` only carries lightweight non-crypto helpers.
- BPF interpreter `lib/test_bpf.c` — that's a test fixture for `kernel/bpf/`, covered in `kernel/00-overview.md` Phase B.
