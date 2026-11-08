<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): Bluetooth

Status (2026-11-08): **B0 done, offline**: the DTS enables the APM USI1
UART and the BCM4389 node (kernel `g1b68efca2dd9`), device-oriole 0.2.11
carries raven's helpers; the first boot with it is next.

## 1. Approach

Raven's ([device/raven/BLUETOOTH-PLAN.md](../raven/BLUETOOTH-PLAN.md)):
the BCM4389's Bluetooth core on the APM USI1 UART through hci_bcm (serdev,
`brcm,bcm4389-bt`), the patch from the owner's factory image
(`BCM.google,gs101-oriole.hcd`), the bootloader's address set when the
controller appears. Oriole's wiring is raven's (Google's
`gs101-common-bluetooth.dtsi`): the node was in the tree already, disabled
"until this kernel and its native CMU_APM baud clock path have been
qualified on Oriole"; this is that qualification.

## 2. Facts

| Fact | Evidence |
| --- | --- |
| UART: APM USI1 at `0x175b0000` (`serial_apm1`, pins `uart16_bus`), BT_REG_ON gpp16-2, device wake gpp14-3, host wake gpa6-6, 3 Mbaud [V] | `gs101-oriole.dts` (fe12fd24f), raven's plan |
| In the tree already: hci_bcm's `brcm,bcm4389-bt` (66a688283), the BT_REG_ON low pulse before power-on (75316f177), the baud rate retry after a reset (b68a344eb), btbcm's `patchram_nowait` (18e52ffa1), the CMU_APM FUNC mux fix in the clock driver (b3664451c: no `assigned-clock-parents` needed) [V] | `git log` on `oriole-touch` |
| The phone: bluez 5.87 with `btmgmt`, `bluetooth.service` enabled (inactive until an adapter appears), `/chosen/config/bt_addr` written by the bootloader (`DC:...`), the patch in `/usr/lib/firmware/updates/brcm/` [V] | SSH 2026-11-08 |
| Raven's open issue: the chip comes up silent after a small share of warm reboots (one of seven, then one of sixteen); `omamobile-bt-addr` rebinds the driver up to three times, a forced restart is the last resort [V] | raven's plan B1-B4 |

## 3. Design

- Kernel: `usi_apm1`, `serial_apm1` and the `bluetooth` child enabled in
  oriole's DTS (commit `1b68efca2dd9`); nothing else changes.
- device-oriole 0.2.11: `omamobile-bluetooth.conf` (`btbcm
  patchram_nowait=1`), raven's `omamobile-bt-addr` with its udev rule and
  `omamobile-bt-addr@.service`, `omamobile-wait-bluetooth` as an sddm
  drop-in (the shell's tile needs the adapter before the session).

## 4. Runbook

| Step | What | Pass | Stop | Rollback |
| --- | --- | --- | --- | --- |
| B0 | offline: DTS, package, build | DTB has `serial@175b0000` okay and the `brcm,bcm4389-bt` child | - | - |
| B1 | boot: hci_bcm binds by itself | `hci0`, "BCM4389C1" patch loaded at 3 Mbaud, powered, the bootloader's address, `bluetoothctl show`; Wi-Fi and the display unaffected | setup timeouts on every rebind | DTS node disabled, repush |
| B2 | scan and pair from the shell | a device paired, A2DP plays | no devices found | - |
| B3 | ten warm reboots | the adapter up each time, rebind count in the journal | silent chip | raven's forced restart |

## 5. Results

(none yet)
