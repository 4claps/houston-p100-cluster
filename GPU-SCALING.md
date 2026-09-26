# GPU scaling benchmarks

Real-hardware, real-workload benchmarking of this box as its P100s are
physically removed one at a time: the actual 9-task x 3-rep Hermes agent
battery, run through a bubblewrap-sandboxed harness against a real
`llama-server` over HTTP — not a synthetic microbenchmark. The question
is how performance scales going from 3 -> 2 -> 1 cards. Quant is stepped
down at each stage to keep the production model (MTP-tuned) fitting in
shrinking VRAM: Q4_K_XL (3 GPU) -> Q3_K_XL (2 GPU) -> Q2_K_XL (1 GPU).

## Status: an ongoing project

**This is an ongoing project, and more will follow.** Every leg planned
at the start is done — 3-GPU, 2-GPU in three PCIe lane configurations
(x16/x16, x16/x8, x8/x8), and 1-GPU — and side tests that grew out of
the results live in [MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md).
Further runs and write-ups will be added to this file and that one over
time. Read everything below as "what the data shows so far", not a final
word.

What the data shows so far:

- **Card count barely matters for single-stream throughput.** Generation
  averages 85.2 t/s on three cards, 85.0 on two, and 76.0 on one (the
  one-card run uses a smaller quant so the model fits).
- **PCIe link width doesn't matter for this workload.** Throughput is
  statistically identical across the three 2-GPU lane variants.
- **Quant doesn't matter for speed either.** Q4_K_XL and Q3_K_XL run at
  the same speed on two cards (85.0 vs. 82.0 t/s).
- **Extra cards cost power, not speed:** roughly 200 W, 270 W and 327 W
  (GPUs plus CPU) with one, two and three cards, for about the same
  single-stream speed. What a second or third card buys is VRAM.

Questions the data so far leaves open:

- How the different card counts behave under concurrent load. All the
  runs here are single-stream; [BENCHMARKING.md](BENCHMARKING.md) has a
  3-card concurrency sweep.
- How much of the one-card vs. multi-card gap is the card count vs. the
  Q2_K_XL vs. Q4_K_XL quant, which this series can't separate.
- How other models behave on the same hardware.

## A note on an early invalid run

An early 3-GPU run was discarded and rerun: its GPU core clocks turned
out to be stuck at the idle clock (405 MHz) for the whole run, so it
measured a card in a low-power state, not the hardware. Every result in
this doc was checked to have run at the normal ~1,328 MHz under load.
The cause of the stuck clocks is unknown and it hasn't recurred. The
lesson is in the method notes below: check GPU clocks, and rerun a
surprising result before explaining it.

## 3-GPU leg (Q4_K_XL)

Model: `Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL.gguf`, MTP speculative decoding
(`--spec-draft-n-max 3 --spec-draft-p-min 0.75`), `-ts 1/1/1`, layer
split, `--parallel 1 --ctx-size 65536`, q8_0 KV cache. This is the exact
production command line, launched manually with the production service
stopped. Cards at x16/x8/x8. Standard harness settings (600s task cap,
8192 max tokens).

**Hermes agent-task results (27 task-reps):**

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 64s |
| err_replay_patch | 100% | 57s |
| err_ambiguous_edit | 100% | 89s |
| err_case_search | 100% | 70s |
| err_hidden_search | 33% | 93s |
| err_big_output | 100% | 47s |
| err_multi_dir | 100% | 68s |
| err_inline_script | 100% | 88s |
| err_big_file_read | 100% | 59s |
| **TOTAL** | **93%** | **70s** |

