# Power Macintosh Yosemite / ATI Rage 128 — emulation reference notes

This is AI slop that was shit out by Claude Opus 5.5 as a byproduct of generating
[the yosemite-slop branch](https://github.com/wurthless-elektroniks/mame/tree/yosemite-slop),
which emulates the Power Macintosh G3 Yosemite (Blue and White) to an extent where it can boot Mac OS 9 and
play 3D-accelerated games like Nanosaur and Unreal at almost-playable speeds on my M4 Mac Mini.

The yosemite-slop branch can't be submitted to upstream MAME because it violates all sorts
of licensing agreements, and it sucks. However the emulation notes might be useful for the
emulation development community at large (hi MAME and DingusPPC devs).

Claude analyzed the following to generate this:

- ATI's RAGE 128 VR/GL Register Reference Manual (RRG-G04100-C)
- 86Box's Rage 128 emulation
- The X.Org r128, Linux aty128fb, Linux r128 DRM and Mesa r128 drivers
- ATI's "Rage 128 driver 1.02" CD for Windows 9x/NT4/2000/XP, including
  the register name table in its OpenGL driver
- Mac OS 9 drivers: the card ROM ndrv, ATI Driver Update, ATI Graphics
  Accelerator and the ATI Rage 128 3D Accelerator RAVE engine
- Mac OS X 10.4 ATIRage128 drivers (1.4.4): kext, GL, 2D and DVD
- Firmware dumps, an ioreg dump and Open Firmware output from my G4 Yikes
  and its OEM Rage 128 GL Mac edition video card
- Side-by-side comparisons with real hardware (Unreal and Nanosaur under
  Mac OS 9 RAVE)

I'll have Claude update these notes as more changes are pushed to the yosemite-slop branch.
There's still a lot of open questions and a lot hasn't been checked. I'm still focused on Classic
Mac OS as the target, Mac OS X will come later. And no, I'm not going to bother with G4 support,
Altivec would require a million extensions to MAME's UML recompiler if it's to run efficiently at all.
Arbee is probably pulling his hair out already about this low effort garbage and I'm not going to make
his life any worse.

Anyway, without further ado, I'll let the extremely thirsty unemployment generating
plagiarism machine take it from here. Have fun, everyone!

## Introduction

These are notes on the ATI Rage 128 (R128) family, mainly the Apple OEM
Rage 128 GL that shipped in the Power Mac G3 (Blue & White) and the Power Mac
G4 (PCI Graphics, "Yikes!"). They come from writing a MAME device for the chip.

They are meant as a hardware reference that can be read **without** the MAME
source. That code was largely machine-generated and is not a good model to copy.
Where the notes describe what that emulation does, they say so. Everything else
is a statement about the hardware, tagged with how much we trust it.

## Source tags

Every non-obvious fact carries one or more of these tags:

| Tag | Meaning |
|-----|---------|
| **[M]** | ATI *RAGE 128 VR/GL Register Reference Manual*, P/N RRG-G04100-C rev 0.02, Jan 1999 (eng. doc rev 2.238, 300 pp). Official, but it covers config/PLL/pixel cache, PCI/AGP, VGA, CRTC/DAC/memory controller, the 2D engine and misc. **The 3D/scaler, CCE and video/DVD chapters are missing.** So it has no PM4 packet formats, no CCE (0x1000–0x13FF) registers and few 3D registers. |
| **[D]** | Seen in a shipping driver: disassembly of ATI's Mac OS 9 / Mac OS X / Windows drivers, or the X.Org r128, Linux aty128fb and r128 DRM sources. "Drivers do this" does not mean "hardware requires this". |
| **[8]** | Taken from 86Box's Rage 128 implementation (`vid_ati_rage128*.c`), which was checked against games on real cards. |
| **[H]** | Compared against a real card (rendering compared side by side on real hardware). |
| **[A]** | Assumption made so the drivers work in emulation. Plausible, unverified. Treat as a hypothesis. |

Where two sources disagree, both are given and the disagreement is listed in
[open-questions.md](open-questions.md).

## Files

| File | Contents |
|------|----------|
| [pci-and-apertures.md](pci-and-apertures.md) | Variants and IDs, PCI config and capabilities, BARs, register map regions, linear/register apertures, endian swappers, MM_INDEX, SURFACE windows and the tile layout, VGA banking |
| [display.md](display.md) | CRTC timing, pixel formats, flips, PLL, DAC/palette, hardware cursor, overlay scaler, interrupts, DDC / GPIO_MONID / I2C engine |
| [2d-engine.md](2d-engine.md) | GUI master control, datatypes, brushes, ROP3, scissors, colour compare, rectangles, blits, host data, Bresenham lines, the scaler registers, GUI DMA |
| [cce-pm4.md](cce-pm4.md) | The CCE command processor: modes, FIFO, ring and indirect buffers, bus mastering through AGP / PCI GART, packet types 0–3, every Type 3 opcode seen and its payload |
| [3d-engine.md](3d-engine.md) | 3D context registers, vertex formats, primitive walking, setup (shading, culling, perspective), texture units, formats and mip layout, filtering, combiners, fog, alpha test, Z/stencil, blending, keys, stipple |
| [mac-drivers.md](mac-drivers.md) | How Mac OS 9 (ROM ndrv, ATI Graphics Accelerator, RAVE engine) and Mac OS X 10.4 drive the card, and the Apple OEM option ROM |
| [yosemite-board.md](yosemite-board.md) | The Blue & White G3 / Yikes! G4 board as it bears on the card: PCI topology, IRQ routing, slot details, emulation pitfalls |
| [open-questions.md](open-questions.md) | Contradictions between sources, unverified assumptions, and a list of tests to run on a real card |

## The chip in one paragraph

The Rage 128 drops the mach64/Rage Pro register layout for a new one, which the
original Radeon later inherited. So Radeon documentation and drivers are often
a useful second opinion, but check before trusting them. It has:

- a standard VGA core;
- a native CRTC with a 64×64 cursor and an overlay scaler;
- a 2D "GUI" engine driven by memory-mapped registers;
- a setup engine and pixel pipeline for 3D with two texture units;
- the **CCE** (Concurrent Command Engine), a microcoded command processor
  that reads "PM4" packets. It takes them from a PIO FIFO or by bus-mastering
  ring and indirect buffers from system memory through AGP or a PCI GART.

All 3D drawing goes through the CCE. In practice so does most accelerated 2D
on Mac OS. Registers in the 0x1400–0x1FFF "GUI" range are FIFOed and
dword-only **[M]**.

## Chip family codes

ATI's own drivers use internal family numbers that don't appear in the PCI IDs.
The Mac OS 9 RAVE engine gets them from the resource manager
(`ARM_GetVRAMInfo`, field 52). It has seen 0x500, 0x510, 0x512, 0x520,
0x550, 0x560 and 0x570, and refuses to run on anything below 0x500 **[D]**.
Which family is which product isn't confirmed. Several driver workarounds key
off these codes (see [mac-drivers.md](mac-drivers.md)).

## Provenance

The source for these notes:

- the `yosemite-slop` branch of `wurthless-elektroniks/mame`
  (`src/devices/bus/pci/rage128.cpp`, `rage128.h`,
  `src/mame/apple/macyosemite.cpp`);
- notes kept while disassembling the Mac OS 9 "ATI Rage 128 3D Accelerator"
  RAVE extension (PEF, built 2002-09-03);
- the register manual cited above;
- testing Unreal (RAVE) and Nanosaur under Mac OS 9 against a real card.

The emulation boots Mac OS 9 to the desktop with 2D and RAVE 3D acceleration,
on both the B&W G3 driver and MAME's Beige G3 (through a PCI slot patch). It
also handles a lot of behaviour seen from the Mac OS X 10.4 ATIRage128 kext,
GL, 2D and DVD drivers (1.4.4). Those notes are marked [D], since how well
OS X runs wasn't the focus.
