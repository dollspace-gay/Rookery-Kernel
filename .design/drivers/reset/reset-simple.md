# Tier-3: drivers/reset/reset-simple.c — one-register-bit reset controller (SoCFPGA, STM32, Allwinner, Aspeed, Synopsys DW, Sophgo, Bitmain, …)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/reset/core.md
upstream-paths:
  - drivers/reset/reset-simple.c
  - include/linux/reset/reset-simple.h
-->

## Summary

`reset-simple` is the generic "N reset lines packed into a contiguous MMIO register file, one bit per line" reset controller — the most common shape in SoC silicon. The driver exports `struct reset_simple_data` (held by every consumer driver that reuses this shape) and `reset_simple_ops` (the `reset_control_ops` vtable wiring `.assert/.deassert/.reset/.status` to bit-toggle on a memory-mapped register). A small built-in platform driver wraps the same primitives for vanilla DT bindings (Allwinner sun6i, STM32 RCC, SoCFPGA Stratix10, Aspeed AST24/25/26 LPC, Synopsys DW, Sophgo CV1800/SG2042, Bitmain BM1880, ZTE, BRCM bcm4908).

Two per-binding switches handle SoC variance: `active_low` (whether a SET bit asserts or deasserts the reset) and `status_active_low` (whether a SET bit means line-is-asserted on read-back). The driver picks them from `struct reset_simple_devdata` matched off the DT `compatible` string; bindings without devdata default to "set=asserted, read-back set=asserted".

This Tier-3 covers `drivers/reset/reset-simple.c` (~210 lines: bit-level update helpers, `reset_simple_ops`, per-compat devdata table, platform probe) and the public header `include/linux/reset/reset-simple.h` (used by per-SoC clk-and-reset combo drivers that embed `reset_simple_data` directly).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct reset_simple_data` | per-instance state: rcdev, membase, lock, active_low, status_active_low, reset_us | `drivers::reset::Simple::Data` |
| `struct reset_simple_devdata` | per-compatible match data: reg_offset, nr_resets, active_low, status_active_low | `Simple::DevData` |
| `reset_simple_ops` | exported `reset_control_ops` vtable | `Simple::Ops` |
| `reset_simple_update(rcdev, id, assert)` | core helper: lock + read-modify-write the bit | `Simple::update` |
| `reset_simple_assert(rcdev, id)` / `_deassert(rcdev, id)` | dispatch into `_update` | `Simple::assert` / `_deassert` |
| `reset_simple_reset(rcdev, id)` | assert → usleep(reset_us) → deassert (refused if `reset_us == 0`) | `Simple::reset` |
| `reset_simple_status(rcdev, id)` | read bit + XOR with `status_active_low` | `Simple::status` |
| `reset_simple_probe(pdev)` | platform-bus probe: ioremap + devdata match + register | `Simple::probe` |
| `reset_simple_dt_ids[]` | per-compat match table | `Simple::DtIds` |
| `to_reset_simple_data(rcdev)` | `container_of` helper for embedding drivers | `Simple::from_rcdev` |

## Compatibility contract

REQ-1: Reset register file is a contiguous MMIO region; line `id` maps to register `membase + (id / 32) * 4`, bit `id % 32` (treats the region as a u32 array, little-endian bit ordering inside each u32).

REQ-2: Default `nr_resets = resource_size(res) * BITS_PER_BYTE` — the full ioremap'd extent. Per-compat devdata may override.

REQ-3: DT binding shape per `Documentation/devicetree/bindings/reset/`: `#reset-cells = <1>` (line index); consumers use `resets = <&rstc N>`.

REQ-4: Per-compat `reg_offset` lets the driver advance `data->membase` past a fixed-offset header (e.g. SoCFPGA Stratix10 starts at `+0x20`).

REQ-5: `active_low` semantics:
- `active_low = false` (default): bit SET means line is asserted; bit CLEAR means deasserted.
- `active_low = true`: bit CLEAR means line is asserted; bit SET means deasserted.

The XOR `assert ^ data->active_low` in `reset_simple_update` chooses set-vs-clear accordingly.

REQ-6: `status_active_low` semantics on `_status`:
- `status_active_low = false`: bit SET on read-back means line currently asserted.
- `status_active_low = true`: bit CLEAR on read-back means line currently asserted.

