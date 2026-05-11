# presence-paradigm

> Identity from a cryptographic key, presence from millimetre-wave radar. Two layers, one architectural fix for multi-person home automation. Companion to [haroldathome.com/presence](https://haroldathome.com/presence). MIT (code) + CC-BY-NC 4.0 (docs).

Most home-automation presence systems try to do two jobs at once. They use a single signal to answer both *is anyone here* and *which person is here*. Bluetooth is unreliable for this because every modern phone rotates its MAC every fifteen minutes, by design, to prevent exactly that kind of tracking.

This stack separates the questions. **Radar** answers *is there a body in this room*. **BLE-IRK** answers *which person is it*. They fuse into a single per-person committed-room sensor that's both reliable and privacy-respecting.

---

## What you get

- **ESPHome firmware** for ESP32 BLE proxies (BLE-only, plus a BLE + LD2450 variant for rooms you want fine-grained presence in)
- **Reference Home Assistant template sensors** for committed_room (single- and multi-person variants)
- **IRK capture firmware** — temporary single-purpose, flash → capture → reflash
- **Optional 2D trilateration centroid sensor** for live-map use cases
- **9 docs** covering the architectural argument, hardware, deployment, calibration, the trap list

---

## What you need

- 1× ESP32 (DevKit V1, ~£6) per room you want presence in
- Optional: 1× HLK-LD2450 mmWave radar (~£9) per room you want fine-grained body-position presence in
- Home Assistant 2024.6+ with ESPHome + Bermuda HACS + Private BLE Device (built-in)
- An iOS / Android / Apple Watch device per inhabitant (whose IRK you'll capture once)

Total hardware for a typical 6-room household with 4 radars: ~£100. See [`docs/hardware.md`](docs/hardware.md).

---

## Quickstart

1. Read [`docs/the-paradigm.md`](docs/the-paradigm.md) — the architectural argument (5 min)
2. Buy hardware per [`docs/hardware.md`](docs/hardware.md)
3. Flash one ESP32 with the IRK capture firmware ([`docs/capture-irk.md`](docs/capture-irk.md)), capture each device's IRK
4. Reflash that board (and the others) with the BLE proxy firmware ([`docs/deploy-esphome.md`](docs/deploy-esphome.md))
5. Add Private BLE Device + Bermuda integrations to HA, paste the IRKs
6. (Optional) add LD2450 to interesting rooms ([`docs/deploy-radar.md`](docs/deploy-radar.md))
7. Drop the committed_room template sensor into your config ([`home-assistant/`](home-assistant/))
8. Calibrate ([`docs/calibrate.md`](docs/calibrate.md))
9. Read [`docs/gotchas.md`](docs/gotchas.md) BEFORE you hit them, not after

---

## Architecture

```
Each phone / watch                     mmWave radar
broadcasts BLE                         (LD2450, identity-blind)
(rotating MAC)                                  │
        │                                       │
        ▼                                       ▼
  ESP32 BLE proxies                    ESPHome on same ESP32
  (one per room)                       (where you've fitted radar)
        │                                       │
        ▼                                       ▼
   Bermuda HACS                         per-room
   (trilateration + area)               binary_sensor.<room>_radar_present
        │
        ▼
  Private BLE Device                            │
  (built-in HA — IRK                           │
   resolution, stable identity)                │
        │                                       │
        └──────────────┬─────────────┬─────────┘
                       │
                       ▼
        committed_room template sensor
        (per-person, debounced, fused)
                       │
                       ▼
        sensor.person_a_committed_room
        sensor.person_b_committed_room
                       │
                       ▼
        consume from automations,
        Lovelace dashboards,
        downstream agents
```

---

## Repo layout

```
presence-paradigm/
├── README.md                                this file
├── LICENSE                                  MIT (code)
├── LICENSE-CONTENT                          CC-BY-NC 4.0 (docs)
├── REDACTION.md                             transparent record of scrubs
├── CHANGELOG.md
├── esphome/
│   ├── README.md
│   ├── presence-room.example.yaml           BLE-only proxy
│   ├── presence-room-with-radar.example.yaml  BLE proxy + LD2450
│   └── irk-capture.example.yaml             temporary IRK capture firmware
├── home-assistant/
│   ├── README.md
│   ├── secrets.example.yaml
│   ├── committed_room_single_person.yaml    single-person fusion sensor
│   ├── committed_room_multi_person.yaml     two-person variant
│   └── ble_trilateration.yaml               OPTIONAL — 2D position sensor
├── docs/
│   ├── the-paradigm.md                      WHY: the two-layer argument
│   ├── hardware.md                          BOM + recommendations
│   ├── deploy-esphome.md                    end-to-end BLE proxy recipe
│   ├── deploy-radar.md                      LD2450 mounting + wiring + ESPHome
│   ├── capture-irk.md                       IRK capture procedure (with security warnings)
│   ├── template-sensor.md                   committed_room logic walkthrough
│   ├── calibrate.md                         Bermuda + room tuning
│   ├── known-limits.md
│   └── gotchas.md                           the trap list
└── credits/
    └── haroldathome.md
```

---

## See also

- [**haroldathome.com/presence**](https://haroldathome.com/presence) — the narrative version of this paradigm
- [**alfiedennen/the-long-take**](https://github.com/alfiedennen/the-long-take) — the Three.js renderer that visualises 24 hours of LD2450 radar data per room. Direct downstream consumer of the radar output here
- [**alfiedennen/mr-graves**](https://github.com/alfiedennen/mr-graves) — companion repo for on-device wake-word detection. Pair the wake word with the committed-room sensor to make voice respond to whoever's actually in the room
- [**Bermuda BLE Trilateration**](https://github.com/agittins/bermuda) — the upstream HACS integration this paradigm depends on
- [**DerekSeaman/irk-capture**](https://github.com/DerekSeaman/irk-capture) — the upstream IRK capture ESPHome component
- [**openWakeWord**](https://github.com/dscripka/openWakeWord) — used for the wake word in the same household

---

## Licences

| Scope | Licence |
|---|---|
| Code (`esphome/`, `home-assistant/`, `.githooks/`) | **MIT** — see [`LICENSE`](LICENSE) |
| Documentation, prose, diagrams | **CC-BY-NC 4.0** — see [`LICENSE-CONTENT`](LICENSE-CONTENT) |
