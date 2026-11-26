# How the Mac drivers use the card

Behaviour seen from Apple/ATI drivers. This is what an emulated card has to
satisfy, and a source of hints about the hardware. Everything here is **[D]**
unless marked otherwise.

## The Apple OEM option ROM

Apple OEM Rage 128 GL:

- `ATY,Rom#` 113-57407-204
- `ATY,Card#` 109-57400-00
- `ATY,Fcode` 1.65

The ROM is a **single FCode image**: PCIR code type 1, with no x86 code. It is
0x12800 bytes (74 KiB) and carries the Mac OS ndrv `.Display_Rage128`
version 1.0.1.

Dump: CRC32 `ab00bb87`, SHA1 `6c0153db8e0316c1c59a42b8f6c4d4c312c598b0`.
The dump ends at the image length given in the PCIR header. Whether anything
follows it in the EEPROM hasn't been checked.

Open Firmware sizes the expansion ROM BAR at 128 KiB on real hardware, which
matches the manual's BIOS_ROM field **[M]**.

The B&W G3 / Yikes! boot ROM has **no** Rage 128 driver of its own. The card's
FCode builds the display node and supplies the ndrv. So an emulated card
without this ROM gets no display under Open Firmware or Mac OS.

## Mac OS 9

There are three separate pieces of ATI code.

### 1. The ndrv (display driver)

There are two versions: the one in the card ROM, and the one installed by the
"ATI Driver Update" extension.

- **Monitor sense:** the ndrv probes the MONID pins (AppleSense) and DDC. With
  the emulated monitor it settles on **640×480**, since the sense code and the
  EDID only offer that.
- Around the sense, the ROM ndrv saves and restores **CRTC_GEN_CNTL byte 3
  with an unaligned 32-bit read**. If the host bridge or emulator mishandles
  unaligned access to a byte-addressable register, CRTC_EN ends up clear and
  the screen blanks. This was seen under MAME and judged a MAME core issue.
- **Cursor:** both ndrvs put the cursor image at VRAM offset 64. For a cursor
  clipped at the top they add a row offset to CUR_OFFSET: the ROM ndrv adds
  yoff·2, the ATI Driver Update one (yoff·16) & 0xFF. That only works if
  CUR_OFFSET's low 4 bits are ignored.

### 2. "ATI Graphics Accelerator" (QuickDraw acceleration)

All of its drawing goes through CCE packets via the resource manager.

- **Fills** use DP_SRC_SOURCE = memory with ROP 0xF0 (PATCOPY). SRC_X/SRC_Y
  are left wherever the last blit ended. The hardware must ignore the source,
  including the source scissor, when the ROP doesn't use it.
- **Transparent blits** use TRANS_BITBLT with CLR_CMP_CNTL = 0x0100_0005, keyed
  on the port's background colour. The GMC leaves CLR_CMP_CNTL_DIS clear. The
  compare stays enabled for following packets.
- **8×8 mono brush paints** show the CNTL header brush layout: BG then FG for
  an FG_BG brush, FG only for FG_LA, then the pattern, then BRUSH_Y_X if GMC
  bit 31 is set.
- **Host data:**
  - mono (glyphs, icons) is sent unswapped whatever HOST_BIG_ENDIAN_EN says;
  - colour is sent as big-endian pixels, with HOST_BIG_ENDIAN_EN set at 16 and
    32 bpp only;
  - CNTL_HOSTDATA_BLT always has the destination-clip GMC bit set.
- **POLYLINE** points are packed like DST_Y_X. The driver sets DP_CNTL
  DST_LAST_PEL around each polyline.

### 3. "ATI Rage 128 3D Accelerator" (RAVE engine)

This is a PEF extension, built 2002-09-03. It registers "ATI 3DRage QD3D Rave
Engine" and exports `TtRegister` / `TtDeRegister`.

- **All** VRAM/AGP allocation and command submission go through
  **ATIResManager** (`ARM_*`, `ARMQSSubmitList`). So 3D rendering is entirely
  CCE command buffers:
  - written little-endian;
  - at most 16384 dwords;
  - odd lengths padded with a Type 2 packet.
- Chip family comes from `ARM_GetVRAMInfo` field 52. Families seen: 0x500,
  0x510, 0x512, 0x520, 0x550, 0x560, 0x570. The driver rejects anything below
  0x500.
- Packets used:
  - Type 0 bursts (with ONE_REG);
  - 3D_RNDR_GEN_PRIM (0x25), with inline vertices;
  - 3D_RNDR_GEN_INDX_PRIM (0x23), with a vertex buffer and indices;
  - CNTL_HOSTDATA_BLT (0x94), CNTL_PAINT_MULTI (0x9A), CNTL_BITBLT_MULTI (0x9B);
  - CNTL_SCALING (0x96), 16 dwords, used to present frames rendered at a lower
    resolution.
- The full context load order is in [3d-engine.md](3d-engine.md).
- **Per-family workarounds:**
  - family 0x500 only: 35 dummy `FLUSH_1 = 0` writes before every primitive;
  - families 0x510/0x512/0x520: set bit 3 of register **0x07A4** (unnamed; not
    in the manual);
  - family 0x512: rewrites TEXTURE_CLR_CMP_MSK_C after every texture palette
    entry.
