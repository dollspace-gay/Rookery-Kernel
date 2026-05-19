# Tier-5 syscall: userfaultfd(2) — syscall 323

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/userfaultfd.c (SYSCALL_DEFINE1(userfaultfd), userfaultfd_register, userfaultfd_copy)
  - include/uapi/linux/userfaultfd.h (UFFDIO_API, UFFDIO_REGISTER, UFFD_FEATURE_*, UFFD_USER_MODE_ONLY)
  - mm/memory.c (handle_userfault)
  - arch/x86/entry/syscalls/syscall_64.tbl (323  common  userfaultfd)
-->

## Summary

`userfaultfd(2)` creates a file descriptor representing a user-space page-fault handler. After the fd is registered against a range of the calling process's address space, every page fault inside that range — major (missing page), minor (existing page hidden by absent ptes), or write-protect — is paused in the kernel and a message is delivered to the fd. A user-space handler reads the message, performs an `UFFDIO_COPY`, `UFFDIO_ZEROPAGE`, `UFFDIO_CONTINUE`, or `UFFDIO_WRITEPROTECT` ioctl to satisfy the fault, and the faulting thread is resumed.

The mechanism is the foundation for live migration of VMs (QEMU post-copy), distributed shared-memory (CRIU/LiveMigration), garbage-collector barriers in managed runtimes, and security sandbox tooling. It also has a long history of being abused for use-after-free exploitation (race-window-extender against in-kernel objects), so modern kernels gate it behind `vm.unprivileged_userfaultfd` and the `UFFD_USER_MODE_ONLY` flag. Critical for: post-copy live-migration, CRIU, GC barriers, hardened sandbox isolation.

## Signature

```c
int userfaultfd(int flags);
```

```c
#define O_CLOEXEC             02000000
#define O_NONBLOCK            00004000
#define UFFD_USER_MODE_ONLY    0x1
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `flags` | `int` | in | Bitmask of `O_CLOEXEC`, `O_NONBLOCK`, `UFFD_USER_MODE_ONLY`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | New file descriptor. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | Reserved bits in `flags`. |
| `EPERM` | `vm.unprivileged_userfaultfd = 0` and caller lacks `CAP_SYS_PTRACE`; or `UFFD_USER_MODE_ONLY` not set when policy mandates it. |
| `ENOMEM` | Cannot allocate `struct userfaultfd_ctx`. |
| `ENFILE` / `EMFILE` | System or per-process fd table exhaustion. |

## ABI surface

```text
__NR_userfaultfd  (x86_64)  = 323
__NR_userfaultfd  (arm64)   = 282
__NR_userfaultfd  (riscv)   = 282
__NR_userfaultfd  (i386)    = 374

/* flags is `int`; only O_CLOEXEC, O_NONBLOCK, UFFD_USER_MODE_ONLY are
   defined. Reserved bits MUST be zero. */
