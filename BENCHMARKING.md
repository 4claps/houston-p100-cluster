# Benchmarking

Benchmark results for this box, kept separate from
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) (which
covers the build/patch process itself) so new runs have an obvious place
to land without that doc growing without bound.

All runs below use the patched build described in
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) unless
otherwise noted, with tensor-split evenly across all three P100s.

## Erratum (2026-09-24): the absolute numbers below were very likely measured with GPU clocks stuck low

Every result in this document was recorded on 2026-09-21 or 2026-09-22.
GPU clocks weren't logged for these runs, but telemetry from the first
3-GPU battery on 2026-09-22 shows the core clocks pinned at 405 MHz (the
P100's idle clock, about 30% of the ~1,328 MHz it normally runs under
load) — see the Correction in
[GPU-SCALING.md](GPU-SCALING.md#correction-the-first-3-gpu-run-was-clock-locked).
Re-running the first table's command on 2026-09-24 gives far higher
numbers than recorded below, which strongly suggests these runs were made
in the same state. The cause is unknown, and by 2026-09-23 the clocks
were normal.

What that means for the numbers below:

- **Absolute throughput is far too low.** The same `llama-bench` command
  as the first table below (Qwen3.6-35B-A3B Q4_K_XL, patched build, all
  three cards, `-ts 1/1/1`) gives pp512 459.7 t/s and tg128 70.6 t/s
  when re-run at normal clocks, versus 89.86 and 19.66 t/s recorded
  below. The dense-model and concurrency figures are probably similarly
  low but were not re-measured.
- **The relative comparisons** (patched vs. baseline, the MTP gain, the
  `p-min` sweep) compare runs made in the same state, so they probably
  still hold in direction, but the percentages haven't been re-measured
  at normal clocks.
- **The explanation for low GPU power draw in the concurrency section is
  very likely wrong.** The cards drew little power because they were
  clocked down, not because single-stream decode is inherently
  memory-bound and low-power. The concurrency "sweet spot" and the
  PCIe/NCCL explanation for its shape should be treated as unverified
  until re-measured.

## Patched vs. baseline: Qwen3.6-35B-A3B (MoE)

Q4_K_XL quant, `-ts 1/1/1`, `-ngl 99`, flash attention on. 5 repetitions
each, `llama-bench` defaults otherwise.

| | pp512 | tg128 (decode) |
|---|---:|---:|
| Baseline (stock `v0.4.0`) | 90.97 t/s | 12.91 t/s |
| Patched (29 patches) | 89.86 t/s | **19.66 t/s** |
| Δ | flat (within noise) | **+52.3%** |

Prompt processing is essentially unchanged, which matches the patch set's
own scope — it's targeted at decode, not prompt processing or
large-batch serving. The measured +52.3% decode improvement is lower than
the patch repo's own headline number for this same model (~+80%), which
is expected: their number used a different quant, MTP speculative
decoding, and a realistic sampler configuration, none of which this
benchmark reproduces — this is a plain `llama-bench` pp/tg comparison,
not a reproduction of their full methodology. The relative improvement
from the patches on identical hardware and identical everything-else is
the useful number here, and it's real and repeatable.

Before this table could be trusted, two false starts had to be ruled
out first:

- The very first full-context load attempt (no `-c` flag) thrashed this
  box's 16GB of RAM — `llama-cli` defaults to the model's native max
  context when none is given, which for this model is large enough that
  loading it alongside a full KV-cache allocation drove the box into
  page-cache thrashing that looked like a hang (disk `read_bytes` grew to
  nearly 2x the model's own file size before it was killed). Fixed by
  always passing an explicit, reasonable `-c` for any interactive test.
- A second apparent hang (100% real wall-clock time, 0% GPU utilization,
  parked in a `futex_do_wait`) turned out to be `llama-cli`'s **default,
  automatic tensor-split** determination taking an very long time or
  getting stuck on this specific 3-GPU, no-NCCL setup. Passing an
  explicit `-ts` value instead of leaving it on auto-detect avoided it
  entirely and every run since has been reliable.

## Patched vs. baseline: Qwen3.8-27B (dense)

Same setup as above, Q5_K_XL quant, but a **dense** model this time
rather than MoE, to see how much of the patch set's benefit carries over.

| | pp512 | tg128 (decode) |
|---|---:|---:|
| Baseline (stock `v0.4.0`) | 41.38 t/s | 2.25 t/s |
| Patched (29 patches) | 41.13 t/s | **3.07 t/s** |
| Δ | flat (within noise) | **+36.4%** |

A real gain, but smaller than the MoE model's +52.3% — expected, since
several of the 29 patches are scoped to MoE routing or to the
gated-delta-net block (Qwen3.5/Qwen3-Next specific), and simply don't
fire at all on a dense model. What's left driving the +36.4% here is the
general `sm_60`/CUDA-kernel and host-side patches, which apply
regardless of architecture.

Decode throughput itself is far lower on this model than on the MoE one
(2-3 t/s vs 12-19 t/s) — not a patch-related regression, just the
architecture difference: all 27B dense parameters activate on every
token here, versus roughly 3B active parameters per token on the MoE
model. Prompt processing is flat in both cases either way, consistent
with the patch set being decode-focused.

## Batch size / concurrency scaling

Prompted by noticing GPU power draw and temps staying very low (under
50W, well off the 250W TDP) during the single-stream benchmarks above —
worth checking whether that's a sign of the cards being underused, or
just an expected property of the workload.

`llama-batched-bench` was run against the patched build (Qwen3.6-35B-A3B,
same setup as its table above) sweeping the number of concurrent
sequences (`-npl`) from 1 to 32, `-npp 128 -ntg 128 -c 16384`:

| Concurrent sequences (B) | Decode t/s (TG) | Total t/s |
|---:|---:|---:|
| 1 | 18.72 | 23.28 |
| 4 | 34.52 | 49.26 |
| 8 | **38.85** | **60.20** |
| 16 | 26.40 | 43.91 |
| 32 | 26.66 | 44.84 |

**Single-stream decode (B=1) is memory-bandwidth-bound, not
compute-bound**: generating each token means reading the active weights
out of VRAM once, with relatively little math done per byte read. The
SMs spend most of their time waiting on memory rather than computing, so
low power draw and low temps at B=1 are expected behavior, not a
misconfiguration — this is true of basically all GPUs doing batch-1 LLM
decode, not specific to the P100.

Confirmed directly: going from 1 to 8 concurrent sequences roughly
**doubles** decode throughput (18.72 → 38.85 t/s), and GPU power during
those runs spiked to 40-48W with individual cards briefly hitting 100%
utilization — well above the ~25-30W / 0% idle baseline seen at B=1.

Throughput **peaks around 8 concurrent sequences and regresses** at 16
and 32 (down to ~26-27 t/s) rather than continuing to climb. That tracks
with two hardware limits already known about this box (see
[HARDWARE.md](HARDWARE.md) and
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md)): two of
the three P100s train at PCIe x8 instead of their slots' native x16, and
this build has no NCCL. Both raise the cost of inter-GPU synchronization
under layer-split tensor parallelism, and that cost looks like it starts
dominating once there's enough concurrent traffic to saturate it.

Also observed while watching `nvidia-smi` during the sweep: only **one
GPU is ever at high utilization at a time**, never two or three
simultaneously. That's expected under the default layer-split mode
(`-sm layer`) with no NVLink — each GPU processes its assigned layers in
turn as the token's activation passes through the pipeline, so there's
no point in the pipeline where all three cards are computing at once.

**Practical takeaway**: for this specific box, **8 concurrent requests
looks like the practical sweet spot** if it's ever run multi-user rather
than single-stream — pushing concurrency further doesn't pay off given
the PCIe/NCCL limits above.

## gppm's power-saving mechanism is inert on P100 (and why it was removed)

`gppm` (github.com/crashr/gppm) was installed early in this box's setup specifically
to manage GPU idle power draw — these P100s sit at an elevated ~25-30W per card
whenever a model is loaded, even fully idle, and gppm was meant to bring that down.
It doesn't, and can't, on this hardware. This section documents why, since gppm has
since been removed and llama.cpp now runs as a plain systemd service instead.

### What gppm actually does

gppm's idle-power mechanism is entirely built on forcing an NVIDIA GPU's reported
performance state ("pstate") down to P8 via `nvidia-pstate`/NVML whenever none of
its managed llama.cpp instances have an active task, and back to P0 when a task
starts. It was written for and tuned against **Tesla P40** (GP102 chip).

### What was actually tested on this box

The pstate-forcing call was tested directly against these Tesla P100s (GP100 chip)
under three separate conditions:

1. **A completely idle GPU, no model loaded, no CUDA context at all.** Forcing
   pstate 8 reported success from the tool, but `nvidia-smi` continued to report
   `P0` and power draw was unchanged (29.90 W → 30.14 W — within measurement
   noise).
2. **`llama-server` loaded and idling** (the actual target scenario — a model
   staying resident most of the time). Same result: pstate stayed `P0`, power
   draw on the tested GPU was exactly unchanged (29.98 W → 29.98 W).
3. **Persistence mode on vs. off.** No measurable difference either way (~30W
   regardless).

Beyond the pstate call itself, two further checks ruled out any other lever:

- **Memory clock has exactly one supported value** on this card (715 MHz) — there
  is no lower-power memory P-state exposed at all, unlike GDDR5-equipped cards
  (P40) which do drop memory clock substantially at idle.
- **Power limit floor is 125W**, far above the ~25-30W idle draw already observed
  — capping the power limit lower has nothing to act on at these idle wattages.

### Why

The working theory: **P40 is GP102**, a die shared with consumer/workstation
Pascal parts that retain a display-capable heritage and the granular idle
power-gating that comes with it. **P100 is GP100**, a pure-HPC die built for
sustained compute, with HBM2 instead of GDDR5, and it was never designed to
power-gate for idle the way a display-capable Pascal part is. There's no
equivalent low-power path exposed through NVML/`nvidia-smi` on this chip at
all — this isn't a gppm configuration problem or a bug to fix, it's a hardware
capability gap.

### What gppm was kept around for instead, and why it was removed anyway

Since the power-management feature — the actual reason it was installed — does
nothing on this hardware, gppm was kept running for a while purely for its
secondary feature: llama.cpp instance supervision (config-driven enable/disable,
auto-restart on crash). That's a reasonable thing to want, but it's also not
worth the extra moving part (a whole separate daemon, venv, and config format)
just for supervision that a plain systemd unit does natively. gppm has been
uninstalled; llama.cpp now runs directly as a systemd service (see
[SOFTWARE.md](SOFTWARE.md)).

## MTP speculative decoding: measured gain

Prompted by a "why aren't the GPUs closer to max" question, which led to
researching what actually moves single-request throughput (as opposed to
concurrency, covered above) — llama.cpp supports self-speculative decoding
via a model's own multi-token-prediction (MTP) head: the model drafts
several tokens ahead in one pass, then verifies them all against the full
model in the same step. Unlike `--parallel`, this doesn't trade latency
for throughput — it's a straight win on a single request, since every
draft token is checked against the real model before being accepted
(deterministic sampling, no accuracy loss).

**The catch: the GGUF has to be MTP-converted, and ours wasn't — even
though it looked like it should already work.** Every server log this
whole time had been printing lines like `model has unused tensor
blk.64.nextn.eh_proj.weight -- ignoring`, which looked like the draft
head was already present. Enabling it (`--spec-type draft-mtp
--spec-draft-n-max 3`) against that file failed outright:

```
common_speculative_init_result: creating MTP draft context against the target model '...'
llama_init_from_model: context type MTP requested but model doesn't contain MTP layers
```

Those `nextn` tensors are present in the regular quant but not in a form
llama.cpp's MTP code recognizes — they're leftover checkpoint artifacts,
not a usable draft head. The fix was switching to unsloth's
`Qwen3.6-35B-A3B-MTP-GGUF` repo, which publishes the same `UD-Q4_K_XL`
quant we were already running, properly MTP-converted (~500MB larger,
same weights otherwise — see [SOFTWARE.md](SOFTWARE.md)). That one
loaded cleanly with the same flags and the MTP draft context actually
initialized.

Measured with the same live single-request test used earlier
(`/completion`, `n_predict: 512`, `temperature: 0`, single slot):

| | Generation (t/s) |
|---|---:|
| Baseline (no MTP) | ~19.2–19.7 |
| MTP (`--spec-draft-n-max 3`) | ~21.7–22.5 |
| Gain | **+15–17%** |

Worth being upfront that this is smaller than MTP's headline numbers
elsewhere (community reports of ~1.5–2x on other hardware). The likely
reason: speculative decoding's payoff depends on how cheap the draft
step is relative to the main model's per-token cost, and that ratio is
evidently less favorable on this hardware/quant/prompt combination than
on the setups those bigger numbers came from. Still a real, deterministic
gain with no measured downside, so it's now the production configuration
(see [SOFTWARE.md](SOFTWARE.md)) — the original non-MTP model file has
been deleted.

## MTP tuning sweep: p-min mattered, n-max and sampler didn't

The MTP result above used llama.cpp's defaults for everything except
`--spec-draft-n-max`. Since the patches repo's own 120 t/s number used a
wider verify batch (width-5) and a tuned acceptance threshold, it was
worth checking whether tuning those flags on our hardware helped the
same way. Same live single-request test as before (`/completion`,
`n_predict: 512`, single slot):

| Config | Generation (t/s) | vs. no-MTP baseline |
|---|---:|---:|
| No MTP | ~19.2–19.7 | — |
| `n-max=3, p-min=0.0` (original production) | ~21.7–22.5 | +15–17% |
| `n-max=5, p-min=0.0` | ~19.2–19.6 | ~0% |
| **`n-max=3, p-min=0.75`** | **~23.9–24.4** | **+23–25%** |
| `n-max=4, p-min=0.75` | ~24.0–24.2 | +23–25% (no gain over n-max=3) |
| `n-max=3, p-min=0.75`, realistic sampler (temp 0.7, top-p 0.8, top-k 20) | ~23.9–24.3 | same as greedy |

**Widening the draft window alone made things worse, not better.**
`n-max=5` with the default acceptance threshold erased essentially all of
the MTP gain. The likely reason: drafting further ahead means the later
draft tokens are less likely to match what the full model would actually
produce, so a wider window without a correspondingly tuned threshold just
adds verify overhead for tokens that mostly get rejected anyway.

**Raising the acceptance threshold (`--spec-draft-p-min 0.75`) was the
real lever**, on top of the original `n-max=3` — an additional +8–9%
beyond the untuned MTP result, for **+23–25% total** over no-MTP. Once
that threshold is set correctly, widening the draft window further
(`n-max=4`) added nothing measurable.

**Sampler strategy made no difference either way** — greedy
(`temperature: 0`) and the "realistic" sampler (temperature 0.7, top-p
0.8, top-k 20) that the patches repo's own numbers used produced
statistically indistinguishable throughput here. Whatever sampler-
dependent effect on MTP acceptance the patches repo's README warns about
("sampler settings change conclusions here by several points"), it
doesn't show up as a throughput difference in this test.

Production now runs `--spec-draft-n-max 3 --spec-draft-p-min 0.75` (see
[SOFTWARE.md](SOFTWARE.md)). None of this tuning touches output
correctness — speculative decoding always verifies drafted tokens
against the full model before accepting them, so every configuration
above produces identical output for a given prompt and seed; only
throughput differs.
