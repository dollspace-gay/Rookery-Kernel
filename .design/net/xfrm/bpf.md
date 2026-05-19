# Tier-3: net/xfrm/bpf — XFRM BPF integration (state + interface kfunc sets)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_state_bpf.c
  - net/xfrm/xfrm_interface_bpf.c
  - include/net/xfrm.h
-->

## Summary
Tier-3 design for the XFRM BPF integration layer: read-only kfunc sets that expose per-SA state (`xfrm_state_bpf.c` — `bpf_xdp_get_xfrm_state` style accessors) and per-XFRMi metadata (`xfrm_interface_bpf.c` — `xfrm_md_info` accessors) to BPF-XDP / BPF-TC programs. Lets userspace tools (Cilium, Calico, etc.) implement IPSec-aware traffic classification + per-tenant policy enforcement at line rate, including looking up SAs by SPI to fetch their associated policy + per-tenant `if_id` for routing decisions.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (consumed: SA state); `net/xfrm/interface.md` (consumed: per-XFRMi metadata); `kernel/bpf/00-overview.md` (BPF infrastructure: kfunc registration, verifier).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-SA BPF kfunc set: `bpf_xdp_get_xfrm_state`, `bpf_xdp_xfrm_state_release`, related accessors | `net/xfrm/xfrm_state_bpf.c` |
| Per-XFRMi BPF kfunc set: `bpf_skb_get_xfrm_info`, `bpf_skb_set_xfrm_info` | `net/xfrm/xfrm_interface_bpf.c` |
| Public API | `include/net/xfrm.h` |

## Compatibility contract

### `bpf_xdp_get_xfrm_state(...)` kfunc

Signature (per upstream `__bpf_kfunc_*` macro):
```c
struct xfrm_state *bpf_xdp_get_xfrm_state(struct xdp_md *ctx,
                                          struct bpf_xfrm_state_opts *opts,
                                          u32 opts__sz);
```

Where `bpf_xfrm_state_opts` carries:
- `error` (int — kfunc returns it on failure)
- `netns_id` (s32 — `-1` for current netns)
- `mark` (u32)
- `daddr` (struct in_addr / in6_addr)
- `spi` (__be32)
- `proto` (u8 — IPPROTO_AH / ESP / IPCOMP)
- `family` (u16 — AF_INET / AF_INET6)

Returns: pointer to `xfrm_state` (refcount-acquired) on success, or NULL on miss with `opts->error` set. Caller MUST release via `bpf_xdp_xfrm_state_release`.

Identical signature so existing Cilium / cilium-libbpf programs work unchanged.

### `bpf_xdp_xfrm_state_release(struct xfrm_state *)` kfunc

Releases the refcount acquired by `bpf_xdp_get_xfrm_state`. Mandatory for verifier safety: kfunc set is registered with `BPF_KFUNC_REG_GET_RELEASE_PAIR` so verifier ensures every `get` has a corresponding `release` on every program path.

### `bpf_skb_get_xfrm_info(struct __sk_buff *skb, struct bpf_xfrm_info *info)` kfunc

Reads per-skb XFRMi metadata (when COLLECT_METADATA mode is enabled — cross-ref `net/xfrm/interface.md` REQ-5). `bpf_xfrm_info`:
- `if_id` (u32)
- `link` (int — underlying ifindex)

Returns 0 on success; -ENOENT if no XFRM metadata attached.

### `bpf_skb_set_xfrm_info(struct __sk_buff *skb, const struct bpf_xfrm_info *info)` kfunc

Writes per-skb XFRMi metadata for downstream pipeline stages to consume. Requires CAP_BPF + the COLLECT_METADATA mode active on the originating XFRMi (verifier-checkable via context).

### kfunc set registration

Kfunc sets are registered with the BPF verifier via `register_btf_kfunc_id_set` per upstream's macro. Each kfunc has:
- BTF metadata for arg-types + return type
- `BPF_PROG_TYPE_*` mask (XDP for `xfrm_state_bpf`; TC + XDP for `xfrm_interface_bpf`)
- KF_TRUSTED_ARGS / KF_RET_NULL / KF_ACQUIRE / KF_RELEASE flags per-kfunc

Identical registration so verifier accepts/rejects programs identically.

### Read-only contract

All XFRM BPF kfuncs are read-only with respect to xfrm_state internal fields (except for the explicit `set_xfrm_info` kfunc which mutates per-skb metadata, not SA state). Callers cannot mutate SA replay window, lifetime, etc.; mutation remains exclusive to NETLINK_XFRM (`net/xfrm/user.md`) + PF_KEYv2 (`net/xfrm/af-key.md`).

