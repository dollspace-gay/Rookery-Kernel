# Tier-3: net/xfrm/nat-keepalive — RFC 3948 NAT-Traversal keepalive emitter

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_nat_keepalive.c
  - include/net/xfrm.h
  - include/uapi/linux/xfrm.h
-->

## Summary
Tier-3 design for the kernel-side NAT-T keepalive emitter (RFC 3948 § 4): for each SA configured with `XFRMA_NAT_KEEPALIVE_INTERVAL` and `encap_type == UDP_ENCAP_ESPINUDP`, the kernel periodically sends a single 1-byte UDP packet (containing only `0xff`) to the peer to refresh NAT mappings on intermediate routers. Without keepalive, NAT boxes typically time out idle UDP mappings after 30-300 seconds, breaking the SA.

Replaces the older userspace-emitted keepalive (where strongSwan/libreswan would manage their own timer); kernel-side emission is more efficient (no userspace wakeup) and survives daemon restarts.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (per-SA `encap` member + `nat_keepalive_interval` field), `net/xfrm/output.md` (uses standard ESP TX path's UDP-encap wrapping; the keepalive packet is wrapped identically).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-SA NAT-T keepalive timer + emitter | `net/xfrm/xfrm_nat_keepalive.c` |
| Public API | `include/net/xfrm.h` |
| UAPI: `XFRMA_NAT_KEEPALIVE_INTERVAL` NLA | `include/uapi/linux/xfrm.h` |

## Compatibility contract

### `XFRMA_NAT_KEEPALIVE_INTERVAL` NLA

`__u32` value in seconds; 0 disables keepalive (the upstream-default; userspace daemons emit keepalives themselves in this case). Identical UAPI.

### Keepalive packet wire format (RFC 3948 § 4)

Per-RFC: a single `0xff` byte sent over UDP/4500 (or whatever port the SA's `encap.encap_sport` configures), with no IKE/ESP header. Rationale: the byte is invalid as both IKE (IKE messages start with the IKE Initiator SPI which is 8 bytes of cookie data) and ESP (ESP starts with 4-byte SPI followed by 4-byte sequence; minimum ESP packet is much larger), so receivers reliably interpret it as a keepalive and drop it without further processing.

Wire format byte-identical so existing strongSwan/libreswan peers handle it correctly.

### Per-SA timer

When SA is added via NEWSA with `XFRMA_NAT_KEEPALIVE_INTERVAL=N` (N > 0):
1. Allocate per-SA hrtimer
2. Schedule first fire at `N` seconds in the future
3. On fire: emit one keepalive packet via the SA's `encap.encap_oa` source + `id.daddr` destination using the standard UDP-encap path
4. Re-schedule for `N` seconds later

On SA delete: cancel timer; free hrtimer.

Identical timer model.

### Counters

Per-netns counter (cross-ref `/proc/net/xfrm_stat`): `XfrmAcquireError` not relevant here; rather, increment a per-SA `nat_keepalive_count` (exposed via `ip xfrm state list`'s extended attributes). Identical.

### Interaction with `XFRMA_OUTPUT_MARK` and per-SA mark

Keepalive packets get the same `skb->mark` as regular ESP packets via that SA — for routing-table selection through XFRMi or fwmark-based policy routing. Identical inheritance.

### Timer accuracy

hrtimer-based; per-SA scheduling jitter ≤ 1ms typical. Identical accuracy characteristics.

### Disable semantics

`XFRMA_NAT_KEEPALIVE_INTERVAL=0` (default if NLA absent): kernel does not emit; userspace daemon manages keepalive itself by sending keepalive UDP packets via a normal IKE socket. This preserves backward compatibility — daemons that don't know about kernel-side keepalive continue to work.

## Requirements

- REQ-1: `XFRMA_NAT_KEEPALIVE_INTERVAL` NLA on NEWSA / UPDSA: `__u32` seconds; 0 = disabled (default).
- REQ-2: Per-SA hrtimer scheduled at interval; first fire at `+ interval` seconds from SA install.
- REQ-3: Keepalive packet wire format: 1-byte `0xff` over UDP using SA's `encap.encap_sport`/`encap.encap_dport`; identical to RFC 3948 § 4.
- REQ-4: Source-address selection: keepalive packet uses SA's `encap.encap_oa` (or `props.saddr` if `encap_oa` zero); destination is `id.daddr`.
- REQ-5: Per-SA mark inheritance: keepalive `skb->mark` matches regular ESP packet mark via SA.
- REQ-6: SA delete cleanup: timer canceled + freed; no use-after-free even on concurrent NEWSA-DELSA-NEWSA.
- REQ-7: SA update via UPDSA with new keepalive interval: timer re-scheduled at new interval.
- REQ-8: Per-SA `nat_keepalive_count` counter incremented per emit; exposed via `ip xfrm state list`.
- REQ-9: Disable preserves backward compat: SA without keepalive interval → kernel doesn't emit; userspace daemon retains responsibility.
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: NAT-T SA with `XFRMA_NAT_KEEPALIVE_INTERVAL=20` test: install SA; tcpdump on outer interface shows 1-byte UDP packets from src/dport=4500 every ~20s. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-2: Wire-format test: capture keepalive packet; payload is exactly `0xff`; UDP length = 9 (8 UDP + 1 byte). (covers REQ-3)
- [ ] AC-3: Source-address test: SA with `encap.encap_oa=192.0.2.1`; keepalive packet has saddr=192.0.2.1. (covers REQ-4)
- [ ] AC-4: Mark-inheritance test: SA with mark=0x100; keepalive packet has mark=0x100 (verifiable via netfilter MARK match counter). (covers REQ-5)
- [ ] AC-5: SA-delete test: install SA; immediately delete; verify timer canceled (no spurious keepalive after delete). (covers REQ-6)
- [ ] AC-6: SA-update test: NEWSA with interval=20; UPDSA with interval=10 → next keepalive fires at +10s, not +20s. (covers REQ-7)
- [ ] AC-7: Counter test: 5 keepalives emitted; SA's `nat_keepalive_count` field reads 5. (covers REQ-8)
- [ ] AC-8: Disable backward-compat test: NAT-T SA without `XFRMA_NAT_KEEPALIVE_INTERVAL` → kernel doesn't emit; userspace daemon's keepalive (if any) traffic visible normally. (covers REQ-9)
- [ ] AC-9: Concurrent NEWSA-DELSA stress test: 1000 iterations of (install SA, immediately delete) → no kernel oops, no leaked timer (verify via /proc/timer_list). (covers REQ-6)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::xfrm::nat_keepalive::NatKeepalive` — top-level
- `kernel::net::xfrm::nat_keepalive::Timer` — per-SA hrtimer mgmt (alloc/start/restart/cancel)
- `kernel::net::xfrm::nat_keepalive::Emitter` — keepalive packet build + UDP-encap wrap + xmit
- `kernel::net::xfrm::nat_keepalive::SaHook` — NEWSA/UPDSA/DELSA hook integration

### Locking and concurrency

- **Per-SA `lock`** (spinlock, inherited from xfrm_state): held during timer arm/cancel
- **hrtimer cancel-sync**: SA delete path uses `hrtimer_cancel_sync` to ensure no in-flight timer callback after free

### Error handling

- `Err(EINVAL)` — keepalive interval set on non-NAT-T SA (no `encap`)
- `Err(ENOMEM)` — hrtimer alloc fail (extremely rare)
- `Err(EAGAIN)` — emit failed due to no route (transient; retry on next interval)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Timer arm/restart/cancel under SA lock (no double-arm; no use-after-free) | `kani::proofs::net::xfrm::nat_keepalive::timer_safety` |
| Keepalive packet build (skb_alloc + 1-byte payload + bounds-check) | `kani::proofs::net::xfrm::nat_keepalive::emit_safety` |
| SA-delete teardown (cancel-sync prevents callback-after-free) | `kani::proofs::net::xfrm::nat_keepalive::delete_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-SA timer state | timer is armed ⇔ `keepalive_interval > 0`; never armed for non-NAT-T SAs | `kani::proofs::net::xfrm::nat_keepalive::state_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/xfrm/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-SA emit fn-ptr table `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-SA timer + skb (cross-ref `net/xfrm/state.md`, `net/skbuff.md`)
- **CONSTIFY**: see above
- **MEMORY_SANITIZE**: freed-after-emit keepalive skb cleared (no key material in keepalive payload, but standard practice)
- **USERCOPY**: not applicable (no userspace data crosses this component)
- **SIZE_OVERFLOW**: timer-interval arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook: none directly (timer-driven kernel-internal emission); LSM gate at NETLINK_XFRM mutation level — `user.md`.
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds `XFRMA_NAT_KEEPALIVE_INTERVAL` and related XFRM netlink attrs on copyin from userland.
- **PAX_KERNEXEC** — keeps the `static const` per-SA emit fn-ptr table and timer-callback code text W^X.
- **PAX_RANDKSTACK** — randomises stack per keepalive-timer entry; defeats ROP from timer-driven attacker stimuli.
- **PAX_REFCOUNT** — wraps `xfrm_state` refs held by the keepalive timer so SA delete during timer fire cannot UAF.
- **PAX_MEMORY_SANITIZE** — zeroes freed-after-emit keepalive skbs; though payload is 0xFF / fixed, the slab page may otherwise carry neighbour data.
- **PAX_UDEREF** — protects the netlink path that configures the keepalive interval from userland deref on malformed attrs.
- **PAX_RAP / kCFI** — protects indirect dispatch through the per-SA emit fn-ptr table and the underlying ESP-in-UDP wrap callback.
- **GRKERNSEC_HIDESYM** — hides `xfrm_nat_keepalive_*`, related timer symbols from unprivileged readers.
- **GRKERNSEC_DMESG** — restricts dmesg so keepalive-emit errors don't leak peer addresses / SPIs.
- **IPsec SAD/SPD CAP_NET_ADMIN** — `XFRMA_NAT_KEEPALIVE_INTERVAL` is set only via CAP_NET_ADMIN-gated XFRM netlink; an unprivileged user cannot adjust the cadence on a foreign SA.
- **XFRM key MEMORY_SANITIZE** — no SA keys touched by this path; underlying ESP-in-UDP wrap keys are wiped at `state.md` on SA destroy.
- **Replay-window protected** — keepalive does not advance the replay counter; the underlying ESP path serialises replay under `x->lock`.
- **Interval SIZE_OVERFLOW** — the `nat_keepalive_interval * HZ` jiffies arithmetic is checked, preventing wrap into a near-zero rearm time that would DoS the box with timer floods.

Rationale: NAT-keepalive is a timer-driven internal emitter, so the attacker surface is the netlink configuration plus the timer/skb lifetime; CAP_NET_ADMIN gating, REFCOUNT on the timer-held SA ref, MEMORY_SANITIZE on emitted skbs, and SIZE_OVERFLOW-checked interval arithmetic collapse the practical attack to the underlying ESP path covered separately.

## Open Questions

(none — RFC 3948 § 4 keepalive format + Linux per-SA timer model exhaustively specified)

## Out of Scope

- Userspace-driven keepalive emission (when `XFRMA_NAT_KEEPALIVE_INTERVAL=0`, daemon's responsibility)
- ESP-in-UDP wrap details (cross-ref `net/xfrm/output.md` REQ-7 — keepalive uses the same wrap path)
- 32-bit-only paths
- Implementation code
