# Dream V3 targeted ngspice simulations

These are focused, simplified testbenches for the board's three analyzable risk areas. They are not a whole-board simulation. The KiCad schematic has no SPICE model assignments, and the Orange Pi, BME280, ADS1115 digital behavior, WS2812B, and TGS2611 do not have complete models in the project.

Run with ngspice 46 or later:

```sh
ngspice -b usb_power_path.cir
ngspice -b ads1115_input.cir
ngspice -b i2c_rise_time.cir
```

Model assumptions:

- `usb_power_path.cir`: Q3 is an ideal voltage-controlled switch with 120 mΩ on resistance (the MOT2301B2 maximum), 20 mΩ estimated upstream copper, and a generic body diode. Exact diode-drop numbers are illustrative; OFF-state conduction and reverse backfeed follow from the topology.
- `ads1115_input.cir`: the ADC is represented by a 10 MΩ input. This demonstrates that the existing 10 kΩ series resistor is not a divider. Sensor resistance is swept by parallel copies at 680 Ω, 1 kΩ, and 6.8 kΩ.
- `i2c_rise_time.cir`: ideal 3.3 V step, lumped bus capacitance, and pull-up resistance. Rise time is measured from 30% to 70% as specified for I²C.

See `simulation_results.csv` and the main audit report for interpretation.
