# DXP4800 Pro – Debian 13 Fan Control Setup

This guide consolidates the previously reported DXP4800 Pro Debian 13 setup, the repository's firmware and implementation evidence, and the dated read-only production inventory below. Its installation instructions describe a future fresh installation, not authorization to change an existing production NAS. Earlier observations and their limits are listed under [Validation](#validation).

Additional branch-specific discrepancies and dated raw evidence are recorded in [technical evidence](TECHNICAL_EVIDENCE.md); the recovery instructions do not depend on that reference.

**Starting the service performs real PWM writes.** Never install, load or unload modules, change a service, run the daemon, or write PWM on a production NAS without explicit authorization, a maintenance window, an independent cooling fallback, and a recovery plan. First perform the read-only checks and dry run below. Do not run multiple fan controllers concurrently.

## Hardware and operating-system scope

The exact DMI product must be `DXP4800 Pro`. `models/dxp4800plus.c` selects the `dxp4800plus` plugin for this model and `DXP4800 Plus`; `fan/it8613_fan.c` defines their mapping. Do not reuse the values for other models.

| Target | it87 hwmon files | IT8613 control / PWM registers | Tachometer registers, low / high |
| --- | --- | --- | --- |
| CPU (`cpu`) | `pwm2`, `pwm2_enable`, `fan2_input` | `0x16 / 0x6b` | `0x0e / 0x19` |
| System (`sys`) | `pwm3`, `pwm3_enable`, `fan3_input` | `0x17 / 0x73` | `0x0f / 0x1a` |

The preferred Linux `it87` path discovers a **unique** dynamic hwmon device named `it8613`; it does not rely on a fixed `hwmonN`. RPM is read from the fan input, not inferred from PWM. PWM is on a 0–255 scale; a PWM floor does not guarantee a physical minimum RPM. The guarded direct fallback uses configuration ports `0x2e/0x2f`, HWM ports `0xa35/0xa36`, and checks chip ID `0x8613` and the HWM logical device. These addresses are evidence, not instructions for manual port access. See [HARDWARE_SUPPORT.md](HARDWARE_SUPPORT.md) and [FAN_CURVE.md](FAN_CURVE.md).

UGOS firmware evidence routes the Plus/Pro family to vendor `ug_it86x_cpufan` and model-specific `hwmonitor-480t`, which uses `/proc/it86/fan`; `hwmonitor.service` and startup-script packaging can vary by UGOS version. Repository evidence associates `ug_201x_sio` with DXP2800 GT, not the DXP4800 Pro fan path. **None of these vendor components is a stock Debian dependency.** The project prefers a compatible `it87` hwmon path when available and provides a guarded direct fallback; the inventoried production system had no active `it87` path, as detailed below. Its `ugreenctl-fand` curve is userspace control, not IT8613 hardware automatic mode. The UGOS route and curve are firmware evidence, not a separate UGOS on-device test established by this guide.

The hwmon backend checks vendor ownership and uses `/run/ugreenctl-it8613.lock`, sets manual mode, and reads back writes. `/run/ugreenctl-fand.lock` prevents two copies of the daemon, but cannot prevent arbitrary third-party sysfs writes. Direct fallback is experimental: it requires the hwmon node to be absent, no active `it87` or vendor owner, chip and logical-device checks, the lock, and `--force --apply`. Do not unload or bypass `it87` to make fallback work. If the hwmon node is missing while `it87` is loaded, stop and investigate; direct writes have not been independently physically validated for this model.

## Production inventory: 1 November 2026

These are time-stamped findings from read-only inventories, not a claim about the present state after that date or an authorization to modify the system. **Evidence labels:** *Directly observed* means observed during the inventory, including file contents or software-reported state; *source-derived* means inferred from the inspected repository, whose historical identity with installed binaries is not established; *historically reported* and *operator recollection* are not newly verified; *unresolved* identifies missing proof. Consolidated technical evidence is in [fan control details](FAN_CONTROL_DETAILS.md), and recovery-file metadata in [files and permissions](RECOVERY_FILES_AND_PERMISSIONS.md). The prior userspace inventory also recorded a separately authorized comparison build, not a hardware test or installation.

