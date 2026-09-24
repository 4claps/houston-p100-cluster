# Miscellaneous benchmarks

One-off side tests that don't belong in the main GPU-scaling series
([GPU-SCALING.md](GPU-SCALING.md)) or the llama-bench-level numbers in
[BENCHMARKING.md](BENCHMARKING.md). Everything here runs the same real
workload as the GPU-scaling benchmarks: the 9-task x 3-rep Hermes agent
battery through a bubblewrap-sandboxed harness against a real
`llama-server` over HTTP, on the patched llama.cpp build (see
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md)), with
both remaining P100s at PCIe x16/x16. Where a run deviates from the
standard harness settings, the deviation is listed explicitly.

**This is an ongoing project, and more will follow.** This file starts
with two side tests — Qwen3.8-27B and the production Qwen3.6 quant, both
on two cards — and further tests will be added as sections here as they
are run. Treat it as a living log, not a finished report.

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
the only working mode. (The 3.07 t/s three-card figure for this model in
`BENCHMARKING.md` was measured while the GPU clocks were stuck low — see
the erratum there — so it isn't comparable to the number above.)

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
- **This replaces an earlier, wrong conclusion.** The first version of
  this section said three cards were ~4x slower than two, based on the
  original 3-GPU run (20.8 t/s). That run had its GPU clocks stuck at
  405 MHz; see the Correction in [GPU-SCALING.md](GPU-SCALING.md#correction-the-first-3-gpu-run-was-clock-locked).
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
