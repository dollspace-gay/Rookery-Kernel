# Tier-5 special entry: uprobe — userspace-probe handler trampoline

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/events/uprobes.c (uprobe_register, handle_swbp, arch_uprobe_pre_xol, xol_take_insn_slot)
  - arch/x86/kernel/uprobes.c (arch_uprobe_analyze_insn, arch_uprobe_pre_xol)
  - include/linux/uprobes.h (struct uprobe, struct uprobe_consumer)
  - kernel/trace/trace_uprobe.c (trace_uprobe userspace ABI surface)
  - include/uapi/linux/perf_event.h (PERF_TYPE_BREAKPOINT, kprobe/uprobe pmu)
-->

## Summary

`uprobe` is **not a userspace syscall**; it is a kernel-internal entry point invoked when a userspace task hits a software-breakpoint instruction (`int3` on x86, `BRK #X` on arm64) that was previously inserted by the uprobe subsystem at a registered userspace address. On hit, the CPU traps into the kernel via the breakpoint exception vector; the handler dispatches into `handle_swbp`, which locates the per-address `struct uprobe`, runs every attached `uprobe_consumer.handler`, executes the displaced original instruction out-of-line in an XOL (execute-out-of-line) slot, and returns the task to userspace.

This document covers the **entry-trampoline contract** (how the kernel finds, dispatches, and resumes a uprobe-tripped task) rather than a numbered syscall. Userspace registers/unregisters uprobes via `perf_event_open(2)` (PMU = "uprobe"), `bpf(BPF_LINK_CREATE, attach_type = BPF_TRACE_UPROBE_MULTI, ...)`, or the tracefs `uprobe_events` text interface.

Critical for: BPF-based observability (Cilium tetragon, bcc / bpftrace, OpenTelemetry eBPF profiler), perf-stat user-PMU counters, USDT static tracepoints, language-runtime hot-patching frameworks, security agents that audit userspace API calls.

## Signature

(no syscall — entered via exception)

```c
/* Kernel entry path (NOT user-callable, shown for the contract): */
asmlinkage __visible void exc_int3(struct pt_regs *regs);
/* exc_int3 -> do_int3 -> handle_swbp(struct pt_regs *regs) */
```

```c
struct uprobe_consumer {
    int (*handler)(struct uprobe_consumer *, struct pt_regs *);
    int (*ret_handler)(struct uprobe_consumer *, unsigned long, struct pt_regs *);
    bool (*filter)(struct uprobe_consumer *, enum uprobe_filter_ctx,
                   struct mm_struct *);
    struct list_head        cons_node;
    __u64                   id;
};
```

## Parameters

(no user-facing parameters; the "input" is the trap state.)

| Implicit input | Source | Meaning |
|---|---|---|
| `regs->ip` (x86) / `regs->pc` (arm64) | trap frame | Address of the breakpoint instruction (= uprobe-registered offset within VMA). |
| `current->mm` | task | Address space whose mapping owns the breakpoint. |
| `regs->orig_ax` / signal pending | trap | Pre-empt-fast-path flags. |

## Return value

(no return — control flows back to userspace after XOL.)

## Errors

| Condition | Effect |
|---|---|
| No `struct uprobe` registered at hit address | `SIGTRAP` delivered to current task. |
| `uprobe_consumer.filter` rejects task | Handler skipped; XOL still executed. |
| XOL slot allocation fails | `SIGSEGV` (cannot safely continue). |
| Handler returns `UPROBE_HANDLER_REMOVE` | Per-task uprobe disarmed (single-step the original; remove BP from this mm). |

## ABI surface

```text
/* Not a syscall — but the userspace registration ABI is: */
perf_event_open(attr={ .type = PERF_TYPE_TRACEPOINT or uprobe-pmu,
                       .config = uprobe_id,
                       .ext_reg_idx = ... }, pid, cpu, group_fd, flags) -> int fd
/* OR */
write("/sys/kernel/tracing/uprobe_events",
      "p:GROUP/NAME /path/to/binary:0xOFFSET %ARGS") -> bytes
/* OR */
bpf(BPF_LINK_CREATE, .attach_type = BPF_TRACE_UPROBE_MULTI, ...) -> int fd

/* Per-arch trap instructions installed at registered offsets: */
x86_64   : 0xCC                  (int3, 1 byte)
arm64    : 0xD420A000            (brk #0x100, 4 bytes — UPROBE_BRK_UPROBE)
arm64-ret: 0xD420C000            (brk #0x600 — UPROBE_BRK_UPROBE_BP)
powerpc  : 0x7FE00008            (trap)
riscv    : 0x00100073            (ebreak)
```

