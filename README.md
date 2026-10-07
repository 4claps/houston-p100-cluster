# houston-p100-cluster

Documentation for `houston`, an NVIDIA Tesla P100 16GB local
LLM inference server. It's a repurposed old NAS build, given a second life
as a dedicated inference box once the hardware had 48GB of usable VRAM
crammed into it. It was built with three P100s; a fourth was added on
2026-10-06, for 64GB.

## Production today (2026-10-06)

Models are served from **Docker**, not from a bare-metal `llama-server` systemd service
(that unit is masked). One Docker Compose service runs
[Kmic-68's P100 fork](https://github.com/Kmic-68/llama.cpp) of llama.cpp, built with
NCCL, across all four cards with tensor split.

| | |
|---|---|
| Model | Qwen3.8-27B, unsloth `UD-Q6_K_XL` (23.5 GiB) |
| Build | Kmic-68 fork at `ae35056eb`, CUDA 12.9.1, NCCL 2.27.3, in a Docker image |
| Split | `-sm tensor -ts 1/1/1/1` (four cards), all layers on GPU, q4_0 KV cache |
| Context | `-c 262144`, one slot |
| Batch | `-b 2048 -ub 2048` |
| Loading | `-lm none -fit off` |
| Speculative decoding | MTP, `--spec-draft-n-max 3 --spec-draft-p-min 0.0` |
| Environment | `GGML_CUDA_P2P=1`, `GGML_CUDA_GRAPHS_PRE_VOLTA=3`, `NCCL_P2P_LEVEL=SYS` |
| Power limit | 150 W per card |

Speed on that configuration:

| Measurement | Result |
|---|---|
| Prompt processing, 64,000-token prompt through the server | 514 tokens/s (389 on three cards) |
| Generation with MTP, 512 tokens through the server | 42.1 tokens/s (37.4 on three cards) |
| Prompt processing, `llama-bench` pp2048 | 629 tokens/s (505 on three cards) |
| Generation without MTP, `llama-bench` tg512 | 39.2 tokens/s (33.1 on three cards) |
| Agent battery (9 tasks x 3, on this exact configuration) | 26 of 27 passed; 120 s mean task time; 454 tokens/s prompt processing and 53.9 tokens/s generation over the run |
| Peak memory | 8.5 GB of 16 GB per card |

All of these were measured on 2026-10-06 at the 150 W limit; the three-card figures in
brackets are from the same session. Details are in
[3gpu-optimization/FOUR-CARDS.md](3gpu-optimization/FOUR-CARDS.md).

The full command and the measured reason for each setting are in
[SOFTWARE.md](SOFTWARE.md#what-production-runs-now-2026-10-06). How these settings were
arrived at is in [3gpu-optimization/](3gpu-optimization/).

Most of the other documents here were written before this change. Where they say
"production" they mean the earlier setup: Qwen3.6-35B-A3B on a patched llama.cpp build
running directly on the host, with layer split and a 125 W limit. Each carries a dated
note saying so.

This repo is documentation only — no Ansible, no provisioning scripts. It
exists so that the non-obvious parts of this build (particularly a BIOS
modification that isn't something you'd want to reverse-engineer twice)
are written down somewhere durable.

## Contents

- [HARDWARE.md](HARDWARE.md) — full parts list and the reasoning behind
  each choice
- [BIOS-FIX.md](BIOS-FIX.md) — the Above 4G Decoding fix: an X79-era MMIOH
  bug that hid the setting entirely, and the process used to patch and
  expose it
- [SOFTWARE.md](SOFTWARE.md) — OS, driver, fan control, the persistent
  150 W GPU power limit, and the settings production runs now
- [LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) — building
  llama.cpp for Pascal and the P100 performance patches
- [BENCHMARKING.md](BENCHMARKING.md) — baseline-vs-patched throughput
  results, MTP speculative-decoding tuning, GPU utilization/power
  findings, and the current 125 W-cap agent-battery baseline
- [GPU-SCALING.md](GPU-SCALING.md) — how performance scales as the
  P100s are removed one at a time (3 → 2 → 1 cards), measured with a
  real agent workload rather than a synthetic benchmark, plus PCIe lane-
  width tests and power draw. Headline results: for single-stream use,
  one, two and three cards all generate at roughly the same speed (about
  76-85 tokens/s on the production model), so extra cards buy VRAM, not
  speed; and PCIe lane width didn't matter
- [MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md) — one-off
  side tests that grew out of the GPU scaling benchmarks: the production
  Qwen3.6 quant on two and three cards, a dense Qwen3.8-27B model, a P2P
  and tensor-split check, and NVIDIA's Nemotron-3.5-Lightning model
- [KMIC68-FORK-AGENT-BATTERY.md](KMIC68-FORK-AGENT-BATTERY.md) — an
  independent P100/Pascal fork ([Kmic-68/llama.cpp](https://github.com/Kmic-68/llama.cpp))
  evaluated in an isolated Docker environment, gated for correctness, then run
  against the real Hermes agent battery at a 125 W power cap
- [3gpu-optimization/](3gpu-optimization/) — the 3-GPU optimization work on the Kmic-68
  fork, all in one folder: a summary ([README](3gpu-optimization/README.md)), every result
  ([RESULTS](3gpu-optimization/RESULTS.md)), the method, the limitations, the plan
  ([PLAN](3gpu-optimization/PLAN.md)), derived data and scripts. The same measurements were
  submitted to the fork as [PR #2](https://github.com/Kmic-68/llama.cpp/pull/2). They cover CUDA graphs, model loading, NCCL,
  the power cap, context fit, long-context output quality, MTP settings, an agent battery,
  concurrent clients and what happens when a client drops mid-prompt; the production
  settings in `SOFTWARE.md` come from them

- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — known issues and workarounds,
  including the `-sm tensor` crash and a GPU falling off the bus (Xid 79)
- [CHANGELOG.md](CHANGELOG.md) — dated list of notable result and doc changes

## Ongoing benchmarking

Benchmarking on this box is an ongoing project, and more will follow.
The GPU scaling and miscellaneous benchmarks are living documents that
will keep growing as further tests are run. Note that the "3x P100"
description above is the build as originally assembled; the GPU-scaling
results in `GPU-SCALING.md` are about what changes when cards are
removed.
