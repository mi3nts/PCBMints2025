# Dream V3 schematic and PCB audit

Date: 2026-07-16  
Verdict: **Do not fabricate this revision yet.** The Orange Pi header mapping is mostly correct, but the design has multiple release blockers in the USB-C power path, ADC input protection, schematic-to-footprint pin mapping, and use of undocumented header pins as hardware I²C.

## Scope and confidence

Files reviewed read-only:

- `Dream_V3.kicad_sch`
- `Dream_V3.kicad_pcb`
- `Dream_V3.kicad_pro`
- `Dream_V3.kicad_prl`

The project was saved by KiCad 10.0 (`20260306` schematic and `20260206` PCB formats). The installed KiCad CLI is 9.0.7, so native ERC and DRC both stopped at the version gate before analysis. No source files were modified or down-converted. I therefore reconstructed connectivity directly from the KiCad S-expressions and cross-checked schematic pins, PCB pads, net names, tracks, vias, zones, and project rules.

Audit coverage:

| Item | Count |
|---|---:|
| Placed schematic symbol units | 148 |
| Reconstructed schematic pins | 359 |
| Schematic electrical nets | 83 |
| PCB footprints | 52 |
| PCB pads | 279 |
| PCB nets | 88 |
| Copper segments | 646 |
| Vias | 61 |
| Copper zones | 1 |

The static net audit is high-confidence for named pin/net mismatches and documented pin functions. Copper-island detection used track centerlines and was manually checked for geometric overlap. The `5V_BUS_USB` and `PI0/TWI3_SDA` apparent centerline splits are joined by overlapping copper. The `+3V3` split is real.

## Release blockers

### 1. Q3 has a schematic/footprint pin-number mismatch

The schematic uses a dual-MOSFET symbol unit with pins 3/4/5, but Q3 has a three-pad SOT-23 footprint with pads 1/2/3. The PCB currently assigns:

| Physical Q3 pad | PCB net | MOT2301B2 function |
|---:|---|---|
| 1 | `switch_gate` | Gate |
| 2 | `5V_BUS_USB` | Drain |
| 3 | `+5V` | Source |

Those physical assignments match the MOT2301B2 pinout, but they do **not** match the schematic symbol. The audit consequently finds schematic Q3.3 on `5V_BUS_USB` while PCB pad 3 is `+5V`, with schematic pins Q3.4 and Q3.5 absent from the footprint.

**Fix:** replace Q3 with a single P-channel MOSFET symbol whose pin numbers are exactly 1=G, 2=D, 3=S; then update the PCB from the schematic and re-check all three nets. Do not repair only the PCB because the mismatch will return on the next annotation/update.

### 2. The USB-C power switch is unsafe and does not turn fully off

As drawn, J11 VBUS feeds Q3 drain, Q3 source feeds `+5V`, R16 pulls the gate to ground, and S1 connects the gate to the drain to request OFF.

- In OFF, Q3's body diode still conducts from USB VBUS to the board. The rail therefore sits at a brownout-prone voltage rather than zero.
- If the Orange Pi or header side is powered separately, R16 holds Q3 on and the MOSFET conducts backward into J11 VBUS. This can energize a USB-C cable/source from the sink side.
- The MOT2301B2 is only a 2 A-class SOT-23 device with up to about 120 mΩ RDS(on). The Orange Pi documentation calls for a 5 V/2 A supply and commonly recommends 3 A headroom.
- C7 is 100 µF behind a path that begins charging through the body diode, with no controlled soft start. The USB Type-C sink limit is 10 µF at the receptacle while unattached unless the applicable controlled-inrush provisions are met.
- Two 5.1 kΩ CC pull-downs correctly identify a sink, but the board neither measures the advertised source current nor negotiates USB PD. It can attempt to draw Orange Pi current from a source that only advertises default current.

The targeted ngspice model used Q3's 120 mΩ maximum on resistance, 20 mΩ estimated upstream copper, and a generic body diode:

| Condition | Simulated result |
|---|---:|
| ON, nominal 0.5 A load | 4.931 V |
| ON, nominal 1 A load | 4.864 V |
| ON, nominal 2 A load | 4.735 V at 1.894 A |
| Q3 loss near the 2 A case | 0.430 W |
| OFF, nominal 0.1 A load | 4.140 V through body diode |
| OFF, nominal 0.5 A load | 4.021 V through body diode |
| Orange-Pi side externally at 5 V | J11 VBUS rises to 4.9999 V |

The exact body-diode voltage is model-dependent; the lack of isolation and reverse-current path are topology facts.

**Fix:** replace Q3/S1 with a 5 V load switch, eFuse, or power mux rated for at least 3 A that provides reverse-current blocking, current limiting, controlled inrush/soft start, and a true enable. If a discrete solution is retained, use back-to-back MOSFETs and verify SOA, thermal rise, startup, cable drop, and reverse supply conditions. Explicitly define whether the Orange Pi's own Type-C input may ever be connected at the same time.

