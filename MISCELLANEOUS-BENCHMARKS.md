# Miscellaneous benchmarks

One-off side tests that don't belong in the main GPU-scaling series
([GPU-SCALING.md](GPU-SCALING.md)) or the llama-bench-level numbers in
[BENCHMARKING.md](BENCHMARKING.md). Everything here runs the same real
workload as the GPU-scaling benchmarks: the 9-task x 3-rep Hermes agent
battery through a bubblewrap-sandboxed harness against a real
`llama-server` over HTTP, on the patched llama.cpp build (see
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md)). The
early sections ran on two cards at PCIe x16/x16; the later three-card,
P2P and Nemotron sections ran with all three cards installed (x16/x8/x8).
Where a run deviates from the standard harness settings, the deviation is
listed explicitly.

**This is an ongoing project, and more will follow.** This file starts
with side tests of Qwen3.8-27B, the production Qwen3.6 quant, a P2P and
tensor-split check, and NVIDIA's Nemotron-3.5-Lightning model, and further
tests will be added as sections here as they are run. Treat it as a living log, not a finished report.

**Patch verification (before any run):** both models below run the same
binary. The source tree has all 29 P100 patches applied (28 files
modified, no `.rej`/`.orig` files), no source file is newer than the
build, and the copy the service runs is byte-identical to that build.
Patches 19 and 20 remain off, as documented.

## Qwen3.8-27B (dense), Q5_K_XL, 2x P100

Model: `Qwen3.8-27B-UD-Q5_K_XL.gguf` (19.43 GiB, plain quant — no MTP
head, so no speculative decoding).

### Fastest config

A short `llama-bench` pass (`-ngl 99 -ts 1/1 -fa 1`, pp512/tg128, 2 reps)
was used only to pick the split mode, not for the results below:

| Split mode | pp512 (t/s) | tg128 (t/s) |
|---|---:|---:|
| `layer` | 137.4 | 13.43 |
| `row` | — failed to load the model — | — |

Row split doesn't load this model on two 16GB cards, so layer split is
the only working mode. For reference, the patched `llama-bench` decode for this model on three
cards is 13.2 t/s (see [BENCHMARKING.md](BENCHMARKING.md)), the same as
the 13.4 t/s above on two cards.

Server flags used for the battery:

```
llama-server -m Qwen3.8-27B-UD-Q5_K_XL.gguf -ngl 99 -ts 1/1 -fa on -sm layer \
  --ctx-size 65536 --cache-type-k q8_0 --cache-type-v q8_0 --parallel 1 \
  --jinja --cache-ram 0 --no-cache-idle-slots \
  --reasoning-budget 7500 \
  --reasoning-budget-message "Thinking budget reached. I will stop deliberating and act on what I know."
```

### Deviations from the standard harness settings

This model thinks at length and decodes at ~12 t/s, so the standard
limits (600s per task, 8192 max tokens per response) would cut it off
mid-thought. Changed for this run only:

| Setting | Standard | This run |
|---|---:|---:|
| Per-task hard timeout | 600s | **900s** |
| Per-request timeout | 600s | **900s** |
| `max_tokens` per response | 8192 | **12288** |
| Thinking budget | none | **7500 tokens** |

- The 7500-token budget is "about ten minutes of thinking" at the
  12.26 t/s decode rate measured live on this server (20 requests)
  before the run — deliberately generous, so the budget only trims
  runaway thinking rather than shaping normal behavior. An earlier
  attempt used a 1024-token budget; that was abandoned (about 3 of 27
  task-reps in) because a budget that tight doesn't reflect how the
  model would really be used. Its partial results are not included in
  the tables below.
- `max_tokens` was raised because with 8192, a full thinking budget
  would leave under 700 tokens for the actual answer or tool call.
- The per-task timeout was hard-coded in the harness; it is now read
  from an environment variable (`ABEVAL_TASK_TIMEOUT_S`, default 600),
  so every other run's behavior is unchanged.

**The thinking budget never actually triggered.** The longest single
generation in the whole run was 4,675 tokens, well under 7,500, and
nothing in the server log mentions the budget. In practice this run is
"no thinking cap" — the budget and raised limits were insurance that
turned out not to be needed for these tasks.

### Hermes agent-task results (27 task-reps)

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 247s |
| err_replay_patch | 100% | 170s |
| err_ambiguous_edit | 100% | 190s |
| err_case_search | 100% | 192s |
| err_hidden_search | 100% | 190s |
| err_big_output | 100% | 199s |
| err_multi_dir | 100% | 183s |
| err_inline_script | 100% | 613s |
| err_big_file_read | 33% | 632s |
| **TOTAL** | **93%** | **291s** |

