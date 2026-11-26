# The Yosemite board (Power Mac G3 Blue & White / G4 "Yikes!")

Notes on the machine side, limited to what matters for the graphics card and
for getting the machine to boot in an emulator.

## Overview

The board is a "New World" Gossamer-family design. The Power Mac G4 (PCI
Graphics), "Yikes!", PowerMac1,2, uses the same board with a G4 CPU. The B&W G3
is PowerMac1,1.

| Part | Details |
|------|---------|
| CPU | PowerPC 750 (G3) at 300/350/400/450 MHz on a **100 MHz** bus. The timebase counts bus clocks / 4 |
| Memory controller / host bridge | Motorola MPC106 "Grackle" |
| PCI-PCI bridge | DEC 21154 (1011:0026 rev 02). Secondary bus 1 |
| I/O | Paddington (Heathrow-family PCI I/O ASIC), behind the 21154 |
| ATA | Paddington ATA for the optical drive. **CMD646** (1095:0646 rev 07) for the hard disks |
| USB | OPTi 82C861 (1045:C861 rev 10) |
| FireWire | TI PCILynx (104C:8000) |
| Audio | Burgundy codec |
| ADB / power | Cuda. The Yikes! board here has a 341S0262 (Cuda 3.02) |
| RAM | 4 PC100 DIMM slots, max 1 GiB. SPD at I2C 0x50–0x53 on the Cuda's I2C bus (unverified for this board) |
| Graphics | **66 MHz slot J12** on the host bus, normally holding the Rage 128 GL |
| Expansion | Three 33 MHz slots J9–J11 behind the 21154 |

## PCI layout

From an ioreg dump of a PowerMac1,2:

| Bus | Dev.fn | Device | IDs | IRQ |
|-----|--------|--------|-----|-----|
| 0 | 00.0 | MPC106 | 1057:0002 | |
| 0 | 0D.0 | DEC 21154 bridge | 1011:0026 rev 02 | |
| 0 | 10.0 | Slot J12 (66 MHz): ATI Rage 128 GL | 1002:5245 | 22 |
| 1 | 00.0 | TI PCILynx FireWire | 104C:8000 | 21 |
| 1 | 01.0 | CMD646 ATA | 1095:0646 rev 07 | 26 |
| 1 | 02.0 | Slot J11 | | 23 |
| 1 | 03.0 | Slot J10 | | 24 |
| 1 | 04.0 | Slot J9 | | 25 |
| 1 | 05.0 | Paddington | 106B:0017 | |
| 1 | 06.0 | OPTi 82C861 USB | 1045:C861 rev 10 | 28 |

The IRQ numbers are Paddington interrupt inputs. Each slot has **one**
interrupt line shared by INTA#–INTD#. A card that drives more than one pin
needs a wired-OR.

## Graphics card notes

- **The boot ROM has no Rage 128 driver.** The card's FCode ROM (113-57407-204)
  builds the display node and supplies the Mac OS ndrv.
  - A PC graphics card has no FCode, so Open Firmware won't drive it. Only Mac
    editions of cards are useful in J12.
  - Some MAME PC PCI card models also assume x86 system details.
- 3dfx shipped Mac editions of the Voodoo3 with FCode. They're possible future
  options for J12.
- The Rage 128 also works in a Beige G3 PCI slot under Mac OS 9 (from hard
  disk). That's how the BAR2 question in the PCI notes came up.

## Emulation pitfalls found while bringing the board up

- **Boot ROM:** only the Yikes! ROM (1.2f2, built 1999-08-19,
  `powermac12_1.2f2.bin`, CRC32 `35be26bf`) is dumped so far.
  - It reports the machine as PowerMac1,2, so Mac OS believes it's a Yikes!.
  - Watch for G4-only SPR accesses when running it on a 750.
  - A genuine PowerMac1,1 ROM is still wanted.
- **Cuda handshake timing:** Cuda and the PPC poll each other's VIA handshake
  lines (TREQ, BYTEACK, TIP, shift register clock/data).
  - Without tight interleaving the firmware black-screens.
  - Several MAME Mac drivers need this; pmac6200 shows Sad Macs without it.
  - Running at perfect quantum all the time costs most of the recompiler's
    speed. The working approach: boost to the finest quantum for ~10 ms after
    every handshake edge or poll, and abort the CPU's timeslice on PPC-side
    handshake changes. This reaches the desktop faster than perfect quantum
    throughout.
- **CMD646:** the generic PCI IDE model can't stand in for it. It needs its own
  device, and the B&W firmware needs programming interface **0x8F** (native
  mode) to read it at all. Its bus-master DMA goes through Grackle's PCI memory
  space, where system RAM is mapped.
- **The OPTi USB controller must be present.** Without it the firmware stops
  at the grey crosshatch screen.
- **ADB** keyboard and mouse must be the default input devices.
- The generic PCI bridge model reads the 21154's secondary status as 0xFFFF.
  It doesn't implement the DEC-specific config registers at 0x40–0xFF.
- The CMD646 boot is slow on this machine. The iMac G3 driver doesn't share
  that problem.
- DIMM sizing: the iMac ROM probes each module through two Grackle banks. It
  hasn't been checked whether this ROM does the same.
- The emulated driver currently uses Cuda firmware 341S0060. The Yikes! board
  has a 341S0262 (3.02). This needs reconciling.