## Compatibility contract

REQ-1: The uprobe trampoline is **never user-callable**; registration via `perf_event_open`, `bpf`, or tracefs is the only legitimate userspace surface. Attempting to issue the trap instruction in userspace without a matching registration yields `SIGTRAP` (or `SIGILL` if the arch policy treats unsolicited `brk` as illegal).

REQ-2: On hit, `handle_swbp` MUST:
  - Find `struct uprobe` via `find_uprobe(inode, offset)` from the trap address resolved through `current->mm` VMAs.
  - If not found: deliver `SIGTRAP` to `current` (matches "uprobe vanished mid-execute" race).
  - If found: invoke each registered consumer's `filter` (with `UPROBE_FILTER_PRE_HANDLER`), then `handler`, in registration order.

REQ-3: Consumer handlers run in **process context** (current is the tripped task), with preemption enabled and IRQs enabled. Sleeping in a uprobe handler is permitted but discouraged (latency tail).

REQ-4: After all handlers run, the kernel arranges XOL execution: a per-mm XOL VMA holds copies of the original (displaced) instruction; the task's IP is rewritten to the XOL slot; on resume, the CPU executes the displaced instruction; a second trap (or arch-specific single-step) returns control so the kernel can fixup PC and continue at the original-address-plus-instr-size.

REQ-5: Uretprobes are paired registration: when consumer registers a `ret_handler`, the kernel hijacks the return-target on entry — see `uretprobe.md`.

REQ-6: Registration is gated by capability:
  - tracefs path: `CAP_SYS_ADMIN`.
  - `perf_event_open` uprobe pmu: `CAP_PERFMON` + write-access to the target binary.
  - `bpf(BPF_LINK_CREATE, BPF_TRACE_UPROBE_MULTI)`: `CAP_BPF` + `CAP_PERFMON`; the BPF program must additionally pass the verifier.

REQ-7: A uprobe is identified by `(inode, offset)`. The kernel does NOT register per-VMA — every mm that maps the inode at any address picks up the breakpoint on next page-fault-in (`uprobe_mmap`, `uprobe_munmap` hooks).

REQ-8: The original instruction at `(inode, offset)` MUST be analyzable by `arch_uprobe_analyze_insn`. Disallowed instructions (RIP-relative without fixup, syscall on certain arches, control-flow with implicit fixup needs) are rejected at registration with `-ENOTSUPP`.

REQ-9: XOL VMA is per-mm; allocation is lazy on first uprobe hit in that mm; it is `PROT_READ | PROT_EXEC` (no write after install) and lives outside RANDMMAP.

REQ-10: Consumer `filter` controls per-mm enablement: returning false skips the handler (and may unsetregister the breakpoint from this mm if all consumers filter out).

REQ-11: BPF uprobe programs see a synthetic `pt_regs` snapshot; helpers like `bpf_probe_read_user`, `bpf_get_current_task`, and `bpf_probe_user_str` are allowed. Direct kernel-memory access is forbidden.

REQ-12: Uprobes do NOT bypass MNT_NOEXEC, file-perm checks, or LSM mmap hooks — registration requires read access to the file, and the resulting BP only fires when the file is executed.

REQ-13: Uprobes are SMP-safe: arming uses `set_swbp` which is single-byte atomic on x86 and uses `aarch64_insn_patch_text` on arm64 (stop-machine class patch on first BP in mm; per-mm BPs use copy-on-write of the page).

REQ-14: Trace events emitted by `trace_uprobe` are visible in `tracefs` and over perf ring buffers with `PERF_RECORD_SAMPLE`.

REQ-15: Uprobe count per-mm is bounded by `sysctl_perf_event_max_uprobe_pending_signals` analog; excessive arming returns `-EBUSY`.

## Acceptance Criteria

