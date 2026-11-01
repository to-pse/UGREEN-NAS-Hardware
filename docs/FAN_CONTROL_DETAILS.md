# DXP4800 Pro: Fan control technical details

This document consolidates existing Debian fan-control investigations. It is not an instruction to access hardware. See the [recovery guide](DEBIAN13_DXP4800PRO.md) and [files and permissions](RECOVERY_FILES_AND_PERMISSIONS.md).

Additional raw observations and the unresolved sensor-documentation discrepancy are in [technical evidence](TECHNICAL_EVIDENCE.md).

**Evidence labels:** *Observed* = recorded on the NAS on 1 November 2026, not necessarily true today; *source-derived* = behavior of the inspected implementation, not proof of a particular hardware transaction; *historically reported* = earlier tests not repeated; *operator recollection* = unverified memory; *unknown* = no adequate proof. The prior inventories were read-only except for a separately documented userspace comparison build. No hardware operations were performed for this consolidation.

## Model, controller and backend

**Observed on 1 November 2026, 17:18–17:22 CET:** exact DMI `DXP4800 Pro`, UGREEN, Debian GNU/Linux 13.7 amd64, kernel `6.12.113+deb13-amd64` (Debian `6.12.113-1`), BIOS `DXP4800Pro V1.5 251030` (2025-10-30). The root-run `ugreenctl-fand.service` was enabled and active, started at 15:57:24 following boot at 15:57:17, `NRestarts=0`; the daemon state was being refreshed. This demonstrates a service start after that boot, not a complete reboot or cooling validation.

**Source-derived:** exact DMI selects plugin `dxp4800plus` for DXP4800 Plus/Pro. This is IT8613 Super I/O, not ACPI EC. Configuration ports are `0x2e/0x2f`, hardware-monitor ports `0xa35/0xa36`, expected chip ID `0x8613`. Values are evidence, **not manual port-access instructions**. Other similarly named devices must not inherit this map.

| Channel | Preferred dynamic `it8613` hwmon files | IT8613 control / PWM | Tachometer low / high |
| --- | --- | --- | --- |
| CPU `cpu` | `pwm2`, `pwm2_enable`, `fan2_input` | `0x16 / 0x6b` | `0x0e / 0x19` |
| System `sys` | `pwm3`, `pwm3_enable`, `fan3_input` | `0x17 / 0x73` | `0x0f / 0x1a` |

The hwmon path requires a **unique** device named `it8613`, not a fixed `hwmonN`. Driver identity and physical wiring require separate proof. Register mapping is source-/firmware-derived, not an independent physical channel test.

**Observed:** Debian package `linux-image-6.12.113+deb13-amd64` supplied `/lib/modules/6.12.113+deb13-amd64/kernel/drivers/hwmon/it87.ko.xz` (resolved under `/usr/lib/modules`), 23116 bytes, SHA-256 `901f0412c16009ba6b5b7ec6d876ebccdeaad438414a0552987437d6e8cb512c`; `CONFIG_SENSORS_IT87=m`. `it87` was not loaded, no `it8613` hwmon node or `/proc/it86` was visible. `modinfo` reported GPL, authors Chris Gauthron and Jean Delvare, `intree=Y`, dependency `hwmon-vid`, no independent module version, and build-time kernel signing metadata. Parameters include `force_id`, `ignore_resource_conflict`, `update_vbat`, `fix_pwm_polarity`; actual runtime values were unavailable because the module was not loaded. No local `it87`/`force_id` setting was found in the examined `/etc` module configuration. Compatibility, binding, parameter needs and Secure Boot acceptance are **unknown**. DKMS listed `led-ugreen/0.3`, not `it87`; the loaded `led_ugreen` is an LED driver, not a proven fan backend.

**Source-derived and inferred:** hwmon is preferred; only an absent node (`-ENODEV`) permits the experimental direct fallback. The installed plugin contains hwmon/direct and `ioperm` references. With no `it87` or hwmon node, direct Super I/O is strongly suspected, **not directly observed**. A lockfile or reported PWM does not prove a transaction. Direct fallback requires exact DMI, no vendor/`it87` owner, process lock `/run/ugreenctl-it8613.lock`, chip and HWM logical-device identification, safe PWM 40–255, manual-bit and PWM readback, and `--force --apply`. Never unload or bypass `it87` to enter it. If `it87` is loaded and the hwmon node is absent, stop. `--force` does not bypass identity, ownership or locking. Separate daemon lock `/run/ugreenctl-fand.lock` cannot prevent arbitrary third-party writes. `fan status` without hwmon can itself attempt direct access; thermal status and the daemon can issue guarded ATA SMART ioctls.

