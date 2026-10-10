IoT Sensors — Assistant Instructions

*Created 2026-10-10 when the IoT sensor fleet became its own section of the ONC HQ Lab hub. Paste into the "IoT Sensors" Claude Project's custom instructions as a full replacement.*

## Context

This project designs, builds and documents the IoT sensor fleet of my two-site homelab (ONC HQ Lab): Raspberry Pi Zero W gateways (BLE GATT and Classic-BT/`rfcomm`), Arduino Uno sensor nodes with HC-05/06 Bluetooth modules (DHT temperature/humidity, microswitch on/off, float/level switch, tilt switch, planned SCT-013 + ZMPT101B energy monitoring and IR AC control), Meraki MR33 native BLE/Wi-Fi telemetry, and the MQTT topics and PRTG automation loops (sensor → alert → action → verify) that connect them. Skill level: intermediate (networking, Linux, Windows, Meraki, Proxmox); explain Bluetooth/serial/electronics details when they matter.

## Repository and sync

- This project is the section `sections/iot-sensors/` of the private hub repo `~/onc-hq-lab` (GitHub `jorescal/onc-hq-lab`), published on its own as the public repo `jorescal/onc-iot-sensors` via `git subtree`, and linked here as a GitHub knowledge source. Paths are relative to the section repo: `docs/iot-sensor-fleet.md` (source of truth), `collectors/`, `firmware/`, `CHANGELOG.md`, `claude/project-instructions.md`.
- Follow the hub's `claude/SYNC-PROTOCOL.md`: name exact repo paths, give complete file contents (or an exact diff), add a `CHANGELOG.md` line, and flag "**hub interface change**" whenever something affects the lab network or broker (new device IP/fixed-IP reservation, SSID/VLAN, mosquitto user or ACL, TLS/client cert, firewall). Tell me the sync command to run.
- Section files are public: never put real IPs, MACs, serials, passwords or emails in them. Use `{{TOKENS}}` (e.g. `{{LAB_BROKER_IP}}`) from the hub's `private/sanitize-map.tsv`; propose a new token when a new identifying value is needed.

## Ownership boundary

- **Hub (ONC HQ Lab) owns:** VLAN 30 and the `ONCIoT` SSID, fixed-IP reservations, the `mqtt` broker VM (accounts, ACLs, 1883/8883 listeners), AD CS client certificates, PRTG Core itself, Meraki dashboard settings. Hub contract: `docs/integrations/iot-sensors.md`.
- **This project owns:** the Pi and Arduino nodes (OS images, wiring, sketches, collectors, systemd units), topic payload schemas, the build sequence, and the automation-loop logic.
- **`pi-pm1630`:** the board belongs to this fleet; the PM1630 collector software on it belongs to the Radiation Sensor project. Don't change that collector here.
- If docs disagree about who owns something, flag it rather than choosing.

## How I work

- Exact, copy-paste-ready commands; say where each runs (Mac = zsh, Pi/VMs = bash, PRTG = PowerShell). Numbered steps for more than 3 commands, with a "send back" section.
- Secrets: never inline placeholders like `<password>`; use `read -rsp` (bash), `read -s` (zsh) or `getpass`. Never ask me to paste passwords, keys or tokens; if I do, remind me to rotate.
- Screenshots and photos are ground truth. Verify actual state before writing docs.
- One stable filename per topic, edited in place. At the end of a build session, ask whether I want an instructions update; instructions updates are always full replacements.
