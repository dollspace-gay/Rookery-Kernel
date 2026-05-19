# Tier-5 syscall: get_thread_area(2) — syscall 211

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/kernel/tls.c (do_get_thread_area)
  - arch/x86/include/asm/desc.h
  - arch/x86/include/uapi/asm/ldt.h (struct user_desc)
  - arch/x86/entry/syscalls/syscall_64.tbl (211  64  get_thread_area)
  - arch/x86/entry/syscalls/syscall_32.tbl (244  i386 get_thread_area)
-->

## Summary

`get_thread_area(2)` reads back one of the calling thread's three TLS (thread-local storage) GDT slots (`GDT_ENTRY_TLS_MIN=12 .. GDT_ENTRY_TLS_MAX=14`) into a user-supplied `struct user_desc`. It is the read-side cousin of `set_thread_area(2)` and is used by `glibc`/`musl` for self-introspection, by `ptrace`-based debuggers (`PTRACE_GET_THREAD_AREA`), and crucially by CRIU during checkpoint to snapshot per-thread GDT TLS state. The kernel decodes the per-thread cached descriptor (`task.thread.tls_array[slot - GDT_ENTRY_TLS_MIN]`) back into the `user_desc` form rather than re-reading from the live GDT — this guarantees a consistent view across CPU migration.

Critical for: CRIU per-thread checkpoint, ptrace-based debuggers (gdb, lldb, strace `-iv`), 32-bit thread-init validation paths in glibc, and any runtime that introspects its own `%gs` base for sanity-checking.

This Tier-5 covers `get_thread_area(2)` on x86_64 and i386. The struct layout is identical between architectures.

## Signature

```c
int get_thread_area(struct user_desc *u_info);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `u_info` | `struct user_desc *` | in/out | On entry: `entry_number` field selects the slot (12, 13, or 14). On exit: full descriptor written. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. `u_info` populated. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL`  | `u_info == NULL`; `entry_number` not in `[GDT_ENTRY_TLS_MIN, GDT_ENTRY_TLS_MAX]`. |
| `EFAULT`  | `u_info` not readable / writable. |
| `ENOSYS`  | Compiled out (rare). |

## ABI surface

```text
__NR_get_thread_area (i386)   = 244
__NR_get_thread_area (x86_64) = 211

GDT_ENTRY_TLS_MIN = 12
GDT_ENTRY_TLS_MAX = 14

struct user_desc {                /* same layout as set_thread_area / modify_ldt */
    unsigned int  entry_number;
    unsigned int  base_addr;
    unsigned int  limit;
    unsigned int  seg_32bit:1;
    unsigned int  contents:2;
    unsigned int  read_exec_only:1;
    unsigned int  limit_in_pages:1;
    unsigned int  seg_not_present:1;
    unsigned int  useable:1;
    unsigned int  lm:1;
};

empty slot, on read, decodes as:
  base_addr=0, limit=0, seg_32bit=0, contents=0,
  read_exec_only=1, limit_in_pages=0, seg_not_present=1, useable=0
```

### Limits

```text
entry_number  : 12, 13, or 14 (others return -EINVAL)
slots/thread  : 3
```

## Compatibility contract

REQ-1: Syscall number is **211** on x86_64, **244** on i386. ABI-stable.

REQ-2: `u_info == NULL` returns `-EINVAL`.

REQ-3: `entry_number ∉ [12,14]` returns `-EINVAL`. No state mutation, no partial write.

REQ-4: On success, the kernel decodes the cached 8-byte descriptor from `task.thread.tls_array[entry_number - GDT_ENTRY_TLS_MIN]` into the `user_desc` fields and copies the full struct to `*u_info`. `entry_number` is left as-is (caller already set it).

REQ-5: Empty slot semantics: a slot that has never been written, or has been cleared via `set_thread_area` with `seg_not_present=1 && base=0 && limit=0`, decodes to:
- `base_addr = 0`
- `limit = 0`
- `seg_32bit = 0`
- `contents = 0`
- `read_exec_only = 1`
- `limit_in_pages = 0`
- `seg_not_present = 1`
- `useable = 0`
- `lm = 0`

This is the canonical "empty descriptor" pattern.

REQ-6: The read is per-thread: it reflects the calling thread's `tls_array`, not the running CPU's live GDT (which may belong to a different thread mid-context-switch).

REQ-7: `ptrace(PTRACE_GET_THREAD_AREA, tid, idx, &u_info)` shares the same decode path against the tracee's `tls_array`.

