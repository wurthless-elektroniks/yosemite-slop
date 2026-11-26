# 2D (GUI) engine

All GUI registers are in 0x1400–0x1FFF. They are FIFOed and dword-only
**[M]**. Drivers can write them directly or through CCE Type 0 packets. Most
Mac OS drawing uses CCE Type 3 `CNTL_*` packets, which load the same state
(see [cce-pm4.md](cce-pm4.md)).

## Status and synchronisation

### GUI_STAT 0x1740 **[M]**

| Bits | Field |
|------|-------|
| 11:0 | GUI_FIFOCNT: free FIFO entries, 0x40 at reset |
| 16 | PM4 busy |
| 17 | MICRO busy |
| 18 | FPU busy |
| 19 | VC busy |
| 20 | IDCT busy |
| 21 | ENG_EV busy |
| 22 | SETUP busy |
| 23 | EDGEWALK busy |
| 24 | ADDRESSING busy |
| 25 | ENG_3D busy |
| 26 | ENG_2D_SM busy |
| 27 | ENG_2D busy |
| 28 | GUI_WB busy |
| 29 | CACHE busy |
| 31 | GUI_ACTIVE (OR of all the busy bits) |

An emulator that finishes every operation at once can return 64 here (all
free, idle).

### FLUSH_1..FLUSH_7 0x1704–0x171C **[M]**

FLUSH_1 blocks FIFOed writes until the level-1 engines are idle. Drivers use
these as pipeline barriers:

- Mac OS 9 writes FLUSH_7 = 0 after every PC_GUI_CTLSTAT purge.
- On chip family 0x500 **only**, Mac OS 9 writes FLUSH_1 = 0 **35 times before
  every primitive**. This looks like an erratum workaround **[D]**.

### WAIT_UNTIL 0x1720 **[M]**

Stalls the engine until **all** the selected conditions hold (AND).

| Bit | Condition |
|-----|-----------|
| 0 | CRTC_OFFSET (flip done) |
| 1 | RE_CRTC_VLINE |
| 2 | FE_CRTC_VLINE |
| 3 | CRTC_VLINE |
| 4–7 | BM_VIP0–3_IDLE |
| 8 | BM_VIDCAP_IDLE |
| 9 | BM_GUI_IDLE |
| 10 | CMDFIFO (needs the entry count in 26:20) |
| 11 | OV0_FLIP |

Values seen from drivers: 0x1, 0x2, 0x4, 0x8 and 0x400, plus bit 31 for the
CRTC2 versions **[D]**. The emulation doesn't stall on these yet.

### Pixel cache **[M]**

`PC_GUI_CTLSTAT` 0x1748 and `PC_NGUI_CTLSTAT` 0x184:

| Bits | Field |
|------|-------|
| 1:0 | FLUSH_GUI |
| 3:2 | RI_GUI |
| 5:4 | FLUSH_NONGUI |
| 7:6 | RI_NONGUI |
| 8 | PURGE_GUI (self-clearing) |
| 9 | PURGE_NONGUI (self-clearing) |
| 24 | DIRTY (status) |
| 25 | PURGE_DOPURGE |
| 26 | FLUSH_DOFLUSH |
| 27 | BUSY_INIT |
| 28 | BUSY_FLUSH |
| 29 | BUSY_GUI |
| 30 | BUSY_NGUI |
| 31 | BUSY |

Mac OS 9 starts each 3D context with PC_GUI_CTLSTAT = 0x100 (PURGE_GUI)
**[D]**.

`PC_GUI_MODE` 0x1744 and `PC_NGUI_MODE` 0x180 hold static setup **[M]**. Mac
OS X writes 0x122 to 0x1744 **[D]**.

`GUI_SCRATCH_REG0–5` 0x15E0–0x15F4 are FIFOed scratch registers **[M]**. Mac
OS X writes a fence stamp to SCRATCH_REG0 after each command buffer **[D]**.

## DP_GUI_MASTER_CNTL (GMC) 0x146C **[M]**

This one register sets most of the operation. It is an alias for fields of
DP_DATATYPE and DP_MIX, plus "load default" controls.

