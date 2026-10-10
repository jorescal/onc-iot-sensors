# Changelog: iot-sensors

Format: `YYYY-MM-DD [origin] summary`. Origin is `hub`, `section` or `import`.

- 2026-10-10 [import] Section created. Moved §6.3c, §6.4, §6.8 and §6.5 verbatim from the hub's
  `homelab_build_plan_v5.md` into `docs/iot-sensor-fleet.md` (tokenized). Known gap: the hub's `ip-vlan-plan.md`
  refers to "build plan 6.3d/6.3e" for the Arduino-C/D (`pi-hvac`) design, but those sections didn't exist in
  the build plan as exported 2026-10-10. The design detail needs to be recovered or rewritten here.
- 2026-10-10 [hub] Added §6.3d (Arduino-C: DHT11 + IR AC control) and §6.3e (Arduino-D: SCT-013 + ZMPT101B energy
  monitor), **reconstructed** from the hub's `ip-vlan-plan.md` references (the original 2026-08-29 build-plan text wasn't
  exported). Only documented facts; open design points listed. Renamed the 6.4 `pi-reserve` row to `pi-hvac` and added
  the 2026-08-29 note that the capacity-fallback role no longer exists.
