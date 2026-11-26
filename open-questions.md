# Open questions, contradictions, and tests for real hardware

What we know we don't know. Each item says what the sources say and what would
settle it. Most could be settled by poking a real card: peek/poke over a
bare-metal or Linux-live-CD probe on a Yikes! G4 or B&W G3.

## Contradictions between sources

### 1. BAR2 size and what lives above 0x2000 / 0x4000

- **Manual [M]:** BAR2 is 16 KiB. Register aperture 0 is at +0 and register
  aperture 1 at +0x2000. `CONFIG_REG_APER_SIZE` reads 0x2000, and
  `CONFIG_REG_1_BASE` bit 13 is hardwired to 1. CONFIG_CNTL bit 4
  (APER_REG_ENDIAN) gives one register aperture a 32-bit swap.
- **Driver [D]:** the Mac OS 9 driver on a Beige G3 writes the PIO FIFO at
  BAR2+0x5000 and HOST_DATA at BAR2+0x57C0. That's +0x4000 from the normal
  addresses, byte-reversed.
- **Emulation [A]:** BAR2 is 32 KiB, a 16 KiB register file repeated, with no
  swap and APER_REG_ENDIAN not implemented. It also returns 0x4000 for
  CONFIG_REG_APER_SIZE and BAR2's own base for CONFIG_REG_1_BASE. Neither
  matches the manual.

**Test:**

1. Size BAR2 by writing 0xFFFFFFFF and reading it back.
2. Read CONFIG_REG_APER_SIZE and CONFIG_REG_1_BASE.
3. Read a known register (e.g. GEN_INT_CNTL, or a scratch register) at +0,
   +0x2000, +0x4000 and +0x6000. Do this with APER_REG_ENDIAN 0 and 1, and
   compare byte order.

### 2. OFFSET_FLIP_CNTL polarity

- **Manual [M]:** CRTC_OFFSET_CNTL bit 16 = 0 flips at vblank, 1 flips at any
  hblank.
- **Emulation [A]:** bit set waits for vblank, clear is immediate.

**Test:** write CRTC_OFFSET mid-frame with the bit clear, then set. Watch
GUI_TRIG_OFFSET (CRTC_OFFSET bit 30, "flip pending") and the picture.

### 3. Power management capability

- **Manual [M]:** D0, D1 and D3, no D2.
- **Emulation:** reports D1 and D2.

**Test:** read config 0x5C–0x63.

### 4. BRUSH_Y_X layout

- The 2D pattern code takes Y from bits 12:8.
- Mac OS 9's 3D stipple writes Y in bits 20:16.
- The manual describes BRUSH_Y_X at 0x1474. Its field layout should settle
  which reading is right, so **check the manual first**.

### 5. PLL VCO range and reference clock

- The manual gives 125–250 MHz.
- ATI's NT miniport falls back to 118–236 MHz with a 29.5 MHz reference.

The Apple card's crystal hasn't been read. It's in the FCode / ndrv PLL data
and on the board.

### 6. CONFIG_CNTL revision field width

- The manual gives CFG_ATI_REV_ID as 19:16.
- The emulation returns 20:16.

The Apple card reports PCI revision 0. ATI's Win9x driver only knows GL/VR
revisions 2 and 3.

### 7. DST/SRC_PITCH_OFFSET offset units

The manual's two descriptions disagree: 32-byte aligned vs 64-bit words. The
emulation uses 32-byte units for both, which works for every driver tested.
That suggests 32 bytes, at least for DST.

**Test:** blit from a SRC_PITCH_OFFSET with an odd 32-byte offset.

### 8. Conflicting notes on Mac OS 9's SETUP_CNTL

One emulation comment says Mac OS 9 writes SETUP_CNTL = 0. Another says it
sets TEXTURE_ST_DIRECT (bit 9). Re-check the RAVE engine's context emit
before relying on either.

### 9. Config mirror writability

The manual calls MMIO 0x0F00–0x0FFF read-only. But Mac OS X sets bus-master
enable by writing 0x0F04.

**Test:** whether the command register (and anything else) is writable through
the mirror.

## Assumptions with no source beyond "the drivers work"

**PCI, bus mastering and the CCE**

- AGP_APER_OFFSET as an offset into the host AGP aperture, with the card's
  GART window fixed at 32 MiB.
- The PIO FIFO window as 0x1000–0x11FF.
- 0x0740 reporting free FIFO entries in its low byte.
- PM4_STAT free count (64 in mode 7?) and the FIFO depth per mode.
- GUI DMA register names (BM_QUEUE_STATUS 0x0A10, table at 0x0A50) and the
  copy direction for command values other than 0x28xxxxxx.