Distinct from `active_low` because some SoCs invert the write polarity vs read polarity.

REQ-7: `reset_us` field (set by embedding drivers, NOT settable via DT for the platform-driver path) gates `_reset`: if 0, `_reset` returns `-ENOTSUPP` (consumer must explicitly assert+deassert). Otherwise `usleep_range(reset_us, reset_us * 2)` between assert + deassert.

REQ-8: Per-instance spinlock (`data->lock`) protects the RMW; taken with `spin_lock_irqsave` so a reset toggle from IRQ context (rare but seen on watchdog/wdt resets) is safe.

REQ-9: `reset_simple_ops` exports `.assert/.deassert/.reset/.status` only — no `.acquire/.release` (resets-simple lines are exclusively assignable through the core's exclusive/shared logic).

REQ-10: Platform driver (`reset_simple_driver`) is `builtin_platform_driver` — must be in `.text`, not modular. The exported `reset_simple_ops` symbol is reused by modular SoC drivers that embed `reset_simple_data`.

REQ-11: `reset_simple_dt_ids[]` enumerates known compatibles: `altr,stratix10-rst-mgr`, `st,stm32-rcc`, `allwinner,sun6i-a31-clock-reset`, `zte,zx296718-reset`, `aspeed,ast2400-lpc-reset`, `aspeed,ast2500-lpc-reset`, `aspeed,ast2600-lpc-reset`, `bitmain,bm1880-reset`, `brcm,bcm4908-misc-pcie-reset`, `snps,dw-high-reset`, `snps,dw-low-reset`, `sophgo,cv1800b-reset`, `sophgo,sg2042-reset`.

REQ-12: Embedding pattern (used by `drivers/clk/sunxi-ng/`, etc.): drivers allocate `reset_simple_data` inside their own state, fill `rcdev.ops = &reset_simple_ops`, and call `devm_reset_controller_register` directly — bypassing the platform driver.

## Acceptance Criteria

- [ ] AC-1: STM32 boot — `compatible = "st,stm32-rcc"` matches; reset controller registers `nr_resets = (rcc-region-size * 8)` lines; consumer drivers (USB, ADC, etc.) probe.
- [ ] AC-2: SoCFPGA Stratix10 boot — `compatible = "altr,stratix10-rst-mgr"`: `nr_resets = 256` (SOCFPGA_NR_BANKS=8 × 32), `reg_offset = 0x20`, `status_active_low = true`; reset lines toggleable via consumer drivers.
- [ ] AC-3: Aspeed AST2500 LPC reset — exclusive get from LPC bridge driver; `_assert` writes bit, `_deassert` clears it; LPC bridge re-initializes cleanly.
- [ ] AC-4: Allwinner sun6i `active_low = true` — `_assert` clears the bit, `_deassert` sets it; cross-check on H3/H5 with peripheral ID; consumer drivers boot.
- [ ] AC-5: Read-back of `_status` reflects current assert state for both `status_active_low = true` (SoCFPGA) and `false` (default) cases.
- [ ] AC-6: Concurrent assert from CPU0 and deassert from CPU1 on different lines in the same bank does not corrupt either bit (RMW under per-instance spinlock).
- [ ] AC-7: `_reset` returns `-ENOTSUPP` on a bare platform-driver instance (reset_us not set); returns 0 + visible pulse on an embedding driver that sets `reset_us = 1000`.

## Architecture

`Simple::Data`:

```
struct Data {
  rcdev:              ResetControllerDev,
  membase:            NonNull<u8>,
  lock:               RawSpinLock,
  active_low:         bool,
  status_active_low:  bool,
  reset_us:           u32,
}
```

`Simple::DevData`:

```
struct DevData {
  reg_offset:         u32,
  nr_resets:          u32,
  active_low:         bool,
  status_active_low:  bool,
}
```

Hot-path `reset_simple_update(rcdev, id, assert)`:
1. `data = container_of(rcdev, struct reset_simple_data, rcdev)`.
2. `bank = id / 32`, `offset = id % 32`.
3. `spin_lock_irqsave(&data->lock, flags)`.
4. `reg = readl(data->membase + bank*4)`.
5. If `assert ^ data->active_low`: `reg |= BIT(offset)`. Else: `reg &= ~BIT(offset)`.
6. `writel(reg, data->membase + bank*4)`.
7. `spin_unlock_irqrestore`.

`_reset`:
1. If `data->reset_us == 0`: return `-ENOTSUPP`.
2. `_assert(rcdev, id)`; if err, return.
3. `usleep_range(reset_us, reset_us*2)`.
4. `_deassert(rcdev, id)`.

`_status`:
1. `reg = readl(data->membase + bank*4)`.
2. Return `!(reg & BIT(offset)) ^ !data->status_active_low`.

Probe flow `reset_simple_probe`:
1. `devdata = of_device_get_match_data(dev)` (NULL if compat has no devdata).
2. Allocate `data` via `devm_kzalloc`.
3. `membase = devm_platform_get_and_ioremap_resource(pdev, 0, &res)`.
4. `spin_lock_init(&data->lock)`.
5. Defaults: `data->rcdev.nr_resets = resource_size(res) * BITS_PER_BYTE`, `data->rcdev.ops = &reset_simple_ops`, `data->rcdev.of_node = dev->of_node`, `data->rcdev.owner = THIS_MODULE`.
6. Apply devdata: override `nr_resets`, set `active_low`, set `status_active_low`, capture `reg_offset`.
7. `data->membase += reg_offset`.
8. `return devm_reset_controller_register(dev, &data->rcdev)`.

Embedding pattern (sunxi-ng / nuvoton-npcm / Ingenic JZ):
```
struct foo_clk_ctx {
  struct reset_simple_data reset;
  /* + clk plumbing */
};
foo_probe(...):
  ctx->reset.rcdev.ops = &reset_simple_ops;
  ctx->reset.rcdev.of_node = dev->of_node;
  ctx->reset.rcdev.nr_resets = FOO_NR_RESETS;
  ctx->reset.membase = clk_mmio_base + RESET_REGS_OFFSET;
  ctx->reset.reset_us = 1000;
  devm_reset_controller_register(dev, &ctx->reset.rcdev);
```

`reset_simple_dt_ids[]` — per-compat match table. Three classes:
- explicit devdata (`altr,stratix10-rst-mgr` → `reset_simple_socfpga`).
- active-low devdata (`allwinner,sun6i-a31-clock-reset`, `zte,zx296718-reset`, `bitmain,bm1880-reset`, etc. → `reset_simple_active_low`).
- bare (no devdata, defaults apply) — `st,stm32-rcc`, `aspeed,ast24/25/26-lpc-reset`, `snps,dw-high-reset`.

No userspace surface: `reset-simple` exports nothing via sysfs, debugfs, char, or netlink. All interaction goes through the kernel-internal `reset_control_*` API on behalf of consumer drivers; user code can only observe effects indirectly (a peripheral block goes online/offline).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `id_in_nr_resets` | OOB | `id < rcdev->nr_resets` enforced by core before dispatch; `bank * 4 < ioremap'd region size` follows. |
| `bank_offset_no_wrap` | OOB | `bank = id / 32`, `offset = id % 32`; `bank * 4` bounded by `nr_resets / 8`. |
| `lock_pair_balanced` | RAII | every `spin_lock_irqsave` paired with `spin_unlock_irqrestore` on all return paths. |
| `reset_us_gate` | INVARIANT | `_reset` short-circuits to `-ENOTSUPP` iff `reset_us == 0`. |
| `xor_polarity_correct` | INVARIANT | `(assert ^ active_low)` selects set-vs-clear exactly per per-compat polarity. |

### Layer 2: TLA+

`models/reset/simple_concurrent_rmw.tla` (future): models concurrent RMW from two CPUs on different bits of the same bank under per-instance spinlock; proves no lost-update and no bit-flip outside the targeted line.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `_update` post: register bit `id%32` in word `id/32` matches the requested assert state per polarity | `reset_simple_update` |
| `_status` post: returned int is `1` iff hardware line is currently asserted per `status_active_low` polarity | `reset_simple_status` |
| `_reset` post: line is deasserted after success; `reset_us` ≤ elapsed wait ≤ `2 * reset_us` | `reset_simple_reset` |
| `probe` post: `rcdev` registered with `nr_resets > 0` and `membase` within `res` | `reset_simple_probe` |

### Layer 4: Verus/Creusot functional

Encoded: for `id ∈ [0, nr_resets)`, after `_assert(id)` then `_deassert(id)`, the underlying MMIO word reads back with bit `id%32` cleared (or set, per polarity) — a closed-form pre/post on the MMIO state observable to the consumer.

## Hardening

- `nr_resets` clamped at probe to `resource_size(res) * BITS_PER_BYTE`; core's `xlate` rejects out-of-range ids before dispatch.
- `reg_offset` applied once at probe; `membase` thereafter points strictly inside the ioremap'd region.
- Per-instance spinlock with IRQ-save; RMW cannot race with concurrent assert on the same bank.
- `reset_us = 0` default refuses `.reset` calls — embedding driver must explicitly opt in with a sane pulse width.
- `of_device_get_match_data` returns NULL for un-matched compatibles; probe applies safe defaults rather than dereffing a NULL devdata.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `reset_simple_data`, `reset_simple_devdata`, and the `reset_controller_dev` slab caches whitelisted; no userland copy path exists, classification still required for slab tracking.
- **PAX_KERNEXEC** — `reset_simple_ops` vtable, `reset_simple_dt_ids[]` match table, and probe path all live in W^X / `__ro_after_init` text/rodata; no patchable hook surface.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `reset_simple_update`/`_assert`/`_deassert`/`_reset` so IRQ-context callers don't expose stack-aligned gadgets.
- **PAX_REFCOUNT** — saturating `refcount_t` on the core `reset_control`'s `kref`; overflow trap defeats imbalanced assert/deassert that would wrap the per-handle reference.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `reset_simple_data` (devres-managed) and any embedding driver's enclosing struct; defense against stale `membase` pointer reuse.
- **PAX_UDEREF** — no userland surface; sysfs/debugfs reset state — if ever exposed — must go through SMAP/PAN-clean canonical copy helpers.
- **PAX_RAP / kCFI** — `reset_simple_ops` (`.assert`, `.deassert`, `.reset`, `.status`) marked kCFI-typed; refuse untagged indirect dispatch into these callbacks.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `reset_simple_dt_ids[]`, `reset_simple_ops`, and per-instance `membase` pointers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict reset-simple probe banners and devdata-mismatch warnings to CAP_SYSLOG so attackers cannot enumerate the SoC reset-bit topology from boot dmesg.
- **MMIO bounded** — `(id / 32) * 4 < resource_size(res)` enforced (via core's `id < nr_resets` clamp) before every readl/writel; defense against OOB MMIO access in case of a malformed DT `nr_resets` override.
- **No userspace surface** — `reset-simple` exposes zero ioctl/sysfs/debugfs/netlink; defense-in-depth ensures an attacker cannot directly drive a reset bit even with `CAP_SYS_ADMIN`.
- **DT binding validated** — per-compatible match enforces `#reset-cells = <1>`; refuse probe on a mis-bound node with the wrong cell count or a missing `reg` resource.
- **Polarity locked at probe** — `active_low` and `status_active_low` set once from devdata at probe; refused mutation thereafter (no runtime sysfs to flip polarity), defense against a hostile DT overlay that would invert all resets.
- **reset_us pulse width clamped** — `usleep_range(reset_us, reset_us * 2)` clamped against a sane ceiling (e.g. 1 second) in embedding drivers; defense against pathological large-pulse stalls that would soft-lockup the kernel during boot.

Rationale: every line in `reset-simple` controls whether a piece of SoC silicon (a DMA-capable USB or NIC block, the watchdog timer, the LPC bridge that gates BIOS/BMC firmware loading) is held inactive or released onto the system bus. Flipping a reset bit at the wrong moment crashes the SoC; flipping the wrong bit due to OOB MMIO offsets can release a DMA master before its firmware is loaded. MMIO clamping, polarity-locked-at-probe, kCFI on the ops, RO match table, and zero userspace surface turn `reset-simple` from "trust the consumer driver" into a structurally bounded toggle that an attacker cannot drive directly.

## Open Questions

- (none at this Tier-3 level)

## Out of Scope

- `core.c` reset framework (covered in `core.md`)
- per-SoC reset drivers that don't embed `reset_simple_data` (Qualcomm AOSS, Microchip MPFS, IMX SCU, etc.) — future Tier-3s
- DT binding YAML schemas (covered in upstream `Documentation/devicetree/bindings/reset/`)
- power-domain entanglements (covered in `drivers/base/power/` future Tier-3)
- Implementation code
