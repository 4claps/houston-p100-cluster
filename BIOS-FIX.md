# The Above 4G Decoding fix

This is the story of the single most time-consuming part of this build:
getting the motherboard to correctly map three 16GB GPUs into address
space it was never designed to hand out.

## The problem

Three Tesla P100s means three 16GB VRAM BARs (PCI Base Address Registers)
that need to be memory-mapped, on top of everything else the system
already has mapped. 48GB alone is well past what 32-bit PCI address space
(4GB) can hold, so each card's BAR has to be mapped as 64-bit and placed
above the 4GB boundary. The BIOS setting that controls this is normally
called **Above 4G Decoding**.

It didn't appear anywhere in this board's stock BIOS menu. Not disabled —
just absent.

## Root cause

Research turned up a known issue on X79-era boards: an **MMIOH (Memory
Mapped I/O High) address space limitation bug**. On some ASUS BIOS
revisions for this chipset, naively enabling Above 4G Decoding without
first fixing the underlying MMIOH limit produces a system that POSTs fine
and then goes to a black screen — no display, no way back in except a
CMOS clear. The working theory is that ASUS responded to this by simply
hiding the option on affected revisions rather than shipping something
that could brick a board's usability for an average user.

That meant two separate problems to solve: fix the actual MMIOH bug in
the firmware, and separately make the (now safe) option visible again in
the setup menu.

## Fixing the MMIOH bug

### 1. Dump the live BIOS

Before touching anything, the running BIOS was read directly off the SPI
flash chip with `flashrom`, rather than trusting whatever ASUS's official
`.CAP` download actually matches what's currently on this specific board:

```
flashrom -p internal -c "W25Q64BV/W25Q64CV/W25Q64FV" -r bios-dump.rom
```

Fedora's kernel hardens `/dev/mem` by default in a way that blocks this
kind of direct flash access. Getting `flashrom` to work at all required
booting with `iomem=relaxed` on the kernel command line.

The dump was read twice, independently, and diffed against itself to
confirm it was clean and not corrupted by a flaky read — this is a
one-shot operation on live production firmware, so a bad read silently
treated as good would have been a bad time.

### 2. Patch with UEFIPatch

