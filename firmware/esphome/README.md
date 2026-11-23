# ESPHome firmware

Firmware to drive two AiP33628 drivers, plus a config that puts
the panel in Home Assistant as a single light.

Mapping, scan timing and the current budget are in
`../../docs/display-map.md`. Every entity and action this exposes is in
`../../docs/home-assistant.md`.

## Start here

The checked-in config targets an ESP32-C3 Zero. The full 16-pin mapping is in
the top-level README; this page covers the ESPHome side.

I recommend you build and flash the firmware onto the C3 Zero before you wire
anything to the clock board. An ESP32 connected via USB with nothing attached
will still boot, join Wi-Fi and turn up in Home Assistant. Then wire the
display and buttons, plus the sensors and buzzer, and the panel lights up.

1. Create the venv and install ESPHome, then fill in `secrets.yaml` (see
   Requirements and Setup below).
2. `esphome run clock.yaml` over USB, from this directory.
3. Adopt the device in Home Assistant.
4. Wire the display and buttons; the full mapping is in the top-level README.
5. Wire the NTC, LDR and buzzer; see the hardware-mods section in the top-level
   README.

Use the firmware in `../esp32/panel-test/` as it drives the panel with no Wi-Fi
and no Home Assistant, so if the display misbehaves it tells you whether the
hardware is right before you start suspecting this component.

## Requirements

ESPHome 2026.8.0 or newer, on Python 3.11 or newer. This repo was tested with
ESPHome 2026.11.0 on Python 3.14.10.

Install it into a virtual environment. Recent Linux distributions and Homebrew
refuse a plain `pip install` into the system Python, and a user install puts
the `esphome` command somewhere that is often not on PATH.

```bash
python3.14 -m venv venv314
source venv314/bin/activate
pip install --upgrade esphome
```

On Windows, create it with `py -3.14 -m venv venv314` and activate with
`venv314\Scripts\activate`. Every `esphome` command below assumes the environment
is active, and you activate it again in each new terminal.

## Which ESP32

The checked-in `clock.yaml` targets an **ESP32-C3 Zero**
(`esp32-c3-devkitm-1`), using the mapping in the top-level README. It uses the
ESP-IDF framework because the logger is on USB Serial/JTAG.

The component still builds for the classic **ESP32-WROOM-32**. To switch back,
set `board: esp32dev`, remove the `logger: hardware_uart: USB_SERIAL_JTAG`
line (the classic ESP32 has no USB Serial/JTAG), and use the WROOM wiring in
`../../docs/wiring.md`. A plain WROOM-32 devkit, the common 30 or 38 pin board
that sells for a few dollars, is what the original build used. Anything in
that family works unchanged: WROOM-32, WROVER, DevKitC, NodeMCU-32S. If the
listing says ESP32 with no letter after it, that is the one. Those boards have
an onboard 5V to 3.3V regulator, so the clock's 5V rail can power them.

The four display pins have to sit below GPIO32. A frame goes out as a single
store to the low GPIO output register, and that register only reaches GPIO0
through GPIO31. `__init__.py` checks this and refuses to build if you pick a
higher pin. Every header pin on the C3 Zero already qualifies.

### S2, S3, C3 and C6

`aip33628.cpp` selects the GPIO register form at compile time and
`__init__.py` re-enables ESP-IDF's `esp_driver_gptimer`, so the frame loop
builds for both the classic ESP32 and the newer chips. For the C3 Zero the
config uses `board: esp32-c3-devkitm-1`, ESP-IDF and the USB Serial/JTAG logger
because the buttons use GPIO20/21. The C3 config has been compile-tested with
ESPHome 2026.11.0; it has not been run on hardware here.

### ESP8266

Nope, this requires an ESP32. The scan leans on `gptimer` and on that single-store frame send, so porting it is a real project rather than a config change.

## Setup

```bash
cp secrets.yaml.example secrets.yaml
```

On Windows that is `copy` instead of `cp`.

Fill in the Wi-Fi, timezone, API key, OTA and fallback AP values. `ap_password`
is the password for the clock's fallback access point; the SSID is the friendly
name plus ` Setup` (from `wifi: ap: ssid` in `clock.yaml`), and the password
needs 8 to 63 characters.

The timezone in `secrets.yaml` is a fallback for a boot with no Home Assistant.
Change it to yours.

Home Assistant pushes its own timezone on every time sync and that path wins,
but only while the `homeassistant` time platform has no timezone of its own.
Do not add one there.

## Build and flash

```bash
esphome run clock.yaml
```

The first build under ESP-IDF compiles the framework and takes longer than an
Arduino build. On the C3 Zero, use its USB-C port; hold BOOT and tap RESET only
if it does not enter download mode. On a WROOM-32 devkit, USB flashing may need
BOOT held while programming.

Once it is on the network, updates go over the air, and the API carries the log
stream:

```bash
esphome run clock.yaml --device wifi-clock.local
esphome logs clock.yaml --device wifi-clock.local
```

Logs are also on the C3 Zero's USB Serial/JTAG, so `esphome logs` can read them
over USB before Wi-Fi is up.

Two things to know before pushing an OTA build:

- **A bad build is not recoverable remotely.** If it fails to bring up Wi-Fi
  the only way back is a USB cable, which means opening the case. Compile
  before uploading. The fallback access point in `clock.yaml` is the safety
  net, so look for its setup SSID before assuming a flash is dead.
