# Kmic-68 fork: isolated Docker evaluation + Hermes agent battery (125 W)

Last updated 2026-09-27.

**Update 2026-10-05:** this document records the evaluation as it was run on 2026-09-27 and is
left unchanged below. Production has since moved to this fork with different settings: an NCCL
build (`NCCL_P2P_LEVEL=SYS`), `-lm none -fit off`, MTP `n-max 3` / `p-min 0.0`, `-b 2048`, and a
150 W cap. See "What production runs now" in [SOFTWARE.md](SOFTWARE.md) and the
[3-GPU measurements](3gpu-optimization/).

Evaluation of [Kmic-68's `llama.cpp` fork](https://github.com/Kmic-68/llama.cpp) (`p100-optimizations`
branch) — a separate, independent set of Pascal/P100 CUDA-kernel optimizations from the
29-patches build this repo otherwise documents. Credit for the fork and its tuning work is
Kmic-68's; this doc only records how it performed on this box's hardware under a real
agent-task workload. The fork ships its own build/tuning docs (`p100-docs/BUILD.md`,
`p100-docs/QUICKSTART.md`, `p100-docs/FINDINGS.md`, `p100-docs/CHANGES.md`) which this
evaluation followed and is worth reading directly for the fork's own kernel-level findings.

**Isolation**: built and run entirely inside a Docker container (`nvidia/cuda:12.9.1-devel-ubuntu22.04`
base), with the production `llama-server` build, its systemd service, and this repo's git checkout
never touched. The NVIDIA Container Toolkit had to be installed on this box first (it wasn't
present) to get `--gpus all` working in Docker at all.

## Build

```
cmake -B build-opt -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=60 \
  -DGGML_CUDA_NCCL=OFF -DGGML_CUDA_FA_QUANTS=all -DCMAKE_BUILD_TYPE=Release
cmake --build build-opt --config Release
```

`CMAKE_CUDA_ARCHITECTURES=60` only, confirmed in the CMake cache — the fork's docs warn that a
second architecture silently disables its Pascal-specific `mmvq.cu` tuning. The
`nvidia/cuda:12.9.1-devel-ubuntu22.04` image's `nvcc` 12.9 + `gcc` 11.4 pairing compiles for
`sm_60` natively; no host CUDA/gcc12 toolchain workaround was needed.

## Correctness gate (`tools/gate.sh`)

