# AT820 TIGA - KiCad PCB Project

## Project Overview

Audio device based on AT820 MCU with USB connectivity, audio codec, mic input, and speaker output.

- **Tool**: KiCad 10.0
- **PCB**: 2-layer, 1.6mm FR4, copper 0.035mm
- **Board area**: ~98mm x 88.5mm (from zone coordinates)

## Schematic Hierarchy

| Sheet | File | Function |
|-------|------|----------|
| Root | `at820_tiga.kicad_sch` | AT820 MCU, CH343P USB-UART, TS5A23157 analog switches, crystal, buttons, connectors |
| Power | `power.kicad_sch` | USB-C input, AMS1117-3.3 LDO, ESD protection, ferrite bead, LED |
| Audio | `audio.kicad_sch` | MIX2909 speaker amp, mic input, audio filtering |

## Key ICs

- **U1 - AT820**: Main MCU (QFN-40-1EP, 24MHz crystal), integrated audio codec with differential mic inputs (LMIC/RMIC) and line outputs (LOUT)
- **CH343P**: USB-to-UART bridge, shares USB-C with AT820 via TS5A23157 analog switch
- **TS5A23157DGS** x3: Dual SPDT analog switches for USB and audio signal routing
- **MIX2909**: Class-D/AB speaker amplifier (SOIC-8-EP), powered from +5V (PVDD)
- **AMS1117-3.3**: 3.3V LDO regulator

## Power Architecture

| Net Name | Voltage | Source | Purpose |
|----------|---------|--------|---------|
| VBUS | 5V | USB-C | Raw USB power |
| +5V | 5V | VBUS via protection | MIX2909 PVDD, peripherals |
| +3V3 | 3.3V | AMS1117 output | Main digital supply |
| +3.3VA | 3.3V | Filtered from +3V3 | Analog supply (codec) |
| +1V8 | 1.8V | AT820 internal PMU | Core supply |

Custom power domains (using `power:+3.3V` symbol with modified Value):
- `AT_DVDD`, `AT_MICBIAS`, `AT_ADC_VDD`, `AT_CODEC_VDD`, `AT_USB_VCC`, `AT_CODEC_REF`, `AT_QSPI_VDD`, `AT_PMU_1V8`, `+3V3_PMU`, `+3V3_OSC`

Ground separation: `GND` (digital) and `GNDA` (analog audio area), connected via 0R resistor.

## Custom Symbol Libraries

- `at820.kicad_sym`: AT820 MCU (40-pin + EP), MY_SW_DPST (custom dual-pole switch)
- `audio.kicad_sym`: MIX2909 audio amp

## Key Signal Labels (main schematic)

- USB: `USB_D+`, `USB_D-`, `CH_USB_D+`, `CH_USB_D-`
- UART: `AT_UART0_TXD/RXD`, `AT_SYS_UART_TXD/RXD`, `CH_UART_TXD/RXD`
- SPI: `AT_SPI0_CLK/MO/MI/CS0`
- I2C: `AT_I2C0_SDA/SCL`
- Audio: `AT_LMIC_P/N`, `AT_RMIC_P/N`, `AT_LOUT_P/N`
- Audio amp: `PA_INP/INN`, `PA_OUTP/OUTN`, `PA_EN`
- Control: `AT_BOOT`, `AT_RST`, `AT_TEST_MODE`, `AT_GPIO3`, `AT_PWM0`
- Debug: `AT_SWDIO`, `AT_SWCLK`

## Connectors

| Ref | Type | Function |
|-----|------|----------|
| J1 | 1x6 | SWD debug |
| J2 | 1x2 | UART |
| J3/J4/J5 | 1x4 | I2C / GPIO |
| J8/J9 | USB-C 16P | USB (one for AT820, one for CH343P) |
| J13 | - | TEST_MODE |

## PCB Design Rules

