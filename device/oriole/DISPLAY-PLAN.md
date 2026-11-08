<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): display driver

Status (2026-11-08): **D1-D6 pass** (NOTES.md): gs-decon drives DSI-1 at
90 Hz with the GPU rendering to scanout, brightness from the shell through
the S6E3FC3 driver (gs-dsim-cmd needs `allow_tx=1`), display off and wake by
the lock screen (the rate drops to 60 Hz while the screen is off). Two NOT
FOR MERGE gs-decon changes were needed (oriole's modes in the gs101 table;
padded rows, since 1080 * 4 is not 64-byte aligned). D6 ran on modules
swapped in by hand with the P10 compatible still in the device tree; the
build with `samsung,s6e3fc3` in the DT (device-oriole 0.2.8) is the next
boot.

## 1. Approach

Raven's ([device/raven/DISPLAY-PLAN.md](../raven/DISPLAY-PLAN.md)): gs-decon
takes the pipeline ABL left running (DECON0 -> DSIM0 -> panel, DSI command
mode, DSC) over from simpledrm; kernel #116 adds gs101's DECON0/DSIM0 nodes
and gs-decon's gs101 match data (hold the boot framebuffer, no VT switch).
The panel is a Samsung S6E3FC3 at 1080x2400, 60/90 Hz: the same DDIC and
resolution as the Pixel 7's S6E3FC3 P10, so one driver
(`panel-samsung-s6e3fc3.c`) serves both as variants: `samsung,s6e3fc3` is
oriole's (Google's `panel-samsung-s6e3fc3.c`: its cold init, 67x148 mm),
`samsung,s6e3fc3-p10` the Pixel 7's. The driver takes a running panel over
without touching it and sends brightness, display on/off and the rate
switch (`F0 5A 5A; 60 00|08; F7 0F; F0 A5 A5`, 60 or 90 Hz, taken at the
next frame) for gs-dsim-cmd's `drm_panel_set_refresh_rate()`.

90 Hz is raven's mechanism ([device/raven/DISPLAY-PLAN.md](../raven/DISPLAY-PLAN.md)
D5.2) with oriole's numbers: gs-dsim-cmd recognises the bootloader's 60 Hz
timers and switches them with the panel; gs-decon's gs101 oriole mode table
gets the 90 Hz mode, and it now offers the highest rate of its table the
DSIM can switch to (90 here, 120 on raven) instead of 120 only. The MIF and
DISP floors are raven's 120 Hz ones; on oriole both clocks sit above them
(2028 and 400 MHz) and 1080x2400 at 90 Hz needs less than half, so they
change nothing.

## 2. Facts

| What | Value | Source |
| --- | --- | --- |
| DECON0 | `0x1c300000`, trigger `TRIG_CON` at `+0x30` reads `0x3070` at hand-off, `0x3061` after our write | NOTES 2026-11-08, raven's plan |
| Panel | `samsung,s6e3fc3`, label `samsung-s6e3fc3`, `androidboot.primary_panel_drv=panel-samsung-s6e3fc3` | stock live DT and bootconfig |
| Reset | gpp24-1 (`reset-gpios = <&gpp24 1>`; high = running) | stock live DT |
| Rails | vci = S2MPG10 LDO22 (`s2mpg10_ldo22m`), vddi = S2MPG10 LDO31 (`ldo31m`, which upstream's oriole DT names "nfc"); the shared framebuffer node holds ldo22m and bucka | stock live DT, `gs101-pixel-common.dtsi` |
| Geometry | 1080x2400; `width-mm`/`height-mm` 66/146 as the Pixel 7's P10 node [I] | stock, `gs201-panther.dts` |
| Refresh | 60 Hz as ABL leaves it; 90 Hz is the panel's maximum (`60 08` + `F7 0F`, stock `panel_samsung_s6e3fc3`) | stock module list, G/ `panel-samsung-s6e3fc3.c` |
| Link | 4 lanes at 1120 Mbit/s (stock DT `dsim-modes/1080x2400 hs-clk = <0x460>`), word clock 70 MHz, DSC 3:1 | stock live DT, stock dmesg "hs_clk is changed to 1120" |
| DSIM timers at 60 Hz (ABL) | `UNDERRUN_CTRL 0x1da7`, `CMD_TE_CTRL1 0x2b343138` (protect 0x2b34, timeout 0x3138), `CMD_TE_CTRL0 0x40` | gs-dsim-cmd's dump |
| DSIM timers at 90 Hz | `UNDERRUN_CTRL 0x0e9d` (0x1da7 less 3850 units: the 5.50 ms a 90 Hz frame saves at 70 MHz / 100; ABL's value is Google's `dsim_calc_underrun` with `te_idle_us` 1000), `CMD_TE_CTRL1 0x1ccd20d0` (ABL's pair at two thirds) | derived as raven's 120 Hz pair; D6 |
| Scanout pool | five 1080x2400 XRGB8888 buffers, 0x3400000, in raven's alloc range | raven's DT |

## 3. Runbook (each step gated, module blacklisted at boot until D3)

| Step | What | Pass | Stop | Rollback |
| --- | --- | --- | --- | --- |
| D0 | offline: PR #116 merged, oriole's DT block, build | builds, DTB has decon0/dsi0/panel | - | - |
| D1 | boot with `gs_decon`, `gs_dsim_cmd`, `panel_samsung_s6e3fc3` blacklisted; then `modprobe gs_decon stage=0` | logs DECON/DPP0 running, 1080x2400, DPP0 address = the boot framebuffer | abort or hang | reboot |
| D2 | `stage=1` | frame-start IRQ counts near 60 Hz | IRQ storm | rmmod, reboot |
| D3 | `stage=2`: KMS takes over, `systemctl restart sddm` | Hyprland on the gs-decon card (DSI-1), simpledrm gone, no phantom output | black past 30 s | rmmod, reboot to simpledrm |
| D4 | panel driver: `brightnessctl` | brightness changes | no answer | as D3 |
| D5 | DPMS off/on from the shell | panel off and back | panel stays off | reboot |
| D6 | 90 Hz (the panel's rate switch, oriole's DSIM timers, the 90 Hz mode) | gs-decon `stats` near 90 Hz, no flip timeouts, DSIM `INTSRC` bit 19 clear | flicker, underrun | `options gs_decon max_refresh=60` |

The initramfs's `SIMPLEFB_TRIG_CON` write stays: gs-decon expects the
trigger unmasked as on raven.

`tools/display-hz` reads gs201's DECON at `0x1c240000` and needs `devmem`;
on oriole DECON0 is at `0x1c300000` and the phone has no `devmem`. The rate
comes from gs-decon's sysfs `stats` (`frame_start` over a timed window,
`refresh_hz`), and the DSIM registers from python's `mmap` of `/dev/mem` at
`0x1c2c0000` (`+0x34` underrun, `+0x88` TE_CTRL1, `+0x50` INTSRC).
