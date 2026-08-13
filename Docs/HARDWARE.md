# Hardware Description

## Design Basis

This document describes the connections found in the current EasyEDA PCB, not a proposed future circuit. Component references, nets, and connector pin numbers were read from PCB document `90c44d6e89e4d436`.

## Functional Blocks

### Microcontroller

The controller is an STM32C071G8U6 in a 28-pin UFQFPN package. It provides a 48 MHz Arm Cortex-M0+ CPU, 64 KiB Flash, 24 KiB SRAM, USB full-speed device support, SPI, timers, ADC, and USART peripherals. This is sufficient for MAX6675 acquisition, USB CDC communication, safety supervision, PWM generation, and a PID controller.

The MCU uses the internal clock in the current design; PC14 and PC15 are not connected to an external crystal.

### Thermocouple Interface

U1 is a MAX6675ISA+T K-type thermocouple-to-digital converter.

| MAX6675 pin | Net | Connection |
|---:|---|---|
| 1 | GND | Ground |
| 2 | GND | Thermocouple negative input, connected to P1 pin 2 |
| 3 | Thermocouple input | Thermocouple positive input, connected to P1 pin 1 |
| 4 | 3V3_SYSTEM | Supply |
| 5 | TC_SCK | MCU PA5 |
| 6 | TC_CS | MCU PA4; R1 provides a 10 kΩ pull-up |
| 7 | TC_SO | MCU PA6 |
| 8 | NC | Not connected |

C1 is a 100 nF local supply bypass capacitor. The converter has a read-only SPI-compatible interface, 0.25 °C resolution, open-thermocouple detection, and a nominal measurement range of 0–1024 °C.

### Filtered Proportional Output

The `HEATER_PWM` signal is generated on MCU PA8. R7 and C8 form the low-pass filter:

- R7: 1 kΩ (`0603WAF1001T5E`).
- C8: 10 µF (`CL21A106KAYNNNE`).
- R8: 10 kΩ (`0603WAF1002T5E`) from the raw PWM node to ground, ensuring a low output while the MCU pin is high-impedance.

The nominal filter time constant is 10 ms and the nominal cutoff frequency is approximately 15.9 Hz. P2 pin 2 is the filtered output and P2 pin 1 is ground. The receiving equipment should present a high input impedance; this output is a control signal and cannot drive a heater directly.

### USB-C Interface

USB is implemented with a 16-pin USB-C receptacle.

- Both USB 2.0 D+ contacts are joined and routed through D2 to MCU PA12.
- Both USB 2.0 D− contacts are joined and routed through D2 to MCU PA11.
- CC1 and CC2 each have a 5.1 kΩ pull-down resistor to ground, identifying the board as a USB power sink/device.
- D2 is a USBLC6-2SC6 USB ESD array.
- F1 is an SMD1206P050TF/13.2 resettable fuse between raw VBUS and `USB_5V`.
- D1 is an SMF5.0A TVS diode on `USB_5V`.
- C5 is 4.7 µF on the protected USB supply.
- The connector shield pads are connected to board ground in the current PCB.

### 24 V to 5 V Power Path

P3 supplies external control power.

1. F2 (`ASMD1812-020-60V`) protects the 24 V input against sustained overcurrent.
2. D3 (`SS36`) is the series reverse-polarity protection diode.
3. D4 (`SMAJ24A`) clamps positive transients on the protected input rail.
4. C9 is 22 µF / 50 V and C10 is 100 nF at the converter input.
5. U2 (`K7805-500R3`) converts the protected 24 V rail to `5V_FROM_24`.
6. C11 is 22 µF / 25 V and C12 is 100 nF at the converter output.

The `K7805-500R3` is a switching regulator module, not a linear 7805. Its absolute input range, derating, output ripple, and required capacitor recommendations must be checked against the exact purchased datasheet before release.

### Power-Source ORing and 3.3 V Rail

`5V_FROM_24` and `USB_5V` are diode-ORed into `5V_SYSTEM`:

- D5: B5819W SL from `5V_FROM_24` to `5V_SYSTEM`.
- D6: B5819W SL from `USB_5V` to `5V_SYSTEM`.
- C13: 10 µF and C14: 100 nF on `5V_SYSTEM`.

This prevents either source from directly back-feeding the other. The Schottky drop means `5V_SYSTEM` will normally be below the active source voltage.

The ME6211C33M5G-N LDO converts `5V_SYSTEM` to `3V3_SYSTEM`. Its CE pin is tied high to `5V_SYSTEM`; C4 and C6 are 4.7 µF input and output capacitors.

### Reset and Boot Controls

- NRST has a 10 kΩ pull-up (R4), a 100 nF capacitor (C7), and a pushbutton to ground.
- PA14/BOOT0 has a 10 kΩ pull-down (R6). The BOOT button drives the node high through R5, 1 kΩ.
- Holding BOOT and pressing RESET selects the STM32 system-memory bootloader when the option-byte configuration permits it.

## Main Parts

| Reference | Part | LCSC code | Function |
|---|---|---|---|
| STM32 | STM32C071G8U6 | C44521811 | MCU |
| U1 | MAX6675ISA+T | C16030 | K-type thermocouple converter |
| U2 | K7805-500R3 | C19188491 | 24 V to 5 V converter |
| 5to3.3 | ME6211C33M5G-N | C82942 | 3.3 V LDO |
| USB | TYPE-C 16PIN 2MD(073) | C2765186 | USB-C receptacle |
| D2 | USBLC6-2SC6 | C2827654 | USB data ESD protection |
| D1 | SMF5.0A | C169426 | USB supply TVS |
| D3 | SS36 | C116712 | Reverse-polarity protection |
| D4 | SMAJ24A | C148222 | 24 V transient clamp |
| D5, D6 | B5819W SL | C8598 | Supply ORing |
| F1 | SMD1206P050TF/13.2 | C43379 | USB resettable fuse |
| F2 | ASMD1812-020-60V | C2982277 | 24 V resettable fuse |
| RST, BOOT | TS-1088-AR02016 | C720477 | Pushbuttons |
| P1 | WJ500V-5.08-2P | C8465 | Thermocouple terminal |

## BOM Data That Still Requires Cleanup

The EasyEDA project does not currently contain valid LCSC purchasing codes for every item:

- P2 and P3 use copied supplier identifiers rather than confirmed `C` numbers.
- R7, R8, and C8 contain manufacturer-part strings in the supplier-ID field.
- R1–R6 and C1–C3/C7 have values and footprints but no confirmed LCSC ordering code.

Assign valid supplier parts before requesting assembly pricing or generating a final BOM.

## Hardware Safety Requirements

- Treat sensor-open as a hard shutdown, not merely a displayed warning.
- Add independent over-temperature protection in the machine power stage.
- Ensure P2 ground is compatible with the external controller. Add isolation if the machine control ground is noisy or at a different potential.
- Do not place the MAX6675 near U2, D3, D4, the heater power stage, or other heat sources; its package temperature is the cold-junction reference.
- Verify all components on the 24 V input for the real supply tolerance and surge environment. A nominal 24 V industrial rail can exceed 24 V during transients.
