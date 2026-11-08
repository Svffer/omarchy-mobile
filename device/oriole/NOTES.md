<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole) bring-up notes

Findings for the Pixel 6 port, newest first. The port is a community effort tracked in [issue #2](https://github.com/derekross/omarchy-mobile/issues/2); the 6 Pro (raven) and 6a (bluejay) share it. Facts are marked [V] (checked, with the source) or [I] (an inference or a recovered report still to check). The October 14–16 phone facts below came from read-only queries of a stock phone; later tests booted Linux on a Pixel 6.

Sources:

- `F/`: Google's public factory image `oriole-cp3a.261005.002.a1` ([developers.google.com/android/images](https://developers.google.com/android/images#oriole)), read with HTTP range requests; facts only, no files kept in the repository.
- `U/`: upstream Linux v7.0, `arch/arm64/boot/dts/exynos/google/gs101*.dts(i)`.
- `P/`: postmarketOS pmaports [`device/testing/device-google-oriole`](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/tree/main/device/testing/device-google-oriole), [`soc-google-gs101`](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/tree/main/device/testing/soc-google-gs101) and [`linux-postmarketos-gs101`](https://gitlab.postmarketos.org/postmarketOS/pmaports/-/tree/main/device/testing/linux-postmarketos-gs101) (7.0.6), which builds [gs101-mainline/linux @ d28cfa42](https://github.com/gs101-mainline/linux/tree/d28cfa4275b95e93f4b7a1e743de31b723bd21b8).

## 2026-11-08: the Mali-G78 under panfrost (GPU), and Plymouth

Fourth boot of slot B, pushed from the Arch root with `bin/omamobile push`
(images and modules over SSH, read back): kernel
`7.3.0-rc6-omamobile-push1-g08d671cd4e0d` from a local branch on
`omamobile` whose one commit enables `&gpu` in `gs101-oriole.dts` on the
S2MPG10/11 rails raven uses (`mali-supply` buck2s, `g3d-l2-supply` buck8s,
`g3d-m-supply` ldo1s, `int-m-supply` buck7m), plus device-oriole 0.2.1 and
the Plymouth command line. [V] on the phone:

- **Plymouth**: with `quiet splash plymouth.graphical plymouth.force-scale=1
  vt.global_cursor_default=0` and the `plymouth quit --retain-splash`
  drop-in, `plymouth-start` and `plymouth-quit` are active, no segfault,
  `systemctl is-system-running` says `running` with no failed unit. SSH 6 s
  into the boot.
- **panfrost deferred** on `get clock failed -517`: the gpu node's clock is
  `GS101_CLK_ACPM_GPU` from the ACPM clock provider, `clk_acpm`
  (`CONFIG_EXYNOS_ACPM_CLK=m`), which PR #222's package blacklisted and
  blocked with `install /bin/false`. `modprobe --ignore-install clk_acpm`
  and the GPU probed at once: `mali-g78 id 0x9202 major 0x0 minor 0x1`,
  `shader_present=0x779fff l2_present=0x1` (MP20), `renderD128` on
  `1c500000.gpu`, clock 151 MHz (no DVFS yet: raven's kernel #189 scales it
  through ACPM). device-oriole 0.2.2 drops `clk_acpm` from the blacklist.
- **Hyprland on the GPU**: after `systemctl restart sddm`, Hyprland holds
  `renderD128` open (7 fds) next to `card0` (simpledrm scanout); the earlier
  "failed to create dri2 screen" warnings are gone, only panvk's
  "Mali-G78 not supported" remains (Vulkan is not used). The owner's lock
  screen came back after the restart.
- Still no thermal zones (no `GS101_ACPM_THERMAL` driver probing here) and
  no battery (`max17042_battery` blacklisted by the package; raven loads it).

## 2026-11-08: the Arch root on userdata (M2): installed with tools/install-phone

A contributor build, all local (RELEASES §10): `keys/oriole` from
`tools/local-key-init`, the published packages from `tools/fetch-published`
(both channels carry the same 45; the pinned `aquamarine` 0.16.0 is in
neither, so it was fetched from `rootfs/packages.pinned`'s URL, checked
against the listed SHA-256 and imported), `omamobile-device-oriole` 0.2.0
built from PR #222's tree, and `KERNEL_OUT=~/omamobile-kernel/out
tools/build-rootfs --local oriole` (28 minutes; every image check passed).
`tools/install-phone --reinstall --bootloader-slots-match
slider-17.0-15199431` from the running initramfs (`bin/omamobile fastboot`
first): the dry run printed exactly six commands; the real run wrote slot
B's three images and `userdata`, set B active and rebooted. [V] on the
phone above:

- **First boot**: SSH at 62 s, hostname set, root on `/dev/sda27` (ext4,
  grown to 225.7 GiB), the user created with the wheel, audio, input and
  video groups; `omamobile-firstboot` finished at about 190 s. `systemctl
  is-system-running` says `degraded` for two units: `plymouth-start`
  (plymouthd segfaults "immediately after starting to daemonize", raven's
  `console=null` crash: `plymouth.graphical` on the command line is the
  fix, not yet applied here) and `systemd-firstboot`.
- **The session**: SDDM autologin into `uwsm start ... Hyprland`; Hyprland
  and quickshell (the Omarchy shell) run on simpledrm's `card0` with
  software rendering: the published kernel enables no GPU node for oriole,
  so Vulkan finds no device, zink fails and Mesa falls back. **The owner
  sees the shell's lock screen on the panel** (2026-11-08, after the
  Node.js install completed over the USB link): Display is `partial`
  (software rendering, no brightness or refresh-rate control) and Storage
  `works` in the matrix.
- **Storage guard under Arch**: still exactly `boot_b`, `vendor_boot_b`,
  `dtbo_b` and `userdata` writable.
- **pacman**: the `[omamobile]` channel is `local`; the phone refused the
  index until the owner's key was added (`pacman-key --add`, `--lsign-key`),
  as RELEASES §10 says.
- **Internet over USB**: the phone routes through 172.16.42.2 with DNS
  1.1.1.1 (the device package's networkd file). On an Omarchy laptop the
  documented NetworkManager `shared` profile needs `dnsmasq` installed, and
  ufw's `DEFAULT_FORWARD_POLICY=DROP` still blocks the forwarded TCP and
  UDP (ICMP got through): `ufw route allow in on <usb-interface>` opens it.
  Without that, firstboot's Omarchy step defers the Node.js install
  ("Node.js install deferred (no network)") and the rest completes.

## 2026-11-08: the screen console (display, step 1) and the console font

Second and third boots of slot B, one change each, flashed with
`bin/omamobile fastboot` and `flash` from the running initramfs:

- **`SIMPLEFB_TRIG_CON="0x1c300030 0x3061"` with `INITRAMFS_FB_WAIT=1`** [V]:
  after the init's write the register reads `0x3061`, fbcon binds to
  simpledrm's `fb0` (`/dev/fb0`, 1080x2400, 32 bpp, stride 4320) and the
  owner sees the init's lines on the panel: black screen, text at the top.
  The panel refreshes, so ABL's TE mask is lifted the same way as on raven
  and cheetah. A dump of `/dev/fb0` over SSH shows the same lines (the
  splash is drawn first and the console's lines land over its top). SSH 2 s
  into the boot.
- **`fbcon=font:TER16x32`** on `VENDOR_CMDLINE` [V]: the kernel command line
  carries it (ABL keeps the vendor_boot line), the console is twice the
  size. Terminus 16x32 is the largest built-in font; a bigger on-screen log
  needs a userspace console later.

## 2026-11-08: first light on the published kernel (M1 done)

Slot B of the Pixel 6 above boots the published kernel from `omamobile`
(`7.3.0-rc6-omamobile-push1-g25368bff83ce`, kernel-2026.11.01-41, built with
the pinned toolchain), with the shared initramfs and the descriptor as
committed today (no `SIMPLEFB_TRIG_CON`, no `USB_SWITCH`). Slot A holds the
owner's LineageOS 24.0 (the phone's OTA had left A without a system; the same
OTA sideloaded once more put it there and made A current and successful).
Images from `DEVICE=oriole bin/omamobile images`; `bin/omamobile fastboot`
then `flash` wrote `boot_b`, `vendor_boot_b` and `dtbo_b`, set B active and
rebooted. All [V] on that phone; captures in `~/android/reference/oriole-m1-20261108/`.

- **Sign of life**: the laptop saw a CDC NCM + ACM gadget ("Pixel 6",
  "Omarchy Mobile") and SSH answered at 172.16.42.1. `uname -a`:
  `7.3.0-rc6-omamobile-push1-g25368bff83ce`, compatible `google,gs101-oriole
  google,gs101`. Up 566 s before the deliberate reboot below, no watchdog
  reset. Both cluster watchdogs appear (`/dev/watchdog0`, `watchdog1`).
- **USB: upstream's description works on gs101 (raven's U1, USB-PLAN
  section 3, passes here)**: with `omamobile`'s `4d015fb82` and no override,
  UDC `configured`, `high-speed`, `11110000.usb-role-switch` reads `device`.
  `dmesg` shows the TCPC/PHY/dwc3 dependency cycles fixed by the kernel, no
  USB error. The host side needs the address by hand or a NetworkManager
  profile with `ipv4.method manual` (the documented `shared` needs dnsmasq).
- **Storage guard**: allowlist `boot_b vendor_boot_b dtbo_b userdata`; 53
  block nodes, exactly those four writable, `/dev/sda` removed, `sdb`-`sdd`
  read-only. Nothing mounted.
- **PMIC and framebuffer**: `exynos-mailbox`, `acpm-protocol` and `sec-acpm`
  are built in on this kernel (the initramfs says so), 72 regulators
  registered, `fb0` is `simpledrmdrmfb` and `/dev/dri/card0` exists. The
  screen stays on ABL's logo: DECON's trigger is still masked (below).
- **Reboot path**: the three MAX77759 modules loaded from the initramfs
  (`max77759`, `nvmem-max77759`, `nvmem-reboot-mode`, `reboot-mode` under
  `/sys/bus/nvmem/devices/max77759-nvmem.3.auto`); `reboot-to bootloader`
  reached fastboot in 6 s; `fastboot --set-active=b` and `fastboot reboot`
  brought SSH back 16 s after the command, `/proc/uptime` 2.3 s at the first
  answer. In fastboot after the first Linux boot: `slot-successful:b: no`,
  `slot-retry-count:b: 2` (ABL does not seem to spend a retry on a
  `reboot-to bootloader`), slot A `successful: yes`. Nothing marks B
  successful (no `omamobile-bootctl` for oriole, on purpose: PR #222 review).
- **DECON's trigger, read before any write**: `0x1c300030` reads **`0x3070`**
  at hand-off, `0x1c300000` `0x06010000`, `0x1c300004` `0x3`. `0x3070` is the
  value cheetah's DECON0 `TRIG_CON` reads before the initramfs writes
  `0x3061` (cheetah's DISPLAY-PLAN: TE bits plus `HW_TRIG_MASK_DECON`,
  `HW_TRIG_EN` clear); raven's reads `0x3071` dark and `0x3061` after the
  write (POWER-PLAN). So `SIMPLEFB_TRIG_CON="0x1c300030 0x3061"` is the
  next, single change, with `INITRAMFS_FB_WAIT=1`.
- **Kernel log**: 418 lines, no oops, BUG or WARN. Three notes:
  `s2mpg10-gpio`/`s2mpg11-gpio: Failed to locate of_node` (the GPIO cells
  without DT nodes, as on raven), a duplicate `debug_kinfo_reserved@fd3fe000`
  / `debug-kinfo-reserved@fd3fe000` reserved-memory overlap (two spellings of
  one node: ABL's injected one and the DT's), and "unable to open an initial
  console" (`console=null`).
- **Not there yet**: no thermal zones and no power supply (the published
  kernel has no gs101 ACPM thermal or MAX1720x gauge for oriole; the kernel
  fragment's symbols are warnings here), `tools/phone-smoke` needs bash and
  so the Arch root (stage 2). 8 CPUs online, 7.8 GiB RAM.

Next, one change per boot: the DECON trigger (screen console and splash),
then the Arch root on `userdata` (`tools/build-packages`, `build-rootfs`,
`install-phone`), then touch (ST FTS) and the panel (S6E3FC3) on the kernel
side.

## 2026-11-08: stage 0 closed on an unlocked Pixel 6 (LineageOS, root) and its factory image

Read-only queries of a retail Pixel 6 (MP1.0, SKU GB7N6, bootloader unlocked)
running a self-built LineageOS 24.0 userdebug (`adb root` works, as
CONTRIBUTING §5.1 allows): `bin/omamobile capture` over adb and in fastboot,
`/sys/firmware/devicetree/base`, and `dd` reads of the `vbmeta_*` headers and
the `boot_*` footers. Nothing was written. The facts below are [V] from that
phone (`O/`) or from Google's factory image `oriole-cp2a.260705.006` (`F2/`,
SHA-256 `ed43367c…`, the build the phone's bootloader and radio came from)
unless marked. Captures stay in `~/android/reference/`, never here.

### The phone (O/)

- **`soc_id` is `<0x9845001 0x0>`, B0**, the first `SOC_IDS` value and what
  the published `gs101-oriole.dts` carries. The live DT's model is
  `Oriole DVT`, root compatible `google,GS101 Oriole`, `google,GS101`.
- **Board entry 0** (`ro.boot.dtbo_idx` 0 = 0x20304/0x10000), `hw-revision`
  MP1.0, `androidboot.hw.soc.rev` 3, `primary_panel_drv`
  `panel-samsung-s6e3fc3`. Confirms the 2026-10-16 retail phone.
- **Bootloader `slider-17.0-15199431`, baseband
  `g5123b-145971-260414-B-15201439`**: F2/'s `android-info.txt` requires
  exactly these two. The ABL reports one `version-bootloader` (no per-slot
  variables), as raven's does. Both slots `slot-successful: yes`,
  `unbootable: no`, retry 2; current slot b (a LineageOS OTA moved it).
- **Partitions**: `has-slot` boot, vendor_boot, dtbo, vbmeta, vbmeta_system,
  vbmeta_vendor, bootloader; no `init_boot`, no `vendor_kernel_boot`. Sizes:
  boot and vendor_boot 0x4000000, dtbo 0x1000000, vbmeta 0x10000, devinfo
  0x2000, userdata 0x395c215000. `BOOT_IMAGES` and `UFS_RW` stand.
- **RAM**: four 2 GiB banks at 0x80000000, 0x880000000, 0x900000000,
  0x980000000 (`/memory@*`, injected by ABL).
- **ABL appends `console=null`** and `exynos_drm.panel_name=samsung-s6e3fc3`
  to `/chosen/bootargs`, after the DT's own line: `console=` we pass is
  dropped, as on raven (raven's descriptor and plymouth note apply).
- **AVB on the phone**: `vbmeta_a` and `vbmeta_b` are stock's layout (chain
  `boot` at location 2, `vbmeta_system` 1, `vbmeta_vendor` 3; `dtbo`,
  `vendor_boot`, `vendor_dlkm` as hash descriptors) with **flags 3**: the
  LineageOS install disabled verification, so this phone would also boot a
  footer-less `boot_b`. That is this phone's state, not the descriptor's
  assumption. Both `boot_a` and `boot_b` carry LineageOS's footer (SHA256_RSA2048,
  rollback index 1790812800, security patch 2026-07-05).
- **Reserved memory against `/proc/iomem`**: every `no-map` node of the live
  DT is a `reserved` range in iomem: gsa 0x90200000+0x400000, tpu_fw
  0x93000000+0x1000000, aoc 0x94000000+0x3000000, xhci_dma 0x97000000+0x400000,
  sec_dram 0xb7200000+0x8e00000, seclog 0xc3000000+0x80000, header
  0xd8100000+0x10000, log_kevents 0xd8110000+0x700000, log_s2d
  0xd8810000+0x1800000, log_array_reset 0xda010000+0xa00000, log_bcm
  0xdd610000+0x400000, log_itmon 0xdda10000+0x10000, sec_pt 0xe0000000+0x2500000,
  cp_rmem_1 0xe8000000+0x1c00000, cp_rmem_2 0xee000000+0x400000, cp_rmem
  0xf0000000+0x800000, cp_msi_rmem 0xf6200000+0x1000, abl 0xf8800000+0x1000000,
  dss_log 0xfd3f0000+0xe000, debug_kinfo 0xfd3fe000+0x1000, ramoops
  0xfd3ff000+0x400000, bldr_log 0xfd800000+0x100000, suspend_diag
  0xfd90c000+0x1000, ect_binary 0x90000000+0x3a000 (`#address-cells` 2,
  `#size-cells` 1: 12-byte `reg` entries). iomem also reserves ranges with no
  DT node: 0xfa3f0000-0xfd3effff (ABL's carve-out; the board DTS's
  `cont_splash_mem` at 0xfac00000 lies inside it), 0xfec00000-0xffbfffff,
  0xffdf0000-0xffffffff, 0x9cef80000-0x9d7ffffff and 0x9d807f000-0x9d816ffff
  [I: ABL's or Trusty's, to name before any of them is used].
- **Hardware the stock-derived kernel (6.1.157) binds**, for the plans:
  panel Samsung S6E3FC3 (`panel_samsung_s6e3fc3`, DSI hs_clk 1120, DECON0/2);
  touch ST FTS "ftm5" on `spi11.0` (driver 5.2.16.16, firmware `ftm5_fw_o6.ftb`,
  limits `stm_fts_production_limits_o6.csv`, `irq_gpio` 35); haptics Cirrus
  **CS40L25A rev B1** with the `cs40l20.bin` wavetable (raven's driver and
  files); speakers **two CS35L41 B2 on `spi13.0`/`spi13.1`** with
  `cs35l41-dsp1-spk-prot.{wmfw,bin}` and `R-…bin`, no fast_switch files (as
  raven); Wi-Fi **BCM4389** (chipid 0x4389, buscorerev 72) on PCIe; Bluetooth
  `btbcm`/`hci_uart`; modem **s5123 over PCIe** (CPIF-NR1, boot image over
  SPI bus 9); AoC `aoc.bin` 13820614-polygon, 69 services; PMIC S2MPG10 and
  S2MPG11; charger MAX77759, gauge `max1720x`; NFC ST54J; fingerprint Goodix;
  GPU `mali_kbase`. So oriole differs from raven in the panel, the touch
  controller and the board glue; the rest of raven's gs101 branches apply.

### The factory image (F2/)

- **`vbmeta.img`**: SHA256_RSA4096, rollback index 1783209600 (2026-07-05),
  flags 0, chains `boot` (location 2), `vbmeta_system` (1), `vbmeta_vendor`
  (3); hash descriptors for abl, bl1, bl2, bl31, dtbo, gsa, ldfw, pbl, tzsw,
  vendor_boot, vendor_dlkm. **`boot.img` carries a signed footer**
  (SHA256_RSA4096, rollback index 1783209600, original size 16629760, padded
  to 0x4000000, `security_patch` 2026-07-05); `vendor_boot` (31936512) and
  `dtbo` (1860257) carry unsigned footers (NONE). So, as on bluejay, slot B's
  `boot` needs a footer and the other two don't: `device.conf` now has
  `AVB_FOOTERS="boot:0x4000000"`, `AVB_ROLLBACK_INDEX=1783209600`,
  `AVB_SECURITY_PATCH=2026-07-05` and `OS_PATCH_LEVEL=2026-07`, this build's
  own values, never higher. A phone on a newer stock build has that build's
  index (bluejay's notes read 1791158400 / 2026-10-05 from CP3A.261005.002.A1,
  the same build id as oriole's current image [I]). On an unlocked phone a
  lower index is a verification error libavb continues past; a missing
  footer is not.
- **`dtbo.img`**: 18 entries, page 4096, as 2026-10-14 read from CP3A;
  entry 0 is 0x20304/0x10000 with `samsung,s6e3fc3` and `st,fts` (the phone's
  entry), 1-5 the other oriole boards, 6-11 raven, 12-17 development boards.
- **`vendor_boot`**: header v4, page 0x800, two DTBs (`soc_id` 0x9845000 and
  0x9845001, compatible `google,gs101`, no model), stock `/chosen/bootargs`
  as 2026-10-14 recorded. `vendor.img` is ext4 (the debugfs path).
- **`tools/extract-vendor-firmware -d oriole` and `tools/extract-vendor-chroot
  -d oriole`** (added today: raven's map with the board `google,gs101-oriole`)
  stage BCM4389C1 Wi-Fi 20.101.90 with the ROW NVRAM and CLM (SKU GB7N6 is a
  ROW SKU, as in husky's table), the BCM4389C1 Bluetooth patch, `aoc.bin`,
  the CS35L41 protection firmware with distinct top and bottom tunings, the
  CS40L25A clab firmware and wavetable, the AoC sensor registry
  (`oriole_dvt.reg` and the other revisions), the gnss chroot (262 files) and
  the modem chroot (893 files, `fstab.gs101`). `tools/phone-provision -d
  oriole` runs raven's steps with the device's own names.

### The descriptor for the first boot of slot B

Three changes from raven's experience, before any oriole boot, each [I]
until oriole's first light says otherwise:

- **No `earlycon`** on `VENDOR_CMDLINE`: raven's ABL passes the vendor_boot
  command line on, and earlycon's writes to the retail phone's unclocked
  UART hung the SoC before any console existed (raven's BRINGUP-M1 section
  7, tries 1-3). Oriole's ABL behaves the same way around the command line
  (it appends `console=null` and the panel name, O/). `regulator_ignore_unused`
  joins, as postmarketOS boots both phones with it.
- **`INITRAMFS_MODULES`**: raven's six (the PMIC over ACPM, the MAX77759
  core, nvmem and reboot-mode), so the simple-framebuffer gets its rails
  and `reboot-to bootloader` reaches fastboot. Oriole has the same
  S2MPG10/11 and MAX77759 (O/ dmesg).
- **`NV_PARTS` and `NV_BACKUP_EXTRA`**: cheetah's set, all seven present on
  this phone (O/), so `tools/nv-backup` works from the first Linux boot. A
  baseline copy of the seven was taken read-only from LineageOS over adb
  root on 2026-11-08 (`~/android/reference/oriole-backup-20261108`,
  SHA256SUMS in phone-provision's format); `efs`, `efs_backup`,
  `modem_userdata` and `persist` were mounted at the time, so the copy
  after the last Android boot, from Linux, is the one provisioning uses.

Left unset on purpose: `SIMPLEFB_TRIG_CON` (gs101's DECON register is read on
a booted oriole before any write), `USB_SWITCH` (raven's switch read closed at
hand-off) and `INITRAMFS_FB_WAIT`. The published kernel's USB description is
upstream's (`dr_mode = "otg"`, `orientation-switch`); raven needed a NOT FOR
MERGE peripheral override before `omamobile` gained `4d015fb82` (the PHY
keeps VBUS valid until Type-C reports), and whether upstream's now enumerates
is raven's open U1 (USB-PLAN section 3). Oriole's first boot is that test for
gs101: no USB device on the laptop means U1 failed here too, and the next
boot carries the override as a local kernel change.

### Install consequences

- `tools/install-phone` wants the current slot `a`: `fastboot --set-active=a`,
  one Android boot from A, then the NV backup (an Android boot changes `efs`
  and `persist`, raven's 2026-10-15 note), then the install. Never take an
  Android OTA afterwards: it writes slot B.
- The ABL reports one bootloader version, so `install-phone` takes
  `--bootloader-slots-match slider-17.0-15199431` once the other slot's has
  been read in fastboot.
- The factory image for the firmware is the phone's own build
  (`oriole-cp2a.260705.006`), not the newest: the AoC firmware is
  GSA-authenticated and the modem's vendor tree pairs with the radio in
  `modem_a`, both from the bootloader package ABL and the radio came with.

## 2026-10-31: build provenance and recovery limits

This describes the unpublished stack in the October 30 smoke report. It is
not a reproducible build of the public Oriole descriptor or an installer
qualification. The records and recovered artifacts were audited offline;
this audit did not reflash or inspect the phone's current partitions.

- **Kernel** [V, recovered source metadata]: the base is
  [`omamobile` commit `4bd35d34abb9`](https://github.com/derekross/linux-omarchy-mobile/commit/4bd35d34abb92a745479a2de62816b28304b4449).
  The native build used a detached worktree at that commit with unpublished
  local changes, producing
  `7.3.0-rc6-oriole-hsi1-late-dev0-g4bd35d34abb9-dirty`.
  There is no published branch containing that complete integration tree.
- **Packages** [V, recovered sources and build receipts]: the mobile source
  was local branch `pixel6-bringup`, based on
  [`1155a7c352ed`](https://github.com/derekross/omarchy-mobile/commit/1155a7c352edf6313fb4e2ff02f054c9cb1902b6),
  with unpublished changes and separate package-source snapshots. The
  `omamobile-device-oriole 0.1.9-2` source matches all 24 files in its build
  receipt; its archived package and the GPU, Wi-Fi, Bluetooth, torch,
  battery, charger, NFC and backlight packages are local builds, not
  packages from the project's published repository. The device package
  was built in an offline ARM64 package container. The smoke report
  establishes the installed versions; source receipts alone do not.
- **Shell** [V, recovered package]: local `omamobile-shell 0.2.12-1` was
  rebuilt from the installed local 0.2.10 source on
  `omamobile-oriole-canvas-budget`, preserving its phone-specific files and
  backporting the Canvas budget fix from
  [`be98c2c22e2b`](https://github.com/derekross/omarchy-mobile/commit/be98c2c22e2b260300c59787861fee200ef2ad58).
  The recovered package's `runtime.js` matches that commit byte for byte.
  This is not a claim that upstream shell 0.2.12 was installed unchanged.
- **Keyring correction** [V, recovered root image]: the local rebuild
  `omamobile-keyring 20261013.oriole1-1` retained the upstream public key
  and added a local public signing key to `omamobile.gpg` and its trusted
  list. The keyring payload matches the installed package's mtree in the
  recovered image; the additional key verifies the archived local device
  package. The local key therefore entered the packaged trust list, not
  only the build environment. The rebuilt source recipe has not yet been
  recovered; this establishes the package contents, not its exact build
  command. This does not follow the contributor-build
  policy, and that rebuilt keyring must not be distributed as the project's
  keyring. Neither the modified keyring nor signing material is included
  in this PR. The correction for the next image is the existing
  [contributor build workflow](../../docs/RELEASES.md#10-signing):
  unchanged upstream keyring, preserved published-package signatures,
  separate `keys/oriole/` and `out/oriole/`, and local-key trust confined to
  the build image. Any later on-device local trust is a separate owner
  action. No installed trust database was changed by this audit.
- **Slot B installation** [I, historical run records]: the initial root
  installation used the local `tools/install-phone` after its dry run,
  writing `boot_b`, `vendor_boot_b`, `dtbo_b` and `userdata`. Later kernel
  deployments used the local `bin/omamobile push` with B-image readback.
  The flags-3 stock-copy write to `vbmeta_b` and the B-success flag
  update at offset `0x35` in `devinfo` were done without project approval.
  The run records report the other `devinfo` bytes preserved and read-only
  protection restored. Early verification tests restored original
  `vbmeta_b`; the later native-root run records retain the B verification
  change. No further `devinfo` or `vbmeta` writes will be made without
  the maintainer's explicit approval. These are historical facts, not
  instructions or an extension of the public installer's permissions.
- **Local deployment tools** [V, recovered October 28 source snapshot]:
  `tools/install-phone` is byte-identical to the version at the mobile
  base `1155a7c352ed` above. `bin/omamobile` has local changes; its `push`
  path adds a `PUSH_MODULES_POLICY=require-match` option, follows image
  symlinks for readback length, and uses systemd's reboot on a native root.
  The complete two-file comparison against that base is in
  [PR #200](https://github.com/derekross/omarchy-mobile/pull/200).
  This is a dated source-snapshot comparison, not proof of the exact
  script bytes used for every earlier deployment, and not a comparison
  against today's main. The recovered Oriole descriptor still allows
  only `boot_b vendor_boot_b dtbo_b userdata`; the metadata writes were
  separate bring-up operations.
- **Slot A and bootloaders** [I, historical run records]: before Linux
  installation, the matching factory build and full OTA booted stock on
  B and then A, both reporting `slider-17.0-15681480`. The separate
  per-slot bootloader-version variables were unsupported, so independent
  partition-version equality was not established. The subsequent Linux
  records preserve A's images and slot record. A fresh partition audit
  has not been performed for this PR. `userdata` now contains Linux;
  returning to Android requires deliberate data restoration, not simply
  switching to slot A.

### Historical B-success write: mechanism and evidence limits

[V, recovered October 18 source and execution transcript] The separate
`mark-b-success-37.py` wrapper made `devinfo` writable with `blockdev --setrw`
on its containing boot LUN and then the partition. It restored the partition
and then the disk to read-only with an EXIT trap, backed by a 20-second
systemd undo timer and a 120-second watchdog. The wrapper did not stop the
storage watcher or change its allowlist. The recovered init's watcher skipped
nodes already marked in `/run/ufs-ro/`; it did not continuously reassert their
read-only flags. This is how the explicit opening remained possible.

The helper wrote a **128-byte header**, with only bit 1 at offset `0x35`
changed, followed by comparison of the full 8192-byte record. The gate's
2026-10-18 09:04:51 UTC execution result reports the one-bit change, unchanged
slot-A/opaque bytes, both nodes restored read-only and refreshed backups
verified. This was not a one-byte write.

The next Fastboot controller completed at 09:06:19 UTC after requiring
`current-slot=b`, `slot-successful:b=yes` and `slot-unbootable:b=no`, without
issuing set-active. The native audit at 09:11:29 UTC reported B success
persisting and A's record unchanged. These are recovered executed assertions
and results; the original Fastboot output and contemporaneous retry count
have not been recovered.

The original gate's before/after images are listed in the private backup
inventory but are not available locally, and the backup host was unreachable
during this audit. The exact bytes at `0x30`–`0x37` therefore remain unverified
for the original pair. This gap is not filled by deriving bytes from the
helper's logic or from a later backup. No new metadata write was attempted.

## 2026-10-30: native boot and RTC alarm status

- [V] A Pixel 6 booted native Linux from slot B with its root on `userdata`, using the unpublished `7.3.0-rc6-oriole-hsi1-late-dev0-g4bd35d34abb9-dirty` kernel and a separate integration package set. A read-only `phone-smoke --here --json` run passed all required checks: storage guard, root, USB and desktop session. It reported no failed units, oops, BUG or WARN, and 48 existing dmesg error lines; sound, sensors and cameras were skipped. This is a test-phone result, not an installation claim for the public images.
- [V, source] On the published kernel's `omamobile` branch, the S2MPG10 alarm callback reads `S2MPS14_REG_ST2` at offset `0x08`. On S2MPG10 that is `INT3M`; [Google's register definitions](https://android.googlesource.com/kernel/google-modules/soc/gs/+/204078758737776959ecc85c527a00fd2cabb62f/include/linux/mfd/samsung/s2mpg10-register.h) place `STATUS2` at `0x0d`, and its [RTC driver](https://android.googlesource.com/kernel/google-modules/soc/gs/+/204078758737776959ecc85c527a00fd2cabb62f/drivers/rtc/rtc-s2mpg10.c) reads the alarm-0 status bit there. The proposed kernel fix, [linux-omarchy-mobile#149](https://github.com/derekross/linux-omarchy-mobile/pull/149), is S2MPG10-specific; the S2MPG12 path remains unchanged. The test phone's unpublished kernel already contains the correction.
- [I] The smoke test does not exercise a live pending alarm, alarm delivery, suspend or wake. Those paths remain to be independently verified.

## 2026-10-16: stage 0, part 2: a retail Pixel 6 (stock, bootloader locked)

Read-only queries (`adb shell getprop`, `/dev/block/by-name`, `fastboot getvar all`) of a stock phone; facts only, no serial numbers.

- **Same build as the factory image** [V]: `ro.build.id` CP3A.261005.002.A1, security patch 2026-10-05, bootloader `slider-17.0-15681480` (F/'s `android-info.txt` requires exactly that one). So the F/ facts below apply to this phone, and `OS_PATCH_LEVEL=2026-10` matches its stock build.
- **The board dtbo entry is `0x20304`/`0x10000`** [V]: `ro.boot.dtbo_idx` is 0, and entry 0 of F/'s `dtbo.img` is board_id 0x20304, board_rev 0x10000, as `device.conf` assumed. `ro.boot.hardware.revision` and `getvar hw-revision` both say MP1.0.
- **No `init_boot` or `vendor_kernel_boot` partition** [V] (`getvar all`, `/dev/block/by-name`). The A/B partitions are `boot`, `vendor_boot`, `dtbo`, `pvmfw`, `vbmeta`, `vbmeta_system`, `vbmeta_vendor`, `modem`, the bootloader images (`abl`, `bl1`, `bl2`, `bl31`, `pbl`, `tzsw`, `gsa`, `ldfw`, `dram_train`) and `dpm`, plus the dynamic partitions in `super`. `BOOT_IMAGES="boot vendor_boot dtbo"` is right.
- **Not slotted** [V]: `persist`, `efs` and `efs_backup` (ext4), `modem_userdata`, `metadata`, `misc`, `frp`, `klog`, `fips`, `devinfo`, `mfg_data`, `super` and `userdata`. `/dev/block/by-name` also lists four whole LUNs, `sda` to `sdd`.
- **Bootloader state** [V]: `product:oriole`, `secure-boot:PRODUCTION`, current slot a, `unlocked:no`. Unlocking (`fastboot flashing unlock`) wipes the phone, and comes before any slot B write.
- **`soc_id` is still unknown** [I]: `/sys/firmware/devicetree/base/soc_id` is not readable without root on stock Android.

## 2026-10-14: stage 0, part 1: facts from the factory image (no phone)

### Boot images and partitions

- **No `init_boot` and no `vendor_kernel_boot`** [V] (F/ image zip): the image ships `boot`, `vendor_boot`, `dtbo`, `pvmfw`, the vbmetas and the dynamic partitions, and `fastboot-info.txt` flashes `boot`, `dtbo`, `pvmfw` and `vendor_boot`. So `BOOT_IMAGES="boot vendor_boot dtbo"`: the DTB and our initramfs go in `vendor_boot`, and the generic (empty) ramdisk and the OS version in `boot`. `bin/omamobile images`, `flash` and `push` handle that layout, and `tests/images-layout` checks it. A retail phone confirms the partition table (2026-10-16).
- **Header v4 for `boot` and `vendor_boot`** [V] (F/): `vendor_boot` page size 0x800, two vendor ramdisks and a bootconfig section. Stock's `boot` header leaves `os_version` at 0 (GKI; the AVB footer carries it), so `OS_VERSION`/`OS_PATCH_LEVEL` in `device.conf` are the factory build's (17.0.0, 2026-10), which is also the stock build on a retail phone (2026-10-16).
- **One image for four boards** [V] (F/ `android-info.txt`): `board=slider|whitefin|oriole|raven`.

### Base DTB

- **Two base DTBs, by `soc_id`** [V] (F/ `vendor_boot`): `<0x9845000 0x00>` and `<0x9845001 0x00>`, both with root compatible `google,gs101`. They are gs101's counterparts of cheetah's `0x9855000`/`0x9855001`. Which is A0 and which is B0, and which one a retail phone has, are [I].
- **Upstream's gs101 DTs carry no `/soc_id`** [V] (U/), while `bin/omamobile images` requires the DTB to be built with the first `SOC_IDS` value. Our oriole board DTS (or a kernel commit) has to add it. postmarketOS boots the upstream DTB without one (P/), so ABL may fall back to the first DTB when nothing matches [I].
- **`/ect` and `/chosen/bootargs`** [V]: stock has both, and upstream `gs101.dtsi`/`gs101-pixel-common.dtsi` keep both ("required to be present by bootloader", "otherwise it crashes").
- **Root `#address-cells = <2>`, `#size-cells = <1>`** [V] (F/), as on gs201: each `reg` entry is 12 bytes (AGENTS.md, reserved memory).
- **Console UART at `0x10A00000`** [V] (F/ stock bootargs: `earlycon=exynos4210,mmio32,0x10A00000`), the same address as gs201's.

### Board dtbo

[V] (F/ `dtbo.img`: 18 entries, page size 4096), read from each entry's strings:

| board_id | board_rev | Panel | Touch | Board |
| --- | --- | --- | --- | --- |
| 0x20302, 0x20303, 0x20304 | 0x10000, 0x10100 (0x20303 also 0x10002) | S6E3FC3 | ST FTS | oriole |
| 0x20202, 0x20203, 0x20204 | 0x10000, 0x10100 (0x20203 also 0x10001) | S6E3HC3 | Samsung `sec_ts` | raven |
| 0x201, 0x20001 | several | S6E3HC2 | both | development boards [I] |

- **Oriole's panel is the Samsung S6E3FC3; its touch controller is an ST FTS on SPI** [V] (F/, and P/'s oriole DT: `"st,spi"` on SPI11, an out-of-tree driver). "goodix" in every entry is the under-display fingerprint sensor [I].
- **Retail board_id/board_rev**: the newest, `0x20304`/`0x10000` (cheetah's is `0x30306`/`0x10000`), as `device.conf` and `oriole-m1-overlay.dts` use. Confirmed with `ro.boot.dtbo_idx` on a retail phone (2026-10-16) [V].

### Display trigger

- **postmarketOS writes `0x3061` to DECON at `0x1c300030` at boot** "to enable framebuffer refreshing" (P/ `soc-google-gs101/00-enable-fb.sh`). That is gs101's counterpart of cheetah's `TRIG_CON` write at `0x1c240030` [I]. Read it first, on a booted phone, before any write, and never use cheetah's address on oriole. It belongs in `device.conf` as `SIMPLEFB_TRIG_CON="0xADDR 0xVALUE"` (the key from #3 and #4), not in code; oriole gets no `SIMPLEFB_TRIG_CON` until that read has been done.

### GPU

- **The Mali-G78 is a Valhall Job Manager GPU, so the kernel driver is Panfrost, not Panthor** [V] (P/: `arm,mali-valhall-jm`, `DRM_PANFROST=y`). Our Mesa package already builds the `panfrost` Gallium driver, which serves both. The GPU node and its supplies are in the gs101-mainline fork, not upstream v7.0 [V] (U/, P/).

### Still needs a phone

- the `soc_id` the phone reports (needs root, or the bootloader's own report);
- with root: the live DT (`/sys/firmware/fdt`) and `/proc/iomem`, to check every stock `reserved-memory` region.

`bin/omamobile capture` collects these into a private folder (`~/android/reference`); post facts, never the files.

And before oriole's first boot, in the tree:

- **An initramfs.** Done in the tree: oriole uses the shared `initramfs/init`, and `bin/omamobile initramfs` packs its `UFS_RW` (`boot_b vendor_boot_b dtbo_b userdata`) into `/etc/omamobile-ufs-rw`, which the init's storage guard reads. A missing or malformed list leaves every partition read-only (`tests/storage-guard`). No phone has booted it yet.
- **`/soc_id` in the board DTS** (see Base DTB).
