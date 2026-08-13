# Rufajx Temperature Control System

## Purpose

This project is a compact controller board for a K-type thermocouple heating system. Its intended process-temperature range is approximately 200–600 °C. The board measures temperature, runs the control algorithm, and provides a 0–3.3 V proportional command to an external heater driver or machine controller.

The PCB does not switch heater power directly. The external heater power stage must provide suitable isolation, current capacity, over-temperature protection, and an independent emergency shutdown path.

## Main Functions

- K-type thermocouple measurement through a MAX6675 converter.
- Closed-loop temperature control on an STM32C071G8U6 MCU.
- Fast warm-up followed by PID regulation with thermal-inertia compensation.
- Filtered 0–3.3 V analog command generated from MCU PWM.
- USB-C device connection for firmware download and USB CDC diagnostics.
- Manual BOOT and RESET buttons for recovery and system-memory DFU entry.
- Operation from either USB 5 V or an external 24 V control supply.
- Input protection for USB and 24 V power paths.

## System Overview

```text
K-type thermocouple
        |
        v
MAX6675 ---- SPI ---- STM32C071 ---- PWM ---- RC filter ---- 0–3.3 V output
                         |
                         +---- USB-C: DFU / CDC diagnostics

24 V input ---- protection ---- 5 V converter ----+
                                                   +---- diode OR ---- 5 V ---- 3.3 V LDO
USB-C VBUS ---- protection ------------------------+
```

## External Connections

| Connector | Pin | Recommended silkscreen | Function |
|---|---:|---|---|
| P1 | 1 | `K+` | K-type thermocouple positive input |
| P1 | 2 | `K-` | K-type thermocouple negative input; tied to board ground in the present design |
| P2 | 1 | `GND` | Proportional-output reference |
| P2 | 2 | `OUT` | Filtered 0–3.3 V heater command |
| P3 | 1 | `GND` | 24 V supply return |
| P3 | 2 | `+24V` | External 24 V control-power input |

## Documentation

- [Hardware](HARDWARE.md)
- [MCU pinout](PINOUT.md)
- [PCB design and release status](PCB_DESIGN.md)
- [Firmware development guide](FIRMWARE_GUIDE.md)

## Current Project Status

The schematic and two-layer PCB layout exist in the EasyEDA project. The PCB is not yet documented as production-ready: the latest automated DRC run reported two USB-C footprint clearance errors and one schematic-to-PCB netlist mismatch. Resolve these items and repeat DRC before generating production files.

## Operating and Safety Limits

- The 200–600 °C figure is the controlled process temperature, not the allowed PCB temperature.
- Keep the PCB, MAX6675 cold junction, connectors, and wiring outside the hot zone.
- Use a K-type probe and cable insulation rated above the maximum process temperature.
- The MAX6675 provides 0.25 °C digital resolution and supports measurements up to 1024 °C, but total system accuracy also depends on the thermocouple, cold-junction temperature, layout, noise, and calibration.
- On sensor-open, invalid-reading, firmware-watchdog, or over-temperature faults, force the output to 0 V.
- Do not rely on software as the only heater safety mechanism.

## Primary References

- [STM32C071G8 product page](https://www.st.com/en/microcontrollers-microprocessors/stm32c071g8.html)
- [STM32C071x8/xB datasheet](https://www.st.com/resource/en/datasheet/stm32c071cb.pdf)
- [MAX6675 product page and datasheet](https://www.analog.com/en/products/max6675.html)
- [STM32 system-memory boot modes (AN2606)](https://www.st.com/resource/en/application_note/an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf)
