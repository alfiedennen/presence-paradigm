# Changelog

## 1.0.0 — 2026-05-11

Initial public release. Extracted from the haroldathome.com private
monorepo (`homeassistant/esphome/presence-*.yaml` for the ESPHome
firmwares, and the `template:` blocks of `homeassistant/config/configuration.yaml`
for the fusion sensors) and generic-ised for public use.

### What's in 1.0.0

- **3 ESPHome reference firmwares**: BLE-only proxy, BLE + LD2450 radar, IRK capture
- **3 Home Assistant template sensor files**: single-person committed_room, multi-person variant, optional 2D trilateration centroid
- **`secrets.example.yaml`** template for the four `!secret` keys ESPHome needs
- **9 documentation files** covering: the architectural argument, hardware bill of materials, end-to-end ESPHome deployment, LD2450 radar deployment, IRK capture procedure (with security warnings), template-sensor logic walkthrough, calibration methodology, known limits, and the trap list

### Reference deployment

The original household runs:
- 6× ESP32 DevKit V1 BLE proxies (one per room: living room, office, bedroom, kitchen, hallway, library)
- 4× HLK-LD2450 mmWave radars in the four most-active rooms (office, library, living room, kitchen)
- Two tracked inhabitants via Pixel Watch 3 + Pixel 9 Pro (one) and Apple Watch + iPhone (the other)
- Bermuda HACS for trilateration distance + area refinement
- Home Assistant Private BLE Device for IRK identity resolution

See [haroldathome.com/presence](https://haroldathome.com/presence) for the
narrative version of the architecture and the deployment story.

### Companion repositories

- [alfiedennen/the-long-take](https://github.com/alfiedennen/the-long-take) — visualises the LD2450 radar output as a 24-hour space-time tube. Direct downstream consumer of the radar layer here
- [alfiedennen/mr-graves](https://github.com/alfiedennen/mr-graves) — on-device wake word for Android. Pair with this paradigm's committed_room sensor to make voice respond contextually to whoever's actually in the room
