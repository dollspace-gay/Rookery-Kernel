---
title: "Tier-5 syscall: personality(2) — syscall 135"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`personality(2)` sets the calling process's execution-domain "personality" — a bitfield that historically emulated foreign-Unix ABI behaviors (SVR4, BSD, SCO, HP-UX, IRIX, Solaris, ...) and today is used almost exclusively for hardening or compatibility toggles such as `ADDR_NO_RANDOMIZE` (disable ASLR), `ADDR_COMPAT_LAYOUT` (use legacy mmap layout), `UNAME26` (make `uname(2)` report kernel version 2.6.x for ancient userspace), `READ_IMPLIES_EXEC` (every readable VMA is also executable; required by old binaries), and `MMAP_PAGE_ZERO` (mmap the zero page so null-pointer accesses don't fault — Wine/DOS emulators).

The personality is stored in `task.personality` (a `u32`) and is split into a low-byte "execution domain" (`PER_*`) plus per-bit modifiers. The execution domain is mostly a no-op in modern Linux (the SVR4 / BSD / iBCS2 emulation layers have been removed); only the modifiers matter for behavior. Passing `0xFFFFFFFF` ("read current") returns the current value without changing it.

Critical for: `setarch(8)` toolchain (which sets `ADDR_NO_RANDOMIZE` for reproducible builds and `UNAME26` for legacy installers), exec-format loaders (binfmt_elf may set `READ_IMPLIES_EXEC` based on ELF headers), CRIU snapshot, and security policy hooks (PaX / grsec lock out `ADDR_NO_RANDOMIZE` setuid).

This Tier-5 covers `personality(2)` on x86_64; the syscall is generic-unistd-equivalent.

### Acceptance Criteria

- [ ] AC-1: `personality(0xFFFFFFFF)` returns the current personality without modifying it.
- [ ] AC-2: `personality(PER_LINUX | ADDR_NO_RANDOMIZE)` then subsequent `mmap(NULL, 4096, PROT_READ, MAP_PRIVATE|MAP_ANON, -1, 0)` returns the same address on repeated execve cycles (ASLR off).
- [ ] AC-3: `personality(UNAME26)` then `uname(2)` returns `release` starting with `"2.6."`.
- [ ] AC-4: `personality(READ_IMPLIES_EXEC)` then `mmap(NULL, 4096, PROT_READ, ...)` returns a VMA with `PROT_READ | PROT_EXEC`.
- [ ] AC-5: `personality(MMAP_PAGE_ZERO)` then `mmap(0, 4096, PROT_READ, MAP_FIXED|MAP_ANON, -1, 0)` succeeds even with `vm.mmap_min_addr > 0`.
- [ ] AC-6: `personality(0x10000000)` (reserved bit) returns `-EINVAL` (or quietly accepts depending on kernel version; modern: `-EINVAL`).
- [ ] AC-7: `personality(persona)` returns the OLD personality (pre-mutation).
- [ ] AC-8: After `fork`, child has the same `personality` as parent.
- [ ] AC-9: After `execve` of a binary with `PT_GNU_STACK PF_R|PF_W`, `READ_IMPLIES_EXEC` is cleared.
- [ ] AC-10: After `execve` of a binary without `PT_GNU_STACK`, `READ_IMPLIES_EXEC` is set.
- [ ] AC-11: `setarch x86_64 --addr-no-randomize ./binary` runs `binary` with `ADDR_NO_RANDOMIZE` set; `personality(0xFFFFFFFF)` from within reflects it.
- [ ] AC-12: Under grsec policy + suid binary, `personality(ADDR_NO_RANDOMIZE)` returns `-EPERM`.
- [ ] AC-13: Per-LTP `personality01..02` pass.

### Architecture

```rust
#[syscall(nr = 135, abi = "sysv")]
pub fn sys_personality(persona: u64) -> isize {
    Personality::dispatch(persona as u32)
}
```

`Personality::dispatch(persona) -> isize`:
1. let old = current.personality;
2. if persona == 0xFFFF_FFFF {
3.   return old as isize;
4. }
5. /* Validate reserved bits */
6. let allowed = PER_MASK | UNAME26 | ADDR_NO_RANDOMIZE | FDPIC_FUNCPTRS
7.              | MMAP_PAGE_ZERO | ADDR_COMPAT_LAYOUT | READ_IMPLIES_EXEC
8.              | ADDR_LIMIT_32BIT | SHORT_INODE | WHOLE_SECONDS
9.              | STICKY_TIMEOUTS | ADDR_LIMIT_3GB;
10. if persona & !allowed != 0 { return Err(EINVAL); }
11. /* Grsec / hardening policy hook */
12. if (persona & ADDR_NO_RANDOMIZE) != 0 && grsec::aslr_locked() {
13.   return Err(EPERM);
14. }
15. /* Apply */
16. current.personality = persona;
17. old as isize

