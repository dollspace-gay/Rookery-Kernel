---
title: "Tier-5 syscall: getrandom(2) — syscall 318"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getrandom(2)` fills a user buffer with cryptographically secure random bytes drawn from the kernel ChaCha20 CSPRNG. Unlike `/dev/urandom` it requires no fd and blocks correctly until the entropy pool is initially seeded (unless `GRND_NONBLOCK` is set, in which case an unseeded pool yields `-EAGAIN`). `GRND_RANDOM` selects the legacy blocking pool (matches `/dev/random`). `GRND_INSECURE` returns bytes from the CSPRNG even before the pool is seeded — explicitly insecure, intended only for early-boot uses where any pseudo-random is acceptable.

This syscall is the boot-time and library-grade entropy interface used by glibc `arc4random`, OpenSSL, systemd-random-seed, sshd host-key generation, ASLR fallback, and stack-canary initialization. Critical for: CSPRNG correctness, early-boot blocking semantics, signal-safe partial reads.

### Acceptance Criteria

- [ ] AC-1: `getrandom(buf, 16, 0)` after boot-seed: returns 16, `buf` filled.
- [ ] AC-2: `getrandom(buf, 16, GRND_NONBLOCK)` before seed: returns `-EAGAIN`.
- [ ] AC-3: `getrandom(NULL, 16, 0)` returns `-EFAULT`.
- [ ] AC-4: `getrandom(buf, 16, 0x10)` (unknown flag) returns `-EINVAL`.
- [ ] AC-5: `getrandom(buf, 16, GRND_RANDOM | GRND_INSECURE)` returns `-EINVAL`.
- [ ] AC-6: `getrandom(buf, INT_MAX + 1ULL, 0)` returns ≤ `INT_MAX`.
- [ ] AC-7: Two successive 32-byte reads produce distinct outputs (probabilistic).
- [ ] AC-8: Signal during blocking wait, zero bytes copied: returns `-EINTR`.
- [ ] AC-9: Signal mid-large-read after partial copy: returns short count, not `-EINTR`.
- [ ] AC-10: `GRND_INSECURE` pre-seed: returns `buflen` bytes (may be weak).
- [ ] AC-11: vDSO path produces identical bytes-stream for equivalent state.

### Architecture

```rust
#[syscall(nr = 318, abi = "sysv")]
pub fn sys_getrandom(buf: UserPtr<u8>, buflen: usize, flags: u32) -> isize {
    Random::do_getrandom(buf, buflen, flags)
}
```

`Random::do_getrandom(buf, buflen, flags) -> isize`:
1. const VALID = GRND_NONBLOCK | GRND_RANDOM | GRND_INSECURE;
2. if (flags & !VALID) != 0 { return -EINVAL; }
3. if (flags & GRND_RANDOM) != 0 && (flags & GRND_INSECURE) != 0 { return -EINVAL; }
4. let n = min(buflen, i32::MAX as usize);
5. if (flags & GRND_INSECURE) == 0 && !crng_seeded() {
6.   if (flags & GRND_NONBLOCK) != 0 { return -EAGAIN; }
7.   wait_for_random_bytes()?;       // EINTR if signal pre-copy
8. }
9. let mut copied = 0;
10. while copied < n {
11.   let chunk = min(n - copied, CHACHA_BLOCK * 4);
12.   let bytes = if (flags & GRND_RANDOM) != 0 {
13.     blocking_pool_extract(chunk)?  // may short-count
14.   } else {
15.     crng_extract(chunk)
16.   };
17.   buf.add(copied).copy_out_partial(&bytes)?;   // EFAULT
18.   copied += bytes.len();
19.   if signal_pending(current()) { break; }
20. }
21. if copied == 0 && signal_pending(current()) { return -EINTR; }
22. copied as isize

`Random::crng_extract(n) -> Vec<u8>`:
1. let mut key = [0u8; 32]; let mut nonce = [0u8; 12];
2. crng_make_state(&mut key, &mut nonce);          // mixes per-cpu state
3. chacha20_block_stream(key, nonce, n)

### Out of Scope

- ChaCha20 block primitive (covered in Tier-3 `lib/crypto/chacha.md`).
- Interrupt-jitter entropy collection (covered in Tier-3 `drivers/char/random-input.md`).
- vDSO codegen per-arch (covered in arch Tier-3 docs).
- Implementation code.

### signature

```c
ssize_t getrandom(void *buf, size_t buflen, unsigned int flags);
```

```c
#define GRND_NONBLOCK   0x0001
#define GRND_RANDOM     0x0002
#define GRND_INSECURE   0x0004
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `buf` | `void *` | out | User buffer to receive random bytes. |
| `buflen` | `size_t` | in | Number of bytes requested. |
| `flags` | `unsigned int` | in | Bitwise OR of `GRND_*`. Unknown bits return `-EINVAL`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes filled. May be less than `buflen` if a signal arrived after a partial fill (signal-safe partial read). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | `GRND_NONBLOCK` set and entropy pool not yet initially seeded. |
| `EINTR` | Signal delivered before any bytes were written (blocking path). |
| `EFAULT` | `buf` user pointer faults during `copy_to_user`. |
| `EINVAL` | Unknown bit in `flags`; `GRND_RANDOM | GRND_INSECURE` mutually-exclusive combo. |