- Default track: 0.15mm, min track: 0.1mm
- Via: 0.6mm dia / 0.3mm drill
- Clearance: 0.15mm
- GND zone: priority 1, covers full board
- GNDA zone: priority 0, covers audio area (118-153.5mm x 56-71.5mm)

## Auto-Download Circuit (CH343P)

CH343P drives the AT820 BOOT/RST lines through a **two-transistor cross-coupled (interlock) circuit** — the classic ESP/NodeMCU auto-program topology. The WCH "no external component" (免外围电路) direct-connection scheme is **not** used.

**Connections (CH343P 343P-version pinout: Pin 12 = DTR, Pin 13 = RTS):**
```
                              Q1 (SS8050, NPN)            Q2 (SS8050, NPN)
CH343P Pin 12 (DTR) ──┬── R7 (10K) ──► Q1.B          ──► Q2.E ── CH343P Pin 12 (DTR)
CH343P Pin 13 (RTS) ──┴── R8 (10K) ──► Q2.B          ──► Q1.E ── CH343P Pin 13 (RTS)

Q1.C ──► AT_RST  ──┬── R9 (2K)  pull-up to +3V3 ──┬── SW2 (Reset Button) to GND ── C12/C13
                                                   └── AT820 RESETN
Q2.C ──► AT_BOOT ──┬── R1 (10K) pull-up to +3V3 ──┬── SW3 (Boot Button)  to GND ── C1/C14
                                                   └── AT820 BOOT_SEL
```
- **Q1 (controls RST):** Base ← DTR (via R7), Emitter = RTS, Collector = AT_RST
- **Q2 (controls BOOT):** Base ← RTS (via R8), Emitter = DTR, Collector = AT_BOOT

**Logic (NPN conducts when V_base − V_emitter > ~0.7 V):**
- DTR=H, RTS=H (idle/port-closed) → both off → BOOT & RST pulled high → MCU runs normally
- DTR=H, RTS=L → Q1 on → RST=L (reset asserted); BOOT high
- DTR=L, RTS=H → Q2 on → BOOT=L; RST high
- RST-low and BOOT-low are mutually exclusive by design (avoids the deadlock that a direct connection has)

**IMPORTANT — flashing-tool compatibility:** This circuit only works with an **esptool-style** tool that drives DTR/RTS in *opposite phase* (classic reset: pulse RTS→reset with BOOT high, then DTR→BOOT low as RST releases). A tool expecting the WCH direct scheme (hold DTR low for BOOT, pulse RTS for RST) will **not** reset, because RST only asserts when DTR=H. Confirm the AT820 download tool uses the esptool-style sequence.

**Pull-ups:** R1 (10K) on BOOT and R9 (2K) on RST are now populated (R1 is no longer NC).

## Known Issues / Review Notes

1. **USB via analog switch**: TS5A23157 (~10R on-resistance) in USB data path may affect signal integrity
2. **Power trace width**: Default 0.15mm is thin for power nets, recommend 0.3-0.5mm
3. **Crystal**: 24MHz with 20pF load caps - verify against crystal CL spec
4. **Button debounce**: Boot/Reset/Wakeup buttons have pull resistors but no debounce caps
5. **No BOM or Gerber exported yet**
6. **Connectors J4-J7**: Generic labels, function not annotated in schematic
7. **CH343P download tool sequence**: Cross-coupled transistor circuit requires an esptool-style (DTR/RTS opposite-phase) flashing tool; a WCH direct-scheme tool will not reset. Verify before relying on auto-download.
8. **MIX2909 SD pin (U5.8 / AT_PWM0)**: No pull resistor — floats at power-up before firmware drives the GPIO; add a pull to the defined (shutdown) state to avoid undefined amp state / POP.
9. **AT820 (U2) Value field blank**: Set symbol Value to `AT820` so it appears in the BOM.

## Datasheets

Available in `docs/` folder: CH343P, MIX2909, speaker (3525 box), power inductor (PCR0420), SS14 diode, switch.