`exec::set_personality(task, new)`:
1. /* Preserve administrative bits across exec */
2. let preserved = task.personality & (ADDR_NO_RANDOMIZE | UNAME26 | ADDR_COMPAT_LAYOUT);
3. task.personality = (new & !ADDR_NO_RANDOMIZE) | preserved;
4. /* binfmt may then set READ_IMPLIES_EXEC based on PT_GNU_STACK absence */

`uname(2) personality_filter(rel: &str) -> String`:
1. if current.personality & UNAME26 == 0 { return rel.to_string(); }
2. /* Real release: "X.Y.Z..." e.g. "7.1.0-rc2" */
3. let (major, minor, patch) = parse_release(rel);
4. let forged_minor = (minor + 40) % 256;
5. format!("2.6.{}+{}", forged_minor, patch)

### Out of Scope

- ASLR implementation (Tier-3 in `mm/aslr.md`).
- `mmap(2)` layout selection (Tier-3 in `mm/mmap.md`).
- binfmt_elf header parsing (Tier-3 in `fs/binfmt_elf.md`).
- `uname(2)` (separate Tier-5 doc).
- `setarch(8)` userspace.
- Implementation code.

### signature

```c
int personality(unsigned long persona);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `persona` | `unsigned long` | in | New personality bitfield, or `0xFFFFFFFF` to read current. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Previous personality (always returned, even on "set" success). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL`  | Reserved bits set; or under grsec policy, attempted to set a forbidden flag (e.g., `ADDR_NO_RANDOMIZE` on a hardened task). |
| `EPERM`   | Under grsec: tried to set `ADDR_NO_RANDOMIZE` on a suid binary or under chroot. |
| `ENOSYS`  | Compiled out (rare). |

### abi surface

```text
__NR_personality (x86_64) = 135
__NR_personality (i386)   = 136

Execution domains (low byte) — PER_MASK = 0x00FF
  PER_LINUX             0x0000
  PER_LINUX_32BIT       0x0000 | ADDR_LIMIT_32BIT
  PER_LINUX_FDPIC       0x0000 | FDPIC_FUNCPTRS
  PER_SVR4              0x0001 | STICKY_TIMEOUTS | MMAP_PAGE_ZERO
  PER_SVR3              0x0002 | STICKY_TIMEOUTS | SHORT_INODE
  PER_SCOSVR3           0x0003 | STICKY_TIMEOUTS | WHOLE_SECONDS | SHORT_INODE
  PER_OSR5              0x0003 | STICKY_TIMEOUTS | WHOLE_SECONDS
  PER_WYSEV386          0x0004 | STICKY_TIMEOUTS | SHORT_INODE
  PER_ISCR4             0x0005 | STICKY_TIMEOUTS
  PER_BSD               0x0006
  PER_SUNOS             0x0006 | STICKY_TIMEOUTS
  PER_XENIX             0x0007 | STICKY_TIMEOUTS | SHORT_INODE
  PER_LINUX32           0x0008
  PER_LINUX32_3GB       0x0008 | ADDR_LIMIT_3GB
  PER_IRIX32            0x0009 | STICKY_TIMEOUTS
  PER_IRIXN32           0x000A | STICKY_TIMEOUTS
  PER_IRIX64            0x000B | STICKY_TIMEOUTS
  PER_RISCOS            0x000C
  PER_SOLARIS           0x000D | STICKY_TIMEOUTS
  PER_UW7               0x000E | STICKY_TIMEOUTS | MMAP_PAGE_ZERO
  PER_OSF4              0x000F
  PER_HPUX              0x0010

Modifier bits (high 24 bits) — ADDR_* / MMAP_PAGE_ZERO / UNAME26 / ...
  UNAME26               0x0020000   /* uname reports 2.6.40+0..39 */
  ADDR_NO_RANDOMIZE     0x0040000   /* disable ASLR for this task */
  FDPIC_FUNCPTRS        0x0080000
  MMAP_PAGE_ZERO        0x0100000
  ADDR_COMPAT_LAYOUT    0x0200000   /* legacy mmap layout */
  READ_IMPLIES_EXEC     0x0400000   /* PROT_READ ⟹ PROT_EXEC */
  ADDR_LIMIT_32BIT      0x0800000   /* mmap stays in low 32 bits */
  SHORT_INODE           0x1000000   /* 16-bit ino_t */
  WHOLE_SECONDS         0x2000000
  STICKY_TIMEOUTS       0x4000000   /* select(2) does not update timeout on EINTR */
  ADDR_LIMIT_3GB        0x8000000   /* mmap stays in low 3 GiB (32-bit compat) */
```

