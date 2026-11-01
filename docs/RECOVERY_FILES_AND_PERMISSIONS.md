# DXP4800 Pro: Recovery files and permissions

This is a consolidated **1 November 2026** inventory, not proof of today's state or authorization to modify it. For the chronological procedure see [Debian 13 recovery](DEBIAN13_DXP4800PRO.md); for controller and sensor evidence see [fan control details](FAN_CONTROL_DETAILS.md). Source-based recovery uses the GitHub `debian13-dxp4800pro` branch; old build directories and the original CLI binary are **not required inputs**. A verified copy, if available, is useful optional rollback material because the historical CLI was not reproduced byte-for-byte.

Additional historical observations and discrepancies are available in [technical evidence](TECHNICAL_EVIDENCE.md).

## Runtime identity and access

**Observed on 1 November 2026 at 17:54 CET:** `ugreenctl-fand` ran as `root` UID/GID `0:0`, no supplementary groups; systemd reported `active/running`, `enabled`, no drop-ins. The speaker device belonged to group `input`. Group IDs and event device numbers may differ on another installation.

The process effective capabilities included `CAP_SYS_RAWIO`; that is relevant to source-described `ioperm`, but is **not** proof of a direct port operation or that other kernel/lockdown protections are satisfied. `PrivateDevices=no`, `DevicePolicy=auto`, no ambient capabilities were reported. Root did not require an added `input` group in the observed configuration; no least-privilege profile was validated.

## Persistent artifacts and ownership

**Observed on disk:** all following files are `root:root` (UID:GID 0:0). Hashes identify files at that time, not necessarily the memory-mapped process. Paths, ownership and permissions are significant for the observed configuration.

| File | Mode | Bytes | SHA-256 | Recovery treatment |
| --- | --- | ---: | --- | --- |
| `/usr/local/bin/ugreenctl` | `0755` | 303624 | `74d2b90cae975a496c58579c2bbfbb70214dd307e89ed726e8e2de30155aeafb` | Build new from GitHub; original optional rollback only, exact provenance unknown. |
| `/usr/local/bin/ugreenctl-fand` | `0755` | 39432 | `464db8865a9bb3ee1c9dd86361a24440dddf5e63dbc9e11772aeca8a95728dca` | Build from source; verified old copy optional rollback. |
| `/usr/local/lib/ugreenctl/models/dxp4800plus.so` | `0755` | 39744 | `9072081fded14e00eff147f8c324c52eeb2edbd28ddd65737be1fb4cd796832d` | Build matching plugin; verified old copy optional rollback. |
| `/usr/local/bin/ugreenctl-fand.before-alarm` | `0755` | 39264 | `afe7f4b0e854e44d7e7dd2b1779738112cc411ce32d767924d28da7bd17cec02` | Optional historical file, use not established. |
| `/etc/ugreenctl/fan-curve.conf` | `0644` | 209 | `b86c84a4735159080c89037bb3002df03beac3787140a171c0af56bff61d1692` | Save actual desired configuration; compare with branch example. |
| `/etc/systemd/system/ugreenctl-fand.service` | `0644` | 449 | `d0b916fa55bdbd75dc7d9455c0027c4fcf17e2d6809747e8ea09fb61ccb07718` | Save actual unit and review against branch example. |

`/usr/local/bin`, `/usr/local/lib/ugreenctl/models` and `/etc/ugreenctl` were `root:root`, `0755`. Only `dxp4800plus.so` was visible in the plugin directory. The actual source file `/etc/ugreenctl/fan-curve.conf` was:

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

The loaded profile was reported by the daemon; each in-memory setting was not independently confirmed. The example and unit in the Debian worktree were byte-identical to the observed files at the earlier comparison, but branch tips and productive settings may change. Preserve actual configuration, local modifications and untracked source work; do not publish secrets.

## Actual systemd unit and references

**Observed:** `/etc/systemd/system/ugreenctl-fand.service` contained:

```ini
[Unit]
Description=UGREEN NAS Fan Control (BearHero)
After=local-fs.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/ugreenctl-fand --config /etc/ugreenctl/fan-curve.conf --state /run/ugreenctl/fan-curve.state --ugreenctl /usr/local/bin/ugreenctl --plugin-dir /usr/local/lib/ugreenctl/models
Restart=on-failure
RestartSec=3
RuntimeDirectory=ugreenctl
RuntimeDirectoryMode=0755
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/multi-user.target.wants/ugreenctl-fand.service` was a `root:root` symlink to the unit, link mode `0777`. It captures a persistent enablement relationship; recreate only in an authorized recovery, not by copying stale runtime state. The examined unit had no `Environment=` or `EnvironmentFile=`, and no examined specific/generic drop-ins or `/etc/default/ugreenctl-fand` / `/etc/sysconfig/ugreenctl-fand` were present. Other sources and actual process environment were not fully accessible. Unit ordering did not explicitly wait for sensors or `it87`.

## Volatile files and optional alarm

| Resource | Observed metadata | Treatment |
| --- | --- | --- |
| `/run/ugreenctl` | directory `root:root`, `0755` | Created by unit; don't require an old copy. |
| `/run/ugreenctl/fan-curve.state` | regular file `root:root`, `0644`, then 262 bytes | Generated state; not proof of hardware airflow and not a recovery input. |
| `/run/ugreenctl-fand.lock`, `/run/ugreenctl-it8613.lock` | empty files `root:root`, `0600` | Created/used at runtime; copied files do not recreate locks. |
| `/dev/input/by-path/platform-pcspkr-event-spkr` | `root:root` symlink `0777`, then `../event7` | Optional alarm path, provided by system. |
| `/dev/input/event7` | character device `root:input`, `0660`, major/minor 13:71 | Event number and permission can change; writable device needed for audible alarm. |

The alarm is enabled in the observed config; absence of the device does not establish that the controller itself stops. Its real failure-triggered sound was not independently validated.

## Backup and rebuild classification

1. **Persistently save:** actual configuration, unit, relevant future drop-ins/environment files and their ownership/permissions, desired enablement relationship, exact source and local changes, toolchain and Debian dependency versions. Keep backups protected and verify integrity.
2. **Rebuild/copy from GitHub:** CLI, daemon and model plugin; reviewed example config and unit can be installed when appropriate. The earlier comparison reproduced daemon and plugin byte-for-byte under specific conditions, not necessarily on future toolchains; CLI differed. Old binaries and `.before-alarm` are optional rollback/history, not mandatory source-recovery inputs.
3. **Recreate at runtime:** `/run` directory, state, locks, device nodes and approved systemd enablement link. Don't restore stale lock/status files.
4. **System prerequisites:** compatible Debian amd64, systemd, loader and `libc6` (then `2.41-12+deb13u4`), DMI, suitable CPU/storage sensors, controlled backend permissions and ownership checks. For builds: C11 compiler, CMake 3.20+, build tool, libc development files; see the recovery guide for packages. Direct `it87` module binding is not established merely by an installed module file.

## Permission limitations and unresolved checks

`getfacl` was unavailable. No xattr names or file capabilities were visible on checked files; this is **not** a full ACL/LSM audit. `lsattr` was denied on both lockfiles; their special flags are unknown. The daemon's `/proc/<pid>/exe`, `maps` and `fd` were inaccessible in the earlier inventory. Actual process environment, minimal required permissions, kernel lockdown, sensor coverage, direct I/O execution, physical cooling and full fresh recovery remain unvalidated. Before a future recovery establish exclusive control, independent cooling, an authorized maintenance window and a safe rollback.
