# 3D engine

The register manual has **almost nothing** on 3D: its 3D/scaler chapter is
missing. Only a handful of the `_C` context registers are documented **[M]**.
Most of what's here comes from 86Box's Rage 128 3D code **[8]**, which was
checked against real games. The rest comes from the drivers' output **[D]**,
and from comparing Mac OS 9 RAVE rendering against a real card **[H]**.

Primitives reach the engine only through the CCE (3D_RNDR_GEN_PRIM and
3D_RNDR_GEN_INDX_PRIM). The state they use is in registers, written with
Type 0 packets beforehand.

## Context registers

| Addr | Name | Source |
|------|------|--------|
| 0x071C | PM4_VC_FPU_SETUP | [D] |
| 0x1800 | TEX_CNTL | [D] |
| 0x1804 | ? (the emulation calls it TRI_LIN_CNTL; unconfirmed, not in the manual) | [D] |
| 0x1968 | TEX_PALETTE_WR_INDEX (name assumed) | [D] |
| 0x196C | TEX_PALETTE_WR (name assumed) | [D] |
| 0x1A00 | SCALE_3D_CNTL | [D] |
| 0x1A0C | COMPOSITE_SHADOW_ID: 24-bit count of 3D primitives | [M] |
| 0x1A24 / 0x1A28 | CLR_CMP_CLR_3D / CLR_CMP_MSK_3D | [M] |
| 0x1820 / 0x1824 | DESTINATION_3D_CLR_CMP_VAL / _MSK | [M] |
| 0x1BC4 | SETUP_CNTL | [D] |
| 0x1BC8 | SOLID_COLOR | [D] |
| 0x1BCC | WINDOW_XY_OFFSET | [D] |
| 0x1BD4 | SETUP_CNTL_PM4 | [D] |
| 0x1C80 | DST_PITCH_OFFSET_C | [M] |
| 0x1C84 | DP_GUI_MASTER_CNTL_C | [M] |
| 0x1C88 / 0x1C8C | SC_TOP_LEFT_C / SC_BOTTOM_RIGHT_C | [M] |
| 0x1C90 | Z_OFFSET_C | [D] |
| 0x1C94 | Z_PITCH_C | [D] |
| 0x1C98 | Z_STEN_CNTL_C | [D] |
| 0x1C9C | TEX_CNTL_C | [D] |
| 0x1CA0 | MISC_3D_STATE_CNTL_REG | [M] |
| 0x1CA4 / 0x1CA8 | TEXTURE_CLR_CMP_CLR_C / _MSK_C (chroma key) | [D] |
| 0x1CAC | FOG_COLOR_C | [D] |
| 0x1CB0 | PRIM_TEX_CNTL_C | [D] |
| 0x1CB4 | PRIM_TEXTURE_COMBINE_CNTL_C | [D] |
| 0x1CB8 | TEX_SIZE_PITCH_C | [D] |
| 0x1CBC–0x1CE4 | PRIM_TEX_0..10_OFFSET_C | [D] |
| 0x1D00 | SEC_TEX_CNTL_C | [D] |
| 0x1D04 | SEC_TEX_COMBINE_CNTL_C | [D] |
| 0x1D08–0x1D30 | SEC_TEX_0..10_OFFSET_C | [D] |
| 0x1D34 | CONSTANT_COLOR_C (ARGB8888) | [M] |
| 0x1D38 / 0x1D3C | PRIM / SEC_TEXTURE_BORDER_COLOR_C | [D] |
| 0x1D40 | STEN_REF_MASK_C | [D] |
| 0x1D44 | PLANE_3D_MASK_C (feeds DP_WRITE_MSK) | [M] |

> **Write protection [M]:** MISC_3D_STATE_CNTL_REG SCALE_3D_FN (bits 9:8) is 0
> at reset. While it's 0, many 3D, scaler and setup registers are **not
> writable**. GMC_3D_FCN_EN (DP_GUI_MASTER_CNTL bit 27) clear also forces
> SCALE_3D_FN, Z_EN and STENCIL_EN off. The emulation doesn't model this yet.
> Drivers that work on hardware will have set SCALE_3D_FN = 2 before loading
> 3D state.

