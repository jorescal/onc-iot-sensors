# IoT Sensor Fleet: Raspberry Pi gateways, Arduino nodes, automation loops

> Section of the ONC HQ Lab hub (`sections/iot-sensors/`), published as `jorescal/onc-iot-sensors`.
> **Moved 2026-10-10** verbatim from `homelab_build_plan_v5.md` Phase 6 (§6.3c, §6.4, §6.8, §6.5). Section numbers are
> kept so existing cross-references still resolve. Network placement (VLAN 30, `ONCIoT` SSID, fixed IPs), broker
> accounts/ACLs and certificates stay in the hub; see the hub's `docs/integrations/iot-sensors.md`.
> Identifying values are `{{TOKENS}}`.

## **6.3c Bluetooth Arduino Sensor Fleet — Ingestion Path & PRTG Historical Monitoring** ⏳ Pre-build, architecture finalized 2026-08-22

**Hardware confirmed (photos, 2026-08-22):** two Arduino Uno R3 boards, each paired with an HC-05/06-style Bluetooth serial module. This is **Classic Bluetooth SPP (Serial Port Profile), not BLE** — the two ingestion paths originally sketched here (Meraki passive-BLE-broadcast, Pi BLE-GATT) both assumed BLE and neither applies. Sensor-to-Arduino mapping as confirmed:

- **Arduino-A:** DHT-style temperature/humidity module + a 15A/125VAC microswitch (on/off state).
- **Arduino-B:** a "Nivel Horizontal" float/level switch + a KY-017 3-pin tilt/mercury switch module.

**Ingestion path (decided): Pi-side `rfcomm` serial bridge.** Each Arduino sketch reads its attached sensors and writes a line/frame to `Serial`, which the paired HC-05/06 module transparently forwards over its Classic Bluetooth link. A Raspberry Pi Zero W (`pi-arduino`, see 6.4) binds each Arduino's module as its own serial device — `sudo rfcomm bind rfcomm0 <Arduino-A MAC> 1`, `sudo rfcomm bind rfcomm1 <Arduino-B MAC> 1` — and a Python script (`pyserial`, not `bleak`) reads both `/dev/rfcommN` devices and republishes each sensor reading to MQTT under `homelab/telemetry/arduino/<node-id>/<metric>` (e.g. `homelab/telemetry/arduino/a/humidity`, `homelab/telemetry/arduino/b/float`), per the existing topic convention (6.2). This replaces both originally-planned BLE paths; **6.3a (Meraki native telemetry) cannot see Classic Bluetooth traffic at all**, since it only observes BLE and Wi-Fi.

**One Pi serves both Arduinos (decided 2026-08-22).** BlueZ supports multiple concurrent `rfcomm` binds from a single adapter — a Classic Bluetooth piconet isn't limited to one active link, and this sensor data volume (a DHT read every few seconds, event-driven switch/tilt state changes) is trivial load for a single ARM11 core. `pi-arduino` binds and services both links concurrently rather than splitting them across two boards. **Capacity test plan:** run both `rfcomm` links on `pi-arduino` and monitor for instability — the Zero W's onboard BCM43438 is a combo Wi-Fi/Bluetooth chip sharing one antenna (relevant here because `pi-arduino` also needs Wi-Fi for its own network connectivity — no Ethernet port, see 6.4), and there are scattered reports of flakiness running multiple simultaneous Classic BT links on that specific chip under concurrent Wi-Fi load. If `pi-arduino` proves unstable running both, the reserved third Pi Zero W (`pi-reserve`) is the fallback — split Arduino-A onto `pi-arduino` and Arduino-B onto `pi-reserve`, each with its own leaf cert and status topic. Not pre-built for this — only stood up if the single-Pi test fails.

Once flowing, **PRTG historical monitoring works identically to every other metric already documented in 6.6** — one EXE/Script Advanced sensor per topic/metric (reusing the `mqtt_ble.ps1` / `mqtt_pm1630.ps1` pattern), each becoming its own PRTG channel with its own historic graph, threshold alerting, and automation-loop trigger, once Phase 5 is built. No new monitoring infrastructure is needed for this — see 6.7 for the separate question of raw time-series storage beyond what PRTG's per-channel history covers.