/* The returned fd is operated on via read(2), poll(2), and ioctl(2). */
```

## Compatibility contract

REQ-1: Syscall number is **323** on x86_64. ABI-stable.

REQ-2: Flag validation:
- `(flags & ~(O_CLOEXEC | O_NONBLOCK | UFFD_USER_MODE_ONLY)) != 0` ⟹ `-EINVAL`.

REQ-3: Privilege gating per `vm.unprivileged_userfaultfd`:
- 0: caller MUST hold `CAP_SYS_PTRACE`; otherwise `-EPERM`.
- 1: any caller may invoke; default on recent distros.
- 2 (grsec-hardened): caller MUST hold `CAP_SYS_PTRACE` AND MUST set `UFFD_USER_MODE_ONLY`; otherwise `-EPERM`.

REQ-4: `UFFD_USER_MODE_ONLY` semantics: the resulting context handles faults only when the faulting access originates from CPL3 (user mode). Kernel-mode accesses (e.g. via `copy_from_user`) bypass the uffd path and either succeed or fail with `-EFAULT` as if no uffd were registered. This closes the classic "race-window-extender" exploit pattern where a malicious user used uffd to stall a kernel `copy_from_user` mid-instruction.

REQ-5: The context is initialized with `UFFD_API_VERSION = 0xAA`. The two-step handshake — `UFFDIO_API` after construction — locks in the supported features (`UFFD_FEATURE_PAGEFAULT_FLAG_WP`, `UFFD_FEATURE_MISSING_HUGETLBFS`, `UFFD_FEATURE_MINOR_HUGETLBFS`, `UFFD_FEATURE_EXACT_ADDRESS`, ...).

REQ-6: The fd implements `read(2)` (delivering `struct uffd_msg`), `poll(2)` (POLLIN when messages queued), `ioctl(2)` (UFFDIO_API, REGISTER, UNREGISTER, WAKE, COPY, ZEROPAGE, CONTINUE, WRITEPROTECT, MOVE).

REQ-7: Per-process accounting: `userfaultfd` creation does NOT charge RLIMIT_MEMLOCK by itself; subsequent `UFFDIO_COPY` allocates pages that ARE accounted to the calling process's `mm->locked_vm` / `RLIMIT_MEMLOCK` depending on the registered VMA flags.

REQ-8: Forward-compat: `UFFDIO_API`-time feature negotiation; new features advertised in `uffdio_api.features` field. Unknown feature bits set by user ⟹ `UFFDIO_API` returns `-EINVAL` and the context is left in a state where only `UFFDIO_API` can be called.

REQ-9: O_CLOEXEC default behavior: per the syscall(2) man page convention, callers SHOULD set `O_CLOEXEC` to avoid leaking the fd across `exec(2)`. The kernel does not force this.

REQ-10: The fd is owned by the calling task; closing it implicitly issues `UFFDIO_UNREGISTER` over every registered range and drains any in-flight faults.

REQ-11: A given mm can have multiple uffd contexts, registered against disjoint ranges. Overlapping registration returns `-EBUSY` from `UFFDIO_REGISTER`.

REQ-12: When a uffd context is destroyed while threads are blocked in `handle_userfault`, those threads receive the equivalent of a fault failure and the kernel returns `-SIGBUS` to user space.

## Acceptance Criteria

- [ ] AC-1: `userfaultfd(O_CLOEXEC)` returns a positive fd; `close()` releases it.
- [ ] AC-2: `userfaultfd(0)` succeeds when `vm.unprivileged_userfaultfd = 1`.
- [ ] AC-3: `userfaultfd(0)` returns `-EPERM` when `vm.unprivileged_userfaultfd = 0` and caller lacks `CAP_SYS_PTRACE`.
- [ ] AC-4: `userfaultfd(UFFD_USER_MODE_ONLY)`: subsequent `UFFDIO_REGISTER` succeeds; in-kernel `copy_from_user` to registered range proceeds without invoking uffd handler.
- [ ] AC-5: `userfaultfd(0x10000000)` returns `-EINVAL` (unknown flag bit).
- [ ] AC-6: `vm.unprivileged_userfaultfd = 2` and `UFFD_USER_MODE_ONLY` not set: `-EPERM`.
- [ ] AC-7: After `UFFDIO_REGISTER`, faulting access into the range triggers a `uffd_msg` readable via `read()`.
- [ ] AC-8: Multiple uffd contexts on disjoint ranges: each receives only its range's faults.
- [ ] AC-9: Closing the uffd fd while a peer thread blocks on `handle_userfault`: peer thread resumes with `SIGBUS`.
- [ ] AC-10: `UFFDIO_API` with unknown feature bit set: `-EINVAL`; context stuck at API state.

## Architecture

```rust
#[syscall(nr = 323, abi = "sysv")]
pub fn sys_userfaultfd(flags: i32) -> isize {
    Userfaultfd::do_userfaultfd(flags)
}
```

`Userfaultfd::do_userfaultfd(flags) -> isize`:
1. const VALID: i32 = O_CLOEXEC | O_NONBLOCK | UFFD_USER_MODE_ONLY;
2. if (flags & !VALID) != 0 { return -EINVAL; }
3. /* Privilege gate */
4. match sysctl_unprivileged_userfaultfd {
5.    0 => if !capable(CAP_SYS_PTRACE) { return -EPERM; },
6.    1 => {},
7.    _ => {
8.        if !capable(CAP_SYS_PTRACE) { return -EPERM; }
9.        if (flags & UFFD_USER_MODE_ONLY) == 0 { return -EPERM; }
10.   }
11. }
12. /* Allocate context */
13. let ctx = Box::new(UserfaultfdCtx {
14.    refcount: AtomicI32::new(1),
15.    mm: current().mm().clone(),
16.    state: UffdState::PreApi,
17.    flags,
18.    fault_wq: WaitQueue::new(),
19.    event_wq: WaitQueue::new(),
20.    fault_pending_list: RwLock::new(Vec::new()),
21.    features: 0,
22.    api_set: false,
23. });
24. /* Anon inode */
25. let mut fd_flags = 0;
26. if (flags & O_CLOEXEC) != 0 { fd_flags |= FD_CLOEXEC; }
27. let mut file_flags = O_RDWR;
28. if (flags & O_NONBLOCK) != 0 { file_flags |= O_NONBLOCK; }
29. let fd = anon_inode_getfd("[userfaultfd]", &userfaultfd_fops, ctx, file_flags)?;
30. fd as isize

`Userfaultfd::handle_userfault(vmf) -> VmFault`:
1. let ctx = vmf.vma.uffd_ctx.upgrade().ok_or(VM_FAULT_SIGBUS)?;
2. if (ctx.flags & UFFD_USER_MODE_ONLY) != 0 && !vmf.user_mode() {
3.    return VM_FAULT_FALLBACK; /* let kernel default handler run */
4. }
5. let msg = UffdMsg::from_fault(vmf);
6. ctx.fault_pending_list.write().push(msg.clone());
7. ctx.fault_wq.wake_all();
8. /* Block until handler resolves */
9. wait_event_killable(ctx.event_wq, msg.resolved.load());
10. if msg.was_signal() { return VM_FAULT_SIGBUS; }
11. VM_FAULT_RETRY

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_validation_total` | INVARIANT | unknown flag bit ⟹ EINVAL. |
| `sysctl_priv_obeyed` | INVARIANT | sysctl=0 ⟹ requires CAP_SYS_PTRACE. |
| `user_mode_only_enforced` | INVARIANT | UFFD_USER_MODE_ONLY ⟹ kernel-mode faults fall through. |
| `ctx_refcount_balanced` | INVARIANT | create ⟹ refcount=1; close ⟹ 0. |
| `pending_drain_on_close` | INVARIANT | close drains fault_pending_list (SIGBUS to waiters). |
| `api_state_machine` | INVARIANT | UFFDIO_API must precede other ioctls. |