The actual fix came from the [xCuri0/ReBarUEFI](https://github.com/xCuri0/ReBarUEFI)
project, which ships a UEFIPatch patch specifically named **"Extend MMIOH
limit to fix Above 4G Decoding (X79)"**.

This patch was applied against the **official ASUS 4801 `.CAP`** file
downloaded from ASUS's support site — not the raw `flashrom` dump. The
raw chip dump is missing the ~2KB ASUS capsule header that USB BIOS
Flashback requires to recognize a file as a valid firmware image. This
was discovered the hard way on the first attempt, when a patched version
of the raw dump was silently rejected by Flashback.

### 3. Extract and transplant modules

The patch touches specific firmware modules, not the whole image, so the
actual transplant was done by hand:

1. **UEFITool 0.28.0** was used to open both the patched image and the
   original 4801 CAP, and extract as `.ffs` files:
   - `IvtQpiandMrcInit` — found in *two* separate firmware volumes in this
     image, both needed extracting
   - `Runtime` — contains a secondary `CpuIo2` patch that's also required
     for the fix to fully take effect
2. **MMTool 4.50.0.23** was then used to replace those same modules inside
   a **fresh, unmodified copy** of the official 4801 CAP.
3. One more module had to be preserved by hand: the board's own identity
   module (GUID `FD44820B...`), which holds this specific board's MAC
   address and serial number data. This was pulled from the original
   `flashrom` dump (not the generic ASUS download) and inserted into the
   rebuilt CAP, so the flashed board keeps its real identity rather than
   picking up placeholder values from ASUS's generic release image.

### 4. Flash

The rebuilt `P9X79PRO.CAP` was flashed via **USB BIOS Flashback** — no
OS or CPU required for this method, which matters since a bad flash at
this stage would otherwise mean a dead board. This succeeded cleanly on
the first attempt, with no boot issues afterward.

## Making the option visible

Flashing the patched firmware fixed the underlying MMIOH bug, but **Above
4G Decoding still didn't show up in the BIOS menu.** This turned out to
be a separate problem from the one above — the option not working and the
option being hidden from the menu were two independent things, both
inherited from how ASUS built this BIOS revision.

### 5. Find the hidden option's NVRAM location

**IRFExtractor-RS** was run against the BIOS's Setup form data to locate
exactly where this now-functional-but-invisible option lives in NVRAM. It
resolved to variable `Setup`, **offset `0x1`**.

### 6. Write it directly

With the exact offset known, the option could be enabled without ever
needing it to appear in a menu, using **`grub-mod-setup_var`**:

1. `modGRUBShell.efi` was renamed to `EFI/Boot/bootx64.efi` and put on a
   bootable USB drive.
2. Secure Boot was temporarily disabled (required to boot an unsigned
   custom EFI binary).
3. Booted into the modded GRUB shell and ran:

   ```
   setup_var 0x1 0x1
   ```

4. The value at that offset was read back **both before and after** the
   write, to confirm the write actually landed rather than trusting the
   command's own exit status.

## Verification

The real test wasn't "does the box boot" — it was whether the GPUs' BARs
were actually mapped as 64-bit, above the 4GB line, at their full size:

```
lspci -vvv | grep -A20 P100
```

All three cards show:

```
Region 1: Memory at ... (64-bit, prefetchable) [size=16G]
```

on their own distinct 64-bit address ranges — confirming Above 4G
Decoding is genuinely active and each 16GB BAR is fully and correctly
mapped, not just that a checkbox got flipped somewhere.

---

# Adding PCIe bifurcation support

A second, independent mod layered on top of the fix above: unhiding the
BIOS menus that expose PCIe bifurcation, so the top x16 slot could be
split x8x8 to run two GPUs off a single slot through a bifurcation riser.

## The problem

Like Above 4G Decoding, bifurcation control (`IOU1/IOU2/IOU3 - PCIe Port`)
existed in the silicon and in the compiled BIOS forms, but the menu path
to reach it — `Advanced > System Agent Configuration > IOH Configuration`
— was hidden. Unlike Above 4G Decoding, this wasn't a functional bug being
masked; the underlying feature already worked, it just had no UI.

## Finding the real guard bytes

A [community guide](https://winraid.level1techs.com/t/guide-adding-bifurcation-support-to-asus-x79-uefi-bios/33018)
for adding bifurcation support to ASUS X79 boards describes the technique
in general — find the `SuppressIf` block guarding the hidden menu entry
and flip it from `True` to `False` — but its example byte patterns were
recorded against a different board's BIOS revision (Sabertooth X79,
BIOS 4701) and didn't literally exist in this image; the QuestionId/
VarStoreId/FormId numbers baked into those opcodes are assigned per
compile, not fixed by the silicon.

So instead of hex-searching for the guide's literal bytes, the Setup
module's HII form data was decompiled to text with **IFRExtractor-RS**,
which is what actually located the real constructs in this image:

```
SuppressIf  { 0A 82 }
    True  { 46 02 }
    Ref Prompt: "PCI Subsystem Settings" ... FormId: 0x40E { 0F 0F AA 0A B0 0A 01 00 00 00 FF FF 00 0E 04 }
End

SuppressIf  { 0A 82 }
    True  { 46 02 }
    Ref Prompt: "System Agent Configuration" ... FormId: 0x41D { 0F 0F 38 08 39 08 04 00 00 00 FF FF 00 1D 04 }
End
```

Two "System Agent Configuration" entries exist in this image: FormId
`0x41C`, already visible, is a stripped-down copy with only PCIe
link-speed and VT-d options; FormId `0x41D`, the one guarded above, is
the full version and the only one containing a `Ref` to `IOH
Configuration` — which is itself unguarded, so unhiding `0x41D` is
sufficient to reach the bifurcation menu.

## The patch

Both guards are the same 2-byte opcode (`True { 46 02 }` — always
suppress). Changing the single opcode byte `46` to `47` (`False` — never
suppress) at each location was enough; nothing else in the image,
including any IOU option or default, was touched:

- Setup module: File GUID `899407D7-99FE-43D8-9A21-79EC328CAC21` (DXE
  driver, `Text: Setup`) → its Tiano-compressed Freeform-subtype-GUID
  section, subtype GUID `97E409E6-4CC1-11D9-81F6-000000000000` — this
  holds the full HII form/string data for the whole setup menu tree.
- Within that section's decompressed body (859,627 bytes): byte `0xC7B86`
  (guards "PCI Subsystem Settings") and byte `0xC7BC9` (guards the full
  "System Agent Configuration"), both `46` → `47`.

The rebuild used **UEFIReplace 0.28.0** to swap that section's body
directly, letting it handle the Tiano decompress/recompress and FFS/FV
checksum bookkeeping:

```
UEFIReplace WORKING_p9x79pro.cap 899407D7-99FE-43D8-9A21-79EC328CAC21 18 \
  setup_hii_patched.bin -o P9X79PRO_bifurcation.cap
```

This produced a `.cap` file the exact same size as the original
(8,390,656 bytes) — the guide's own procedure is to remove and reinsert
the section by hand precisely to handle cases where the size changes;
here it didn't, and re-extracting and diffing all 197 firmware files by
GUID between the original and patched images confirmed only the Setup
file differed, and only in the expected way (both guards now read
`SuppressIf False`, every IOU option/default byte-identical to stock).

Flashed the same way as the Above 4G Decoding fix: **USB BIOS Flashback**,
file renamed to `P9X79PRO.CAP`.

## Verification

New menu path: `Advanced > System Agent Configuration > IOH Configuration
> IOUn - PCIe Port`. `IOU1` only offers `x4x4`/`x8` (8-lane budget); `IOU2`
and `IOU3` both offer `x4x4x4x4`/`x4x4x8`/`x8x4x4`/`x8x8`/`x16`, defaulting
to `x16`.

Which IOU drives which physical slot isn't stated anywhere in the BIOS
text — it's fixed by this board's trace routing. **IOU2 is confirmed to
be the top slot** on this specific P9X79 PRO: after setting IOU2 to x8x8
in the new menu,

```
lspci -tv
lspci -s 00:02.0 -vvv | grep -E 'LnkCap|LnkSta'
lspci -s 00:02.2 -vvv | grep -E 'LnkCap|LnkSta'
```

showed the IOU2 root port pair (Intel "Root Port 2a"/"2c", PCI
`00:02.0`/`00:02.2`) split into two independent PCIe Gen3 x8 links — one
half (`01:00.0`) already carrying a live Tesla P100 at negotiated Gen3 x8,
the other half (port 2c, bus 02) trained and ready but still empty,
awaiting the next GPU to be added to the riser's second slot.

(IOU3's root ports, `00:03.0`/`00:03.2`, also show bifurcated x8x8 in this
image, both halves already populated by the other two original P100s —
pre-existing from before this mod, unrelated to the top slot.)

## Rollback

Same as the Above 4G Decoding fix: re-flash the known-good `.cap` via USB
BIOS Flashback, or clear CMOS. A CMOS clear resets IOU1/2/3 back to their
compiled defaults (x8/x16/x16) but does **not** re-hide the menu — that's
an image-level change, not an NVRAM variable — so the x8x8 selection would
just need to be re-set afterward.

**Caveat:** this mod does not survive a stock ASUS BIOS update; it would
need to be redone from scratch against whatever new base image ASUS
ships.