Best `err_hidden_search` result in the series (100%, versus 0-67% for
the Qwen3.6 legs), and 93% overall — the same as the Qwen3.6
Q4_K_XL runs on two and three cards. It pays for it in wall time: 291s average versus 72s
for Qwen3.6-35B-A3B Q3_K_XL on the same two cards.

Both failures (`err_big_file_read` reps 1 and 2, 879s and 429s) were
context overflows, not timeouts: the agent read the 6000-line file in
many overlapping chunks, filled the 64K context, and this eval config
runs with auto-compaction disabled, so it hard-stopped. Same failure
mode seen once on Qwen3.6 in the x8/x8 leg. No task hit the 900s cap;
the slowest completed task took 772s (`err_inline_script`).

### Throughput (116 requests, from `llama-server`'s own per-request timings)

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 24.2 | 191.7 | 123.0 |
| Generation (t/s) | 9.4 | 13.2 | 12.0 |

### System telemetry (3,806 samples, ~2h17m, 2s interval)

| Metric | min | max | avg |
|---|---:|---:|---:|
| CPU power (W) | 21.3 | 78.6 | 54.0 |
| CPU package temp (°C) | 45.0 | 67.0 | 60.8 |
| CPU avg clock (MHz) | 3292.8 | 3764.5 | 3635.7 |
| GPU0 power (W) | 34.2 | 217.3 | 127.6 |
| GPU0 temp (°C) | 49.0 | 73.0 | 67.5 |
| GPU0 utilization (%) | 0 | 100 | 57.2 |
| GPU1 power (W) | 36.0 | 218.6 | 132.2 |
| GPU1 temp (°C) | 48.0 | 72.0 | 67.1 |
| GPU1 utilization (%) | 0 | 100 | 58.0 |
| Shroud fan speed (RPM) | 1542 | 2896 | 2609.0 |

Power draw is the highest of any run on this box (avg ~128-132W per
card, peaks ~218W): a dense 27B model reads every weight for every
token, so the cards stay busy rather than idling between MoE expert
lookups.

### 3 cards

The same run on all three cards (`-ts 1/1/1`, x16/x8/x8) with identical
settings (7,500-token thinking budget, 900s task cap, 12,288 max tokens).
Both runs share the settings above, so the two card counts compare
directly.

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 252s |
| err_replay_patch | 100% | 205s |
| err_ambiguous_edit | 100% | 235s |
| err_case_search | 100% | 259s |
| err_hidden_search | 100% | 162s |
| err_big_output | 100% | 170s |
| err_multi_dir | 100% | 166s |
| err_inline_script | 67% | 583s |
| err_big_file_read | 67% | 386s |
| **TOTAL** | **93%** | **269s** |

The two misses: `err_inline_script` rep 1 hit the 900s task cap (the only
timeout in any Qwen3.8 run; the slowest task on two cards took 772s), and
`err_big_file_read` rep 2 was a context overflow, the same failure seen
elsewhere in this series. The thinking budget never fired (longest
generation 4,826 tokens).

| | 2 cards | 3 cards |
|---|---:|---:|
| Ok% | 93% | 93% |
| Avg task wall | 291s | 269s |
| Prompt processing avg (min–max) | 123.0 (24.2–191.7) t/s | 157.0 (64.6–251.4) t/s |
| Generation avg (min–max) | 12.0 (9.4–13.2) t/s | 11.9 (9.9–13.2) t/s |
| Busy GPUs + CPU, average power | 314 W | 371 W (peak 643 W) |
| Energy per task | ~91 kJ | ~100 kJ |

As with the MoE models, a third card adds prompt-processing speed here
(+28%) but no generation speed, and costs about 57 W more. Battery wall
time was about 126 minutes (3,402 telemetry samples).

Per card on three cards, by slot: the x16 card (`01:00.0`) averaged 98 W
and 56.5°C (max 65); the middle-slot x8 card (`02:00.0`) averaged 116 W
and 73.7°C (**max 80°C**); the other x8 card (`03:00.0`) averaged 102 W
and 52.5°C (max 59). The middle-slot card's core clock dropped as low as
**810 MHz** at its hottest (average across busy cards 1,322 MHz), the
clearest thermal throttling in any run so far. Sustained dense-model load
is the case where that card's airflow matters most.

## Qwen3.6-35B-A3B (MoE), Q4_K_XL, 2x P100

Model: `Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL.gguf` — the production model and
quant, run here on two cards instead of the three it normally uses.
Standard harness settings (600s task cap, 8192 max tokens, no thinking
budget), so this run is directly comparable to the GPU-scaling legs.