### Mac OS 9's full 3D context load **[D]**

The order the RAVE engine emits a full context in:

1. PC_GUI_CTLSTAT = 0x100 (PURGE_GUI)
2. FLUSH_7 = 0
3. SCALE_3D_CNTL
4. 0x1C80–0x1CAC (12 registers)
5. PRIM_TEX_CNTL_C … TEX_SIZE_PITCH_C
6. PRIM_TEX_0–10_OFFSET_C
7. SEC_TEX_CNTL_C, SEC_TEX_COMBINE_CNTL_C
8. SEC_TEX_0–10_OFFSET_C
9. CONSTANT_COLOR_C … SEC_TEXTURE_BORDER_COLOR_C
10. BRUSH_SCALE ×5
11. BRUSH_Y_X ×35 (a 32-row polygon stipple through BRUSH_DATA0–31)
12. STEN_REF_MASK_C, PLANE_3D_MASK_C
13. SETUP_CNTL_PM4 = 0
14. PM4_VC_FPU_SETUP
15. 0x1804
16. SETUP_CNTL, SOLID_COLOR = 0, WINDOW_XY_OFFSET

## MISC_3D_STATE_CNTL_REG 0x1CA0

| Bits | Field | Source |
|------|-------|--------|
| 7:0 | REF_ALPHA (alpha test reference) | [M] |
| 9:8 | SCALE_3D_FN: 0 none, 1 scaling, 2 texture/shading | [M] |
| 10 | SCALE_PIX_REP: replicate instead of filter, for the scaler | [M] |
| 13:12 | ALPHA_COMB_FCN. The emulation treats bit 13 as "subtract" (src − dst); other values add | [M] / [A] |
| 14 | FOG_TABLE_EN (table fog, not emulated) | [M] |
| 19:16 | ALPHA_BLND_SRC | [M] |
| 23:20 | ALPHA_BLND_DST | [M] |
| 26:24 | ALPHA_TEST_OP | [M] |
| 31:30 | CLR_CMP_FCN_3D | [M] |

### Blend factors (SRC/DST, 4 bits) **[8]/[A]**

| Value | Factor |
|-------|--------|
| 0 | zero |
| 1 | one |
| 2 | src colour |
| 3 | 1 − src colour |
| 4 | src α |
| 5 | 1 − src α |
| 6 | dst α |
| 7 | 1 − dst α |
| 8 | dst colour |
| 9 | 1 − dst colour |
| 10 | saturate: min(src α, 1 − dst α) for RGB, 1 for alpha |
| 11 | "both src α": as a source factor, also forces dst = 1 − src α |
| 12 | "both inv src α": as a source factor, also forces dst = src α |

A destination factor of 11 behaves as 12.

Blending only happens when TEX_CNTL_C bit 9 (ALPHA_ENABLE) is set. ONE/ZERO is
just a plain write. Unreal under RAVE draws most of its world with blending
enabled and ONE/ZERO **[D]**.

### Alpha test and depth compare functions

ALPHA_TEST_OP and the Z function use the same 3-bit encoding: 0 never,
1 less, 2 less-or-equal, 3 equal, 4 greater-or-equal, 5 greater, 6 not equal,
7 always **[8]/[A]**.

The alpha test compares fragment α (0–255) against REF_ALPHA.

### 3D colour key (CLR_CMP_FCN_3D)

| Value | Effect |
|-------|--------|
| 0 | No keying |
| 1 | Reject every texel |
| 2 | Reject texels that **differ** from CLR_CMP_CLR_3D under CLR_CMP_MSK_3D |
| 3 | Reject texels that **match** it |

It tests the nearest texel of texture unit 0, as ARGB8888 **[A]**.

## TEX_CNTL_C 0x1C9C **[8]/[D]**

