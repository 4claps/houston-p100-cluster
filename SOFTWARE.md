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

**Current state (2026-10-06):** the temperature-driven service above is the one running.
A second unit, `fans-full.service` ("Force all case fans to full speed", running
`/usr/local/bin/fans-full.sh`), exists for benchmark runs: it was in use from 2026-10-03
to 2026-10-06 and for the four-card test, and holds the shroud fan at about
2,850-3,050 RPM. It is not enabled at boot. Only one of the two should run at a time
(stop one, start the other). The measurements in `3gpu-optimization/` were taken with
the fans at full; at full speed the cards sat at 34-39°C idle with a model loaded and
peaked at 65-68°C during sustained prompt processing at 150 W. Temperatures under the
temperature-driven service at 150 W on four cards have not been measured.

## GPU power limit

All three P100s are power limited to **150 W** (the P100's minimum is 125 W; the
default is 250 W). The limit was 125 W until 2026-10-05; that is the cap the
agent-battery baseline in
[BENCHMARKING.md](BENCHMARKING.md#current-baseline-agent-battery-with-a-125-w-power-cap)
was measured under. It used to be applied by hand and was lost on every reboot; since
2026-09-29 it is applied at boot by a systemd unit, and `nvidia-persistenced.service` is
enabled so NVIDIA persistence mode is on for all three cards.

Why 150 W: a sweep on the current production stack (Qwen3.8-27B on the Kmic-68 fork,
tensor split, NCCL) found the 125 W cap was limiting throughput, with the driver's
power-cap reason active in up to 80% of busy samples during generation.

| Cap | Generation (tg512) | Prompt processing (pp2048) | GPU power while generating, 3 cards | Hottest card |
|---|---|---|---|---|
| 125 W | 31.25 t/s | 506.0 t/s | 356 W | 54°C |
| 150 W | 33.29 t/s (+6.5%) | 530.4 t/s (+4.8%) | 430 W | 58°C |
| 175 W | 33.66 t/s (+7.7%) | 549.0 t/s (+8.5%) | 484 W | 60°C |

150 W takes most of the generation gain; 175 W adds almost nothing to generation for
another 54 W. Generation per GPU watt is 12% lower at 150 W than at 125 W. Peak draw of
the three cards plus the CPU package was about 500 W at 150 W. Instantaneous per-card
peaks at the 150 W cap reached 169-177 W in later use; the cap is an average, not a
hard ceiling. Full method and data: section 1.11 of the
[3-GPU measurements](3gpu-optimization/RESULTS.md).

`/etc/systemd/system/nvidia-power-limit.service`:

```ini
[Unit]
Description=Set NVIDIA GPU power limit
After=nvidia-persistenced.service
Wants=nvidia-persistenced.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/nvidia-smi -pl 150

[Install]
WantedBy=multi-user.target
```

`ExecStart` has no `-i` flag, so the limit applies to every GPU present at boot. To set
this up on a rebuilt box:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nvidia-persistenced.service nvidia-power-limit.service
```

Check it with `nvidia-smi --query-gpu=index,persistence_mode,power.limit --format=csv`;
all three should show `Enabled` and `150.00 W`.

**If a card of a different model is added,** a single blanket `-pl 150` may be wrong for
it. Run `nvidia-smi -L` first to confirm the GPU indexes (they can shift when a card is
added), then replace the single `ExecStart` with one line per GPU:

```ini
ExecStart=/usr/bin/nvidia-smi -i 0 -pl 150
ExecStart=/usr/bin/nvidia-smi -i 1 -pl 150
ExecStart=/usr/bin/nvidia-smi -i 2 -pl X
```

After editing the unit: `sudo systemctl daemon-reload && sudo systemctl restart nvidia-power-limit.service`

## Kernel arguments

`pcie_aspm=off` was added to the kernel command line with `grubby` while
troubleshooting the riser problem described in
[TROUBLESHOOTING.md](TROUBLESHOOTING.md#gpu-falls-off-the-bus-xid-79-and-xid-154).
It did not fix anything and has no effect on the P100s, which report "ASPM not
supported". It is harmless, so it was left in place, but don't count it as part of the
fix and don't carry it over to a rebuild expecting it to matter.

## Running llama-server today

Production models are launched through **Docker Compose**, not directly as a systemd service.
The Compose setup (two profiles — one per model currently in rotation — with the exact commands,
which model file each uses, and a RUNPATH gotcha worth reading before touching the file layout)
lives alongside the model files themselves on this box, not in this repo; see its own README for
the how-to rather than looking for it here.

`llama-server.service` (below) used to be left `enabled` at the systemd level, which meant a
reboot could let it auto-start and grab the GPUs out from under Compose. As of 2026-10-05 it is
**masked**, so it can no longer start.

### What production runs now (2026-10-06)

The Compose profile in use serves **Qwen3.8-27B, unsloth `UD-Q6_K_XL`**, on the
[Kmic-68 fork](https://github.com/Kmic-68/llama.cpp) at `ae35056eb`, built with NCCL
(`-DGGML_CUDA_NCCL=ON`), inside the fork's Docker image:

```
environment: GGML_CUDA_P2P=1  GGML_CUDA_GRAPHS_PRE_VOLTA=3  NCCL_P2P_LEVEL=SYS
             LLAMA_SPEC_SAMPLE_TEMP=1.0  LLAMA_SPEC_DRAFT_TOPK=20
llama-server -m Qwen3.8-27B-UD-Q6_K_XL.gguf --jinja --cache-ram 0 --no-cache-idle-slots
    --parallel 1 -c 262144 -ngl 99 -sm tensor -ts 1/1/1/1 -fa 1 -ctk q4_0 -ctv q4_0
    -b 2048 -ub 2048 -lm none -fit off
    --spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.0
    -ngld 99 -ubd 64 -ctkd q4_0 -ctvd q4_0
    --temp 1.0 --top-k 20 --top-p 0.95 --min-p 0.0 --metrics
```

Why each setting differs from the fork's reference command (measured on this box; the
numbers and method are in the [3-GPU measurements](3gpu-optimization/),
submitted to the fork as [PR #2](https://github.com/Kmic-68/llama.cpp/pull/2)):

| Setting | Why |
|---|---|
| NCCL build with `NCCL_P2P_LEVEL=SYS` | Prompt processing 276 → 500 t/s at depth 0 and 211 → 316 t/s on a 129,000-token prompt; generation 29.8 → 31.2 t/s. Without `NCCL_P2P_LEVEL=SYS`, NCCL picks a host-memory transport on this board. Output is repeatable but not bit-identical to the non-NCCL build. |
| `-lm none` | This box has 16 GB of RAM for a 23.5 GiB model; with the default mmap loading, generation was 11% slower and noisy, and loading took about twice as long. |
| `-ts 1/1/1/1` (four cards, since 2026-10-06) | Against three cards on the same day: a 64,000-token prompt 389 → 514 t/s, generation with MTP 37.4 → 42.1 t/s, and 8.5 GB per card instead of about 11. Output differs from three cards by about as much as turning NCCL on does, and two four-card runs were byte-identical. See [FOUR-CARDS.md](3gpu-optimization/FOUR-CARDS.md). |
| `-fit off` | Keeps the explicit `-ts` layout; used in every measurement. |
| MTP `n-max 3`, `p-min 0.0` | Fastest setting under both greedy and sampled decoding on this model (the fork's reference is 4 and 0.2). |
| `-b 2048` (was 32768) | The server acts on a dropped request only between decode calls, and `-b` sets how many prompt tokens go into one call. A request dropped mid-way through a 64,000-token prompt held the slot for 72-74 s at `-b 32768` and about 4 s at `-b 2048`, for 0.3% of prompt-processing speed. |
| `-c 262144` | Fits with about 5 GB free per card (11.1 GB peak with a 129,000-token prompt). |
| 150 W cap | See "GPU power limit" above. |

Two things to know when operating it:

- **Do not poll `/slots` quickly.** With another client requesting `/slots` five times a
  second, the server did not notice a dropped request until its prompt had finished
  processing (about 80 s), at any `-b`. `/health` every 5 s had no such effect.
- **One slot.** `-np 2` works with this stack but a second client still waits for a long
  prompt to finish, and it costs 0.7-1.2 GB more per card.

- **`--metrics` is on.** Scraping `/metrics` faster than about once a second may have the
  same effect as fast `/slots` polling; that has not been tested.

- **Comes back after a reboot.** The service has `restart: unless-stopped`, so Docker starts
  it again at boot if it was running when the machine went down. If it was stopped by hand
  (`docker compose stop`), it stays stopped. Set on 2026-10-06 after a reboot left production
  down; the auto-start itself has not been tested with a reboot.

On these settings a 64,000-token prompt processes at about 514 t/s and replies generate at
42-56 t/s with MTP. Peak draw of the four cards was about 650 W, roughly 710 W with the CPU,
on the 1000 W supply.

## Running llama-server via systemd (superseded by Docker Compose, kept for reference)

This section describes how production launches worked before the move to Docker Compose. It's
kept because the SELinux/RPATH gotchas below are still real properties of this build and this
box, even though nothing currently launches this way — useful if a bare systemd unit is ever
needed again for some reason.

The patched llama.cpp build (see
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md)) used to run as
a plain systemd service — no external process manager. An earlier setup
used `gppm` for this, mainly for its instance-supervision feature after
its actual power-management purpose turned out not to work on this
hardware (see [BENCHMARKING.md](BENCHMARKING.md)); it was removed in
favor of systemd's own supervision, which did the same job with one
fewer moving part.

`llama-server-moe.service`:

```ini
[Unit]
Description=llama-server - Qwen3.6-35B-A3B production (patched P100 build, MTP speculative decoding, tuned)
After=network.target

[Service]
Type=simple
User=<service-user>
Group=<service-user>
ExecStart=/opt/llama.cpp-p100/build/bin/llama-server --host 0.0.0.0 --port 8080 -m /path/to/models/Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL.gguf -ngl 99 -ts 1/1/1 -fa on -sm layer --ctx-size 65536 --cache-type-k q8_0 --cache-type-v q8_0 --parallel 1 --jinja --cache-ram 0 --no-cache-idle-slots --spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.75
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`--spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.75`
enables MTP (multi-token prediction) speculative decoding, tuned — see
[BENCHMARKING.md](BENCHMARKING.md) for what it does, the measured gain,
and the tuning sweep that found `--spec-draft-p-min 0.75` (not the
default of `0.0`) was the setting actually worth changing. It needs an MTP-converted GGUF,
not the regular quant: the model file is
`Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL.gguf` (unsloth's
`Qwen3.6-35B-A3B-MTP-GGUF` repo), not the plain
`Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` used earlier — same quant level,
~500MB larger for the added draft head, otherwise the same model
weights.

`--cache-ram 0 --no-cache-idle-slots` disables llama-server's cross-request
host-RAM prompt cache — carried over from another llama-server
deployment's production config after its bake-off found the opposite (cache left on)
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
sudo chown -R <service-user>:<service-user> /opt/llama.cpp-p100
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

**Docker Compose sidesteps this whole problem.** A container isn't subject to the `init_t`/
`user_home_t` SELinux interaction that forced the binary under `/opt` in the first place, so the
Compose setup just bind-mounts the build wherever it happens to live — no `/opt` copy step, no
SELinux relabeling. It has its own RPATH gotcha instead (an absolute, not relative, RPATH baked
into the binary at link time), documented in the Compose setup's own README, not here.
