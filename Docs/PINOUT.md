# Pinout Reference

## 1. MCU

The current PCB uses **U5 STM32G0B1CBT6N in LQFP-48**. Do not use the non-N EasyEDA symbol or the former STM32C071/STM32G030 pin definitions. The N pinout uses package pins 30 and 31 for the dedicated VSS/VDDIO2 supply pair.

### 1.1 Complete LQFP-48 map

| Package pin | MCU pin/function | Board net | Project use |
|---:|---|---|---|
| 1 | PC13 | — | Reserved |
| 2 | PC14-OSC32_IN | — | Reserved |
| 3 | PC15-OSC32_OUT | — | Reserved |
| 4 | VBAT | 3V3_SYSTEM | Backup-domain supply tied to 3.3 V |
| 5 | VREF+ | VREFBUF_2V5 | Internal 2.5 V reference output; C19/C22 only |
| 6 | VDD/VDDA | 3V3_SYSTEM | Digital and analog supply |
| 7 | VSS/VSSA | GND | Digital and analog ground |
| 8 | PF0-OSC_IN | — | Reserved |
| 9 | PF1-OSC_OUT | — | Reserved |
| 10 | PF2-NRST | NRST | Reset button and SWD header |
| 11 | PA0 | HEATER_FB_ADC | ADC input for actual P2 voltage |
| 12 | PA1 | RS485_DE | USART2_RTS_DE, AF1; active-high transmit enable |
| 13 | PA2 | RS485_TX | USART2_TX, AF1 |
| 14 | PA3 | RS485_RX | USART2_RX, AF1 |
| 15 | PA4 | HEATER_DAC | DAC1_OUT1 heating command |
| 16 | PA5 | TC_SCK | MAX6675 serial clock |
| 17 | PA6 | TC_SO | MAX6675 serial data input |
| 18 | PA7 | — | Reserved |
| 19 | PB0 | RELAY_CONTROL | Relay MOSFET gate control |
| 20 | PB1 | — | Reserved |
| 21 | PB2 | — | Reserved |
| 22 | PB10 | — | Reserved |
| 23 | PB11 | — | Reserved |
| 24 | PB12 | — | Reserved |
| 25 | PB13 | TC_CS | MAX6675 active-low chip select |
| 26 | PB14 | — | Reserved |
| 27 | PB15 | — | Reserved |
| 28 | PA8 | — | Reserved; not the heating DAC |
| 29 | PA9 | — | Reserved |
| 30 | VSS | GND | Ground for the VDDIO2 supply domain |
| 31 | VDDIO2 | 3V3_SYSTEM | Dedicated I/O/USB supply; C24 100 nF plus local 4.7 µF required before release |
| 32 | PA10 | — | Reserved |
| 33 | PA11 [PA9] | USB D- | USB device data minus |
| 34 | PA12 [PA10] | USB D+ | USB device data plus |
| 35 | PA13 | SWDIO | SWD data |
| 36 | PA14-BOOT0 | SWCLK/BOOT0 | SWD clock and boot-mode strap |
| 37 | PA15 | — | Reserved |
| 38 | PD0 | — | Reserved |
| 39 | PD1 | — | Reserved |
| 40 | PD2 | — | Reserved |
| 41 | PD3 | — | Reserved |
| 42 | PB3 | — | Reserved |
| 43 | PB4 | — | Reserved |
| 44 | PB5 | — | Reserved |
| 45 | PB6 | — | Reserved |
| 46 | PB7 | — | Reserved |
| 47 | PB8 | — | Reserved |
| 48 | PB9 | — | Reserved |

The bracketed labels on pins 33 and 34 are alternate-function labels shown by the EasyEDA symbol. The routed USB pair is PA11/PA12.

VDDIO2 is a power input, not a GPIO. Keep it at the same 3.3 V rail as VDD, never leave it floating, and never apply it while VDD is absent. Pin 30 must remain a direct GND connection.

### 1.2 Firmware pin definitions

The production firmware uses STM32Cube HAL/LL. Keep one board-specific header as the single source of truth, using GPIO ports, GPIO pin masks, and alternate-function constants rather than Arduino board-number aliases. The following is the required mapping; Cube-generated macro names may differ, but the physical pins must not:

```c
#define HEATER_FB_GPIO_Port       GPIOA
#define HEATER_FB_Pin             GPIO_PIN_0

#define RS485_DE_GPIO_Port        GPIOA
#define RS485_DE_Pin              GPIO_PIN_1
#define RS485_DE_AF               GPIO_AF1_USART2

#define RS485_TX_GPIO_Port        GPIOA
#define RS485_TX_Pin              GPIO_PIN_2
#define RS485_TX_AF               GPIO_AF1_USART2

#define RS485_RX_GPIO_Port        GPIOA
#define RS485_RX_Pin              GPIO_PIN_3
#define RS485_RX_AF               GPIO_AF1_USART2

#define HEATER_DAC_GPIO_Port      GPIOA
#define HEATER_DAC_Pin            GPIO_PIN_4
#define TC_SCK_GPIO_Port          GPIOA
#define TC_SCK_Pin                GPIO_PIN_5
#define TC_SO_GPIO_Port           GPIOA
#define TC_SO_Pin                 GPIO_PIN_6
#define RELAY_GPIO_Port           GPIOB
#define RELAY_Pin                 GPIO_PIN_0
#define TC_CS_GPIO_Port           GPIOB
#define TC_CS_Pin                 GPIO_PIN_13
```