### abi surface

```text
__NR_getrandom  (x86_64)  = 318
__NR_getrandom  (arm64)   = 278
__NR_getrandom  (riscv)   = 278
__NR_getrandom  (i386)    = 355

/* Buffer length capped at INT_MAX bytes; longer requests
   silently clamp to INT_MAX (returning < buflen is legal). */
/* GRND_INSECURE removes the initial-seed barrier; the
   underlying CSPRNG state may be entirely zero on early boot. */
```

### compatibility contract

REQ-1: Syscall number is **318** on x86_64. ABI-stable since Linux 3.17.

REQ-2: `flags & ~(GRND_NONBLOCK | GRND_RANDOM | GRND_INSECURE)` non-zero → `-EINVAL`.

REQ-3: `GRND_RANDOM | GRND_INSECURE` simultaneously set → `-EINVAL` (semantically contradictory: blocking-pool vs always-non-blocking).

REQ-4: Default (`flags == 0`):
- If CSPRNG seeded: copy `buflen` bytes (or clamped to INT_MAX), return count.
- Else: block in `wait_for_random_bytes()` until first-seed event; then copy.

REQ-5: `GRND_NONBLOCK`:
- If CSPRNG seeded: as default.
- Else: return `-EAGAIN` immediately without copying.

REQ-6: `GRND_RANDOM`:
- Draw from the historical `/dev/random` blocking pool.
- May return a short count if entropy estimate exhausts mid-read.

REQ-7: `GRND_INSECURE`:
- Draw from CSPRNG regardless of seed state.
- May return uniform-zero or attacker-predictable bytes during very early boot — caller MUST treat output as best-effort PRNG, never as key material.

REQ-8: Signal handling: if a signal arrives mid-blocking-wait with zero bytes copied → `-EINTR` (restartable iff `SA_RESTART`). If signal arrives after a partial copy, syscall returns the short count (signal-safe contract).

REQ-9: Buffer length clamp: `buflen > INT_MAX` returns at most `INT_MAX` bytes; not an error.

REQ-10: No fd, no offset, no state — `getrandom(2)` is stateless from the caller's perspective.

REQ-11: vDSO acceleration: an architecture-dependent vDSO `__vdso_getrandom` may serve small reads without syscall entry (still backed by the kernel CSPRNG state mapped read-only to userspace). vDSO path enforces the same flag and seed semantics.

REQ-12: Per-`/proc/sys/kernel/random/urandom_min_reseed_secs` reseed cadence applies; getrandom always observes the latest reseed.

REQ-13: Per-`/proc/sys/kernel/random/boot_id` and `entropy_avail` are not affected by getrandom calls (they reflect pool state, not consumption).

