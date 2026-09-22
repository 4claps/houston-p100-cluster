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

## Running llama-server in production

The patched llama.cpp build (see
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md)) runs as
a plain systemd service — no external process manager. An earlier setup
used `gppm` for this, mainly for its instance-supervision feature after
its actual power-management purpose turned out not to work on this
hardware (see [BENCHMARKING.md](BENCHMARKING.md)); it's been removed in
favor of systemd's own supervision, which does the same job with one
fewer moving part.

`llama-server-moe.service`:

```ini
[Unit]
Description=llama-server - Qwen3.6-35B-A3B production (patched P100 build, MTP speculative decoding)
After=network.target

[Service]
Type=simple
User=duncan
Group=duncan
ExecStart=/opt/llama.cpp-p100/build/bin/llama-server --host 0.0.0.0 --port 8080 -m /home/duncan/models/Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL.gguf -ngl 99 -ts 1/1/1 -fa on -sm layer --ctx-size 65536 --cache-type-k q8_0 --cache-type-v q8_0 --parallel 1 --jinja --cache-ram 0 --no-cache-idle-slots --spec-type draft-mtp --spec-draft-n-max 3
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`--spec-type draft-mtp --spec-draft-n-max 3` enables MTP (multi-token
prediction) speculative decoding — see [BENCHMARKING.md](BENCHMARKING.md)
for what it does and the measured gain. It needs an MTP-converted GGUF,
not the regular quant: the model file is
`Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL.gguf` (unsloth's
`Qwen3.6-35B-A3B-MTP-GGUF` repo), not the plain
`Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` used earlier — same quant level,
~500MB larger for the added draft head, otherwise the same model
weights.

`--cache-ram 0 --no-cache-idle-slots` disables llama-server's cross-request
host-RAM prompt cache — carried over from the bc250 cluster's own
production config after their bake-off found the opposite (cache left on)
could get the server OOM-killed when unrelated large prompts piled up in
it. Worth keeping on any future llama-server production unit on this box.

### The binary has to live under `/opt`, not `/home`

The build lives at `~/src/llama.cpp-v040-patched/build` (see
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md)), but a
systemd unit **cannot execute a binary there directly** — Fedora's SELinux
policy denies `init_t` (systemd's own domain) from executing anything
labeled `user_home_t`, which is the default label for everything under a
user's home directory. This isn't a permissions issue `chmod` can fix; it
showed up first with `gppm`'s own Python venv (`Unable to locate
executable ... Permission denied` from systemd, an SELinux AVC `denied
{ read }` on the interpreter symlink underneath), and the same restriction
applies to `llama-server` itself.

The fix is the same one used for `gppm`: copy the build to somewhere
under `/opt`, which gets the `usr_t` label systemd is allowed to execute:

```
sudo mkdir -p /opt/llama.cpp-p100
sudo cp -r ~/src/llama.cpp-v040-patched/build /opt/llama.cpp-p100/build
sudo chown -R duncan:duncan /opt/llama.cpp-p100
```

Note that the copied binary still links its shared libraries (`libggml-cuda.so.0`
etc.) from the *original* `~/src/...` path — the build bakes in an
absolute RPATH rather than a relative one, so copying the tree doesn't
change where those get loaded from. That turned out not to matter:
loading a shared library from `user_home_t` via the dynamic linker is not
the same SELinux operation as `execve`-ing a binary or symlink from
there, and it was confirmed working with a `systemd-run` test before this
was relied on for the real service. Only the entry-point binary itself
needs to live under `/opt`; if that stops being true (e.g. after a
different SELinux policy update), rebuilding with `-DCMAKE_INSTALL_RPATH
'$ORIGIN'` or using `patchelf` would be the real fix rather than copying
the whole tree.
