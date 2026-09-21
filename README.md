# ESPHome component for Toshiba AC

An ESPHome external component that lets an ESP module replace the optional Wi-Fi adapter (RB-N105S-G / RB-N106S-G) of a Toshiba residential AC and expose the unit to Home Assistant natively. Compatible with the Suzumi / Shorai / Seiya families and other Toshiba models that speak the same UART protocol.

## About this fork

This repository is a fork of [pedobry/esphome_toshiba_suzumi](https://github.com/pedobry/esphome_toshiba_suzumi). All credit for the original protocol reverse-engineering and component architecture goes to [pedobry](https://github.com/pedobry) and prior contributors.

We forked it to focus on modern ESPHome (2026.9+) on ESP32 targets — in particular the ESP32-C6 — and to add a small number of fixes and cleanups on top of the upstream code. The functional changes are:

- **Fan-mode string handling fix.** `StringToFanLevel` compared raw pointers instead of string contents, so Home Assistant's custom fan modes (`Low-Medium`, `Medium-High`) were silently ignored. Now compared with case-insensitive string equality.
- **RX buffer hardening.** On a floating RX pin the receiver could accept the frame header byte but never a valid length, growing `rx_message_` without bound until `std::bad_alloc`. The buffer is now reserved up-front and capped at 128 bytes with a warn-and-discard on overflow. Observed on an ESP32-C6 with no UART peer connected.
- **Const-correctness and const-ref parameters** in the hot paths (`send_to_uart`, `checksum`), and moved the static handshake tables out of the header into the translation unit.
- **GCC 14 / ESPHome 2026.9 warning cleanup.** Dropped meaningless top-level `const` on return-by-value signatures, added explicit `int` casts for enum `%d` format specifiers, and wrapped `LogString*` values with `LOG_STR_ARG(...)` before passing to `%s`. Zero warnings originating from this component under the ESP-IDF 5.5 / GCC 14 toolchain.
- **Secure-by-default example configurations.** Both example YAMLs enable Home Assistant API encryption *and* encrypted OTA updates out of the box, and reference the same set of secret names so `secrets.yaml` can be shared between them.

None of the wire-protocol bytes changed. If you are running the upstream repo happily on an older ESPHome release, there is no reason to switch.

## Reference hardware architecture

The intended target for this fork is the ESP32-C6 bridge PCB described in [b1scuitdev/Toshiba-ESP32C6-Bridge](https://github.com/b1scuitdev/Toshiba-ESP32C6-Bridge). See the [images folder](https://github.com/b1scuitdev/Toshiba-ESP32C6-Bridge/tree/main/images) in that repo for the schematic, PCB, and enclosure. The firmware in *this* repo runs on that hardware unchanged.

If you are wiring up a bare ESP32 with a level shifter instead, the component still works — see the Pinout section below.

A separate repository containing the full end-to-end reference architecture (hardware + firmware + Home Assistant integration) will be published later; this component is one building block of that stack.

## Requirements

- ESPHome **2026.9** or newer. Older ESPHome releases are not supported by this fork; use the upstream [pedobry/esphome_toshiba_suzumi](https://github.com/pedobry/esphome_toshiba_suzumi) tags if you are still on 2025.x.
- ESP-IDF framework (recommended) or Arduino framework on ESP32.
- Home Assistant with the ESPHome integration.

## Supported Toshiba units

Any unit that has the optional CN22 connector and accepts the RB-N105S-G / RB-N106S-G Wi-Fi adapter, including:

- Seiya RAS-B24 J2KVG-E
- Suzumi Plus RAS-B18, B22 and B24 PKVSG-E
- Shorai Premium RAS-B18, B22 and B24 J2KVRG-E
- Daiseikai 9 RAS-B10, B13 and B16 PKVPG-E
- Shorai Edge RAS-B07, B10, B13, B16, B18, B22 and B24 J2KVSG-E
- Seiya RAS-B10, B13, B16 and B18 J2KVG-E
- Suzumi Plus RAS-B10, B13 and B16 PKVSG-E
- Shorai Premium RAS-B10, B13 and B16 J2KVRG-E

## Hardware

Tested on:

- Seeed Studio **XIAO ESP32-C6** (recommended, matches the reference bridge PCB)
- Generic **ESP32** dev boards (WROOM-32D class), used with an external level shifter

You need a 5 V ↔ 3.3 V level shifter between the AC unit and the ESP. Any generic bidirectional level shifter module works (search Aliexpress for "level shifter"). On the reference bridge PCB this is on-board.

### Pinout

The AC unit exposes the CN22 connector on an extension cable (usually pink + blue wiring). Match to the ESP UART pins as follows:

| CN22 pin | Wire color | Signal        | XIAO ESP32-C6 | Generic ESP32 |
|:--------:|:----------:|:--------------|:-------------:|:-------------:|
| 1        | 🟦 blue     | UART RX (ESP TX) | GPIO1      | GPIO33        |
| 2        | 🟪 pink     | GND           | GND           | GND           |
| 3        | ⬛️ black    | +5 V (Vin)    | 5 V           | 5 V           |
| 4        | ⬜️ white    | UART TX (ESP RX) | GPIO0      | GPIO32        |
| 5        | 🟪 pink     | **do NOT connect** | —        | —             |

Matching connector on the ESP side: JST PA2.0 ([example](https://www.aliexpress.com/item/1005007176563512.html)).

> ⚠️ **WARNING** ⚠️
>
> Do **not** connect CN22 pin 5 (the outermost pink wire) to anything. Double-check the wiring before powering the AC unit. Always disconnect the AC unit from mains before connecting or disconnecting the ESP. Shorts or wrong wiring will damage the AC unit main board.

## Installation

1. Install Home Assistant and the [ESPHome add-on](https://esphome.io/guides/getting_started_hassio.html).

2. Add a new ESP device in the ESPHome dashboard and let it flash and generate the base configuration.

3. Edit the node YAML and add the UART + `toshiba_suzumi` climate blocks. Two starting points are included in this repo:
   - [example.yaml](example.yaml) — minimal generic ESP32 (nodemcu-32s / Arduino framework) with every optional feature listed as a commented line. Use this as a reference for the available options.
   - [example_esp32c6_xiao.yaml](example_esp32c6_xiao.yaml) — full Seeed XIAO ESP32-C6 build on ESP-IDF, including RF antenna switch setup, USB-JTAG logging, persistent Wi-Fi LED state, and a Home Assistant off-timer.

Minimal snippet:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/dennisvo/esphome_toshiba_suzumi
    components: [toshiba_suzumi]

uart:
  id: uart_bus
  tx_pin: GPIO1     # GPIO33 on generic ESP32
  rx_pin: GPIO0     # GPIO32 on generic ESP32
  parity: EVEN
  baud_rate: 9600

climate:
  - platform: toshiba_suzumi
    name: living-room
    id: living_room
    uart_id: uart_bus
    outdoor_temp:
      name: Outdoor Temp
    power_select:
      name: "Power level"

# Encrypted Home Assistant API
api:
  encryption:
    key: !secret device_encryption_key

# Encrypted OTA updates (no shared password needed)
ota:
  - platform: esphome
    encryption:

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "Toshiba-AC-Fallback"
    password: !secret ap_password
```

See the [Secrets](#secrets) section below for what to put in `secrets.yaml`.

The component compiles into the ESPHome build directly from GitHub — nothing to install manually.

Once the node is on your network, ESPHome will discover it in Home Assistant and ask for the encryption key from the node config. All entities then populate automatically.

### Secrets

Both example YAMLs enable API encryption and encrypted OTA by default and reference values from `secrets.yaml`. In the ESPHome dashboard open the three-dot menu → **Secrets editor** and make sure the following keys exist:

| Key | What to put there |
|-----|-------------------|
| `wifi_ssid` | Your Wi-Fi SSID |
| `wifi_password` | Your Wi-Fi password |
| `device_encryption_key` | 32-byte base64 key. Secures the Home Assistant API connection and, via `ota.encryption:`, the OTA session. Generate one with `esphome config-value api.encryption.key` or via the ESPHome dashboard's key generator. |
| `ap_password` | Password for the fallback captive-portal hotspot the device exposes if it cannot join Wi-Fi. |

With `ota.encryption:` enabled (as in both examples) no separate OTA password is required — the OTA session key is negotiated over the encrypted API connection. If you prefer a shared OTA password instead, replace the `encryption:` block with `password: !secret ota_password` and add `ota_password` to `secrets.yaml`.

![HomeAssistant ESPHome entity](/images/HA_entity.png)

Create a Thermostat card on the dashboard:

![HomeAssistant card](/images/HA_card.png)

### Temperature range (FrostGuard)

Home Assistant's thermostat defaults to a range of 17–30 °C. If your unit supports "8 degrees" a.k.a. FrostGuard, enable it via `supported_presets`. The range is then extended to 5–30 °C: setting a target above 17 °C switches to Standard mode, below 17 °C switches to FrostGuard.

## Filtering incorrect values (127 / 254 / 255)

The component automatically filters out sentinel readings at the code level:

- Temperature readings of `127` (which the unit transmits when it cannot measure temperature or when it is off) are ignored.
- Compressor load and current values of `254` or `255` are ignored.

You do not need to add manual filters for these values. To filter out other values, use the standard ESPHome filter syntax:

```yaml
outdoor_temp:
  name: Outdoor Temp
  filters:
    - filter_out: 127
```

### Vertical air directions

Some units support fixed vertical air directions in addition to vertical swing. ESPHome's native climate swing modes are limited to `off`, `vertical`, `horizontal`, and `both`, so fixed positions are exposed as a Home Assistant `select` entity:

```yaml
vertical_air_direction:
  name: "Vertical air direction"
```

In Toshiba service documentation this corresponds to the horizontal louver, which controls vertical air direction. Options: `Off`, `Swing`, `Top`, `Middle Top`, `Middle`, `Middle Bottom`, `Bottom`.

### Self-cleaning status

Some Toshiba units run a post-shutdown self-cleaning cycle. During this cycle the indoor fan can continue running even though the climate entity reports OFF. Enable detection with the optional `self_clean` binary sensor:

```yaml
self_clean:
  name: "Self Clean"
```

The climate entity stays OFF while self-cleaning is active — ESPHome climate has no self-cleaning mode or action, so a separate binary sensor is the correct representation.

Cycle behavior per the Toshiba service manual:

- Runs only after cooling or dry operation and only if that ran for at least 10 minutes; then runs for a fixed ~30 minutes. Does not run after heating, fan-only, or a short (<10 min) cooling/dry run.
- The indoor fan runs at low speed to dry the coil; the compressor stays off. The unit reports itself as not operating, which is why the climate entity is OFF.
- Enabling / disabling the cycle itself is a unit-side procedure (the indoor unit's `RESET` button together with a remote-control diagnosis code). This component reports the cycle but does not control it.

## Outdoor / indoor unit diagnostics (ODU / IDU sensors)

Some Toshiba units periodically send extended status messages from the outdoor (ODU) and indoor (IDU) units containing compressor load, refrigerant temperatures, and fan speed. All of these are optional — add only the ones you want.

| Sensor | Description | Unit |
|--------|-------------|------|
| `cdu_load` | Compressor load (frequency) | % |
| `cdu_iac` | Compressor current / EEV actuation | A |
| `cdu_td_temp` | CDU discharge pipe temperature | °C |
| `cdu_ts_temp` | CDU suction pipe temperature | °C |
| `cdu_te_temp` | CDU evaporator temperature | °C |
| `fcu_tc_temp` | FCU heat exchanger temperature | °C |
| `fcu_tcj_temp` | FCU heat exchanger junction temperature | °C |
| `fcu_fan_rpm` | FCU fan speed | RPM |

The unit publishes these on its own — no polling required.

### Native energy and power monitoring

Units that expose the internal energy counters can be read directly. Requires a `time` component so the driver can sync wall time with the AC unit:

```yaml
time:
  - platform: homeassistant
    id: hass_time

climate:
  - platform: toshiba_suzumi
    # ...
    time_id: hass_time
    energy:
      name: "Daily Energy"
    power:
      name: "Realtime Power"
```

`power` is a real-time estimate in Watts derived from the rate of change of the AC's internal energy counter.

### Estimating power consumption (fallback)

For older units without native energy registers, estimate power via `cdu_load` (compressor load %) and your unit's rated input power in Home Assistant:

```yaml
sensor:
  - platform: template
    sensors:
      ac_estimated_power:
        friendly_name: "AC Estimated Power"
        unit_of_measurement: "W"
        value_template: "{{ (states('sensor.compressor_load') | float(0)) / 100 * 880 }}"
```

Replace `880` with your unit's rated power input in watts (check the datasheet). Feed it into the [HA Riemann sum integral](https://www.home-assistant.io/integrations/integration/) to accumulate energy over time.

## Scan for unknown sensors

Different Toshiba units expose different feature registers. To discover what your unit supports, add a template button that runs the built-in scanner:

```yaml
button:
  - platform: template
    name: "Scan for unknown sensors"
    icon: "mdi:reload"
    on_press:
      - lambda: |-
          auto* controller = static_cast<toshiba_suzumi::ToshibaClimateUart*>(id(living_room));
          controller->scan();
```

Then watch the ESPHome logs while pressing the button:

![ESPHome log](/images/scan_log.png)

## Credits

- Upstream component: [pedobry/esphome_toshiba_suzumi](https://github.com/pedobry/esphome_toshiba_suzumi)
- Reference hardware: [b1scuitdev/Toshiba-ESP32C6-Bridge](https://github.com/b1scuitdev/Toshiba-ESP32C6-Bridge)
- Prior art / related work: [toremick/shorai-esp32](https://github.com/toremick/shorai-esp32), [Vpowgh/TConnect](https://github.com/Vpowgh/TConnect)
- Community: [ESPHome Toshiba Discord](https://discord.gg/wYYFawvqfr)

If the upstream project has been useful to you, consider supporting the original author:

<a href="https://www.buymeacoffee.com/pedobryk" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 60px !important;width: 217px !important;" ></a>
