<!-- SPDX-License-Identifier: MIT -->
# Pixel 6 first-light policy

Initial standalone package, version 0.2.0. No kernel, rootfs integration or
hardware qualification is supplied. Installation stages files only and enables
no services. The optional system setup command configures locale/time and
enables timesyncd.

Portable helpers, build script, USB network policy and storage udev policy
come from the cheetah package. The storage rules continue an existing
initramfs guard: they do not set partitions read-only themselves. A guarded
initramfs and matching kernel remain prerequisites. USB gadget creation and
networkd activation are also external prerequisites. USB has fallback metric
2000, below ordinary Wi-Fi preference.

All four sleep modes are disabled. UPower selects PowerOff on critical
battery. Coldplug blocks alias and explicit modprobe for
max77759_charger, gpio_max77759, arm_dsu_pmu, at24 and max17042_battery.
The [coldplug audit](../../device/oriole/NOTES.md) found matching unbound devices and confirmed that
modprobe dry runs refused these modules; it did not load them or establish
their behavior. `clk_acpm` left the list on 2026-11-08: it is the GPU's clock
provider over ACPM, and panfrost deferred forever without it; `max17042_battery` left it the same day,
the gauge reporting the battery as on raven; `max77759_charger` followed with
raven's `omamobile-charger.conf` options and `omamobile-charge-guard`, once the
battery node was in the device tree. 0.2.6 loads the display stack (gs-decon,
the S6E3FC3 panel driver) at boot, capped at 60 Hz. This package does not qualify the underlying hardware.
Built-in drivers are outside modprobe policy.

No battery startup, staged boot checker, automatic boot-success, radio,
charging or GS201 hardware services are included. Boot-control tooling is
omitted while slot marking remains on hold. Native ARM64 build and
installation passed with a private companion preserving
existing integration safeguards; see the 2026-11-07 entries in
[NOTES](../../device/oriole/NOTES.md). Standalone installation and a boot of
this package combination remain unqualified.

The PowerOff action has not been qualified for oriole; Linux power-off on other
Pixels can restart instead.

Run test/all for a native helper build and
staged installation checks. Version 0.2.0 sorts above the local integration
package 0.1.9-2, satisfying existing hardware-package version dependencies.
That integration package carries
boot and battery safeguards absent here. Replacing it requires a separately
qualified migration and rollback; do not remove it as a routine upgrade.
