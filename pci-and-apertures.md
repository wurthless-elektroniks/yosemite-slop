# PCI, apertures and memory layout

## Variants and PCI IDs

Vendor 0x1002 (ATI). The device ID is two ASCII letters, as on other ATI parts.
Class 0x030000 (VGA).

| ID | Letters | Part | Bus | Notes |
|----|---------|------|-----|-------|
| 0x5245 | RE | Rage 128 GL | PCI | 312+16 BGA **[M]** |
| 0x5246 | RF | Rage 128 GL | AGP 1x/2x | 312+16 BGA **[M]** |
| 0x524B | RK | Rage 128 VR | PCI | 256+16 BGA **[M]** |
| 0x524C | RL | Rage 128 VR | AGP | 256+16 BGA **[M]** |
| 0x5044 | PD | Rage 128 Pro | PCI | |
| 0x5046 | PF | Rage 128 Pro | AGP 4x | |

ATI's Windows 2000 miniport knows 43 IDs in four families **[D]**:

- Rage 128: RE/RF, RK/RL/RM
- Rage 128 4X: SE–SH, SK–SN
- Rage 128 Pro: PA–PX, cycling PCI / AGP 2x / AGP 4x
- Rage 128 Pro II/Ultra: TF, TL, TR–TU

The Mac OS X 10.4 accelerator matches only RE, RF, RK, RL, PE, PF, PR, TR and
the Mobility LE/LF. So it won't load on a PD **[D]**.

**Apple OEM Rage 128 GL** (`ATY,Rage128y`), from a Power Mac G4 (PCI Graphics)
ioreg dump **[D]**:

- 1002:5245, revision **0x00**;
- config dword 0x2C (subsystem) = 0xB530_0408;
- 16 MiB SDRAM.

ATI's Win9x driver only knows Rage 128 GL/VR revisions 2 and 3, so revision 0 on
the Apple board is worth noting **[D]**.

The emulation gives the VR 8 MiB, the GL 16 MiB and the Pro 32 MiB as
defaults **[A]**. CONFIG_MEMSIZE allows up to 32 MiB of VRAM **[M]**.

## Configuration space

| Offset | Contents |
|--------|----------|
| 0x04 | Status after reset: capabilities list, medium DEVSEL. Open Firmware reports the PCI GL as fast-back-to-back capable but **not** 66 MHz capable (status 0x0290) **[D]**. AGP parts: 0x0230 **[A]** |
| 0x2C | Subsystem IDs, set by writing ADAPTER_ID_W at cfg 0x4C **[M]** |
| 0x34 | Capabilities pointer: 0x50 on AGP parts, 0x5C on PCI parts **[M]** (the PCI value is **[A]**) |
| 0x3C | Interrupt pin A. The emulation uses MIN_GNT 0x08, MAX_LAT 0x00 **[A]** |
| 0x50 | AGP capability: ID 2, next 0x5C, revision 1.0 **[M]** |
| 0x54 | AGP status: RQ depth 0x1F (31:24), SBA (bit 9), rates 1x/2x (0x3), or 1x/2x/4x (0x7) on the Pro **[A]** |
| 0x58 | AGP command, writable bits 0x1F00_0307 **[A]** |
| 0x5C | Power management capability **[M]**: the manual says D0/D1/D3 and **no D2**. The emulation reports PMC 0x0602 (PM 1.1, D1 and D2), which contradicts that (see open questions) |
| 0x60 | PMCSR, power state in bits 1:0 |

The register aperture holds a **read-only image of config space at MMIO
0x0F00–0x0FFF** **[M]**. Mac OS X sets bus-master enable by writing the
command register through MMIO 0x0F04 **[D]**. So at least the command register
must be writable there, despite "read-only".

## BARs