| Bit | Field |
|-----|-------|
| 0 | Z_ENABLE |
| 1 | Z_WRITE_ENABLE |
| 3 | STENCIL_ENABLE |
| 4 | TEXMAP_ENABLE (primary texture) |
| 5 | SEC_TEXMAP_ENABLE |
| 7 | FOG_ENABLE |
| 9 | ALPHA_ENABLE (blending) |
| 10 | ALPHA_TEST_ENABLE |
| 11 | SPEC_LIGHT_ENABLE |
| 12 | CHROMA_KEY_ENABLE |
| 13 | Alpha mask: texels with α = 0 aren't drawn |

**Bit 13 is the name we're least sure of.** The name is borrowed from the
Rage Pro's TEX_AMASK_AEN **[A]**. The *effect* is confirmed on real hardware:
Mac OS 9 cuts out Nanosaur's leaves with it. Those are 1555 textures, with
blending and the alpha test both off. Unreal's masked textures set it too
**[H]**. Like the chroma key, it tests the nearest texel of unit 0.

`SCALE_3D_CNTL` 0x1A00 bit 30, TEX_MAP_ALPHA_IN_TEXTURE: when clear, texture
alpha reads as 0xFF *after* the chroma key test **[8]**.

## Vertices

### VC_FORMAT (CCE_VC_FRMT)

Each bit adds fields to every vertex. The order in memory is fixed. Every
field is one dword. Colour components are **IEEE floats, 0.0–1.0** **[A]**,
except the packed ARGB forms.

| Bit | Field | Dwords |
|-----|-------|--------|
| — | X, Y, Z (float) | 3 |
| 0 | RHW (1/W) | 1 |
| 1 | DIFFUSE_BGR (B, G, R floats) | 3 |
| 2 | DIFFUSE_A (float) | 1 |
| 3 | DIFFUSE_ARGB (packed 8888) | 1 |
| 4 | SPEC_BGR (floats) | 3 |
| 5 | SPEC_F (fog factor, float) | 1 |
| 6 | SPEC_FRGB (packed: fog in 31:24, then RGB) | 1 |
| 7 | S, T (texture set 0) | 2 |
| 8 | S2, T2 (texture set 1) | 2 |
| 9 | RHW2 | 1 |

The bit sizes are confirmed by the dword counts in Mac OS 9's vertex data
**[D]**. Mac OS 9 uses:

- untextured: 0x6, 0x26, 0x36;
- textured: 0x85, 0x87, 0x95, 0x97, 0xA5, 0xB5, …;
- |0x300 for a second texture.

Missing fields default to colour white, specular black and fog factor 1.0
(no fog) **[A]**.

`WINDOW_XY_OFFSET` 0x1BCC is added to every vertex: X offset in bits 31:20
(signed), Y offset in bits 19:4 **[A]**.

### VC_CNTL (the dword after VC_FORMAT) **[D]**

| Bits | Field |
|------|-------|
| 3:0 | Primitive: 1 points, 2 lines, 3 line strip, 4 triangle list, 5 triangle fan, 6 triangle strip, 7 "type 2" triangles (treated as a list) |
| 5:4 | Walk: 1 indexed from the vertex buffer, 2 sequential from the vertex buffer, 3 inline in the packet |
| 31:16 | Number of vertices |

- **3D_RNDR_GEN_PRIM (0x25):** walk 3. Vertices follow VC_CNTL in the packet.
- **3D_RNDR_GEN_INDX_PRIM (0x23):** the vertex buffer address comes first.
  Walk 2 reads vertices one after another from it. Walk 1 reads 16-bit
  indices from the packet, two per dword, low half first. Mac OS 9 uses walk 1
  ("WALK_LIST" in its terms) **[D]**.
- **Strips:** odd triangles swap their first two vertices, keeping a
  consistent winding.

## Setup engine

### PM4_VC_FPU_SETUP 0x071C **[8]**

| Bits | Field |
|------|-------|
| 0 | Front face is counter-clockwise |
| 2:1 | Back-face function |
| 4:3 | Front-face function |
| 6:5 | COLOR_FCN: 0 solid, 1 flat, 2 and 3 Gouraud |
| 14 | Flat shading takes the third vertex (else the first) |
| 18 | Draw every face solid |

Face functions: 0 cull, 1 points, 2 lines, 3 solid.

