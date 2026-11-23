# HU-058 ESPHome — NTC, LDR and Buzzer on an ESP32-C3 Zero

Open firmware for the HU-058 family clocks, with added buzzer, NTC and light
sensor. This README documents the ESP32-C3 Zero pin mapping used by the
checked-in ESPHome config; the full project write-up is in the original readme.

## Original readme

The full project description, wiring notes, calibration details, example Home
Assistant automations and firmware overview live in the original readme:

👉 **[README.original.md](README.original.md)**

## Where things are

| Path | What |
| --- | --- |
| `docs/` | Wiring, hardware, protocol, display map, LED layout, Home Assistant entities |
| `firmware/` | ESPHome component/config and the bare-metal PlatformIO panel test |
| `reference/` | Datasheet and the C3 Zero pinout |

See the original readme for the documentation index and details.

## ESP32-C3 Zero pins and features

The C3 Zero is a SuperMini-class board: a single-core 160 MHz RISC-V
ESP32-C3 with 4 MB flash, 400 KB SRAM, 2.4 GHz Wi-Fi and BLE 5, USB-C via the
chip's native USB/JTAG (GPIO18/19), an onboard antenna and a 5 V to 3.3 V
regulator. The header exposes 5V, 3V3, GND and GPIO0–10 plus GPIO18–21
(GPIO11–17 are used by the SPI flash); GPIO18/19 are the USB D-/D+ lines, ADC
input is on GPIO0–4 (ADC1) and GPIO5 (ADC2), GPIO2/8/9 are strapping pins, and
this board puts the onboard WS2812 RGB LED on GPIO10, BOOT on GPIO9 and UART0
on GPIO20/21. This is not the WROOM-32 pinout once documented here:
GPIO34/35/25 do not exist on the C3.

[ESP32-C3 Zero pinout schematic](reference/esp32-c3-zero-pinout-high-resolution.svg)

## Mapping for the 16-pin MCU footprint

The C3 Zero header is labelled by GPIO number (`IO0`–`IO10`, `IO18`–`IO21`,
plus `5V`, `3V3` and `GND`), so each row below names both the DIP-16 socket pin
and the C3 GPIO. As drawn in the schematic, top to bottom, the left edge runs
5V, GND, 3V3, IO0–IO5 and the right edge runs IO21, IO20, IO19, IO18, IO10,
IO9, IO8, IO7, IO6. The display and button signals can go on almost any pin;
the choices here avoid the strapping pins, put each bus pair on adjacent pins
and keep the NTC and LDR on ADC1. This assumes the stock STC8G1K17 and ESP-01S
are removed, as described in `docs/wiring.md`.

| Socket pin | Net | C3 GPIO | Notes |
| --- | --- | --- | --- |
| 1 | CLK_1 | GPIO6 (IO6) | Display bus 2 clock, output |
| 2 | DATA_1 | GPIO7 (IO7) | Display bus 2 data, output |
| 3 | R_T | GPIO3 (IO3) | NTC, ADC1 channel 3 |
| 4 | R_P | GPIO4 (IO4) | LDR, ADC1 channel 4 |
| 5 | DATA | GPIO1 (IO1) | Display bus 1 data, output |
| 6 | +5V | 5V | Optional; powering via P3 works too |
| 7 | — | — | Not connected |
| 8 | GND | GND | Common ground |
| 9 | S1 | GPIO20 (IO20) | Top button, input with pullup |
| 10 | S2 | GPIO21 (IO21) | Bottom button, input with pullup |
| 11 | R_B | GPIO5 (IO5) | Buzzer, see the note below |
| 12 | — | — | Not connected |
| 13 | — | — | Not connected |
| 14 | CLK on HU-058D, RTC on HU-058/HU-058SE | GPIO0 (IO0) | Display bus 1 clock on the HU-058D only |
| 15 | RXD_2 | — | ESP-01S serial link on the HU-058D, unused after removal |
| 16 | CLK on HU-058/HU-058SE, TXD_2 on HU-058D | GPIO0 (IO0) | Display bus 1 clock on the HU-058/HU-058SE only |

Socket pins 14 and 16 carry the same display bus 1 clock on different board
revisions, so connect whichever one is CLK on your board — check the silkscreen
revision before soldering.

- GPIO2, GPIO8 and GPIO9 are left unused on purpose: they are the strapping
  pins, and GPIO9 is also the BOOT button. GPIO10 is not used either, because
  it drives the onboard WS2812 RGB LED.
- The NTC and LDR are on IO3/IO4: ADC1 is only on IO0–IO4, IO2 is a strapping
  pin, and IO5 is ADC2, which cannot be used while the Wi-Fi driver is running.
- GPIO20 and GPIO21 are UART0. If the logger stays on UART0 its TX will
  drive the S2 net and its RX setup can disturb S1, so move the logger to the
  USB Serial/JTAG (`hardware_uart: USB_SERIAL_JTAG`, if your ESPHome version
  supports it) or turn UART0 logging off. The ROM may also print its boot
  banner on IO21 at reset, so S2 may toggle briefly before the firmware starts.
- With socket pin 11 left high-impedance, R5 pulls the C3 pin towards the +5V
  PNP emitter and the ESD clamp holds it a little above 3.3V. The C3 is not 5V
  tolerant; a clamp or level shifter is the clean fix. Per `docs/wiring.md` the
  PNP is off with the pin high-impedance and on with it low, so leave GPIO5
  undriven until the beep script runs.
- Socket pins 3 and 4 sit on dividers whose tops are fed from +5V on a stock
  board, so they can rise above 3.3V (the LDR in the dark, the NTC in the
  cold). The original conversion already cuts that 5V feed and connects 3.3V
  instead; if you have not done that mod, do it (or add a divider) before
  connecting them to GPIO3/GPIO4.
- `clock.yaml` already uses this mapping: `board: esp32-c3-devkitm-1`,
  `clk_pin`/`data_pin` = CLK/DATA, `clk2_pin`/`data2_pin` = CLK_1/DATA_1, the
  ADC pins, both buttons and the buzzer `pin:`. The component selects the
  right GPIO register form at compile time, so only `board:` changes if you
  use a different module.
