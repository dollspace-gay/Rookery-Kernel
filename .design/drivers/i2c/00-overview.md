# Tier-2: drivers/i2c — I2C bus subsystem (core + algorithms + adapters + slave + i2c-dev + SMBus + muxes)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/i2c/
  - include/linux/i2c.h
  - include/uapi/linux/i2c.h
  - include/uapi/linux/i2c-dev.h
-->

## Summary

Tier-2 wrapper for the I2C / SMBus subsystem — the framework underneath every chassis sensor (hwmon, thermal), every laptop touchpad/touchscreen via i2c-hid, every audio codec, every camera sensor, every PMIC, every EEPROM, every clock-generator, every RTC, every fan controller, every DDC HDMI/DP-AUX bridge. Components: **core** (`i2c-core-*.c` — bus type registration, adapter+algorithm+client lifecycle, ACPI / DT / fwnode enumeration, OF prober, slave-mode core, SMBus emulation-via-I2C-msg), **algos/** (per-algorithm implementations: bit-banging, PCA9564, Smbus-PEC), **busses/** (per-controller LLDs: i801 Intel SMBus, designware i2c-designware-{platform,pci}, piix4 AMD/VIA SMBus, mlxcpld, ali15x3, ali1535, ali1563, amd756, sis5595, sis630, sis96x, viapro, isch, nforce2, ismt, scmi, etc. — ~60 controllers), **i2c-dev** (`/dev/i2c-<N>` chardev for userspace I2C access), **i2c-mux** + **muxes/** (per-mux drivers: pca954x, gpio, mlxcpld, demux-pinctrl), **i2c-atr** (Address Translator), **i2c-smbus** (SMBus alert + ARP + Host-Notify), **i2c-stub** (in-memory stub for testing), **i2c-slave-eeprom** + **i2c-slave-testunit** (slave-mode example drivers), **i2c-boardinfo** (legacy board-info enumeration), **i2c-core-of-prober** (DT-based device probing).

## Compatibility contract — outline

- `/dev/i2c-<N>` chardev: `I2C_SLAVE`, `I2C_SLAVE_FORCE`, `I2C_TENBIT`, `I2C_FUNCS`, `I2C_RDWR`, `I2C_SMBUS`, `I2C_PEC` IOCTLs byte-identical (i2c-tools / i2cdetect / i2cset / i2cget / i2cdump / i2ctransfer consume unchanged).
- Per-adapter `/sys/bus/i2c/devices/i2c-<N>/{name,new_device,delete_device,...}` byte-identical.
- Per-client `/sys/bus/i2c/devices/<adapter-num>-<addr-hex>/{name,modalias,driver,subsystem}` byte-identical.
- `MODULE_DEVICE_TABLE(i2c, ...)` modalias `i2c:<name>` byte-identical (ACPI: `acpi:<HID>`; OF: `of:NaTalcCacc`); depmod loads right driver.
- ACPI + DT + fwnode + swnode enumeration paths preserved.
- SMBus emulation-via-I2C-msg semantics identical (SMBus quick / byte / word / block / process-call / proc-call32 / read-block / write-block all mapped).
- Slave mode (CONFIG_I2C_SLAVE) works for in-tree slave drivers.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/i2c/core.md` | `i2c-core-{base,acpi,of,of-prober,slave,smbus}.c` + `i2c-core.h`: bus core + enumeration + slave |
| `drivers/i2c/dev.md` | `i2c-dev.c`: `/dev/i2c-<N>` chardev |
| `drivers/i2c/smbus.md` | `i2c-smbus.c`: SMBus alert / ARP / Host-Notify |
| `drivers/i2c/algos.md` | `algos/`: per-algorithm (bit-bang / PCA9564 / SMBus-PEC) |
| `drivers/i2c/mux.md` | `i2c-mux.c` + `muxes/`: I2C mux framework + per-mux drivers |
| `drivers/i2c/atr.md` | `i2c-atr.c`: Address Translator |
| `drivers/i2c/busses-x86.md` | `busses/i2c-{i801,piix4,ali*,sis*,viapro,isch,ismt,nforce2}.c`: x86 chipset SMBus |
| `drivers/i2c/busses-designware.md` | `busses/i2c-designware-*.c`: Synopsys DesignWare I2C (Intel SoC) |
| `drivers/i2c/slave-eeprom-testunit.md` | `i2c-slave-eeprom.c` + `i2c-slave-testunit.c` |
| `drivers/i2c/stub.md` | `i2c-stub.c`: in-memory test stub |

## Compatibility outline (top-level)

- REQ-O1: `/dev/i2c-<N>` IOCTLs + sysfs surface byte-identical (i2c-tools consume unchanged).
- REQ-O2: `MODULE_DEVICE_TABLE(i2c, …)` + ACPI/OF modalias byte-identical.
- REQ-O3: Adapter+client+driver registration source-compat for in-tree consumers.
- REQ-O4: SMBus emulation semantics identical.
- REQ-O5: I2C slave mode source-compat.
- REQ-O6: TLA+ models declared (adapter lock + bus-arbitration on shared SMBus, mux child-bus selection race-freedom).
- REQ-O7: Hardening: row-1 features per `00-security-principles.md` + `/dev/i2c-*` CAP_SYS_RAWIO + LSM mediation.

## Acceptance Criteria

- [ ] AC-O1: `i2cdetect -y 0` lists devices on a reference x86 SMBus matching upstream baseline.
- [ ] AC-O2: i2c-hid touchpad on reference laptop enumerates + binds + works.
- [ ] AC-O3: i2c-tools `i2ctransfer` round-trip on i2c-stub.
- [ ] AC-O4: kselftest `tools/testing/selftests/i2c/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/i2c/adapter_lock.tla` | `drivers/i2c/core.md` (proves: adapter `bus_lock` + per-msg arbitration; concurrent transfers on shared adapter serialize correctly) |
| `models/i2c/mux_select.tla` | `drivers/i2c/mux.md` (proves: mux child-bus select + concurrent access never produces cross-channel data leak) |

Layer-4: `core.md` — `i2c_transfer(adap, msgs, num)` post: every msg.buf bounds-checked against msg.len; per-adap timeout enforced.

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-adapter + per-client refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-controller `i2c_algorithm`, per-driver `i2c_driver` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | msg.len + SMBus block-size arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed i2c_msg buffers cleared (carry sensor secrets, smartcard PINs) | § Default-on configurable |

GR-RBAC: per-role disallow `/dev/i2c-*` open (defends against userspace flashing EEPROMs / changing PMIC voltage / re-keying TPM via SMBus).

I2C-specific reinforcement: PMIC writes (voltage rails) require CAP_SYS_RAWIO + LSM mediation (defense against reglator-from-userspace brick attack); EEPROM writes via i2c-dev gated similarly.

## Open Questions
(none at Tier-2)

## Out of Scope
- ARM-only SoC controllers (`busses/i2c-{at91,bcm2835,brcmstb,cadence,exynos5,gpio,imx,jz4780,meson,mt65xx,mt7621,mv64xxx,mxs,nomadik,npcm7xx,nuvoton-cpld,octeon,omap,pasemi,powermac,pxa,riic,rk3x,rockchip,rzv2m,s3c2410,scmi,sh_mobile,sirf,sprd,st,stm32,sun6i,synquacer,tegra,tegra-bpmp,thunderx,uniphier,wmt,xgene,xiic,xlp9xx,xlr}.c`) — keep present compile-gated off
- 32-bit-only paths
- Implementation code
