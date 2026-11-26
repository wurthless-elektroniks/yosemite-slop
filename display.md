# Display: CRTC, PLL, DAC, cursor, overlay, interrupts, DDC

## VGA mode vs native mode

`CRTC_GEN_CNTL` (0x50) bit 24, EXT_DISP_EN, switches the display from the VGA
core to the native ("extended") CRTC **[M]**. While it's set, the VGA CRTC
registers don't drive the screen.

## CRTC_GEN_CNTL 0x50 **[M]**

| Bits | Field |
|------|-------|
| 0 | CRTC_DBL_SCAN_EN: each source line shown twice **[D]** (aty128fb) |
| 10:8 | PIX_WIDTH: 1 = 4 bpp, 2 = 8 bpp, 3 = 15 bpp (1555), 4 = 16 bpp (565), 5 = 24 bpp, 6 = 32 bpp. 7 is treated as direct RGB 3:3:2 **[A]** |
| 16 | CUR_EN, hardware cursor |
| 19:17 | CUR_MODE |
| 24 | EXT_DISP_EN |
| 25 | CRTC_EN |
| 26 | DISP_REQ_EN_B: default 1 = display requests **disabled** |

## CRTC_EXT_CNTL 0x54 **[M]**

| Bit | Field |
|-----|-------|
| 8 | HSYNC_DIS |
| 9 | VSYNC_DIS |
| 10 | DISPLAY_DIS |

aty128fb and the X.Org r128 driver blank (and do DPMS) by clearing CRTC_EN or
setting DISPLAY_DIS **[D]**. DISP_REQ_EN_B only gates the CRTC's memory
requests. It shouldn't be treated as a blank.

> **Emulation caveat:** blanking is currently *ignored*. Around its monitor
> sense, the Mac OS 9 ROM ndrv saves and restores CRTC_GEN_CNTL byte 3 with an
> unaligned 32-bit read. Under MAME, CRTC_EN ended up clear afterwards. That
> was judged to be a MAME core problem with unaligned accesses, not card
> behaviour.

## Timing **[D]** (aty128fb / r128)

| Register | Fields |
|----------|--------|
| `CRTC_H_TOTAL_DISP` 0x200 | H_TOTAL (8:0) and H_DISP (23:16), both in 8-pixel characters, minus 1 |
| `CRTC_H_SYNC_STRT_WID` 0x204 | |
| `CRTC_V_TOTAL_DISP` 0x208 | V_TOTAL (10:0) and V_DISP (26:16), in lines, minus 1 |
| `CRTC_V_SYNC_STRT_WID` 0x20C | |
| `CRTC_PITCH` 0x22C | Pitch in units of 8 pixels **[M]** |
| `CRTC_OFFSET` 0x224 | Scanout start byte address, bits 24:0, bits 2:0 zero **[M]**. Bit 30 GUI_TRIG_OFFSET (read: flip pending). Bit 31 OFFSET_LOCK |
| `CRTC_OFFSET_CNTL` 0x228 | TILE_LINE (4:0), TILE_ALIGN (10:8), TILE_EN (15), OFFSET_FLIP_CNTL (16) **[M]** |

Pixel layout in VRAM is little-endian:

- 15 bpp is x1555;
- 16 bpp is 565;
- 24 bpp is packed B, G, R bytes;
- 32 bpp is xRGB8888.

In direct-colour modes **each component goes through the palette** (gamma), so
the palette has to be loaded with a ramp **[D]**.

### Page flipping

The manual says OFFSET_FLIP_CNTL **0 = flip at vertical blank, 1 = flip at any
horizontal blank** **[M]**. The emulation has it the other way round: bit set
latches the new CRTC_OFFSET at vblank, bit clear takes it immediately. That's
a known contradiction, to be fixed or confirmed on hardware.

Drivers do flips through the CCE: `WAIT_UNTIL`, `CRTC_GUI_TRIG_VLINE`,
`CRTC_OFFSET(_CNTL)`, `DST_OFFSET`. CRTC2 versions have bit 31 set in the
WAIT_UNTIL value **[D]**.

`CRTC_GUI_TRIG_VLINE` 0x218 **[M]**:

| Bits | Field |
|------|-------|
| 10:0 | START |
| 26:16 | END |
| 31 | Read: raster is in range |

### Line counter and status

| Register | Notes |
|----------|-------|
| `CRTC_VLINE_CRNT_VLINE` 0x210 | Bits 10:0 are the line that raises the VLINE interrupt (written). Bits 26:16 read the current line **[D]** |
| `CRTC_CRNT_FRAME` 0x214 | 21-bit frame counter **[D]** |
| `CRTC_STATUS` 0x5C | Bit 0 = in vblank now. Bit 1 = VBLANK_SAVE, sticky, write 1 to clear **[D]** |
| `SNAPSHOT_VH_COUNTS` 0x240, `SNAPSHOT_F_COUNT` 0x244 | |