| BAR | Size | Contents |
|-----|------|----------|
| BAR0 `MEM_BASE` | 64 MiB, prefetchable (bits 31:26) **[M]** | Linear aperture 0 at +0, aperture 1 at +32 MiB. Both map the same VRAM with separate byte swappers. `CONFIG_APER_SIZE` (0x108) reads 32 MiB. `CONFIG_APER_1_BASE` bit 25 is hardwired to 1 **[M]** |
| BAR1 `IO_BASE` | 256 bytes **[M]** | I/O window onto MMIO 0x00–0xFF, including MM_INDEX/MM_DATA, so every register is reachable through I/O |
| BAR2 `REG_BASE` | **16 KiB** (bits 31:14) per the manual **[M]** | Register aperture 0 at +0, register aperture 1 at +0x2000. `CONFIG_REG_APER_SIZE` (0x110) reads 0x2000. `CONFIG_REG_1_BASE` (0x10C) bit 13 is hardwired to 1 **[M]** |
| ROM | 128 KiB (`BIOS_ROM` bits 31:17) **[M]** | x86 VGA BIOS on PC boards. Open Firmware FCode with the Mac OS ndrv on Mac boards |

> **BAR2 is unresolved.** The manual says 16 KiB holding two 8 KiB register
> apertures. But the Mac OS 9 driver on a Beige G3 writes PIO FIFO packets to
> BAR2+0x5000 and host data to BAR2+0x57C0–0x57DC. Those are the CCE FIFO
> (0x1000) and HOST_DATA0 (0x17C0) plus **0x4000**. Its writes there are
> byte-reversed, like the rest of its register writes. On the Beige G3 the
> menu bar and window contents were missing until the emulation decoded BAR2
> as 32 KiB, with the upper 16 KiB repeating the registers (no swap) **[A]**.
> The real card's BAR2 sizing needs checking (see open questions).

## Register map regions **[M]**

| Range | Contents |
|-------|----------|
| 0x0000–0x00FF | Non-GUI registers, also in I/O space (BAR1) |
| 0x0100–0x0EFF | Non-GUI registers, MMIO only |
| 0x0F00–0x0FFF | PCI config image (read-only per the manual, see above) |
| 0x1000–0x13FF | CCE (the PIO FIFO window starts at 0x1000) |
| 0x1400–0x1FFF | GUI registers: FIFOed, **dword access only** |

Display and config registers allow 8, 16 and 32-bit access **[M]**. One Mac
OS 9 ndrv does an **unaligned** 32-bit read to save and restore one byte of
CRTC_GEN_CNTL **[D]**. See [display.md](display.md).

In the emulation, MMIO 0x3B0–0x3DF also reaches the VGA I/O registers, as on
later ATI parts **[A]**.

### Indirect access

- `MM_INDEX` 0x00 / `MM_DATA` 0x04 **[M]**. With `MM_INDEX` bit 31 (MM_APER)
  clear, MM_DATA reaches the register at that offset. With it set, MM_DATA
  reaches linear aperture 0 at that offset. Accesses through MM_DATA are
  **not** subject to the SURFACE windows (below) **[A]**.
- `CLOCK_CNTL_INDEX` 0x08 / `CLOCK_CNTL_DATA` 0x0C: PLL registers, byte-wide.
  The index is in bits 4:0, PLL_WR_EN is bit 7, and PPLL_DIV_SEL is bits 9:8
  **[M]**. Later parts have PLLs above 0x1F (P2PLL at 0x2A–0x2C) **[D]**, so
  the emulation decodes 6 index bits.

## Byte order: CONFIG_CNTL (0xE0) **[M]**

