# CCE (PM4) command processor

The CCE is a microcoded command processor. The driver loads its microcode,
then feeds it "PM4" packets, either:

- by **PIO**, writing dwords into a FIFO window in the register aperture; or
- by **bus mastering**, from a ring buffer, plus indirect buffers launched from
  it, in VRAM or in system memory through AGP or the PCI GART.

**None of this is in the register manual**, because its CCE chapter is
missing **[M]**. Everything here comes from drivers (X.Org/Linux r128 DRM, Mac
OS 9 / Mac OS X, ATI's NT driver) and from what made them work **[D]/[A]**.

On Mac OS 9 **all** 3D, and nearly all accelerated 2D, goes through the CCE
via ATI's resource manager (`ATIResManager`, `ARM_*` calls, `ARMQSSubmitList`)
**[D]**. Command buffers there are:

- little-endian;
- at most 16384 dwords;
- padded to an even length with a Type 2 packet (0x80000000).

## Microcode

| Register | Notes |
|----------|-------|
| `PM4_MICROCODE_ADDR` 0x7D4 | Write address (8 bits) |
| `PM4_MICROCODE_RADDR` 0x7D8 | Read address |
| `PM4_MICROCODE_DATAH` 0x7DC | High dword |
| `PM4_MICROCODE_DATAL` 0x7E0 | Low dword; writing it advances the address |

The store is 256 words of 2 dwords. Reading DATAL also advances the read
address **[D]**.

An emulator can accept the microcode and interpret packets directly. The
packet formats below are what ATI's microcode implements.

`PM4_MICRO_CNTL` 0x7FC bit 30, FREERUN, starts the microengine **[D]**.

## PM4_BUFFER_CNTL 0x704 **[D]**

| Bits | Field |
|------|-------|
| 31:28 | Mode (below) |
| 27 | NO_UPDATE: don't write the read pointer back to memory |
| 4:0 | Ring size: 2^n *qwords* (2^(n+1) dwords) |

Modes:

| Mode | Name (r128_reg.h) | Primary FIFO depth |
|------|-------------------|--------------------|
| 0 | NONPM4: CCE off | — |
| 1 | 192PIO | 192 |
| 2 | 192BM | 192 |
| 3 | 128PIO_64INDBM | 128 |
| 4 | 128BM_64INDBM | 128 |
| 5 | 64PIO_128INDBM | 64 |
| 6 | 64BM_128INDBM | 64 |
| 7 | 64PIO_64VCBM_64INDBM | 64 |
| 8 | 64BM_64VCBM_64INDBM | 64 |
| 15 | 64PIO_64VCPIO_64INDPIO | 64 |

PIO modes are 1, 3, 5, 7 and 15. Bus-master ring modes are 2, 4, 6 and 8.

**Mac OS X runs mode 7**: PIO for the primary queue, with bus-mastered vertex
and indirect buffers **[D]**.

## Free-entry and status registers

| Register | Notes |
|----------|-------|
| `PM4_STAT` 0x7B8 | Bits 11:0 = free primary FIFO entries. Bit 31 = busy **[D]** |
| 0x740 (name unknown) | Mac OS 9's resource manager polls bits 7:0 (or 8:0) here for free PIO FIFO entries before each submission. It waits for at least 6, or for the size of what it's about to send. It writes 0 here when it resets the CCE **[D]** |

Mac OS X waits for **at least 6 free entries in mode 7** before each
submission. **If it never sees them, it silently drops the buffer** **[D]**.
Whatever an emulator reports here must never read lower than that while idle.

## PIO FIFO window

Linux and Mac OS X alternate writes between `PM4_FIFO_DATA_EVEN` 0x1000 and
`PM4_FIFO_DATA_ODD` 0x1004. The Mac OS 9 resource manager instead writes
packets to **consecutive** addresses from 0x1000 (up to 0x1014 seen) **[D]**.

The emulation takes any write in 0x1000–0x11FF as a FIFO write. The size is
assumed, after the Radeon's 512-byte primary queue aperture at the same
address **[A]**.

Also see the BAR2 note in [pci-and-apertures.md](pci-and-apertures.md). On a
Beige G3, Mac OS 9 writes the FIFO at BAR2+0x5000.

## Ring buffer (bus-master modes) **[D]**

| Register | Notes |
|----------|-------|
| `PM4_BUFFER_OFFSET` 0x700 | Ring base (card address) |
| `PM4_BUFFER_DL_WPTR` 0x714 | Write pointer, in dwords. Writing it kicks the CCE |
| `PM4_BUFFER_DL_RPTR` 0x710 | Read pointer, in dwords |
| `PM4_BUFFER_DL_RPTR_ADDR` 0x70C | Where to write RPTR back after fetching (unless NO_UPDATE) |
| `PM4_BUFFER_WM_CNTL` 0x708 | Watermarks |
| `PM4_BUFFER_ADDR` 0x7F0 | Reads OFFSET + RPTR·4 **[A]** |

## Indirect buffers **[D]**

Write `PM4_IW_INDOFF` 0x738 (address), then `PM4_IW_INDSIZE` 0x73C (size in
dwords, 23 bits). Writing INDSIZE fires the buffer.

Mac OS X sends INDOFF and INDSIZE as **one two-register Type 0 packet**
through the PIO FIFO. So an indirect buffer can start in the middle of
processing a Type 0 packet, and the outer packet's state has to survive it.

Mac OS X gives indirect-buffer addresses as plain GART offsets (masked to 25
bits). Linux ORs in the 32 MiB GART window base. Taking command-fetch
addresses as GART offsets either way handles both **[A]**.

## Bus-master address translation

Card addresses below 32 MiB are VRAM. Addresses 32–64 MiB are the GART
window. Command fetch (ring and indirect) always comes from system memory.

How each bus-master client reaches system memory **[D]/[A]**:

- **AGP:** `PCI_GART_PAGE` bit 0 set. Bus address = `AGP_BASE` (0x170) +
  `AGP_APER_OFFSET` (0x178) + offset into the window. The chipset's GART
  translates from there.
- **PCI GART:** `PCI_GART_PAGE` bit 0 clear. `PCI_GART_PAGE` bits 31:12 point
  to a flat table of little-endian dwords, one per 4 KiB page. Each entry is
  the page's bus address in bits 31:12. The manual describes this table as up
  to 8192 entries and bit 0 as "disable" **[M]**.
- **`BM_CHUNK_0_VAL` 0xA18** overrides the routing per client:

  | Bit | Override |
  |-----|----------|
  | 21 | BM_PTR_FORCE_TO_PCI: the RPTR writeback address is a raw bus address. Linux programs it this way for PCI cards |
  | 22 | BM_PM4_RD_FORCE_TO_PCI: command fetch uses the PCI GART even when AGP is selected. **Mac OS X sets this on PCI cards** |
  | 23 | BM_GLOBAL_FORCE_TO_PCI: everything uses the PCI GART. GUI DMA sets this too |

The CPU can only rewrite GART entries between command buffers. So caching
translations for the length of one buffer or primitive is safe.

## Packet formats **[D]**

Every packet starts with a header dword. Bits 31:30 give the type.

### Type 0: register writes

```
31:30  0
29:16  count: count+1 data dwords follow
15     ONE_REG: all data goes to the same register (else consecutive registers)
10:0   register dword index (register byte address >> 2)
```

### Type 1: two register writes

```
31:30  1
21:11  second register index
10:0   first register index
```

Two data dwords follow.

### Type 2: filler

`0x80000000`. No payload. Used to pad buffers to an even length.

### Type 3: commands

```
31:30  3
29:16  count: count+1 payload dwords follow
15:8   opcode (bit 7 = the "CNTL_" form, which starts with GUI master control state)
```

## Type 3 opcodes

The opcode is shown without bit 7. The `CNTL_` form is opcode | 0x80. Payload
layouts are as implemented and confirmed against driver output. Names follow
the r128 DRM where it has them.

| Op | Name | Notes |
|----|------|-------|
| 0x10 | NOP | |
| 0x11 | PAINT | After the header: pairs of (top << 16 \| left), (bottom << 16 \| right), **bottom-right exclusive** **[A]** |
| 0x12 | BITBLT | Triples of (srcX << 16 \| srcY), (dstX << 16 \| dstY), (w << 16 \| h) |
| 0x14 | HOSTDATA_BLT | (y << 16 \| x), (h << 16 \| w), dword count, then the data |
| 0x15 | POLYLINE | Points as (y << 16 \| x), i.e. DST_Y_X packing. Each segment's last point is drawn by the next one. The final point is drawn only if DP_CNTL DST_LAST_PEL is set. Mac OS 9 sets it around each polyline **[D]** |
| 0x16 | SCALING | Header, then an 11-dword scaler block (below). Mac OS 9 presents RAVE frames rendered at a lower resolution with CNTL_SCALING (0x96, 16 dwords) **[D]** |
| 0x17 | TRANS_SCALING | Header, CLR_CMP_CNTL, CLR_CMP_CLR_SRC, CLR_CMP_CLR_DST, then the scaler block |
| 0x18 | POLYSCANLINES | Per scanline: (span count << 16 \| y), then spans as (x_right << 16 \| x_left), **right edge exclusive** **[A]**. As ATI's NT driver sends them |
| 0x19 | NEXT_CHAR | (y << 16 \| x), a control dword, glyph bitmaps. Not emulated |
| 0x1A | PAINT_MULTI | Pairs of (x << 16 \| y), (w << 16 \| h). Note the different packing from PAINT |
| 0x1B | BITBLT_MULTI | As BITBLT. The microcode picks copy direction; the emulation picks the overlap-safe direction itself **[A]** |
| 0x1C | TRANS_BITBLT | Header, CLR_CMP_CNTL, CLR_CMP_CLR_SRC, CLR_CMP_CLR_DST, then BITBLT triples **[A]** |
| 0x1D | PLY_NEXTSCAN | More scanlines for the previous POLYSCANLINES |
| 0x1E | SET_SCISSORS | (top << 16 \| left), then (bottom << 16 \| right) inclusive |
| 0x1F | ? | One dword (value 1) closing each NT driver batch. A marker; ignored |
| 0x20 | 3D_SAVE_CONTEXT | Ignored |
| 0x21 | 3D_PLAY_CONTEXT | Ignored |
| 0x23 | 3D_RNDR_GEN_INDX_PRIM | Vertex buffer address, vertex buffer size, VC_FORMAT, VC_CNTL, then indices (two 16-bit indices per dword, low half first) when walk = 1. See [3d-engine.md](3d-engine.md) |
| 0x25 | 3D_RNDR_GEN_PRIM | VC_FORMAT, VC_CNTL, then the vertices inline (walk = 3) |
| 0x28 | ? | A vertex-buffer draw built by the Windows D3D driver. Not decoded |
| 0x2C | LOAD_PALETTE | Texture palette. The first dword is taken as the start index, then entries **[A]** |
| 0x2D | PURGE | Ignored |
| 0x2E | NEXT_VERTEX_BUNDLE | Not emulated |
| 0x31 | ? | One dword (0, 0x40, 0x100 or 0x400) around flushes and DVD motion compensation. Ignored |

Mac OS 9's packet set, from the RAVE engine **[D]**:

- PKT0 register bursts, including ONE_REG;
- 3D_RNDR_GEN_PRIM (0x25) and 3D_RNDR_GEN_INDX_PRIM (0x23);
- CNTL_HOSTDATA_BLT (0x94), CNTL_PAINT_MULTI (0x9A), CNTL_BITBLT_MULTI (0x9B);
- CNTL_SCALING (0x96).

### The CNTL_ header

The `CNTL_` forms start with GUI master control and the state it selects, in
this order **[A]** (consistent with NT and Mac OS 9 output):

1. GMC (DP_GUI_MASTER_CNTL).
2. SRC_PITCH_OFFSET, if GMC bit 0 is set and the op has a source.
3. DST_PITCH_OFFSET, if GMC bit 1 is set.
4. SRC_SC_BOTTOM_RIGHT, if GMC bit 2 is set and the op has a source.
5. SC_TOP_LEFT then SC_BOTTOM_RIGHT, if GMC bit 3 is set.
6. Brush, by GMC brush type:
   - mono FG/BG: background colour, foreground colour, then the pattern;
   - mono FG/LA: foreground colour only, then the pattern;
   - pattern size: 2 dwords for 8×8, 1 for 1×8 and 32×1, 32 for 32×32;
   - 8×8 colour: 16 × bytes-per-pixel dwords;
   - 1×8 colour: 2 × bytes-per-pixel dwords;
   - solid: one colour;
   - none: nothing.
7. BRUSH_Y_X, if GMC bit 31 is set.
8. DP_SRC_FRGD_CLR then DP_SRC_BKGD_CLR, for ops whose source is mono (blits
   with SRC_DATATYPE ≠ 3, and HOSTDATA_BLT).

ATI's NT driver sends CNTL_BITBLT with all four GMC bits 0–3 set, in 9
dwords. The Mac drivers set the destination-clip bit on CNTL_HOSTDATA_BLT
**[D]**.

The 8×8 mono brush layout (BG+FG for FG_BG, FG only for FG_LA) was read
directly from Mac OS 9 paint packets **[D]**. The other mono sizes are assumed
to match.

### Scaler block (SCALING / TRANS_SCALING) **[8]**

Eleven dwords, after 86Box:

| # | Contents |
|---|----------|
| 0 | MISC_3D_STATE_CNTL_REG. Bit 10 replicates pixels instead of filtering. Bits 23:16 are the blend functions |
| 1 | TEX_CNTL_C. Bit 9 enables alpha blending |
| 2 | A combine control (unused) |
| 3 | Source datatype (texture format codes) |
| 4 | Source offset |
| 5 | Source pitch, in units of 8 pixels, in bits 8:0 |
| 6 | Source start in **half pixels**: Y in 29:16, X in 13:0 |
| 7 | X increment, 4.12, in bits 19:4 |
| 8 | Y increment, 4.12, in bits 19:4 |
| 9 | (dst_x << 16) \| dst_y |
| 10 | (height << 16) \| width |

The destination, clipping, ROP, colour compare and write mask come from the
GMC header, as for a blit.

The Mac OS 9 driver leaves the colour compare enabled after a TRANS_BITBLT, so
a following SCALING inherits it **[D]**.