UGOS firmware evidence associates this family with `ug_it86x_cpufan`, `hwmonitor-480t` and `/proc/it86/fan`; these are not stock Debian dependencies or a new on-device UGOS validation. Firmware image analysis SHA-256 `7bb2746324ac852475727cf23f7016517996b91dfb293d2292d531c1e71581b0`.

## Software curve and sensor selection

`ugreenctl-fand` is a userspace curve, not IT8613 hardware automatic mode. **Source-derived** DXP4800 Plus/Pro profile `stock-4800plus` uses thresholds `stop/start/mid/full/max` in °C and PWM points `idle/mid/full/max`:

| Source | Thresholds | System PWM | CPU PWM |
| --- | --- | --- | --- |
| CPU | 42 / 50 / 70 / 78 / 90 | 65 / 125 / 200 / 235 | 60 / 125 / 205 / 230 |
| HDD | 30 / 40 / 46 / 52 / 55 | 65 / 125 / 200 / 235 | CPU channel uses CPU only |
| NVMe | 50 / 55 / 60 / 65 / 70 | 65 / 125 / 200 / 235 | CPU channel uses CPU only |

System target is the highest applicable source and CPU floor, with 65 °C → PWM 100 and 90 °C → PWM 205, interpolated between. PWM has a safety floor 40, not a physical RPM guarantee; software does not reproduce a zero-PWM stop. A reduction may be delayed 60 seconds. These are documented source/firmware points, not new measurement of the installed binary or evidence for another model.

The inspected source recognizes CPU names `coretemp`, `k8temp`, `k10temp`, `zenpower`. `cpu_celsius` is the first valid glob-sorted `temp*_input` in the first usable CPU device; `cpu_peak_celsius` is the maximum across recognized devices and is preferred for the curve. HDD `drivetemp` and NVMe `nvme`/`nvme-pci` use maxima across valid attributes/devices regardless of labels. Invalid/unreadable/negative or post-conversion >125 °C readings are ignored; normal hwmon values are millidegrees, converted to integer °C.

If no valid HDD hwmon reading exists globally, source describes a guarded ATA power-mode check and SMART read for eligible running rotational ATA `sd*` devices, with attribute 194 preferred to 190. One valid `drivetemp` value suppresses this global fallback even if other disks are uncovered. Do not equate this with passive sysfs access or guarantee it cannot affect every bridge. `require_storage_sensor=true` requires **one HDD or NVMe reading**, not every installed drive. Missing CPU or both storage categories requests failsafe PWM 255 for both channels. A blocked/failed write can prevent later channel attempts; neither requested nor reported failsafe proves RPM or airflow.

**Observed, 17:47 CET, non-atomic sysfs snapshot:** `coretemp` CPU peak 48 °C; `nvme0` Samsung SSD 990 PRO 1TB at PCI `0000:02:00.0`: Composite 44.85 °C, Sensor 1 44.85 °C, Sensor 2 53.85 °C; `nvme1` AirDisk 128GB SSD at PCI `0000:04:00.0`: 41.85 / 41.85 / 42.85 °C. `sda`, ATA WDC WD60EFRX-68M, was the only visible rotating HDD and had no visible `drivetemp`. Separate daemon state at 17:46:54 reported CPU 44, CPU peak 46, HDD 35, SSD 53 °C; HDD source was **not observed** and SMART fallback is only plausible. `acpitz` 27.8 °C, `enp3s0` PHY/MAC 51 °C and `spd5118` 40.25 °C were also visible but not selected by these name filters. No physical drive-slot mapping, complete drive coverage or standby behavior was established.

## Failure detection and historical validation

**Source-derived:** the example and observed on-disk configuration use `failure_threshold=3`, `beep_on_failure=true`, `beep_repeat_seconds=60`. Three consecutive temperature/control errors trigger an optional 100 ms, 1000 Hz PC-speaker tone; attempts are throttled with in-memory monotonic time and reset on restart. A successful cycle resets the failure count. An unavailable speaker does not stop fan control; the alarm only detects errors reported by the control loop, not every physical fan failure. State file values record software actions, not continuous hardware readback, independently measured RPM or airflow.

