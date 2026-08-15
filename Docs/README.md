# Rufajx Temperature-Control Board

## 1. Purpose

This board is the measurement and command controller for a Rufajx industrial hot-air machine. It is designed to:

1. measure an insulated-junction K-type thermocouple;
2. regulate an approximately 200–600 °C process using a direction-aware state machine, short-horizon thermal prediction, feed-forward, and gain-scheduled PI/PID control;
3. produce a monitored 0–5 V heating command;
4. expose USB for firmware/service communication and SWD for recovery and debugging; and
5. provide a normally closed relay contact intended to interrupt an external 220 VAC circuit.

The 0–5 V output commands the external machine controller. P4 is a separate hazardous-mains contact path and is not part of the analog-output circuit.

## 2. Current Design Baseline

This documentation describes the latest live EasyEDA design named `PCB2`, inspected on 2026-08-15.

| Item | Current implementation |
|---|---|
| MCU | U5 STM32G0B1CBT6N, LQFP-48 N pinout |
| Temperature converter | U1 MAX6675ISA+T |
| Qualified probe type | K-type, insulated/ungrounded junction |
| Temperature command range | Approximately 200–600 °C |
| Analog command | 0–5 V at P2 |
| DAC source | PA4 / DAC1_OUT1 using the internal 2.5 V VREFBUF |
| Output amplifier | U3 TLV9351IDBVR, non-inverting gain 2.05 |
| Output verification | PA0 ADC feedback sampled after R11 |
| USB | USB 2.0 Type-C device |
| Debug/programming | 5-pin SWD header, BOOT button, RESET button |
| Primary supply | Nominal, stable 24 V DC |
| Alternate service supply | USB 5 V |
| MCU supply domains | VDD/VDDA and VDDIO2 at 3.3 V; VSS/VSSA and VSS at GND |
| PCB CAD status | Strict EasyEDA DRC: 0 reported violations under the current rules |
| 220 VAC release status | Blocked pending relay/footprint isolation redesign and mains-specific rule verification |

## 3. System Signal Flow

```text
Insulated K-type probe
        |
        v
Low-leakage ESD + 47 ohm / 100 nF input filter
        |
        v
MAX6675 --> STM32G0B1 --> validation --> predictive state machine / PI/PID
                                            |
                                            v
                                  2.5 V DAC on PA4
                                            |
                                            v
                              TLV9351 gain 2.05 + R11
                                            |
                                            +--> P2: 0–5 V output
                                            |
                                            +--> divider/filter --> PA0 ADC feedback
```

The normally closed relay is separate from P2. P4 is a 220 VAC interruption contact and must be treated as hazardous live during layout, assembly, test, and service.

## 4. External Connectors

| Connector | Pin | Recommended silkscreen | Function |
|---|---:|---|---|
| P1 | 1 | K+ | K-type thermocouple positive lead |
| P1 | 2 | K- | K-type negative lead; electrically tied to board GND |
| P2 | 1 | GND | 0–5 V command return |
| P2 | 2 | 0-5V OUT | Heating command output |
| P3 | 1 | GND | 24 V input return |
| P3 | 2 | +24V | Nominal stable 24 V input |
| P4 | 1 | NC (L) | Hazardous 220 VAC normally closed contact |
| P4 | 2 | COM (L) | Hazardous 220 VAC common contact |
| H1 | 1 | GND | SWD ground |
| H1 | 2 | 3V3 | Target reference voltage |
| H1 | 3 | NRST | Target reset |
| H1 | 4 | SWCLK/BOOT0 | SWD clock and BOOT0 strap |
| H1 | 5 | SWDIO | SWD bidirectional data |

## 5. Power Architecture

- P3 enters through F3, D7 reverse-polarity protection, D8 SMAJ24A transient suppression, and C16/C17 bypassing.
- U2 K7805-500R3 converts `24V_PROTECTED` to `5V_FROM_24`.
- USB VBUS has its own resettable fuse, TVS, and input capacitance.
- D5 and D6 diode-OR the 24 V-derived 5 V and USB 5 V into `5V_SYSTEM`.
- LDO1 ME6211C33M5G-N creates `3V3_SYSTEM`.
- U3 and the 24 V relay coil use `24V_PROTECTED`.
- U5 pin 31 VDDIO2 is tied to `3V3_SYSTEM`; pin 30 VSS is tied to GND. C24 100 nF is present. Add a local 4.7 µF capacitor at this supply pair before production release.

