# PCB Design and Manufacturing Notes

## 1. Verified Design Baseline

This document records the latest reviewed EasyEDA board `PCB2` (UUID `6d748ac1a8e87f99`) as of 2026-08-16. The editable source is `PCB Design/Rufajx温控系统.eprj2`.

| Item | Reviewed value |
|---|---:|
| Board | Approximately 65 mm × 60 mm, two copper layers |
| Components | 77 |
| Routed line segments | 525 |
| Standard vias | 48 |
| GND stitching vias identified | 12 |
| Copper pours | One top GND pour and one bottom GND pour |
| Multi-layer regions | 3, configured as `NO_POURS` |
| Strict EasyEDA DRC | 0 reported violations under the active rules |

The active rule configuration is named `Gridopoly8L_Production_v1`, although the board is two-layer. Rename or explicitly verify that profile before release so its name cannot be mistaken for the manufactured stack-up.

A zero-error DRC proves only compliance with the configured CAD rules. It does not prove mains isolation, controlled impedance, EMC, thermal behavior, assembly quality, or machine safety.

## 2. Functional Placement

The board is separated into the following functional areas:

- P1, D3, R14, C18, and U1: thermocouple input;
- U5, local bypassing, RESET/BOOT, and H1: MCU and debug;
- U3, P2, the gain network, and PA0 feedback divider: precision 0–5 V output;
- U6, R19/R20/R21/R22, D4, and P5: RS-485;
- RLY1 and P4: hazardous 220 VAC contact path;
- P3, F3, D7/D8, and U2: 24 V input and conversion;
- USB1, D2, F1, D1, and C5: USB interface;
- D5/D6 and LDO1: low-voltage source selection and 3.3 V generation.

Preserve the physical separation between the thermocouple/precision-analog areas and the relay, mains, USB, RS-485, and 24 V current loops.

## 3. Critical Routing Regression Baseline

Re-measure these values after every placement or routing update.

| Signal | Approx. routed length | Layer use | Width | Vias |
|---|---:|---|---:|---:|
| K+ | 6.84 mm | Top | 0.40 mm | 0 |
| Filtered thermocouple T+ | 4.31 mm | Top | 0.40 mm | 0 |
| TC_SCK | 7.40 mm | Top | 0.20 mm | 0 |
| TC_SO | 9.47 mm | Top/Bottom | 0.20 mm | 2 |
| TC_CS | 14.95 mm | Top | 0.25 mm | 0 |
| USB D+ | 28.433 mm | Top/Bottom | 0.20/0.15 mm | 2 |
| USB D- | 28.470 mm | Top/Bottom | 0.20/0.15 mm | 2 |
| RS485_A | 19.000 mm | Top | — | 0 |
| RS485_B | 18.952 mm | Top | — | 0 |
| HEATER_DAC | 24.27 mm | Top/Bottom | 0.25 mm | 2 |
| HEATER_FB_ADC | 34.34 mm | Top/Bottom | 0.25 mm | 2 |
| 24V_IN | 6.61 mm | Top | 0.50 mm | 0 |
| 24V_PROTECTED | Approximately 105 mm | Top/Bottom | 0.40/0.50 mm | 2 |
| 220 VAC contact traces | — | Top | Approximately 3.0 mm | 0 |

USB mismatch is approximately 0.037 mm. RS-485 mismatch is approximately 0.048 mm. Length matching alone does not prove impedance or EMC performance; reference-plane continuity and the manufactured stack-up remain controlling factors.

## 4. Thermocouple Layout

The K-type signal is only tens of microvolts per degree Celsius. Preserve all of the following:

1. keep P1, D3, R14, C18, and U1 grouped;
2. keep K+ and filtered T+ short and via-free;
3. keep both paths away from relay, mains, USB, RS-485, DAC output, and converter loops;
4. maintain a quiet continuous GND reference;
5. do not share a narrow return with relay-coil or switching currents;
6. keep the MAX6675 cold-junction area away from hot components and asymmetric heat flow;
7. connect any cable shield to chassis/PE at the machine boundary, not to K+ or K-.

The current signal traces are short and via-free, which is good. D3's GND pad is approximately 9.34 mm from the nearest GND via. Add an immediate GND via, preferably within 1 mm, so ESD current does not travel through the sensitive ground region.

