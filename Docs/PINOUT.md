# STM32C071G8U6 Pinout

## Assigned Signals

The table below is derived from the current PCB netlist. Package-pin numbers refer to the 28-pin STM32C071G8U6 device.

| Package pin | MCU pin | Current PCB function | Direction | Notes |
|---:|---|---|---|---|
| 1 | PC14-OSC32_IN | Unused | — | No 32.768 kHz crystal fitted |
| 2 | PC15-OSC32_OUT | Unused | — | No 32.768 kHz crystal fitted |
| 3 | VDD/VDDA | 3V3_SYSTEM | Power | MCU digital and analog supply |
| 4 | VSS/VSSA | GND | Power | MCU digital and analog ground |
| 5 | PF2-NRST | RESET | Input | 10 kΩ pull-up, 100 nF to GND, reset button to GND |
| 6 | PA0 | Unused | — | Available only by PCB modification; no connector |
| 7 | PA1 | Unused | — | Available only by PCB modification; no connector |
| 8 | PA2 | Unused | — | No connector |
| 9 | PA3 | Unused | — | No connector |
| 10 | PA4 | TC_CS | Output | MAX6675 active-low chip select; 10 kΩ pull-up |
| 11 | PA5 | TC_SCK | Output | MAX6675 serial clock |
| 12 | PA6 | TC_SO | Input | MAX6675 serial data output |
| 13 | PA7 | Unused | — | No connector |
| 14 | PB0 | Unused | — | No connector |
| 15 | PB1 | Unused | — | No connector |
| 16 | PA8 | HEATER_PWM | Output | Timer PWM input to R7/C8 filter |
| 17 | PC6 | Unused | — | No connector |
| 18 | PA11 | USB D− | Bidirectional | USB full-speed negative data |
| 19 | PA12 | USB D+ | Bidirectional | USB full-speed positive data |
| 20 | PA13 | SWDIO function, not routed | Bidirectional | No SWD connector in the current PCB |
| 21 | PA14-BOOT0 | BOOT / SWCLK function | Input / debug | 10 kΩ pull-down; BOOT button drives high through 1 kΩ; no SWD connector |
| 22 | PA15 | Unused | — | No connector |
| 23 | PB3 | Unused | — | No connector |
| 24 | PB4 | Unused | — | No connector |
| 25 | PB5 | Unused | — | No connector |
| 26 | PB6 | Unused | — | No connector |
| 27 | PB7 | Unused | — | No connector |
| 28 | PB8 | Unused | — | No connector |

## Firmware Constants

Use symbolic STM32 pin names rather than raw package-pin numbers:

```cpp
constexpr PinName PIN_TC_CS      = PA_4;
constexpr PinName PIN_TC_SCK     = PA_5;
constexpr PinName PIN_TC_SO      = PA_6;
constexpr PinName PIN_HEATER_PWM = PA_8;
```

Depending on the STM32 Arduino core API selected by PlatformIO, application code may use `PA4`/`PA5`/`PA6`/`PA8` instead of the `PinName` constants above. Verify the selected generic STM32C071G8 variant before compiling.

## Connector Pinout

| Connector | Pin 1 | Pin 2 |
|---|---|---|
| P1 thermocouple | `K+` | `K-` |
| P2 proportional output | `GND` | `OUT` |
| P3 control power | `GND` | `+24V` |

Orient silkscreen labels at the actual wire-entry side of each terminal. Do not rely only on the schematic symbol orientation, because the physical footprint may be rotated on the PCB.

## USB and Debugging Constraints

- PA11 and PA12 are committed to native USB on this board.
- No independent UART header is present. Use USB CDC for command and diagnostic text during normal operation.
- SWD remains an MCU capability, but PA13/SWDIO and PA14/SWCLK are not routed to a debug header in the present layout. A standard ST-Link cannot be attached without test access or a board revision.
- A blank MCU must therefore be programmed through the ROM USB DFU path using the BOOT and RESET buttons, unless factory programming is arranged separately.
- Because PA14 is shared with BOOT0/SWCLK, any future SWD header must account for the existing BOOT pull-down and button network.

## Reserved Firmware Behavior

- Drive `HEATER_PWM` low immediately after reset and before enabling the timer.
- Keep `TC_CS` high while idle.
- Do not enable an alternate function on USB pins while USB CDC or DFU is active.
- Treat every currently unused pin as unconnected; do not assume it reaches a test point.