**Throughput** (113 real requests, from `llama-server`'s own per-request timings):

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 52.2 | 622.1 | 325.8 |
| Generation (t/s) | 68.8 | 98.1 | 85.2 |

MTP draft acceptance averaged 96.0%. The `llama-bench` sanity check
(layer split, no MTP) gave pp512 459.7 t/s and tg128 70.6 t/s.

**System telemetry** (925 samples, ~34min, 2s interval):

| Metric | min | max | avg |
|---|---:|---:|---:|
| CPU power (W) | 23.4 | 61.3 | 52.6 |
| CPU package temp (°C) | 45.0 | 64.0 | 59.1 |
| CPU avg clock (MHz) | 3292.0 | 3751.4 | 3623.0 |
| GPU0 power (W) | 31.3 | 190.5 | 94.1 |
| GPU0 temp (°C) | 39.0 | 57.0 | 52.7 |
| GPU0 utilization (%) | 0 | 100 | 44.1 |
| GPU1 power (W) | 33.7 | 184.7 | 85.1 |
| GPU1 temp (°C) | 46.0 | 61.0 | 56.0 |
| GPU1 utilization (%) | 0 | 100 | 36.5 |
| GPU2 power (W) | 37.3 | 199.4 | 94.7 |
| GPU2 temp (°C) | 52.0 | 75.0 | 69.4 |
| GPU2 utilization (%) | 0 | 100 | 36.9 |
| Shroud fan speed (RPM) | 1724 | 3054 | 2687.7 |

GPU core clocks averaged 1,327 MHz (min 1,189, max 1,328). Each card is
busy well under half the time (37–44% utilization), since layer split
runs the cards one after another. The warmest card is the one in the
middle slot (`02:00.0`, x8), at 69°C average and 75°C peak. (The
telemetry columns GPU0-GPU2 are ordered by card UUID, not slot: GPU0 is
`03:00.0`, GPU1 is `01:00.0` and GPU2 is `02:00.0`.)

## 2-GPU leg (Q3_K_XL, x16/x16)

Model: `Qwen3.6-35B-A3B-MTP-UD-Q3_K_XL.gguf`, same MTP flags
(`--spec-draft-n-max 3 --spec-draft-p-min 0.75`), `-ts 1/1`, `--parallel
1 --ctx-size 65536`. Manual `llama-server` on port 8080 (production
service stopped for the duration of this leg).

### Hardware bring-up notes: PCIe lanes and USB 3

This leg was run right after removing the third P100. Getting the two
remaining cards to full x16/x16 took some detective work, recorded here
because the board has non-obvious behavior.

On this board the PCIe link widths depend on which slots are populated:
**x16/x8/x8** across three P100s, and **x16/x16** with only two P100s in
the dedicated x16 slots. Getting to x16/x16 was harder than it sounds,
and the cause wasn't the cards:

- The box is headless with no integrated graphics, and its network
  connection is a **USB Wi-Fi dongle that was plugged into a USB 3
  port**.
- With just the two P100s in the x16 slots, the machine appeared not
  to POST. With no integrated graphics there was no video output to
  show why.
- Adding a GT 610 as a display card made it boot normally, but at an
  x16/x8/x8 lane split, which left the second P100 at **x8** — downgraded
  from its native x16, per `lspci`/`nvidia-smi`. (The GT 610 only went in
  at this point — it can't be fitted alongside three P100s, since the
  P100s occupy all three double-wide slots.)
- Moving the GT 610 into the second x16 slot made boot watchable on a
  monitor. Once logged in at the console the links showed x16/x16 —
  but the box had **no IP address**.
- The reason: **x16/x16 uses up all of the board's PCIe lanes, which
  disables its USB 3 ports** — including the one the dongle was in.
  The machine had been booting fine all along; it just had no video
  and no network.
- The fix: move the dongle to a **USB 2 port** and put the P100 in the
  second x16 slot. Everything then worked as expected, at x16/x16.

Whether lane width affects throughput at all is tested in the lane
variants below. It doesn't.

**Hermes agent-task results (27 task-reps):**

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 64s |
| err_replay_patch | 100% | 86s |
| err_ambiguous_edit | 100% | 69s |
| err_case_search | 100% | 71s |
| err_hidden_search | 0% | 62s |
| err_big_output | 100% | 53s |
| err_multi_dir | 100% | 69s |
| err_inline_script | 100% | 75s |
| err_big_file_read | 100% | 101s |
| **TOTAL** | **89%** | **72s** |

`err_hidden_search` is the unstable task across the whole series — it
ranges from 0% to 100% depending on the run, at only 3 reps per cell — so
no quant effect can be read into this result.

**Throughput** (124 real requests, from `llama-server`'s own per-request timings):

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 63.8 | 517.5 | 283.9 |
| Generation (t/s) | 65.4 | 97.0 | 82.0 |

**System telemetry** (1,006 samples, ~33min, 2s interval):

| Metric | min | max | avg |
|---|---:|---:|---:|
| CPU power (W) | 21.4 | 58.7 | 50.6 |
| CPU package temp (°C) | 46.0 | 64.0 | 58.9 |
| CPU avg clock (MHz) | 3424.4 | 3756.5 | 3638.7 |
| GPU0 power (W) | 33.7 | 190.1 | 102.9 |
| GPU0 temp (°C) | 46.0 | 67.0 | 61.7 |
| GPU0 utilization (%) | 0 | 100 | 50.5 |
| GPU1 power (W) | 35.1 | 193.7 | 113.8 |
| GPU1 temp (°C) | 45.0 | 68.0 | 63.3 |
| GPU1 utilization (%) | 0 | 100 | 55.5 |
| Shroud fan speed (RPM) | 1448 | 2626 | 2358.4 |

### Also run with the production quant (Q4_K_XL)

This leg used Q3_K_XL because the quant was being stepped down alongside
the card count. Q4_K_XL, the production quant, also fits on two cards,
so it was run later as a side test — the full write-up is in
[MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md#qwen36-35b-a3b-moe-q4_k_xl-2x-p100).
**The results were around the same as this Q3_K_XL leg:**

| Two cards, x16/x16 | Gen avg (t/s) | PP avg (t/s) | Avg task wall | Ok% |
|---|---:|---:|---:|---:|
| Q3_K_XL (this leg) | 82.0 | 283.9 | 72s | 89% |
| Q4_K_XL (side test) | 85.0 | 302.8 | 83s | 93% |

Q4_K_XL is as fast as Q3_K_XL on two cards, so the quant step-downs in
this series were only ever needed to make the model fit in VRAM, not
because quant affects speed. (The 3-GPU rerun then matched this
Q4_K_XL result: 85.2 vs. 85.0 t/s generation.)

## 2-GPU lane-configuration variants (x16/x8, x8/x8)

The two remaining P100s were physically moved into different PCIe slot
combinations to test whether link width affects throughput. Same Q3_K_XL model and flags as the
x16/x16 leg above; only the physical slot layout changes between runs.

### x16/x8

GPU0 (`01:00.0`) at x16, GPU1 (`03:00.0`) at x8 — same physical cards,
same Q3_K_XL model/flags as the x16/x16 leg, just GPU1's link
deliberately left downgraded instead of fixed.

**Result: essentially no difference from x16/x16.**

**Hermes agent-task results (27 task-reps):**

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 70s |
| err_replay_patch | 100% | 94s |
| err_ambiguous_edit | 100% | 116s |
| err_case_search | 100% | 89s |
| err_hidden_search | 0% | 67s |
| err_big_output | 100% | 61s |
| err_multi_dir | 100% | 69s |
| err_inline_script | 100% | 131s |
| err_big_file_read | 100% | 94s |
| **TOTAL** | **89%** | **88s** |

Same 89% ok% and the same `err_hidden_search` 0/3 as x16/x16.

**Throughput** (121 real requests):

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 26.8 | 517.0 | 295.1 |
| Generation (t/s) | 59.5 | 96.5 | 83.2 |

**System telemetry** (1,194 samples, ~40min, 2s interval):

| Metric | min | max | avg |
|---|---:|---:|---:|
| CPU power (W) | 21.4 | 60.3 | 51.8 |
| CPU package temp (°C) | 48.0 | 64.0 | 58.8 |
| CPU avg clock (MHz) | 3492.0 | 3767.1 | 3638.1 |
| GPU0 power (W) | 33.5 | 187.2 | 98.9 |
| GPU0 temp (°C) | 46.0 | 66.0 | 60.7 |
| GPU0 utilization (%) | 0 | 100 | 49.1 |
| GPU1 power (W) | 34.2 | 190.7 | 115.9 |
| GPU1 temp (°C) | 45.0 | 67.0 | 63.5 |
| GPU1 utilization (%) | 0 | 100 | 57.4 |
| Shroud fan speed (RPM) | 1378 | 2641 | 2367.8 |

Both throughput (avg PP 295.1 vs. 283.9, avg gen 83.2 vs. 82.0) and
telemetry (GPU power/utilization within a couple percent either way) are
statistically indistinguishable from the x16/x16 leg. **Halving GPU1's
link width from x16 to x8 had no measurable effect** on this single-
stream, layer-split workload.

So link width isn't a factor here: x8 still has plenty of bandwidth for the
small per-token activations that cross cards in layer-split mode.

### x8/x8

Both P100s downgraded to x8 (bus addresses shifted again to `02:00.0`
and `03:00.0` after the physical move — re-verified before running).
Same Q3_K_XL model/flags as the other two 2-GPU legs.

**Result: still no meaningful difference from x16/x16 or x16/x8.** Link
width doesn't matter for this workload.

**Caveat: cooling changed for this leg too, separately from the PCIe
variable.** GPU1 ran noticeably hotter than in the other two 2-GPU legs
— avg 74.8°C / max 80°C here vs. avg ~63°C / max ~67-68°C in the x16/x16
and x16/x8 legs — more than the ~8W higher avg power draw would explain
on its own, and the case fan (`fan3`) spun up accordingly (avg 2908.7
RPM here vs. ~2360 in the other two legs). Likely an airflow-path change
from physically reseating the cards, not a hardware fault — nowhere
near the P100's thermal throttle point, and it doesn't appear to have
affected throughput (see below). Flagged here since it's a second
variable (alongside link width) that changed for this specific leg.

**Hermes agent-task results (27 task-reps):**

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 63s |
| err_replay_patch | 100% | 91s |
| err_ambiguous_edit | 100% | 81s |
| err_case_search | 100% | 94s |
| err_hidden_search | 0% | 59s |
| err_big_output | 100% | 72s |
| err_multi_dir | 100% | 94s |
| err_inline_script | 100% | 170s |
| err_big_file_read | 67% | 147s |
| **TOTAL** | **85%** | **97s** |

One new failure: `err_big_file_read-r1` (exit 1, 160s). Checked the
actual transcript — not a hardware/thermal issue. The agent searched a
6000-line file inefficiently (re-reading overlapping chunks without
tracking what it had already covered), ran out of context before
finding the target line, and this eval config runs with
`compression.enabled: false` (no auto-compaction), so it hit a hard
context overflow rather than recovering. A model-behavior/search-
strategy failure, not an infrastructure one — same category of thing
that would happen on any hardware given this exact conversation path.

**Throughput** (137 real requests):

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 26.0 | 516.2 | 291.4 |
| Generation (t/s) | 55.8 | 97.5 | 80.8 |

**System telemetry** (1,326 samples, ~45min, 2s interval):

| Metric | min | max | avg |
|---|---:|---:|---:|
| CPU power (W) | 21.2 | 59.9 | 51.0 |
| CPU package temp (°C) | 44.0 | 62.0 | 56.6 |
| CPU avg clock (MHz) | 3437.7 | 3761.3 | 3638.9 |
| GPU0 power (W) | 32.3 | 190.9 | 97.9 |
| GPU0 temp (°C) | 45.0 | 63.0 | 56.8 |
| GPU0 utilization (%) | 0 | 100 | 47.6 |
| GPU1 power (W) | 34.6 | 209.6 | 124.6 |
| GPU1 temp (°C) | 46.0 | 80.0 | 74.8 |
| GPU1 utilization (%) | 0 | 100 | 55.8 |
| Shroud fan speed (RPM) | 1398 | 3047 | 2908.7 |

**Conclusion across all three 2-GPU lane variants**: prompt processing
avg (283.9 / 295.1 / 291.4 t/s) and generation avg (82.0 / 83.2 / 80.8
t/s) for x16/x16, x16/x8, and x8/x8 respectively are all within noise of
each other. **PCIe link width, from x16/x16 down to x8/x8, has no
measurable effect on this single-stream, layer-split workload on this
box.**

## 1-GPU leg (Q2_K_XL, one card at x16)

Model: `Qwen3.6-35B-A3B-MTP-UD-Q2_K_XL.gguf` (11.70 GiB), same MTP flags
(`--spec-draft-n-max 3 --spec-draft-p-min 0.75`), no `-ts` (single
device), `--parallel 1 --ctx-size 65536`, q8_0 KV cache. Fits with room
to spare: 13.3 of 16.4 GB used with the full 65K context. Manual
`llama-server` on port 8080, production service stopped for the run.
Standard harness settings (600s task cap, 8192 max tokens).

A short `llama-bench` baseline (no MTP; used only as a sanity check, not
for the results below) gave pp512 472.0 t/s and tg128 69.95 t/s.

**Hermes agent-task results (27 task-reps):**

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 81s |
| err_replay_patch | 100% | 101s |
| err_ambiguous_edit | 100% | 110s |
| err_case_search | 100% | 106s |
| err_hidden_search | 0% | 72s |
| err_big_output | 100% | 82s |
| err_multi_dir | 100% | 106s |
| err_inline_script | 100% | 114s |
| err_big_file_read | 100% | 110s |
| **TOTAL** | **89%** | **98s** |

Every task exited cleanly; the only misses are all three
`err_hidden_search` reps, the same noisy task that failed 0-67% on every
other Qwen3.6 leg. Overall ok% (89%) equals the 2-GPU Q3_K_XL leg's.

**Throughput** (125 real requests, from `llama-server`'s own per-request timings):

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 64.9 | 362.5 | 247.7 |
| Generation (t/s) | 52.3 | 85.9 | 76.0 |

MTP draft acceptance averaged 94.6%.

**System telemetry** (1,380 samples, ~46min, 2s interval):

| Metric | min | max | avg |
|---|---:|---:|---:|
| CPU power (W) | 20.1 | 58.2 | 50.1 |
| CPU package temp (°C) | 44.0 | 60.0 | 55.9 |
| CPU avg clock (MHz) | 3459.7 | 3762.6 | 3637.3 |
| GPU0 power (W) | 33.5 | 203.9 | 149.9 |
| GPU0 temp (°C) | 46.0 | 76.0 | 71.4 |
| GPU0 utilization (%) | 0 | 100 | 87.5 |
| Shroud fan speed (RPM) | 1386 | 3033 | 2795.0 |

The single card runs far busier than any card in the multi-GPU legs
(avg utilization 87.5% vs. ~50-58% per card on two cards, avg power ~150W
vs. ~103-116W). With one card there's no second stage to wait on, so it
computes almost continuously. With more cards, each card is idle more of
the time (37–57% average utilization per card on two or three cards).

### Summary across all legs

All legs: same Hermes battery, same MTP flags, standard harness
settings, patched build, GPU clocks at ~1,328 MHz. The 2-GPU rows are the
x16/x16 configuration (the lane variants were within noise of it).

| Cards | Quant | Gen avg (t/s) | PP avg (t/s) | Avg task wall | Ok% |
|---:|---|---:|---:|---:|---:|
| 3 | Q4_K_XL | 85.2 | 325.8 | 70s | 93% |
| 2 | Q4_K_XL | 85.0 | 302.8 | 83s | 93% |
| 2 | Q3_K_XL | 82.0 | 283.9 | 72s | 89% |
| 1 | Q2_K_XL | 76.0 | 247.7 | 98s | 89% |

The 2-GPU Q4_K_XL row comes from
[MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md#qwen36-35b-a3b-moe-q4_k_xl-2x-p100),
run specifically to separate quant from card count.

- **Card count barely matters for single-stream speed.** Three cards and
  two cards give the same generation speed (85.2 vs. 85.0 t/s) and
  similar prompt processing (325.8 vs. 302.8 t/s). Layer split runs the
  cards one after another, so extra cards don't add speed at batch 1.
- **One card gets ~90% of the multi-card speed** (76.0 vs. 85.0 t/s
  generation, 247.7 vs. 302.8 t/s prompt processing). Part of that gap is
  the Q2_K_XL vs. Q4_K_XL quant rather than the card count (the Q4→Q3
  step alone cost ~3 t/s on two cards, so some of the 9 t/s is quant),
  and this leg can't separate the two. For single-stream use, extra
  cards buy little speed — their value is capacity: they let the box run
  higher quants like Q4_K_XL and bigger context/quant combinations that
  don't fit on one 16GB card.
- **Quality vs. speed**: ok% is 89-93% everywhere at 3 reps per cell,
  which is within the noise of the one unstable task
  (`err_hidden_search`); the series doesn't show a clear quality cost
  from dropping cards or quant levels for this workload.
- **Concurrency was not tested** here on any card count. The 3-card
  concurrency sweep in `BENCHMARKING.md` peaks at 8 parallel requests;
  under concurrent load, extra cards may pay off in ways they can't at
  batch 1.

## Power draw

Average power while a battery was running, from the 2-second telemetry
polling in each leg: the sum of every GPU's board power (as reported by
`nvidia-smi`) plus CPU package power (from RAPL).

| Leg | GPUs (W) | CPU package (W) | Total (W) | Peak sample (W) | Avg task wall | Energy per task |
|---|---:|---:|---:|---:|---:|---:|
| 3 cards, Q4_K_XL (rerun) | 274 | 53 | **327** | 601 | 70s | ~23 kJ |
| 2 cards, Q4_K_XL | 219 | 51 | **270** | 437 | 83s | ~22 kJ |
| 2 cards, Q3_K_XL (x16/x16) | 217 | 51 | **268** | 418 | 72s | ~19 kJ |
| 1 card, Q2_K_XL | 150 | 50 | **200** | 258 | 98s | ~20 kJ |

Energy per task is average total power times average task wall time. The
other two 2-GPU lane variants (x16/x8, x8/x8) draw 267 and 274 W — within
about 2% of the x16/x16 figure, consistent with link width not mattering.

**General numbers:** roughly **200 W** with one card, **270 W** with two,
and **330 W** with three, for GPUs plus CPU while the workload runs.

What these numbers do and don't cover:

- **Not whole-machine power.** Motherboard, RAM, fans, drives and
  power-supply conversion losses aren't measured. A rough guess is that
  draw at the wall runs somewhere around 50-100 W higher than the totals
  above — that is an estimate, not a measurement; only a plug-in power
  meter would give the real figure.
- **Extra cards cost power without adding single-stream speed:** about
  +70 W going from one card to two and +57 W going from two to three,
  for the same ~85 t/s. Energy per task rises only modestly (~20 → ~22 →
  ~23 kJ) because the tasks finish a little faster on the multi-card
  setups. Per-card draw falls as cards are added (~150 W on one card,
  ~100-125 W each on two, ~85-95 W each on three) because each card is
  busy less of the time.
- These are single-stream runs. Power under concurrent load wasn't
  measured.
- A 125 W per-card power cap (the P100's minimum) was also tested on the
  3-card Qwen3.6 run: about 17% less power for 3.5% slower generation.
  See [BENCHMARKING.md](BENCHMARKING.md#current-baseline-agent-battery-with-a-125-w-power-cap).
- A handful of samples per leg where the CPU energy counter wrapped
  around are excluded from the CPU average (the poller leaves them
  blank).

## Method notes and pitfalls

How each leg was run, in outline:

1. Confirm the hardware state: card count, and each card's actual PCIe
   link width (`nvidia-smi --query-gpu=pcie.link.width.current,...` and
   `lspci -vv` `LnkSta`). Bus addresses shift when cards are moved, so
   re-list them (`lspci | grep -i nvidia`) after any physical change.
2. Stop the production service — it is enabled at boot and starts with
   its original 3-GPU layout after every reboot — then launch a manual
   `llama-server` with the leg's flags and confirm `/health`.
3. Point the harness's model config at that server under a fresh
   model-id, and clear any stale results for that id.
4. Start the metrics poller (2s interval: CPU power/temp/clock, per-GPU
   power/temp/utilization/core and memory clocks, fan speed) before the
   battery.
5. Launch the battery detached, and verify exactly one eval process is
   running before walking away.
6. When it finishes: stop the poller, take the per-task table from the
   harness's own report, pull per-request prompt/generation t/s from
   the server's own `print_timing` log lines (a manually launched server
   logs wherever its output was redirected, not to the journal), and
   compute telemetry stats from the poller CSV. **Check the GPU core clocks
under load** (~1,328 MHz here; anything near 405 MHz means the cards are
stuck in a low-power state).

Battery wall time: ~33-48 minutes for every valid leg (34 minutes for
the 3-GPU rerun, 33-45 for the 2-GPU legs, ~48 for the 1-GPU leg).

Pitfalls worth not repeating:

- **Check GPU clocks before trusting any result.** An early run with the
  cards stuck at their 405 MHz idle clock looked like a real finding until
  a same-state rerun showed it was an artifact. Low GPU power under load
  is the tell, and a surprising result deserves a rerun before
  explanations are built on it.
- `llama-bench` does not support `--spec-type`/MTP flags — MTP is a
  `llama-server`-only feature. Use the real Hermes harness, not
  `llama-bench`, for anything meant to match this methodology.
  (`llama-bench` was used only for quick split-mode sanity checks in the
  miscellaneous doc, never for the results.)
- The metrics poller orders its per-card columns (GPU0, GPU1, ...) by
  card UUID, not by PCI slot or `nvidia-smi` index. Totals are
  unaffected, but a per-card column can't be read as "the card in slot
  N" without mapping UUIDs to slots (`nvidia-smi --query-gpu=index,uuid,pci.bus_id`).
  In the two-card lane-variant legs the per-card power and temperature
  columns may be swapped between the two cards.
- Hermes Agent hard-requires >=64K context on the *actual* per-slot
  budget, not just what's claimed in its config — `--ctx-size` divides
  evenly across `--parallel` slots, so verify the real per-slot number.
- Don't relaunch a step because a status check landed before the
  process finished; verify the process is actually gone before
  concluding it crashed. Every early "silent failure" here was a run
  that was still going.
- Results are cached by (model-id, task, rep). A stale model-id with old
  results silently skips re-running and replays old data, so always
  clear a model-id's results before a fresh attempt with the same id.
- `pgrep -f`/`pkill -f` against a wrapper script's own command line
  self-matches when the wrapper's source contains the search string
  (e.g. a `while pgrep -f X; do ...` loop matches itself forever).
  Match on a PID captured at launch, or use the `[x]pattern` bracket
  trick, instead.
- On this board, populating the two x16 slots for full x16/x16 **disables
  the USB 3 ports** (USB 2 still works). On a headless machine with no
  integrated graphics that uses a USB Wi-Fi dongle, a dongle left in a
  USB 3 port means the box comes up with no video and no network and
  looks exactly like a failed POST. Keep the dongle on a USB 2 port, and
  keep a cheap display card on hand for debugging.
- A host's network address can change between sessions. Verify its SSH
  host key fingerprints match before trusting a new address, rather than
  assuming stale notes are still right.
