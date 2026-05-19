---
title: "Tier-5 syscall: getcpu(2) — syscall 309"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getcpu(2)` returns the current CPU number and NUMA node number on which the calling thread is running. It is primarily a vDSO-accelerated path: on x86_64 the vDSO `__vdso_getcpu` reads the CPU+node from a per-CPU TSC_AUX register populated by the kernel at context switch (via `RDTSCP`/`RDPID`), avoiding the syscall altogether. The syscall remains as the canonical entry point and the fallback when the vDSO is unavailable.

The third `tcache` argument is historic and now unused; pre-vDSO versions used it as a per-thread cache to amortize the syscall cost across calls. It is reserved and must be `NULL` in new code.

Critical for: NUMA-aware allocators, per-CPU lock-free data structures, libnuma, GLIBC `sched_getcpu(3)` (vDSO-first wrapper), profiling.

### Acceptance Criteria

- [ ] AC-1: `getcpu(&c, &n, NULL)` returns 0; `c < nr_cpu_ids`; `n < MAX_NUMNODES`.
- [ ] AC-2: After `sched_setaffinity` to CPU 3, `getcpu` returns c == 3.
- [ ] AC-3: `getcpu(NULL, NULL, NULL)` returns 0 (no-op valid).
- [ ] AC-4: `cpu = faulting_addr` → `-EFAULT`.
- [ ] AC-5: `node = faulting_addr` → `-EFAULT`.
- [ ] AC-6: `tcache` non-NULL but unmapped: ignored, returns 0.
- [ ] AC-7: On UMA system, returned `node` == 0.
- [ ] AC-8: vDSO path matches syscall path for the same CPU.
- [ ] AC-9: Repeated calls without migration return same (cpu, node).

### Architecture

```rust
#[syscall(nr = 309, abi = "sysv")]
pub fn sys_getcpu(
    cpu: UserPtrMut<u32>,
    node: UserPtrMut<u32>,
    tcache: UserPtrMut<GetcpuCache>,
) -> isize {
    Sched::do_getcpu(cpu, node, tcache)
}
```

`Sched::do_getcpu(cpu, node, _tcache) -> isize`:
1. /* Pin briefly to read SMP id consistently */
2. preempt_disable();
3. let c = raw_smp_processor_id() as u32;
4. let n = cpu_to_node(c) as u32;
5. preempt_enable();
6. if !cpu.is_null() { cpu.copy_out(&c)?; }            // EFAULT
7. if !node.is_null() { node.copy_out(&n)?; }          // EFAULT
8. /* tcache deliberately ignored */
9. 0

`Vdso::vdso_getcpu(cpu, node, _tcache) -> i64` (userspace-side; lives in vDSO image):
1. /* Read TSC_AUX via RDPID (or RDTSCP) */
2. let aux: u32 = unsafe { rdpid() };
3. let c = aux & 0xfff;
4. let n = aux >> 12;
5. if !cpu.is_null() { *cpu = c; }
6. if !node.is_null() { *node = n; }
7. 0

`Vdso::write_cpu_aux_on_switch(cpu, node)` (kernel-side, per-context-switch):
1. let aux = ((node & 0xfffff) << 12) | (cpu & 0xfff);
2. wrmsr(MSR_TSC_AUX, aux);

### Out of Scope

- `sched_setaffinity(2)` / `sched_getaffinity(2)` (covered in their own Tier-5 docs).
- `set_mempolicy(2)` / `mbind(2)` NUMA mempolicy (covered separately).
- vDSO infrastructure (`arch/x86/entry/vdso/*`) — covered under arch Tier-3 vDSO doc.
- CPU hotplug machinery (covered separately).
- libnuma / `numa_node_of_cpu(3)` userspace surface.
- Implementation code.

### signature

```c
int getcpu(unsigned *cpu, unsigned *node, struct getcpu_cache *tcache);
```

```c
struct getcpu_cache {
    unsigned long blob[128 / sizeof(long)];   /* opaque; deprecated */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `cpu` | `unsigned *` | out (optional) | If non-NULL, kernel writes current logical CPU id. |
| `node` | `unsigned *` | out (optional) | If non-NULL, kernel writes NUMA node id of current CPU. |
| `tcache` | `struct getcpu_cache *` | in/out (deprecated) | Must be NULL on Linux >= 2.6.24; ignored if set. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; requested fields written. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `cpu` or `node` user pointer faults during copy_to_user. |

### abi surface

```text
__NR_getcpu (x86_64) = 309
__NR_getcpu (arm64)  = 168
__NR_getcpu (riscv)  = 168
__NR_getcpu (i386)   = 318

/* vDSO symbol: __vdso_getcpu (x86_64). vDSO contract:
   - cpu/node packed into TSC_AUX MSR by the kernel on context switch.
   - userspace RDPID or RDTSCP recovers both with no kernel transition.
   - layout: TSC_AUX = (node << 12) | (cpu & 0xfff). */
```

### compatibility contract

REQ-1: Syscall number is **309** on x86_64. ABI-stable since 2.6.19.

REQ-2: `cpu` semantics: logical CPU id in range `[0, nr_cpu_ids)`. NOT pinned — the kernel may migrate the thread after the call returns; the value is a snapshot.

REQ-3: `node` semantics: NUMA node id in `[0, MAX_NUMNODES)`; on UMA systems always 0.