Identical immutability contract.

## Requirements

- REQ-1: `bpf_xdp_get_xfrm_state` kfunc: signature byte-identical to upstream; returns refcount-acquired `xfrm_state *` on hit, NULL + `opts->error` on miss.
- REQ-2: `bpf_xdp_xfrm_state_release` kfunc: paired with `bpf_xdp_get_xfrm_state`; verifier enforces get/release pairing on every path.
- REQ-3: `bpf_skb_get_xfrm_info` kfunc: reads `if_id` + `link` from per-skb XFRMi metadata; returns 0 on hit, -ENOENT on miss.
- REQ-4: `bpf_skb_set_xfrm_info` kfunc: writes `if_id` + `link` to per-skb metadata; requires CAP_BPF + active COLLECT_METADATA.
- REQ-5: kfunc set registration: BTF metadata + per-prog-type mask + per-kfunc flags identical to upstream's `register_btf_kfunc_id_set` invocations.
- REQ-6: Read-only contract: SA state cannot be mutated via BPF; only `bpf_skb_set_xfrm_info` mutates (and only per-skb metadata, not SA fields).
- REQ-7: Verifier integration: BPF programs that use `bpf_xdp_get_xfrm_state` are verified with refcount-aware path analysis (verifier in `kernel/bpf/00-overview.md` proves: every program path has paired release).
- REQ-8: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: BPF program test: load a BPF-XDP program calling `bpf_xdp_get_xfrm_state` with valid SPI/daddr/family → kfunc returns SA pointer; subsequent `bpf_xdp_xfrm_state_release` releases. (covers REQ-1, REQ-2)
- [ ] AC-2: Verifier-rejection test: BPF program calls `bpf_xdp_get_xfrm_state` but doesn't call release on one path → verifier rejects with refcount-leak error. (covers REQ-2, REQ-7)
- [ ] AC-3: Miss test: BPF program calls `bpf_xdp_get_xfrm_state` with non-existent SPI → returns NULL + `opts->error == -ENOENT`. (covers REQ-1)
- [ ] AC-4: BPF-TC test: program calls `bpf_skb_get_xfrm_info` on skb with COLLECT_METADATA enabled → returns `if_id` matching XFRMi configuration. (covers REQ-3)
- [ ] AC-5: BPF-TC set test: program calls `bpf_skb_set_xfrm_info(if_id=42)`; downstream pipeline observes `if_id=42` per skb metadata. (covers REQ-4)
- [ ] AC-6: kfunc-registration test: `bpftool kfunc list` includes `bpf_xdp_get_xfrm_state` + `bpf_xdp_xfrm_state_release` + `bpf_skb_get_xfrm_info` + `bpf_skb_set_xfrm_info` with correct prog-type masks. (covers REQ-5)
- [ ] AC-7: Read-only test: BPF program attempting to mutate `xfrm_state.id.spi` → verifier rejects (no kfunc/helper path allows mutation). (covers REQ-6)
- [ ] AC-8: Hardening section present and follows template. (covers REQ-8)

## Architecture

### Rust module organization

- `kernel::net::xfrm::bpf::XfrmStateBpf` — `bpf_xdp_get_xfrm_state` + `bpf_xdp_xfrm_state_release`
- `kernel::net::xfrm::bpf::XfrmIfaceBpf` — `bpf_skb_get_xfrm_info` + `bpf_skb_set_xfrm_info`
- `kernel::net::xfrm::bpf::KfuncRegistration` — kfunc set BTF + flags registration

### Locking and concurrency

- **Per-SA refcount via `Refcount`**: acquired by `bpf_xdp_get_xfrm_state`; released by paired kfunc
- **No new global locks**: all kfuncs are RCU-side or refcount-protected
- **BPF-program execution context**: NMI-safe / RCU-side reads only

### Error handling

