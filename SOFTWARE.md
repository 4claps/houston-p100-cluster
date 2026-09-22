# Software

## Base OS

Fedora 44 Server, headless.

## NVIDIA driver

Installed from **RPM Fusion's legacy 580xx branch** specifically:

```
akmod-nvidia-580xx
xorg-x11-drv-nvidia-580xx-cuda
```

This isn't the default choice — Fedora 44's regular `akmod-nvidia`
package pulls driver 590 and later, and that branch **dropped Pascal
architecture support entirely**. Without the `580xx`-suffixed packages,
these three P100s simply wouldn't be recognized by the driver at all.
Anyone reinstalling or updating this box needs to keep pulling from the
`580xx` branch specifically, not just "whatever RPM Fusion's nvidia
package currently is" — a routine `dnf upgrade` that lets the driver
float to the newest branch would silently break all three GPUs.

Currently installed: driver `580.178.04`, which also brought CUDA 13.0
API support. Note that CUDA 13's own compiler toolchain (`nvcc`) has
*separately* dropped the ability to **compile** for Pascal
(`sm_60`/`compute_60`), even though the 580xx driver runs Pascal cards
fine — building anything CUDA-based for these cards needs a CUDA 12.x
toolchain's `nvcc`, kept alongside the newer driver, not instead of it.

## GPU fan control

The P100's stock cooling assumes rack airflow (see
[HARDWARE.md](HARDWARE.md)), so shroud fan speed is driven off actual GPU
temperature rather than the motherboard's own fan curve, which has no
visibility into GPU thermals at all.

This is a small systemd service, `gpu-fan-control.service`, running
`/usr/local/bin/gpu-fan-control.sh`:

```bash
#!/bin/bash
HWMON=$(for d in /sys/class/hwmon/hwmon*; do
  [ "$(cat $d/name)" = "nct6776" ] && echo "$d"
done)

PWM="$HWMON/pwm2"
PWM_ENABLE="$HWMON/pwm2_enable"
echo 1 > "$PWM_ENABLE"

while true; do
  TEMP=$(nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits | sort -n | tail -1)
  if [ "$TEMP" -le 40 ]; then SPEED=70
  elif [ "$TEMP" -ge 75 ]; then SPEED=255
  else SPEED=$(( 70 + (TEMP - 40) * (255 - 70) / (75 - 40) )); fi
  echo "$SPEED" > "$PWM"
  sleep 5
done
```

It polls `nvidia-smi` for the hottest of the three GPUs every 5 seconds
and linearly ramps the shroud fans between a 70/255 idle floor and full
speed across a 40-75°C window.

The motherboard's Super I/O chip is an **nct6776**, and the shroud fans
turned out to be wired to its **`pwm2`** channel specifically — this
wasn't documented anywhere and wasn't reliably found by `pwmconfig`'s
automated detection wizard (it either missed the channel or gave
ambiguous/inconsistent results across runs). It was found the reliable
way instead: manually writing values to each `pwmN` file under
`/sys/class/hwmon/hwmonX/` one at a time and watching/listening for which
physical fan actually responded.

If this service is ever reinstalled from scratch on different hardware or
after a board swap, don't assume `pwm2` still maps to the same physical
fan header — repeat the manual cycling to confirm.

## SSH / remote access

Remote access for automated tooling (Claude Code / Hermes) is through a
**dedicated `hermes` system user**, rather than a personal account — the
same access pattern used elsewhere in this infrastructure. This account
is deliberately scoped down:

- No sudo access (`hermes` is not in `wheel` or any sudo-capable group)
- No group memberships beyond its own primary group
- A single authorized key in `~/.ssh/authorized_keys`, nothing else

The intent is that anything an agent does on this box through that
account is limited to whatever `hermes`'s own file permissions allow —
it isn't a backdoor into a personal account or root, and any change that
actually needs elevated privilege has to go through a human with real
credentials on this box, not through an agent's SSH session.
