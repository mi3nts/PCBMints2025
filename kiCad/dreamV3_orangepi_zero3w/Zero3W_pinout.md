# DREAM V3 for Orange Pi Zero 3W

Open `Dream_V3_Zero3W.kicad_pro` in KiCad 10. The reference is the A733 Zero3W V1.2. Directions below refer to the top-side view with the MINTS-AI title at the top. J1 numbers are physical header pins.

| Connector | Position | Interface | J1 SDA / SCL |
|---|---|---|---|
| J12 | Upper left, vertical | I²C1, PB8 / PB7 | 38 / 40 |
| J14 | Left edge, vertical | I²C1, PB8 / PB7 | 38 / 40 |
| J3 | Upper right, vertical | I²C0, PB3 / PB2 | 3 / 5 |
| J13 | Lower right, horizontal | I²C0, PB3 / PB2 | 3 / 5 |

All four use Würth 679305124022. Pin 1 supplies +5 V, pin 2 is SDA, pin 3 is SCL, and pins 4 and 5 connect directly together on ground. SDA and SCL on these ports connect to 3.3 V GPIO. Mating sensors must accept 3.3 V signalling and must not pull the lines up to 5 V. Two devices on the same bus must have different active I²C addresses. For two fixed-address devices with the same address, use one left port and one right port.

## Other header connections

| Function | Zero3W header and direction |
|---|---|
| J4 software I²C | SDA on pin 29 (PE12), SCL on pin 31 (PE13). Existing Q4 level shifter and connector pinout retained. Update the software GPIO selection for the A733. |
| J8 and onboard ADS1115 | Share hardware I²C0 with J3/J13. U3 SDA is pin 3 and U3 SCL is pin 5. The old header pins 22 and 15 are unused. |
| BME280 interfaces | Q1 connects to I²C0 and Q2 connects to I²C1. R1 and R13 schematic pin numbering follows the routed pull-up arrays. |
| J5 COZIR | UART7, host TX from J1.16 (PL2) to J5.5, host RX at J1.18 (PL3) from J5.7 |
| J6 serial | UART2, host TX from J1.11 (PB0) to J6.3, host RX at J1.13 (PB1) from J6.4 |
| J7 OPC-N3 | SPI3. MOSI on J1.19, MISO on J1.21, SCLK on J1.23, CS0 on J1.24. |
| RGB LED data | J1.32, PD1 / PWM0-1, through R8 |
| Power at J1 | +5 V on 2/4, +3.3 V on 1/17, GND on 6/9/14/20/25/30/34/39 |

The vendor Linux image documents enabling I²C0, I²C1, UART2, UART7 and SPI3 through its hardware configuration. The documented I²C2, I²C3 and UART6 mappings overlap the SPI3 pins. Reserve those pins for J7. Configure J4 separately as open-drain software I²C. Bus enumeration and sensor operation still need verification on the installed OS image.

## Verification and remaining issues

KiCad 10.0.6 reported zero unconnected pads, zero board DRC errors, 63 board warnings and 16 schematic-parity findings. The original Orange Pi design produced two unconnected items, 69 board warnings and 32 parity findings. The revised schematic ERC reported six errors and 119 warnings. The six errors also occurred in the original. Four are power-input drive declarations and two are the J7 shield pins. The reports accompany the project.

The four five-pin connectors were checked against the exported schematic netlist and against the connected copper in the refilled PCB. Each GND pair has a direct track between pads 4 and 5. All 40 J1 pad assignments and the J4, J5, J6, J7, J8, Qwiic-bus pull-up array R1/R13 and ADS1115 U3 signal assignments were also compared with the PCB.

Fabrication is pending review. In particular, the inherited Q3 schematic and footprint disagree on the USB power path. U1 and R14 also have pin-number mismatches. The starting repository's `gpt_audit/Dream_V3_Audit_Report.md` discusses potential backfeed and the ADC input circuit. This revision has not been powered, fitted to a Zero3W or tested with sensors. The top render shows connector placement. Some inherited component models are unavailable, so it cannot verify assembly height or enclosure fit.

## Source drawings

Used the original `kiCad/dreamV3_orangepi` for comparison and its `kiCad/DreamV3_orangepi_sps30` derivative for the J13 connector layout and related routing. The supplied `OrangePiZero3W/OPI ZERO 3W V1_2 Schematic Diagram.pdf`, sheet 18, and `OrangePi_Zero3W_A733_User Manual_v1.0.pdf`, printed page 99 and sections 3.16.3–3.16.6, establish the header numbering and alternate functions.