- **Do not open the USB serial port for about a minute after an OTA.** On
  adapters that assert DTR, opening the port resets the board before ESPHome
  marks the new partition valid, and the device rolls back to the previous
  image with no error anywhere.

## Adopt in Home Assistant

Home Assistant finds the device on its own. Look under Settings > Devices and
Services for a discovered ESPHome node.

It asks for an encryption key. That is `api_key` out of your `secrets.yaml`.

Do this before you wire anything. Until the device is adopted the clock shows
the time and ignores everything else, because the light, the switches and
every effect entity live on the Home Assistant side.

## Wiring

Six signal wires plus ground into the empty DIP-16 MCU footprint: four display
signals and two buttons. The NTC, LDR and buzzer are on GPIO3, GPIO4 and
GPIO5; the top-level README has the full 16-pin mapping and the sensor/buzzer
mods.

`../../docs/wiring.md` is the original WROOM-32 build guide; the C3 pins are in
the table below and in the top-level README.

| C3 GPIO | Pin | Net |
| --- | --- | --- |
| GPIO0 | 14 on HU-058D; 16 on HU-058 / HU-058SE | CLK, driver 1 |
| GPIO1 | 5 | DATA, driver 1 |
| GPIO6 | 1 | CLK_1, driver 2 |
| GPIO7 | 2 | DATA_1, driver 2 |
| GPIO20 | 9 | S1, top button |
| GPIO21 | 10 | S2, bottom button |
| GND | 8 | GND |

## Configuration

```yaml
aip33628:
  id: panel
  clk_pin: GPIO0
  data_pin: GPIO1
  clk2_pin: GPIO6
  data2_pin: GPIO7
  time_id: ha_time
  twelve_hour: false
  blink_colon: true
  max_current: 15
```

| Key | Default | What |
| --- | --- | --- |
| `clk_pin`, `data_pin` | required | Driver 1 bus, below GPIO32 |
| `clk2_pin`, `data2_pin` | required | Driver 2 bus, below GPIO32 |
| `time_id` | none | A time source. Without one the panel shows dashes. |
| `max_current` | 15 | Ceiling on `IS[3:0]`, 0 to 15 |
| `twelve_hour` | true | 12h or 24h. Power on default, the switch owns it |
| `blink_colon` | true | Power on default, the switch owns it after that |

The light platform takes an `aip33628_id` and is otherwise a normal RGB light.

## What it does

- Shows the time
- Colon blinks once a second, and the upper dot drops while the network is
  down, so a glance at the panel says whether the time is still being kept
  accurately.
- Four dashes while there is no valid time. The HU-058D has no RTC, and the
  RTC on the HU-058/HU-058SE is unused by this firmware, so a cold boot with no
  network means no time at all until the network comes back.
- One RGB "light" entity for the whole display, plus per digit color, per LED
  color, gradients, a color cycle and a flash effect.
- Temporary number displays light the degree mark when their unit is C or F.
- A lamp test lights every populated LED white for three seconds, then restores
  the previous display settings.

## Color resolution

Each COM slot is split into four binary weighted sub-frames, which gives 16
duty cycle levels per channel and 4096 colors.

The AiP33628 has no grayscale engine, so every one of those levels costs a
sub-frame. More levels would mean more sub-frames per COM pair, a shorter tick
and a narrower window. Four is where the flicker margin still looks
comfortable at 416Hz.

## Current

`max_current` is a fixed ceiling on `IS[3:0]`, 0 to 15. The Home Assistant
brightness slider maps onto that range and nothing else touches it.

It deliberately does not depend on what is on screen. Counting lit sinks to
hold a milliamp budget gives you a panel that dims itself once a second as the
colon blinks, gets brighter when you pick a saturated color because fewer
sinks are lit, and stalls part way through a fade.

The stock firmware never did this either. It held `IS` at 0xD across colon on
and colon off, and ran 0xF with a white digit lit, drawing 404mA against a
250mA label.

If the drivers run hotter than you want, lower `max_current`. That trades
brightness for heat without making the brightness depend on the content.

## Brightness

Sixteen current steps, spread across the whole slider.

Gamma correction lives in the component, and `gamma_correct` on the light is
set to 1.0 so it is not applied twice.

It has to live there because a plain gamma curve assumes the output can reach
zero, and this panel bottoms out at `IS` 0, which is 2.5mA. With a stock curve
the bottom 29 percent of the slider sits clamped against that floor doing
nothing. The component interpolates perceived output between the floor and
full instead, which uses all sixteen steps and leaves only the natural width
of one step at the bottom.

Color components go through the same exponent before they are rounded onto
duty cycle levels, because duty is linear light and Home Assistant sends gamma
encoded values. Skipping that step rounds every pastel up toward a saturated
color, most visibly by turning pink into white.

Both arrive linear from the light platform. The magnitude comes from
`current_values_as_brightness()` rather than `current_values.get_brightness()`,
because only the helper carries the transition state, without which a fade
never moves.

## Scan timing

The scan runs from a `gptimer` interrupt, not from `esp_timer`. ESPHome excludes
`esp_driver_gptimer` from ESP-IDF builds by default; `__init__.py` re-enables it
for this component.

The esp_timer task dispatch path runs at task priority on core 0 next to the
WiFi task, which preempts it and stretches whichever COM slot is lit at the
time. A full duty slot rides that out. A 40us sub-frame does not, and the same
jitter reads as uneven digits and a visible pulse on any color that is not
saturated.
