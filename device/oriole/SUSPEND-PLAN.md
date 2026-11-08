<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): suspend

Status (2026-11-08): **S1-S5 pass** on kernel `g1b68efca2dd9`: pm_test
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

## 4. Next

| Step | What | Pass |
| --- | --- | --- |
| S6 | the next push: `mem_sleep` reads `[s2idle] deep` at boot | - |
| S7 | battery: awake vs asleep from the gauge's charge counter, cable out | a number to compare with raven's 293 mA |
| S8 | the shell's auto sleep (`omamobile-sleepd`, toggle `suspend-off` off) | sleeps when locked and dark, wakes on Power |
