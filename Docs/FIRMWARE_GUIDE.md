# Firmware Development Guide

## 1. Toolchain and Target

The intended stack is:

- Visual Studio Code;
- PlatformIO;
- STM32Cube HAL as the primary framework, with LL for timing-critical or low-overhead paths;
- STM32CubeProgrammer for ROM DFU and recovery;
- an ST-Link-compatible SWD probe for initial programming and debugging.

The exact hardware target is **STM32G0B1CBT6N, LQFP-48**. The `N` package pinout adds the VDDIO2/VSS supply pair. This is a hardware supply difference, but the firmware device family, flash size, linker layout, USB peripheral, and pin assignments must still match STM32G0B1CB.

The custom PCB may not have a built-in PlatformIO board ID. If necessary, commit a project-specific manifest named `rufajx_g0b1cb`. Do not silently build for an unrelated Nucleo board. The current repository does not yet contain firmware or a custom board manifest; these files must be added before claiming a buildable target.

An RTOS is not required. Use a fixed-rate, non-blocking scheduler and keep the control algorithm as a testable C/C++ module.

```ini
[env:rufajx_g0b1cb]
platform = ststm32
framework = stm32cube
board = rufajx_g0b1cb
upload_protocol = stlink
debug_tool = stlink
monitor_speed = 115200
```

Lock the PlatformIO platform, STM32CubeG0 package, compiler, USB middleware, linker script, and board manifest after validation.

Primary references:

- [STM32CubeG0 firmware package](https://github.com/STMicroelectronics/STM32CubeG0)
- [PlatformIO ST STM32 platform](https://docs.platformio.org/en/latest/platforms/ststm32.html)
- [STM32G0B1 datasheet](https://www.st.com/resource/en/datasheet/stm32g0b1me.pdf)
- [STM32 system-memory boot mode, AN2606](https://www.st.com/resource/en/application_note/an2606-stm32microcontroller-system-memory-boot-mode-stmicroelectronics.pdf)

## 2. Firmware Pin Map

| Function | MCU pin | Direction/peripheral |
|---|---|---|
| Thermocouple clock | PA5 | GPIO output |
| Thermocouple data | PA6 | GPIO input |
| Thermocouple chip select | PB13 | GPIO output, active low |
| Heating DAC | PA4 | DAC1_OUT1 |
| Heating-output feedback | PA0 | ADC input |
| Relay control | PB0 | GPIO output, active high |
| RS-485 direction | PA1 | USART2_RTS_DE, AF1, active-high transmit enable |
| RS-485 transmit | PA2 | USART2_TX, AF1 |
| RS-485 receive | PA3 | USART2_RX, AF1 |
| USB D- | PA11 | USB FS |
| USB D+ | PA12 | USB FS |
| SWDIO | PA13 | SWD |
| SWCLK/BOOT0 | PA14 | SWD / boot strap |
| Reset | PF2-NRST | Hardware reset |

Hardware-only supply pins:

- package pin 30 VSS -> GND;
- package pin 31 VDDIO2 -> 3.3 V.

No firmware configuration replaces these physical supply connections.

## 3. Deterministic Startup

Startup must default to no analog heating request and no RS-485 drive:

1. keep DAC disabled or at zero;
2. configure PB0 low immediately; note that this leaves the normally closed 220 VAC relay path closed;
3. configure PA1/RS485_DE low before USART initialization so the transmitter is disabled and the receiver is enabled;
4. initialize the independent watchdog;
5. initialize the verified system clock, HSI48/CRS path, and USB;
6. enable VREFBUF in 2.5 V mode;
7. wait for the VREFBUF ready flag;
8. calibrate and enable ADC;
9. enable DAC1_OUT1 at zero;
10. initialize MAX6675 pins with TC_CS high;
11. initialize USART2 and the bounded RS-485 parser;
12. collect valid temperature and output-feedback samples;
13. run self-tests;
14. enter IDLE only if all checks pass.

Do not enable heating merely because USB, RS-485, or 24 V is present. The normally closed relay cannot guarantee a safe startup state; independent machine protection must already be active.

## 4. Software Architecture

| Module | Responsibility |
|---|---|
| `board_pins` | Single source of truth for physical pins |
| `thermocouple` | MAX6675 transaction, open-circuit detection, timestamps |
| `temperature_filter` | Sample validation, spike rejection, light filtering, and rate estimation |
| `analog_output` | VREFBUF, DAC, calibration, clamp, slew limiting |
| `output_feedback` | ADC sampling, voltage reconstruction, mismatch detection |
| `control` | Direction-aware state machine, prediction, feed-forward, and scheduled PI/PID |
| `safety` | Limits, watchdog, reset reason, fault latching |
| `relay` | Explicit normally closed mains-relay semantics |
| `usb_service` | USB CDC commands, telemetry, DFU request |
| `rs485_transport` | USART2 half-duplex direction, framing, CRC, timeouts, and field commands |
| `settings` | Versioned, CRC-protected calibration and tuning profiles |

The control loop, USB parser, RS-485 parser, and logging must not block the 250 ms temperature schedule or watchdog service. HAL callbacks and interrupts should enqueue bounded work; protocol parsing and telemetry formatting run outside interrupt context.

## 5. MAX6675 Acquisition

### 5.1 Transaction

1. keep TC_CS high between reads;
2. pull TC_CS low;
3. clock 16 bits MSB first using PA5/PA6;
4. raise TC_CS;
5. reject the sample if the open-thermocouple bit is set;
6. extract the 12-bit temperature value and multiply by 0.25 °C.

Use a 250 ms acquisition period. Never reuse a stale sample without checking its timestamp.

### 5.2 Validation

Reject or latch a fault for:

- open-thermocouple status;
- invalid bit patterns;
- temperature outside the configured sensor/process range;
- sample timeout;
- discontinuity beyond a validated rate of change;
- implausibly stuck data while the process should be changing;
- excessive disagreement with the reference instrument during service/calibration.

Use a short median/spike rejector followed by a low-rate exponential filter. Never average through an open-circuit or invalid-data event.

## 6. VREFBUF, DAC, and 0–5 V Command

### 6.1 Reference startup

VREF+ is the internal VREFBUF output. Firmware must:

1. select the 2.5 V range;
2. enable the buffer;
3. wait for the `VRR`/reference-ready flag;
4. verify that ADC/DAC initialization completed before control is enabled.

C19 1 µF and C22 100 nF are the local reference capacitors. Never drive PA4 as a digital output while the analog path is active.

### 6.2 Command conversion

```text
Ideal output = DAC_code / 4095 * 2.5 V * 2.05
Ideal code for 5.000 V = 4095 * 5.000 / 5.125 = approximately 3995
```

Use measured calibration:

```text
DAC_code = clamp((requested_voltage - output_offset) / volts_per_code,
                 minimum_code,
                 calibrated_5V_code)
```

Store offset, volts-per-code, calibration version, and CRC in nonvolatile memory. Never use code 4095 as the normal 5 V command.

### 6.3 Safe output behavior

- command 0 V during reset, DFU, update, self-test, and fault;
- clamp every result to 0.000–5.000 V before DAC conversion;
- reject NaN/infinite values;
- reset or back-calculate the PID integrator after saturation or mode changes;
- apply a validated slew limit during normal control;
- bypass the normal slew limit when a safety fault commands zero;
- compare the actual P2 voltage with the command.

## 7. ADC Output Feedback

The PA0 ADC samples the actual P2 node after R11.

```text
divider_ratio = 51 / (56 + 51) = 0.4766355
Vout_estimated = ADC_code / 4095 * VREF_actual / divider_ratio
```

At 5.000 V output, the ADC input is nominally 2.383 V. Use hardware-filter settling time and digital oversampling/averaging before comparison.

Required logic:

- wait for the DAC/op-amp/filter to settle after a normal step;
- distinguish stuck-high from unable-to-rise behavior;
- log command, raw ADC code, reconstructed voltage, and error;
- latch persistent disagreement;
- never allow the feedback trim loop to exceed the 0–5 V clamp.

An initial engineering threshold such as 0.15 V for 500 ms may be used during bring-up, but production thresholds must come from real-load and temperature testing.

## 8. Predictive Temperature-Control Design

### 8.1 Controlled plant and optimization objective

The hot-air fan runs continuously at a fixed speed. The firmware controls heating only:

```text
0% request   -> 0.000 V
100% request -> 5.000 V
Vcommand     = 5.000 * request_fraction
```

Heating is active and fast; cooling is passive and substantially slower. The controller must therefore use different logic and identified parameters for rising and falling setpoints.

The primary performance objective is minimum settling time, defined as the elapsed time from a setpoint command until the filtered temperature remains inside the configured stability band for the configured dwell time. Engineering defaults are:

- stability band: setpoint ±2 °C;
- stability dwell: 30 s;
- cooling undershoot soft target: no more than 10 °C;
- heating overshoot: measured and logged, but not used as the primary optimization target.

These engineering values are service-panel parameters. Safety limits remain independent of performance tuning. Large heating overshoot normally increases settling time because the constant-speed fan cannot actively remove stored heat, so the predictor should still avoid unnecessary overshoot.

### 8.2 Measurement and trend estimation

Run acquisition and control on a fixed 250 ms schedule:

1. validate the MAX6675 frame, open-circuit bit, timestamp, range, and discontinuity;
2. pass valid samples through a rolling spike/median rejector;
3. apply a light low-pass filter, initially around a 0.5 s time constant;
4. estimate `dT/dt` by linear regression over an initial 2 s history window;
5. estimate a bounded, low-pass-filtered acceleration term only if real data proves it improves prediction.

Do not calculate rate from only two adjacent 0.25 °C samples. Do not use heavy averaging that adds significant measurement delay.

### 8.3 States

| State | Output and purpose |
|---|---|
| `IDLE` | DAC = 0; validate configuration, sensor, reference, and output feedback |
| `FULL_HEAT` | High/full allowed power while safely far below a higher setpoint |
| `HEAT_APPROACH` | Reduce power using predicted residual rise before the setpoint |
| `ZERO_COOL` | DAC = 0 for the fastest available passive cooldown |
| `COOL_BRAKE` | Restore controlled heating before the falling temperature undershoots excessively |
| `HOLD` | Feed-forward plus gain-scheduled PI/PID regulation |
| `FAULT` | Immediate DAC = 0, latched fault, and relay-open request when 24 V is available |

A new setpoint is classified immediately. A higher target selects the heating path; a lower target selects the cooling path. A small change that is already within the hold entry region may remain in `HOLD`. A fault overrides every state.

### 8.4 Short-horizon prediction

Use separately identified heating and cooling horizons:

```text
v = regression_slope(temperature_history)
a = bounded_filtered_rate_change(v)
Tpred(H) = Tfiltered + v * H + 0.5 * a * H * H
```

- `H_up(T)` represents residual rise/coasting behavior after heating is reduced.
- `H_down(T)` represents the delay before restored heating affects the sensor during cooldown.
- If the acceleration estimate is not repeatable, set `a = 0` and retain the more robust first-order prediction.
- Clamp rate, acceleration, horizon, and predicted temperature to validated physical ranges.

`H_up` and `H_down` must come from the actual heater, thermal mass, airflow, probe position, and operating temperature. They are not interchangeable.

### 8.5 Rising setpoint

In `FULL_HEAT`, command the configured maximum allowed output while evaluating:

```text
Tcoast = Tpred(H_up)
```

Enter `HEAT_APPROACH` when `Tcoast` reaches the configured prediction margin below the setpoint. Keep the integrator disabled and calculate an approach command such as:

```text
u = clamp(uFF(setpoint)
          + Kapproach_up * (setpoint - Tpred)
          - Krate_up * dT/dt,
          0,
          Umax)
```

The controller may permit a small or moderate transient overshoot when that reduces total settling time. It must never intentionally cross a safety trip threshold.

### 8.6 Falling setpoint

On a lower setpoint, immediately bypass normal slew limiting, command DAC = 0, freeze/back-calculate the integrator, and enter `ZERO_COOL`. This is the fastest available cooling action with a constant-speed fan.

During cooldown, evaluate the temperature expected when restored heating can affect the sensor:

```text
Tfree = Tpred(H_down)
```

Enter `COOL_BRAKE` before the actual temperature reaches the setpoint when `Tfree` crosses the configured recovery margin. Restore power smoothly using the target feed-forward value and predicted error:

```text
u = clamp(uFF(setpoint)
          + Kapproach_down * (setpoint - Tpred),
          0,
          Ucool_limit)
```

The default cooldown tuning should aim to keep transient undershoot within 10 °C without sacrificing the primary minimum-settling-time objective.

### 8.7 Hold controller

Identify steady heating command at 300, 450, and 600 °C and linearly interpolate `uFF`, `Kp`, `Ki`, and optional `Kd` between the validated points:

```text
error = setpoint - Tfiltered
u_raw = uFF(setpoint) + Kp(setpoint) * error + integral - Kd(setpoint) * dT/dt
integral += (Ki(setpoint) * error + Kaw * (u_sat - u_raw)) * dt
u = slew_limit(clamp(u_raw, 0, 1))
```

Use derivative on measurement, back-calculation anti-windup, and bumpless transfer. On entry to `HOLD`, preload the integrator so the requested output does not jump:

```text
integral = previous_output - uFF - proportional - derivative
```

The default hold-entry condition is temperature inside ±2 °C with absolute rate below 0.2 °C/s. Settling-time qualification additionally requires the temperature to remain inside the stability band for the configured 30 s dwell. State-transition hysteresis and minimum dwell times prevent chatter.

### 8.8 Setpoint changes and fault interaction

- Accept normal control setpoints only inside the compiled process range, currently up to 600 °C.
- Re-evaluate direction and prediction immediately after every accepted setpoint change.
- Never carry an accumulated heating integral into `ZERO_COOL`.
- Reject start, setpoint, and tuning-apply commands while `FAULT` is latched.
- After a fault reset, return to `IDLE` with DAC = 0; require a separate explicit start command.

### 8.9 Parameter ownership

The USB service panel may edit parameters, but the MCU is the authority and validates every value.

| Class | Examples | Apply policy |
|---|---|---|
| Live bounded tuning | setpoint, `Kp`, `Ki`, `Kd`, feed-forward, output limit, prediction margins | May apply while running with bumpless transfer |
| IDLE-only engineering | `H_up`, `H_down`, filters, state thresholds, stability band/dwell, overtemperature settings, reset margin | Apply only in `IDLE` with commanded and measured P2 below 0.1 V |
| Non-bypassable firmware policy | 0–5 V clamp, 600 °C setpoint ceiling, 800 °C absolute ceiling, immediate fault-to-zero behavior, manual fault reset, settings validation | The service panel cannot disable or exceed these limits |

Use a staged edit/validate/apply/save transaction. Store settings with schema version, length, sequence number, and CRC using an atomic or dual-slot update. Invalid settings prevent heating rather than silently reverting during a run.

## 9. Safety and Relay Handling

### 9.1 Immediate fault action

Every control-critical fault must synchronously:

1. bypass the normal output slew limiter;
2. write DAC code zero;
3. clear or back-calculate the controller integrator;
4. enter and latch `FAULT`;
5. use PA0 ADC feedback to verify that P2 falls below 0.1 V within the validated output-discharge interval; and
6. if 24 V is present, set PB0 high to energize the coil and open P4.

The ADC is an input monitor. PA4 DAC is the signal that is forced to zero.

### 9.2 Mandatory latched faults

- thermocouple open, invalid, implausible, or stale for more than 750 ms;
- measured temperature at or above the service-panel absolute limit, whose default and compiled maximum are 800 °C;
- measured temperature at or above `setpoint + relative_overtemperature_delta`, with a 70 °C default;
- output feedback remaining high while the DAC command is zero;
- persistent command/feedback disagreement after the allowed settling interval;
- VREFBUF, ADC, or DAC initialization failure;
- watchdog reset during heating;
- invalid calibration or settings CRC;
- control-loop timing failure or internal assertion.

Both overtemperature comparisons run independently; either one trips. The browser cannot raise the absolute limit above the compiled 800 °C ceiling or disable the relative limit.

### 9.3 Manual reset gate

Accept a manual reset request only when all of the following remain true:

- the thermocouple has produced valid samples continuously for at least 5 s;
- measured P2 feedback is below 0.1 V;
- temperature is below `setpoint + reset_margin`, with a 30 °C default; and
- the settings/calibration records pass validation.

A successful reset enters `IDLE`, leaves DAC at zero, and does not restart heating.

Critical limitation: with board power absent, PB0 is low, the coil is de-energized, and the 220 VAC NC-COM path is closed. The relay is not an independent fail-safe and cannot replace a thermostat, thermal fuse, safety relay/contactor, or equivalent hardware protection.

The current PCB is also blocked from production mains use until the relay/footprint isolation redesign in `PCB_REQUIRED_FIXES.md` is completed.

## 10. USB CDC Service Interface

Recommended commands:

- read temperature, raw MAX6675 word, and sample age;
- set/clear setpoint within policy limits;
- read state, predicted temperature, PID terms, and output command;
- read raw ADC and reconstructed P2 voltage;
- read faults and reset reason;
- stage, validate, apply, save, export, and restore bounded tuning profiles;
- report settling time, peak temperature, minimum temperature, state transitions, and fault history;
- request DFU.

Never provide a service command that bypasses temperature, voltage, or fault limits.

Recommended 4 Hz telemetry fields are timestamp, setpoint, raw and filtered temperature, `dT/dt`, predicted temperature, state, DAC command, reconstructed P2 voltage, feed-forward, P/I/D terms, active tuning profile, and fault flags. USB disconnection must not stop the local controller or relax any safety check.

## 11. RS-485 Field Interface

U6 ties active-high DE and active-low /RE together. PA1 low is the reset and idle state: the transmitter is disabled and the receiver is enabled. PA1 high enables transmission and disables reception.

Prefer the verified USART2 hardware driver-enable function on PA1 using HAL_RS485Ex_Init() or an equivalent LL configuration. Set DE polarity and assertion/deassertion times from oscilloscope measurements. If direction is controlled manually, assert PA1 before starting the frame and return it low only after the USART transmission-complete flag confirms that the final stop bit has left the shift register. TXE alone is not sufficient.

The transport must provide:

- a bounded receive buffer and maximum frame length;
- address, function/command, payload length, and CRC validation;
- inter-byte and complete-frame timeouts;
- rejection of malformed, repeated, stale, or unauthorized writes;
- rate-limited telemetry and diagnostics;
- counters for CRC, framing, overrun, timeout, and bus-contention errors;
- deterministic recovery to receive mode after timeout or reset.

The application may use Modbus RTU or a versioned project-specific binary protocol; select and document one before implementation. Do not silently combine both interpretations on the same port.

RS-485 may read telemetry, set a bounded setpoint, and stage validated tuning parameters. It must use the same authorization, range, state, CRC, and IDLE-only policies as USB. No field-bus command may bypass the 0–5 V clamp, overtemperature trips, sensor validation, fault latch, manual-reset gate, or watchdog.

P5 carries A and B only. The hardware is non-isolated and relies on a verified common ground elsewhere in the machine. Firmware cannot correct excessive ground offset or common-mode voltage. Communications loss must be logged and handled according to the machine policy, but it must never disable local temperature safety or force an unsafe output.

R22 120 ohm is populated only when this board is one of the two selected electrical bus ends. Termination is a machine-wiring/assembly decision, not a firmware setting.
## 12. USB DFU and Recovery

The STM32G0B1 system-memory bootloader supports USB DFU on PA11/PA12. The board routes these pins to USB-C and provides manual BOOT/RESET plus SWD recovery.

### 12.1 Manual entry

1. hold BOOT;
2. press and release RESET;
3. release BOOT;
4. connect with STM32CubeProgrammer in USB mode.

### 12.2 Software-requested entry

A USB CDC `dfu` command may:

1. validate the request;
2. command 0 V and verify feedback falls;
3. request the relay open if 24 V is present and the machine requires it;
4. stop control tasks and flush the response;
5. store a retained/no-init RAM magic value;
6. reset;
7. detect and clear the magic in earliest startup;
8. deinitialize interrupts, SysTick, USB, clocks, and peripherals;
9. load the ROM bootloader stack/vector using the address from the current STM32G0B1 AN2606 entry;
10. jump and allow USB to re-enumerate as STM32 DFU.

Never copy a boot address from another STM32 family. Test this path after every linker, core, option-byte, or boot configuration change. SWD is the recovery path if DFU fails.

Entering ROM DFU stops application safety control. Heating must already be disabled by an independent machine mechanism.

## 13. Production Calibration and Plant Identification

### 13.1 Temperature

Compare each probe/board combination, or the qualified sampling plan, against a traceable reference at:

- room/ambient;
- one mid-range process point;
- one high point near the intended maximum.

Record probe batch, fixture, airflow, soak time, board temperature, raw data, reference result, and residual error.

### 13.2 0–5 V output

With the actual machine input connected:

1. verify 0 V with DAC disabled;
2. measure approximately 0.5, 2.5, 4.5, and 5.0 V;
3. fit offset and volts-per-code;
4. verify ADC feedback at every point;
5. store coefficients and CRC;
6. repeat at expected low/high board temperatures;
7. define pass limits for offset, gain, nonlinearity, noise, and settling.

### 13.3 Thermal plant

Keep the fan at its normal fixed speed and log every 250 ms control sample. Begin at the lowest safe temperature and progress to higher temperatures only after the preceding test passes.

1. Exercise bounded 20%, 40%, 60%, 80%, and 100% output steps to characterize heating rate versus temperature and command.
2. During controlled heating, command zero and record continued temperature rise, peak temperature, and decay to identify `H_up`.
3. During passive cooldown, restore a known bounded command and measure sensor response delay to identify `H_down`.
4. Establish steady feed-forward commands at 300, 450, and 600 °C.
5. Fit separate rising and falling first-order-plus-dead-time models where they adequately describe the data.
6. Tune prediction/approach behavior with the integrator disabled.
7. Add proportional control, then only enough integral to remove static error; add derivative only if measured settling improves.
8. Store validated temperature-scheduled parameters and retain raw test logs with firmware and hardware revision identifiers.

Do not optimize against heating overshoot alone. Compare parameter sets using total settling time while recording peak temperature, cooling undershoot, output saturation, and every safety margin.

## 14. Bring-Up Order

1. Visual inspection and resistance checks with power removed.
2. Current-limited USB power; verify 5 V, 3.3 V, VDDIO2, and reset behavior.
3. SWD chip identification, erase, program, reset, and debug.
4. Verify VREF+ reaches 2.5 V after VREFBUF is enabled.
5. Verify USB CDC enumeration and reset behavior.
6. Hold RS485_DE low, initialize USART2, and verify receive-idle behavior.
7. Test RS-485 transmit direction, final-stop-bit timing, CRC rejection, and automatic return to receive mode using safe short cabling.
8. Test MAX6675 with a known insulated probe and open circuit.
9. Apply current-limited 24 V; verify protected 24 V and 5 V conversion.
10. Sweep P2 with a high-impedance load and compare ADC feedback.
11. Verify PB0 high opens P4 NC-COM using safe low voltage only.
12. Test manual DFU, software DFU, and SWD recovery.
13. Complete the PCB mains-isolation redesign and safety review.
14. Use an isolated/interlocked fixture for any 220 VAC relay test.
15. Integrate with the machine at limited heating power.
16. Validate the final RS-485 topology, cable, common-ground offset, selected termination, and error rate.
17. Identify the thermal plant and tune staged control only after sensor/output safety tests pass.
## 15. Release Tests

Release evidence must cover:

- pin-map compile-time checks;
- MAX6675 extraction, open-circuit, stale, noisy, and implausible data;
- DAC saturation and calibration;
- ADC reconstruction and mismatch faults;
- all control-state transitions;
- rising and falling prediction with separately identified horizons;
- 300 to 600 °C, 500 to 300 °C, and ±10/±30 °C setpoint transitions;
- time to enter and remain in the configured ±2 °C stability band for the configured dwell;
- long-duration hold tests at 300, 450, and 600 °C;
- PID anti-windup and bumpless transfer;
- service-panel parameter bounds, IDLE-only policy, atomic save, CRC rejection, and profile restore;
- absolute-limit and `setpoint + 70 °C` overtemperature trips;
- manual fault reset gates: 5 s valid sensor, P2 below 0.1 V, and temperature below `setpoint + 30 °C`;
- watchdog and reset-reason handling;
- active-high relay/open-contact behavior;
- USB command bounds and malformed inputs;
- RS-485 DE timing, framing/CRC/timeouts, malformed writes, bus contention, termination, common-mode, and communications-loss behavior;
- manual DFU, software DFU, and SWD recovery;
- USB-only, 24 V-only, and dual-source power cycling;
- independent machine overtemperature shutdown;
- isolated 220 VAC relay functional and endurance testing after hardware approval.
