# Hardware

## Host

- **Motherboard**: ASUS P9X79 PRO (X79 chipset, LGA2011)
- **CPU**: Intel Core i7-3930K (Sandy Bridge-E, 6c/12t, 3.2GHz base). Kept
  rather than upgrading to a Xeon E5 v2 (the natural drop-in upgrade for
  this socket) — the workload here is entirely GPU-bound, so the extra
  cores and IPC a Xeon swap would have bought weren't worth the cost and
  effort.
- **RAM**: 16GB DDR3-1600 (4x4GB), no ECC.

## GPUs

- **3x NVIDIA Tesla P100 PCIe 16GB** (Pascal, GP100GL), 48GB total VRAM.
  This is three separate 16GB pools, not a unified 48GB space — there's no
  NVLink on the PCIe variant of the P100, so cross-GPU tensor splits go
  over PCIe.

## Power

- **Corsair RM1000x 1000W 80+ Gold**, fully modular (refurbished unit).
- **3x PCIe-8-pin-to-EPS adapter cables.** Each P100 draws its power
  through an 8-pin EPS connector, not the PCIe 8-pin connector its
  physical socket resembles. A single PCIe 8-pin lead is only rated for
  150W continuous — well short of the P100's 250W board power — so each
  card is fed by *two* PCIe 8-pin cables from the PSU, combined into one
  EPS 8-pin at the card end. Getting this wrong (a single PCIe 8-pin
  straight into the EPS socket) is a real fire risk under sustained
  inference load, not just an out-of-spec curiosity.

## Case

- **NZXT Source 530 (CA-SO530-M1)**, full tower. Needed for the physical
  length of three double-wide, full-length datacenter cards plus airflow
  clearance.

## Cooling

The P100 PCIe is a passive, blower-less design — it expects rack-mounted
front-to-back airflow that a tower case simply doesn't provide on its own.
Cooling is retrofitted with:

- **3D-printed fan shrouds**, custom-designed for a 120mm fan across all
  three GPUs, printed in PETG rather than PLA
  specifically for the heat tolerance — these shrouds sit directly against
  a card's exhaust-side heatsink fins and get genuinely warm in operation.
- **Arctic P12 Pro 120mm fans**, chosen for their high static pressure
  (6.9 mmH2O) rather than raw airflow (CFM) — the P100's heatsink fin
  stack is dense enough that a high-airflow/low-pressure fan would mostly
  just move air around the shroud rather than through the fins.
- These shroud fans are driven by GPU temperature via a custom systemd
  service, not by the motherboard's own default fan curve — see
  [SOFTWARE.md](SOFTWARE.md) for how that's wired up.

## Display

None of the P100s have a video output — they're compute-only Tesla cards
— and all three of the board's double-wide slots are occupied by the
GPUs. A spare/old PCIe x1 card was used for display output during initial
setup and BIOS work. (Note: as of this writing, no display adapter shows
up in `lspci` — the box now runs fully headless over SSH, so it's possible
this card was pulled out again once console access was no longer needed.
Worth physically checking if a monitor is ever needed here again.)