## 5. Precision 0–5 V and ADC Feedback

Treat U3, its gain network, P2, and PA0 feedback as one analog block:

- R9/R10 directly at U3 IN-;
- R7/C8 directly at U3 IN+;
- R15/R16 together using the specified 0.1% parts;
- R17/C23 at PA0;
- feedback sensed after R11 at the actual P2 terminal node;
- P2 return through a quiet, low-impedance GND path;
- no shared narrow return with relay, RS-485 ESD, or converter current.

C15 100 nF and C20 1 µF currently sit roughly 3.9–5.8 mm from U3 V+. Move C15 immediately next to U3 V+ and GND, ideally with a loop below 2 mm; keep C20 close behind it.

The HEATER_DAC and HEATER_FB_ADC routes use two vias each. Their low bandwidth makes the lengths acceptable only if the opposite-layer GND reference remains continuous and the routes stay outside the mains isolation zone.

## 6. USB Layout

The MCU-to-D2 D+/D- routes are closely length matched and use equal layer changes. Preserve:

- coupled routing and equal via count;
- uninterrupted opposite-layer GND reference;
- no stubs or plane voids;
- D2 in the connector-to-MCU signal path;
- distance from mains, relay contacts, and converter loops.

The connector-to-D2 subsegments are less symmetric, but are short enough for USB full speed if the pair remains coupled. D2's GND connection is approximately 2.75 mm from the nearest GND via. Add a dedicated via within about 1 mm of the protection ground pad.

## 7. RS-485 Layout and Termination

The intended physical order is:

`U6 THVD1400DR -> R19/R20 10 ohm -> D4 SM712 -> P5`

The A/B pair is on the top layer with no vias and is matched to about 0.048 mm, which is good. Required release actions are:

- move D4 closer to P5; the present connector-to-protection path is roughly 8–13.5 mm, while 3–5 mm is preferred;
- add a GND via within about 1 mm of D4 pin 3; the present nearest via is approximately 2.58 mm away;
- move C25 100 nF directly to U6 VCC/GND and keep C26 1 µF close;
- retain R21 10 kΩ so DE and /RE default low, placing the transceiver in receive mode;
- keep P5 pin 1 = B and pin 2 = A consistent in schematic, silkscreen, cable, and firmware documentation;
- validate common-mode voltage because this interface is non-isolated and P5 has no dedicated ground pin.

R22 120 Ω must be an assembly option, DNP option, solder bridge, or jumper-controlled termination. Fit it only when this board is one of the two selected electrical ends of the RS-485 bus. Being at the end of one branch in a passive star does not automatically make it a valid terminated bus end; fitting 120 Ω at every branch endpoint overloads the driver and degrades the network.

## 8. Power, Ground, and ESD Returns

### 8.1 24 V path

Preserve the physical sequence P3 -> F3 -> D7 -> D8/C16/C17 protected node -> loads. F3 is correctly close to P3. Keep the TVS loop compact and route `24V_PROTECTED` away from the thermocouple and ADC returns.

### 8.2 Low-voltage rails

- C2/C3 at U5 VDD/VDDA;
- C19/C22 at VREF+ without merging VREF+ into the 3.3 V plane;
- C24 100 nF at VDDIO2/VSS and a local 4.7 µF added before release;
- LDO input/output capacitors at the corresponding pins;
- C13/C14 at `5V_SYSTEM`;
- every 100 nF bypass capacitor placed closer than its associated bulk capacitor.

### 8.3 Ground pours

Before every Gerber release:

- repour top and bottom GND;
- inspect islands and necked returns;
- confirm reference continuity beneath USB and all layer changes;
- keep GND copper out of the complete mains isolation corridor;
- add short ESD-return vias at D2, D3, and D4;
- add local stitching around USB, RS-485, and analog boundaries without creating mains-barrier violations.

## 9. Hazardous 220 VAC Relay Area

P4 and relay nets `$1N128` and `$1N129` are hazardous mains copper.

### 9.1 Measured present geometry

| Relationship | Approx. copper-edge spacing |
|---|---:|
| Mains trace to relay 24 V coil side | 3.10 mm |
| Mains trace to P2-area GND | 3.28 mm |
| Relay contact pad to relay coil pad | 3.53 mm |
| Mains copper to analog feedback area | 4.77–4.94 mm |