### SETUP_CNTL 0x1BC4 **[8]**

| Bits | Field |
|------|-------|
| 5:3 | COLOR_FCN. 1–3 select flat shading from vertex 0, 1 or 2 |
| 9 | TEXTURE_ST_DIRECT: the driver has already multiplied S and T by RHW; the setup engine shouldn't |

**Shading:**

- *solid* uses CONSTANT_COLOR_C, with no specular;
- *flat* uses the chosen vertex's colour, but the fog factor is still
  interpolated;
- *Gouraud* interpolates everything.

**Perspective:** S and T are divided per pixel by the interpolated RHW, unless
PRIM_TEX_CNTL_C bit 14 (PERSPECTIVE_DIS) is set. Unless TEXTURE_ST_DIRECT is
set, the setup engine first multiplies the vertex S/T by RHW **[8]**.

### Scissor and rasterisation

The 3D scissor is SC_TOP_LEFT_C / SC_BOTTOM_RIGHT_C, inclusive, clamped to
0–4095 **[A]**. The 2D aux scissors (AUX_SC_CNTL) also apply. The Linux DRM
uses them for cliprects **[D]**.

The emulation samples at pixel centres (+0.5) with a top-left fill rule
**[A]**. Neither has been checked against hardware.

Lines and points are sampled once per pixel along the major axis **[A]**.

### Polygon stipple

When DP_GUI_MASTER_CNTL_C's brush type is 9 (32×32 mono, leave-alone),
polygons are stippled with BRUSH_DATA0–31 as 32 rows. This is how Mesa's r128
driver selects it **[D]**.

Mac OS 9 rewrites BRUSH_Y_X for every primitive as the stipple origin: X in
bits 4:0, **Y in bits 20:16**. It loads the rows bottom first, as Mesa does
**[D]**.

Bit order used **[A]**: for stipple column *x*, test bit
`(x & ~7) | (~x & 7)` of the row dword. That's little-endian bytes, MSB first
within each byte. Unverified.

## Texture units

There are two units, primary and secondary. Their registers are the same.

### PRIM_TEX_CNTL_C 0x1CB0 / SEC_TEX_CNTL_C 0x1D00 **[8]**

| Bits | Field |
|------|-------|
| 0 | Secondary unit only: read the second S/T set (S2/T2) |
| 3:1 | MIN_BLEND (minification filter) |
| 6:4 | MAG_BLEND (magnification filter) |
| 7 | MIP_MAP_DISABLE |
| 9:8 | Wrap mode for S |
| 12:11 | Wrap mode for T |
| 14 | PERSPECTIVE_DISABLE (read from the primary unit) |
| 19:16 | Texture datatype |
| 25:24 | Palette entry format for indexed textures: 0 RGB565, 1 ARGB1555, 2 ARGB4444 |

MIN_BLEND values:

| Value | Filter |
|-------|--------|
| 0 | Nearest |
| 1 | Linear |
| 2 | Nearest within the nearest mip level |
| 3 | Linear within the nearest level |
| 4 | Nearest, blended between levels |
| 5 | Trilinear |

MAG_BLEND: odd values are linear. The secondary unit also treats 2 as linear.
Unreal under Mac OS 9 writes 5 **[8]/[D]**.

Wrap modes: 0 wrap, 1 mirror, 2 clamp, 3 border colour (from
*_TEXTURE_BORDER_COLOR_C).

### TEX_SIZE_PITCH_C 0x1CB8 **[8]**

Bits 15:0 are for the primary unit, bits 31:16 for the secondary. Each half
is:

| Bits | Field |
|------|-------|
| 3:0 | TEX_PITCH: log2 width of the largest level (also its pitch, in texels) |
| 7:4 | TEX_SIZE: log2 of the larger side |
| 11:8 | TEX_HEIGHT: log2 height |
| 15:12 | TEX_MIN_SIZE: log2 of the smallest level present |

### Mip level layout **[8]** (Mac OS X calls it "reverse order")