## Pixel clock (PLL) **[M]**

PLL registers are reached through CLOCK_CNTL_INDEX/DATA (see the PCI notes).

| Index | Register |
|-------|----------|
| 01 | CLK_PIN_CNTL |
| 02 | PPLL_CNTL |
| 03 | PPLL_REF_DIV: M in 9:0. ATOMIC_UPDATE bit 15 (read = pending, write = trigger). REF_DIV_SRC 17:16 |
| 04–07 | PPLL_DIV_0–3: N in 10:0, post-divider in 18:16 |
| 08 | VCLK_ECP_CNTL: VCLK_SRC_SEL 1:0, where 3 = PPLL **[D]** |
| 09 | HTOTAL_CNTL |
| 0A | X_MPLL_REF_FB_DIV |
| 0B | XPLL_CNTL |
| 0C | XDLL_CNTL |
| 0D | XCLK_CNTL |
| 0E | MPLL_CNTL |
| 0F | MCLK_CNTL |
| 10 | AGP_PLL_CNTL |
| 12 | FCP_CNTL |
| 13 | PLL_TEST_CNTL |

`CLOCK_CNTL_INDEX` bits 9:8 (PPLL_DIV_SEL) choose which PPLL_DIV_n is in use.

Post-divider encoding: 0 = /1, 1 = /2, 2 = /4, 3 = /8, 4 = /3, 6 = /6,
7 = /12 **[M]**. 5 is reserved. The emulation assumes /16, as on later ATI
parts **[A]**.

```
PPllClk = N × ref / M        (VCO)
pixel clock = PPllClk / post
```

The manual's legal VCO range is **125–250 MHz** **[M]**. ATI's NT miniport
instead falls back to a **118–236 MHz** VCO and reference divider 64 when it
finds no PLL block in the BIOS. It assumes a **29.50 MHz** reference crystal
then **[D]**.

Linux on PowerPC reads the dividers back, so an emulated card should come up
with plausible values, e.g. REF_DIV 0x40 and PPLL_DIV_3 = 0x0003_01B5 (VCO
201.4 MHz / 8 = 25.18 MHz) **[A]**.

The Mac reference clock hasn't been measured.

## DAC and palette **[M]**

| Register | Notes |
|----------|-------|
| `PALETTE_INDEX` 0xB0 | Write index in 7:0, read index in 23:16 |
| `PALETTE_DATA` 0xB4 | A whole 24-bit entry: B 7:0, G 15:8, R 23:16. Each access advances the index |
| `DAC_CNTL` 0x58 | DAC_8BIT_EN bit 8 (else 6-bit). DAC_MASK 31:24 (reset 0xFF) |

The VGA DAC ports 3C6–3C9 use the same palette, with VGA-style 3-byte
sequencing **[A]**.

There are two 256-entry palettes, the second for CRTC2. `DAC_CNTL` bit 5
selects which one the ports access **[D]** (as on Radeon; CRTC2 isn't
emulated).

## Hardware cursor

64×64 at 2 bpp **[M]**.

| Register | Notes |
|----------|-------|
| `CUR_OFFSET` 0x260 | Image address in VRAM |
| `CUR_HORZ_VERT_POSN` 0x264 | X in 26:16, Y in 10:0 **[D]** |
| `CUR_HORZ_VERT_OFF` 0x268 | Hot-spot / clip offset: X in 21:16, Y in 5:0 **[D]** |
| `CUR_CLR0` 0x26C, `CUR_CLR1` 0x270 | 24-bit RGB colours |

Image format **[D]**: 16 bytes per row. The first 8 bytes are 64 AND bits, the
next 8 are 64 XOR bits, MSB first within each byte.

| AND | XOR | Pixel |
|-----|-----|-------|
| 0 | 0 | CUR_CLR0 |
| 0 | 1 | CUR_CLR1 |
| 1 | 0 | transparent |
| 1 | 1 | invert screen |

With double scan each cursor row is shown twice.

**CUR_OFFSET low 4 bits are ignored** (the cursor is fetched in whole 16-byte
rows) **[A]**. Evidence: both Mac OS 9 ndrvs put the image at offset 64 and,
for a cursor clipped at the top, add a row offset that only works if the low
bits are dropped. The ROM ndrv adds yoff·2. The "ATI Driver Update" ndrv adds
(yoff·16) & 0xFF **[D]**. A vertical offset shortens the cursor from the top
rather than skipping rows at the bottom **[A]**.

## Overlay scaler (OV0)

The register names follow r128_reg.h. The semantics are what the drivers need
**[D]/[A]**.