**Historically reported:** a prior Debian test built successfully, passed `thermal-curve` and `fan-failure`, produced a speaker tone, and during CPU load reported ~39→56 °C, CPU PWM 60→79, system PWM 65→83 and return after cooling. Raw measurements, exact test conditions and automatic alarm on a real failure remain unavailable/unvalidated. A later documentation build passed the same two fixture tests; neither test proves physical cooling.

**Operator recollection:** an earlier unspecified access reportedly failed without `--force` and apparently worked with it; command, error text and cause are unknown. It is not a safety justification.

## Binary provenance and CLI decision

**Observed, 1 November 2026:** ELF64 x86-64, dynamically linked, GCC comment `14.2.0`; runtime `libc6:amd64 2.41-12+deb13u4`, loader `/lib64/ld-linux-x86-64.so.2`, highest observed required symbol `GLIBC_2.38`, no RPATH/RUNPATH. `libc.so.6` was the only declared `DT_NEEDED` library; the CLI link command used `-ldl` without a separate `libdl.so` entry. The plugin is dynamically loaded and is not a CLI `DT_NEEDED` entry. Installed artifacts:

| Artifact | Bytes | SHA-256 | ELF build ID |
| --- | ---: | --- | --- |
| `ugreenctl` | 303624 | `74d2b90cae975a496c58579c2bbfbb70214dd307e89ed726e8e2de30155aeafb` | `165f43f373e379e4182e6a4c01a0d411812d6392` |
| `ugreenctl-fand` | 39432 | `464db8865a9bb3ee1c9dd86361a24440dddf5e63dbc9e11772aeca8a95728dca` | `5b02d72591807880cb589f4f47824fca620a4681` |
| `dxp4800plus.so` | 39744 | `9072081fded14e00eff147f8c324c52eeb2edbd28ddd65737be1fb4cd796832d` | `1294a9f734fa3291999360f419ce6cc93254febf` |
| `ugreenctl-fand.before-alarm` | 39264 | `afe7f4b0e854e44d7e7dd2b1779738112cc411ce32d767924d28da7bd17cec02` | `e75c5d25511e1a8aa946c5e80425da95138344d3` |

The separately built CLI was also 303624 bytes but SHA-256 `949e32d01329c69b25bdb43e370f4da391c2151045c6d2ea98d827fae1fa2bad`, build ID `556afdf244ed119330fff47e231d2f02f917fa54`; **not byte-identical**. The comparison daemon and plugin were byte-identical, not proof of original commit. Build used feature worktree commit `7dab8f0ee46fd4eecb3db73c380e40075f9d2bf5`, CMake 3.31.6, GCC 14.2.0, binutils 2.44, make 4.4.1, Release `-O3 -DNDEBUG`, libdir `lib`, C flags `-std=c11 -Wall -Wextra -Wpedantic`; plugin flags included `-fPIC`. The then Debian target commit was `adf00ec7113171cbc42b6b8088b33dfe1a3747b3`; tracked program/build sources matched between those commits. A branch tip can change.

**Historically reported manual comparison:** `exit_on_failure` and configurable `beep_device` were removed, speaker path made fixed, while `failure_threshold`, `beep_on_failure`, `beep_repeat_seconds` remained; `ugreenctl_fan_curve_evaluate_plan()` was reported unchanged between the compared source files. The stated intent was for the daemon to continue on failure and for the speaker path not to be configurable. This was not independently repeated and does not prove installed binary origin or equivalent behavior. Old source/build candidates and a shell-history install reference likewise did not establish original CLI provenance. Recovery policy is to build from `to-pse/UGREEN-NAS-Hardware` branch `debian13-dxp4800pro`; the old CLI is not a required input, but an existing verified copy may be kept for rollback.

## Open validation points

The actual direct port transaction, physical CPU/system mapping, RPM and airflow, all-drive sensor coverage and standby behavior, per-channel failsafe, alarm on real failure, complete boot/journal history, crash/hang behavior and supervised rollback remain **unknown**. The non-root inventory could not read all system journal entries or the daemon's `/proc/<pid>/exe`, `maps` and `fd`; absence of visible entries is not evidence of absence. Complete fresh-install source recovery and future branch compatibility have not been tested. All future active validation requires explicit authorization, exclusive control, independent cooling and a safe shutdown plan.