The 11 offset registers (`*_TEX_0..10_OFFSET_C`) hold one level each. The
**largest** level is in slot `TEX_SIZE − TEX_MIN_SIZE`, and each lower slot
holds the next smaller level, down to TEX_MIN_SIZE in slot 0.

So a full chain from 256×256 to 1×1 puts the base level in `TEX_8_OFFSET`
and the 1×1 level in `TEX_0_OFFSET`. Each level's pitch equals its width.

Only slots the filter can actually select are fetched. Unreal under Mac OS 9
leaves garbage such as 0x4EBA0098 in PRIM_TEX_0_OFFSET_C (which looks like a
system memory address), because it doesn't mipmap **[D]**.

Offset bits 25:0 are the card address (bit 25 set means GART memory). Bits
31:30 select a tiled layout **[8]**.

**LOD** is log2 of the larger screen-space footprint, measured on the largest
level **[A]** (standard). There is no LOD bias.

### Texture datatypes

| Value | Format | Bits |
|-------|--------|------|
| 0 | VQ / S3TC compressed (not emulated) | — |
| 1 | CI4 (nibble order **[A]**: low nibble is the even texel) | 4 |
| 2 | CI8 | 8 |
| 3 | ARGB 1555 | 16 |
| 4 | RGB 565 | 16 |
| 5 | RGB 888 | 24 |
| 6 | ARGB 8888 | 32 |
| 7 | RGB 332 | 8 |
| 8 | Y8 | 8 |
| 9 | RGB8 / grey | 8 |
| 10 | Indexed, 16-bit texels **[A]** | 16 |
| 11 / 12 | VYUY / YVYU 4:2:2 (the emulation uses luma only) | 16 |
| 13 | **AI88**: alpha in the high byte, intensity in the low byte. Undocumented; 86Box found it in Serious Sam **[8]** | 16 |
| 14 | AYUV 4444 (treated as ARGB8888) | 32 |
| 15 | ARGB 4444 | 16 |

### Texture palette

Palettised textures read a 256-entry palette. Each entry's low 16 bits hold a
colour in the format given by bits 25:24 of the unit's control register.

Two ways to load it:

- the CCE LOAD_PALETTE packet (0x2C);
- registers 0x1968 / 0x196C: write a start index to 0x1968, then stream
  entries to 0x196C, which auto-increment **[D]**.

Mac OS 9 writes **0x2000** to 0x1968 before streaming. The emulation takes the
entry number from the low byte **[A]**.

On chip family 0x512, Mac OS 9 rewrites TEXTURE_CLR_CMP_MSK_C after every
palette entry, presumably an erratum workaround **[D]**.

## Texture combiners **[8]**

PRIM_TEXTURE_COMBINE_CNTL_C (0x1CB4) and SEC_TEX_COMBINE_CNTL_C (0x1D04):

| Bits | Field |
|------|-------|
| 3:0 | COMB_FCN (colour function) |
| 7:4 | COLOR_FACTOR |
| 8 | COMB_FCN_MSB (changes function 0) |
| 13:10 | INPUT_FACTOR |
| 17:14 | ALPHA_COMB_FCN |
| 21:18 | ALPHA_FACTOR (7 = 1 − texture α, else texture α) |
| 27:25 | INPUT_FACTOR_ALPHA (1 = constant α, else input α) |

The "input" to the primary unit is the interpolated vertex colour. For the
secondary unit it's the primary unit's output.

COLOR_FACTOR (*f*):

| Value | Factor |
|-------|--------|
| 0 | Constant colour |
| 1 | 1 − constant |
| 4 | Texture |
| 5 | 1 − texture |
| 6 | Texture α |
| 7 | 1 − texture α |
| 8 | Input |

INPUT_FACTOR (*x*):

| Value | Factor |
|-------|--------|
| 2 | Constant colour |
| 3 | Constant α |
| 4 / 8 | Input (interpolated, or previous stage) |
| 5 / 9 | Input α |

COMB_FCN (the same table is used for the alpha function):

