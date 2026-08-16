# Hardware Reference

## 1. Scope and Baseline

This document describes the current Rufajx temperature-control schematic and the matching EasyEDA `PCB2` layout inspected on 2026-08-16. The controller is U5, an STM32G0B1CBT6N in LQFP-48. The board accepts a nominal 24 V machine supply and can also power its low-voltage control electronics from USB during service.

P4 is intentionally used to interrupt a 220 VAC external circuit. It is hazardous-mains circuitry and is not part of the P2 analog command path.

## 2. Functional Blocks

| Block | Main devices | Function |
|---|---|---|
| Controller | U5 STM32G0B1CBT6N | USB, RS-485, DAC, ADC, thermocouple interface, relay control, PID |
| Temperature input | U1 MAX6675ISA+T | K-type conversion and cold-junction compensation |
| Analog output | U3 TLV9351IDBVR | Scales the 2.5 V DAC domain to a monitored 0–5 V output |
| Output feedback | R15, R16, R17, C23 | Divides and filters the actual P2 voltage for PA0 ADC |
| 24 V front end | F3, D7, D8, C16, C17 | Overcurrent, reverse-polarity, transient, and local bypass protection |
| 24 V to 5 V | U2 K7805-500R3 | Low-power machine-supply conversion |
| 5 V source selection | D5, D6, C13, C14 | Prevents backfeed between USB and the 24 V converter |
| 3.3 V rail | LDO1 ME6211C33M5G-N | MCU and MAX6675 supply |
| USB | USB1, D2, F1, D1 | Type-C USB 2.0 service/programming interface |
| RS-485 | U6 THVD1400DR, D4 SM712-02HTG | Non-isolated, half-duplex field communication at P5 |
| Mains interrupt | RLY1, Q1, D9 | MCU-driven 24 V relay with 220 VAC NC/COM at P4 |
| Programming/debug | H1 | SWDIO, SWCLK/BOOT0, NRST, 3.3 V, and GND |

## 3. STM32G0B1 Controller

### 3.1 Device and signals

U5 is an STM32G0B1CBT6N. The `N` device/symbol exposes a dedicated VDDIO2/VSS pair at pins 31 and 30. Main assigned signals are:

- PA0: P2 output feedback ADC;
- PA1: USART2 hardware driver-enable output for RS-485;
- PA2: USART2 TX to the RS-485 transceiver;
- PA3: USART2 RX from the RS-485 transceiver;
- PA4: DAC1_OUT1 heating command;
- PA5, PA6, PB13: MAX6675 SCK, SO, and CS;
- PB0: relay-coil control;
- PA11, PA12: USB D- and D+;
- PA13, PA14: SWDIO and SWCLK/BOOT0;
- PF2: NRST.

### 3.2 Supply and reference pins

- VBAT is tied to `3V3_SYSTEM`.
- VDD/VDDA is tied to `3V3_SYSTEM` and bypassed by C2 100 nF plus C3 4.7 µF.
- VSS/VSSA is tied to GND.
- Pin 31 VDDIO2 is tied to `3V3_SYSTEM`.
- Pin 30 VSS is tied to GND.
- C24 100 nF bypasses the VDDIO2 supply pair. ST's supply recommendation also shows a local 4.7 µF capacitor for this pair; add it before production release.
- VDDIO2 must not be applied while VDD is absent.

VREF+ is not tied to 3.3 V. It is the internal VREFBUF output node, configured by firmware for 2.5 V and stabilized by C19 1 µF plus C22 100 nF to GND. The 2.5 V VREFBUF range is valid with the 3.3 V VDDA supply. Firmware must enable VREFBUF, select the 2.5 V range, and wait for the ready flag before enabling the DAC or trusting ADC readings.

## 4. K-Type Thermocouple Input

### 4.1 Electrical path

P1 pin 1 is K+ and P1 pin 2 is K-. K- is intentionally tied to board GND because the MAX6675 requires T- to be externally grounded.

The K+ path contains:

- D3 TPD1E01B04DPYR, a low-leakage ESD protector from K+ to GND;
- R14 47 ohm in series;
- C18 100 nF from the filtered T+ node to GND;
- U1 MAX6675 T+ input.

The nominal corner frequency is:

```text
fc = 1 / (2*pi*47 ohm*100 nF) = 33.9 kHz
```

U1 is powered from the filtered 3.3 V branch through R8 and is bypassed by C1 100 nF plus C21 1 µF.

### 4.2 Probe boundary

Only an insulated/ungrounded-junction K-type probe is qualified. The thermocouple junction must not be electrically connected to the probe sheath or machine chassis. If the cable has a shield, connect it to chassis/PE at one end; do not use either thermocouple conductor as the shield connection.

The P1 pin-2 silkscreen remains `K-`, even though the node is electrically board GND. This is expected for this MAX6675 implementation.

### 4.3 Converter behavior

The MAX6675 reports 0.25 °C code resolution and an open-thermocouple flag. It requires at least about 220 ms between conversions; the project acquisition period is 250 ms.

Resolution is not total accuracy. Probe tolerance, connector alloy transitions, terminal and PCB thermal gradients, cable routing, machine grounding, cold-junction conditions, and EMC behavior must be included in production validation.

