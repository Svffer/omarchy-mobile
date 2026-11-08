<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): display driver

Status (2026-11-08): D0 under way. Simpledrm shows the console and the
shell after the initramfs's DECON trigger write (NOTES.md); brightness,
display off and 90 Hz need gs-decon on gs101 (raven's kernel #116, merged
into the local branch) and a panel driver.

## 1. Approach

Raven's ([device/raven/DISPLAY-PLAN.md](../raven/DISPLAY-PLAN.md)): gs-decon
takes the pipeline ABL left running (DECON0 -> DSIM0 -> panel, DSI command
mode, DSC) over from simpledrm; kernel #116 adds gs101's DECON0/DSIM0 nodes
and gs-decon's gs101 match data (hold the boot framebuffer, no VT switch).
The panel is a Samsung S6E3FC3 at 1080x2400, 60/90 Hz: the same DDIC and
resolution as the Pixel 7's S6E3FC3 P10, whose driver
(`panel-samsung-s6e3fc3.c`, `samsung,s6e3fc3-p10`) takes a running panel
over without touching it and only sends brightness and display on/off. It
serves oriole's bring-up as is; oriole's own variant (compatible
`samsung,s6e3fc3`, Google's `panel-samsung-s6e3fc3.c` for the 60/90 Hz
commands) follows.

## 2. Facts

| What | Value | Source |
| --- | --- | --- |
| DECON0 | `0x1c300000`, trigger `TRIG_CON` at `+0x30` reads `0x3070` at hand-off, `0x3061` after our write | NOTES 2026-11-08, raven's plan |
| Panel | `samsung,s6e3fc3`, label `samsung-s6e3fc3`, `androidboot.primary_panel_drv=panel-samsung-s6e3fc3` | stock live DT and bootconfig |
| Reset | gpp24-1 (`reset-gpios = <&gpp24 1>`; high = running) | stock live DT |
| Rails | vci = S2MPG10 LDO22 (`s2mpg10_ldo22m`), vddi = S2MPG10 LDO31 (`ldo31m`, which upstream's oriole DT names "nfc"); the shared framebuffer node holds ldo22m and bucka | stock live DT, `gs101-pixel-common.dtsi` |
| Geometry | 1080x2400; `width-mm`/`height-mm` 66/146 as the Pixel 7's P10 node [I] | stock, `gs201-panther.dts` |
| Refresh | 60 Hz as ABL leaves it; 90 Hz is the panel's maximum (stock `panel_samsung_s6e3fc3`) | stock module list |
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
| D6 | 90 Hz (oriole's own panel driver) | `tools/display-hz` near 90 | flicker | 60 Hz |

The initramfs's `SIMPLEFB_TRIG_CON` write stays: gs-decon expects the
trigger unmasked as on raven.
