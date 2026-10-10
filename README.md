# ONC IoT Sensors

The IoT sensor fleet of the ONC HQ homelab: Raspberry Pi Zero W gateways (BLE GATT and Classic-BT/`rfcomm`),
Arduino Uno sensor nodes with HC-05/06 Bluetooth modules, Meraki native BLE/Wi-Fi telemetry, and the MQTT topic
and automation-loop design that ties them to a Mosquitto broker and PRTG.

> This folder is a **section of the ONC HQ Lab hub** (`sections/iot-sensors/`), also published on its own
> through `git subtree`. Identifying values are `{{TOKENS}}`.

## Status (2026-10-10)

| Node | Role | Status |
|---|---|---|
| `pi-pm1630` | BLE GATT collector for the PM1630 dosimeter (software: radiation-sensor section) | ✅ Live since 2026-09-03 |
| `pi-arduino` | Classic-BT/`rfcomm` collector for Arduino-A and Arduino-B | ⏳ Architecture final, not built |
| `pi-hvac` | Classic-BT/`rfcomm` collector for Arduino-C (DHT11 + IR AC control) and Arduino-D (SCT-013 + ZMPT101B energy) | ⏳ Proposed 2026-08-29, nothing purchased |
| Meraki MR33 (Aparta) | Native BLE + Wi-Fi telemetry → `homelab/telemetry/ble/meraki` | ✅ Live since 2026-08-13/15 (hub-configured) |
| ESP32 sensors | Future | — |

## Layout

| Path | What it is |
|---|---|
| `docs/iot-sensor-fleet.md` | Fleet design: Arduino ingestion (§6.3c), Pi fleet (§6.4), build sequence (§6.8), automation loops (§6.5) |
| `collectors/` | Pi-side collector code (`rfcomm` bridge, systemd units). Empty until `pi-arduino` is built |
| `firmware/` | Arduino sketches. Empty until the nodes are flashed |
| `claude/project-instructions.md` | Instructions for the "IoT Sensors" Claude Project |
| `CHANGELOG.md` | Every change to this section |

## Interface with the hub

The hub owns the network (VLAN 30, `ONCIoT` SSID, fixed IPs), the broker (accounts, ACLs, TLS) and AD CS client
certificates. This section owns the devices, what runs on them, topic payloads and the automation loops. See the
hub's `docs/integrations/iot-sensors.md`.