**Wire format and build order (confirmed 2026-08-24):** each Arduino sends one plain-text `key=value` line per reading cycle (~2s), covering all its attached sensors in one line — e.g. Arduino-A: `humidity=45.2,temperature=23.1,onoff=1`; Arduino-B: `float=1,tilt=0`. Deliberately lightweight (no JSON library needed on the Uno's limited RAM). `pi-arduino`'s collector script splits each line on commas and republishes each key as its own **retained** MQTT message under `homelab/telemetry/arduino/<node-id>/<metric>`, same retain+age-check freshness pattern already proven for `pm1630` (6.6). Build order is strictly sequential, not parallel: **Arduino-A (DHT + microswitch) is wired, coded, paired, and verified fully end-to-end — sensor through to a confirmed MQTT message — before any work starts on Arduino-B.** This is also where the concurrent-`rfcomm` capacity test actually begins: only once Arduino-B's `rfcomm1` link joins Arduino-A's already-running `rfcomm0` link on `pi-arduino` does the real dual-link load exist to test. Full staged sequence in 6.8.

**Hardware note — voltage level and serial port choice:** the HC-05/06 module's logic is 3.3V; the Arduino Uno's TX pin drives 5V. A voltage divider (or logic-level shifter) is required on the Arduino-TX → module-RX line to avoid over-driving the module — the module's TX (3.3V) into the Arduino's RX is fine as-is. Use `SoftwareSerial` on two spare digital pins for the Bluetooth link rather than hardware Serial (pins 0/1), so the USB/Serial Monitor stays free for sketch upload and debugging.

## **6.4 Raspberry Pi Zero W fleet — BLE & Classic-BT/rfcomm gateways** ⏳ Architecture finalized 2026-08-22, not yet built

**Hardware correction:** this section previously assumed a Raspberry Pi 4 with a 64-bit OS. The actual board (confirmed via photo, back-of-board silkscreen) is a **Raspberry Pi Zero W V1.1** — single-core ARM11 (armv6), 512 MB RAM. That CPU cannot run a 64-bit OS at all; every board in this fleet needs **Raspberry Pi OS Lite (32-bit/armhf)**. Expect meaningfully less headroom than the original Pi 4 plan assumed, especially for `pi-pm1630`'s persistent BLE GATT connection — this is part of why the fleet is split across three boards by role instead of stacking everything on one.

**No Ethernet port on the Zero W.** Unlike every other VLAN 30 device (mqtt, vault, adcs — all wired), these three Pis join over onboard Wi-Fi. The lab's only existing SSID (`ONCAdmin`) has VLAN tagging disabled and lands everything on VLAN 10 (Management) — not acceptable for these devices. **Blocking prerequisite, not yet done:** a new Meraki SSID with VLAN tagging enabled, tagged to VLAN 30, **and** the Aparta/Rancho AP uplink switch ports (currently `Allowed VLANs: 10 only`) widened to also permit VLAN 30 — both steps required, see `ip-vlan-plan.md` Wireless section for the planned config and the switch-port gotcha. Nothing in this section can be built until that SSID exists.

**Fleet roles (decided 2026-08-22):**

| Pi | Role | Radio work | Status |
| --- | --- | --- | --- |
| `pi-pm1630` | PM1630 BLE GATT collector (migrated from Mac, see 6.3b) | Persistent BLE GATT connection via `bleak`/BlueZ | ⏳ Planned |
| `pi-arduino` | Classic-BT/rfcomm collector for both Arduino-A and Arduino-B (see 6.3c) | Two concurrent `rfcomm` binds via `pyserial` | ⏳ Planned |
| `pi-reserve` | Unassigned — capacity fallback if `pi-arduino` can't stably hold both Arduino links at once (see 6.3c) | none yet | ⏳ Reserved, not built |

**Update 2026-08-24 — general BLE gateway role deferred, not folded in.** The original general-purpose BLE gateway role from this section's earlier draft (`ble_scanner.py` → `homelab/telemetry/ble/<mac>`) is **not** part of `pi-pm1630`'s build. Running continuous BLE discovery scanning on the same adapter that holds a persistent GATT connection to the PM1630 risks contention — the same class of risk already flagged for `pi-arduino`'s dual `rfcomm` links. `pi-pm1630` stays minimal: PM1630 collector only, reusing the existing `pm1630` mosquitto credentials (see 6.3b) — no new account needed for this build. The general gateway role, and therefore which Pi first authenticates via 8883 mTLS, is now an open question rather than a default — whichever of `pi-pm1630` or `pi-arduino` requests its Client Auth leaf cert first closes that gap (see 6.8, Stage 4).

**OS image gotcha, confirmed for both boards:** Raspberry Pi OS **Bookworm** (the current default in Raspberry Pi Imager) dropped support for armv6 boards entirely — the Zero W won't boot it. Both Pis need **Raspberry Pi OS (Legacy, 32-bit) Lite**, the Bullseye-based image, selected explicitly in Imager rather than accepting the default.

**Per-Pi build steps (repeat for `pi-pm1630` and `pi-arduino`; `pi-reserve` only if the capacity test forces a split — see 6.3c):**

- Flash **Raspberry Pi OS (Legacy, 32-bit) Lite** with hostname `<name>.lab.local`. Join the new VLAN 30 Wi-Fi SSID (see prerequisite above) — no wired option on this board.
- `pi-pm1630`: `sudo apt install -y python3-pip bluetooth bluez; pip3 install bleak paho-mqtt`.
- `pi-arduino`: `sudo apt install -y python3-pip bluetooth bluez rfcomm; pip3 install pyserial paho-mqtt`. Pair each Arduino's HC-05/06 module once (`bluetoothctl` scan/pair/trust), then bind persistently (`sudo rfcomm bind rfcomm0 <Arduino-A MAC> 1`, `rfcomm1` for Arduino-B) — needs a udev rule or a systemd `ExecStartPre` so the binds survive reboot, not just a manual one-off.
- Both Pis bootstrap on **plaintext 1883 first** (matching how every other integration in this lab was proven) — the Client Auth leaf cert request and switch to 8883 mTLS is the last step of the build, not the first (see 6.8, Stage 4).
- `pi-pm1630`: deploy the ported `pm1630_mqtt_publish.py` (BlueZ, not CoreBluetooth — see 6.3b), publishing as the existing `pm1630` user. `pi-arduino`: deploy the new `rfcomm`-reading collector script (not yet written), parsing the `key=value` line format from 6.3c and publishing to `homelab/telemetry/arduino/<node-id>/<metric>` as the new `pi-arduino` user (real mosquitto account created in Stage 2, see 6.8 — not just the commented scaffold in 6.3's ACL block).
- Each Pi sets an LWT (`homelab/status/pm1630` for `pi-pm1630`, reusing the existing topic; `homelab/status/pi-arduino` for the new Pi) — set up now even though PRTG (Phase 5) isn't built yet, so it can hook in later without a collector rewrite.
- Enable a systemd service per collector (`Restart=always`) — mirrors the originally-planned `ble-scanner.service` pattern.

## **6.8 Confirmed Build Sequence — Next Session** ⏳ Not started (confirmed 2026-08-24)

Everything below was walked through and confirmed one decision at a time (network prerequisite → `pi-pm1630` scope → `pi-arduino`/`pi-reserve` scope → broker auth approach → wire format → build order) before committing it here, per the project's own "verify before writing" rule. This is the authoritative sequence for the next build session — 6.3b, 6.3c, and 6.4 above carry the supporting rationale; this section is the checklist.

**Stage 0 — Network prerequisite (blocks everything below).** Create the new Meraki SSID (VLAN tagging enabled, VLAN ID 30); widen SW Rancho port 2 and SW Aparta port 2's allowed VLANs to include 30 (both required together, see `ip-vlan-plan.md` Wireless section); verify with a laptop/phone that joining the SSID actually gets a `192.168.30.x` DHCP lease. **✅ Complete 2026-08-31** — see `ip-vlan-plan.md` Change Log.

**Stage 1 — `pi-pm1630` (Mac → Pi migration, minimal build).** Flash, join the new SSID, install `bleak`+`paho-mqtt`, port `pm1630_mqtt_publish.py` to BlueZ, deploy as a systemd service, reuse the existing `pm1630` credentials (no new account). Verify over 1883: data on `homelab/telemetry/pm1630/data`, `homelab/status/pm1630` LWT online. Retire the Mac script once confirmed — update the PM1630 project's `pm1630-radiation-sensor.md` and this doc's 6.3b at that point. **✅ Complete 2026-09-03.**

**Stage 2 — `pi-arduino` (Pi build only, no Arduino wiring yet).** Flash, join the SSID, install `bluetooth bluez rfcomm`+`pyserial`+`paho-mqtt`. On the mqtt VM: create the real `pi-arduino` mosquitto account and activate its ACL stanza (write `homelab/telemetry/arduino/#`, write `homelab/status/pi-arduino`, read `homelab/control/pi-arduino/#`). `pi-reserve` stays completely untouched.

**Stage 3 — Arduino-A end-to-end (DHT + microswitch).** Wire both sensors to Arduino-A. Wire the HC-05/06 module via `SoftwareSerial` with a voltage divider on Arduino-TX→module-RX (see 6.3c hardware note). Confirm the module's exact identity (HC-05 vs HC-06) and default pairing PIN while wiring — doesn't change the plan, just useful to know. Sketch emits `humidity=..,temperature=..,onoff=..` every ~2s. On `pi-arduino`: pair/trust the module, `rfcomm bind rfcomm0`, persist the bind across reboot. Pi-side collector reads `/dev/rfcomm0`, publishes each metric retained under `homelab/telemetry/arduino/a/<metric>`. Verify with `mosquitto_sub` before proceeding.

**Stage 4 — mTLS layer (both Pis).** Request Client Auth leaf certs from ADCS (`pi-pm1630.lab.local`, `pi-arduino.lab.local`), switch both collectors from 1883 to 8883, verify the handshake and that data still flows. **This is what finally closes the 8883 mTLS verification gap** that's been open since the mqtt VM was built (2026-08-08).

**Stage 5 — Arduino-B (only after Stage 3 is fully verified).** Wire the float/level switch and KY-017 tilt switch to Arduino-B, wire a second HC-05/06 module, same `float=..,tilt=..` line format. Pair the second module, `rfcomm bind rfcomm1` on `pi-arduino`, extend the Pi-side script to read both `rfcomm0` and `rfcomm1` concurrently, publish under `homelab/telemetry/arduino/b/<metric>`. **This is where the concurrent-`rfcomm` capacity test actually begins** — watch for instability running both links at once. If it shows up, provision `pi-reserve` and move Arduino-B's link there.

**Explicitly out of scope for this build pass:** `pi-pm1630`'s general BLE gateway role, `pi-reserve` (untouched unless Stage 5's capacity test forces it), and any PRTG sensors (Phase 5 now in progress on `proxmox-b`, see Phase 5 above) — though the LWT/status topics above are being built now specifically so PRTG can hook in later without a redesign.

## **6.5 Automation Loop Examples**

Concrete sensor → alert → action → verify loops for the MQTT + BLE pipeline:

### **Loop 1: BLE/Arduino gateway offline detection**

- Sensor: PRTG subscribes to homelab/status/<pi-name> with 60s poll — one sensor per fleet device (`pi-pm1630`, `pi-arduino`, and `pi-reserve` if it ever gets built, see 6.4).

- Trigger: if last message > 120s ago → sensor goes Warning; > 300s → Down.

- Action: Execute Program → SSH into wg-a → try to ping the Pi; if reachable but silent, restart bluetoothd via SSH (and re-bind rfcomm on pi-arduino, since a dropped Classic BT link doesn't self-heal the way a BLE reconnect does).

- Verify: within 60s, sensor sees a fresh homelab/status/<pi-name> message → returns Green.

- Audit: PRTG publishes to homelab/audit/prtg/ble-restart with timestamp + outcome.

### **Loop 2: MQTT broker overload**

- Sensor: PRTG SSH sensor reads $SYS/broker/clients/connected on mqtt VM.

- Trigger: connected clients > threshold or broker RAM > 80%.

- Action: log to homelab/audit/prtg/mqtt-load; if sustained > 5 min, send admin alert.

- Verify: follow-up sensor tracks recovery.

### **Loop 3: Temperature threshold breach (future ESP32 sensor)**

- Sensor: PRTG subscribes to homelab/telemetry/env/<location>/temperature.

- Trigger: value > 35°C or < 5°C for 3 consecutive readings.

- Action: PRTG publishes to homelab/control/hvac/<zone>/target with corrective setpoint.

- Verify: next reading trending back into range within 10 minutes.

