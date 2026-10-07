# Changelog

Notable changes to the benchmark results and docs in this repo, newest
first. Dates are when the change merged.

## 2026-10-06

- **Added** a fourth Tesla P100 and moved production to four cards (`-ts 1/1/1/1`). Measured
  on the production configuration against three cards the same day: a 64,000-token prompt
  389 → 514 t/s (+32%), generation with MTP 37.4 → 42.1 t/s (+13%), 8.5 GB per card instead
  of about 11. The agent battery on the four-card service passed 26 of 27 tasks at 120 s
  mean task time. Full results in
  [3gpu-optimization/FOUR-CARDS.md](3gpu-optimization/FOUR-CARDS.md).
- **Updated** "Production today" in [README.md](README.md), "What production runs now" and
  the fan-control note in [SOFTWARE.md](SOFTWARE.md), and the GPU entry in
  [HARDWARE.md](HARDWARE.md) for four cards. The fans are back on the temperature-driven
  service; `--metrics` is now on the server command.

## 2026-10-05

- **Changed** the GPU power limit from 125 W to 150 W per card
  (`nvidia-power-limit.service` now runs `nvidia-smi -pl 150`). A sweep on the current
  production stack showed the 125 W cap was limiting throughput: 150 W gives +6.5%
  generation and +4.8% prompt processing, and 175 W adds little more to generation. Table
  and reasoning in the "GPU power limit" section of [SOFTWARE.md](SOFTWARE.md);
  [HARDWARE.md](HARDWARE.md) updated to match.
- **Added** "What production runs now" to [SOFTWARE.md](SOFTWARE.md): Qwen3.8-27B
  `UD-Q6_K_XL` on the Kmic-68 fork with an NCCL build (`NCCL_P2P_LEVEL=SYS`), `-lm none`,
  `-fit off`, `-c 262144`, MTP `n-max 3` / `p-min 0.0` and `-b 2048`, with the measured
  reason for each setting and two operating notes (do not poll `/slots` quickly; one slot).
- **Added** [3gpu-optimization/](3gpu-optimization/): the 3-GPU optimization work on the
  Kmic-68 fork in one folder (summary, results, method, limitations, plan, derived data and
  scripts). The measurements were also submitted to the fork as
  [PR #2](https://github.com/Kmic-68/llama.cpp/pull/2).
- **Added** a "Production today" section to [README.md](README.md): models are now served
  from Docker, not a bare-metal systemd service, with the current configuration and speeds.
  [BENCHMARKING.md](BENCHMARKING.md), [GPU-SCALING.md](GPU-SCALING.md),
  [MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md) and
  [LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) each carry a note that
  "production" in them means the earlier bare-metal setup.
- **Noted** in [SOFTWARE.md](SOFTWARE.md) that the fans are currently held at full speed by
  `fans-full.service`, with `gpu-fan-control.service` enabled but not running, and that
  `llama-server.service` is now masked ([GPU-SCALING.md](GPU-SCALING.md) updated to match).
- **Noted** at the top of [KMIC68-FORK-AGENT-BATTERY.md](KMIC68-FORK-AGENT-BATTERY.md) that
  it describes the fork as first evaluated (125 W, no NCCL, MTP 4 / 0.2, `-b 32768`), and in
  the baseline section of [BENCHMARKING.md](BENCHMARKING.md) that the 125 W baseline is the
  earlier model and build. Their numbers are unchanged.

## 2026-09-29

- **Changed** the GPU power limit from a manual setting that was lost on every reboot
  to a persistent one: all three Tesla P100s are now capped at 125 W (down from the
  250 W default) by a new `nvidia-power-limit.service` systemd unit that runs at boot,
  and `nvidia-persistenced.service` is enabled so persistence mode is on for every
  card. The unit contents, enable steps and what to change if a different card model
  is added are in the new "GPU power limit" section of [SOFTWARE.md](SOFTWARE.md).
- **Changed** the hardware layout and documented the riser failure that caused it.
  One P100 on a riser at `02:00.0` repeatedly fell off the bus (Xid 79, then Xid 154)
  about 7 minutes after boot, once with a kernel panic in the `nvidia` module. The riser
  was removed and the card moved to a direct slot, and it has been stable under load
  since. The GT 610 test card was also removed. The three cards now sit at `01:00.0`,
  `03:00.0` and `05:00.0`, each at PCIe Gen 3 x8. See the new Xid 79 section in
  [TROUBLESHOOTING.md](TROUBLESHOOTING.md), the updated PCIe section of
  [HARDWARE.md](HARDWARE.md), and the note on `pcie_aspm=off` in
  [SOFTWARE.md](SOFTWARE.md).
- **Noted** that the benchmark results in [BENCHMARKING.md](BENCHMARKING.md),
  [GPU-SCALING.md](GPU-SCALING.md) and [MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md)
  were measured on the earlier layout (x16/x8/x8, with a card at `02:00.0`) and, for
  most of them, before the 125 W limit was persistent. Their numbers and PCI addresses are
  unchanged.

## 2026-09-27

- **Updated** `SOFTWARE.md`, `BENCHMARKING.md`, and `GPU-SCALING.md` to reflect that
  production models now launch via Docker Compose, not `llama-server.service` directly.
  The old systemd how-to in `SOFTWARE.md` is kept as reference (its SELinux/RPATH
  findings are still real), relabeled as superseded. The Compose setup itself, and its
  own RUNPATH gotcha, live outside this repo alongside the model files — see the note in
  `SOFTWARE.md` for where. Flagged, not fixed: `llama-server.service` is still `enabled`
  at the systemd level, so a reboot would let it auto-start and contend with Compose for
  the GPUs.

- **Added** [KMIC68-FORK-AGENT-BATTERY.md](KMIC68-FORK-AGENT-BATTERY.md): an
  independent P100/Pascal `llama.cpp` fork ([Kmic-68/llama.cpp](https://github.com/Kmic-68/llama.cpp))
  built and gated in an isolated Docker environment (production untouched), then run
  against the real Hermes agent battery at the documented 125 W power cap: 96% ok%
  (27 task-reps), 188.0 t/s avg prompt processing, 45.1 t/s avg generation on the dense
  Qwen3.8-27B model with `-sm tensor` + MTP speculative decoding. Every server flag is cited
  directly to the fork's own `CHANGES.md`/`QUICKSTART.md` rather than borrowed from this repo's
  unrelated 29-patches-build tuning. A side-by-side against this repo's existing Qwen3.8-27B
  benchmark is included, with its confounds (MTP vs. none, context size, power cap) stated
  plainly rather than treated as a controlled comparison.

## 2026-09-26

- **Added** per-card temperature, power and fan detail (including time above
  60/65/70°C) for the 125 W baseline and the 250 W run to
  [BENCHMARKING.md](BENCHMARKING.md). The middle-slot card is the only one that
  approaches throttling temperatures: 61°C average and 65°C peak under the
  cap versus 69°C and 75°C uncapped.

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
