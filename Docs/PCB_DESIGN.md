# PCB Design and Release Checklist

## Current Layout

The current EasyEDA design is a compact two-layer PCB with the following placement zones:

- P1 and the MAX6675 thermocouple front end are at the upper-left edge.
- P2, the filtered 0–3.3 V output, is on the left edge.
- The STM32, BOOT button, and RESET button occupy the upper-center area.
- P3 and the 24 V protection components are on the upper-right side.
- The 24 V-to-5 V converter and bulk capacitors occupy the center/lower-right area.
- USB-C and its protection parts are on the bottom edge.
- The 5 V ORing and 3.3 V LDO are located between the power sources and MCU area.
- Two mechanical mounting holes are present near opposite board corners.

## Verified PCB Connectivity

The current PCB netlist contains these key assignments:

| Function | PCB net / destination |
|---|---|
| MAX6675 chip select | STM32 pin 10, PA4 |
| MAX6675 clock | STM32 pin 11, PA5 |
| MAX6675 serial output | STM32 pin 12, PA6 |
| PWM source | STM32 pin 16, PA8 |
| USB D− | STM32 pin 18, PA11 |
| USB D+ | STM32 pin 19, PA12 |
| 24 V input | P3 pin 2 to F2 |
| Analog command | Filter output to P2 pin 2 |

## Latest Automated DRC Result

Status: **HOLD — do not release Gerber files yet.**

The EasyEDA API DRC was run with strict checking and verbose output. It returned three errors:

1. USB-C footprint clearance: pad `USB_A1B12` to a slot region is 0.171 mm; the active rule requires at least 0.18 mm.
2. USB-C footprint clearance: pad `USB_B1A12` to a slot region is 0.171 mm; the active rule requires at least 0.18 mm.
3. The PCB and schematic netlists do not match.

Required actions:

1. Run **Import Changes** from the schematic and inspect every reported difference. Do not accept changes blindly.
2. Confirm that the USB-C footprint is the exact footprint for the purchased C2765186 connector.
3. Resolve the two pad-to-slot clearance violations by correcting the footprint or by using a documented fabrication-rule exception approved by the PCB manufacturer.
4. Repeat schematic ERC, PCB DRC, and netlist comparison until no unexplained errors remain.

## Thermocouple Layout Requirements

- Keep the MAX6675 close to P1 and away from U2, D3, D4, the heater driver, and other heat sources.
- Keep the thermocouple input path short, paired, and isolated from USB, PWM, switching-regulator, and high-current traces.
- Do not route a PWM or switching node directly beneath or beside the thermocouple input.
- Avoid copper structures that conduct heat from the power section into the MAX6675 cold-junction area.
- If field testing shows noise sensitivity, reserve footprints for a differential input filter recommended by the MAX6675 datasheet; do not add large unmatched capacitors from only one thermocouple lead to ground.
- Label P1 unambiguously as `K+` and `K-` on the wire-entry side.

## USB Layout Requirements

- Route D+ and D− as a short, coupled differential pair with consistent geometry.
- Avoid stubs and unnecessary vias between the Type-C receptacle, D2, and the MCU.
- Place D2 as close as practical to the USB connector entry point; the protected side faces the MCU.
- Keep ESD return paths short and wide to ground.
- Join duplicate Type-C D+ contacts together and duplicate D− contacts together before the protected data path.
- Keep CC1 and CC2 pull-down resistors close to the connector.

## Power Layout Requirements

- Place F2, D3, and D4 close to P3 so surge and reverse-polarity current does not cross the logic area.
- Keep the `24V_PROTECTED` switching-current loop compact.
- Place C9/C10 close to U2 input and C11/C12 close to U2 output.
- Use trace widths and copper areas appropriate for the actual control-board current, fuse trip behavior, and temperature rise.
- Keep `5V_FROM_24`, `USB_5V`, and `5V_SYSTEM` visually and electrically distinct until the D5/D6 ORing point.
- Place every 100 nF bypass capacitor next to the IC supply pin it serves, with a direct ground return.
- Use a continuous ground reference where possible. Do not force thermocouple or USB return current through the switching-converter current loop.

## PWM Output Layout Requirements

- Keep R7 and C8 near P2 or near the final signal path so the filtered trace is short.
- Keep the filtered `OUT` node away from the raw PWM trace and switching-power nodes.
- The external load should be high impedance. If a long cable or low-impedance receiver is required, add an op-amp buffer and suitable cable protection in a future revision.
- Label P2 as `GND` and `OUT`; optionally add `0–3.3V` beside the connector.

## Mechanical and Assembly Checks

- Confirm terminal-block wire-entry direction and screwdriver access after enclosure installation.
- Confirm USB-C shell and board-edge position against the enclosure opening.
- Verify mounting-hole diameter, keepout, fastener head clearance, and chassis-ground policy.
- Check component polarity markings for D1–D6 and C9.
- Confirm pin-1 markings for STM32, U1, U2, and the LDO remain visible after assembly.
- Replace non-orderable supplier IDs in the BOM before JLCPCB assembly quotation.

## Production Release Checklist

- [ ] Schematic ERC reviewed.
- [ ] Schematic-to-PCB update completed with no unexplained differences.
- [ ] PCB DRC has no unexplained errors.
- [ ] USB-C footprint verified against the connector drawing.
- [ ] All copper pours rebuilt and inspected.
- [ ] 24 V polarity, TVS polarity, and diode-OR polarity checked.
- [ ] K-type connector polarity checked against silkscreen.
- [ ] BOM contains valid supplier codes for every assembled part.
- [ ] Pick-and-place rotations reviewed in the JLCPCB preview.
- [ ] Gerber files inspected in an independent viewer.
- [ ] First article tested from current-limited USB and 24 V supplies.
- [ ] USB DFU, USB CDC, sensor-open shutdown, and 0 V fail-safe verified.
- [ ] Thermal test completed at the intended 200–600 °C process range while the PCB remains within component ambient limits.