REQ-8: Per-fork / per-clone(CLONE_SETTLS): the child's `tls_array` is initialized correctly; an immediate `get_thread_area` from the child returns the expected descriptor.

REQ-9: Per-execve: all three slots clear; reads return the empty-descriptor pattern.

REQ-10: x86_64 native non-compat: still works. The fields returned reflect the 32-bit TLS slots regardless of process bitness. A 64-bit process that has not used `set_thread_area` will see empty descriptors in all three slots.

REQ-11: x32 ABI: same syscall number (211) is reachable via `0x40000000 | 211`. Behavior identical.

REQ-12: Atomic snapshot: the kernel reads `tls_array[idx]` under preempt-disabled scope so that an in-flight `set_thread_area` from another thread cannot tear the read. Since `tls_array` is per-thread, cross-thread tear is impossible by construction; the lock is defense-in-depth.

REQ-13: Decode is the exact inverse of `set_thread_area`'s encode for any value the latter produces. DPL bits in the cached descriptor are forced 3 by encode and are not exposed to userspace.

REQ-14: The `useable` (AVL) bit is read back verbatim from what was last installed.

REQ-15: `lm` bit: always 0 on read — the TLS path forbids `lm=1` on install; legacy / corrupt `tls_array` entries are decoded as `lm=0`.

REQ-16: No side effects: `get_thread_area` is pure (modulo the writeback). Repeated calls return the same data until `set_thread_area` / `clone(CLONE_SETTLS)` / `execve` mutates the slot.

## Acceptance Criteria

- [ ] AC-1: `set_thread_area(&in)` with valid descriptor, then `get_thread_area(&out)` with `out.entry_number = in.entry_number` returns 0 and `out` equals `in` modulo the `entry_number` write-back.
- [ ] AC-2: `get_thread_area(NULL)` returns `-EINVAL`.
- [ ] AC-3: `get_thread_area(&u)` with `u.entry_number = 11` returns `-EINVAL`.
- [ ] AC-4: `get_thread_area(&u)` with `u.entry_number = 15` returns `-EINVAL`.
- [ ] AC-5: On a freshly `fork`'d child, `get_thread_area(&u)` with `u.entry_number = 12` returns the parent's slot-12 descriptor.
- [ ] AC-6: On a freshly `execve`'d process, `get_thread_area(&u)` for all three slots returns the empty-descriptor pattern (`base=0,limit=0,seg_not_present=1,read_exec_only=1`).
- [ ] AC-7: `ptrace(PTRACE_GET_THREAD_AREA, tid, 12, &u)` returns the tracee's slot-12 descriptor verbatim.
- [ ] AC-8: After `set_thread_area` clears a slot, `get_thread_area` on that slot returns the empty descriptor.
- [ ] AC-9: Concurrent thread A `set_thread_area`(slot 12) and thread B `get_thread_area`(slot 12) never tear: B's read is either before or after A's write.
- [ ] AC-10: `get_thread_area` does NOT modify any field other than the descriptor body; `u.entry_number` is preserved.
- [ ] AC-11: Per-LTP `get_thread_area01` passes.

## Architecture

```rust
#[syscall(nr = 211, abi = "sysv")]
pub fn sys_get_thread_area(u_info: UserPtr<UserDesc>) -> isize {
    GetThreadArea::dispatch(u_info)
}
```

`GetThreadArea::dispatch(u_info) -> isize`:
1. if u_info.is_null() { return Err(EINVAL); }
2. let in_desc: UserDesc = u_info.read()?;
3. let slot = in_desc.entry_number as usize;
4. if slot < GDT_ENTRY_TLS_MIN || slot > GDT_ENTRY_TLS_MAX { return Err(EINVAL); }
5. let idx = slot - GDT_ENTRY_TLS_MIN;
6. let enc: u64 = preempt_disabled(|| current.thread.tls_array[idx]);
7. let mut out: UserDesc = decode_descriptor(enc);
8. out.entry_number = slot as u32;
9. u_info.write(&out)?;
10. Ok(0)