### Limits

```text
persona            : u32; reserved bits (above 0x8FFFFFFF) ⟹ -EINVAL
read-current        : persona == 0xFFFFFFFF returns current, no mutation
inherit             : fork copies parent's personality
execve              : default-preserves personality; binfmt may override (e.g., binfmt_elf sets READ_IMPLIES_EXEC if PT_GNU_STACK is missing)
```

### compatibility contract

REQ-1: Syscall number is **135** on x86_64. ABI-stable since 1.2.

REQ-2: `persona == 0xFFFFFFFF`: returns the current `task.personality` value without modification. This is the documented "query" idiom.

REQ-3: Otherwise: stores `persona & 0xFFFFFFFF` (truncate to 32 bits, since `task.personality` is `u32`) into `current.personality`. Returns the OLD value.

REQ-4: Reserved bits (bits 0x10000000 .. 0x80000000 outside the defined mask) MUST cause `-EINVAL`. Defined mask is `PER_MASK | UNAME26 | ADDR_NO_RANDOMIZE | FDPIC_FUNCPTRS | MMAP_PAGE_ZERO | ADDR_COMPAT_LAYOUT | READ_IMPLIES_EXEC | ADDR_LIMIT_32BIT | SHORT_INODE | WHOLE_SECONDS | STICKY_TIMEOUTS | ADDR_LIMIT_3GB`.

REQ-5: `PER_LINUX` (0): no behavioral change beyond modifier bits.

REQ-6: `ADDR_NO_RANDOMIZE`: future mmap / brk / stack-setup calls do NOT randomize addresses. Affects ALL ASLR-relevant placement (mmap base, stack base, brk base, vDSO, exec image for PIE). Persists until cleared or execve replaces it (binfmt may clear).

REQ-7: `UNAME26`: subsequent `uname(2)` and `/proc/sys/kernel/osrelease` return a forged version string of the form "2.6.MM+PP" derived from the actual kernel version, where MM is `(real_minor + 40) % 256` and PP is the patchlevel — chosen so any kernel ≥ 3.0 reports a 2.6.x-shaped string. Defeats old installers that hard-check for `2.6`.

REQ-8: `READ_IMPLIES_EXEC`: every subsequent `mmap(PROT_READ, ...)` is silently upgraded to `PROT_READ | PROT_EXEC`. Required by ELF binaries lacking `PT_GNU_STACK` (typically pre-2.6.13). Set by binfmt_elf at execve based on header inspection.

REQ-9: `MMAP_PAGE_ZERO`: `mmap` of the zero page (address 0..PAGE_SIZE) is permitted by the kernel for this task; used by DOSEMU, Wine NT layer, BSD compat. Default kernel refuses such mappings (`mmap_min_addr` sysctl).

REQ-10: `ADDR_COMPAT_LAYOUT`: future mmap uses the legacy 2.6.x-style layout (heap grows up, mmap grows down from `TASK_SIZE/3`) rather than the modern layout. Useful for tools that assume specific address ranges.

REQ-11: `ADDR_LIMIT_32BIT` / `ADDR_LIMIT_3GB`: confine future mmap to low 32 bits or low 3 GiB respectively. Used by `setarch i386 ...` to simulate 32-bit address layout for testing.

REQ-12: `SHORT_INODE`: legacy `stat(2)` returns 16-bit `ino_t` (truncates high bits). Affects `oldstat` / SVR3 emulation only.

REQ-13: `WHOLE_SECONDS`: `time(2)`-equivalent returns whole seconds only, not microseconds.

REQ-14: `STICKY_TIMEOUTS`: `select(2)` / `poll(2)` interrupted by signal do NOT update the timeout argument — old SVR4 semantics.

REQ-15: `FDPIC_FUNCPTRS`: function pointers in FDPIC ABI are descriptor pointers, not code pointers. Set by binfmt_elf_fdpic at execve.

REQ-16: Per-fork: child inherits `task.personality` verbatim.

REQ-17: Per-execve: binfmt's `set_personality()` is called; default-keeps modifier bits and resets execution-domain low byte. Specifically:
- binfmt_elf preserves `ADDR_NO_RANDOMIZE`, `UNAME26`, `ADDR_COMPAT_LAYOUT` (administrative choices stay).
- binfmt_elf sets `READ_IMPLIES_EXEC` only if `PT_GNU_STACK` is missing or `PF_X`.
- binfmt_elf clears `STICKY_TIMEOUTS`, `WHOLE_SECONDS`, `SHORT_INODE`, `MMAP_PAGE_ZERO` (these are domain-specific, not administrative).

