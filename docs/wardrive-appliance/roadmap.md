# Wardrive Appliance Roadmap

This roadmap keeps the first implementation focused. The project should prove the HuginnESP sensor pipeline before trimming large parts of Ragnar or adding SDR.

## Guiding principles

1. Build the data pipeline first.
2. Do not destructively remove unrelated Ragnar features until the appliance path works.
3. Keep Wi-Fi/BLE discovery passive.
4. Keep GPS centralized on the Pi.
5. Preserve raw sensor data alongside normalized observations.
6. Prefer small testable milestones over large rewrites.
7. Treat SDR as a plugin, not a core dependency.

## Phase 0 — Documentation and planning

Status: in progress.

Deliverables:

- architecture document;
- hardware document;
- Huginn serial protocol notes;
- roadmap;
- initial GitHub issues for Ragnar and HuginnESP.

Exit criteria:

- `wardrive-appliance` branch has enough documentation to guide implementation;
- first implementation tickets are clear;
- scope is limited to Pi + two C5s + GPS.

## Phase 1 — Huginn serial ingestion

Goal: read two HuginnESP sensors reliably from the Pi.

Deliverables:

- `wardrive_appliance` Python package skeleton;
- `HuginnSerialProvider`;
- serial device discovery;
- JSON line parser;
- basic observation normalization;
- malformed line handling;
- simple diagnostic CLI.

Example target command:

```bash
python -m wardrive_appliance.cli --serial /dev/ttyACM0 --serial /dev/ttyACM1
```

Example target output:

```text
Huginn sensor detected on /dev/ttyACM0
Huginn sensor detected on /dev/ttyACM1
Wi-Fi observations: 123
BLE observations: 48
Malformed lines: 0
```

Exit criteria:

- two C5s stream concurrently;
- process runs for at least 30 minutes;
- malformed/non-JSON lines do not crash ingestion;
- disconnect of one C5 does not stop the whole process.

## Phase 2 — Huginn dedicated sensor modes

Goal: use two C5s efficiently by assigning one to Wi-Fi and one to BLE.

HuginnESP work:

- add Wi-Fi-only continuous mode;
- add BLE-only continuous mode;
- add stable sensor identifier to announce/status output;
- keep existing wardrive mode for single-device operation;
- document commands and behavior.

Ragnar appliance work:

- role assignment config;
- send mode command after sensor identification;
- persist role/sensor mapping;
- show role status in CLI.

Desired host commands:

```text
mode wifi-continuous
mode ble-continuous
mode wardrive
status
stop
```

Exit criteria:

- C5 #1 stays in Wi-Fi mode;
- C5 #2 stays in BLE mode;
- roles survive reboot when sensor IDs are stable;
- positional `/dev/ttyACM*` assignment is no longer required.

## Phase 3 — GPS provider and correlation

Goal: attach host-side GPS fixes to observations.

Deliverables:

- USB GPS provider;
- NMEA parsing;
- latest-fix cache;
- no-fix handling;
- observation correlation;
- optional pending/backfill marker for observations captured before GPS fix.

Exit criteria:

- observations include GPS position when a fix exists;
- observations are preserved when GPS has no fix;
- GPS status appears in diagnostic CLI;
- stale GPS fixes are detected and not silently treated as current.

## Phase 4 — Session storage

Goal: persist observations to local storage in a durable session model.

Deliverables:

- session lifecycle;
- local SQLite database or equivalent;
- normalized observation tables;
- raw payload preservation;
- unique network/device tracking;
- basic session summary.

Suggested tables:

- `sessions`;
- `sensors`;
- `gps_fixes`;
- `observations`;
- `wifi_networks` or derived unique view;
- `ble_devices` or derived unique view;
- `events`.

Exit criteria:

- a drive can be stopped and resumed without corrupting storage;
- session summary can be generated from stored data;
- raw Huginn records are preserved;
- duplicate observations are handled without losing signal/time/location history.

## Phase 5 — Basic status UI

Goal: expose a minimal appliance status page.

Initial UI should show:

