# Firmware Development Guide

## Toolchain

The intended application stack is PlatformIO with the official STM32 Arduino core.

Recommended development tools:

- Visual Studio Code with the PlatformIO extension.
- PlatformIO `ststm32` platform with `framework = arduino`.
- STM32CubeProgrammer for ROM DFU programming and recovery.
- A USB CDC terminal for commands and diagnostic output.

The official STM32 Arduino core supports generic STM32C071G8 devices from core version 2.11.0. Pin the platform/core versions after the first validated build so future package updates do not silently change USB, clock, timer, or upload behavior.

Because this is a custom board, confirm that the installed PlatformIO platform exposes a compatible STM32C071G8 board definition. If it does not, add a project-local board manifest such as `boards/rufajx_stm32c071g8u6.json`; do not select a different MCU merely because its board ID exists.

A starting `platformio.ini` structure is:

```ini
[env:rufajx_stm32c071g8u6]
platform = platformio/ststm32
framework = arduino
board = rufajx_stm32c071g8u6
upload_protocol = dfu
monitor_speed = 115200
build_flags =
  -D PIO_FRAMEWORK_ARDUINO_ENABLE_CDC
```

The custom board manifest must specify STM32C071G8U6, 64 KiB Flash, 24 KiB SRAM, the correct STM32C0 variant, and the correct upload limits. Validate the final settings against the installed STM32 core and linker script.

## Hardware Pin Definitions

```cpp
constexpr PinName PIN_TC_CS      = PA_4;
constexpr PinName PIN_TC_SCK     = PA_5;
constexpr PinName PIN_TC_SO      = PA_6;
constexpr PinName PIN_HEATER_PWM = PA_8;
```

See [PINOUT.md](PINOUT.md) for the complete package-pin table and debug limitations.

## MAX6675 Acquisition

The MAX6675 output is a 16-bit read-only frame:

- Bits 14:3 contain the temperature code.
- Each count represents 0.25 °C.
- Bit 2 indicates an open thermocouple.
- Bit 15 and bits 1:0 are not temperature data.

Use an SPI mode compatible with the MAX6675 timing diagram and keep chip select high between transfers. Allow at least the datasheet conversion interval; a 250 ms control sample period is a safe initial value.

Recommended acquisition checks:

1. Read the complete 16-bit frame.
2. Reject the sample immediately if the open-thermocouple bit is set.
3. Reject impossible values, communication timeouts, and large single-sample discontinuities.
4. Convert valid raw counts to degrees Celsius.
5. Apply only light measurement filtering so control delay is not excessive.

The fail-safe response to any invalid sensor state is `HEATER_PWM = 0` and a latched fault requiring an explicit clear command.

## PWM-to-Voltage Output

PA8 drives the R7/C8 low-pass filter. With R7 = 1 kΩ and C8 = 10 µF:

- Time constant: approximately 10 ms.
- Cutoff frequency: approximately 15.9 Hz.
- Ideal average output: `VOUT ≈ duty × 3.3 V` for a high-impedance load.

Configure a timer PWM frequency well above the filter cutoff. A starting value of 20 kHz provides low ripple and remains inaudible; at a 48 MHz timer clock it also permits useful duty-cycle resolution. Verify the real timer clock and Arduino-core timer allocation before fixing the period.

At startup:

1. Configure PA8 as a low output before enabling the PWM peripheral.
2. Start with 0% duty cycle.
3. Enable heating only after a valid sensor reading, valid setpoint, and completed safety checks.

The output-voltage relationship should be calibrated on assembled hardware because GPIO high voltage, diode drops in the supply path, RC tolerance, and external input loading affect the result.

## Control Strategy

Use a state machine around the PID controller rather than running unrestricted PID from power-up.

### Suggested States

| State | Behavior |
|---|---|
| `SAFE_OFF` | Output 0%; wait for valid configuration and sensor |
| `RAPID_HEAT` | High or full output while safely below target |
| `APPROACH` | Reduce output before the target using temperature slope and thermal inertia |
| `PID_HOLD` | Closed-loop regulation around the setpoint |
| `FAULT` | Output 0%; latch and report the cause |

### PID Recommendations

- Start with a 250 ms control interval to match the MAX6675 conversion rate.
- Use proportional-on-error and derivative-on-measurement to reduce derivative kick.
- Add integral anti-windup whenever output saturates at 0% or 100%.
- Freeze or back-calculate the integrator during `RAPID_HEAT` and faults.
- Apply output clamps and an optional slew-rate limit.
- Estimate temperature slope with a filtered derivative.
- Before reaching the setpoint, reduce power using a predicted temperature such as `T_predicted = T_measured + tau × dT/dt`; tune `tau` from step-response data.
- Use different entry and exit thresholds for mode transitions to prevent chatter.

