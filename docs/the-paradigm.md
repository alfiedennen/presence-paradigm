# The paradigm — identity from a key, presence from radar

This document is the WHY behind everything else. The architectural argument for separating "is anyone here" from "which person is here" into two independent layers.

## The problem with one-signal presence

Most home-automation presence systems try to do two jobs at once. They use a single signal — Bluetooth, Wi-Fi, a phone's GPS — to answer both *is anyone here* and *which person is here*. Bluetooth in particular is unreliable for this because every modern phone and watch rotates its MAC address every fifteen minutes, by design, to prevent exactly the kind of tracking smart-home systems try to do.

You end up with a system that:

- **Loses identity at every MAC rotation** — the phone the system was tracking thirty seconds ago is, as far as Bluetooth can tell, a completely new device now
- **Conflates "someone is here" with "this specific person is here"** — when you only have one signal, you can't separate the two questions
- **Fails on multi-person households** — when both inhabitants are home, a single-signal system can't tell whose phone it just saw
- **Compromises privacy when patched** — the common workaround is to read the device's permanent MAC via OS APIs, which means defeating the rotation feature your platform vendor specifically added to protect you

## The fix — two layers

Separate the questions. Use one signal that answers each one well.

### Layer 1 — presence

**Millimetre-wave radar** (Hi-Link LD2450 or similar) mounted in each room you care about. Detects bodies — *any* body — at about eight readings a second, reporting position and Doppler speed. It does not know who anyone is. It is reliable at the question *is there a body in this room?* and excellent at the question *how is it moving?*. It is identity-blind by design.

### Layer 2 — identity

**Bluetooth — but with a critical twist.** Each wearable has a permanent cryptographic *Identity Resolving Key*, or IRK, which can be captured once with a brief sniffing procedure (see `capture-irk.md`). Once the IRK is known, a custom firmware on a few ESP32 nodes distributed through the house can resolve the rotating MAC addresses back to a stable identity, in real time. The identity is private — it cannot be tracked by anyone without the IRK. It survives MAC rotation, deep sleep, and most other things that defeat naïve Bluetooth tracking.

This is what Home Assistant's built-in **Private BLE Device** integration does. You paste each inhabitant's IRK once, and HA produces a `device_tracker` and `sensor.<device>_area` for each that survive MAC rotation forever.

### Combining

The two layers combine into a third sensor: **per-room occupants**.

If the radar fires in the library and the identity layer says Person A's watch is in the library, the room reports `[person_a]`.

If radar fires but no person is identified, it reports `[unknown]` — a guest, a pet, a reflection.

If two people are identified in the same room and the radar reports two bodies, it reports `[person_a, person_b]`.

The committed-room template sensor in `home-assistant/committed_room_single_person.yaml` is the reference implementation of this fusion.

## Why this is unusual

Most homes that have radar do not have an identity layer. Most homes that have identity tracking use a method that defeats privacy. Combining IRK-resolved identity with radar presence gives a multi-person home the property of *attribution* — knowing who is where — without either compromising privacy or losing reliability.

It also means you can:

- Heat the room the inhabitant is actually in, not the one their phone said it was in fifteen minutes ago
- Light the room they walked into without knowing who, if anyone, just walked in
- Hand off conversations between rooms when they cross from kitchen to library
- Tell the wake-word listener on the office wall whether the inhabitant standing in front of it is the one who set up the system or a guest
- Run "everyone is home" / "everyone is out" automations that actually work

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

## What this paradigm does NOT solve

- **Devices that don't broadcast BLE** — old phones, dumbphones, kids' tablets without Bluetooth. The IRK path needs your device to broadcast resolvable BLE; it doesn't help with anything that doesn't.
- **Guests** — visitors don't have IRKs registered with your HA. They show up as `[unknown]` body presence; the system knows someone is in the kitchen but not who. (Arguably correct behaviour.)
- **Pet vs human** — radar can't tell a body from a body. Cats moving across a desk fire `at_desk`. Dogs fire kitchen presence. (Often a feature: see `haroldathome.com/the-long-take` — "the kitchen tube is busier than the others, mostly because the dogs go in and out of the garden through the back door.")
- **Floor distinction** — radar is 2D; if you have a sensor on the floor below detecting body activity, you can't easily tell if it's the same person who just moved one floor down vs a different person. The trilateration centroid sensor's `floor` attribute helps but isn't perfect.
- **Multiple people in the same room with radar** — LD2450 reports up to three targets but doesn't track them across frames; identity must come from the BLE side.

These are the bounds. Within them, the system works.

## Further reading

- `hardware.md` — bill of materials
- `deploy-esphome.md` — flashing the BLE proxies
- `deploy-radar.md` — the radar integration
- `capture-irk.md` — capturing IRKs (with security warnings)
- `template-sensor.md` — the fusion logic walkthrough
- `calibrate.md` — Bermuda + room-classification tuning
- `gotchas.md` — the trap list
- [haroldathome.com/presence](https://haroldathome.com/presence) — the narrative version of this paradigm