## 5. Precision 0–5 V Output

### 5.1 DAC and amplifier

PA4 uses DAC1_OUT1 with the internal 2.5 V reference. R7 1 kohm feeds U3 IN+, C8 100 nF filters the amplifier input, and R18 100 kohm pulls `HEATER_DAC` to GND while PA4 is high impedance.

U3 is a non-inverting amplifier:

- R9 = 10.5 kohm, 0.1%, from output to IN-;
- R10 = 10.0 kohm, 0.1%, from IN- to GND.

```text
Gain = 1 + R9/R10 = 2.05
Theoretical full scale = 2.5 V * 2.05 = 5.125 V
Nominal DAC code for 5.000 V = approximately 3995 / 4095
```

The extra analog headroom allows calibration to reach 5.000 V without relying on the exact top DAC code. Firmware must clamp the commanded output to 0.000–5.000 V and use measured gain/offset coefficients.

U3 runs from `24V_PROTECTED` and is bypassed by C15 100 nF plus C20 1 µF. Its output passes through R11 100 ohm to P2. U4 TPD1E10B06DPYR protects the terminal node from ESD. P2 pin 1 is GND; pin 2 is `HEATER_OUT_5V`.

R11 limits transient and short-circuit current but creates load-dependent drop. The ADC circuit samples after R11, so firmware can observe the actual terminal voltage.

### 5.2 ADC feedback

- R15 = 56 kohm, 0.1%, from `HEATER_OUT_5V` to `HEATER_FB_DIV`;
- R16 = 51 kohm, 0.1%, from `HEATER_FB_DIV` to GND;
- R17 = 1 kohm from `HEATER_FB_DIV` to PA0;
- C23 = 100 nF from PA0 to GND.

```text
VADC / VOUT = 51 / (56 + 51) = 0.4766355
At VOUT = 5.000 V, VADC = 2.383 V
```

This is below the 2.5 V ADC reference while using most of the range. ADC feedback supports calibration and fault detection; it does not provide independent hardware shutdown.

### 5.3 Required load qualification

- Verify the actual hot-air-machine input impedance.
- Measure 0, 0.5, 2.5, 4.5, and 5.0 V at P2 under the real load.
- Measure R11 drop, output noise, settling, temperature drift, and ADC reconstruction error.
- Treat sustained short circuit, stuck-high output, or persistent command/feedback mismatch as a latched fault.

## 6. 24 V Input and Rail Generation

P3 pin 1 is GND and pin 2 is `24V_IN`. The protection order is:

1. F3 ASMD1812-020-60V resettable fuse;
2. D7 SS36 series reverse-polarity diode;
3. D8 SMAJ24A TVS from the protected rail to GND;
4. C16 22 µF/50 V and C17 100 nF;
5. `24V_PROTECTED`.

`24V_PROTECTED` supplies U2, U3, and the relay coil. U2 K7805-500R3 generates `5V_FROM_24`.

The K7805-500R3 input limit is 32 V. It is retained only under the explicit assumption that the machine supply is a stable nominal 24 V source with startup, shutdown, relay, and heater transients remaining comfortably below that limit. If measurements approach 30 V, replace U2 with a 45–60 V-rated converter and requalify the input protection.

## 7. USB Power and Data

USB1 is a USB 2.0 device-only Type-C interface:

- CC1 and CC2 each use 5.1 kohm Rd to GND;
- duplicate D+ contacts are joined and duplicate D- contacts are joined;
- D2 USBLC6-2SC6 protects the data pair;
- F1 protects VBUS;
- D1 SMF5.0A and C5 4.7 µF protect and bypass USB 5 V.

D5 and D6 B5819W diode-OR `5V_FROM_24` and `USB_5V` into `5V_SYSTEM`. C13 10 µF and C14 100 nF bypass the combined rail. LDO1 generates `3V3_SYSTEM` with 4.7 µF at input and output.

USB powers only the low-voltage control electronics. It does not power U3 or the 24 V relay coil. P2 full-output and relay tests require the 24 V input.

## 8. RS-485 Interface

U6 is a 3.3 V THVD1400DR half-duplex RS-485 transceiver. Its MCU-side connections are:

- PA1 / USART2_RTS_DE: `RS485_DE`;
- PA2 / USART2_TX: `RS485_TX` to U6 D;
- PA3 / USART2_RX: `RS485_RX` from U6 R;
- U6 `/RE` and `DE` are tied together at `RS485_DE`;
- R21 10 kohm pulls `RS485_DE` low, which disables the driver and enables the receiver during reset or while PA1 is high impedance;
- C25 100 nF and C26 1 uF bypass the U6 3.3 V supply.

The field-side path is:

- U6 pin 7 B through R19 10 ohm to `RS485_B`;
- U6 pin 6 A through R20 10 ohm to `RS485_A`;
- D4 SM712-02HTG protects A and B to board GND;
- P5 pin 1 is B and P5 pin 2 is A.

