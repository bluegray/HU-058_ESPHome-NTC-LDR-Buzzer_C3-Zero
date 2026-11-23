# Bare-metal panel firmware

Drives the HU-058D display from an ESP32-C3 Zero with nothing but the Arduino
core.

Everything specific to this board sits in the pin constants and tables near the
top of `src/main.cpp`, and the only hardware-dependent code is those constants
and the GPIO register macros. It is also the fastest way to find out which LED
positions your particular board has fitted.

The mapping it uses is documented in `../../../docs/display-map.md` and matches
the ESPHome config in `../../esphome/clock.yaml`. The full 16-pin mapping and
the C3 wiring notes are in the top-level README.

The color scheme here is the simple one: three duty levels per channel, from
at most two sub-frames per COM pair, which is close to what the stock 8051
does. The ESPHome component uses four binary weighted sub-frames and gets
sixteen levels. Start here, then read that if you want more colors.

## Wiring

Four signals and a ground, into the empty DIP-16 MCU footprint. Do not install
the STC8G1K17 (the stock 8051) or the ESP-01S, and pull them if you already
did. Two devices on one bus will fight.

Full build notes are in `../../../docs/wiring.md`; its wire table is the
WROOM-32 version, while the C3 pins are in the table below.

| C3 GPIO | Socket pin | Net | Use |
| --- | --- | --- | --- |
| GPIO0 | 14 on HU-058D; 16 on HU-058 / HU-058SE | CLK | Driver 1 clock |
| GPIO1 | 5 | DATA | Driver 1 data |
| GPIO6 | 1 | CLK_1 | Driver 2 clock |
| GPIO7 | 2 | DATA_1 | Driver 2 data |
| GND | 8 | GND | Common ground |

On the bench, power the C3 Zero from its USB-C port and the clock from its own
USB, and tie the grounds together through socket pin 8. Do not also wire the
C3's 5V pin to the clock's 5V rail in that setup. Use the C3 Zero's own USB-C
port for flashing and the serial monitor.

## Build

```bash
pio run -t upload
pio device monitor
```

The `platformio.ini` selects `esp32-c3-devkitm-1` and builds with
`ARDUINO_USB_CDC_ON_BOOT=1`, so the monitor works over the C3's USB-C port. If
the upload does not start, hold BOOT, tap RESET, release BOOT, and retry.

## Commands

Single keypresses over the USB serial monitor.

| Key | Pattern |
| --- | --- |
| `l` | Lamp test, all red, IS 0x5 |
| `t` | 2:12 AM in red with the colon lit |
| `d` | 1234 in red, green, blue and purple |
| `6` | Week display `-6-` in blue |
| `w` | Toggle single position walk, one step per second |
| `n` | Next walk step |
| `+` `-` | Raise or lower IS |
| `?` | This list |

Walk: It lights one of the 40 addressable positions at a
time and prints which driver, COM pair and LED it is driving, so you can read
your board's real population straight off the panel.


## Levels

The 3.3V outputs of the ESP32-C3 drive the panel perfectly without a level
shifter.

If levels ever are marginal the symptom is not a dead display but an
intermittent one, because the latch triggers on a data edge while the clock is
high. Watch for flicker, a wrong digit for one frame, or garbage that clears
itself. A 74AHCT125 on the four lines is the fix.

## Current draw

The sinks are constant current, so total draw is the number of lit sinks times
`IS` and does not fall with the scan duty. The vendor rates the board at
250mA.

| State | Both drivers | Per driver dissipation |
| --- | --- | --- |
| Lamp test, all red, IS 0x5 | 180mA | 0.27W |
| Lamp test, all red, IS 0xF | 485mA | 0.73W |
| Full white, IS 0xF | 1.2A | 1.4W |

The heat lands in the driver rather than the LED, since each sink drops VDD
minus the LED forward voltage. Do not leave the panel at a high `IS` with most
of it lit.

Nothing here commands full white, but the level array can express it.
