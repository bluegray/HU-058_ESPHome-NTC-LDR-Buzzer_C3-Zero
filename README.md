# HU-058 ESPHome — NTC, LDR and Buzzer on an ESP32-C3 Zero

Open firmware for the HU-058D WiFi clock, with added buzzer, NTC and light
sensor. Built for an ESP32-C3 Zero running ESPHome.

## Original readme

The full project description, wiring notes, calibration details, example Home
Assistant automations and firmware overview live in the original readme:

👉 **[README.original.md](README.original.md)**

## Where things are

| Path | What |
| --- | --- |
| `docs/` | Wiring, hardware, protocol, display map, LED layout, Home Assistant entities |
| `firmware/` | ESPHome component/config and the bare-metal PlatformIO panel test |
| `reference/` | AiP33628 datasheet |

See the original readme for the documentation index and details.

## ESP32-C3 Zero pins and features

The C3 Zero is a SuperMini-class board: a single-core 160 MHz RISC-V
ESP32-C3 with 4 MB flash, 400 KB SRAM, 2.4 GHz Wi-Fi and BLE 5, USB-C via the
chip's native USB/JTAG (GPIO18/19), an onboard antenna and a 5 V to 3.3 V
regulator. The header exposes 5V, 3V3 and GND plus only GPIO0–10, GPIO20 and
GPIO21 (GPIO11–17 are used by the SPI flash); ADC input is on GPIO0–4 (ADC1)
and GPIO5 (ADC2), GPIO2/8/9 are strapping pins, and most boards put an RGB LED
on GPIO8, BOOT on GPIO9 and UART0 on GPIO20/21. This is not the WROOM-32 pinout
once documented here: GPIO34/35/25 do not exist on the C3.

## Suggested mapping for the 16-pin MCU footprint

The C3 Zero header is labelled by GPIO number (`IO0`–`IO10`, `IO20`, `IO21`,
plus `5V`, `3V3` and `GND`), so each row below names both the DIP-16 socket pin
and the C3 GPIO. The display and button signals can go on almost any pin; the
choices here avoid the strapping pins and keep the NTC and LDR on ADC1.

| Socket pin | Net | C3 GPIO | Notes |
| --- | --- | --- | --- |
| 1 | CLK_2 | GPIO5 (IO5) | Display bus 2 clock, output |
| 2 | DATA_2 | GPIO6 (IO6) | Display bus 2 data, output |
| 3 | R_T | GPIO3 (IO3) | NTC, ADC1 channel 3 |
| 4 | R_P | GPIO4 (IO4) | LDR, ADC1 channel 4 |
| 5 | DATA_1 | GPIO7 (IO7) | Display bus 1 data, output |
| 6 | +5V | 5V | Optional; powering via P3 works too |
| 7 | — | — | Not connected |
| 8 | GND | GND | Common ground |
| 9 | S1 | GPIO20 (IO20) | Top button, input with pullup |
| 10 | S2 | GPIO21 (IO21) | Bottom button, input with pullup |
| 11 | R_B | GPIO0 (IO0) | Buzzer, see the note below |
| 12 | — | — | Not connected |
| 13 | — | — | Not connected |
| 14 | CLK_1 on HU-058D, RTC on HU-058/HU-058SE | GPIO10 (IO10) | Display bus 1 clock on the HU-058D only |
| 15 | RXD_2 | — | ESP-01S serial link, unused once it is removed |
| 16 | CLK_1 on HU-058/HU-058SE, TXD_2 on HU-058D | GPIO10 (IO10) | Display bus 1 clock on the HU-058/HU-058SE only |

Here `CLK_n`/`DATA_n` mean display bus n. The original docs and board label
bus 1 as plain `CLK`/`DATA` and bus 2 as `CLK_1`/`DATA_1`; the rename only makes
the numbering consistent.

Socket pins 14 and 16 carry the same display bus 1 clock on different board
revisions, so either one goes to GPIO10 — check the silkscreen revision before
soldering.

- GPIO2, GPIO8 and GPIO9 are left unused on purpose: they are the strapping
  pins, and GPIO8/GPIO9 also carry the onboard RGB LED and BOOT button.
- GPIO20 and GPIO21 are UART0. If the logger stays on UART0 its TX will
  fight the S2 net, so move the logger to the native USB serial
  (`hardware_uart: USB_CDC`, where your framework supports it) or turn UART0
  logging off.
- With socket pin 11 left high-impedance, the PNP base sits at +5V and the C3
  pin is held at about 5V through R5 and its ESD clamp. The C3 is not 5V
  tolerant; a clamp or level shifter is the clean fix. Per `docs/wiring.md` the
  PNP is off with the pin high-impedance and on with it low, so leave GPIO0
  undriven until the beep script runs.
- Socket pins 3 and 4 sit on dividers fed from +5V, so they can rise above 3.3V
  (the LDR in the dark, the NTC in the cold). Reference the top of R1/R2 to 3V3
  or add a divider before connecting them to GPIO3/GPIO4.
- Update `clock.yaml` to match: `board:`, `clk_pin`/`data_pin` (CLK_1/DATA_1),
  `clk2_pin`/`data2_pin` (CLK_2/DATA_2), both ADC pins, both button pins and the
  buzzer `pin:`, plus the `send_pair_()` register-union change described in
  `firmware/esphome/README.md`.
