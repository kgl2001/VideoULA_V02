# VideoULA V02 — BBC Micro Video ULA Replacement
## 64-pin Xilinx XC9572XL CPLD Version

This repository contains the hardware design and firmware for the VideoULA V02, a drop-in replacement for the VC2069 Video ULA (IC6) in the Acorn BBC Micro Model B and Master 128.

---

## Project Overview

The original BBC Micro Video ULA (IC6) is a custom ASIC that is no longer manufactured. Failed or missing ULAs result in a non-functional machine. This project provides a modern replacement using a CPLD and a small external SRAM for palette storage.

There are three hardware versions of this replacement, each in a separate repository:

| Version | CPLD | Macrocells | Tool | Repository |
|---|---|---|---|---|
| V01 | Xilinx XC9572XL-10-VQ44 | 72 | ISE 14.7 | [VideoULA_V01](https://github.com/kgl2001/VideoULA_V01) |
| **V02** | Xilinx XC9572XL-10-VQ64 | 72 | ISE 14.7 | This repo |
| V03 | Atmel ATF1504AS-10AU44 | 64 | Quartus 13.0 SP1 | [VideoULA_V03](https://github.com/kgl2001/VideoULA_V03) |

All three versions share the same 28-pin DIP footprint as the original ULA and are pin-compatible with the BBC Micro motherboard.

---

## V02 Hardware

- **CPLD**: Xilinx XC9572XL-10-VQ64 (72 macrocells, 3.3V core, 5V tolerant I/O)
- **SRAM**: ISSI IS61LV256AL-10TLI (32K×8, 3.3V, TSOP-28, 10ns) — only locations 0-15 and bits 0-3 used
- **Oscillator**: 48MHz SMD oscillator (3225 package, 3.3V)
- **LDO**: XC6206P332MR (SOT-23, 3.3V) — powers CPLD core, oscillator and clock buffer
- **Clock buffer**: 74LV6T17 — buffers all 5 clock outputs (8/4/2/1MHz, CRTC) for Master 128 compatibility (5V powered); 6th channel buffers the 16MHz clock input
- **Power selection**: Solder bridge selects 3.3V or 5V for SRAM supply

### Key improvements over V01

- 64-pin CPLD package — more I/O pins available
- All clock outputs buffered via 74LV6T17 — improved compatibility with Master 128 custom ICs which require higher logic 1 voltage
- 3.3V SRAM option (IS61LV256AL-10TLI, 10ns) — faster and runs from 3.3V rail
- 3.3V oscillator — reduces CPLD input clamping current and heat
- 74LV6T17 used for both clock output buffering and 16MHz CLK_IN conditioning (Schmitt trigger)
- Separate CLK_16M and CLK_48M GCK pins — no solder bridge needed to switch modes, just reprogram
- SRAM_nWE pull-up to 3.3V — consistent with CPLD drive level

### Hardware Revisions

| Revision | Status | Changes |
|---|---|---|
| Rev01 | PCB designed, not yet built or tested | Initial release, largely based on V01 Rev03 |
| Rev02 | PCB designed, not yet built or tested | Repositioned JTAG header |

Note: Neither revision has been built or tested. The V02 design incorporates all improvements identified during V01 testing.

---

## Firmware

The firmware is written in Verilog and built using Xilinx ISE 14.7. The same Verilog source file (`src/VideoULA.v`) is shared between V02 and V03.

### Operating Modes

Two firmware builds are required — one per operating mode. Both CLK_16M and CLK_48M are wired to dedicated GCK pins, so no solder bridge changes are needed to switch modes — simply reprogram the CPLD with the appropriate `.jed` file.

#### 48MHz Mode (recommended)
- Uses the onboard 48MHz oscillator as the master clock
- Generates all BBC clock outputs (8/4/2/1MHz, CRTC, 6MHz) internally
- Provides clean, phase-locked 6MHz output for the SAA5050 teletext chip (tD ≈ 20ns ✓)
- Works reliably on all machines from cold power-up
- Build: uncomment `` `define CLK_48MHZ `` in `VideoULA.v`

#### 16MHz Mode
- Uses the BBC motherboard's 16MHz clock as the master clock
- The 48MHz oscillator is optional in 16MHz mode:
  - **Not fitted**: BBC's existing 6MHz circuit drives the SAA5050 as normal
  - **Fitted**: CPLD generates clean phase-locked 6MHz output (connect via IC37 replacement board)
- Build: uncomment `` `define CLK_16MHZ `` in `VideoULA.v`

### ISE Fitter Settings (critical)

| Setting | Value | Notes |
|---|---|---|
| Implementation Template | **Optimise Speed** | Density causes timing failures at temperature |
| Macrocell Power | **Std** | Low Power prevents BBC from booting |
| Output Slew Rate | Fast | |
| Default Powerup Value | Low | |

### Build Instructions

1. Open Xilinx ISE 14.7
2. Create a new project targeting `XC9572XL-10-VQ64`
3. Add `src/VideoULA.v` and `src/VideoULA.ucf` to the project
4. Select the operating mode by uncommenting the appropriate `define` in `VideoULA.v`
5. Set fitter options as above
6. Run `Implement Design` to generate the `.jed` file
7. Program the CPLD using iMPACT or a compatible JTAG programmer

---

## SRAM Wiring

The external SRAM stores the 16-entry colour palette (4 bits per entry):

| SRAM Pin | Connection |
|---|---|
| A0-A3 | CPLD SRAM_ADDR[0:3] |
| A4-A14 | GND (tie low) |
| D0-D3 | CPLD SRAM_DATA[0:3] (bidirectional) |
| D4-D7 | Leave unconnected |
| /WE | CPLD SRAM_nWE |
| /OE | GND (always output enabled) |
| /CS | GND (always selected) |

---

## 6MHz Clock for Mode 7 (Teletext)

- **48MHz mode**: CLK_6M pin provides clean phase-locked 6MHz (tD ≈ 20ns ✓). Connect to SAA5050 via IC37 replacement board.
- **16MHz mode, oscillator fitted**: CLK_6M provides clean phase-locked 6MHz, self-correcting every 1µs. Connect via IC37 replacement board.
- **16MHz mode, oscillator not fitted**: CLK_6M output ignored. BBC motherboard's existing 6MHz circuit used as normal.

---

## PCB Design Notes

The PCB has been designed using KiCAD V10 as a 2-layer board. It can easily be changed to a 4-layer board with GND/Power planes on the inner layers if signal integrity is a concern.

### JLCPCB Fabrication

The board design includes a 5×5mm silkscreen box on the underside of the PCB. This is used by JLCPCB to position a QR code serial number. When ordering from JLCPCB:

- Select **Specify position** in the QR code / 2D barcode options during the ordering process
- If this option is not selected, JLCPCB will print the 5×5mm silkscreen box as-is and attempt to place the QR code at a different location of their choosing

If not using JLCPCB, the chosen fabricator will likely print the 5×5mm box as a plain white square on the PCB. To avoid this, remove the silkscreen box from the PCB design and regenerate the Gerber files before submitting.

### Assembly

A combined CPL (Component Placement List) and BOM (Bill of Materials) Excel file is included in the Gerbers folder. This can be used directly with the JLCPCB PCBA (PCB Assembly) service.

Note that the JLCPCB Economy PCBA process only assembles components on one side of the PCB. Select the **top side** for assembly — this leaves only the SRAM and its associated decoupling capacitor to be hand soldered to the underside of the board.

---

## Known Limitations

- **Xilinx XC9572XL is obsolete** — becoming harder to source. V03 uses the still-in-production ATF1504AS as an alternative.
- **SCART/HDMI adapter compatibility**: Some SCART→HDMI display adapters may show minor pixel artefacts with non-standard screen modes. RGBtoHDMI and direct CRT connections are not affected.

---

## Compatibility

Designed for BBC Micro Model B and Master 128. The buffered clock outputs via 74LV6T17 provide the higher logic 1 voltage required by some Master 128 custom ICs.

---

## Credits

- **Ken Lowe** — PCB design, Verilog development and testing
- **hoglet (Stardot forums)** — Clock architecture analysis, DRAM timing analysis, community testing
- **Stardot community** — Testing, feedback and suggestions

---

## Licence

Hardware: [Creative Commons BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

Firmware: [MIT Licence](https://opensource.org/licenses/MIT)