### Fastest config

Same short `llama-bench` check as above (`-ngl 99 -ts 1/1 -fa 1`,
no MTP, since `llama-bench` doesn't support it):

| Split mode | pp512 (t/s) | tg128 (t/s) |
|---|---:|---:|
| `layer` | 455.6 | 72.53 |
| `row` | — failed to load the model — | — |

Layer split again, and again the only mode that loads. On three cards
the same `llama-bench` check gives 70.6 t/s decode (see the three-card
section below), about the same as the 72.53 t/s here on two cards.

Server flags used for the battery (the production flags, with the
tensor split changed for two cards):

```
llama-server -m Qwen3.6-35B-A3B-MTP-UD-Q4_K_XL.gguf -ngl 99 -ts 1/1 -fa on -sm layer \
  --ctx-size 65536 --cache-type-k q8_0 --cache-type-v q8_0 --parallel 1 \
  --jinja --cache-ram 0 --no-cache-idle-slots \
  --spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.75
```

### Hermes agent-task results (27 task-reps)

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 64s |
| err_replay_patch | 100% | 94s |
| err_ambiguous_edit | 100% | 75s |
| err_case_search | 100% | 69s |
| err_hidden_search | 33% | 95s |
| err_big_output | 100% | 84s |
| err_multi_dir | 100% | 72s |
| err_inline_script | 100% | 104s |
| err_big_file_read | 100% | 94s |
| **TOTAL** | **93%** | **83s** |

Every task exited cleanly; the only misses are 2 of 3 `err_hidden_search`
reps failing the objective check. That task has been the noisy one
across every leg (0-67% on the other Qwen3.6 runs, 100% on Qwen3.8), and
with only 3 reps per cell a 33% result versus 0% or 67% isn't a
meaningful difference between quants.

### Throughput (116 requests, from `llama-server`'s own per-request timings)

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 63.4 | 511.2 | 302.8 |
| Generation (t/s) | 65.9 | 99.0 | 85.0 |

MTP draft acceptance averaged 95.6% across those requests.

### System telemetry (1,146 samples, ~38min, 2s interval)

| Metric | min | max | avg |
|---|---:|---:|---:|
| CPU power (W) | 21.3 | 60.6 | 50.9 |
| CPU package temp (°C) | 45.0 | 64.0 | 58.6 |
| CPU avg clock (MHz) | 3390.3 | 3765.2 | 3638.2 |
| GPU0 power (W) | 34.0 | 191.4 | 102.9 |
| GPU0 temp (°C) | 47.0 | 67.0 | 62.1 |
| GPU0 utilization (%) | 0 | 100 | 50.7 |
| GPU1 power (W) | 35.6 | 194.8 | 115.7 |
| GPU1 temp (°C) | 46.0 | 69.0 | 63.7 |
| GPU1 utilization (%) | 0 | 100 | 55.6 |
| Shroud fan speed (RPM) | 1450 | 2641 | 2379.5 |

### What this run settles

The same model and quant on two cards and on three cards, plus the
Q3_K_XL run from the GPU-scaling series for comparison (all at normal GPU
clocks):

| | Cards | Gen avg (t/s) | PP avg (t/s) | Avg task wall | Ok% |
|---|---:|---:|---:|---:|---:|
| Q4_K_XL (rerun 2026-09-24) | 3 | **85.2** | **325.8** | **70s** | 93% |
| Q4_K_XL | 2 | 85.0 | 302.8 | 83s | 93% |
| Q3_K_XL (x16/x16) | 2 | 82.0 | 283.9 | 72s | 89% |

- **Quant doesn't affect speed here.** Q4_K_XL on two cards is as fast
  as Q3_K_XL on two cards (85.0 vs. 82.0 t/s).
- **A third card adds no single-stream speed.** Three cards and two
  cards give the same generation speed (85.2 vs. 85.0 t/s), so for
  single-stream use the extra cards are about capacity, not speed.
- Quality is the same: 93% on both two and three cards, with the misses
  in the noisy `err_hidden_search` task.

## P2P bandwidth test and tensor split (three cards)

A check on whether direct GPU-to-GPU (P2P) transfers or tensor split
could help the multi-card setup. Run on three cards at x16/x8/x8 with the
GPUs idle.

### P2P bandwidth and latency

NVIDIA's `p2pBandwidthLatencyTest` sample, built for sm_60 with CUDA 12.9
(the sample was removed from the cuda-samples `master` branch in the v13.4
update, so it was built from the `v12.9` tag). All pairs report
"can access peer".

| | P2P off | P2P on |
|---|---|---|
| One-way bandwidth between cards | 6.0–6.5 GB/s | 5.2–6.6 GB/s |
| Two-way bandwidth between cards | 6.6–6.7 GB/s | 10.3 GB/s |
| GPU-to-GPU latency | 10.5–18.2 µs | 1.2–1.3 µs |

P2P is active: latency drops about 10x and two-way bandwidth rises about
55%. One-way bandwidth doesn't improve (and is slightly lower for some
pairs), and the ~6 GB/s ceiling is probably set by the x8 links. On a
workload that moves only small activations between cards per token, a
saving of ~10-15 microseconds is tiny next to a per-token time of ~12 ms.