REQ-18: Per-CRIU: dump captures `task.personality`; restore reissues `personality()` after exec to recreate the bits.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `reserved_bits_rejected` | INVARIANT | per-dispatch: undefined bits ⟹ -EINVAL. |
| `query_no_mutation` | INVARIANT | per-dispatch: persona == 0xFFFFFFFF ⟹ no write. |
| `returns_old_value` | INVARIANT | per-dispatch: ok ⟹ return == old_personality. |
| `aslr_locked_under_grsec` | INVARIANT | per-dispatch: ADDR_NO_RANDOMIZE + grsec ⟹ -EPERM. |
| `execve_preserves_admin_bits` | INVARIANT | per-set_personality: ADDR_NO_RANDOMIZE/UNAME26/ADDR_COMPAT_LAYOUT preserved. |

### Layer 2: TLA+

`kernel/personality.tla`:
- States: per-task personality u32; flag-set semantics for ADDR_*, UNAME26, READ_IMPLIES_EXEC.
- Properties:
  - `safety_reserved_bits` — set ⟹ persona ⊆ allowed_mask.
  - `safety_query_no_mutation` — read-current is pure.
  - `safety_return_old` — return == old.
  - `safety_execve_preserves` — administrative bits survive exec.
  - `liveness_terminates` — call returns in bounded time.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: persona == 0xFFFFFFFF ⟹ personality unchanged | `Personality::dispatch` |
| `dispatch` post: ok ⟹ personality == persona; return == old | `Personality::dispatch` |
| `dispatch` post: err ⟹ personality unchanged | `Personality::dispatch` |
| `set_personality` post: administrative bits preserved | `exec::set_personality` |

### Layer 4: Verus/Creusot functional

Per-`personality(2)` man-page semantic equivalence. LTP `personality01..02` and `setarch(8)` smoke pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`personality(2)` reinforcement:

- **Per-reserved-bit `-EINVAL`** — defense against per-future-flag silent acceptance.
- **Per-grsec ASLR-lock** — defense against per-ASLR-bypass via `setarch --addr-no-randomize`.
- **Per-suid `READ_IMPLIES_EXEC` clear** — defense against per-suid-binary inheriting unsafe semantics.
- **Per-`MMAP_PAGE_ZERO` administrative-only** — defense against per-NULL-deref exploitation; usually disabled by `mmap_min_addr` sysctl.
- **Per-`STICKY_TIMEOUTS` scoped** — defense against per-timeout-update bypass leaking signal-mask info.
- **Per-execve administrative-preserve** — defense against per-setarch policy bypass via exec.
- **Per-CRIU snapshot integrity** — defense against per-restore inconsistency.

### grsecurity / pax surface

- **GRKERNSEC_RAND** — `ADDR_NO_RANDOMIZE` may be denied entirely per-policy (returns `-EPERM`).
- **PaX RANDMMAP / RANDEXEC** — `ADDR_NO_RANDOMIZE` ignored at mmap time when PaX RANDMMAP is on; bit is stored but has no behavioral effect (defense in depth).
- **UNAME26 personality-hidden under suid** — when a suid binary calls `personality(0xFFFFFFFF)`, the returned value masks out `UNAME26` and `ADDR_NO_RANDOMIZE` to prevent the binary from making policy decisions based on attacker-set bits.
- **PaX UDEREF** — no user pointer in `personality`, but the syscall is a common pre-exploit fingerprint and is hooked for audit.
- **PAX_RANDKSTACK at personality entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_CHROOT_NO_PERSONALITY** — inside a grsec chroot, `personality(persona != PER_LINUX)` returns `-EPERM`.
- **Per-grsec RBAC** — `+P` flag (personality-set-allow) gates per-subject; default policy permits PER_LINUX only.
- **Per-grsec audit** — every `personality(persona != 0xFFFFFFFF)` logged with task cred, old value, new value, and RIP.
- **GRKERNSEC_SETXID** — `personality` denied on tasks that have just performed setuid/setgid until the next execve, preventing personality-based privilege escalation paths.
- **PaX MPROTECT integration** — `READ_IMPLIES_EXEC` cannot be set on a PaX MPROTECT-locked task; returns `-EPERM`.
- **Per-grsec `MMAP_PAGE_ZERO`-block** — even with successful personality set, `mmap(0, ...)` is denied; the personality bit is recorded but inoperative.