### 3. TGS2611 can overdrive the 3.3 V ADS1115 input

The gas-sensor divider is 5 V → TGS2611 sensor resistance → `ADS_ANALOG_0` → R3 2 kΩ → GND. R19 is then 10 kΩ in series to the ADS1115. Because the ADC input is high impedance, R19 does not meaningfully divide the voltage.

Using the Figaro specified 0.68 kΩ to 6.8 kΩ sensor-resistance range at the test condition:

| Sensor resistance | ngspice ADS pin |
|---:|---:|
| 0.68 kΩ | 3.727 V |
| 1.0 kΩ | 3.330 V |
| 6.8 kΩ | 1.135 V |

At VDD=3.3 V, TI recommends analog inputs remain between GND and VDD and gives an absolute maximum of GND−0.3 V to VDD+0.3 V. The 0.68 kΩ case is about 3.73 V, above the 3.6 V absolute maximum. Sensor warm-up, tolerance, gas concentration, contamination, or a wiring fault can make the worst case more severe. The programmable full-scale range does not relax this pin-voltage limit.

**Fix:** design the input for a 0–5 V fault envelope, not just the nominal sensor range. Use a calculated divider plus an external clamp and RC filter, or buffer/scale it with a suitable op-amp. Keep clamp current out of the 3.3 V rail when unpowered. Verify ADS1115 settling with the chosen source impedance and calibrate the changed gas-sensor transfer function. The simulated 10 kΩ/33 kΩ illustrative load reduces the worst test point to 2.83 V, but it is not a final recommendation.

### 4. Pins 29 and 31 are not documented as hardware TWI3 on this board

The design labels Orange Pi header pins 29/31 as `PI0/TWI3_SDA` and `PI15/TWI3_SCL`, then uses Q4 to create a 5 V I²C bus. Orange Pi's official Zero 2W table lists these pins only as PI0 and PI15; its supported 40-pin I²C overlays are `pi-i2c0`, `pi-i2c1`, and `pi-i2c2`. The SoC may contain more TWI controllers, but that does not establish that this pin pair has the required mux or supported device-tree overlay.

**Fix:** move J4 to documented TWI0, TWI1, or TWI2 pins, or prove the PI0/PI15 pin-mux function in the H618 pin-control data and test a custom device-tree overlay on the exact kernel image. GPIO bit-banged I²C is a possible fallback, but the nets should then be named and documented as GPIO, not TWI3.

### 5. Schematic-to-PCB pad numbering is broken in several places

These are not cosmetic; they compromise forward annotation, ERC/DRC correlation, and future edits:

| Item | Defect |
|---|---|
| J1 pin 1 | PCB pad number is blank; copper is on `+3V3` |
| J5 pin 7 | PCB pad number is blank; copper is on `PI14/UART4_RX` |
| U1 EN | PCB pad is named `en` instead of schematic pin 3 |
| Q3 | Schematic pins 3/4/5 versus footprint pads 1/2/3 |
| R14 | Array-symbol unit uses pins 3/4, but PCB has ordinary resistor pads 1/2 |
| R11 | Has a footprint in the schematic but no PCB footprint at all |

R11 appears to duplicate pull-up functionality already present in R5. Either delete R11 cleanly or place and connect the intended array; do not leave a schematic-only fitted part.

### 6. `+3V3` is routed as two separate PCB islands

One island contains J1 pin 1 (the blank-numbered pad) and the main 3.3 V loads. The other contains J1 pin 17 and Q4 pads 2/5. They become common only through the plugged-in Orange Pi's internal 3.3 V plane.

**Fix:** route J1 pins 1 and 17 together on this PCB. Do not use the compute module as an accidental jumper. Confirm zero unrouted connections in KiCad 10 after refill.

## Orange Pi 40-pin mapping

The physical J1 net assignment otherwise matches the official Zero 2W 40-pin table:

| Function group | Header pins | Result |
|---|---|---|
| 3.3 V / 5 V / GND | 1, 2, 4, 6, 9, 14, 17, 20, 25, 30, 34, 39 | Electrical names correct; pad 1 numbering and pin-25 ground thermal need repair |
| TWI1 | 3 SDA, 5 SCL | Pass |
| UART4 | 7 TX, 16 RX | Pass |
| UART0 debug | 8 TX, 10 RX | Pass; correctly treated as debug |
| UART5 | 11 TX, 13 RX | Pass |
| TWI0 | 15 SCL, 22 SDA | Pass |
| SPI1 | 19 MOSI, 21 MISO, 23 CLK, 24 CS0, 26 CS1 | Pass |
| TWI2 | 27 SDA, 28 SCL | Pass |
| Claimed TWI3 | 29 PI0, 31 PI15 | Fail as documented hardware I²C; see blocker 4 |
| PWM/GPIO remainder | 12, 18, 32, 33, 35–38, 40 | GPIO identity matches official table |

