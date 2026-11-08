<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): Wi-Fi, then Bluetooth

Status (2026-11-08): **W1-W4 pass by hand** (section 6): on kernel
`g7330c3817bea` every stage ran clean under the watchdog, the link came up
Gen2 x1 on the first try, brcmfmac runs firmware 20.101.90 with oriole's
CLM and scans; joining a network and W5 (the stack at boot, device-oriole
0.2.10) are next.
Bluetooth: raven's path ([device/raven/BLUETOOTH-PLAN.md](../raven/BLUETOOTH-PLAN.md)),
after Wi-Fi.

## 1. Approach

Raven's ([device/raven/WIFI-PLAN.md](../raven/WIFI-PLAN.md)): the BCM4389 on
PCIe GEN4A_1 (HSI2) through cheetah's `pcie-gs201`, `phy-gs201-pcie` and
`brcmfmac` (all in the tree; raven's own gs101 branch is not published, so
the gs101 port is redone here from Google's `pcie-exynos-gs101-rc-cal.c`),
the chip's DMA from a restricted pool, firmware and NVRAM from the owner's
factory image. Oriole and raven share the SoC, the chip, the pins and the
rails, so the SoC nodes are gs101's and the board block is oriole's.

## 2. Facts

`G/` is Google's gs101 device tree (`devices/google/gs101/dts`, LineageOS's
`gs-6.1` checkout), `S/` the stock live DT of this phone
(`~/android/reference/oriole-lineage-20261108/live.dts`), `K/` Google's
`google-modules/soc/gs/drivers/pci/controller/dwc`.

| Fact | Evidence |
| --- | --- |
| Wi-Fi on `pcie_1` at `0x14520000`: elbi `0x14520000`, PHY `0x14550000`, PCS `0x14540000`, DBI `0x14800000`, config `0x60ffe000`, I/A `0x14500000`, memory window `0x60000000` (16 MiB), IRQ GIC SPI 509 (`0x1fd`), PMU offset `0x3ec4`, `ip-ver 0x984500`, 1 lane, Gen2, I/A on, L1SS on, SICD on, dynamic PHY isolation on, MSI off, cache coherency off [V] | `S/` `pcie@14520000`, `G/gs101-pcie.dtsi` |
| WL_REG_ON `gpp14-2` (output, 2.5 mA), host wake `gpa7-0` (EINT, pull-down), device wake `gph2-5` (input, pull-down); PERST `gph2-0`, CLKREQ `gph2-1` [V] | `G/gs101-common-wlan.dtsi`, `S/` `wlan` node, `gs101-pinctrl.dtsi` |
| PCIe rails: `vreg1` S2MPG10 LDO17 (`vdd085_pcie1`), `vreg2` LDO19 (`vdd18_pcie1`), both always on in stock [V] | `S/` phandles 0x317/0x318 |
| gs101's PHY calibration differs from gs201's only in: no 38.4 MHz input clock blocks (common and per lane), its own TX pre-emphasis (`0x824 0x10`, `0x82c 0x4f`, `0x818 0x1f`, `0x820 0x00`) and RX tuning (17 writes ending `0xc08 0x05`, `0xc3c 0x04`, `0xc40 0x04`), no Auto FBB or DFE, ERIO gate bit 4 only, no PLL/bias off delay (`PCS 0x190`); the power-down writes `0x404 0xa8` and disables the PMA (our driver already did both) [V] | `diff K/pcie-exynos-gs201-rc-cal.c K/pcie-exynos-gs101-rc-cal.c` |
| gs101's `soc` node translated 1 GiB only: widened to 2 GiB for the window and config space (raven hit this) [V] | `gs101.dtsi`, raven's WIFI-PLAN |
| gs101's CMU_HSI2 has a hole in gs201's gate range (`0x20e0`): the `trace=1` CMU dump aborted on raven, so it is gs201-only now [V] | raven's WIFI-PLAN |
| Firmware on the phone: `/usr/lib/firmware/updates/brcm/brcmfmac4389c1-pcie.google,gs101-oriole.{bin,txt,clm_blob}` (BCM4389C1 20.101.90, ROW NVRAM and CLM, SKU GB7N6) and `BCM.google,gs101-oriole.hcd` [V] | NOTES 2026-11-08, `ls` on the phone |
| Modules in the kernel: `pcie-gs201`, `pcie-gs201-sysmmu`, `phy-gs201-pcie`, `brcmfmac` (+ `-wcc`), `hci_uart`, `btbcm`; `CONFIG_DMA_RESTRICTED_POOL=y`, `CONFIG_PCI_PWRCTRL=y` [V] | `out/.config`, the phone's module tree |
| The phone has `/dev/watchdog0` (s3c2410), `systemd-run` and pstore, so `tools/wifi-w1-capture --watchdog` works as on cheetah; its device checks (`14520000.pcie`, `14550000.phy`) match gs101's addresses [V] | `ls` on the phone, the tool |

## 3. Design