- `Err(ENOENT)` — SA not found / no XFRM metadata on skb
- `Err(EINVAL)` — bad opts struct / bad family
- Set via `opts->error` field for kfuncs that return pointer (NULL signals failure)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `bpf_xdp_get_xfrm_state` refcount-acquire (sound under RCU read-side) | `kani::proofs::net::xfrm::bpf::get_state_safety` |
| `bpf_xdp_xfrm_state_release` refcount-release (idempotent on already-zero refcount? — should be impossible per verifier) | `kani::proofs::net::xfrm::bpf::release_state_safety` |
| `bpf_skb_get_xfrm_info` skb-metadata read (no out-of-bounds) | `kani::proofs::net::xfrm::bpf::get_info_safety` |
| `bpf_skb_set_xfrm_info` skb-metadata write (CAP_BPF + COLLECT_METADATA gate) | `kani::proofs::net::xfrm::bpf::set_info_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-SA refcount across BPF kfunc | refcount ≥ 1 between `get` and `release`; never negative | `kani::proofs::net::xfrm::bpf::refcount_invariants` |
| kfunc set registration | every registered kfunc has matching BTF metadata + non-NULL fn-ptr | `kani::proofs::net::xfrm::bpf::registration_invariants` |

### Layer 4: Functional correctness (opt-in; depends on `kernel/bpf/00-overview.md` Layer 4)

- **Verifier refcount-pairing soundness theorem** (cross-ref `kernel/bpf/00-overview.md`) — proves: BPF verifier rejects programs that fail to call `bpf_xdp_xfrm_state_release` on any reachable program path. Owned by BPF-verifier Tier-3, consumed here.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-SA refcount acquire/release uses `Refcount` (saturating) — verifier guarantees pairing | § Mandatory |
| **CONSTIFY** | kfunc registration tables `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, CONSTIFY**: see above
- **MEMORY_SANITIZE**: SA state read by BPF programs is read-only; no key material reachable through these kfuncs (verified at design time — kfunc accessors return aggregate `xfrm_state` pointer but verifier restricts member access)
- **USERCOPY**: not applicable (BPF programs run in kernel context with verifier-checked memory access)

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_BPF)` already required for BPF-program load; GR-RBAC policy can deny per-subject (default empty).
- LSM hook `security_xfrm_state_pol_flow_match` indirectly invoked through SA-lookup paths (cross-ref `net/xfrm/state.md`).
- Useful default GR-RBAC policy: deny BPF program loading with XFRM kfunc usage outside gradm-marked `bpf_xfrm_user` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds any user-supplied BPF map keys/values that touch the XFRM lookup edge (e.g. tunnel-id maps).
- **PAX_KERNEXEC** — keeps the `static const struct btf_kfunc_id_set` registrations and `xfrm_state`-helper kfunc tables text W^X.
- **PAX_RANDKSTACK** — randomises stack per kfunc dispatch entry; protects against ROP through BPF-controlled call sites.
- **PAX_REFCOUNT** — wraps the `xfrm_state.refcnt` acquire/release that the kfunc pair `bpf_xdp_get_xfrm_state`/`bpf_xdp_xfrm_state_release` performs; verifier-enforced pairing plus saturating refcount.
- **PAX_MEMORY_SANITIZE** — zeroes any scratch BPF map values that BPF programs use to stage XFRM lookup context.
- **PAX_UDEREF** — irrelevant inside BPF (no userland deref), but applies on the bpf(2) syscall path that installs these programs.
- **PAX_RAP / kCFI** — protects indirect dispatch of kfunc resolution from BPF text into the kernel `xfrm_state_lookup_byspi`/`xfrm_state_put` shim; BPF lookup hook is exactly this surface.
- **GRKERNSEC_HIDESYM** — hides BPF-XFRM kfunc symbol names from `kallsyms` so non-root cannot enumerate the available hook set.
- **GRKERNSEC_DMESG** — restricts dmesg so verifier-reject traces don't leak BPF program structure / SA SPIs.
- **BPF lookup hook PAX_KERNEXEC** — the `xfrm_state_lookup_byspi` callable from BPF is reached via a `static const` kfunc table whose .text is non-writable; no runtime patching.
- **AEAD/AH/ESP cipher PAX_RAP** — BPF cannot reach cipher dispatch directly; the kfunc set exposes only read-only `xfrm_state` fields, verifier-restricted.
- **CAP_BPF + CAP_NET_ADMIN gating** — programs that link XFRM kfuncs require both CAPs; gradm `bpf_xfrm_user` role recommended as the only subject allowed.
- **kfunc allowlist** — only enumerated kfunc IDs are exported; an attacker who controls a BPF program cannot synthesise a call into arbitrary XFRM internals.

Rationale: BPF + XFRM is the most direct way for a privileged userland program to drive arbitrary indirect calls against IPsec SA state. CONSTIFY on the kfunc registration, PAX_RAP/kCFI on the dispatch edge, verifier-enforced REFCOUNT pairing, and the two-CAP gate (CAP_BPF + CAP_NET_ADMIN) confine this surface to a static, statically-checked allowlist.

## Open Questions

(none — XFRM BPF kfunc sets are exhaustively specified by upstream)

## Out of Scope

- BPF verifier internals (cross-ref `kernel/bpf/00-overview.md`)
- SA storage (cross-ref `net/xfrm/state.md`)
- XFRMi storage (cross-ref `net/xfrm/interface.md`)
- 32-bit-only paths
- Implementation code
