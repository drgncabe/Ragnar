# Wardrive Appliance Documentation

This directory contains the initial planning documentation for the `wardrive-appliance` branch.

The project goal is to turn this Ragnar fork into a focused passive wardriving appliance built around:

- Raspberry Pi Zero 2W host;
- Waveshare ETH/USB Hub HAT;
- SEENGREAT SCap UPS for graceful shutdown;
- two ESP32-C5 devices running HuginnESP;
- centralized USB GPS on the Pi;
- optional SDR support in a later phase.

## Documents

| Document | Purpose |
|---|---|
| [Architecture](architecture.md) | Defines the host/sensor split, data flow, normalized observation model, and first milestone. |
| [Hardware](hardware.md) | Defines the first hardware stack, USB topology, power assumptions, GPS bring-up, SCap integration, and later SDR expansion. |
| [Huginn Serial Protocol Notes](huginn-serial-protocol.md) | Defines how the Pi-side appliance should discover, command, parse, and monitor HuginnESP sensors over USB serial. |
| [Roadmap](roadmap.md) | Defines the phased implementation plan from documentation through serial ingestion, GPS, storage, UI, exports, shutdown, cleanup, and SDR. |

## Version 1 target

Version 1 is complete when the appliance can:

1. boot on a Pi Zero 2W;
2. detect two HuginnESP ESP32-C5 sensors;
3. assign one C5 to Wi-Fi scanning and one C5 to BLE scanning;
4. read observations continuously over USB serial;
5. correlate observations with Pi-side GPS;
6. store a durable session locally;
7. show a basic live status page;
8. export Wi-Fi observations to WiGLE CSV;
9. shut down cleanly when the SCap UPS indicates external power loss.

## Development stance

The branch should avoid removing major Ragnar features until the new appliance path works end-to-end. The first priority is a reliable sensor pipeline, not UI polish or SDR support.

This remains an unofficial derivative of Ragnar. Existing license, attribution, and provenance requirements must be preserved.