| Register | Notes |
|----------|-------|
| `OV0_SCALE_CNTL` 0x420 | Bit 30 = overlay enable. Bit 31 = soft reset (self-clearing). Source format in 11:8 |
| `OV0_Y_X_START` 0x400, `OV0_Y_X_END` 0x404 | X in 10:0, Y in 26:16. End is exclusive |
| `OV0_VID_BUF0/1/2_BASE_ADRS` 0x440/0x444/0x448 | 16-byte aligned. Planar YUV uses buffer 0 for Y, 1 for U and 2 for V |
| `OV0_VID_BUF_PITCH0/1_VALUE` 0x460/0x464 | Luma pitch, chroma pitch |
| `OV0_H_INC` 0x480 | 4.12 fixed point, shifted left by `OV0_STEP_BY` − 1 (0x484, low 3 bits) |
| `OV0_V_INC` 0x424 | 6.20 fixed point |
| `OV0_P1_X_START_END` 0x494 | Bits 19:16 = first source pixel |
| `OV0_REG_LOAD_CNTL` 0x410 | Bit 3 (LOCK_READBACK) follows bit 0 (LOCK). Mac OS X spins on it before programming the overlay **[D]** |

Source formats used:

| Value | Format |
|-------|--------|
| 3 | RGB 1555 |
| 4 | RGB 565 |
| 6 | xRGB 8888 |
| 0x9 | Planar YUV 4:1:0 |
| 0xA | Planar YUV 4:2:0 |
| 0xB | Packed Y0 U Y1 V |
| 0xC | Packed U Y0 V Y1 |

Colour keying:

| Register | Notes |
|----------|-------|
| `OV0_GRAPHICS_KEY_CLR` 0x4EC / `_MSK` 0x4F0 | Graphics key |
| `OV0_VIDEO_KEY_CLR` 0x4E4 / `_MSK` 0x4E8 | Video key |
| `OV0_KEY_CNTL` 0x4F4 | Video key function in 2:0, graphics key function in 6:4. Bit 8 combines them with AND (set) or OR (clear) |

Key functions: 0 false, 1 true, 4 equal under the mask, 5 not equal. Where the
combined result is false, the overlay pixel replaces the graphics pixel.

## Interrupts **[M]**

`GEN_INT_CNTL` 0x40 enables. `GEN_INT_STATUS` 0x44 latches; write 1 to clear.

| Bit | Source |
|-----|--------|
| 0 | CRTC_VBLANK |
| 1 | CRTC_VLINE |
| 2 | CRTC_VSYNC |
| 3 | SNAPSHOT |
| 16 | BUSMASTER_EOL |
| 17 | I2C |
| 18 | MPP_GP |
| 19 | GUI_IDLE |
| 24 | VIPH |

Status bits 8 and 9 show CAP0/CAP1 activity. The card has one interrupt line
(INTA#).

`GEN_RESET_CNTL` 0xF0 **[M]**:

| Bit | Reset |
|-----|-------|
| 0 | SOFT_RESET_GUI (resets the 2D/3D engines and the CCE packet state) |
| 8 | VCLK |
| 9 | PCLK |
| 10 | ECP |
| 11 | DISPENG_XCLK |
| 12 | MEMCTLR_XCLK |

## Monitor ID, DDC and I2C

### GPIO_MONID 0x68 **[M]**

This register drives the 4 MONID pins. Used for Apple monitor sense
(AppleSense), or bit-banged DDC.

| Bits | Field |
|------|-------|
| 3:0 | A (output values) |
| 11:8 | Y (pin levels, read) |
| 19:16 | EN (output enables) |
| 27:24 | MASK |

For DDC, **MONID2 = SCL and MONID1 = SDA**. A pin with EN set drives its A
value. Otherwise it floats high, and the monitor may pull SDA low.

`GPIO_MONIDB` 0x6C is the DDC interface on LCD-equipped boards **[M]**.

### I2C engine

The names follow r128/Radeon. The behaviour is modelled to what drivers poll
**[A]**.

| Register | Notes |
|----------|-------|
| `I2C_CNTL_0` 0x90 | Bit 0 DONE (write 1 to clear). Bit 1 NACK. Bit 5 soft reset (empties the FIFO). Bit 8 START (first FIFO byte is the slave address). Bit 10 RECEIVE. Bit 12 GO |
| `I2C_CNTL_1` 0x94 | Data byte count in 3:0 (0 = 16) |
| `I2C_DATA` 0x98 | 16-byte FIFO |

### Monitor modelling notes

The Mac OS 9 guest only offers **640×480**. The emulated monitor's DDC EDID
(slave 0x50) lists only 640×480 at 60/72/75 Hz, with range limits of
31–38 kHz / 59–76 Hz. Without DDC, the monitor ID sense pattern also only
reports 640×480.

Real Apple monitors may report other sense codes. The ROM ndrv probes
AppleSense before DDC.