All Orange Pi header GPIO is 3.3 V logic.

## Important improvements

### Connector voltage conventions and input protection

- J2 and J8 are correctly powered as 3.3 V Qwiic connectors.
- J4 uses a Qwiic connector/value but supplies 5 V. SparkFun defines Qwiic power and logic as 3.3 V. Even though Q4 level-shifts the signals, this connector can damage a normal Qwiic module and should not be labeled or keyed as Qwiic.
- J6 and J9 also use Qwiic-style connectors at 5 V while their UART/I²C signals connect directly to 3.3 V Orange Pi GPIO. A 5 V-powered peripheral can return 5 V on RX/SDA/SCL.
- J3 supplies 5 V with direct 3.3 V TWI1 signals; J7 supplies 5 V with direct 3.3 V SPI signals. Ensure every peripheral is explicitly 3.3 V logic or add translation.
- J10 exposes three ADC inputs without ground, reference/power, RC filtering, or external clamps. Add ground adjacent to the analog signals and protection appropriate to the external environment.
- Add ESD/TVS protection at J11 VBUS/CC and at externally accessible signal connectors. No such devices are present.

### WS2812B logic and decoupling

D1 is powered from 5 V and driven from Orange Pi pin 32 through 330 Ω. Generic/older WS2812B data sheets specify VIH as 0.7×VDD, or 3.5 V at 5 V, so a 3.3 V GPIO is not guaranteed. Some newer variants specify 2.7 V, but the BOM says only `WS2812B_5050` and does not lock a compatible revision.

**Fix:** specify the exact qualified LED part and add an HCT/AHCT-family 3.3-to-5 V buffer if required. Put a 100 nF capacitor immediately at D1 VDD/GND; the current 5 V capacitors are remote.

### LDO and sensor decoupling

U1 is a fixed 3.3 V AP2112 fed from the Orange Pi 3.3 V rail. It therefore operates in dropout and cannot guarantee a regulated, ripple-rejected 3.3 V `BME_3V3` rail. It may function at the BME280's low current, but it does not provide the intended isolation.

**Fix:** feed U1 from 5 V if its dissipation/noise path is acceptable, or replace it with a ferrite/RC filter or load switch suited to the actual isolation goal. Put 100 nF directly at each BME280 supply-pin group; the shared 100 nF capacitor is roughly 11–14 mm away. Keep the regulator's input/output capacitors close to its pins and check their values against the AP2112 stability requirements.

### Thermal sensor placement

IC1's heater dissipates about 280 mW at 5 V. It is placed between the two BME280s, only about 9–10 mm from each. This will bias temperature and humidity readings and can also couple heat through the board from the Orange Pi.

**Fix:** move the gas sensor to a thermally isolated board edge or peninsula, add slots if mechanically acceptable, and place the BME280s at an airflow-exposed edge away from the gas heater, regulator, Orange Pi CPU, and exhaust. Validate with a steady-state thermal test and an external reference sensor.

### I²C pull-ups

Most pull-ups are 10 kΩ. The ngspice RC test gives 30–70% rise times:

| Pull-up and bus C | Rise time | 100 kHz limit (1 µs) | 400 kHz limit (300 ns) |
|---|---:|---|---|
| 10 kΩ, 35 pF | 296.6 ns | Pass | Barely pass |
| 10 kΩ, 100 pF | 847.3 ns | Pass | Fail |
| 10 kΩ, 200 pF | 1694.6 ns | Fail | Fail |
| 4.7 kΩ, 100 pF | 398.2 ns | Pass | Fail |
| 2.2 kΩ, 100 pF | 186.4 ns | Pass | Pass |

With several connectors, cables, MOSFET level shifters, and module pull-ups, 35 pF is easy to exceed. Make pull-ups configurable, measure actual capacitance/rise time, and choose the total parallel pull-up for the intended bus speed and sink-current capability. Avoid unintentionally paralleling every external module's pull-ups.

### PCB return paths, power copper, and DRC rules

