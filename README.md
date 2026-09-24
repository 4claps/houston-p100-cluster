# houston-p100-cluster

Documentation for `houston`, a 3x NVIDIA Tesla P100 16GB local
LLM inference server. It's a repurposed old NAS build, given a second life
as a dedicated inference box once the hardware had 48GB of usable VRAM
crammed into it.

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
- [SOFTWARE.md](SOFTWARE.md) — OS, driver, and fan control
- [LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) — building
  llama.cpp for Pascal and the P100 performance patches
- [BENCHMARKING.md](BENCHMARKING.md) — baseline-vs-patched throughput
  results, MTP speculative-decoding tuning, and GPU utilization/power
  findings
- [GPU-SCALING.md](GPU-SCALING.md) — how performance scales as the
  P100s are removed one at a time (3 → 2 → 1 cards), measured with a
  real agent workload rather than a synthetic benchmark, plus PCIe lane-
  width tests and power draw. Headline result: for single-stream use,
  two cards were about 4x faster than three
- [MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md) — one-off
  side tests that grew out of the GPU scaling benchmarks, such as running the
  production Qwen3.6 quant on two cards and a dense Qwen3.8-27B model

## Ongoing benchmarking

Benchmarking on this box is an ongoing project, and more will follow.
The GPU scaling and miscellaneous benchmarks are living documents that
will keep growing as further tests are run. Note that the "3x P100"
description above is the build as originally assembled; the GPU-scaling
results in `GPU-SCALING.md` are about what changes when cards are
removed.
