<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 (oriole): suspend

Status (2026-11-08): **S1-S4 pass** on kernel `g1b68efca2dd9`: pm_test
dry runs, a 30 s s2idle with an RTC wake, and ten 20 s cycles in a row with
Wi-Fi kept connected through brcmfmac's D3 and Bluetooth up. device-oriole
0.2.12 carries raven's sleep policy and hook; `mem_sleep_default=s2idle`
is in the vendor cmdline for the next push. Next: `systemctl suspend` woken
by the power key (S5, with the owner), then the shell's auto sleep.

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

## 4. Next

| Step | What | Pass |
| --- | --- | --- |
| S5 | with the owner: `systemctl suspend` (through logind, so the hook runs), an RTC alarm 120 s out as a backstop, the owner presses Power | woken by `gpio-keys` (KEY_POWER), the panel lights, the lock screen answers touch, Wi-Fi and Bluetooth back |
| S6 | the next push: `mem_sleep` reads `[s2idle] deep` at boot | - |
| S7 | battery: awake vs asleep from the gauge's charge counter, cable out | a number to compare with raven's 293 mA |
| S8 | the shell's auto sleep (`omamobile-sleepd`, toggle `suspend-off` off) | sleeps when locked and dark, wakes on Power |
