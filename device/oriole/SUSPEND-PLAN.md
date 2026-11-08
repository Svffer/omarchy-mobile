<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): suspend

Status (2026-11-08): **S1-S6 and S8 pass; suspend is on for the owner**
(the shell's auto sleep and Suspend in the menu). Earlier: **S1-S5 pass** on kernel `g1b68efca2dd9`: pm_test
dry runs, RTC-woken s2idle (one 30 s, then ten 20 s cycles) and
`systemctl suspend` woken by the power key; Wi-Fi kept its association
through brcmfmac's D3 in the rtcwake runs, while through logind
NetworkManager drops and rejoins it, and Bluetooth links drop in every
sleep (S5's correction). device-oriole 0.2.12 carries
raven's sleep policy and hook; `mem_sleep_default=s2idle` is in the vendor
cmdline for the next push (S6). Next: battery numbers (S7) and the shell's
auto sleep (S8).

## 1. Approach

Raven's ([device/raven/SUSPEND-PLAN.md](../raven/SUSPEND-PLAN.md)) and
cheetah's: suspend-to-idle, since the firmware's PSCI SYSTEM_SUSPEND
("deep") needs a gs101 SoC sleep sequence Linux doesn't have. Everything
raven needed is in the tree already: the exynos-pmu CPU idle hint kept in
s2idle, gs101's UFS `spm_lvl` 5 default (fb5d682cf and linux#188), the
brcmfmac D3 handshake by control message, gs-decon without a VT switch
around sleep (44c22bf81).

## 2. Facts (S0, read-only, 2026-11-08)

| Item | Value |
| --- | --- |
| `/sys/power/state` | `freeze mem disk` |
| `mem_sleep` | `s2idle [deep]` at boot: deep would not wake; `echo s2idle` at runtime for the tests, `mem_sleep_default=s2idle` in `VENDOR_CMDLINE` from the next push |
| `pm_test` | available (`none core processors platform devices freezer`) |
| UFS | `spm_lvl` 5, `rpm_lvl` 3 |
| RTC | `rtc0` = `s2mpg10-rtc`; its alarm reaches the CPU as IRQ 105 (`gpa0-6`, S2MPG10) |
| Wakeup sources | `gpio-keys`, pmic-1, pmic-2, `s2mpg10-rtc`, the Wi-Fi PCIe function `0000:01:00.0`, `max170xx_battery`, `max77759-charger` |
| Policy before 0.2.12 | `AllowSuspend=no` (`20-omamobile-unqualified-sleep.conf`); shell toggle `suspend-off` |

## 3. Results

**S1, pm_test dry runs (14:45, `g1b68efca2dd9`, runtime s2idle):**
`freezer` and `devices`, 5 s each, both `rc=0`; brcmfmac "hot resume", no
UFS error, no failed device; afterwards Wi-Fi connected, Bluetooth powered.
gs-decon's resume is a no-op, as on raven: the panel's state is the
session's.

**S3, one real s2idle (14:46):** `rtcwake -m freeze -s 30`, `rc=0`, 31 s
by the wall clock (the kernel's monotonic clock barely moves in s2idle),
woken by IRQ 105 (the PMIC carrying the RTC alarm). brcmfmac: "D3 ACK by
control message after 13454 us", then "hot resume"; Wi-Fi still connected
and pinging (8.7 ms), Bluetooth powered. `suspend_stats` 3 ok / 0 fail.

**S4, ten cycles (14:48-14:53):** `rtcwake -m freeze -s 20`, 8 s awake
between:

| Cycles | rc | Slept | Wake IRQ | Wi-Fi connected | Ping | Bluetooth |
| --- | --- | --- | --- | --- | --- | --- |
| 1-10 | 0 | 21 s each | 105 | yes | yes | powered |

`suspend_stats` +10 ok, +0 fail; no UFS, D3, commonring or HCI error.

**A bug found on the way:** the Bluetooth address rule (raven's) matched
connection nodes too (`hci0:11`, `DEVTYPE=link`, created when the
headphones connect), and their units failed. device-oriole 0.2.12 limits it
to `DEVTYPE=host`; raven's package has the same rule.

**S5, `systemctl suspend` and the power key (14:57, device-oriole 0.2.12):**
with the owner holding the phone, `systemctl suspend -i` (logind, so the
hook runs) with an RTC alarm 120 s out as a backstop. systemd-sleep froze
`user.slice` and slept at 14:57:31; the owner pressed Power and it woke at
14:57:57, **woken by IRQ 109, `gpa10-1 KEY_POWER`** (logged by the
`omamobile-sleep` hook), before the alarm (cleared afterwards). brcmfmac's
D3 ACK in 13.6 ms; afterwards Wi-Fi connected and pinging (8.9 ms),
Bluetooth powered, no UFS error. The owner confirmed the button wakes it.

Correction, from the journal read afterwards: through logind, Wi-Fi did
**not** stay connected. NetworkManager takes the device down on
PrepareForSleep ("state change: deactivating -> disconnected (reason
'sleeping')" at 14:57:30) and rejoins after the wake; the ping above came
after that rejoin. The rtcwake runs of S3/S4 bypass logind, which is why
they kept the association. And the Bluetooth headphones dropped too: the
kernel's `hci_suspend_sync()` disconnects every link before sleep
("soft disconnect everything", `HCI_ERROR_REMOTE_POWER_OFF`); the
WH-1000XM4 reconnected about 3 s after the wake (bluetoothd reloading the
A2DP endpoint at 14:58:00, also at 14:52:49 during S4). Separately, every
reconnect logs "Unable to connect Hands-Free Voice gateway: socket(STREAM,
RFCOMM): Protocol not supported": the kernel has no `CONFIG_BT_RFCOMM`, so
headset calls (HFP/HSP) cannot work, asleep or awake.

**Bluetooth audio and auto sleep (omamobile-shell 0.2.102):** the owner
chose to keep headphones connected rather than save battery while they
are: `omamobile-sleepd` refuses sleep while a `bluez_output` sink exists,
paused or not ("Bluetooth audio is connected (...)"); playback (a RUNNING
sink) and calls (a logind inhibitor) were refused already. On the phone,
with the WH-1000XM4 paused, the check sees `bluez_output....1 SUSPENDED`.
Headset calls still need `CONFIG_BT_RFCOMM` in the kernel.

**S6, the push (nineteenth boot, `g1b68efca2dd9` with `CONFIG_BT_RFCOMM=m`,
`mem_sleep_default=s2idle`):** `mem_sleep` reads `[s2idle] deep` at boot;
`rfcomm.ko` loads on demand (`bt-proto-3`) and an RFCOMM socket opens, so
bluez can offer the headset profiles; Wi-Fi joined, Bluetooth powered,
90 Hz. Then, for the owner, the shell's auto sleep turned on (`omarchy-toggle
omamobile-sleep on`) and Suspend unhidden in the menu (`suspend-off` off);
`omamobile-sleepd check` answers "the lock isn't secure" with the phone
unlocked, as it should.

**S8, the shell's auto sleep (nineteenth boot):** the owner locked the
phone and left it: `omamobile-sleepd` "sleeping (screen off and locked for
30 s)" at 15:21:19, asleep until the owner's press of Power at 15:23:12
("woken by IRQ 109 ... KEY_POWER", "a power-key wake: lighting the lock
screen"). **Passes.**

**Headset profile (same boot):** with RFCOMM, the WH-1000XM4's card offers
`headset-head-unit` and `headset-head-unit-cvsd` (HFP) next to the A2DP
profiles, and no "Protocol not supported" is logged. A call through it
waits for the modem.

**Reconnect at boot:** the headphones didn't connect by themselves; the
owner connected them by hand. bluetoothd tried once when it started
(15:20:31, "avdtp_connect_cb() connect to ...: Host is down (112)": the
headphones weren't on yet) and bluez doesn't retry a device that was never
connected in this boot; the headphones, switched on later, reach for the
host they last used. Not a port fault; a small "reconnect trusted audio
devices for a while after boot" helper would cover it.

**Sleep power, first look (owner, 15:25-15:55, cable out):** one
unbroken 29.4 min sleep with no dark wake cost about 6 % of a 3828 mAh
battery (85 % of design): about 430 mA, against about 23 mA for AOSP's
seven days. s2idle only idles the CPUs; MIF stays at 2028 MHz as the
bootloader left it, and only SYS_SLEEP (not in Linux for gs101; not done on
cheetah either) gets near AOSP. Two avoidable loads were on:

- **The display:** with `screen_off` 0, DPMS off left DECON scanning black
  frames at 60 Hz into an awake panel. `screen_off=2` at runtime: DPMS off
  masks the trigger (0 frames in 2 s), DPMS on runs the oriole variant's
  cold init and scans at 90 Hz again (no flip timeout). The first DPMS off
  skipped the panel's sleep ("Skipping disable of already disabled
  panel"): a panel taken over from the bootloader wasn't marked prepared;
  fixed in the panel driver (`50a7df7b9`, the next push). device-oriole
  0.2.13 sets `options gs_decon screen_off=2`.
- **Wi-Fi:** the BCM4389 stayed powered with its link up. device-oriole
  0.2.13 ships `/etc/omamobile/wifi-unload-for-sleep`, the hook's unload
  mode. A logind sleep with the screen off: the panel asleep, brcmfmac and
  `pcie_gs201` unloaded, `wlan-reg-on` disabled; on the lit screen Wi-Fi
  came back but NetworkManager joined only after 44 s (it waits for its own
  periodic scan on a device it takes as new). 0.2.14's hook asks for a
  scan as soon as the device is available: **joined 13 s after the screen
  lit.**

## 4. Next

| Step | What | Pass |
| --- | --- | --- |
| S7 | battery: awake vs asleep from the gauge's charge counter, cable out | a number to compare with raven's 293 mA |