`decode_descriptor(enc) -> UserDesc`:
1. let base = ((enc >> 16) & 0xFF_FFFF) | (((enc >> 56) & 0xFF) << 24);
2. let limit_raw = (enc & 0xFFFF) | (((enc >> 48) & 0xF) << 16);
3. let access = (enc >> 40) & 0xFF;
4. let flags = (enc >> 52) & 0xF;
5. if is_empty_encoding(enc) {
6.   return UserDesc {
7.     entry_number: 0, base_addr: 0, limit: 0,
8.     seg_32bit: 0, contents: 0, read_exec_only: 1,
9.     limit_in_pages: 0, seg_not_present: 1, useable: 0, lm: 0,
10.  };
11. }
12. UserDesc {
13.   entry_number: 0,                            // caller fills in
14.   base_addr: base as u32,
15.   limit: limit_raw as u32,
16.   seg_32bit: ((flags >> 2) & 1) as u32,
17.   contents: ((access >> 2) & 3) as u32,       // type field [3:2]
18.   read_exec_only: !((access >> 1) & 1) as u32,
19.   limit_in_pages: ((flags >> 3) & 1) as u32,
20.   seg_not_present: !((access >> 7) & 1) as u32,
21.   useable: ((flags >> 0) & 1) as u32,         // AVL
22.   lm: ((flags >> 1) & 1) as u32,              // L bit
23. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `slot_in_range` | INVARIANT | per-dispatch: slot ∈ {12,13,14} or EINVAL. |
| `decode_inverse_encode` | INVARIANT | per-decode: for any value encode() produces, decode(encode(d)) == d. |
| `empty_decodes_canonical` | INVARIANT | per-decode: empty encoding ⟹ canonical empty descriptor. |
| `preempt_safe_snapshot` | INVARIANT | per-read: `tls_array[idx]` read under preempt-off. |
| `no_side_effects` | INVARIANT | per-dispatch: only writes `*u_info`; no `tls_array` mutation. |

### Layer 2: TLA+

`arch/x86/get_thread_area.tla`:
- States: per-thread tls_array[3]; observer view via get_thread_area.
- Properties:
  - `safety_decode_consistent` — for any thread T, get_thread_area(slot) == decode(T.tls_array[slot-12]).
  - `safety_no_tear` — concurrent set/get on same slot: get sees pre- or post-state, never partial.
  - `safety_per_thread` — A.get_thread_area(slot) observes A.tls_array, never B's.
  - `liveness_terminates` — call returns errno or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: u_info populated, return 0 ⟹ slot valid | `GetThreadArea::dispatch` |
| `decode_descriptor` post: round-trip with encode is identity | `decode_descriptor` |
| `preempt_disabled` post: tls_array read atomically | `GetThreadArea::dispatch` |

### Layer 4: Verus/Creusot functional

Per-`get_thread_area(2)` man-page semantic equivalence. Round-trip property `set_thread_area(d) ; get_thread_area(slot) ⟹ d'` where d' equals d modulo the `entry_number` write-back. LTP `get_thread_area01` passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`get_thread_area(2)` reinforcement:

- **Per-slot bounds check** — defense against per-out-of-range GDT read.
- **Per-thread `tls_array` snapshot** — defense against per-mid-write tear (preempt-off scope).
- **Per-DPL bits scrubbed on decode** — defense against per-kernel-DPL-leak (DPL bits in cached form are always 3; not surfaced).
- **Per-`lm` forced 0 on decode** — defense against per-corrupt-tls_array misuse.
- **Per-empty-slot canonical decode** — defense against per-uninitialized-memory leak.
- **Pure read semantics** — defense against per-tls_array-write through a `get_*` syscall.

## Grsecurity / PaX surface

- **PaX UDEREF on `u_info`** — defense against per-malicious-user-pointer kernel deref bug.
- **PAX_RANDKSTACK at get_thread_area entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_KMEM** — `tls_array` never exposed via `/proc/<pid>/mem` or `/dev/kmem`; this syscall is the only legitimate read path.
- **PaX SEGMEXEC integration** — decoded base/limit are clipped against the SEGMEXEC code-segment limit if a SEGMEXEC task reads (prevents disclosure of out-of-band high-half kernel addresses encoded into the slot).
- **Per-grsec audit** — `get_thread_area` is typically NOT audited (read-only, high volume from runtimes); opt-in only.
- **GRKERNSEC_CHROOT_NO_INTROSPECT** — inside grsec chroot with introspection denial, `get_thread_area` from non-self ptrace returns `-EPERM`.
- **Per-grsec RBAC** — `+T` flag controls TLS introspection per-subject.
- **PaX kernel-pointer leak prevention** — decode never exposes raw 8-byte hardware descriptor; only the public `user_desc` view.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- x86 segmentation hardware semantics (Tier-3 in `arch/x86/segments.md`).
- `set_thread_area(2)` (separate Tier-5 doc).
- `ptrace(PTRACE_GET_THREAD_AREA, ...)` (covered in `ptrace.md`).
- CRIU checkpoint/restore details (Tier-3 in `kernel/checkpoint.md`).
- Implementation code.