| Value | Result |
|-------|--------|
| 0 | "Disable": output the texture colour. With COMB_FCN_MSB, output max(*f* − *x*, 0) |
| 1 | *f* |
| 2 | *x* |
| 3 | *f* · *x* (modulate) |
| 4 | 2·*f*·*x* |
| 5 | 4·*f*·*x* |
| 6 | *f* + *x* |
| 7 | *f* + *x* − 0.5 |
| 8 / 9 / 10 / 12 | Blend *f* and *x* by α: 8 uses input α, 9 texture α, 10 constant α |
| 11 | *f* + *x*·(1 − texture α) |
| 13 | *f*·(1 − texture α) + *x* |
| 14 | 2·(*f* + *x* − 0.5) |
| 15 | Blend by the constant colour per channel |

Results are clamped to 0–1.

ALPHA_COMB_FCN 0 outputs texture α in the first stage and input α in the
second.

## Per-pixel order (as emulated)

1. 3D scissor
2. Stipple
3. Aux scissors
4. Depth read and test. With no stencil, a failed test exits here
5. Each enabled texture unit:
   - sample, then apply the alpha mask, the chroma key and CLR_CMP_FCN_3D
     (unit 0, nearest texel);
   - force α if TEX_MAP_ALPHA_IN_TEXTURE is clear;
   - combine
6. Specular add (if SPEC_LIGHT_ENABLE)
7. **Vertex fog**: `colour = colour·f + FOG_COLOR·(1 − f)`, where *f* is the
   interpolated SPEC_F and 1.0 means no fog. Mac OS 9 computes the factor on
   the CPU for every fog mode **[D]**
8. Alpha test
9. Stencil test and update
10. Blend
11. Pack to the destination format, apply PLANE_3D_MASK_C & DP_WRITE_MASK,
    write
12. Z write

The destination is DST_PITCH_OFFSET_C (same packing as DST_PITCH_OFFSET). Its
format is DP_GUI_MASTER_CNTL_C bits 11:8.

The chroma key uses TEXTURE_CLR_CMP_CLR_C / _MSK_C. It rejects texels equal to
the key under the mask, testing the nearest texel of unit 0 before the alpha
override **[8]**.

## Depth and stencil **[8]/[A]**

### Z_STEN_CNTL_C 0x1C98

| Bits | Field |
|------|-------|
| 2:1 | Z_PIX_WIDTH: 0 = 16-bit Z; 1 = 24-bit Z with 8-bit stencil in the top byte; 2 = 32 bits holding 24-bit Z in the upper three bytes |
| 6:4 | Z test function (encoding above) |
| 14:12 | Stencil function |
| 18:16 | Stencil-fail operation |
| 22:20 | Z-pass operation |
| 26:24 | Z-fail operation |

The stencil operation positions are as on Radeon **[A]**.

Stencil operations: 0 keep, 1 zero, 2 replace, 3 increment (saturating),
4 decrement (saturating), 5 invert. Stencil only works with Z_PIX_WIDTH = 1.

`STEN_REF_MASK_C` 0x1D40: reference in 7:0, compare mask in 23:16, write mask
in 31:24.

`Z_OFFSET_C` 0x1C90 is the buffer base.

`Z_PITCH_C` 0x1C94 is the pitch in units of 8 pixels. Bit 0 is ignored. Bit 16
means a tiled Z buffer.

Z values are the interpolated vertex Z (0–1) scaled to the buffer's range
**[A]**.

## Things that look wrong but match real hardware **[H]**

Don't chase these.

### Unreal, Mac OS 9, RAVE renderer

Unreal's RAVE renderer is buggy and causes problems on real hardware. These
issues should happen both on the retail release and when the game is patched
to version 224.

- Fog effects are missing. Obvious examples are: moat in the intro flyby,
  ventilation shafts in Vortex Rikers, crossroads/mining area in Depths of
  Rrajigar, final teleporter in Outpost 3J, Illumination mothership generator
  explosion, various areas in The Darkening.
- Glitching/misshaped polygons: as the camera turns past the castle tower in the
  flyby, a large mis-shaded polygon appears to the right of the screen, and, to the
  right of that, there will be another polygon with a glitching texture.
- Incorrect alpha rendering: leaves on trees in NyLeve's Falls have a black outline.
