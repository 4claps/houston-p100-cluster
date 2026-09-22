# houston-p100-cluster

Documentation for `houston`, a 3x NVIDIA Tesla P100 16GB local
LLM inference server. It's a repurposed old NAS build, given a second life
as a dedicated inference box once the hardware had 48GB of usable VRAM
crammed into it.

This repo is documentation only — no Ansible, no provisioning scripts. It
exists so that the non-obvious parts of this build (particularly a BIOS
modification that isn't something you'd want to reverse-engineer twice)
are written down somewhere durable.

## Contents

- [HARDWARE.md](HARDWARE.md) — full parts list and the reasoning behind
  each choice
- [BIOS-FIX.md](BIOS-FIX.md) — the Above 4G Decoding fix: an X79-era MMIOH
  bug that hid the setting entirely, and the process used to patch and
  expose it
- [SOFTWARE.md](SOFTWARE.md) — OS, driver, and fan control
- [LLAMA-CPP.md](LLAMA-CPP.md) — building llama.cpp for Pascal, the P100
  performance patches, and the baseline-vs-patched benchmark