- [ ] AC-1: Registering a uprobe at `(/bin/ls, malloc_offset)` and running `ls`: handler fires once per `malloc` call.
- [ ] AC-2: Filter returning false on the second mm: handler skipped in that mm.
- [ ] AC-3: Registering a non-analyzable insn (e.g. an x86 PUSHF followed by FAR-call sequence not supported): `-ENOTSUPP`.
- [ ] AC-4: Trapping at an un-registered address (race: deregister-while-tripping): `SIGTRAP` delivered, no handler fires.
- [ ] AC-5: Handler `UPROBE_HANDLER_REMOVE` return removes breakpoint from this mm only; other mms continue tracing.
- [ ] AC-6: XOL slot exhaustion: graceful fallback to single-step in-place where the arch supports it.
- [ ] AC-7: Caller without `CAP_PERFMON` cannot open uprobe via perf_event_open.
- [ ] AC-8: BPF uprobe program: handler called with valid `pt_regs` view.
- [ ] AC-9: `tracefs/events/uprobes/<name>/enable=1` arms; new processes mapping the file pick up the BP.
- [ ] AC-10: Multi-consumer registration at the same offset: handlers fire in registration order.
- [ ] AC-11: Concurrent hits on different CPUs: no XOL-slot corruption; each task gets its own slot.
- [ ] AC-12: Uretprobe pair: entry handler + return handler invoked once each per call (see `uretprobe.md`).

## Architecture

```rust
/* Non-syscall — exception entry. */
#[exception(kind = "breakpoint")]
pub fn exc_int3(regs: &mut PtRegs) {
    if Uprobe::handle_swbp(regs).is_ok() { return; }
    /* Fall through to normal SIGTRAP delivery. */
    Signal::force_sig_info(SIGTRAP, regs);
}
```

`Uprobe::handle_swbp(regs) -> Result<()>`:
1. let addr = regs.bp_addr();                                   // arch-specific
2. let mm   = current_mm();
3. let (inode, offset) = mm.vma_at(addr).ok_or(NoUprobe)?.resolve_file_offset(addr);
4. let uprobe = Uprobe::find_uprobe(inode, offset).ok_or(NoUprobe)?;
5. let _rcu = rcu_read_lock();
6. let mut decision = HandlerDecision::Continue;
7. for cons in uprobe.consumers.iter() {
8.   if let Some(filter) = cons.filter {
9.     if !filter(cons, UPROBE_FILTER_PRE_HANDLER, mm) { continue; }
10.  }
11.  let r = (cons.handler)(cons, regs);
12.  if r == UPROBE_HANDLER_REMOVE { decision = HandlerDecision::RemoveFromMm; }
13. }
14. if uprobe.has_ret_handler() {
15.   Uretprobe::hijack_return(uprobe, regs)?;                  // see uretprobe.md
16. }
17. /* Execute displaced instruction out-of-line */
18. let slot = Uprobe::xol_take_slot(mm, uprobe)?;              // SIGSEGV on alloc fail
19. arch_uprobe_pre_xol(&uprobe.arch, regs, &slot);
20. /* Return from exception; on next trap (single-step done) arch_uprobe_post_xol
21.    will be invoked to fix up PC. */
22. if decision == HandlerDecision::RemoveFromMm {
23.   Uprobe::clear_breakpoint_in_mm(mm, uprobe);
24. }
25. Ok(())
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `handler_in_registration_order` | INVARIANT | per-handle_swbp: consumers iterated in cons_node insertion order. |
| `filter_before_handler` | INVARIANT | per-consumer: filter (if any) invoked before handler. |
| `no_handler_when_no_uprobe` | INVARIANT | per-handle_swbp: find_uprobe miss ⟹ no handler runs. |
| `xol_slot_isolated` | INVARIANT | per-mm XOL: distinct tasks receive distinct XOL slots. |
| `rcu_read_held_during_dispatch` | INVARIANT | per-handle_swbp: consumer iteration under RCU. |
| `cap_check_at_register_only` | INVARIANT | per-trap path: never checks creds (already checked at register). |

### Layer 2: TLA+