- session active/inactive;
- runtime;
- GPS fix state;
- Wi-Fi sensor state;
- BLE sensor state;
- observation counts;
- unique Wi-Fi/BLE counts;
- database status;
- free disk space;
- power state when available.

Exit criteria:

- page loads on Pi Zero 2W without excessive CPU use;
- status updates without restarting capture;
- UI remains readable on phone and small local displays.

## Phase 6 — Exporters

Goal: generate useful files from each session.

Initial exporters:

- WiGLE CSV for Wi-Fi observations;
- KML or GeoJSON for map review;
- CSV summary for BLE devices;
- session JSON metadata.

Exit criteria:

- export can run after a session;
- export failure does not corrupt session storage;
- exported Wi-Fi data can be validated against a known external tool or viewer.

## Phase 7 — Auto-start and SCap shutdown

Goal: make the appliance practical in a vehicle.

Deliverables:

- systemd service for appliance startup;
- config file for sensor roles and GPS;
- SCap GPIO monitor;
- graceful stop sequence;
- database flush;
- filesystem sync;
- clean shutdown.

Power-loss sequence:

```text
external power loss
        |
        v
mark session closing
        |
        v
stop accepting optional new work
        |
        v
flush database
        |
        v
sync filesystem
        |
        v
shutdown -h now
```

Exit criteria:

- appliance starts on boot;
- capture begins without manual login;
- external power loss triggers clean shutdown;
- no database corruption after repeated power-loss tests.

## Phase 8 — Ragnar feature trimming

Goal: reduce the fork to a focused wardrive appliance after the new path works.

Candidate removals or hidden features:

- vulnerability scanning;
- offensive/security-testing modules not needed for passive wardriving;
- heavy AI features;
- unrelated web UI sections;
- unneeded dependencies.

This should happen after the new appliance path is functional, not before.

Exit criteria:

- branch remains bootable and testable after each cleanup;
- license/provenance requirements are preserved;
- documentation clearly describes the derivative nature of the project.

## Phase 9 — SDR provider

Goal: add optional RF observations without making SDR mandatory.

Deliverables:

- `SdrProvider` interface;
- derived RF observation schema;
- short periodic sweep support;
- GPS correlation;
- storage integration;
- optional status page section.

Initial observation fields:

- timestamp;
- latitude;
- longitude;
- frequency start;
- frequency end;
- peak frequency;
- peak power;
- average power;
- bandwidth estimate;
- source device.

Exit criteria:

- appliance runs normally without SDR;
- SDR can be enabled by config;
- RF observations do not overwhelm Pi Zero 2W storage or CPU;
- raw IQ recording is not enabled by default.

## Phase 10 — Field hardening

Goal: make repeated drives reliable.

Deliverables:

- reconnect handling;
- watchdog behavior;
- log rotation;
- disk space guardrails;
- session recovery after improper shutdown;
- configuration backup/export;
- simple health check command.

Exit criteria:

- repeated vehicle power cycles do not require manual intervention;
- failed sensors are visible in status output;
- low disk space does not crash the appliance;
- logs do not fill the SD card.

## First GitHub issues to create

Recommended Ragnar issues:

1. Add wardrive appliance Python package skeleton.
2. Add Huginn serial provider.
3. Normalize Wi-Fi and BLE observations.
4. Add diagnostic CLI for two Huginn sensors.
5. Add GPS provider and latest-fix correlation.
6. Add SQLite session storage.
7. Add basic status web page.
8. Add WiGLE CSV exporter.
9. Add SCap UPS shutdown service.
10. Add SDR provider placeholder.

Recommended HuginnESP issues:

1. Add Wi-Fi-only and BLE-only continuous scan modes.
2. Add stable `sensor_id` to announce and status output.
3. Document host role-assignment expectations.
4. Add or confirm ESP32-C5-Zero build target details.

## Version 1 definition of done

Version 1 is complete when the appliance can:

- boot on a Pi Zero 2W;
- detect two HuginnESP C5 sensors;
- assign one to Wi-Fi and one to BLE;
- read observations continuously;
- correlate observations with Pi GPS;
- store a complete session;
- show a basic live status page;
- export Wi-Fi data to WiGLE CSV;
- shut down cleanly when SCap indicates power loss.