**Display and VGA**

- MEM_VGA_WP_SEL / RP_SEL as two 32 KiB page numbers.
- CUR_OFFSET ignoring its low four bits. A cursor vertical offset shortens the
  image rather than skipping rows.
- The I2C engine bit assignments.

**2D engine**

- The pattern origin (BRUSH_Y_X) is added to screen coordinates.
- The CNTL packet brush operands: one colour for FG_LA mono brushes, a
  BRUSH_Y_X dword when GMC bit 31 is set.
- PAINT takes two corners, where PAINT_MULTI takes a position and size.
- The TRANS_BITBLT payload: CLR_CMP_CNTL, CLR_CMP_CLR_SRC and CLR_CMP_CLR_DST
  between the GMC state and the blits.
- POLYSCANLINES spans as (x_right << 16) | x_left, right edge exclusive.
- POLYLINE points as (y << 16) | x, with DP_CNTL DST_LAST_PEL deciding whether
  the final point is drawn.
- Copy direction for CCE BITBLT_MULTI. The microcode chooses on hardware.
- Memory-source rectangles ignore SRC_X/SRC_Y and the source scissor when the
  ROP doesn't use the source.
- HOST_BIG_ENDIAN_EN swaps colour host data per pixel and leaves mono host data
  alone. 24 bpp behaviour is unknown.
- SCALE_PITCH units, 0x1A20 as the scaler source format, and blending only
  when MISC_3D_STATE_CNTL_REG sets a blend function.

**3D engine**

- Float colour components in vertex formats as 0.0–1.0.
- The texture level layout (from 86Box).
- CI4 nibble order.
- 0x1968 taking the palette entry number in its low byte. Mac OS 9 writes
  0x2000 there.
- The LOAD_PALETTE payload layout.
- The polygon stipple bit and row order.
- The TEX_CNTL_C bit 13 name. Its effect is confirmed on hardware.
- Sample position (pixel centre), the fill rule (top-left) and LOD
  computation.

## Not emulated at all

These are worth characterising on hardware if anyone has the time.

- Tiled scanout (CRTC_OFFSET_CNTL TILE_EN).
- CRTC2, and LVDS/TMDS panel scaling.
- Dithering, table fog, LOD bias, VQ/S3TC textures (datatype 0).
- The direct (non-CCE) setup engine. The Windows NT GDI GradientFill path
  programs RED_X_INC … ALPHA_START (0x1A40–0x1A6C) and a trigger that hasn't
  been decoded.
- Trapezoid and small-text GUI primitives, and the NEXT_CHAR packet.
- Scaler blocks: source tiling, the source start phase when filtering, YUV
  sources.
- WAIT_UNTIL actually stalling.
- The DVD sub-picture and motion compensation / IDCT blocks.
- The 3D register write protection while SCALE_3D_FN = 0.
- Registers 0x07A4 (bit 3 set by Mac OS 9 on families 0x510/0x512/0x520) and
  0x1804. Neither is named in the manual.

## Suggested probe checklist

To run on a real Rage 128 GL. Anything marked **[A]** in the other files is
fair game; these are the ones that matter most for an emulator.

1. BAR sizes (all four, including ROM). Read the whole option ROM EEPROM past
   0x12800.
2. Register aperture copies at +0x2000 / +0x4000, with APER_REG_ENDIAN 0 and 1.
3. Config space 0x00–0x7F directly and through MMIO 0x0F00. Try writing via
   the mirror.
4. Reset values of every register in 0x0000–0x01FF and of the PLL registers
   0x00–0x3F.
5. CONFIG_MEMSIZE, CONFIG_CNTL revision bits, MEM_CNTL.
6. GUI_STAT, PM4_STAT and 0x0740 when idle, in each PM4_BUFFER_CNTL mode.
7. OFFSET_FLIP_CNTL behaviour.
8. Mono 8×8 / 32×32 brush bit order and BRUSH_Y_X fields (draw a known pattern
   with nonzero origins, read VRAM back).
9. HOST_BIG_ENDIAN_EN at 8/16/24/32 bpp, colour and mono.
10. A CCE PAINT vs PAINT_MULTI with known operands, to settle the
    inclusive/exclusive edges.
11. A textured triangle with known S/T, RHW and filters, read back pixel by
    pixel. This checks the sample position, fill rule, mip slot layout and
    CI4 nibble order.
12. Write 3D context registers with SCALE_3D_FN = 0 and read them back, to see
    which are protected.