`kernel/uprobe.tla`:
- States: per-(inode,offset) uprobe registry, per-mm BP-installed set, per-task IP / XOL slot.
- Properties:
  - `safety_no_handler_without_registration` — trap at unregistered address ⟹ SIGTRAP, no consumer fired.
  - `safety_filter_gates` — filter false ⟹ handler not invoked.
  - `safety_xol_uniqueness` — concurrent tasks get distinct XOL slots.
  - `safety_remove_scoped_to_mm` — UPROBE_HANDLER_REMOVE only affects current mm.
  - `liveness_trap_resumes` — every handled trap returns control to userspace.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `handle_swbp` post: success ⟹ at least one consumer evaluated ∨ no_consumers | `Uprobe::handle_swbp` |
| `xol_take_slot` post: returned slot is RX, distinct per concurrent task | `Uprobe::xol_take_slot` |
| `arch_uprobe_pre_xol` post: regs.ip points into XOL VMA | `arch_uprobe_pre_xol` |
| `clear_breakpoint_in_mm` post: BP byte restored ∧ uprobe.consumers in this mm = 0 | `Uprobe::clear_breakpoint_in_mm` |

### Layer 4: Verus / Creusot functional

Per-`Documentation/trace/uprobetracer.rst` semantic equivalence; bcc/bpftrace selftests pass; `tools/testing/selftests/bpf/uprobe_multi*` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`uprobe` trampoline reinforcement:

- **Per-handler RCU iteration** — defense against per-consumer-list mutation race.
- **Per-mm XOL isolation** — defense against per-cross-mm XOL leak.
- **Per-XOL slot RX-only** — defense against per-W^X violation in XOL.
- **Per-find_uprobe miss = SIGTRAP** — defense against per-stale-BP zombie hits.
- **Per-arch insn analyzer pre-register** — defense against per-malformed-displaced-insn corruption.
- **Per-filter pre-handler** — defense against per-cross-mm unsolicited handler.
- **Per-no-creds-in-trap-path** — defense against per-trap-time TOCTOU on caps.

## Grsecurity / PaX surface

- **Verifier-gated uprobe attach (CAP_SYS_ADMIN-attached only)** — grsec elevates uprobe registration from `CAP_PERFMON` to `CAP_SYS_ADMIN` in `init_user_ns`. Userspace probing inside a container is denied unless the namespace was explicitly delegated this capability via `bpf_token`. Stops in-container attackers from hot-patching glibc.
- **GRKERNSEC_HARDEN_TRACE** — global toggle to disable userspace tracing entirely (uprobe, kprobe, ftrace), regardless of caps. Production hardening default for many distros.
- **PAX_KERNEXEC during XOL slot install** — XOL VMA is installed under W^X enforcement; the displaced instruction copy is written before the pages are flipped from RW to RX, never both.
- **PaX UDEREF on `pt_regs` access** — kernel handlers reading user state (e.g. `bpf_probe_read_user`) go through UDEREF/SMAP; a malicious handler cannot read kernel memory via a forged user pointer.
- **GRKERNSEC_HIDESYM in XOL VMA naming** — XOL VMA in `/proc/<pid>/maps` is labeled with a stable name (`[uprobes]`) but the underlying address is excluded from kallsyms-like leaks.
- **PAX_REFCOUNT on uprobe consumer refcounts** — saturating refcounts; over-deregister underflow panics rather than UAF.
- **GRKERNSEC_AUDIT_BPF + uprobe attach audit** — every uprobe register/unregister is audited with consumer id, target inode, offset, creds. Sufficient for forensic reconstruction of an in-process patch.
- **CAP_BPF + CAP_PERFMON split** — even with grsec relaxed, attach via bpf-link requires both caps; one alone is insufficient.
- **Anti-fingerprint** — uprobe slot allocation is randomized within the XOL VMA, denying offset-based slot prediction.
- **PAX_NOEXEC compatible** — XOL pages are RX, never RWX; PAX_NOEXEC validation is enforced.
- **No_new_privs disables uprobe attach to self** — a task with NNP set cannot have new uprobes attached to itself from outside (per consumer filter), preventing late-injection in sandboxed children.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `uretprobe` return-trampoline (Tier-5 separate doc — return-side handler).
- `bpf(BPF_LINK_CREATE, BPF_TRACE_UPROBE_MULTI, ...)` registration ABI (covered in `bpf.md`).
- `perf_event_open` uprobe PMU (covered in `perf_event_open.md`).
- kprobes (Tier-3 in `kernel/kprobes.md`).
- ftrace dynamic tracing (Tier-3 in `kernel/ftrace.md`).
- Implementation code.
