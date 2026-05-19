# Tier-5 syscall: bpf(2) — syscall 321

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/bpf/syscall.c (SYSCALL_DEFINE3(bpf), __sys_bpf, bpf_prog_load, bpf_map_create, ...)
  - kernel/bpf/verifier.c (bpf_check, bpf_verifier_env)
  - kernel/bpf/core.c (bpf_prog_alloc, bpf_prog_run, JIT dispatch)
  - kernel/bpf/token.c (bpf_token_create, capability gating)
  - include/uapi/linux/bpf.h (union bpf_attr, enum bpf_cmd, BPF_PROG_TYPE_*, BPF_MAP_TYPE_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (321  common  bpf)
-->

## Summary

`bpf(2)` is the unified multiplexed entry point for the Berkeley Packet Filter (BPF) subsystem. A single integer `cmd` selects one of ~40 commands (program load/verify, map create/lookup/update/delete, link create, iterator open, BTF load, token create, prog/map/link pin/get, etc.), and a single tagged-union pointer `attr` of caller-declared `size` carries the per-command payload. The kernel performs `copy_from_user` of the first `size` bytes, zero-extends/truncates to its internal `union bpf_attr`, then dispatches.

bpf(2) is the gatekeeper for an in-kernel, verifier-checked virtual machine. Critical for: cilium / observability, sched_ext, XDP, kprobes/uprobes/tracepoints, LSM-BPF, cgroup BPF, struct_ops BPF, and the new BPF-token capability-split model.

## Signature

```c
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

```c
enum bpf_cmd {
    BPF_MAP_CREATE           = 0,
    BPF_MAP_LOOKUP_ELEM      = 1,
    BPF_MAP_UPDATE_ELEM      = 2,
    BPF_MAP_DELETE_ELEM      = 3,
    BPF_MAP_GET_NEXT_KEY     = 4,
    BPF_PROG_LOAD            = 5,
    BPF_OBJ_PIN              = 6,
    BPF_OBJ_GET              = 7,
    BPF_PROG_ATTACH          = 8,
    BPF_PROG_DETACH          = 9,
    BPF_PROG_TEST_RUN        = 10,
    BPF_PROG_GET_NEXT_ID     = 11,
    BPF_MAP_GET_NEXT_ID      = 12,
    BPF_PROG_GET_FD_BY_ID    = 13,
    BPF_MAP_GET_FD_BY_ID     = 14,
    BPF_OBJ_GET_INFO_BY_FD   = 15,
    BPF_PROG_QUERY           = 16,
    BPF_RAW_TRACEPOINT_OPEN  = 17,
    BPF_BTF_LOAD             = 18,
    BPF_BTF_GET_FD_BY_ID     = 19,
    BPF_TASK_FD_QUERY        = 20,
    BPF_MAP_LOOKUP_AND_DELETE_ELEM = 21,
    BPF_MAP_FREEZE           = 22,
    BPF_BTF_GET_NEXT_ID      = 23,
    BPF_MAP_LOOKUP_BATCH     = 24,
    BPF_MAP_LOOKUP_AND_DELETE_BATCH = 25,
    BPF_MAP_UPDATE_BATCH     = 26,
    BPF_MAP_DELETE_BATCH     = 27,
    BPF_LINK_CREATE          = 28,
    BPF_LINK_UPDATE          = 29,
    BPF_LINK_GET_FD_BY_ID    = 30,
    BPF_LINK_GET_NEXT_ID     = 31,
    BPF_ENABLE_STATS         = 32,
    BPF_ITER_CREATE          = 33,
    BPF_LINK_DETACH          = 34,
    BPF_PROG_BIND_MAP        = 35,
    BPF_TOKEN_CREATE         = 36,
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `cmd` | `int` | in | One of `BPF_*`; selects per-command dispatch. |
| `attr` | `union bpf_attr *` | in/out | Tagged union; per-command interpretation. May be `NULL` only for self-test paths. |
| `size` | `unsigned int` | in | Caller-declared sizeof(*attr); kernel uses `min(size, sizeof(union bpf_attr))` for the copy and validates trailing bytes are zero. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Success; usually a new file descriptor (for `BPF_PROG_LOAD`, `BPF_MAP_CREATE`, `BPF_OBJ_GET`, `BPF_LINK_CREATE`, `BPF_TOKEN_CREATE`) or an info-bytes-written count. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Missing `CAP_BPF` (or `CAP_SYS_ADMIN` for legacy compat) for the requested cmd / prog-type / map-type. |
| `EINVAL` | Bad `cmd`, bad `size`, non-zero reserved fields, invalid prog/map/btf params, verifier rejection. |
| `EFAULT` | `attr` user pointer faults during copy_from_user / copy_to_user. |
| `ENOMEM` | Out of memory (prog text, map alloc, btf load, verifier scratch). |
| `EACCES` | Verifier rejected program (insn count, restricted helper, OOB access, unbounded loop). |
| `E2BIG` | `size` larger than kernel-supported attr layout AND the kernel-unknown trailing region is non-zero. |
| `ENOENT` | Lookup-by-id / `BPF_OBJ_GET` path not present. |
| `EEXIST` | Pin path already exists. |
| `ENOSPC` | Map full and not BPF_F_NO_PREALLOC. |
| `ENODEV` | `BPF_LINK_CREATE` target device or cgroup gone. |
| `EBUSY` | Map frozen, link non-detachable, prog active. |

## ABI surface

```text
__NR_bpf  (x86_64)  = 321
__NR_bpf  (arm64)   = 280
__NR_bpf  (riscv)   = 280
__NR_bpf  (i386)    = 357

/* attr is union; kernel-known size is sizeof(union bpf_attr). */
/* Forward-compat: caller may pass size > kernel-size if the trailing
   bytes are zero (EINVAL otherwise). */
/* Backward-compat: caller may pass size < kernel-size; missing tail
   is treated as zero. */
```

## Compatibility contract

REQ-1: Syscall number is **321** on x86_64. ABI-stable.

REQ-2: `cmd` selects per-dispatch table entry; unknown cmd -> `-EINVAL`.

REQ-3: `size` validation: if `size > sizeof(union bpf_attr)`, kernel verifies bytes `[sizeof(bpf_attr), size)` are zero (forward-compat extension probe), else `-E2BIG`.

REQ-4: `copy_from_user(&kattr, attr, min(size, sizeof(union bpf_attr)))`; trailing kernel-side fields zeroed.

REQ-5: Capability gating per-cmd:
- `BPF_PROG_LOAD` of tracing/lsm prog types: `CAP_BPF` + `CAP_PERFMON` (or `CAP_SYS_ADMIN` legacy).
- `BPF_PROG_LOAD` of network prog types: `CAP_BPF` + `CAP_NET_ADMIN`.
- `BPF_MAP_CREATE`: `CAP_BPF`.
- `BPF_TOKEN_CREATE`: `CAP_SYS_ADMIN` in `init_user_ns` (delegating capability bundle to a non-init userns).
- Unprivileged BPF: disabled by default (`sysctl_unprivileged_bpf_disabled = 2` in hardened mode).

REQ-6: BPF token (cmd `BPF_TOKEN_CREATE`): produces an fd that, when passed via `bpf_attr.prog_token_fd` / `bpf_attr.map_token_fd`, delegates a chosen subset of cmds, prog types, map types and helper set to a child userns. Token cannot raise privilege beyond its issuer's caps.

REQ-7: Program load runs the verifier (`bpf_check`) which:
- Bounds-checks every memory access.
- Tracks register types/ranges via abstract interpretation.
- Limits instructions (`BPF_COMPLEXITY_LIMIT_INSNS = 1,000,000`).
- Limits state-prune branches.
- Rejects unbounded loops (allows bounded `bpf_loop()` helper).
- Verifies helper-call signatures.

REQ-8: Maps are typed (HASH, ARRAY, PERCPU_*, LRU_*, RINGBUF, SOCKHASH, ...). Each cmd validates `map_type` against caller caps and against per-map operation table.

REQ-9: `BPF_OBJ_PIN` and `BPF_OBJ_GET` use bpffs (fstype `bpf`). Path is namespace-relative; bind-mounts permitted.

REQ-10: `BPF_PROG_TEST_RUN` invokes a packet/context against a loaded prog without attaching it; intended for selftests and userspace fuzzers.

REQ-11: `BPF_LINK_CREATE` produces a refcounted attachment fd; closing the fd auto-detaches. `BPF_LINK_UPDATE` atomically swaps the attached prog.

REQ-12: Per-`sysctl_kernel.unprivileged_bpf_disabled`:
- 0: unprivileged BPF allowed.
- 1: disabled, runtime-toggleable (root only).
- 2: disabled, locked until reboot (hardened mode).

REQ-13: Per-`sysctl_kernel.bpf_stats_enabled`: optional run-time accounting of prog runtimes; off by default.

REQ-14: Per-JIT: `bpf_jit_enable`, `bpf_jit_harden`, `bpf_jit_kallsyms`. `bpf_jit_harden=2` blinds constants (defense vs JIT spray).

REQ-15: All `union bpf_attr` reserved fields MUST be zero on entry; the kernel rejects non-zero with `-EINVAL` to keep extension headroom.

## Acceptance Criteria

- [ ] AC-1: `bpf(BPF_MAP_CREATE, attr, sizeof attr)` with `BPF_MAP_TYPE_HASH` returns a positive fd; close releases the map.
- [ ] AC-2: `bpf(BPF_PROG_LOAD, ...)` of a valid socket-filter returns a positive fd.
- [ ] AC-3: `bpf(BPF_PROG_LOAD, ...)` of a program with OOB access returns `-EACCES` and writes verifier log if requested.
- [ ] AC-4: `bpf(BPF_MAP_LOOKUP_ELEM, ...)` on a non-existent key returns `-ENOENT`.
- [ ] AC-5: `cmd = 9999` returns `-EINVAL`.
- [ ] AC-6: `size = sizeof(union bpf_attr) + 4` with non-zero tail returns `-E2BIG`.
- [ ] AC-7: `size = sizeof(union bpf_attr) + 4` with zero tail succeeds.
- [ ] AC-8: Caller without `CAP_BPF` (and unprivileged_bpf_disabled=2): `BPF_PROG_LOAD` returns `-EPERM`.
- [ ] AC-9: `BPF_TOKEN_CREATE` from non-init userns without delegation: `-EPERM`.
- [ ] AC-10: `BPF_LINK_CREATE` then close-fd: auto-detach observed.
- [ ] AC-11: Verifier complexity overflow: `-EACCES` with log "BPF program is too large".
- [ ] AC-12: `bpf_jit_harden=2`: constants in JITed image are not directly observable (blinded XOR).

## Architecture

```rust
#[syscall(nr = 321, abi = "sysv")]
pub fn sys_bpf(cmd: i32, attr: UserPtr<BpfAttr>, size: u32) -> isize {
    Bpf::do_bpf(cmd, attr, size)
}
```

`Bpf::do_bpf(cmd, attr_ptr, size) -> isize`:
1. let cmd = BpfCmd::try_from(cmd).map_err(|_| EINVAL)?;
2. Bpf::check_capability_for_cmd(cmd, &current_creds())?;     // EPERM
3. let kattr = Bpf::copy_attr_from_user(attr_ptr, size)?;     // EFAULT / E2BIG
4. match cmd {
5.   BpfCmd::MapCreate         => Bpf::map_create(&kattr),
6.   BpfCmd::MapLookupElem     => Bpf::map_lookup(&kattr),
7.   BpfCmd::MapUpdateElem     => Bpf::map_update(&kattr),
8.   BpfCmd::MapDeleteElem     => Bpf::map_delete(&kattr),
9.   BpfCmd::ProgLoad          => Bpf::prog_load(&kattr),
10.  BpfCmd::ObjPin            => Bpf::obj_pin(&kattr),
11.  BpfCmd::ObjGet            => Bpf::obj_get(&kattr),
12.  BpfCmd::ProgAttach        => Bpf::prog_attach(&kattr),
13.  BpfCmd::LinkCreate        => Bpf::link_create(&kattr),
14.  BpfCmd::LinkUpdate        => Bpf::link_update(&kattr),
15.  BpfCmd::TokenCreate       => Bpf::token_create(&kattr),
16.  BpfCmd::BtfLoad           => Bpf::btf_load(&kattr),
17.  /* ... ~40 commands ... */
18.  _                         => Err(EINVAL),
19. }

`Bpf::copy_attr_from_user(uptr, size) -> Result<BpfAttr>`:
1. const K: usize = size_of::<BpfAttr>();
2. let n = min(size as usize, K);
3. let mut kattr = BpfAttr::zeroed();
4. unsafe { uptr.copy_in_partial(&mut kattr, n)?; }
5. /* Probe forward-compat trailing tail. */
6. if (size as usize) > K {
7.   let tail = (size as usize) - K;
8.   let mut probe = [0u8; 64];
9.   for off in step_by(tail, 64) {
10.    let n = min(64, tail - off);
11.    uptr.add(K + off).copy_in_partial(&mut probe[..n])?;
12.    if probe[..n].iter().any(|b| *b != 0) { return Err(E2BIG); }
13.  }
14. }
15. Ok(kattr)
```

`Bpf::prog_load(attr) -> isize`:
1. Bpf::validate_prog_type_cap(attr.prog_type, attr.prog_token_fd)?;
2. let prog = BpfProg::alloc(attr.insn_cnt, attr.prog_type)?;
3. prog.copy_insns_from_user(attr.insns, attr.insn_cnt * BPF_INSN_SZ)?;
4. bpf_check(&mut prog, &verifier_opts(attr))?;               // EACCES on reject
5. if bpf_jit_enable { bpf_jit_compile(&mut prog)?; }
6. let fd = anon_inode_getfd("bpf-prog", &bpf_prog_fops, &prog)?;
7. Ok(fd as isize)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cmd_dispatch_total` | INVARIANT | every BpfCmd variant has exactly one dispatch arm. |
| `attr_copy_bounded` | INVARIANT | copy never exceeds `min(size, sizeof BpfAttr)`. |
| `tail_zero_check` | INVARIANT | size > K & non-zero tail ⟹ E2BIG. |
| `cap_check_before_work` | INVARIANT | capability denied ⟹ no allocation, no FD created. |
| `verifier_must_run` | INVARIANT | prog_load: bpf_check called before exposing fd. |
| `token_no_priv_raise` | INVARIANT | token capability subset ⊆ issuer caps. |

### Layer 2: TLA+

`kernel/bpf-syscall.tla`:
- States: per-cmd dispatch, per-attr copy, per-cap-check, per-verifier-decision, per-fd-install.
- Properties:
  - `safety_no_fd_without_cap` — never install fd if cap denied.
  - `safety_verifier_pre_install` — fd install strictly after verifier accepts.
  - `safety_token_scope` — token-granted caps ⊆ issuer caps.
  - `safety_attr_tail_zero` — tail validation invariant.
  - `liveness_dispatch_terminates` — every bpf() returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_bpf` post: returns -EINVAL for unknown cmd | `Bpf::do_bpf` |
| `copy_attr_from_user` post: kattr matches user attr (zero-padded) | `Bpf::copy_attr_from_user` |
| `prog_load` post: success ⟹ verifier accepted | `Bpf::prog_load` |
| `token_create` post: token.caps ⊆ current.caps | `Bpf::token_create` |

### Layer 4: Verus / Creusot functional

Per-`bpf(2)` man-page + libbpf semantic equivalence. Selftests: `tools/testing/selftests/bpf/` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`bpf(2)` reinforcement:

- **Per-cmd capability gate** — defense against per-priv-escalation via unguarded subcommand.
- **Per-attr tail zero-probe** — defense against per-extension-field smuggling.
- **Per-verifier mandatory pre-install** — defense against per-OOB-load.
- **Per-JIT constant blinding (bpf_jit_harden=2)** — defense against per-JIT-spray.
- **Per-`unprivileged_bpf_disabled=2`** — defense against per-unpriv-prog exploit class.
- **Per-token capability subset** — defense against per-token-priv-raise.
- **Per-prog complexity limit** — defense against per-verifier-DoS.
- **Per-map RLIMIT_MEMLOCK accounting** — defense against per-map-mem-DoS.

## Grsecurity / PaX surface

- **PaX UDEREF on `attr` copy_from_user** — defense against per-attr-pointer kernel-deref bug; SMAP forced.
- **GRKERNSEC_BPF_HARDEN** — global toggle to require `CAP_BPF` for every bpf cmd, disable unprivileged BPF entirely, and force `bpf_jit_harden=2`. Even root-in-userns is rejected unless the userns is init.
- **CAP_BPF capability split** — bpf prog-load/map-create now requires explicit `CAP_BPF` (not the omnibus `CAP_SYS_ADMIN`); tracing additionally requires `CAP_PERFMON`; networking additionally `CAP_NET_ADMIN`.
- **BPF token + CAP_SYS_ADMIN issuer in init_userns** — the only legitimate way to delegate BPF to a non-init userns; grsec enforces that the token cannot widen the issuer's caps and that any token whose issuer credentials change is invalidated.
- **PAX_REFCOUNT on prog/map/link refcounts** — defense against per-refcount-overflow UAF.
- **GRKERNSEC_HIDESYM on JIT image kallsyms** — JITed prog symbols not exposed via `/proc/kallsyms`; `bpf_jit_kallsyms` forced 0.
- **PaX KERNEXEC on JIT pages** — defense against per-W^X violation; JIT pages mapped RX-only post-emit.
- **Verifier-log copy under PaX UDEREF** — verifier reject log copy_to_user uses SMAP guard.
- **GRKERNSEC_PROC_GETPID consistency** — bpffs pin paths visible via /proc only to the issuing user.
- **PAX_USERCOPY_HARDEN on prog insn copy_from_user** — bounded copy uses whitelisted slab.
- **Per-token revocation on creds change** — defense against per-stale-token use.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-prog-type verifier semantics (covered in Tier-3 `kernel/bpf/verifier.md`).
- Per-map-type slow paths (covered in Tier-3 `kernel/bpf/maps.md`).
- JIT codegen per-arch (covered in arch Tier-3 docs).
- Implementation code.