- The only plane is a hatched B.Cu GND zone with 1.0 mm hatch and 1.5 mm gaps. Use a solid ground plane; add a F.Cu ground pour where useful and stitching vias near connectors, layer changes, and board edges.
- The USB power path includes 0.2 mm neckdowns and small via transitions before widening. Use a dedicated power netclass, broad pours, and multiple adequately sized vias for 2–3 A. Verify copper temperature rise and full cable-to-header voltage drop.
- Project DRC minima are `min_clearance=0.0` and `min_track_width=0.0`; the default netclass clearance is also 0.0 mm. These settings do not enforce a manufacturable floor. Set rules to the selected fabricator's capability and create explicit power/analog/I²C classes.
- A `starved_thermal` error is explicitly excluded for J1 pin 25 GND on B.Cu. Repair the connection and remove the exclusion; do not waive a compute-module ground-pin defect.
- The B.Cu GND zone uses 0.8 mm clearance, which creates large return-path voids on this dense two-layer board. Re-evaluate after choosing actual fabrication clearances.
- If cost permits, a four-layer stack with an uninterrupted ground plane is the cleaner solution for the Orange Pi, ADC, multiple buses, and external connectors.

### Schematic hygiene and reproducibility

- Nine J1 pins carry no-connect flags while also carrying labels; J1.7 and J1.11 are actually routed onward to J5/J6. Remove no-connect markers from used nets. For intentionally unused GPIO, use either a no-connect marker or a net label, not both.
- J7 shield pads and U1 NC are electrically dangling. Make an explicit shield/chassis strategy and mark genuine NC pins correctly.
- The TGS2611 schematic symbol's datasheet field points to Figaro's SR-6 socket data, not the TGS2611-C00 sensor data sheet.
- `fp-lib-table` references only `${KIPRJMOD}/tgs2611.pretty`, but that directory is absent from the supplied project folder. Many other footprints and 3D models use custom/global library names. Archive all custom symbols, footprints, models, and environment variables with the design so another machine can regenerate it.

### Mechanical integration

The PCB uses a generic 2×20 socket footprint and does not include a complete Orange Pi Zero 2W body/courtyard/3D model. The official module is 30×65×1.2 mm. Header pin mapping alone cannot prove mechanical fit.

Add the exact module outline, mounting holes, underside component heights, connector access, microSD clearance, heatsink volume, and Wi-Fi antenna copper/component keepout. Verify the B.Cu-mounted socket orientation in 3D and with a 1:1 print before ordering.

## Recommended fix order

1. Replace/fix Q3, U1, R14, J1, and J5 symbols/footprints so every schematic pin number exactly matches every physical pad; resolve R11.
2. Redesign the USB-C power path for true OFF, reverse blocking, soft start/inrush control, current limiting, and at least 3 A headroom.
3. Add a protected 0–5 V-capable analog front end for ADS1115 AIN0 and protection/filtering for J10.
4. Move the claimed TWI3 bus to documented I²C pins or prove and test the pin mux/software overlay.
5. Join both J1 3.3 V pins on-board; make ground solid; repair J1.25; widen power copper and via arrays.
6. Correct 5 V/Qwiic and direct 5 V-peripheral logic risks; qualify/level-shift the WS2812B input.
7. Move the gas heater away from the BME280s and complete the Orange Pi mechanical/antenna keepouts.
8. Open the corrected project in KiCad 10, refill zones, run ERC and DRC with zero unexplained errors and zero unrouted items, then generate Gerbers/IPC-356 and inspect them independently.

## Simulation artifacts

The accompanying `spice/` directory contains:

- `usb_power_path.cir`
- `ads1115_input.cir`
- `i2c_rise_time.cir`
- `simulation_results.csv`
- `README.md`

These simulations intentionally cover only the portions that can be meaningfully modeled from the available project. There are no SPICE model assignments in the KiCad schematic, so a whole-board ngspice export would create false confidence.

## Primary references

- [Orange Pi Zero 2W official wiki and 40-pin table](https://www.orangepi.org/orangepiwiki/index.php/Orange_Pi_Zero_2W)
- [Orange Pi Zero 2W product page](https://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-Zero-2W.html)
- [MOT2301B2 manufacturer product page](https://en.mot-mos.com/small/smallsignalMOS_415.html)
- [TI ADS1115 data sheet](https://www.ti.com/lit/ds/symlink/ads1115.pdf)
- [Figaro TGS2611-C00 product information](https://www.figarosensor.com/product/docs/tgs2611-c00_product%20information%28fusa%29rev02.pdf)
- [Diodes AP2112 data sheet](https://www.diodes.com/datasheet/download/AP2112.pdf)
- [SparkFun Qwiic guidance](https://learn.sparkfun.com/tutorials/qwiic-multiport-hookup-guide/hardware-overview)
- [NXP I²C-bus specification UM10204](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
- [USB-IF Type-C Release 2.0, including VBUS sink capacitance](https://www.usb.org/sites/default/files/USB%20Type-C%20Spec%20R2.0%20-%20August%202019.pdf)
- [Current USB-IF Type-C specification library](https://usb.org/document-library/usb-type-cr-cable-and-connector-specification-release-25)
- [Worldsemi WS2812B family data sheet](https://www.world-semi.co.kr/_files/ugd/89cd03_1023b0e9d135431aa1e6491bfc318112.pdf)