REQ-4: `tcache` is ignored (post-2.6.24 vDSO obviated the cache). Per-glibc passes NULL.

REQ-5: vDSO entry: `__vdso_getcpu(cpu, node, tcache)` runs in userspace; reads `TSC_AUX` via `RDPID` (or `RDTSCP` fallback); writes `*cpu` and `*node`. Returns 0 on success. Never faults the kernel.

REQ-6: Kernel syscall path: invoked when vDSO unavailable (e.g. CONFIG_COMPAT_VDSO=n, or older glibc fallback). Reads `smp_processor_id()` while pinned; writes via `put_user`.

REQ-7: Per-`smp_processor_id()`: requires preemption disabled or pin via `migrate_disable`. The syscall path uses `raw_smp_processor_id` inside a brief preempt-disable region.

REQ-8: Per-32-bit compat: same syscall, same struct (cache size is in `unsigned long` so it changes; kernel still ignores).

REQ-9: Per-NUMA-policy: `getcpu` does not alter mempolicy; consumers (libnuma) use it for hints only.

REQ-10: Per-`set_mempolicy(2)` interaction: callers that bind to a node typically combine `getcpu` (current node) + `mbind(2)` to keep allocations local.

REQ-11: Per-CPU hotplug: between `getcpu` returning and the caller acting, the CPU may go offline. Caller must handle CPU disappearance.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpu_in_range` | INVARIANT | returned cpu < nr_cpu_ids. |
| `node_in_range` | INVARIANT | returned node < MAX_NUMNODES. |
| `null_pointers_ok` | INVARIANT | NULL cpu / node ⟹ no fault, no copy. |
| `tcache_ignored` | INVARIANT | tcache value (incl. unmapped) does not affect outcome. |
| `preempt_disable_balanced` | INVARIANT | preempt_disable paired with preempt_enable. |
| `vdso_msr_layout` | INVARIANT | TSC_AUX = (node << 12) | (cpu & 0xfff). |

### Layer 2: TLA+

`kernel/getcpu.tla`:
- States: per-call preempt-disable, read-smp-id, read-node, write-cpu, write-node, preempt-enable.
- Properties:
  - `safety_consistent_pair` — (cpu, node) snapshot is consistent (cpu's node matches `cpu_to_node(cpu)`).
  - `safety_null_skip` — NULL pointer skips copy without fault.
  - `liveness_terminates` — call always returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getcpu` post: returns 0 ∨ -EFAULT | `Sched::do_getcpu` |
| `write_cpu_aux_on_switch` post: TSC_AUX encodes (cpu, node) | `Vdso::write_cpu_aux_on_switch` |
| `vdso_getcpu` post: returned cpu/node match TSC_AUX layout | `Vdso::vdso_getcpu` |

### Layer 4: Verus / Creusot functional

Per-`getcpu(2)` man-page + glibc `sched_getcpu(3)` semantic equivalence. Selftests: `tools/testing/selftests/vDSO/vdso_test_getcpu.c` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getcpu(2)` reinforcement:

- **Per-`preempt_disable` brief region** — defense against per-cpu-id read race.
- **Per-`copy_to_user` all-or-nothing** — defense against per-partial-write info-leak.
- **Per-`tcache` ignored** — defense against per-pointer-deref bug via deprecated arg.
- **Per-vDSO TSC_AUX layout fixed** — defense against per-encoding drift.
- **Per-CPU-hotplug consistency** — defense against per-stale-cpu-id after offline.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `cpu` / `node` copy_to_user** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **getcpu side-channel mitigation via vDSO** — the vDSO path reads `TSC_AUX` (populated at context switch) and never enters the kernel; defense against per-syscall-timing side-channel that could fingerprint scheduler decisions. Grsec policy can additionally force the vDSO path by removing the syscall entry from the seccomp default-allow list for sandboxed processes (the vDSO still works because it never traps).
- **GRKERNSEC_HIDESYM on `TSC_AUX` encoding** — the (node << 12) | (cpu & 0xfff) layout uses only the documented bits; grsec asserts no kernel-pointer / canary / address bits leak via the upper 32 bits of TSC_AUX MSR even if userspace probes it directly with RDPID.
- **PAX_USERCOPY_HARDEN on cpu/node copy_to_user** — bounded 4-byte copies use whitelisted region.
- **CPU-pinning side-channel reduction** — repeated `getcpu` polling (used to infer scheduler decisions for other tenants on the host) is rate-limited per-UID under hardened policy; defense against per-co-tenant scheduling-correlation side-channel.
- **PaX KERNEXEC on `raw_smp_processor_id` inline** — defense against per-W^X violation.
- **vDSO image PaX MPROTECT** — vDSO mapped RX-only (no W); defense against per-vDSO patch injection.
- **GRKERNSEC_NO_GETPID-style** — node numbers returned in a userns can be remapped (host node N → container node 0) under hardened policy; defense against per-NUMA-topology leak across container boundaries.
- **Per-`tcache` NULL-only enforcement under hardened policy** — non-NULL `tcache` rejected with `-EINVAL` to remove the legacy attack surface; defense against per-deprecated-arg-pointer bug class.
- **PaX PAX_REFCOUNT-style consistency** — `cpu_to_node(cpu)` lookup uses array bounded by `nr_cpu_ids`; defense against per-OOB-read.