- Kernel (`oriole-touch`): `google,gs101-pcie-phy` and `google,gs101-pcie`
  as match data of cheetah's drivers (PR candidate: the gs101 tables, the
  CMU dump gated); `gs101.dtsi` gets `pcie_1_phy` and `pcie_1` (disabled,
  gs101's clocks and IRQ) and the 2 GiB `soc` ranges (PR candidate);
  oriole's DTS enables them with the Wi-Fi child, `wlan_pwr` (WL_REG_ON
  through pwrctrl, 200 ms), the wake pins, a 16 MiB `restricted-dma-pool`
  below 4 GiB and the rails always on (NOT FOR MERGE until W3).
- Device package: `omamobile-wifi.conf` blacklists `pcie_gs201`,
  `phy_gs201_pcie`, `pcie_gs201_sysmmu` and `brcmfmac` for the bring-up;
  after W4 it becomes raven's: only `pcie_gs201_sysmmu` stays out, and
  NetworkManager with `usb0` unmanaged (already in the package's
  `10-omamobile-usb.conf`).

## 4. Risks (raven's, from cheetah)

- **PCIe ELBI while the PHY is isolated hangs the SoC**; never read
  controller registers from userspace; every load by hand runs under
  `tools/wifi-w1-capture --watchdog` with the log streamed to the laptop.
- **The HSI2 SysMMU/S2MPU select by VID and PCIe shares VID 0 with UFS**:
  keep the restricted DMA pool, never enable VID 0 translation, do not load
  `pcie_gs201_sysmmu`.
- Slot B spends an ABL retry per boot ([slot-b-retries]); re-arm before a
  push. A watchdog reset is a warm reset into the same slot.

## 5. Runbook

| Step | What | Pass | Stop | Rollback |
| --- | --- | --- | --- | --- |
| W0 | offline: drivers, nodes, DTS, package, build | builds; DTB has `pcie@14520000` okay, `wifi@0,0`, the pool | - | - |
| W1 | boot `g7330c3817bea` with the blacklist; `tools/wifi-w1-capture --watchdog 1` (clocks only) | probe reaches stage 1, no hang | hang | watchdog reset; keep blacklisted |
| W2 | stages 2-4: PHY isolation off, PHY config (PLL lock), controller setup; WL_REG_ON low | PLL lock 1, no hang | abort, hang | pstore capture, stop |
| W3 | stage 5: WL_REG_ON, PERST, training | link up Gen2 x1, `14e4:4441` in `lspci` | no link in 3 tries | PERST low, unload |
| W4 | `--brcmfmac`: firmware 20.101.90 from the pool; NetworkManager joins a network | `wlan0`, an IP, a ping over Wi-Fi | firmware timeout | unload |
| W5 | at boot: drop the blacklist (keep `pcie_gs201_sysmmu`), one clean boot | Wi-Fi up without SSH help | - | blacklist again |

## 6. Results

**2026-11-08, kernel `g7330c3817bea` (sixteenth boot), device-oriole 0.2.9:**

- **Boot** with the blacklist: no PCIe or Wi-Fi module loaded by itself,
  `14520000.pcie` and `14550000.phy` present, the 16 MiB restricted pool at
  `0xff000000`, `vdd085_pcie1` 0.85 V and `vdd18_pcie1` 1.8 V on,
  `wlan-reg-on` off, the display still at 90 Hz. `systemctl
  is-system-running` `running`.
- **W1** (`tools/wifi-w1-capture --watchdog 1`): CMU_HSI2 option
  `0xf1000000` (automatic gating on), the seven controller clocks enabled,
  UFS still reading.
- **W2** (stage 2): PHY out of isolation (PMU `0x3ec4`), power-down writes,
  first PMA/PCS reads, first ELBI writes and reads: `QCH_SEL 0x577`,
  `SOFT_RESET 0xf`, `DEVICE_TYPE 0x4`. No hang.
- **W2** (stage 3): the gs101 PMA tables (common, 1 lane), resets released,
  external XO, **PLL lock 1, CDR lock 1, OC done 1** (`PMA 0x3f0 0xf`,
  `0xfc0 0xf8`, `PCS 0x188 0x2`), the I/A sequencer programmed.
- **W2** (stage 4): DBI readable, `ID 0xa5a5144d`, DWC version
  `0x3531302a`, D-state 0, LTSSM 0.
- **W3** (stage 5): WL_REG_ON through pwrctrl, PERST released, **link up on
  try 1, Gen2 x1**; `lspci`: `01:00.0 Network controller: Broadcom
  [14e4:4441] (rev 03)`.
- **W4** (`--brcmfmac 5`): PCIe protocol version 7, **Firmware BCM4389/3
  version 20.101.90 (c1 release2) FWID 01-3f632e0d**, CLM `Google.O6sub6`
  (2023-01-13), WLC 14.4; `wlp1s0` managed by NetworkManager (`usb0`
  unmanaged, the USB route stays), a scan listed 8 networks on 2.4 and
  5 GHz. The MAC is random (`be:d7:...`), as on raven. No saved Wi-Fi
  profile on this phone yet: the owner joins from the shell.
- The three watchdog disarms each printed "busy" once and succeeded on the
  tool's retry. Logs: `logs/wifi-w1-20261108-1409*-s{1,2,3,4}.log`,
  `-1410*-s5.log` (5731 lines with brcmfmac's debug on; `debug` was reset
  to 0 afterwards).