### Tensor split with P2P

llama.cpp in this build supports `--split-mode tensor` and a
`GGML_CUDA_P2P` switch. Production uses layer split with P2P off.
`llama-bench` on three cards (Q4_K_XL, no MTP):

| Split mode | pp512 (t/s) | tg128 (t/s) |
|---|---:|---:|
| `layer` (production) | 459.7 | 70.6 |
| `tensor`, P2P on | 625.8 | crashes |
| `tensor`, P2P off | 624.2 | crashes |

Tensor split is faster for prompt processing (+36%) but **crashes during
decode** on three cards (`GGML_ASSERT(bcj.nodes[i]) failed` in the
multi-GPU backend), with or without P2P; with MTP enabled the server
crashes on the first request the same way. The log also says the
optimized all-reduce path failed to initialize (`n_devices != 2?`) and
fell back to a slower generic path, so the P2P fast path appears to need
exactly two GPUs. Tensor split is not usable for generation on three
cards in this build. It has not been tried on two cards.

**Build note:** linking the sample through the conda gcc's own sysroot
stamped an `x86-64-v3` (AVX2) requirement into the binary, which the
Sandy Bridge-E CPU can't run (`CPU ISA level is lower than required`).
Compiling with `nvcc` and the conda gcc, then linking with the system
`g++`, fixes it.

## NVIDIA Nemotron-3.5-Lightning-30B-A3B, Q4_K_XL, 2 and 3 cards

Model: `NVIDIA-Nemotron-3.5-Lightning-30B-A3B-UD-Q4_K_XL.gguf` (unsloth
quant, 25.5 GB). A different model family from everything above: a hybrid
Mamba + Transformer mixture-of-experts (`nemotron_h_moe`, 31B total, 3.5B
active) with MTP layers and a thinking mode. It loads and runs on the
patched build with no changes, and MTP speculative decoding works
(draft acceptance 92-93%). A quick `llama-bench` check on two cards
(layer split, no MTP) gave pp512 380.8 t/s and tg128 87.7 t/s.

**Cards.** Both runs had all three P100s installed (x16/x8/x8). The
"2 cards" run used `CUDA_VISIBLE_DEVICES` to expose only two of them —
the x16 card and one x8 card — so the third card sat idle (~25 W). The
"3 cards" run used all three with `-ts 1/1/1`. On two cards the model is
a tight fit: one card held 14,531 of 16,384 MiB with the 65K context.

**Server flags** (other than the card selection, identical for both):

```
llama-server -m NVIDIA-Nemotron-3.5-Lightning-30B-A3B-UD-Q4_K_XL.gguf -ngl 99 -fa on -sm layer \
  --ctx-size 65536 --cache-type-k q8_0 --cache-type-v q8_0 --parallel 1 \
  --jinja --cache-ram 0 --no-cache-idle-slots \
  --spec-type draft-mtp --spec-draft-n-max 3 --spec-draft-p-min 0.75 \
  --temp 1.0 --top-p 0.95
```

**Sampling.** The model's authors recommend temperature 1.0 and top-p
0.95, unlike the other runs in this series, which used the server
defaults (the harness sends no sampling parameters itself). A short probe
of 7 checkable prompts x 4 reps under each setting, interleaved, passed
28 of 28 both ways with the same speed (118.3 vs. 118.5 t/s) and MTP
acceptance (89.1% vs. 89.0%), so it couldn't separate them. The
recommended setting was used. Thinking was left at the chat template's
default, and the standard harness limits applied (600s task cap, 8192 max
tokens).

### Hermes agent-task results (27 task-reps each)

