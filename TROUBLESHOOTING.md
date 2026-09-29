# Troubleshooting

Known issues and workarounds for running llama.cpp on the Houston 3x Tesla P100 rig.

## `-sm tensor` crashes on second request (GGML_ASSERT(bcj.nodes[i]))

**TL;DR:** `--split-mode tensor` loads the model and successfully serves one request, then crashes on the *next* request with `GGML_ASSERT(bcj.nodes[i]) failed` inside `ggml-backend-meta.cpp`. Reproduces with 2 or 3 P100s, regardless of context size, MTP, or prompt-cache settings. `--split-mode layer` does not hit this and is the current stable config for this build.

Tracked upstream: [ggml-org/llama.cpp#29466](https://github.com/ggml-org/llama.cpp/issues/29466)

**Workaround:** run with `-sm layer` instead of `-sm tensor` until this is fixed upstream.

## GPU falls off the bus (Xid 79 and Xid 154)

**TL;DR:** one P100 repeatedly dropped off the PCIe bus about 7 minutes after boot. The
cause was the riser it was plugged into, not the card, the power limit or the power
supply. Removing the riser and moving the card to a direct slot fixed it (2026-09-29).

**Symptoms:** the kernel log shows `Xid 79` ("GPU has fallen off the bus"), followed by
`Xid 154` (GPU reset required). The card disappears from `nvidia-smi`. Once it also caused
a kernel panic in the `nvidia` module ("Fatal exception in interrupt").

**What it was:** the affected card (UUID ending `c6af34c825cc`) was on a riser at PCI
`02:00.0`, on root port `00:02.2`. It failed at roughly the same point after every boot,
and the following did not help:

- Running with and without the 125 W power limit, so it is not a power-limit problem.
- Giving the riser its own PSU power.
- `pcie_aspm=off` on the kernel command line. This has no effect on P100s, which report
  "ASPM not supported" (see [SOFTWARE.md](SOFTWARE.md#kernel-arguments)).

**Fix:** the riser was removed and the card moved to a direct slot. It has been stable
under load since. The GT 610 that had been in the x16 slot was removed at the same
time; it was only used to test slot link width and the 580 driver ignored it. The three
cards are now at `01:00.0`, `03:00.0` and `05:00.0`, each at PCIe Gen 3 x8. Bus IDs shift
when cards move, so identify cards with `nvidia-smi -L` (index and UUID), not by PCI
address.

**If a card drops with Xid 79:**

1. Do a **full power cycle**: shut down and switch the PSU off for 30 seconds. A reboot
   alone may not recover the card.
2. If it repeats on the same slot, suspect the **riser, cable or slot** before the card.
3. Run a **swap test**: move the card to another slot, or swap it with a known-good
   card. If the problem follows the card, the card is bad; if it stays with the slot,
   it's the slot, riser or cable.