Do not choose final PID gains from calculations alone. Record a safe step response on the real heater, thermal mass, probe location, and airflow configuration, then tune and validate at several setpoints between 200 °C and 600 °C.

## Mandatory Safety Logic

Force the output to zero for any of these conditions:

- Open thermocouple or invalid MAX6675 frame.
- Temperature above the configured hard limit.
- Temperature rise while output is commanded off.
- No reasonable temperature rise during prolonged high-power operation.
- USB command timeout, if external supervision is mandatory.
- Internal watchdog reset or corrupted configuration.
- Control-loop timing overrun.

Use the independent watchdog. Store configuration with a version, length, and CRC. Never resume heating automatically after a watchdog reset, brownout, or firmware update unless the machine-level safety analysis explicitly permits it.

## USB CDC Command Interface

Use native USB CDC for normal commands and diagnostics because the current PCB has no separate UART connector.

Suggested commands:

```text
STATUS
SET TEMP <degrees_C>
SET ENABLE <0|1>
SET PID <kp> <ki> <kd>
SET LIMIT <degrees_C>
SAVE
FAULT CLEAR
DFU
```

Return structured, machine-readable responses. Include measured temperature, predicted temperature, setpoint, duty cycle, filtered output estimate, controller state, fault flags, and firmware version in `STATUS`.

## USB DFU and Firmware Recovery

### First Programming or Manual Recovery

1. Connect the USB-C cable.
2. Hold BOOT.
3. Press and release RESET.
4. Release BOOT.
5. Confirm that the STM32 ROM DFU device appears in STM32CubeProgrammer.
6. Program and verify the firmware.

The exact boot mode depends on option bytes and the STM32C071 system-memory bootloader behavior. Verify the production option-byte configuration against ST application note AN2606.

### Software-Requested DFU

The firmware can support a `DFU` command over USB CDC. A robust implementation should:

1. Authenticate or otherwise guard the command if unintended updates are a risk.
2. Force heater output to 0% and latch the safe state.
3. Flush the CDC response and stop control peripherals.
4. Store a one-time DFU request marker in a retained location supported by the MCU design.
5. Trigger a system reset.
6. In the earliest startup code, consume the marker, deinitialize clocks and interrupts, and transfer control to the documented STM32C071 ROM bootloader entry.
7. Let the ROM bootloader re-enumerate as a USB DFU device.

Do not hard-code a system-memory address copied from another STM32 family. Derive the entry sequence and address from the current STM32C071 reference documentation and AN2606. Keep the physical BOOT/RESET recovery path even after software DFU works.

## Bring-Up Sequence

1. Inspect for shorts and power the board from a current-limited USB supply.
2. Verify `USB_5V`, `5V_SYSTEM`, and `3V3_SYSTEM` before fitting or enabling external 24 V power.
3. Enter ROM DFU with BOOT/RESET and program a minimal firmware image.
4. Confirm USB CDC enumeration and reset behavior.
5. Read room-temperature data from the MAX6675 and test sensor-open detection.
6. Measure PWM and filtered output at 0%, 25%, 50%, 75%, and 100% duty.
7. Repeat power checks from a current-limited 24 V supply; confirm no back-feed into USB VBUS.
8. Connect the external heater controller with heater power disabled and confirm 0–3.3 V scaling.
9. Perform low-temperature closed-loop tests before testing the full 200–600 °C range.
10. Validate every fault path with real hardware before enabling unattended heating.

## References

- [PlatformIO ST STM32 platform documentation](https://docs.platformio.org/en/latest/platforms/ststm32.html)
- [Official STM32 Arduino core](https://github.com/stm32duino/Arduino_Core_STM32)
- [STM32 Arduino upload methods](https://github.com/stm32duino/Arduino_Core_STM32/wiki/Getting-Started)
- [STM32 system-memory boot modes (AN2606)](https://www.st.com/resource/en/application_note/an2606-stm32-microcontroller-system-memory-boot-mode-stmicroelectronics.pdf)
- [USB DFU protocol in STM32 bootloaders](https://www.st.com/resource/en/application_note/cd00264379-usb-dfu-protocol-used-in-the-stm32-bootloader-stmicroelectronics.pdf)
- [MAX6675 datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/max6675.pdf)