Qwen3.8-27B, Q6_K_XL quant (the quant the fork's own gate script expects):

| Check | Result | Gate |
|---|---:|---|
| Decode (`tg256`, cold cards) | 21.76 ± 4.01 t/s | upstream reference at fork point: 17.51 t/s |
| Perplexity | 2.6067 ± 0.0198 | band: 2.6209 ± 0.0199 |
| Flash-attention op tests | 4/4 backends passed | — |

All three passed cleanly before any model-loading or agent-battery testing proceeded, per the
fork's own gating discipline ("run perplexity early — a change can pass the op suite and still
be wrong in real inference").

## Server flags, and why each one — cited from Kmic-68's own `CHANGES.md`/`QUICKSTART.md`

This fork's own repo, not `houston-p100-cluster`'s unrelated fork, is the source of truth for
every flag below:

| Flag | Value | Evidence |
|---|---|---|
| `-sm tensor -ts 1/1/1` | tensor split, 3 cards | `CHANGES.md` §4/§6 describe extensive tensor-parallel correctness work (peer-copy races found and fixed, e.g. commit `b67848c64`) — this is the fork's primary design target, not an experimental mode. |
| `-ctk q4_0 -ctv q4_0` | quantized KV cache | `CHANGES.md`'s header states every commit in the changelog is measured against "q4_0 KV cache, `-sm tensor`" — not a guess, the fork's whole optimization set is built and validated around this. |
| `--spec-type draft-mtp --spec-draft-n-max 4 --spec-draft-p-min 0.2` | MTP speculative decoding | Kmic-68's own `qwen-server` reference config (`QUICKSTART.md`). §10 explains why: this fork added its own depth-scheduled draft-length logic (commit `8736a7ef3`) on top of `p-min`, so a naive higher `p-min` doesn't transfer the way it might on a different fork. |
| `-ngld 99 -ubd 64 -ctkd q4_0 -ctvd q4_0` | draft-context tuning | Fork-added feature (commit `74de4a1bd`, "a separate ubatch for the draft context"). |
| `-b 32768 -ub 2048` | batch sizing | §12: "`qwen-server` keeps `-ub 2048` for text only" — their own reference server's setting. |
| `GGML_CUDA_GRAPHS_PRE_VOLTA=3` | env | §11 commit `5d27ef852`: "**the serving default**" — not optional tuning, their default. |
| `GGML_CUDA_P2P=1` | env | §4: peer copies over a dedicated stream for the tensor-parallel exchange — validated for this exact split mode. |
| `LLAMA_SPEC_SAMPLE_TEMP=1.0 LLAMA_SPEC_DRAFT_TOPK=20` | env | §11 commits `dbe945b03`/`eb1bf26fe`: sampled MTP draft + speculative-sampling verify, measured **+15% tokens/cycle at 2k, +8% at 260k**, lossless — but only paired with real sampling (below), per commit `5b46e14ca`: "every MTP figure before it was measured at temp 0.3." |
| `--temp 1.0 --top-k 20 --top-p 0.95 --min-p 0.0` | sampling | §11 commit `5b46e14ca`: "serving uses the model card's sampling." |
| `--parallel 1` | | Without it, llama-server defaults to 4 slots; at this context size that caused an outright OOM kill *during model load*, before a single request was served — this box only has 15 GB RAM. |
| `-c 262144` | full native context | Every depth measurement in `CHANGES.md` §10–§13 is anchored around 2k–260k context, which this flag matches explicitly. |
| `--jinja --cache-ram 0 --no-cache-idle-slots` | general llama.cpp hygiene | Not fork-specific, but load-bearing regardless: omitting `--cache-ram 0`/`--no-cache-idle-slots` produced a real host-RAM OOM kill on this box during testing (llama.cpp's default cross-request prompt cache growing unbounded across large-context tasks). |

Dropped: `--no-spec-draft-backend-sampling` — appears nowhere in Kmic-68's docs, and §10 commit
`66bbd1212` suggests it's a no-op under `-sm tensor` anyway ("both the draft and verify samplers
run on the CPU" already in this mode).

## Hermes agent battery — 125 W power cap

Standard harness settings (9 tasks x 3 reps), run against this box's real production model file
(Qwen3.8-27B dense, Q5_K_XL quant) with all three cards capped at 125 W, matching this repo's
documented 125 W baseline discipline (see `BENCHMARKING.md`), using the server flags above.

**Results (27 task-reps):**

| Task | ok% | avg wall |
|---|---:|---:|
| err_python_env | 100% | 125s |
| err_replay_patch | 100% | 142s |
| err_ambiguous_edit | 100% | 164s |
| err_case_search | 100% | 114s |
| err_hidden_search | 100% | 105s |
| err_big_output | 100% | 105s |
| err_multi_dir | 100% | 109s |
| err_inline_script | 100% | 141s |
| err_big_file_read | 67% | 303s |
| **TOTAL** | **96%** | **145s** |

`err_big_file_read` is the one weak cell — same task that's shown instability on other legs in
this repo's other benchmark docs at low rep counts; not treated as a fork-specific regression
without more reps.

**Throughput** (128 real requests, from `llama-server`'s own per-request timings):

| | min | max | avg |
|---|---:|---:|---:|
| Prompt processing (t/s) | 90.5 | 240.9 | 188.0 |
| Generation (t/s) | 30.4 | 56.7 | 45.1 |

MTP draft acceptance averaged **59.5%** across all sampled requests. This looks low next to a
naive expectation, but it isn't an acceptance-ratio optimization: `--spec-draft-n-max 4` drafts
more tokens per verify cycle than a shorter draft would, so even a lower per-token acceptance
percentage lands more net tokens per cycle — consistent with `CHANGES.md`'s own framing of the
sampled-draft feature as a tokens-per-cycle win, not an acceptance-ratio win.

**System telemetry** (1,843 samples, 71 min, 2s interval, from `poll-metrics.py`; GPU0-2 columns
are ordered by card UUID, not physical slot, per this repo's usual caveat):

| Metric | min | avg | max |
|---|---:|---:|---:|
| CPU power (W) | 23.8 | 58.5 | 64.7 |
| CPU package temp (°C) | 48.0 | 59.5 | 64.0 |
| CPU avg clock (MHz) | 3502.4 | 3621.2 | 3778.1 |
| GPU0 power (W) | 32.3 | 92.0 | 168.8 |
| GPU0 temp (°C) | 42.0 | 50.0 | 54.0 |
| GPU0 utilization (%) | 0 | 52.0 | 100 |
| GPU1 power (W) | 34.4 | 94.5 | 165.6 |
| GPU1 temp (°C) | 50.0 | 57.2 | 61.0 |
| GPU1 utilization (%) | 0 | 82.3 | 100 |
| GPU2 power (W) | 38.4 | 95.0 | 169.3 |
| GPU2 temp (°C) | 56.0 | 65.9 | 69.0 |
| GPU2 utilization (%) | 0 | 53.2 | 100 |
| Shroud fan speed (RPM) | 1950 | 2495 | 2694 |

GPU core clocks while busy (>50% utilization): GPU0 1261 MHz avg (min 1088, max 1328), GPU1
1236 MHz avg (min 1088, max 1328), GPU2 1227 MHz avg (min 974, max 1328) — all in the normal
~1,328 MHz busy-clock range this repo's other legs check for (no stuck-405 MHz failure mode).

Total GPUs+CPU power: avg 339.8 W, peak sample 477.8 W.

**`-sm tensor` did not crash** across this entire battery (many sequential and slot-interleaved
requests, several with large contexts) — worth noting against `TROUBLESHOOTING.md`'s documented
second-request `-sm tensor` crash (`GGML_ASSERT(bcj.nodes[i])`, tracked upstream as
ggml-org/llama.cpp#29466) on the 29-patches build. Whether that's because this fork's tensor-split
implementation differs enough to avoid the bug, or the crash is more narrowly triggered than
"any second request," hasn't been investigated — flagged here as a data point, not a fix.

## Side by side vs. this repo's existing Qwen3.8-27B dense benchmark

`MISCELLANEOUS-BENCHMARKS.md` already has a Qwen3.8-27B dense, Q5_K_XL benchmark on the
29-patches build, 3 cards. It's the closest existing same-model reference in this repo, but the
comparison has real confounds, listed after the table — this isn't a controlled A/B, both builds
differ in more than one variable at once.

| | Existing benchmark (29-patches build) | This fork |
|---|---:|---:|
| Cards | 2x P100 | **3x P100** |
| Split mode | `-sm layer` | `-sm tensor` |
| Speculative decoding | none (plain quant, no MTP head) | MTP, `n-max 4`/`p-min 0.2` |
| Context | 65536 | 262144 |
| Power | default (uncapped, ~250 W) | capped at 125 W |
| Task ok% | 93% | **96%** |
| Avg task wall | 291s | **145s** |
| Prompt processing avg | 123.0 t/s | **188.0 t/s** |
| Generation avg | 12.0 t/s | **45.1 t/s** |
| Busy GPUs+CPU avg power | 314 W | **339.8 W** (at a lower cap, but a third card) |

**Confounds, plainly stated:** this fork's run used all three P100s; the existing benchmark's
2-card figures are used here since that's its primary Qwen3.8-27B result (it also has a 3-card
side test: 157.0 t/s prompt / 11.9 t/s generation / 371 W, still far below this fork's generation
number). A third card is more compute and VRAM available regardless of fork, so some of the
prompt-processing gap is card count, not only the fork. The existing benchmark has no MTP at all
(a plain quant with no draft head), so most of the generation-speed gap is MTP existing on one
side and not the other, not purely a fork-vs-fork comparison. Context differs (65536 vs.
262144). Power differs (the existing run is uncapped at ~250 W; this fork's run is capped at
125 W on a third card and still generates far faster). The existing benchmark also runs with a
raised 900s task timeout and 12,288 max-tokens limit for its own reasoning-budget reasons; this
fork's run used the harness's standard 600s/8192 limits. None of these differences were
controlled for, so treat this table as "what each setup actually achieves on this hardware," not
as an isolated measurement of the fork's improvement.