### System and installed kernel module

**Directly observed:** UGREEN `DXP4800 Pro`, Debian GNU/Linux `13.7` (trixie), architecture `x86_64` / `amd64`, kernel `6.12.113+deb13-amd64` (Debian package version `6.12.113-1`), BIOS `DXP4800Pro V1.5 251030`.

The installed `it87` module file belongs to Debian's `linux-image-6.12.113+deb13-amd64` package; `CONFIG_SENSORS_IT87=m`. Its path was `/lib/modules/6.12.113+deb13-amd64/kernel/drivers/hwmon/it87.ko.xz`, resolving to `/usr/lib/modules/6.12.113+deb13-amd64/kernel/drivers/hwmon/it87.ko.xz`; SHA-256 of the compressed file: `901f0412c16009ba6b5b7ec6d876ebccdeaad438414a0552987437d6e8cb512c`. Package ownership and this inventory hash do not establish an independent archive integrity check.

**Directly observed:** `it87` was **not loaded**, no `it8613` hwmon device was present, and no `it87` binding was established. An installed module is not necessarily the active driver. DKMS reported `led-ugreen/0.3` for this kernel, but no `it87` entry; the loaded `led_ugreen` identifies itself as an LED driver, not a proven fan controller. The Debian module's compatibility and ability to bind this chip remain **unresolved**.

### Service and productive configuration

**Directly observed:** `ugreenctl-fand.service` was `enabled`, `active (running)`, running as root. systemd reported start at `2026-11-01 15:57:24 CET` after the then-current boot at `15:57:17`, `NRestarts=0`, and no unit drop-ins. Its unit file was `/etc/systemd/system/ugreenctl-fand.service`. The complete observed process command line was:

```text
/usr/local/bin/ugreenctl-fand --config /etc/ugreenctl/fan-curve.conf --state /run/ugreenctl/fan-curve.state --ugreenctl /usr/local/bin/ugreenctl --plugin-dir /usr/local/lib/ugreenctl/models
```

**Directly observed:** the productive configuration file referenced by that command was `/etc/ugreenctl/fan-curve.conf`, owned by `root:root` with mode `0644`; its complete on-disk contents were:

```ini
profile=stock-4800plus
interval_seconds=10
downshift_delay_seconds=60
failsafe_pwm=255
require_storage_sensor=true
allow_unvalidated_writes=true
failure_threshold=3
beep_on_failure=true
beep_repeat_seconds=60
```

