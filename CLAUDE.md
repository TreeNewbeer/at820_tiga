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

CH343P uses WCH's "no external component" (免外围电路) direct-connection scheme for MCU download, **no transistor cross-coupling needed**.

**Connections:**
```
CH343P Pin 12 (DTR) ──→ AT_BOOT ──┬── R1 (10K/NC) pull-up to +3V3
                                    ├── SW6 (Boot Button) to GND
                                    ├── C36 (0.1uF) parallel to button
                                    └── AT820 BOOT_SEL

CH343P Pin 13 (RTS) ──→ AT_RST  ──┬── R4 (2K) pull-up to +3V3
                                    ├── SW3 (Reset Button) to GND
                                    └── AT820 RESETN
```

**How it works (per CH343P spec section 5.5 "DTR dual-mode MCU download"):**
- DTR defaults HIGH during idle/power-up → BOOT stays HIGH → MCU runs normally
- Only download tool explicitly sets DTR LOW → BOOT LOW, then RTS pulses RST → enters bootloader
- Unlike traditional UART chips, opening serial port does NOT auto-assert DTR

**Open item:** R1 (10K) is marked NC. CH343P DTR is tri-state during power-up. If AT820 BOOT_SEL has no internal pull-up, R1 should be populated to prevent floating during power-up.

## Known Issues / Review Notes

1. **USB via analog switch**: TS5A23157 (~10R on-resistance) in USB data path may affect signal integrity
2. **Power trace width**: Default 0.15mm is thin for power nets, recommend 0.3-0.5mm
3. **Crystal**: 24MHz with 20pF load caps - verify against crystal CL spec
4. **Button debounce**: Boot/Reset/Wakeup buttons have pull resistors but no debounce caps
5. **No BOM or Gerber exported yet**
6. **Connectors J4-J7**: Generic labels, function not annotated in schematic
7. **R1 (10K/NC) on BOOT**: Verify AT820 has internal pull-up on BOOT_SEL; if not, populate R1

## Datasheets

Available in `docs/` folder: CH343P, MIX2909, speaker (3525 box), power inductor (PCR0420), SS14 diode, switch.