The K7805-500R3 is retained only because the machine supply has been confirmed stable. Normal operation must remain below its 32 V input limit with margin. The design is not specified for automotive load dump or other high-energy surge environments.

## 6. Control and Safety Model

The constant-speed fan makes the plant asymmetric: heating is active and fast, while cooling is passive and slower. Firmware therefore uses explicit states:

- startup/self-test and `IDLE`;
- full-power rapid warm-up;
- predicted heating approach;
- zero-power passive cooldown;
- predicted cooldown braking;
- feed-forward plus gain-scheduled PI/PID hold;
- latched fault.

The primary performance objective is minimum time to enter and remain within the configured stability band. Engineering defaults are ±2 °C for 30 s and a 10 °C cooling-undershoot soft target. Heating overshoot is logged but is not the primary optimization target because excessive stored heat normally increases total settling time.

The USB service panel exposes bounded tuning profiles. Live gains and feed-forward values may use bumpless updates; prediction, filter, state, and safety settings apply only in `IDLE` with P2 below 0.1 V. MCU validation, a 600 °C setpoint ceiling, an 800 °C absolute ceiling, the 0–5 V clamp, and immediate fault-to-zero behavior cannot be bypassed by the browser.

Mandatory faults include an open, stale, or implausible thermocouple; the configurable absolute overtemperature limit; temperature at least 70 °C above the setpoint by default; DAC/ADC feedback mismatch; invalid calibration; and watchdog/reset faults. Fault response writes PA4 DAC to zero immediately, verifies P2 using PA0 ADC, and requests the relay to open when 24 V is available.

Manual reset requires 5 s of valid sensor data, P2 below 0.1 V, and temperature below the setpoint plus a default 30 °C margin. Reset returns to `IDLE` and never resumes heating automatically.

The relay is normally closed. With its coil de-energized, P4 NC and COM are connected; energizing the coil opens the 220 VAC path. Loss of board power therefore closes rather than opens this path. Software, MCU power, and this relay cannot replace a separate thermostat, thermal fuse, safety contactor, or equivalent machine-level protection.

## 7. Mains-Isolation Status

The current board has top- and bottom-layer GND pours plus three multi-layer `NO_POURS` regions around the relay area. These regions prevent copper-pour fill only; they do not forbid traces, vias, pads, or manual fills.

The measured relay contact-to-coil pad edge gap is approximately 3.53 mm. The project uses a conservative production target of at least 8 mm between hazardous mains copper and all SELV copper until the applicable product standard establishes the final value. The current DRC has no explicit mains-to-SELV net rule, so zero reported violations do not prove mains safety.

The board is not ready for 220 VAC production release until the relay/footprint and PCB isolation barrier are redesigned and reviewed. See [PCB_REQUIRED_FIXES.md](PCB_REQUIRED_FIXES.md).

## 8. Documentation Map

- [HARDWARE.md](HARDWARE.md): electrical implementation and major parts
- [PINOUT.md](PINOUT.md): MCU, connector, and debug pin map
- [PCB_DESIGN.md](PCB_DESIGN.md): verified live layout state and manufacturing rules
- [FIRMWARE_GUIDE.md](FIRMWARE_GUIDE.md): PlatformIO/Arduino architecture, predictive control, calibration, service panel, and DFU
- [PCB_REQUIRED_FIXES.md](PCB_REQUIRED_FIXES.md): mandatory production-release actions in Chinese

## 9. Release Boundary

The low-voltage measurement, USB, DAC, ADC-feedback, and power functions form a coherent engineering baseline. The full board is not production-approved for 220 VAC while the isolation blockers remain. Do not infer electrical safety, heater safety, EMC compliance, analog accuracy, or relay load capability solely from a clean CAD DRC. Complete every P0 item and record first-article and machine-level evidence before approving a batch.
