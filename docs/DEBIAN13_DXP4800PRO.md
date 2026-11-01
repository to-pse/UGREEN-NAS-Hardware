# DXP4800 Pro – Debian 13 Fan Control Setup

This document describes the tested fan-control setup on a UGREEN NASync DXP4800 Pro running Debian 13.

## Build and install

Install the required build dependencies as documented in the project's README.

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DCMAKE_INSTALL_LIBDIR=lib

cmake --build build -j "$(nproc)"
ctest --test-dir build -R '^(thermal-curve|fan-failure)$' --output-on-failure
sudo cmake --install build
```

The relevant installed files are:

- `/usr/local/bin/ugreenctl`
- `/usr/local/bin/ugreenctl-fand`
- `/usr/local/lib/ugreenctl/models/dxp4800plus.so`

## Install fan-control configuration

```bash
sudo install -d /etc/ugreenctl

sudo install -m 644 examples/fan-curve-dxp4800pro.conf \
  /etc/ugreenctl/fan-curve.conf

sudo install -m 644 packaging/systemd/ugreenctl-fand.service \
  /etc/systemd/system/ugreenctl-fand.service
```

## Enable the service

Before enabling the service, verify that the CLI, daemon, model plugin and configuration have been installed correctly.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ugreenctl-fand
```

## Verify operation

```bash
systemctl status ugreenctl-fand --no-pager
cat /run/ugreenctl/fan-curve.state
```

Expected normal operation:

- `status=running`
- Applied CPU and system PWM values match the requested values.
- `detail=` contains no error.

The software daemon is responsible for active fan control. Do not run multiple fan-control daemons simultaneously.

## Optional PC-speaker alarm

The example configuration enables the PC-speaker alarm:

```ini
failure_threshold=3
beep_on_failure=true
beep_repeat_seconds=60
```

The alarm uses `/dev/input/by-path/platform-pcspkr-event-spkr` and generates a 100 ms tone at 1000 Hz.

The speaker device must be available for audible alarms. Failure to sound the alarm does not stop fan control.

## Recovery

If a new daemon version causes problems, restore a previously backed-up working binary and restart the service.

If the daemon cannot be started, investigate the error and ensure the hardware remains adequately cooled. Do not leave the system under significant load without functioning fan control.

## Validation

Tested on Debian 13 with UGREEN DXP4800 Pro:

- Successful CMake build
- `thermal-curve` and `fan-failure` tests passed
- PC-speaker tone successfully tested
- CPU stress test: CPU temperature increased from approximately 39 °C to 56 °C
- CPU PWM increased from 60 to 79
- System PWM increased from 65 to 83
- PWM values returned to their initial levels after cooling

Automatic alarm triggering during a real fan-control failure has not been tested on production hardware.