This is the observed production file, distinct from the repository example used in the [recovery procedure](#source-based-recovery-chronological-procedure). The state file confirmed the profile, but every individual parameter actually held in process memory was not independently checked; the file could have changed after process start.

**Source-derived:** in the inspected `core/fan_daemon.c`, `allow_unvalidated_writes=true` causes `--force` to be passed on corresponding PWM invocations, alongside `--apply`. This is a security-sensitive permission for potentially experimental writes, not proof of safety and not a bypass of identity, ownership, locking, or application checks. The installed binary's exact source revision is **unresolved**. No production setting was changed for this documentation.

**Operator recollection:** on an earlier attempt, an access reportedly failed without `--force` and apparently worked with it. The command, exact error text and technical cause are no longer known. This recollection is not a technically verified finding, and `--force` must not be used to override failed safety checks.

### Installed artifacts and recovery decision

The observed `ugreenctl`, `ugreenctl-fand` and `dxp4800plus.so` files were `root:root`, `0755`; file paths, sizes and hashes are in [recovery files and permissions](RECOVERY_FILES_AND_PERMISSIONS.md), and build IDs and comparisons in [fan control details](FAN_CONTROL_DETAILS.md). The optional historical `ugreenctl-fand.before-alarm` file's use is unknown. The inventory found no Debian package owner for the three main artifacts, and the running process image could not be independently compared with the files.

The installed CLI and the comparison build were not byte-identical; the CLI's historical origin is **unknown**. Recovery uses a new build from the Debian branch rather than requiring the original binary. A verified old copy is optional rollback material, not a prerequisite. The [technical evidence](TECHNICAL_EVIDENCE.md) preserves the historical candidates and their limitations.

### Runtime dependencies and recovery files

The observed programs use the dynamic loader and `libc6`; `dxp4800plus.so` is a dynamically loaded application dependency. The root-run service, configuration and unit permissions, checksums, runtime paths, speaker access, and ACL limitations are recorded in [recovery files and permissions](RECOVERY_FILES_AND_PERMISSIONS.md). The branch example and unit were byte-identical to the observed files at the earlier comparison; that does not prove their installation origin.

### Control-path assessment and measurement boundaries

**Directly observed:** static analysis of the installed plugin found both hwmon and direct IT8613 functions and `ioperm` references. The daemon's state file was updated, reporting `model=dxp4800plus`, `profile=stock-4800plus`, `status=running`, and at one snapshot applied CPU PWM `60` and system PWM `65`. These are **software-reported** values, not independently measured hardware values.

**Source-derived:** the inspected model code attempts hwmon first and, specifically when no device is found (`-ENODEV`), describes a guarded direct IT8613 Super-I/O fallback; the direct implementation describes control/PWM writes and readback. Together with the absent `it87`/`it8613` hwmon path and the installed plugin's `ioperm` references, the strongly suspected production path is: temperature readings → `ugreenctl-fand` → `ugreenctl` → `dxp4800plus` plugin → direct IT8613 Super-I/O. **The actual execution of a direct port access was not observed.** A subsequent comparison build produced byte-identical daemon and plugin files, but the CLI differed and the historical build origin remains unproven. Neither the state file nor a lock-file existence proves a particular backend was executed.

The CPU and system channel mappings, including PWM/control and low/high tachometer registers, are recorded in the table in [Hardware and operating-system scope](#hardware-and-operating-system-scope). They are **source-derived model mappings**, not independently verified physical wiring or a port-access procedure.

Keep these evidence levels separate: (1) software-reported requested/applied PWM in the state file; (2) actual hardware register readback, which the inspected source describes but this inventory did not independently observe; (3) independently measured fan RPM; and (4) actual airflow. The latter two were not independently measured in this inventory. None establishes the next.

**Historically reported:** earlier CPU-load and speaker observations are retained under [Validation](#validation), but were not repeated by these inventories.

**Unresolved:** exact installed source revision and build, direct observation of the executed Super-I/O path, complete sensor coverage including standby drives, physical channel mapping and independently measured RPM/airflow, failure behavior and safe rollback, and complete boot and journal history. The non-root inventory could not read all system journal entries; no visible entries do not prove that none exist.

### Temperature sources and observed coverage

**Directly observed on 1 November 2026 at about 17:47 CET:** `coretemp` exposed CPU package and core temperatures; two `nvme` hwmon devices each exposed three readings. The highest CPU value read in this non-atomic snapshot was 48 °C. `nvme0` / Samsung SSD 990 PRO 1TB at PCI `0000:02:00.0` had 44.85, 44.85 and 53.85 °C; `nvme1` / AirDisk 128GB SSD at PCI `0000:04:00.0` had 41.85, 41.85 and 42.85 °C. These are sysfs controller associations, not independently verified physical slots; values change with time. Other visible sources were `acpitz` 27.8 °C, network PHY/MAC 51 °C and `spd5118` 40.25 °C; their hwmon names are not selected by the inspected fan-temperature filters.

One visible rotating ATA disk, `sda` / `WDC WD60EFRX-68M`, had **no visible `drivetemp` hwmon source**. The separately read daemon state at 17:46:54 nevertheless reported `cpu_celsius=44`, `cpu_peak_celsius=46`, `hdd_celsius=35`, `ssd_celsius=53`, `status=running`. These are software-reported values from a different instant, not independent proof of sensor provenance, per-disk coverage, fan RPM or airflow. An ATA SMART fallback described in the source is a plausible HDD-value origin, but no such transaction or actual source was observed. This inventory did not issue ATA/SMART commands; the already running daemon may independently do so according to its implementation.

**Source-derived:** CPU names `coretemp`, `k8temp`, `k10temp` and `zenpower` are recognized. The displayed `cpu_celsius` is the first valid glob-sorted temperature of the first usable matching device; `cpu_peak_celsius` is the maximum over matching CPU readings and is preferred for curve evaluation. HDD `drivetemp` and NVMe `nvme` / `nvme-pci` use their respective maximum valid readings across devices and attributes, regardless of labels. If no valid HDD hwmon value exists, the code may check ATA power mode and perform SMART reads on eligible active rotating ATA disks; a valid `drivetemp` reading for one HDD suppresses that global fallback even if other HDDs lack readings. The implementation is not a promise of complete or wake-free coverage for every disk or bridge.

The on-disk production configuration has `require_storage_sensor=true`: at least one valid HDD **or** NVMe temperature suffices, not one per installed drive. Missing CPU or both storage categories requests configured failsafe PWM on both channels; an actual write or cooling is not guaranteed. The visible CPU and NVMe values and the reported state indicate available sources at this snapshot, but HDD provenance, standby behavior, complete coverage and safety sufficiency remain **unresolved**. Full values, code-selection details and limitations: [fan control details](FAN_CONTROL_DETAILS.md).

## Source-based recovery: chronological procedure

This procedure is for a **future, separately authorized recovery**, not an instruction to operate on the current NAS. The source and build steps do not themselves command fan hardware; OS/package installation and copying files change the target system. Enabling the daemon immediately performs hardware operations, including possible PWM writes and guarded ATA SMART ioctls. **A complete fresh-install recovery, the rebuilt CLI's direct fallback and independent cooling have not been validated.** If a safe backend or independent cooling cannot be established, stop rather than force activation.

1. **Prepare and install Debian.** Arrange a maintenance window, console/out-of-band access, independently monitored cooling, and a safe shutdown and rollback plan. Install stock Debian 13 amd64 on the intended DXP4800 Pro; verify the exact DMI product is `DXP4800 Pro`. Do not run a second fan controller. Back up existing configuration and units before overwriting anything.
2. **Preflight the target before building or writing fan settings.** Confirm exact DMI, exclusive controller ownership and CPU/storage sensor coverage. These checks do not intentionally change fan settings; service-name searches alone are not exhaustive:

   ```sh
   cat /etc/os-release
   uname -r
   uname -m
   cat /sys/class/dmi/id/product_name
   systemctl list-units --type=service --all | grep -Ei 'fan|hwmonitor' || true
   lsmod | grep -E '^(it87|ug_it86x_sio|ug_it86x_cpufan|ug_201x_sio) ' || true
   test -e /proc/it86 && echo 'Vendor interface present: investigate ownership'
   modinfo it87 2>/dev/null | head -n 20
   for d in /sys/class/hwmon/hwmon*; do
     [ -r "$d/name" ] || continue
     printf '%s: ' "$d"; cat "$d/name"
   done
   ```

   If `lsmod` or `modinfo` is unavailable, the result is **unknown**, not proof that a driver is absent. `kmod` supplies these tools and can be installed in the authorized package step below. The hwmon name listing alone does not establish temperature coverage: inspect suitable CPU and storage readings independently, without waking drives, under an approved plan. Inspect the backend without direct port probing. A unique `it8613` hwmon node is preferred when compatible `it87` is already available; `modinfo` or the node name alone does not prove chip identity or binding. If hwmon is absent while `it87` is loaded, stop and investigate. If no hwmon node and no `it87` or vendor owner exist, the direct fallback is **experimental** and not independently validated here; use it only under a separately approved plan with chip/logical-device identification, owner checks, lock, PWM 40–255 and readback. Never load, unload or bypass a module merely to force fallback. `--force` does not bypass protections; direct writes also require `--apply`. A dry run without `--apply` does not prove cooling or a successful write. Do not use `fan status` when hwmon is absent: it may access Super I/O. `thermal status` and the daemon may issue ATA SMART ioctls.
3. **Obtain the source.** On the fresh target, ensure `git` is available; if not, install the Debian `git` package as an authorized system change. Clone the specified fork and select the target branch in a new directory:

   ```sh
   git clone --branch debian13-dxp4800pro https://github.com/to-pse/UGREEN-NAS-Hardware.git ugreenctl-recovery
   cd ugreenctl-recovery
   git status --short --branch
   git log -1 --oneline
   ```

   Record the resolved commit and review the branch, `examples/fan-curve-dxp4800pro.conf` and `packaging/systemd/ugreenctl-fand.service`: a branch can advance. The inventory's target commit was `adf00ec7113171cbc42b6b8088b33dfe1a3747b3`; it is a historical reference, not proof that any later branch tip has identical behavior.
4. **Install build prerequisites** as an authorized system change:

   ```sh
   sudo apt update
   sudo apt install build-essential cmake python3 kmod
   ```

5. **Configure and build** entirely from the new clone; no old local build directory or original CLI binary is needed:

   ```sh
   cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
     -DCMAKE_INSTALL_PREFIX=/usr/local -DCMAKE_INSTALL_LIBDIR=lib
   cmake --build build -j "$(nproc)"
   ```

6. **Run the documented fixture tests:**

   ```sh
   ctest --test-dir build -R '^(thermal-curve|fan-failure)$' --output-on-failure
   ```

   Stop on failure or if either named test is absent. These tests do not validate physical fans, backend availability, or the full recovery. Review other tests for hardware and filesystem effects before running them. Recheck ownership and sensor availability before activation; conditions can change after preflight.
7. **Install the built artifacts** only after authorization and backup:

   ```sh
   sudo cmake --install build
   ```

   This existing CMake install step installs `/usr/local/bin/ugreenctl`, `/usr/local/bin/ugreenctl-fand`, `/usr/local/lib/ugreenctl/models/dxp4800plus.so` **and other project binaries, plugins and documentation**; it can overwrite files. It does not install the fan configuration or unit.
8. **Install the reviewed configuration** using the existing example, only after confirming the destination and backing up any existing file:

   ```sh
   sudo install -d -o root -g root -m 0755 /etc/ugreenctl
   sudo install -o root -g root -m 0644 examples/fan-curve-dxp4800pro.conf \
     /etc/ugreenctl/fan-curve.conf
   ```

   The example includes `require_storage_sensor=true`, `failsafe_pwm=255` and the security-sensitive `allow_unvalidated_writes=true`; review it for the actual target. The requirement is at least one HDD **or** NVMe reading, not every drive. Missing CPU or both storage categories requests failsafe on both channels, but an unsuccessful write means cooling is not guaranteed.
9. **Install the systemd unit** only after authorization and backup:

   ```sh
   sudo install -o root -g root -m 0644 packaging/systemd/ugreenctl-fand.service \
     /etc/systemd/system/ugreenctl-fand.service
   ```

   Before activation verify the installed binary, plugin, configuration and unit paths, ownership and permissions; see [recovery files and permissions](RECOVERY_FILES_AND_PERMISSIONS.md). Confirm exclusive ownership, sensors and independent cooling. The unit runs as root and creates its runtime directory.

   **Hardware operation:** starting the controller can immediately change PWM and issue ATA SMART ioctls. Only after a separate authorization and these checks, activate it:

   ```sh
   sudo systemctl daemon-reload
   sudo systemctl enable --now ugreenctl-fand
   ```

   Stop before starting the service if the approved safety conditions are not met.
10. **Verify under supervision:** inspect `systemctl status ugreenctl-fand --no-pager`, `journalctl -u ugreenctl-fand.service -b --no-pager` and `/run/ugreenctl/fan-curve.state`. Confirm a fresh timestamp, normal status and no errors, and independently monitor both fan RPM and temperatures/airflow. A software state or requested failsafe alone does not prove either fan is cooling. Check actual HDD/NVMe coverage and standby behavior separately under an appropriate plan. If cooling cannot be confirmed, reduce load and shut down safely; do not overlap controllers during rollback.

## Recovery

The unit runs as root, creates `/run/ugreenctl`, and uses `Restart=on-failure`, `RestartSec=3`, and `TimeoutStopSec=10`. It is ordered after `local-fs.target`, not explicitly after `it87` or temperature sensors. A clean exit, hung process, persistent write error, sensor loss or systemd start limit may not be repaired by restart. A stopped daemon does not restore a prior curve or establish hardware automatic mode; PWM persistence through driver or machine stops is not guaranteed.

Before an authorized maintenance window, prepare backed-up working binaries, plugin, unit and configuration and an independent cooling fallback. Stop the replacement controller before restoring a previous controller; never overlap them. Manual PWM is a hardware write requiring separate authorization and is not a substitute for supervised automatic control. If safe cooling cannot be confirmed, reduce load and shut down safely.

For a later separately authorized recovery, retain and independently verify a protected, versioned backup of:

1. When available, verified copies of the installed `ugreenctl` and `ugreenctl-fand` files and the `dxp4800plus.so` plugin, with paths, permissions, owners, hashes and build IDs, for rollback; retain `ugreenctl-fand.before-alarm` only as optional historical material. The old CLI is not required for the selected source-based recovery.
2. The productive `fan-curve.conf`, systemd unit and start symlink, plus any subsequently discovered drop-ins or environment files, with permissions and ownership. Handle possible credentials separately and do not publish them.
3. The exact source trees and any local modified or untracked work, without disturbing either worktree; preserve known branch/commit IDs, build configuration, logs and toolchain versions. The original installed CLI source/build remains unknown; a newly built CLI must not be presented as a byte-identical historical match.
4. Compatible Debian amd64 runtime dependencies, particularly the loader, `libc6` and required plugin ABI, plus documented package versions and trusted sources. Preserve the compiler, CMake, binutils, make, development headers and flags needed for a future rebuild.
5. Runtime directory, sensor, lock, DMI and optional speaker requirements as **recreated runtime conditions**, not as proof supplied by copied stale state/lock files. Verify backup integrity and plan a supervised recovery test with independent cooling; the full restoration has not yet been practically validated.

After a kernel update or reboot, re-check the actual control backend, any loaded driver and its compatibility, any unique dynamic `it8613` node, both RPM readings, temperature coverage, unit ordering, state and journal. The inventoried production system had no `it8613` node; do not load or unload a module merely to make the expected path appear. A separately sourced module may need rebuilding and signing for a new kernel. Do not assume hwmon numbers, module availability, or sensor ordering persist.

If the daemon cannot be started, investigate the error and ensure the hardware remains adequately cooled. Do not leave the system under significant load without functioning fan control.

## Validation

The previous branch documentation reports the following observations on Debian 13 with UGREEN DXP4800 Pro; they are not new tests performed during this documentation task and do not establish fresh-install or complete hardware validation:

- Successful CMake build
- `thermal-curve` and `fan-failure` tests passed
- PC-speaker tone successfully tested
- CPU stress test: CPU temperature increased from approximately 39 °C to 56 °C
- CPU PWM increased from 60 to 79
- System PWM increased from 65 to 83
- PWM values returned to their initial levels after cooling

Automatic alarm triggering during a real fan-control failure has not been tested on production hardware.

The preceding documentation work also successfully built with CMake and ran the hardware-independent `thermal-curve` and `fan-failure` tests, 2 passed and 0 failed; no hardware operation was performed for that work. Firmware thresholds and controller evidence are documented in [FAN_CURVE.md](FAN_CURVE.md) and [HARDWARE_SUPPORT.md](HARDWARE_SUPPORT.md); the analyzed universal-image SHA-256 is `7bb2746324ac852475727cf23f7016517996b91dfb293d2292d531c1e71581b0`.

Still open: `it87` compatibility/binding on the inventoried kernel, exact installed source revision and direct-access proof, full CPU/HDD/NVMe coverage including standby disks, boot ordering and complete journal history, physical fan mapping and independent RPM, per-channel failsafe and actual airflow, crash/restart behavior, and supervised rollback. A fresh-install test, module operations, sensor-failure injection, PWM writes, load tests, reboots and kernel-update tests were not performed during the documentation and inventory tasks. Perform any future hardware validation only with explicit authorization and an independently safe cooling setup.