### Layer 2: TLA+

`fs/userfaultfd.tla`:
- States: per-arg-validate, per-priv-check, per-ctx-alloc, per-anon-inode, per-fault-handle, per-resolve.
- Properties:
  - `safety_no_priv_without_cap` — sysctl=0 path requires CAP_SYS_PTRACE.
  - `safety_user_mode_only` — UFFD_USER_MODE_ONLY closes kernel-fault race window.
  - `safety_no_overlapping_register` — UFFDIO_REGISTER on overlapping range ⟹ EBUSY.
  - `liveness_create_terminates` — userfaultfd() returns.
  - `liveness_fault_resolves_or_sigbus` — every faulting thread eventually resumes or SIGBUS.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_userfaultfd` post: returns fd ≥ 0 or negative errno | `Userfaultfd::do_userfaultfd` |
| `do_userfaultfd` post: capable(CAP_SYS_PTRACE) checked when sysctl=0 | `Userfaultfd::do_userfaultfd` |
| `do_userfaultfd` post: UFFD_USER_MODE_ONLY enforced when sysctl=2 | `Userfaultfd::do_userfaultfd` |
| `handle_userfault` post: kernel-mode fault + USER_MODE_ONLY ⟹ FALLBACK | `Userfaultfd::handle_userfault` |

### Layer 4: Verus / Creusot functional

Per-`userfaultfd(2)` man-page + selftests `tools/testing/selftests/mm/uffd-*` semantic equivalence. QEMU post-copy migration smoke test passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`userfaultfd(2)` reinforcement:

- **Per-sysctl unprivileged-uffd gate** — defense against per-unprivileged-fault-race exploits.
- **Per-UFFD_USER_MODE_ONLY** — defense against per-kernel-mode-fault race window.
- **Per-ctx refcount strict get/put** — defense against per-ctx UAF.
- **Per-state-machine API-first** — defense against per-protocol confusion.
- **Per-feature-negotiation strict** — defense against per-extension smuggling.
- **Per-RLIMIT_MEMLOCK applied to UFFDIO_COPY** — defense against per-uffd page allocation DoS.
- **Per-mm scope** — defense against per-cross-mm fault leakage.
- **Per-close drains waiters with SIGBUS** — defense against per-stuck-thread.

## Grsecurity / PaX surface

- **PaX UDEREF on UFFDIO ioctls** — defense against per-user-addr kernel-deref on COPY/ZEROPAGE source pointers; SMAP forced.
- **GRKERNSEC_HARDEN_USERFAULTFD** — sets `vm.unprivileged_userfaultfd = 2` and enforces it: requires `CAP_SYS_PTRACE` AND `UFFD_USER_MODE_ONLY`. Even with CAP_SYS_PTRACE, refusing `UFFD_USER_MODE_ONLY` returns `-EPERM`. This is the configuration that closes the race-window-extender exploit class.
- **RLIMIT_MEMLOCK on userfaultfd allocations** — every UFFDIO_COPY/ZEROPAGE/CONTINUE charges the resulting page against the calling task's locked_vm so that an attacker cannot pin unbounded memory via uffd.
- **PaX KERNEXEC on handle_userfault dispatch** — function-pointer indirection trampolined.
- **PAX_REFCOUNT on userfaultfd_ctx.refcount** — saturating overflow; defense against per-refcount-overflow UAF.
- **GRKERNSEC_AUDIT_IPC** — every `userfaultfd(2)` invocation that succeeds emits an audit record (task, uid, flags) so SIEM can detect post-exploit setup.
- **GRKERNSEC_HIDESYM on uffd_msg.event_data** — kernel addresses are stripped before being placed in any user-visible message field (no leak of vma->vm_start unless the user already mapped the range).
- **PAX_USERCOPY_HARDEN on read(2) of uffd_msg** — fixed-size message copy via whitelisted slab.
- **GRKERNSEC_NO_USERFAULTFD** — Kconfig knob to compile out the entire subsystem; CAP_SYS_PTRACE has no effect because the syscall returns `-ENOSYS`. Recommended for hardened minimal kernels.
- **Per-uffd context per-task ownership** — defense against per-cross-task uffd-msg leakage; receiving a uffd fd across `SCM_RIGHTS` re-checks CAP_SYS_PTRACE and UFFD_USER_MODE_ONLY at first ioctl.
- **PaX MPROTECT interaction** — UFFDIO_WRITEPROTECT cannot upgrade PROT_NONE to PROT_WRITE; it can only modify W status of pages already mapped.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- UFFDIO ioctls semantics (covered in Tier-3 `fs/userfaultfd.md`).
- Post-copy live migration protocol (out-of-tree QEMU).
- handle_userfault interaction with hugetlb/shmem (covered in Tier-3 mm docs).
- Implementation code.
