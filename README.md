# UGREEN DXP Fan CLI

Simple fan control CLI for UGREEN DXP4800 Plus, DXP6800 Pro and iDX6011 Pro NAS systems running Debian or Proxmox with the UGREEN `it87` hwmon driver.

This repo does not replace the kernel driver. It installs a clean `fan` command and a boot service that applies a safe hardware-auto curve by default.

Big credit goes to [IT-Kuny/UGREEN-DXP-FAN-NAS-Driver](https://github.com/IT-Kuny/UGREEN-DXP-FAN-NAS-Driver). That project did the hard hardware-driver work that makes this CLI possible; this repo is a user-facing control layer on top of it.

Tested on:

- UGREEN DXP4800 Plus
- Proxmox VE 8.4
- IT8613E exposed through the out-of-tree `it87` driver
- Fan channels `pwm2` and `pwm3`

Also supported:

- UGREEN iDX6011 Pro, through the `it5571` EC backend in the UGREEN `it87` driver. Fan channels `pwm1`-`pwm4` (CPU pair `pwm1`/`pwm2`, system pair `pwm3`/`pwm4`). See [iDX6011 Pro](#idx6011-pro).

## Install

First install and load the UGREEN-compatible `it87` driver. The hardware should expose an `it8613` (DXP4800 Plus) or `it5571` (iDX6011 Pro) device under `/sys/class/hwmon`.

Then install this CLI:

```sh
curl -fsSL https://raw.githubusercontent.com/smashah/ugreen-dxp-fan-cli/main/install.sh | sudo bash
```

The installer:

- installs `/usr/local/bin/fan`
- installs `/usr/local/sbin/ugreen-fan-mode` as a compatibility symlink
- writes `/etc/ugreen-fan.conf`
- writes `it87` module-load config
- creates and enables `ugreen-fan-auto.service`
- creates and enables `ugreen-fan-graph.timer`
- applies `fan auto` immediately
- starts graph collection immediately

Default auto target is `35c`. To install with a different target:

```sh
curl -fsSL https://raw.githubusercontent.com/smashah/ugreen-dxp-fan-cli/main/install.sh | sudo bash -s -- --target 40
```

Graph collection is enabled by default at a 10-second interval. To change that during install:

```sh
curl -fsSL https://raw.githubusercontent.com/smashah/ugreen-dxp-fan-cli/main/install.sh | sudo bash -s -- --graph-interval 30
```

To install without graph collection:

```sh
curl -fsSL https://raw.githubusercontent.com/smashah/ugreen-dxp-fan-cli/main/install.sh | sudo bash -s -- --no-graph
```

## Usage

```sh
fan              # show current mode, PWM values, RPMs, curve, and sensors output
fan status       # same as above
fan auto         # apply saved hardware auto curve
fan 35c          # save/apply auto target temperature
fan auto 35c     # same as fan 35c
fan full         # full-speed mode
fan max          # same as fan full
fan 255          # manual raw PWM value
fan 50%          # manual percentage
fan off --yes    # manual PWM 0, guarded because it can overheat the NAS
fan graph        # terminal graph for the last 24h
fan graph status # graph config, timer state, and sample count
fan graph on     # enable graph collection
fan graph off    # disable graph collection
fan graph interval 30 # poll every 30 seconds
```

`fan 35c` persists the target in `/etc/ugreen-fan.conf`, so the same target is used by the boot service.

`fan 255` may show as `full` on this driver. That is expected: the IT8613E `it87` driver reports max duty as full-speed mode.

## Auto Curve

For the default `35c` target the CLI programs:

| Channel | Sensor | Point 1 (stop) | Point 2 (start) | Point 3 (full) | Start PWM |
| --- | --- | ---: | ---: | ---: | ---: |
| `pwm2` | `temp1` | `0c` | `35c` | `55c` | `180` |
| `pwm3` | `temp2` | `0c` | `32c` | `42c` | `150` |

In the `it87` curve, the fan stops below point 1, runs at the start PWM from point 2, and reaches full speed at point 3. Point 1 is always `0c`, so `fan auto` never stops a fan.

Changing the target shifts points 2 and 3:

- `pwm2`: `target`, `target + 20c`
- `pwm3`: `target - 3c`, `target + 7c`

This is intentionally conservative. On the DXP4800 Plus tested, the previous invalid/weak auto settings let the CPU package hit the thermal limit. This curve keeps hardware auto mode but gives it a more useful fan response.

## DXP6800 Pro

The DXP6800 Pro uses the same IT8613E chip as the DXP4800 Plus, with three fans:

| Channel | Fan | `fan auto` |
| --- | --- | --- |
| `pwm2` | CPU fan | Hardware curve on `temp1` (CPU) |
| `pwm3` | Rear system fan | Hardware curve on `temp2` (board) |
| `pwm4` | Rear system fan | Fixed `FIXED_PWM_PERCENT` (default 40%) |

`CHANNELS=auto` selects `pwm2 pwm3 pwm4` when the DMI product name is `DXP6800 Pro`. Configs written by older installers contain `CHANNELS="pwm2 pwm3"`; change that line to `CHANNELS=auto` to include `pwm4`.

The `it87` driver cannot program an IT8613E `pwm4`/`pwm5` curve: their `auto_point` temperature attributes write `pwm3`'s registers. `fan auto` therefore runs `pwm4`/`pwm5` in manual mode at `FIXED_PWM_PERCENT` (20-100). No hardware curve on this board follows drive temperatures; raise `FIXED_PWM_PERCENT` for sustained drive workloads.

The stock kernel `it87` does not support the IT8613E. Do not work around it with `force_id=0x8628`: the IT8628E register layout differs for PWM4/PWM5.

## iDX6011 Pro

The iDX6011 Pro fans are driven by the embedded controller, which supports only manual duty and its own firmware curve:

| Command | iDX6011 Pro behavior |
| --- | --- |
| `fan auto`, `fan 35c` | Returns all fans to the EC firmware curve (`pwmN_enable=2`). `fan 35c` saves the target, but the EC cannot use it. |
| `fan full`, `fan max` | Manual mode at PWM 255. |
| `fan 50%`, `fan 128` | Manual mode at that duty on every channel. |
| `fan status` | Shows `-` for PWM on channels in EC auto mode, where the duty register holds the last manual value. |

For temperature-following curves on this model, use `ugreen-fan-control.service` from [IT-Kuny/UGREEN-DXP-FAN-NAS-Driver](https://github.com/IT-Kuny/UGREEN-DXP-FAN-NAS-Driver) in a curve mode (`silent`, `quiet`, `turbo`) and keep `fan` for status and graphs. Run only one fan manager: that service writes PWM only when its computed target changes, so `fan` commands and `ugreen-fan-auto.service` at boot override it until then. Disable `ugreen-fan-auto.service` when using the curve service, and pass `--no-enable` when reinstalling `fan`.

The plain iDX6011 (non-Pro, `it8622` hwmon) is not supported by `fan`.

## Services

Check the boot service:

```sh
systemctl status ugreen-fan-auto.service
```

The service runs:

```sh
/usr/local/bin/fan auto
```

It is a one-shot service, so `active (exited)` is normal.

Graph collection uses a timer:

```sh
systemctl status ugreen-fan-graph.timer
```

The timer runs a short one-shot service:

```sh
/usr/local/bin/fan graph collect
```

There is no resident graph daemon. Each collection reads hwmon values once, appends one compact TSV row, and prunes history older than 24 hours.

The installer disables `fancontrol.service` only when the service exists but `/etc/fancontrol` is empty or missing. If CoolerControl or another fan manager is actively controlling these channels, disable that policy or it can overwrite `fan`.

## Configuration

`/etc/ugreen-fan.conf`:

```sh
AUTO_TARGET_C=35
CHANNELS=auto
GRAPH_ENABLED=1
GRAPH_INTERVAL_SEC=10
HISTORY_FILE="/var/lib/ugreen-fan/history.tsv"
FIXED_PWM_PERCENT=40
```

`CHANNELS=auto` uses `pwm2 pwm3` on `it8613`, `pwm2 pwm3 pwm4` on the DXP6800 Pro, and `pwm1 pwm2 pwm3 pwm4` on `it5571`. If another UGREEN model exposes different PWM channels, set `CHANNELS` explicitly, for example `CHANNELS="pwm1 pwm2"`.

## Fan Graph

Graph history is enabled by default and polls every 10 seconds:

```sh
fan graph
```

The graph stores only the last 24 hours. At the default interval that is at most 8640 samples. The history file is intentionally small, plain text, and pruned on every collection:

```sh
/var/lib/ugreen-fan/history.tsv
```

Useful commands:

```sh
fan graph status
sudo fan graph off
sudo fan graph on
sudo fan graph interval 30
```

`fan graph interval` accepts 5 to 3600 seconds. Longer intervals reduce writes and sample count. Shorter intervals below 5 seconds are rejected.

## Uninstall

```sh
sudo ./uninstall.sh
```

Or keep config files:

```sh
sudo ./uninstall.sh --keep-config
```

## Development

Run syntax checks and the fake-hwmon smoke test:

```sh
bash -n fan install.sh uninstall.sh tests/smoke.sh tests/install.sh
bash tests/smoke.sh
bash tests/install.sh
```

## Safety

Fan control can overheat hardware. Keep `fan status` open and watch temperatures after changing modes. `fan off --yes` exists for testing only and should not be used unattended.

## Credits

This project depends on and is inspired by the UGREEN DXP `it87` driver work from [IT-Kuny/UGREEN-DXP-FAN-NAS-Driver](https://github.com/IT-Kuny/UGREEN-DXP-FAN-NAS-Driver). Please star and credit that repository too; without it, this CLI would not be useful.
