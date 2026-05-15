# ESPHome LIN Bus Component

A LIN (Local Interconnect Network) bus component for ESPHome, for communicating with the many cheap LIN actuators, sensors, fans, flaps, motors, pumps and solenoids found in vehicles.

**Platform:** ESP32 family only (ESP-IDF framework). Validated on ESP32-C3.

> [!IMPORTANT]
> This is a fork of [swifty99/linbus](https://github.com/swifty99/linbus) to fix a bug and enable some features that I needed for my project. The plan is to send these changes upstream at some point.

## Installation

Add this to your ESPHome YAML:

```yaml
external_components:
  - source: github://swifty99/linbus
    components: [linbus]
```

## Quickstart

See [`example.yaml`](example.yaml) for a complete, runnable configuration. Minimum master-mode setup:

```yaml
esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: esp-idf

linbus:
  id: lin_master
  tx_pin: GPIO10
  rx_pin: GPIO6
  cs_pin: GPIO7
  baud_rate: 19200
  uart_port: UART_NUM_1
  mode: master
  schedule:
    - lin_id: 33
      interval: 200ms
  on_frame:
    - lin_id: 33
      then:
        - lambda: |-
            ESP_LOGD("linbus", "Got %d bytes for id 33", data_length);
```

## Configuration Reference

```yaml
linbus:
  id: lin_master                    # Optional
  tx_pin: GPIO10                    # Required
  rx_pin: GPIO6                     # Required
  cs_pin: GPIO7                     # Optional — LIN transceiver chip-select
  baud_rate: 19200                  # Optional (default 19200). Common: 9600, 19200, 20000
  uart_port: UART_NUM_1             # Optional (default UART_NUM_1)
  mode: master                      # Optional — master | slave | listener (default master)
  schedule:                         # Optional — master-only scheduling table (max 16)
    - lin_id: 34
      interval: 100ms
  responses:                        # Optional — pre-buffered publisher data (max 16)
    - lin_id: 33
      data: [0x00, 0x00, 0x00, 0x00]
      checksum: enhanced            # enhanced (LIN 2.x, default) | classic (LIN 1.x)
  on_frame:                         # Optional — automations per LIN ID
    - lin_id: 33
      then:
        - lambda: ...
```

Frame IDs are 6-bit decimal values (0–63). IDs 60–63 are reserved (diagnostic / future use) and not currently supported.

## Modes

- **master** — drives the bus. Sends frame headers on the configured `schedule`. Also publishes data for any `lin_id` in `responses`.
- **slave** — listens passively for headers. When a header matches one of this node's `responses`, the node transmits the buffered data.
- **listener** — receives only, never transmits. Useful for sniffing an existing bus or bench-testing.

## Schedule & Responses — the one rule that matters

LIN has **no collision detection**. If two nodes respond to the same `lin_id`, the bus corrupts and stops working. You must design your bus so that **each `lin_id` has exactly one publisher**. The master's schedule simply sends headers — whoever is configured to respond to that ID puts the bytes on the wire.

Lay this out explicitly before you configure anything:

```
ID  Publisher  Purpose            Update Rate
--  ---------  -----------------  -----------
10  Slave A    Temperature        200ms
11  Master     Pump Command       100ms
12  Slave B    Status             100ms
```

Communication directions all use the same mechanism:
- **Slave → master:** master schedules header for a slave's ID; slave publishes; master receives via `on_frame`.
- **Master → slave:** master schedules header for an ID the master itself publishes (listed in its own `responses`); slaves receive via `on_frame`.
- **Slave → slave:** master schedules the header; publisher slave transmits; listener slaves receive.

## on_frame Trigger

Fires when a complete, valid frame with the given `lin_id` is received. Exposed variables:

- `data` (`std::vector<uint8_t>`) — up to 8 bytes received.
- `data_length` (`uint8_t`) — actual length.

```yaml
on_frame:
  - lin_id: 33
    then:
      - lambda: |-
          if (data.size() >= 2) {
            float temp = ((data[0] << 8) | data[1]) / 10.0f;
            ESP_LOGD("linbus", "temp=%.1f", temp);
          }
```

### High-frequency frames: use template sensors

At 10–50 ms intervals, `on_frame` runs thousands of times a minute. Heavy logic inside it will overwhelm ESPHome. Keep the automation trivial (copy into a global) and read at a controlled rate from a template sensor:

```yaml
globals:
  - id: lin_frame_33
    type: std::vector<uint8_t>
    restore_value: no
    initial_value: '{}'

linbus:
  # ...
  on_frame:
    - lin_id: 33
      then:
        - lambda: id(lin_frame_33) = data;

sensor:
  - platform: template
    name: "LIN Temperature"
    unit_of_measurement: "°C"
    update_interval: 1s
    lambda: |-
      auto d = id(lin_frame_33);
      if (d.size() >= 2) return ((d[0] << 8) | d[1]) / 10.0f;
      return {};
```

## Action: `linbus.update_response`

Updates the pre-buffered data this node publishes for a given `lin_id`. The new bytes are sent the next time that header appears on the bus.

```yaml
number:
  - platform: template
    name: "Pump Speed"
    min_value: 0
    max_value: 100
    step: 1
    optimistic: true
    on_value:
      then:
        - linbus.update_response:
            id: lin_master
            lin_id: 48
            data: !lambda |-
              uint8_t speed = (uint8_t) x;
              return {speed, 0x01, 0x00, 0x00};
```

Parameters:
- `id` — the `linbus` component ID (optional when only one is configured).
- `lin_id` — required, 0–63, templatable.
- `data` — required, 1–8 bytes, static list or lambda returning `std::vector<uint8_t>`.

## Diagnostic Sensors

Expose bus-health counters as standard ESPHome sensors:

```yaml
sensor:
  - platform: linbus
    linbus_id: lin_master
    master_requests:
      name: "LIN Master Requests"
    id_requests_answered:
      name: "LIN Requests Answered"
    data_bytes_received:
      name: "LIN Data Bytes"
    checksum_errors:
      name: "LIN Checksum Errors"
    send_failures:
      name: "LIN Send Failures"
    bus_load:
      name: "LIN Bus Load"        # percent, MEASUREMENT
    error_rate:
      name: "LIN Error Rate"      # percent, MEASUREMENT
```

All fields are optional — add only the ones you need.

## Current Limitations

- No event-triggered (spontaneous) frames.
- No diagnostic frames (IDs 60–63) or NAD/SID services.
- No sleep / wake-up signalling.
- No node configuration services (LIN 2.x config).
- ESP32 / ESP-IDF only. RP2040, ESP32 Arduino, and ESP8266 from the v1 codebase are **not** supported.

## Migrating from v1

If you used the old `lin:` component from this repo, nothing carries over:

| v1 | v2 |
|---|---|
| `lin:` YAML key | `linbus:` |
| `uart_id` + separate `uart:` block | `tx_pin` / `rx_pin` / `uart_port` / `baud_rate` on `linbus:` directly |
| `linbus.send` action | `linbus.update_response` |
| `linbus.request_pid` action | removed — use `schedule:` instead |
| Custom forked `uart:` component | removed — uses ESP-IDF UART driver directly |
| ESP32 Arduino / RP2040 / ESP8266 | ESP-IDF on ESP32 only |

## Credits & References

- Original slave code inspiration: [Fabian-Schmidt/esphome-truma_inetbox](https://github.com/Fabian-Schmidt/esphome-truma_inetbox)
- LIN 2.1 specification: https://lin-cia.org/fileadmin/microsites/lin-cia.org/resources/documents/LIN-Spec_Pac2_1.pdf
- Upstream source branch: [swifty99/esphome @ lin_bus](https://github.com/swifty99/esphome/tree/lin_bus)
- API shape mirrors ESPHome's [CAN bus component](https://esphome.io/components/canbus.html).

## License

GPLv3 — see [`LICENSE`](LICENSE).