| Bits | Field |
|------|-------|
| 1:0 | APER_0_ENDIAN: 0 none, 1 16 bpp swap (bytes swapped within each 16-bit half), 2 32 bpp swap (full dword reversal) |
| 3:2 | APER_1_ENDIAN, same encoding |
| 4 | APER_REG_ENDIAN: 32-bit swap for the register aperture (which one isn't stated; not emulated) |
| 8 | CFG_VGA_RAM_EN |
| 9 | CFG_VGA_IO_DIS |
| 19:16 | CFG_ATI_REV_ID (chip revision) |

The two linear apertures exist **for PowerMac** **[M]**: a big-endian host
gets one register aperture it can map cacheable and linear apertures with
separate swap settings. VRAM itself is little-endian. All engine-side
structures (command buffers, vertex data, textures, GART tables, descriptors)
are little-endian.

`DP_DATATYPE` bit 29, HOST_BIG_ENDIAN_EN, swaps host data (see the 2D notes)
**[M]**.

## Other memory configuration

| Register | Notes |
|----------|-------|
| `CONFIG_MEMSIZE` 0xF8 | VRAM size in bytes, 1 MiB granularity, max 32 MiB **[M]** |
| `CONFIG_MEMSIZE_EMBEDDED` 0x114 | Reads 0 in the emulation **[A]** |
| `MEM_CNTL` 0x140 | Memory type. The emulation sets 1 (SDR SGRAM) at reset **[A]** |
| `AGP_BASE` 0x170, `AGP_CNTL` 0x174 | |
| `AGP_APER_OFFSET` 0x178 | AGP image within the card's address space. The manual gives 32 MiB **[M]**. The emulation treats it as an offset into the host's AGP aperture, with the card-side window fixed at 32 MiB **[A]** |
| `PCI_GART_PAGE` 0x17C | PCI scatter-gather table base (8192 dword entries). Bit 0 = disable the PCI GART **[M]**. See [cce-pm4.md](cce-pm4.md) |
| `BUS_CNTL` 0x30 | Bit 6 = BUS_MASTER_DIS **[D]** |

## The card's own address space

Engine addresses (texture offsets, Z offset, vertex buffers, ring, indirect
buffers) are 26-bit card addresses:

- **0 – 32 MiB:** local VRAM (mirrored if smaller).
- **32 MiB – 64 MiB:** the AGP / PCI GART window. Linux (`R128_AGP_OFFSET`) and
  Mac OS X both put GART memory at card address 0x0200_0000 **[D]**.

Texture offsets use bits 25:0 for the address and bits 31:30 to choose a tiled
layout **[8]**.

## Surfaces and tiling

### SURFACE windows (0x0B00–0x0B3C) **[M]**

The manual confirms these as Rage 128 registers. Mac OS 9 programs them;
nothing else seen does **[D]**.

| Register | Notes |
|----------|-------|
| `SURFACE_DELAY` 0xB00 | POW2 (3:0, default 3), NONPOW2 (7:4, default 5), SURF_TRANSLATION_DIS (bit 8, default 1) |
| `SURFACEn_LOWER_BOUND` 0xB04 + 0x10·n | 64-byte granular, bits 25:0 |
| `SURFACEn_UPPER_BOUND` 0xB08 + 0x10·n | 64-byte granular, bits 25:0 |
| `SURFACEn_INFO` 0xB0C + 0x10·n | SURFn_PITCHSEL in bits 4:0 |

There are 4 surfaces (n = 0–3).

PITCHSEL (bytes per line): 0 = linear (window off), 1–7 = 64, 128 … 4096,
8 = 640, 9 = 1280, 10 = 2560, 11 = 5120, 12 = 1600, 13 = 3200, 14 = 6400,
15 = 832, 16 = 1664, 17 = 3328, 18 = 1920, 19 = 3840.

Model used **[A]**: a host access through a linear aperture whose address falls
in [LOWER, UPPER) sees a tiled surface as if it were linear, at that window's
pitch. Every pitch is a multiple of 64, so a dword access never crosses a tile
column.

### Tile layout **[8]**

Tiles are 64 bytes × 16 lines (1 KiB), stored row by row. Byte `xb` of line
`y`, for a surface of pitch `P` bytes, is at:

```
16·P·(y / 16) + 1024·(xb / 64) + 64·(y % 16) + (xb % 64)
```

This only applies when P is a nonzero multiple of 64. Otherwise treat the
surface as linear. 86Box notes that ATI's programming guide **misprints the
third term**.

Tiling is selected per surface:

- `DST_PITCH_OFFSET` / `SRC_PITCH_OFFSET` bit 31 (2D);
- `DST_PITCH_OFFSET_C` bit 31 (3D colour);
- `Z_PITCH_C` bit 16 (Z);
- texture offset bits 31:30;
- `CRTC_OFFSET_CNTL` TILE_EN (scanout, not emulated).

## VGA

There is a standard VGA core. The palette, DAC mask and 8-bit DAC mode are
shared with the native CRTC (see [display.md](display.md)).

`MEM_VGA_WP_SEL` 0x38 / `MEM_VGA_RP_SEL` 0x3C bank the 64 KiB VGA window. The
emulation treats each as two 32 KiB page numbers: low 16 bits for the lower
half of the window, high 16 bits for the upper half. This applies only in
packed-pixel (chain-4) modes, as on mach64 **[A]**.

A VGA-class card decodes A0000–BFFFF and 3B0–3DF whenever the PCI memory / I/O
decoders are on.