REQ-14: `GRND_INSECURE` is permitted only when `CONFIG_RANDOM_TRUST_BOOTLOADER` or `random.trust_cpu=1` set, or operator explicitly opts in; grsec hardening removes this flag (see § Grsecurity).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | unknown flag bit ⟹ EINVAL pre-copy. |
| `mutex_random_insecure` | INVARIANT | (GRND_RANDOM & GRND_INSECURE) ⟹ EINVAL. |
| `pre_seed_block_or_eagain` | INVARIANT | !seeded ∧ !INSECURE ⟹ (block ∨ EAGAIN). |
| `signal_partial_or_eintr` | INVARIANT | signal ⟹ (short-count if copied>0) ∨ (EINTR if copied==0). |
| `clamp_int_max` | INVARIANT | buflen > INT_MAX ⟹ copied ≤ INT_MAX. |
| `copy_to_user_bounded` | INVARIANT | total bytes written to user == returned count. |

### Layer 2: TLA+

`drivers/char/random-getrandom.tla`:
- States: per-flag-validate, per-seed-check, per-block/eagain, per-extract, per-copy-out, per-signal.
- Properties:
  - `safety_no_copy_before_seed_unless_insecure`.
  - `safety_no_copy_after_signal_with_zero_bytes`.
  - `safety_flag_combo_validated_first`.
  - `liveness_eventually_returns_or_seeded`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getrandom` post: ret > 0 ⟹ user buf populated ret bytes | `Random::do_getrandom` |
| `crng_extract` post: output uniformly distributed over CSPRNG | `Random::crng_extract` |
| `wait_for_random_bytes` post: crng_seeded() == true | `Random::wait_for_random_bytes` |
| `GRND_INSECURE` post: returns regardless of seed state | `Random::do_getrandom` |

### Layer 4: Verus / Creusot functional

Per-`getrandom(2)` man-page and per-`drivers/char/random.c` ChaCha20 stream cipher semantic equivalence. LTP tests `getrandom01..04` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getrandom(2)` reinforcement:

- **Per-flag validation pre-copy** — defense against per-extension-field smuggling.
- **Per-INT_MAX clamp** — defense against per-buflen-overflow.
- **Per-signal-safe partial read** — defense against per-restart-loss-of-entropy.
- **Per-pre-seed block** — defense against per-weak-key (boot-time CSPRNG empty).
- **Per-vDSO path consistency** — defense against per-vdso-side-channel divergence.

### grsecurity / pax surface

- **PaX UDEREF on `buf` copy_to_user** — defense against per-attr-pointer kernel-deref bug; SMAP forced on every chunk.
- **GRND_INSECURE bound by `grsec_random_insecure_disable`** — when GRKERNSEC_RANDOM_HARDEN is set, `GRND_INSECURE` returns `-EINVAL`. Insecure-pre-seed reads are categorically denied; callers must block.
- **CAP_SYS_ADMIN gate on `GRND_INSECURE`** — under hardened mode, even when allowed it requires `CAP_SYS_ADMIN` in init_userns.
- **GRKERNSEC_RANDOM_LOCKDOWN** — `kernel_lockdown(integrity)` blocks all writes to `/proc/sys/kernel/random/*` (reseed cadence, write_wakeup_threshold). getrandom output unaffected, but pool tampering forbidden.
- **PAX_USERCOPY_HARDEN on output chunk copy_to_user** — bounded copy uses whitelisted bounce buffer; no slab leak.
- **Per-CPU CSPRNG state KASLR-protected** — defense against per-state-leak (forces re-extraction from primary pool on suspicious read).
- **GRKERNSEC_HIDESYM on `/proc/kallsyms` for random.c symbols** — defense against locating CSPRNG state via symbol address.
- **GRKERNSEC_AUDIT_GETRANDOM** — every `GRND_INSECURE` call logged with PID, comm, buflen, audit trail. Used to detect misuse in production.
- **GRKERNSEC_VDSO_HARDEN** — vDSO getrandom page mapped read-only, never executable; per-task state under PaX page-table isolation.
- **PAX_REFCOUNT on entropy-extractor refcounts** — defense against per-refcount-overflow during stress.
- **Per-task rate-limit on getrandom** — when `grsec_random_rate_limit_bytes_per_sec` set, callers exceeding the per-uid budget are throttled (returns short reads). Defense against per-CSPRNG-DoS by tight-loop reads.
- **Pool boot-trust disabled by default** — `random.trust_bootloader=0` and `random.trust_cpu=0` forced; CSPRNG must collect interrupt-jitter entropy itself before seeding.