R22 120 ohm is an **optional** termination resistor directly across A and B. Populate it only when this board is selected as one of the two physical ends of a passive RS-485 bus. Do not populate R22 on every branch endpoint or every node. For production, implement the termination as DNP-by-default or as an explicitly controlled assembly option and record which two bus endpoints are terminated.

This interface is not galvanically isolated. P5 exports only A and B because the machine installations are expected to share a signal-ground reference elsewhere; D4 also returns surge current to the board logic ground. The external common-ground connection does not remove the transceiver's common-mode limitation. Ground-potential difference, coupled transients, and steady common-mode voltage must remain within the THVD1400 datasheet operating and absolute-maximum limits. Use an isolated RS-485 implementation, or redesign the grounding and protection strategy, if cabling length, building wiring, or machine bonding cannot guarantee that boundary.

## 9. Normally Closed 220 VAC Relay Interface

RLY1 is currently an SRD-24VDC-SL-C:

- coil pin 4: `24V_PROTECTED`;
- coil pin 1: Q1 low-side drain;
- Q1: 2N7002K N-channel MOSFET;
- R12: series gate resistor from PB0;
- R13: gate pull-down to GND;
- D9: flyback diode, cathode at `24V_PROTECTED`, anode at the Q1/coil node.

P4 exports the contact path:

- P4 pin 1: NC / `$1N128`;
- P4 pin 2: COM / `$1N129`.

| PB0 `RELAY_CONTROL` | Coil | P4 NC-COM |
|---|---|---|
| Low / MCU unpowered | De-energized | Closed |
| High, with 24 V present | Energized | Open |

This is not loss-of-power fail-safe. Board power loss closes the external 220 VAC path.

### 9.1 Current isolation limitation

The current PCB is not released for this mains use:

- closest relay contact-pad to coil-pad edge spacing: approximately 3.53 mm;
- project conservative mains-to-SELV target: at least 8 mm until the governing standard defines the final value;
- three multi-layer `NO_POURS` regions stop pours only, not tracks, vias, pads, or fills;
- no explicit mains-to-SELV net-to-net DRC rule is active;
- current zero-error DRC therefore does not prove safe isolation.

Replace or redesign the relay/footprint and the board isolation barrier before production. A relay family explicitly documenting 8 mm clearance and creepage, such as an applicable Omron G2RL version, is an example for evaluation. Select the exact part from actual contact current, inrush, load category, lifetime, coil voltage, temperature, approvals, and enclosure requirements.

## 10. Reset, Boot, and SWD

- NRST has R4 10 kohm pull-up, C7 100 nF, and a button to GND.
- PA14-BOOT0/SWCLK has R6 10 kohm pull-down. The BOOT button applies 3.3 V through R5 1 kohm.
- H1 provides GND, 3.3 V target reference, NRST, SWCLK/BOOT0, and SWDIO.

Because PA14 is both SWCLK and BOOT0, do not press BOOT during an active SWD session.

## 11. Major Parts and LCSC References

| Ref. | Part | LCSC |
|---|---|---|
| U5 | STM32G0B1CBT6N | C5270160 |
| U1 | MAX6675ISA+T | C16030 |
| U2 | K7805-500R3 | C19188491 |
| U3 | TLV9351IDBVR | C1517412 |
| LDO1 | ME6211C33M5G-N | C82942 |
| D3 | TPD1E01B04DPYR | C779389 |
| U4 | TPD1E10B06DPYR | C48260 |
| D2 | USBLC6-2SC6 | C2827654 |
| U6 | THVD1400DR | Confirm before BOM release |
| D4 | SM712-02HTG | Confirm before BOM release |
| RLY1 | SRD-24VDC-SL-C | C15840 |
| Q1 | 2N7002K-T1-GE3 | C81445 |
| D8 | SMAJ24A | C148222 |
| D7 | SS36 | C116712 |
| F3 | ASMD1812-020-60V | C2982277 |
| USB1 | TYPE-C 16PIN 2MD(073) | C2765186 |
| H1 | PZ254V-11-05P | C492404 |
| P5 | WJ2EDGV-5.08-02P-14-00A | C8461 |
| P1–P4 | WJ2EDGV-5.08-02P-14-00A | C8461 |

Library metadata is not a substitute for the purchased manufacturer's datasheet. Before BOM release, lock manufacturer part number, supplier code, footprint, tolerance, power/voltage rating, dielectric, temperature grade, and substitution policy for every item.

## 12. Primary References

- [STM32G0B1 datasheet](https://www.st.com/resource/en/datasheet/stm32g0b1me.pdf)
- [STM32 system-memory boot mode, AN2606](https://www.st.com/resource/en/application_note/an2606-stm32microcontroller-system-memory-boot-mode-stmicroelectronics.pdf)
- [MAX6675 datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/max6675.pdf)
- [THVD1400 datasheet](https://www.ti.com/lit/ds/symlink/thvd1400.pdf)
- [Omron G2RL relay-family datasheet](https://components.omron.com/sites/components.omron.com.us/files/datasheet_pdf/J117-E1.pdf)
- [IEC 60664-1 insulation-coordination overview](https://webstore.iec.ch/en/publication/59671)
- [STM32CubeG0 firmware package](https://github.com/STMicroelectronics/STM32CubeG0)