| Bits | Field | Notes |
|------|-------|-------|
| 0 | SRC_PITCH_OFFSET_CNTL | 0 = load SRC offset/pitch from DEFAULT_OFFSET/DEFAULT_PITCH |
| 1 | DST_PITCH_OFFSET_CNTL | 0 = load DST offset/pitch from defaults |
| 2 | SRC_CLIPPING | 0 = source scissor = DEFAULT_SC_BOTTOM_RIGHT |
| 3 | DST_CLIPPING | 0 = destination scissor = (0,0)–DEFAULT_SC_BOTTOM_RIGHT |
| 7:4 | BRUSH_DATATYPE | → DP_DATATYPE 11:8 |
| 11:8 | DST_DATATYPE | → DP_DATATYPE 3:0 |
| 13:12 | SRC_DATATYPE | → DP_DATATYPE 17:16 |
| 14 | BYTE_PIX_ORDER | → DP_DATATYPE 30. 1 = LSB first for mono data |
| 15 | CONVERSION_TEMP | |
| 23:16 | ROP3 | → DP_MIX 23:16 |
| 26:24 | DP_SRC_SOURCE | → DP_MIX 10:8 |
| 27 | GMC_3D_FCN_EN | 0 clears SCALE_3D_FCN, Z_EN and STENCIL_EN |
| 28 | CLR_CMP_CNTL_DIS | |
| 29 | AUX_CLIP_DIS | |
| 30 | WR_MSK_DIS | Write mask forced to 0xFFFFFFFF |
| 31 | LD_BRUSH_Y_X | A BRUSH_Y_X dword is present in the PM4 packet |

The mapping into DP_DATATYPE / DP_MIX bit positions is how the emulation does
it, and it's consistent with r128_reg.h **[D]**. A GMC write reloads the
defaults *at the time of the write* **[A]**.

| Register | Notes |
|----------|-------|
| `DEFAULT_OFFSET` 0x16E0 | |
| `DEFAULT_PITCH` 0x16E4 | 10 bits |
| `DEFAULT_SC_BOTTOM_RIGHT` 0x16E8 | Right in 12:0, bottom in 28:16. Reset 0x1FFF_1FFF **[A]** |

### Destination datatypes **[M]**

| Value | Format | Bytes/pixel |
|-------|--------|-------------|
| 2 | 8 bpp (palettised) | 1 |
| 3 | ARGB 1555 | 2 |
| 4 | RGB 565 | 2 |
| 5 | 24 bpp | 3 |
| 6 | ARGB 8888 | 4 |
| 7 | RGB 332 | 1 |
| 8 | Y8 | 1 |
| 9 | RGB8 (8-bit grey) | 1 |
| 11 | VYUY 4:2:2 | 2 |
| 12 | YVYU 4:2:2 | 2 |
| 14 | AYUV 4:4:4 | 4 |
| 15 | ARGB 4444 | 2 |

Types 7–15 are 3D only **[M]**.

### Source datatypes

