# Benchmarking

Benchmark results for this box, kept separate from
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) (which
covers the build/patch process itself) so new runs have an obvious place
to land without that doc growing without bound. Results on a real agent
workload are in [GPU-SCALING.md](GPU-SCALING.md) and
[MISCELLANEOUS-BENCHMARKS.md](MISCELLANEOUS-BENCHMARKS.md).

All runs below use the patched build described in
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) unless
otherwise noted, with tensor-split evenly across all three P100s
(`-ts 1/1/1`, slots at x16/x8/x8). They were measured on 2026-09-24, with
the GPU core clocks checked at ~1,328 MHz under load. The `llama-bench`
runs on the MoE model use the MTP-converted GGUF (llama-bench doesn't use
the MTP layers).

## Patched vs. baseline: Qwen3.6-35B-A3B (MoE)

Q4_K_XL quant, `-ts 1/1/1`, `-ngl 99`, flash attention on. 5 repetitions
each, `llama-bench` defaults otherwise.

| | pp512 | tg128 (decode) |
|---|---:|---:|
| Baseline (stock `v0.4.0`) | 460.98 ± 11.87 t/s | 54.69 ± 1.24 t/s |
| Patched (29 patches) | 468.18 ± 4.24 t/s | **69.54 ± 1.31 t/s** |
| Δ | +1.6% (within noise) | **+27.2%** |

Prompt processing is essentially unchanged, which matches the patch set's
own scope — it's targeted at decode, not prompt processing or
large-batch serving. The measured +27% decode improvement is well below
the patch repo's own headline number for this model (~+80%), which is
expected: their number used a different quant, MTP speculative decoding,
and a realistic sampler, none of which this benchmark reproduces — this
is a plain `llama-bench` pp/tg comparison, not a reproduction of their
full methodology. The relative improvement from the patches on identical
hardware and identical everything-else is the useful number.

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
| Baseline (stock `v0.4.0`) | 141.81 ± 0.33 t/s | 10.50 ± 0.00 t/s |
| Patched (29 patches) | 141.06 ± 1.40 t/s | **13.22 ± 0.00 t/s** |
| Δ | −0.5% (within noise) | **+25.9%** |

A real gain, and close to the MoE model's +27%, even though several of
the 29 patches are scoped to MoE routing or to the gated-delta-net block
(Qwen3.5/Qwen3-Next specific) and don't fire on a dense model. That
suggests the general `sm_60`/CUDA-kernel and host-side patches, which
apply regardless of architecture, carry most of the benefit.

Decode throughput itself is far lower on this model than on the MoE one
(13 vs. 70 t/s) — not a patch-related regression, just the architecture
difference: all 27B dense parameters activate on every token, versus
roughly 3B active parameters per token on the MoE model. Prompt
processing is flat in both cases, consistent with the patch set being
decode-focused.

## Batch size / concurrency scaling

`llama-batched-bench` against the patched build (Qwen3.6-35B-A3B, same
setup as its table above), sweeping the number of concurrent sequences
(`-npl`) from 1 to 32, `-npp 128 -ntg 128 -c 16384`:

| Concurrent sequences (B) | Decode t/s (TG) | Total t/s |
|---:|---:|---:|
| 1 | 69.09 | 49.95 |
| 4 | 152.56 | 224.86 |
| 8 | **174.64** | **278.29** |
| 16 | 112.97 | 193.02 |
| 32 | 125.99 | 214.67 |

Going from 1 to 8 concurrent sequences raises decode throughput about
2.5x (69 → 175 t/s). Throughput **peaks at 8 concurrent sequences and
regresses** at 16 and 32 (to ~113-126 t/s) rather than continuing to
climb. The cause isn't determined. PCIe link width doesn't explain it on
its own (link width has no measurable effect on single-stream decode; see
[GPU-SCALING.md](GPU-SCALING.md)), so the cost is more likely in how the
work is synchronized across the three cards once enough concurrent
requests are in flight, but that hasn't been tested.