The project currently uses ordinary object spacing of roughly 0.18–0.20 mm and has no explicit MAINS-to-SELV net-pair rule. The three `NO_POURS` regions stop pours only; they do not stop traces, vias, pads, or manual fills. Therefore zero DRC errors do not prove safe 220 VAC separation.

### 9.2 Required isolation redesign

Use a conservative provisional target of at least 8 mm clearance and creepage between hazardous mains and every SELV conductor until the applicable product standard, overvoltage category, pollution degree, material group, altitude, and insulation class establish the final value.

Production release requires:

1. a relay and footprint with documented coil-to-contact isolation for the final application;
2. rerouting the 3 mm mains trace away from coil and low-voltage copper;
3. moving P2/U3/feedback circuitry out of the mains boundary;
4. no-copper/no-track/no-via/no-fill isolation corridors on both layers;
5. explicit MAINS-to-SELV net-pair rules for `$1N128` and `$1N129`;
6. inclusion of the unused relay contact pad in the hazardous zone;
7. verified distance to board edge, mounting holes, USB, SWD, P1, P2, P3, P5, all GND copper, and all low-voltage pads;
8. relay, terminal, trace, fuse, load-category, inrush, lifetime, enclosure, and touch-protection qualification.

A slot can increase creepage but cannot increase the air clearance between existing relay pads. A larger pour keepout alone cannot solve a 3.53 mm physical pad gap.

The current 3 mm trace width may be adequate for a low-current contactor/control circuit, but is not approved for an unspecified heater current. Record the actual continuous current, inrush, copper thickness, terminal rating, relay rating, and allowable temperature rise. Use an external suitably rated contactor if the relay is expected to carry heater power.

## 10. SWD and Production Test Access

H1 provides GND, 3.3 V target reference, NRST, SWCLK/BOOT0, and SWDIO. Production fixtures should also measure:

- `24V_PROTECTED`;
- `5V_FROM_24`;
- `5V_SYSTEM`;
- `3V3_SYSTEM`;
- VREF+ / 2.5 V;
- `HEATER_OUT_5V`;
- `HEATER_FB_ADC`;
- RS-485 A and B during a communications test.

Do not connect an ordinary low-voltage fixture to P4 while mains is present.

## 11. Net and Silkscreen Cleanup

The reviewed board contains two orphan track segments named `3V3_SYSTEM` with no connected pads, while the active rail is `3V3_System`. Delete the orphan segments and rerun ECO/DRC.

Most user-facing connector labels are currently on bottom silkscreen. Duplicate essential labels on the top side adjacent to the actual wire-entry pins:

- P1: K+ / K-;
- P2: GND / 0–5V;
- P3: GND / +24V;
- P4: NC (L) / COM (L) plus a high-voltage warning;
- P5: B / A;
- H1: GND / 3V3 / NRST / SWCLK / SWDIO.

Verify that differential-pair rule polarity labels match the actual USB and RS-485 nets; do not rely on the rule name alone.

## 12. DFM Release Checklist

Before generating production files:

- synchronize schematic and PCB with no pending ECO;
- delete orphan tracks and repour both GND planes;
- run strict DRC and a separate mains-to-SELV review;
- configure R22 as the approved termination option;
- verify D2/D3/D4 return vias and U3/U6 decoupling placement;
- verify every footprint, polarity, part number, voltage/current rating, and assembly substitution;
- verify top-side connector labels and pin 1 orientation;
- confirm solder-mask slivers, slots, annular rings, board edge, and mounting-hole strategy with the PCB supplier;
- confirm panel/global/local fiducial requirements with the assembler;
- archive Gerber, drill, BOM, CPL, schematic PDF, DRC output, mains review, test plan, and tagged Git revision together.

## 13. Production Evidence Required

Release only after documented:

1. first-article continuity and rail tests;
2. thermocouple accuracy/noise tests on the real machine;
3. 0–5 V calibration and ADC feedback under the real machine input load;
4. RS-485 topology, termination, common-mode, ESD, error-rate, and cable-length tests;
5. relay/footprint isolation redesign and mains-specific review;
6. relay functional, inrush, and life testing at the actual load;
7. USB enumeration, DFU, and SWD recovery;
8. hot/cold thermal soak;
9. applicable ESD/EFT/EMC testing;
10. independent machine-level overtemperature and power-loss safety testing.
