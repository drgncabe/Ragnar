# Wardrive Appliance Hardware

This document defines the first target hardware stack for the wardrive appliance.

The goal is to keep version 1 simple: Pi Zero 2W as the host, two ESP32-C5 boards as radio sensors, one USB GPS receiver, and clean shutdown support through the SEENGREAT SCap UPS board.

## Version 1 target hardware

| Component | Role | Notes |
|---|---|---|
| Raspberry Pi Zero 2W | Host/logger/UI | Runs the appliance software, storage, GPS correlation, web UI, and exports. |
| Waveshare ETH/USB Hub HAT | USB expansion and maintenance Ethernet | Provides ports for both C5s and GPS; Ethernet is useful for retrieving data without touching Wi-Fi. |
| SEENGREAT SCap UPS Board | Power hold-up and shutdown trigger | Provides enough reserve time to sync storage and halt after external power loss. |
| ESP32-C5-Zero #1 | Wi-Fi sensor | Runs HuginnESP; assigned to Wi-Fi-only discovery mode. |
| ESP32-C5-Zero #2 | BLE sensor | Runs HuginnESP; assigned to BLE-only discovery mode. |
| USB GNSS receiver | GPS/time source | Prefer u-blox-based receivers or other reliable NMEA-compatible units. |
| High-endurance microSD | Local storage | 32 GB minimum; 64 GB or 128 GB preferred. |
| Pi onboard Wi-Fi | Management AP | Used for configuration/status access, not primary scanning. |
| Optional SDR | Future RF sensor | Out of scope for initial bring-up. |

## Physical topology

```text
Vehicle power / USB-C source
          |
          v
SEENGREAT SCap UPS
          |
          v
Raspberry Pi Zero 2W
          |
          v
Waveshare ETH/USB Hub HAT
      |          |          |
      |          |          +-- USB GNSS
      |          +------------- ESP32-C5 #2, BLE role
      +------------------------ ESP32-C5 #1, Wi-Fi role
```

The Pi's onboard Wi-Fi should remain available for a local management AP or normal network connectivity. The Ethernet port on the Waveshare HAT should be treated as the preferred maintenance interface when available.

## Power assumptions

The first version should be designed around modest current draw:

- Pi Zero 2W;
- Waveshare USB/Ethernet HAT;
- two ESP32-C5 boards;
- one USB GPS receiver.

The first version should not assume that a HackRF or other higher-power SDR can be added without revisiting power budget and thermal behavior.

Before field use, validate:

- boot reliability with all USB devices connected;
- no undervoltage warnings on the Pi;
- C5 boards remain attached under load;
- GPS remains attached under load;
- storage sync completes before SCap reserve is exhausted;
- SCap power-loss GPIO signal is stable and debounced.

## USB device identity

Linux device names such as `/dev/ttyACM0` and `/dev/ttyACM1` are not stable enough to encode sensor role.

The appliance should identify HuginnESP boards by serial announce/status output and then assign roles in software.

Preferred flow:

```text
Enumerate candidate serial ports
        |
        v
Open each port
        |
        v
Wait for HuginnESP announce or send status probe
        |
        v
Read board / firmware / sensor_id / caps
        |
        v
Assign role from config
        |
        v
Send mode command
```

## Suggested first bench test

Initial test stack:

```text
Pi Zero 2W
Waveshare ETH/USB Hub HAT
C5 #1
C5 #2
```

No GPS, SDR, or SCap automation yet.

Test goals:

- both C5 boards enumerate;
- both serial ports can be opened;
- both stream HuginnESP data;
- unplug/replug recovery is understood;
- continuous read test runs for at least 30 minutes.

Example diagnostic command target:

```bash
python -m wardrive_appliance.cli --serial /dev/ttyACM0 --serial /dev/ttyACM1
```

Expected diagnostic output:

```text
Huginn sensor detected on /dev/ttyACM0
Huginn sensor detected on /dev/ttyACM1
Wi-Fi observations: 123
BLE observations: 48
Malformed lines: 0
Last Wi-Fi observation: 42 ms ago
Last BLE observation: 87 ms ago
```

## GPS bring-up

After serial ingestion is stable, add the GPS receiver.

Validation checklist:

- GPS appears as a stable serial device;
- NMEA sentences are readable;
- first fix time is acceptable;
- position accuracy is captured;
- speed is captured if available;
- latest GPS fix can be associated with C5 observations;
- no-fix state is handled cleanly.

The appliance should preserve observations even when GPS has no fix, but mark them as ungeoreferenced or pending GPS backfill.

## SCap UPS integration

SCap integration should be added after the data pipeline is stable.

Expected behavior:

```text
External power removed
        |
        v
SCap GPIO changes state
        |
        v
wardrive appliance receives power-loss event
        |
        v
stop accepting new session data
        |
        v
flush database
        |
        v
generate or mark exports pending
        |
        v
sync filesystem
        |
        v
shutdown -h now
```

The shutdown handler should be conservative. A clean halt matters more than finishing every export immediately.

## SDR expansion

SDR should be treated as version 2 or later.

Recommended first SDR step:

- add an `SdrProvider` interface;
- store derived RF observations, not raw IQ;
- use short periodic sweeps;
- correlate results with the centralized GPS fix;
- keep SDR optional and disabled by default.

Possible SDR classes:

| SDR | Use | Notes |
|---|---|---|
| RTL-SDR | sub-GHz / ADS-B / ISM experimentation | Low cost, not suitable for 2.4/5 GHz Wi-Fi spectrum. |
| HackRF | broader spectrum sweeps | Higher power and CPU/storage impact. Validate power before vehicle use. |

## Enclosure considerations

Version 1 should prioritize serviceability over polish.

Recommended enclosure features:

- access to USB ports;
- ventilation around Pi and USB hub;
- strain relief for vehicle power;
- visible LEDs, or optional light pipes;
- GPS antenna placement away from noisy electronics;
- easy microSD access during development;
- label C5 boards physically as Wi-Fi and BLE, even though software should not rely on port order.