| Value | Meaning |
|-------|---------|
| 0 | Mono, foreground/background (DP_SRC_FRGD_CLR 0x15D8 / DP_SRC_BKGD_CLR 0x15DC) |
| 1 | Mono, foreground / leave alone (clear bits don't draw) |
| 3 | Colour, same format as the destination |

These are as in r128_reg.h **[D]**.

### DP_SRC_SOURCE **[M]**

| Value | Source |
|-------|--------|
| 2 | Memory (SRC_OFFSET/PITCH, SRC_X/Y) |
| 3 | Host data |
| 4 | Host data, byte-aligned (each row starts on a fresh byte) |

These are ignored when SCALE_3D_FCN ≠ 0.

## Surfaces, pitch and coordinates

| Register | Notes |
|----------|-------|
| `DST_OFFSET` 0x1404, `SRC_OFFSET` 0x15AC | Byte offsets, 26 bits |
| `DST_PITCH` 0x1408, `SRC_PITCH` 0x15B0 | 10 bits, units of **8 pixels** |
| `DST_PITCH_OFFSET` 0x142C, `SRC_PITCH_OFFSET` 0x1428 | Combined form: offset in 20:0, pitch in 30:21 (pixels/8), tile in bit 31 **[M]** |

The manual contradicts itself on the units of the packed offset. The DST
description says 32-byte aligned; the SRC description says 64-bit words.
Treating both as **32-byte units** (`offset = field << 5`) works with every
driver tested **[D]/[A]**.

At 24 bpp the pitch is taken as units of 8 *bytes* rather than 8 pixels
**[A]**.

Coordinates are signed 16-bit. There are many aliases, which differ in
packing and in whether they start the operation:

| Register | Packing | Starts op? |
|----------|---------|-----------|
| `DST_X` 0x141C / `DST_Y` 0x1420 / `SRC_X` 0x1414 / `SRC_Y` 0x1418 | single | no |
| `DST_Y_X` 0x1438, `SRC_Y_X` 0x1434 | Y in 31:16, X in 15:0 | no |
| `DST_X_Y` 0x1594, `SRC_X_Y` 0x1590 | X in 31:16, Y in 15:0 | no |
| `DST_X_SUB` 0x15A4 / `DST_Y_SUB` 0x15A8 | 1/16-pixel units **[A]** | no |
| `DST_WIDTH` 0x140C | | no |
| `DST_HEIGHT` 0x1410 | | **yes** |
| `DST_HEIGHT_WIDTH` 0x143C, `_8` 0x158C, `_BW` 0x15B4 | H in 29:16, W in 13:0 | **yes** |
| `DST_WIDTH_HEIGHT` 0x1598 | W in 29:16, H in 13:0 | **yes** |
| `DST_HEIGHT_Y` 0x15A0 | H in 29:16, Y in 15:0 | **yes** |
| `DST_WIDTH_X` 0x1588, `DST_WIDTH_X_INCY` 0x159C | W in 29:16, X in 15:0 | **yes** |

After a rectangle, DST_Y moves on by its height in the Y direction (so stacked
rectangles can be drawn with one coordinate write each) **[A]**.

### DP_CNTL 0x16C0 **[D]**

| Bit | Field |
|-----|-------|
| 0 | X direction (1 = left to right) |
| 1 | Y direction (1 = top to bottom) |
| 2 | Y major (lines) |
| 5 | DST_LAST_PEL: draw the last pixel of a line |

Alias `DP_CNTL_XDIR_YDIR_YMAJOR` 0x16D0 packs the same three bits at 31 (X
dir), 15 (Y dir) and 2 (Y major).

Overlapping screen-to-screen copies need the directions chosen by the driver
(or by the CCE microcode for BITBLT_MULTI, see the CCE notes).

## Scissors

| Register | Notes |
|----------|-------|
| `SC_LEFT` 0x1640, `SC_RIGHT` 0x1644, `SC_TOP` 0x1648, `SC_BOTTOM` 0x164C | Inclusive |
| `SC_TOP_LEFT` 0x16EC, `SC_BOTTOM_RIGHT` 0x16F0 | Packed (Y in 31:16, X in 15:0) |
| `SRC_SC_RIGHT` 0x1654, `SRC_SC_BOTTOM` 0x165C, `SRC_SC_BOTTOM_RIGHT` 0x16F4 | Source scissor |

Negative coordinates are never drawn **[A]**.

The source scissor only matters when the ROP actually uses the source. The Mac
OS 9 accelerator paints with DP_SRC_SOURCE = memory and ROP 0xF0 (PATCOPY).
It leaves SRC_X/SRC_Y wherever the last blit ended. If those stale source
coordinates are clipped against SRC_SC_BOTTOM_RIGHT, later fills come out
short **[D]**.

### Auxiliary scissors (AUX_SC_CNTL 0x1660, AUXn_SC_* from 0x1664)

Three extra rectangles: AUX1 at 0x1664, AUX2 at 0x1674, AUX3 at 0x1684. Each
has LEFT, RIGHT, TOP and BOTTOM in consecutive dwords.

In AUX_SC_CNTL, rectangle *i* has an enable bit at 2·*i* and a mode bit at
2·*i*+1. Mode 0 is OR: the pixel must be inside at least one OR rectangle.
Mode 1 is NAND: the pixel must be outside it. GMC bit 29 turns them all off.
The Linux DRM uses them for cliprects **[D]**.

## Brushes (patterns)

| Register | Notes |
|----------|-------|
| `DP_BRUSH_FRGD_CLR` 0x147C, `DP_BRUSH_BKGD_CLR` 0x1478 | Brush colours |
| `BRUSH_DATA0–63` 0x1480–0x157C | Pattern data **[M]** |
| `BRUSH_Y_X` 0x1474 | Pattern origin **[M]** |
| `BRUSH_SCALE` 0x1470 | Pixels per BRUSH_X step in 7:0 (3D lines only, 0 = 256); start in 15:8 **[M]** |

### Brush types (GMC 7:4) **[M]**

| Value | Brush |
|-------|-------|
| 0 / 1 | 8×8 mono, FG/BG or FG/leave-alone |
| 4 / 5 | 1×8 mono |
| 6 / 7 | 32×1 mono (also used for line styles) |
| 8 / 9 | 32×32 mono (the OpenGL polygon stipple, see 3D) |
| 10 | 8×8 colour |
| 12 | 1×8 colour |
| 13 | Solid (DP_BRUSH_FRGD_CLR) |
| 15 | None. Reserved for the PM4 parser to mean "no brush" |

Odd mono types leave background pixels alone.

### Data layout **[A]** (matches the Mac OS 9 and NT drivers' output)

- **8×8 mono:** 8 bytes in BRUSH_DATA0–1, one byte per row. Row *r* is byte
  (*r* & 3) of dword *r* >> 2. Bit order within each byte follows
  BYTE_PIX_ORDER (MSB first by default).
- **32×1 and 32×32 mono:** one dword per row. With MSB first, bit 31 is the
  leftmost pixel. For 32×1, lines index by position along the line rather than
  by X.
- **Colour brushes:** packed pixels at the destination depth, little-endian
  through BRUSH_DATA (64 pixels for 8×8, 8 for 1×8).
- **Origin:** the pattern is indexed by (screen X + BRUSH_X, screen Y +
  BRUSH_Y). The emulation takes X from BRUSH_Y_X bits 4:0 and Y from bits
  12:8 for 2D patterns, but from bits **20:16** for the 3D stipple, because
  Mac OS 9's RAVE engine writes it that way. One of these is probably wrong.
  See open questions.

## ROP3 and the pixel write

The ROP3 code works the usual Windows way: bit (P<<2 | S<<1 | D) of the code
is the result for that combination of pattern, source and destination. All
256 are valid. Useful ones: 0xCC SRCCOPY, 0xF0 PATCOPY, 0xAA (no-op, D), 0x66
XOR, 0x5A PATINVERT.

What the emulation does per pixel, in order:

1. Destination scissor, then the aux scissors.
2. Pattern. A leave-alone mono bit stops here.
3. Colour compare, unless GMC bit 28 is set.
4. ROP3.
5. Write mask (`DP_WRITE_MASK` 0x16CC, unless GMC bit 30 is set).
6. Write.

### Colour compare

`CLR_CMP_CNTL` 0x15C0 **[D]**:

| Bits | Field |
|------|-------|
| 2:0 | Source function |
| 10:8 | Destination function |
| 24 | CLR_CMP_SRC: the source function tests the *source* pixel (else the destination) |

Functions: 0 always draw, 1 never draw, 4 draw if equal to the compare colour,
5 draw if different. Compare colours are in `CLR_CMP_CLR_SRC` 0x15C4 and
`CLR_CMP_CLR_DST` 0x15C8. `CLR_CMP_MASK` 0x15CC applies to both (reset
0xFFFFFFFF).

The Mac OS 9 accelerator uses CLR_CMP_CNTL = **0x0100_0005** for transparent
blits: draw source pixels that differ from the key. The key is the port's
background colour **[D]**.

## Rectangles and blits

An operation starts when a width/height register is written (table above).

- **Solid / pattern fill:** source unused.
- **Screen-to-screen blit:** DP_SRC_SOURCE = 2, colour source. Reads SRC_X/Y
  from SRC_OFFSET/PITCH.
- **Mono expansion from memory:** DP_SRC_SOURCE = 2 with SRC_DATATYPE 0 or 1.
  The mono bitmap pitch is SRC_PITCH × 8 × 8 *bits* per row **[A]**.
- **Host data:** DP_SRC_SOURCE = 3 or 4. The rectangle waits for data at
  `HOST_DATA0–7` 0x17C0–0x17DC (any of them), ended by `HOST_DATA_LAST`
  0x17E0 **[M]**.

### Host data formats

**Mono** data is a stream of bytes in memory order. Each dword is taken as its
4 bytes, low address first, and bits within a byte follow BYTE_PIX_ORDER.

**HOST_BIG_ENDIAN_EN does not affect mono data.** The Mac OS 9 accelerator
sends the same unswapped glyph and icon bitmaps at 8 bpp (bit clear) and at 16
and 32 bpp (bit set) **[D]**.

**Colour** data is packed pixels. With HOST_BIG_ENDIAN_EN, each 16 bpp pixel
is byte-swapped (swap mode 1), and 32 bpp pixels are fully reversed. Mac OS 9
copies big-endian pixels into its command buffers unswapped and sets the bit
at 16 and 32 bpp only **[D]**. What happens at 24 bpp isn't known.

24 bpp colour data is a byte stream, three bytes per pixel **[A]**.

## Bresenham lines

| Register | Notes |
|----------|-------|
| `DST_BRES_ERR` 0x1628, `DST_BRES_INC` 0x162C, `DST_BRES_DEC` 0x1630 | 18-bit signed |
| `DST_BRES_LNTH` 0x1634 (or `_SUB` 0x1638) | Length; writing it draws the line |

The line starts at DST_X/Y, with direction and major axis from DP_CNTL.

At each step: plot, step the major axis, add INC to the error. When the error
is ≥ 0, step the minor axis and add DEC **[A]** (standard). Afterwards DST_X/Y
and DST_BRES_ERR are left at the end point.

## Scaler registers (direct, non-CCE)

A scaled copy from a texture-format source into the 2D destination. It starts
when `SCALE_DST_HEIGHT_WIDTH` is written.

**Mac OS X draws its 32-bit ARGB cursor this way** **[D]**: blended, with
MISC_3D_STATE_CNTL_REG = 0x0054_0100 (source alpha / inverse source alpha),
and clipped to the cursor rectangle with SC_TOP_LEFT_C / SC_BOTTOM_RIGHT_C.

| Register | Notes |
|----------|-------|
| `SCALE_SRC_HEIGHT_WIDTH` 0x1994 | H in 29:16, W in 13:0 |
| `SCALE_OFFSET_0` 0x1998 | Source address |
| `SCALE_PITCH` 0x199C | Units of 8 texels **[A]** |
| `SCALE_X_INC` 0x19A0, `SCALE_Y_INC` 0x19A4 | 16.16 |
| `SCALE_HACC` 0x19A8, `SCALE_VACC` 0x19AC | 16.16 start accumulators |
| `SCALE_DST_X_Y` 0x19B0 | X in 31:16, Y in 15:0 |
| `SCALE_DST_HEIGHT_WIDTH` 0x19B4 | Trigger |
| 0x1A20 | Source datatype (texture format codes). Name assumed: SCALE_3D_DATATYPE **[A]** |

The destination format comes from DP_GUI_MASTER_CNTL_C. Blending happens only
when MISC_3D_STATE_CNTL_REG sets a blend function **[A]**.

The CCE SCALING packets carry their own 11-dword scaler block instead (see the
CCE notes).

## Bus-master GUI DMA (VRAM → system memory)

The register names follow r128_reg.h; the details are from Mac OS X **[D]**.

Mac OS X copies surfaces from VRAM to system memory with a table of 16-byte
little-endian descriptors:

| Dword | Contents |
|-------|----------|
| 0 | Frame buffer offset |
| 1 | System memory (bus) address |
| 2 | Command: byte count in 20:0, 0x2800_0000 seen in the upper bits, bit 31 = end of list |
| 3 | Status |

Sequence: wait for `BM_QUEUE_STATUS` 0x0A10 bit 27 (busy) to clear. Write the
table address with bit 0 set to **0x0A50** (called BM_GUI_TABLE here; the name
is unconfirmed). Wait again.

Other command values, and the opposite copy direction, haven't been seen.
`BM_CHUNK_0_VAL` routing applies (GUI DMA sets PM4_RD and GLOBAL force-to-PCI).