Under the default layer-split mode (`-sm layer`) with no NVLink, only one
GPU is ever at high utilization at a time: each card processes its
assigned layers in turn as the token's activation passes through the
pipeline, so no point in the pipeline has all three cards computing at
once. The same is true of the single-stream runs, where each card is busy
well under half the time.

**Practical takeaway**: for this box, **8 concurrent requests looks like
the practical sweet spot** if it's ever run multi-user rather than
single-stream. This sweep was only run on three cards.

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

## MTP speculative decoding

llama.cpp supports self-speculative decoding via a model's own
multi-token-prediction (MTP) head: the model drafts several tokens ahead
in one pass, then verifies them all against the full model in the same
step. Unlike `--parallel`, this doesn't trade latency for throughput —
it's a straight win on a single request, since every draft token is
checked against the real model before being accepted (deterministic
sampling, no accuracy loss).

**The catch: the GGUF has to be MTP-converted, and a regular quant isn't
— even though it looks like it should already work.** Server logs for the
regular quant print lines like `model has unused tensor
blk.64.nextn.eh_proj.weight -- ignoring`, which looks like the draft head
is already present. Enabling it (`--spec-type draft-mtp
--spec-draft-n-max 3`) against that file fails outright:

```
common_speculative_init_result: creating MTP draft context against the target model '...'
llama_init_from_model: context type MTP requested but model doesn't contain MTP layers
```

Those `nextn` tensors are present in the regular quant but not in a form
llama.cpp's MTP code recognizes — they're leftover checkpoint artifacts,
not a usable draft head. The fix is unsloth's
`Qwen3.6-35B-A3B-MTP-GGUF` repo, which publishes the same quants
properly MTP-converted (~500MB larger, same weights otherwise — see
[SOFTWARE.md](SOFTWARE.md)). That loads cleanly with the same flags and
the MTP draft context actually initializes.

### Measured gain and tuning sweep

Live single-request test against `llama-server` (Qwen3.6-35B-A3B
Q4_K_XL, `-ts 1/1/1`, 16K context, one slot): `/completion`,
`n_predict: 512`, `temperature: 0` unless noted, prompt cache off, one
warm-up request then 4 measured requests per configuration.

| Config | Generation (t/s) | vs. no MTP | MTP draft acceptance |
|---|---:|---:|---:|
| No MTP | 66.5 (66.0–66.8) | — | — |
| `n-max=3, p-min=0.0` | 90.8 (90.7–90.8) | +36.5% | 67.5% |
| `n-max=5, p-min=0.0` | 68.3 (68.2–68.3) | +2.7% | 43.0% |
| **`n-max=3, p-min=0.75`** | **92.4** (92.0–92.7) | **+39.0%** | 93.9% |
| `n-max=4, p-min=0.75` | 90.3 (90.1–90.4) | +35.8% | 93.1% |
| `n-max=3, p-min=0.75`, realistic sampler (temp 0.7, top-p 0.8, top-k 20) | 87.0 (84.4–90.1) | +30.8% | 90.6% |

**Raising the acceptance threshold (`--spec-draft-p-min 0.75`) is the
real lever.** With the default threshold, drafting further ahead
(`n-max=5`) erased essentially all of the MTP gain: the later draft
tokens rarely match what the full model would produce (43% acceptance),
so the extra verify work is mostly wasted. With `p-min 0.75`, acceptance
rises to ~94% and the gain reaches +39%; widening the window further
(`n-max=4`) adds nothing.

**A realistic sampler costs about 6%.** Sampling with temperature 0.7,
top-p 0.8 and top-k 20 lowers draft acceptance (90.6% vs. 93.9%) and
generation speed (87.0 vs. 92.4 t/s), though MTP still gives +31% over no
MTP. None of this touches output correctness — speculative decoding
always verifies drafted tokens against the full model before accepting
them, so every configuration produces identical output for a given
prompt and seed; only throughput differs.

Production runs `--spec-draft-n-max 3 --spec-draft-p-min 0.75` (see
[SOFTWARE.md](SOFTWARE.md)).
