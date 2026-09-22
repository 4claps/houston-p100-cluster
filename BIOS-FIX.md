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