| Task | 2 cards ok% | 2 cards wall | 3 cards ok% | 3 cards wall |
|---|---:|---:|---:|---:|
| err_python_env | 100% | 128s | 100% | 161s |
| err_replay_patch | 100% | 141s | 100% | 152s |
| err_ambiguous_edit | 100% | 136s | 67% | 175s |
| err_case_search | 100% | 125s | 100% | 166s |
| err_hidden_search | 33% | 129s | 0% | 101s |
| err_big_output | 100% | 126s | 100% | 114s |
| err_multi_dir | 100% | 119s | 100% | 166s |
| err_inline_script | 100% | 137s | 100% | 143s |
| err_big_file_read | 0% | 187s | 33% | 206s |
| **TOTAL** | **81%** | **136s** | **78%** | **154s** |

The `err_big_file_read` failures (3 of 3 reps on two cards, 2 of 3 on
three) are the same context overflow seen elsewhere in this series: the
agent reads the 6,000-line file in many chunks, fills the 64K context,
and this harness has auto-compaction off. The other misses are
`err_hidden_search` (the unstable task across the whole series) and one
`err_ambiguous_edit` rep on three cards. Every other task passed. At 3
reps per cell, the 81% vs. 78% difference between card counts is within
noise.

### Throughput (from `llama-server`'s own per-request timings)

| | 2 cards | 3 cards |
|---|---:|---:|
| Requests | 132 | 134 |
| Prompt processing avg (min–max) | 334.4 (108.5–552.6) t/s | 444.4 (96.9–668.6) t/s |
| Generation avg (min–max) | 103.6 (74.2–116.5) t/s | 103.9 (85.7–113.8) t/s |
| MTP draft acceptance | 91.9% | 92.6% |

### Power, temperature and clocks

| | 2 cards | 3 cards |
|---|---:|---:|
| Battery wall time | ~64 min | ~71 min |
| Busy GPUs + CPU, average | 288 W | 352 W |
| Busy GPUs + CPU, peak sample | 437 W | 581 W |
| Energy per task (avg power x avg task time) | ~39 kJ | ~54 kJ |
| Idle third card | ~25 W (31°C) | — |
| GPU core clock while busy | 1,315 MHz avg (min 1,088) | 1,327 MHz avg (min 1,189) |

Per card (by slot, from the telemetry):

| Card | 2 cards: power / temp avg (max) / util | 3 cards: power / temp avg (max) / util |
|---|---|---|
| x16 (`01:00.0`) | 105 W / 59°C (66) / 52% | 91 W / 57°C (63) / 43% |
| x8 (`02:00.0`, middle slot) | 128 W / **77.5°C (80)** / 63% | 103 W / **71.5°C (79)** / 42% |
| x8 (`03:00.0`) | idle: 25 W / 32°C | 103 W / 55°C (59) / 49% |

The card in the middle slot (`02:00.0`) runs 15-20°C hotter than the
others in both runs, peaking at 79-80°C. It was also the warmest card in
the Qwen3.6 three-card rerun. On two cards its core clock dipped as low as
1,088 MHz, the lowest of any run, which may be early thermal limiting but
isn't confirmed. Worth checking that card's airflow.

### How it compares

Same cards and harness, but different models and different sampling:

| Model, Q4_K_XL | Cards | Gen (t/s) | PP (t/s) | Avg task wall | Ok% |
|---|---:|---:|---:|---:|---:|
| Qwen3.6-35B-A3B | 2 | 85.0 | 302.8 | 83s | 93% |
| Qwen3.6-35B-A3B | 3 | 85.2 | 325.8 | 70s | 93% |
| Nemotron-3.5-Lightning-30B-A3B | 2 | 103.6 | 334.4 | 136s | 81% |
| Nemotron-3.5-Lightning-30B-A3B | 3 | 103.9 | 444.4 | 154s | 78% |

- **Faster tokens, slower and less reliable tasks.** Nemotron generates
  about 22% faster (104 vs. 85 t/s) and reads prompts faster, yet its
  tasks take about 1.6-2.2x longer and pass less often (78-81% vs. 93%).
  The likely reason is more reasoning and more turns per task, but tokens
  and turns per task weren't measured here.
- **Card count barely matters for generation again:** 103.6 vs. 103.9
  t/s on two vs. three cards, consistent with the Qwen3.6 result. Prompt
  processing did rise with three cards (+33%, vs. +8% for Qwen3.6); the
  cause isn't determined.
- **Power and energy:** three cards drew about 64 W more than the two
  busy cards alone, or about 40 W more counting the idle third card's
  ~25 W. Energy per task (39-54 kJ) is roughly 1.8-2.5x Qwen3.6's
  (~22 kJ), mostly because the tasks take longer.
- **Caveats:** two model families, different sampling settings, and only
  3 reps per task, so small differences shouldn't be over-read.
