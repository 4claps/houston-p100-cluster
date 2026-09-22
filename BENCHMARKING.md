# Benchmarking

Benchmark results for this box, kept separate from
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) (which
covers the build/patch process itself) so new runs have an obvious place
to land without that doc growing without bound.

All runs below use the patched build described in
[LLAMA-CPP-P100-ENHANCEMENTS.md](LLAMA-CPP-P100-ENHANCEMENTS.md) unless
otherwise noted, with tensor-split evenly across all three P100s.

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
