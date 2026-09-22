# Building and validating llama.cpp for Pascal

This covers getting llama.cpp actually running well on these three P100s:
the CUDA toolchain problems that stood in the way of a working build, the
Pascal-specific performance patches applied on top, and the tests run to
confirm each step actually worked rather than just compiled.

## The CUDA toolchain problem

The installed NVIDIA driver (580.178.04, from RPM Fusion's `580xx` legacy
branch — see [SOFTWARE.md](SOFTWARE.md)) reports CUDA 13.0 API support,
and the obvious move is to install the matching CUDA 13.x toolkit. That
doesn't work here:

**CUDA 13's `nvcc` has dropped the ability to compile for `sm_60`
(Pascal) entirely.** It's not a flag or a config issue — passing
`-DCMAKE_CUDA_ARCHITECTURES=60` to a CUDA 13.4 build fails outright with
`nvcc fatal : Unsupported gpu architecture 'compute_60'`. The 580xx
driver still *runs* Pascal cards fine; it's specifically the compiler
that has moved on.

The fix: install a second, older CUDA toolkit **alongside** the CUDA 13.x
one, and point the build at it. NVIDIA's Fedora 44 repo only ships CUDA
13.x, but their Fedora 41 repo still carries CUDA 12.9, which does
support `sm_60`. Installing just the compiler components needed
(`cuda-nvcc-12-9`, `cuda-cudart-devel-12-9`, `cuda-driver-devel-12-9`,
`cuda-nvrtc-devel-12-9`, `cuda-cccl-12-9`, `libcublas-devel-12-9`,
`cuda-crt-12-9`) from that repo, kept disabled by default and enabled
only for that one install, coexists cleanly at `/usr/local/cuda-12.9`
next to the CUDA 13.4 install.

That surfaced a second problem: CUDA 12.9's `nvcc` refuses Fedora 44's
system compiler (gcc 16) outright — `gcc versions later than 14 are not
supported`. Adding `-allow-unsupported-compiler` doesn't actually fix
this; it's not just a version-check gate, gcc 16's `<type_traits>` header
uses C++ standard library constructs `nvcc`'s frontend can't parse at
all, so forcing past the check just produces a wall of real compile
errors instead.

The fix: an **isolated gcc 12 toolchain via micromamba/conda-forge**
(`gxx_linux-64=12`), used only as `CMAKE_CUDA_HOST_COMPILER` for the CUDA
compilation step. The rest of the build (plain C++ files) still compiles
with the system gcc 16 — only `nvcc`'s own host-compiler calls go through
the isolated gcc 12. This is a well-known pattern for exactly this
nvcc/host-compiler version mismatch, and avoids touching the system
compiler at all.

Working configure line (baseline, unpatched build):

```
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=60 \
  -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA_F16=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.9/bin/nvcc \
  -DCUDAToolkit_ROOT=/usr/local/cuda-12.9 \
  -DCMAKE_CUDA_HOST_COMPILER=<path-to-conda-env>/bin/x86_64-conda-linux-gnu-g++ \
  -G Ninja
```

**Test**: built `llama-cli`, confirmed via `ldd` that it links against
`libcuda.so.1` (the driver) and the CUDA 12 runtime/cuBLAS libraries, then
ran a real generation against a small GGUF model while watching
`nvidia-smi` — GPU utilization rose to 26-33% across all three cards
during generation with correct output, confirming this isn't silently
falling back to CPU.

## Pascal-specific performance patches

[`shinbunbun/llama-cpp-p100-patches`](https://github.com/shinbunbun/llama-cpp-p100-patches)
ships 29 patches specifically measured on a P100 (GP100), exploiting the
card's lack of the DP4A instruction in favor of its full-rate FP16 FMA
path. Despite the repo being Nix-flake-based, the patches themselves are
plain `.patch` files under `patches/`, generated against upstream tag
`v0.4.0` at zero fuzz — no Nix required to use them.

```
git clone --branch v0.4.0 --depth 1 https://github.com/ggml-org/llama.cpp llama.cpp-patched
cd llama.cpp-patched
for p in ../llama-cpp-p100-patches/patches/*.patch; do
  patch -p1 -F0 < "$p" || { echo "FAILED: $p"; break; }
done
```

All 29 applied cleanly against a pristine `v0.4.0` checkout, in numeric
order, with zero fuzz. Patches 19 and 20 (`fuse-pre-add-rms-norm` /
`fuse-add-unary-mul`) are left at their upstream default of **off** —
the patch set's own docs flag these as producing non-deterministic
decode output specifically on MoE models, which is exactly the model
class this box runs. Not worth the small throughput gain for that
trade-off.

A baseline (unpatched) and patched tree were built from the **same**
`v0.4.0` tag, same CUDA/gcc setup as above, for a clean A/B — see
Benchmark below.

## `llama-bench` tensor-split gotcha

Worth flagging since it produces a confusing failure: `llama-cli` and
`llama-server` accept `-ts` (tensor-split) as **comma**-separated values
(`-ts 1,1,1`), but `llama-bench` expects **slash**-separated values
(`-ts 1/1/1`). Passing the comma form to `llama-bench` doesn't error —
it silently fails to parse, defaults to putting every layer on device 0,
and then dies with `cudaMalloc failed: out of memory` trying to fit the
whole model on a single 16GB card. Nothing in the error message points
back at the tensor-split flag; it took a verbose (`-v`) run to see the
actual `layer N assigned to device CUDA0` lines for every layer before
that was obvious.

## Benchmark

Model: Qwen3.6-35B-A3B, Q4_K_XL quant, tensor-split evenly across all
three P100s (`-ts 1/1/1`, `-ngl 99`, flash attention on). 5 repetitions
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
