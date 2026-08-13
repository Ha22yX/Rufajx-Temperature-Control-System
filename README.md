<div align="center">

# Rufajx Temperature Control System

**A custom control board for real-time hot-air heater temperature measurement and proportional regulation.**

[简体中文](README.zh-CN.md) · [Hardware](Docs/HARDWARE.md) · [Pinout](Docs/PINOUT.md) · [PCB Status](Docs/PCB_DESIGN.md) · [Firmware Guide](Docs/FIRMWARE_GUIDE.md)

![STM32C071](https://img.shields.io/badge/MCU-STM32C071-03234B?style=flat-square)
![Temperature](https://img.shields.io/badge/Target-200--600%C2%B0C-CB6C3D?style=flat-square)
![Sensor](https://img.shields.io/badge/Sensor-K--Type%20%2B%20MAX6675-2C7A68?style=flat-square)
![EDA](https://img.shields.io/badge/EDA-EasyEDA-5B63B7?style=flat-square)
![Status](https://img.shields.io/badge/Status-PCB%20validation%20required-D39B32?style=flat-square)

</div>

![Project overview image](.github/assets/readme-hero.svg)

## Why This Exists

This project was developed for Rufajx, the user's father's company, to control the heating temperature of a hot-air machine in real time. A K-type thermocouple measures the process temperature, the MCU applies fast warm-up and PID control, and the board sends a filtered 0–3.3 V proportional command to the machine's external heater controller.

The board is the measurement and control layer. It does **not** switch heater power directly, and it must be combined with an independently protected power stage.

## What Is Implemented

- K-type thermocouple conversion with MAX6675 cold-junction compensation.
- STM32C071G8U6 control core with enough Flash, RAM, timers, SPI, and USB for this application.
- Filtered 0–3.3 V command output generated from MCU PWM.
- Native USB-C for ROM DFU recovery and planned USB CDC diagnostics.
- USB 5 V and protected 24 V control-power inputs with diode ORing.
- BOOT and RESET controls for first programming and recovery.
- Hardware documentation for a 200–600 °C hot-air heating process.

## Hardware Stack

| Layer | Component | Role |
|---|---|---|
| Temperature input | K-type thermocouple + MAX6675 | Cold-junction-compensated digital temperature acquisition |
| Control | STM32C071G8U6 | Sampling, safety state machine, PID, PWM, USB |
| Command output | PA8 PWM + 1 kΩ / 10 µF RC filter | High-impedance 0–3.3 V proportional signal |
| Service interface | USB-C + USBLC6-2SC6 | DFU programming and USB CDC diagnostics |
| Field power | 24 V protection + K7805-500R3 | Converts machine control power to 5 V |
| Logic power | ME6211C33M5G-N | Generates the 3.3 V rail |
| Design source | EasyEDA `.eprj2` | Editable schematic and PCB project |

## Repository Contents

```text
.
├── PCB Design/
│   └── Rufajx温控系统.eprj2       # EasyEDA project
├── Docs/
│   ├── README.md                 # Detailed English project overview
│   ├── HARDWARE.md               # Circuit blocks and component choices
│   ├── PINOUT.md                 # MCU and connector pin assignments
│   ├── PCB_DESIGN.md             # Layout notes and release checklist
│   └── FIRMWARE_GUIDE.md         # PlatformIO, DFU, PID, and bring-up guide
└── .github/assets/
    └── readme-hero.svg           # Repository overview graphic
```

## Open the Design

1. Install and open EasyEDA.
2. Open `PCB Design/Rufajx温控系统.eprj2`.
3. Review [the pinout](Docs/PINOUT.md) and [hardware description](Docs/HARDWARE.md).
4. Before manufacturing, complete every blocking item in [the PCB release checklist](Docs/PCB_DESIGN.md).

Firmware source has not yet been added to this repository. [FIRMWARE_GUIDE.md](Docs/FIRMWARE_GUIDE.md) records the intended PlatformIO + Arduino architecture without claiming that a buildable firmware project already exists.

## External Connections

| Connector | Pin 1 | Pin 2 | Purpose |
|---|---|---|---|
| P1 | `K+` | `K-` | K-type thermocouple input |
| P2 | `GND` | `OUT` | Filtered 0–3.3 V proportional output |
| P3 | `GND` | `+24V` | Machine control-power input |

## Current Validation Status

The PCB layout exists, but it is not yet marked production-ready. The latest EasyEDA DRC reported:

- Two USB-C footprint pad-to-slot clearances of 0.171 mm against a 0.18 mm rule.
- A schematic-to-PCB netlist mismatch.
- Several BOM entries without confirmed LCSC ordering codes.

These are release-blocking items and are documented in [PCB_DESIGN.md](Docs/PCB_DESIGN.md).

## Safety

- 200–600 °C is the process-temperature target, not the PCB ambient rating.
- Sensor-open, over-temperature, watchdog, and invalid-data faults must force the command output to 0 V.
- Use independent hardware over-temperature protection and an emergency heater shutdown path.
- Keep the controller and MAX6675 cold junction away from the hot-air stream and power-stage heat.
- Validate the complete machine before unattended operation.

## Project Status

- Hardware schematic: designed
- PCB layout: designed, validation fixes required
- Firmware architecture: documented
- Firmware implementation: not yet included
- Production qualification: not completed

## Ownership

Developed for **Rufajx** as a custom hot-air machine temperature-control system. All rights are reserved unless the project owner publishes separate licensing terms.
