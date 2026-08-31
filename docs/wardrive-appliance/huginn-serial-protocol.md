# Huginn Serial Protocol Notes

This document describes how the wardrive appliance should consume HuginnESP sensors over USB serial.

It is not intended to replace the HuginnESP documentation. It defines the host-side expectations for this Ragnar wardrive appliance branch.

## Transport

HuginnESP devices communicate over USB serial.

Initial host defaults:

| Setting | Value |
|---|---|
| Baud | 460800 |
| Data bits | 8 |
| Parity | none |
| Stop bits | 1 |
| Record format | newline-delimited text |

The stream may contain both JSON records and plain text log/alert lines. The host parser must handle both safely.

## Host discovery flow

The appliance should not rely on fixed Linux device paths.

Recommended discovery process:

```text
1. Enumerate candidate serial devices.
2. Open each candidate with the expected serial settings.
3. Read briefly for a boot announce.
4. If no announce appears, send `status`.
5. Accept the device only if it returns the expected HuginnESP JSON shape.
6. Record port, board, firmware, capabilities, and sensor_id if available.
7. Assign role from appliance configuration.
8. Send the requested mode command.
```

Candidate device paths may include:

```text
/dev/ttyACM*
/dev/ttyUSB*
/dev/serial/by-id/*
```

Where possible, `/dev/serial/by-id/*` should be preferred for diagnostics because it is more stable than `/dev/ttyACM*`.

## Current announce expectation

Current HuginnESP behavior includes a boot announce resembling:

```json
{"device":"HuginnESP","fw":"1.0","board":"esp32-c5","caps":["wifi","ble"]}
```

The parser should treat `device == "HuginnESP"` as the primary signature.

## Desired future announce shape

For two-C5 operation, a stable sensor identifier is strongly preferred.

Desired shape:

```json
{
  "device": "HuginnESP",
  "fw": "1.1",
  "board": "esp32-c5",
  "sensor_id": "huginn-c5-a1b2c3",
  "caps": ["wifi", "ble"]
}
```

The `sensor_id` should be stable across boots and should not depend on Linux port order.

Possible sources:

- ESP32 chip MAC;
- eFuse-derived identifier;
- persisted generated identifier;
- explicit configured name.

## Input line classes

The host should classify each line as one of:

| Class | Description | Handling |
|---|---|---|
| JSON observation | Wi-Fi/BLE/GPS/status data | Parse and normalize. |
| JSON status | Response to `status`, `gps`, or future commands | Parse and update sensor state. |
| Plain text log | Startup/progress messages | Store in debug log or ignore. |
| Plain text alert | High-signal BLE alert blocks | Preserve as optional raw alert events. |
| Malformed JSON | Partial/corrupt line | Count and continue. |

The parser must never crash the appliance because one serial line is malformed.

## Current Wi-Fi observation shape

Expected Wi-Fi JSON line:

```json
{
  "type": "WIFI",
  "mac": "AA:BB:CC:DD:EE:FF",
  "ssid": "ExampleSSID",
  "rssi": -62,
  "channel": 6,
  "auth": "WPA2"
}
```

Host normalization target:

| Source field | Normalized field |
|---|---|
| `type` | `type = wifi` |
| `mac` | `address` |
| `ssid` | `name` |
| `rssi` | `rssi` |
| `channel` | `channel` |
| `auth` | `security` |

## Current BLE observation shape

Expected BLE JSON line:

```json
{
  "type": "BLE",
  "mac": "11:22:33:44:55:66",
  "name": "ExampleBLEDevice",
  "rssi": -71
}
```

Host normalization target:

| Source field | Normalized field |
|---|---|
| `type` | `type = ble` |
| `mac` | `address` |
| `name` | `name` |
| `rssi` | `rssi` |

## GPS fields from HuginnESP

HuginnESP can optionally append GPS fields when built with GPS support and when a fix is available.

Possible fields:

```json
{
  "lat": 32.000000,
  "lon": -81.000000,
  "speed_kph": 44.6,
  "speed_mps": 12.4
}
```

For this appliance, GPS should be centralized on the Pi first. If Huginn-provided GPS fields appear, they should be preserved in the raw payload and may be used later for comparison or fallback.

## Desired role commands

Two-C5 operation needs dedicated modes.

Desired host commands:

```text
mode wifi-continuous
mode ble-continuous
mode wardrive
stop
status
```

Alternative command names are acceptable if HuginnESP standardizes on different wording. The host code should isolate command strings in one place.

## Role assignment

Initial role assignment should be configuration-driven.

Example configuration:

```yaml
huginn_sensors:
  - role: wifi
    sensor_id: huginn-c5-a1b2c3
  - role: ble
    sensor_id: huginn-c5-d4e5f6
```

During early development, fallback assignment may be positional:

```yaml
huginn_sensors:
  - role: wifi
    port: /dev/ttyACM0
  - role: ble
    port: /dev/ttyACM1
```

Port-based assignment should be considered temporary and less reliable.

## Parser behavior

The parser should:

- trim line endings;
- ignore empty lines;
- parse JSON lines when possible;
- keep raw line text for debugging;
- distinguish observation JSON from status JSON;
- preserve unknown JSON fields;
- count malformed lines;
- rate-limit noisy log output;
- reconnect if a serial device disappears and returns.

## Reconnection behavior

If a sensor disconnects:

```text
mark sensor offline
stop reading from dead port
continue session if other sensors are healthy
rescan serial ports periodically
re-identify HuginnESP devices
restore configured role
send mode command again
resume ingestion
```

A disconnected BLE sensor should not stop Wi-Fi capture, and a disconnected Wi-Fi sensor should not stop BLE capture.

## Timing

Each received observation should receive a host-side `received_at` timestamp immediately after a complete line is read.

If the source provides its own observation timestamp later, the normalized model should preserve both:

- `observed_at`, from sensor if available;
- `received_at`, from Pi host.

For version 1, `observed_at` may equal `received_at`.

## Error handling

The host should track at least:

- total lines received;
- JSON observations parsed;
- non-JSON lines received;
- malformed JSON count;
- last received timestamp;
- last valid observation timestamp;
- reconnect count;
- current assigned role;
- current mode command sent.

These fields should feed the diagnostic CLI and later the web status page.

## Security and safety boundary

The wardrive appliance should treat HuginnESP as a passive discovery sensor. Host-side code should not add deauthentication, association, exploitation, or active attack workflows to the Huginn provider.