- **Texture palette:** 0x1968 = 0x2000, then entries streamed to 0x196C.
- **Polygon stipple:** BRUSH_Y_X is rewritten per primitive as the stipple
  origin (X 4:0, Y 20:16). Rows are loaded bottom first.
- **Vertex formats:**
  - untextured: 0x6, 0x26, 0x36;
  - textured: 0x85, 0x87, 0x95, 0x97, 0xA5, 0xB5, …;
  - with a second texture: | 0x300.
- **Vertex fog** is computed on the CPU for every fog mode and passed in
  SPEC_F.
- **Page flips** go through the CCE: WAIT_UNTIL, CRTC(2)_GUI_TRIG_VLINE,
  CRTC(2)_OFFSET(_CNTL), DST_OFFSET.
  - WAIT_UNTIL values seen: 0x1 (CRTC_OFFSET), 0x4 (FE_CRTC_VLINE),
    0x8 (CRTC_VLINE), 0x400 (CMDFIFO).
  - The CRTC2 versions have bit 31 set. The VR/GL manual documents no CRTC2.
- **SURFACE windows** (0x0B00–0x0B3C) are programmed. Mac OS 9 is the only
  driver seen doing this.

Games checked against a real card: **Unreal** (RAVE renderer, retail and patc
224) and **Nanosaur**. See the "looks wrong but matches hardware" and
"performance reference" sections of [3d-engine.md](3d-engine.md).

Under Unreal:

- MAG_BLEND = 5;
- unused mip offset slots are left full of garbage;
- masked textures use TEX_CNTL_C bit 13;
- most world geometry is drawn with blending on and ONE/ZERO factors.

Under Nanosaur, the leaves are cut out by TEX_CNTL_C bit 13 on 1555 textures,
with blending and the alpha test off.

### Resource manager CCE handling

- Free PIO entries are polled at **0x0740** bits 7:0 (or 8:0) before each
  submission. It needs at least 6, or the size of the submission.
- 0 is written to 0x0740 on CCE reset.
- PIO packets go to consecutive addresses from 0x1000, not alternating
  EVEN/ODD.
- On a **Beige G3** (card in a PCI slot), the same driver writes the FIFO at
  BAR2+0x5000 and HOST_DATA at BAR2+0x57C0–0x57DC, byte-reversed. See the BAR2
  question in [pci-and-apertures.md](pci-and-apertures.md).

## Mac OS X 10.4 (ATIRage128 kext, GL, 2D and DVD drivers 1.4.4)

- **Device match:** RE, RF, RK, RL, PE, PF, PR, TR, plus the Mobility LE/LF.
  It won't load on a PD.
- **Bus master enable** is set through the config mirror at MMIO 0x0F04.
- **CCE mode 7:** PIO primary queue, bus-mastered indirect and vertex buffers.
  - Before each submission it waits for ≥ 6 free entries in PM4_STAT. It
    **silently drops** the buffer if it never sees them.
  - INDOFF/INDSIZE are sent as one two-register Type 0 packet through the PIO
    FIFO.
  - Indirect-buffer addresses are plain GART offsets, masked to 25 bits.
- On PCI cards it sets **BM_CHUNK_0_VAL BM_PM4_RD_FORCE_TO_PCI**. GUI DMA also
  sets GLOBAL_FORCE_TO_PCI.
- **GART memory** is at card address 32 MiB (the same as Linux).
- Writes a **fence** stamp to GUI_SCRATCH_REG0 after each buffer.
- Writes 0x122 to 0x1744 (PC_GUI_MODE?).
- **Cursor:** a 32-bit ARGB image drawn with the scaler registers (SCALE_*,
  source format in 0x1A20). It's blended with MISC_3D_STATE_CNTL_REG =
  0x0054_0100 and clipped with SC_TOP_LEFT_C / SC_BOTTOM_RIGHT_C.
- **Overlay:** spins on OV0_REG_LOAD_CNTL LOCK_READBACK before programming.
- **Readback:** copies surfaces from VRAM to system memory with bus-master GUI
  DMA (descriptor table at 0x0A50, busy in 0x0A10 bit 27). See the 2D notes.
- **Textures** use the "reverse order" mip layout.
- The **DVD player** uses the sub-picture (0x540–0x588) and motion
  compensation (0x1840–0x1898, IDCT at 0x1F80–0x1FBC) blocks. None of this is
  decoded.

## Other drivers worth disassembling

ATI's Windows 9x/NT4/2000/XP "Rage 128 driver 1.02" CD. Its **OpenGL ICD
carries ATI's own register name table**, which is the source of several names
used here. The NT display driver is the source of:

- the POLYSCANLINES layout;
- the 0x1F packet marker;
- the GDI GradientFill path, which programs RED_X_INC … ALPHA_START
  (0x1A40–0x1A6C) and a direct (non-CCE) setup-engine trigger that hasn't been
  decoded.
