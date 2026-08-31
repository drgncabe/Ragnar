# Wardrive Appliance Architecture

This document defines the initial architecture for the `wardrive-appliance` branch of this unofficial Ragnar derivative.

The goal is to build a narrow, vehicle-powered wardriving appliance that keeps Ragnar's useful GPS, session, mapping, and export concepts while moving radio discovery to external ESP32-C5 sensors running HuginnESP firmware.

## Design intent

The appliance should be:

- small enough to run on a Raspberry Pi Zero 2W;
- reliable enough to start automatically when vehicle power is applied;
- safe enough to shut down cleanly when vehicle power is removed;
- modular enough to support Wi-Fi, BLE, GPS, and later SDR sensors;
- intentionally narrower than full Ragnar.

This branch should avoid a large destructive cleanup until the sensor pipeline is proven.

## High-level topology

```text
Vehicle USB power
       |
       v
SEENGREAT SCap UPS
       |
       v
Raspberry Pi Zero 2W
       |
       +-- USB GNSS receiver
       +-- ESP32-C5 #1 running HuginnESP as Wi-Fi sensor
       +-- ESP32-C5 #2 running HuginnESP as BLE sensor
       +-- optional SDR receiver, later
       +-- onboard Pi Wi-Fi for management AP / web UI
       +-- Ethernet through Waveshare ETH/USB Hub HAT for maintenance
```

## Responsibilities

### Raspberry Pi Zero 2W

The Pi is the host and data appliance. It should handle:

- sensor discovery;
- HuginnESP serial ingestion;
- centralized GPS correlation;
- session lifecycle;
- SQLite or similar local storage;
- live diagnostic status;
- export generation;
- web configuration/status UI;
- SCap UPS power-loss handling;
- optional SDR processing later.

The Pi should not be responsible for active Wi-Fi channel scanning in the first version.

### ESP32-C5 #1

Dedicated Wi-Fi discovery sensor.

Initial target behavior:

- boot into HuginnESP;
- announce over USB serial;
- accept a command from the Pi to enter Wi-Fi-only capture mode;
- emit newline-delimited JSON Wi-Fi observations;
- avoid BLE scanning while assigned to Wi-Fi duty.

### ESP32-C5 #2

Dedicated BLE discovery sensor.

Initial target behavior:

- boot into HuginnESP;
- announce over USB serial;
- accept a command from the Pi to enter BLE-only capture mode;
- emit newline-delimited JSON BLE observations and BLE alert events;
- avoid Wi-Fi scanning while assigned to BLE duty.

### GPS receiver

GPS should be centralized on the Pi rather than attached separately to either C5.

This lets one location/time stream be correlated with:

- Wi-Fi observations;
- BLE observations;
- future SDR observations;
- track logging;
- speed and distance calculations;
- session metadata.

### SDR receiver

SDR is intentionally out of scope for the first working milestone.

When added, the SDR provider should produce derived RF observations rather than raw IQ recordings by default. Examples:

- timestamp;
- GPS position;
- frequency range;
- peak frequency;
- peak power;
- average power;
- bandwidth estimate;
- optional classification.

## Data flow

```text
HuginnESP Wi-Fi JSON
          |
          v
Huginn serial provider
          |
          v
normalized observation
          |
          +-- latest GPS fix
          +-- monotonic receipt timestamp
          +-- sensor identity
          v
session storage
          |
          +-- live status
          +-- exports
          +-- maps
```

The same pipeline should handle BLE and future SDR observations.

## Normalized observation model

The first normalized schema should be deliberately small and extensible.

```json
{
  "observed_at": "2026-08-31T17:30:00.000Z",
  "received_at": "2026-08-31T17:30:00.050Z",
  "session_id": "2026-08-31_133000",
  "sensor_id": "huginn-c5-a1b2c3",
  "sensor_role": "wifi",
  "type": "wifi",
  "address": "AA:BB:CC:DD:EE:FF",
  "name": "ExampleSSID",
  "rssi": -62,
  "channel": 6,
  "security": "WPA2",
  "position": {
    "lat": 32.000000,
    "lon": -81.000000,
    "accuracy_m": 3.5,
    "speed_mps": 12.4
  },
  "raw": {
    "type": "WIFI",
    "mac": "AA:BB:CC:DD:EE:FF",
    "ssid": "ExampleSSID",
    "rssi": -62,
    "channel": 6,
    "auth": "WPA2"
  }
}
```

Fields should be nullable where the sensor does not provide them. The raw source record should be preserved so later parsers can be improved without losing original data.

## Sensor abstraction

The wardrive appliance code should treat each data source as a provider.

```text
SensorProvider
  start()
  stop()
  observations()
  status()
```

Initial providers:

- `HuginnSerialProvider`
- `GpsProvider`

Later providers:

- `SdrProvider`
- `LinuxMonitorModeProvider`, optional
- `RemoteSensorProvider`, optional

## First milestone

The first milestone is not a polished UI. It is a durable sensor pipeline.

A successful first milestone means:

- two C5 boards are visible to the Pi over USB;
- both are identified as HuginnESP devices;
- one can be assigned to Wi-Fi duty;
- one can be assigned to BLE duty;
- newline-delimited JSON can be parsed continuously;
- basic counts are displayed;
- malformed lines do not crash the process;
- the process can run for at least 30 minutes without serial disconnects or memory growth.

## Non-goals for the first milestone

- full Ragnar feature parity;
- vulnerability scanning;
- offensive tooling;
- SDR capture;
- polished map UI;
- WiGLE upload automation;
- automatic shutdown;
- long-term database optimization.

## Naming and provenance

This branch remains an unofficial derivative of Ragnar. Documentation and UI should avoid presenting this project as the official Ragnar project. Existing license and provenance requirements must be preserved.