PA1, PA2, and PA3 use USART2 alternate function AF1. PA1 may be controlled by the USART hardware driver-enable function through `HAL_RS485Ex_Init()` or an equivalent verified LL configuration. Do not configure PA1 as an unrelated RTS signal.

## 2. External Connectors

### 2.1 P1 — K-type thermocouple

| Pin | Silkscreen | Net | Requirement |
|---:|---|---|---|
| 1 | K+ | K+ | Positive thermocouple lead |
| 2 | K- | GND / MAX6675 T- | Negative lead; grounded by this circuit |

Only an insulated/ungrounded-junction K-type probe is qualified. Do not connect a grounded-junction probe without redesigning and validating the isolation/grounding strategy.

### 2.2 P2 — heating command

| Pin | Silkscreen | Net | Function |
|---:|---|---|---|
| 1 | GND | GND | Signal return |
| 2 | 0-5V | HEATER_OUT_5V | Monitored analog heating command |

P2 is a signal output, not a heater-power output.

### 2.3 P3 — machine power

| Pin | Silkscreen | Net | Function |
|---:|---|---|---|
| 1 | GND | GND | 24 V return |
| 2 | +24V | 24V_IN | Nominal stable 24 V input |

### 2.4 P4 — hazardous 220 VAC relay contact

| Pin | Silkscreen | Net | Function |
|---:|---|---|---|
| 1 | NC (L) | `$1N128` / relay NC | Hazardous 220 VAC normally closed contact |
| 2 | COM (L) | `$1N129` / relay COM | Hazardous 220 VAC common contact |

P4 is separate from P2. With no board power or `RELAY_CONTROL` low, NC and COM are connected. Both pins are hazardous live during machine operation and are not SELV test points. The current PCB isolation geometry is not yet approved for production mains use.

### 2.5 P5 — non-isolated RS-485

| Pin | Silkscreen | Net | Function |
|---:|---|---|---|
| 1 | B | RS485_B | RS-485 B conductor |
| 2 | A | RS485_A | RS-485 A conductor |

P5 carries only A and B. U6 THVD1400DR is not galvanically isolated, so both systems must share a validated ground reference elsewhere in the machine. The resulting ground-potential difference and bus common-mode voltage must remain within the transceiver ratings under normal operation and faults.

R22 is the optional 120 ohm A-to-B termination. Populate it only when this PCB is selected as one of the two physical ends of an RS-485 segment. Being the endpoint of a passive branch does not by itself justify termination, and a passive multi-branch bus must not place 120 ohm at every branch endpoint.

Always verify A/B interoperability with the actual external controller because some equipment vendors use opposite A/B naming conventions.

### 2.6 USB1 — USB Type-C

USB1 implements a USB 2.0 device:

- both D+ contacts join `USB D+`;
- both D- contacts join `USB D-`;
- CC1 and CC2 each have a 5.1 kohm pull-down;
- VBUS becomes `VBUS_RAW`, then passes through the USB power-protection block;
- connector GND and shield tabs connect to board GND.

## 3. SWD Header H1

H1 is a 1x5, 2.54 mm programming/debug header.

| H1 pin | Signal | Connect to debugger |
|---:|---|---|
| 1 | GND | GND |
| 2 | 3V3_SYSTEM | VTref / target reference |
| 3 | NRST | nRESET |
| 4 | SWCLK/BOOT0 | SWCLK |
| 5 | SWDIO | SWDIO |

The debugger must not source an incompatible target voltage. H1 pin 2 is primarily the target-voltage reference unless the approved programming procedure explicitly powers the board from the probe.

## 4. Buttons

| Button | Normal state | Pressed state |
|---|---|---|
| RST | NRST pulled high by 10 kohm | NRST connected to GND |
| BOOT | PA14/BOOT0 pulled low by 10 kohm | BOOT0 connected to 3.3 V through 1 kohm |

To enter the ROM bootloader manually:

1. disconnect or stop an active SWD session;
2. hold BOOT;
3. press and release RST;
4. release BOOT.

The exact USB DFU behavior must match the STM32G0B1 entry in ST application note AN2606 and the programmed option bytes.

## 5. Signal Polarity Summary

| Signal | Active state | Meaning |
|---|---|---|
| TC_CS | Low | Select/read MAX6675 |
| RELAY_CONTROL | High | Energize coil and open P4 NC-COM |
| HEATER_DAC | Analog | Increasing voltage increases requested heating |
| RS485_DE | Low | Transmitter disabled and receiver enabled; reset/idle state |
| RS485_DE | High | Transmitter enabled and receiver disabled |
| NRST | Low | Reset MCU |
| BOOT0 | High during reset | Select system-memory boot path |

Firmware must not assume that a low relay-control output interrupts mains. Low or unpowered means the normally closed P4 circuit remains closed.

Because U6 ties `DE` and active-low `/RE` together, the bus direction changes as one unit. Firmware must return `RS485_DE` low only after the USART transmission-complete condition, not merely after the transmit-data register becomes empty.
