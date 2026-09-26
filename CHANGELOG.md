# Changelog

Notable changes to the benchmark results and docs in this repo, newest
first. Dates are when the change merged.

## 2026-09-25

- **Added** a **new baseline** to [BENCHMARKING.md](BENCHMARKING.md): the
  agent battery on Qwen3.6-35B-A3B Q4_K_XL, 3 cards, with every card
  power-capped at 125 W. Versus the default 250 W it is about 3.5% slower
  in generation, 12% slower in prompt processing, and uses 17% less power
  and 6% less energy per task, with the hottest card ~10°C cooler. The
  driver's throttle reason was "SW Power Cap" (~29% of busy samples), with
  no thermal throttling.

## 2026-09-24

- **Changed** the results in [BENCHMARKING.md](BENCHMARKING.md): re-measured
  the patched-vs-baseline, concurrency and MTP results at verified GPU
  clocks and removed the stale numbers (PR #11).
- **Added** the Nemotron-3.5-Lightning results, a 3-card Qwen3.8-27B run, and
  a P2P / tensor-split check to
  [MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md) (PRs #10, #11).
- **Fixed** the 3-GPU results: an early run had its GPU clocks stuck at the
  idle clock and was discarded and rerun (PR #9).

## 2026-09-23

- **Added** [GPU-SCALING.md](GPU-SCALING.md) and
  [MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md), and the PCIe
  lane / USB 3 behavior notes in [HARDWARE.md](HARDWARE.md) (PR #6).

## 2026-09-22

- **Changed** production to MTP speculative decoding and added the MTP
  tuning sweep (PR #5).
- **Added** [BENCHMARKING.md](BENCHMARKING.md), split out of the
  llama.cpp doc, and documented why gppm was removed (PR #4).

## 2026-09-21

- **Added** the Qwen3.8-27B dense-model benchmark (PR #3) and the first
  llama.cpp P100 build and patch documentation (PR #1).
