<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): touch (ST FTS "FTM5" on SPI11)

Status (2026-11-08): facts gathered, no driver in the tree, nothing booted.
Oriole is the first device here with an STMicroelectronics FingerTipS
controller; raven's touch is Samsung's S6SY79x (`s6sy761`, kernel #8) on the
same SPI11, with the same reset line and rails, so raven's DT block is the
template and only the driver is new.

## 1. Facts

[V] from the stock live DT and kernel log of a retail Pixel 6 (NOTES.md
2026-11-08) and from the factory image; [P] from postmarketOS's gs101 fork.

| What | Value |
| --- | --- |
| Controller | ST FTS, Google's driver calls it `fts`, "FTM5" (`st,dchip_id = <0x36 0x48>`), driver 5.2.16.16 on stock |
| Bus | SPI11 (`spi@10D40000`, USI11), chip select 0, `spi-max-frequency` 10 MHz; `cs-gpios` gpp23-3 [P] |
| Interrupt | `st,irq-gpio` gpa9-0, active low, `IRQF_TRIGGER_LOW \| IRQF_ONESHOT` ("irq_gpio = 35" on stock) |
| Reset | gpp26-0, active low (raven's touch reset is the same pin) |
| Rails | `vdd` ldo25m (1.8 V), `avdd` ldo26m (3.3 V): raven's `gs101-raven.dts` already fixes both for its touch |
| Pins | `spi11_bus`, `spi11_cs`, `ts_irq`, `ts_reset`, and `ap2aoc`/`aoc2ap_irq` (the AoC shares the controller for always-on gestures; TS_SPI_SEL low gives the bus to the AP, as raven's `&gpp26` note says) |
| Panel geometry | `st,max-coords = <1079 2399>`, `st,udfps-coords = <540 1769>` (the under-display fingerprint spot), `st,grip_area` 15 |
| Firmware | `/vendor/firmware/ftm5_fw_o6.ftb` (stock: "Firmware in the chip matches the firmware to flash", FW VER 0049, CONFIG 0308), limits `stm_fts_production_limits_o6.csv`. Both in the factory image; never in the repository |
| Stock behaviour | the controller answered at boot; heatmap and offload (`st,touch_offload_id` "00o6") are Google extras we don't need |

## 2. Driver options

1. **Import ST's GPL driver as Google ships it** (postmarketOS does:
   `drivers/input/touchscreen/fts_spi/` in gs101-mainline/linux, 11 source
   files, about 600 KB, `TOUCHSCREEN_ST_FTS_V521_SPI`, GPL-2.0 by
   STMicroelectronics 2016). Problems: `fts_fw.h` embeds a 100 KB firmware
   array (a vendor blob inside source, which this repository never carries:
   it must go and the driver request `ftm5_fw_o6.ftb` from
   `/usr/lib/firmware/updates` instead); the code targets Android's kernel
   (sysfs/proc test interfaces, offload, heatmap) and needs porting to 7.3
   APIs. A NOT FOR MERGE bring-up import, as raven's AoC and Wi-Fi are
   today, with a plan to replace it.
2. **A lean mainline-style driver for the FTM5 SPI protocol**: input only
   (multi-touch events from the FIFO, reset, regulators, firmware check
   without flashing). Mainline's `stmfts.c` is the I2C FTS of older
   generations and does not speak the FTM5 SPI framing (the 5.2 driver's
   `ftsIO.c`: read frames with a dummy byte, "a reply can arrive on the
   second read frame", the interface dozes after 1-2 ms of silence, as
   raven's S6SY79x notes also found). More work up front, upstreamable.

Recommendation: 1 to see touch move, in a `omamobile-gs101-oriole-touch`
branch marked NOT FOR MERGE, firmware from the vendor image, the Android
extras compiled out; 2 once the protocol is understood on the phone.

## 3. Device tree (kernel, `gs101-oriole.dts`)

As raven's `&usi11`/`&spi_11` block, with the FTS node instead of the
S6SY792's: `compatible` of the driver chosen, `reg = <0>`,
`spi-max-frequency = <10000000>`, `interrupts-extended = <&gpa9 0
IRQ_TYPE_LEVEL_LOW>`, `reset-gpios = <&gpp26 0 GPIO_ACTIVE_LOW>`,
`vdd-supply`/`avdd-supply` ldo25m/ldo26m, `touchscreen-size-x/y` 1080/2400,
and raven's TS_SPI_SEL hog. Disabled by default; a `device/oriole/overlays/`
dtso turns it on for the gated boot (CONTRIBUTING §6.1).

## 4. Gates

| # | What | Pass | Rollback |
| --- | --- | --- | --- |
| T1 | kernel with the driver as a module, node disabled; boot | boots as before, module loads by hand without probing | previous images |
| T2 | overlay enables the node; `modprobe` by hand | the controller answers (chip id 0x36 0x48), firmware version read, `evtest` shows a multitouch device | `rmmod`, reboot |
| T3 | touch the screen | Hyprland moves the pointer / the shell reacts; `tools/phone-smoke` touch ok | as T2 |
| T4 | module at boot, node enabled in the DTS | the lock keypad works by finger | overlay off |

Until T3 the phone is driven over SSH; `tools/phone-mirror` (VNC) is the
way to use the shell meanwhile.
