# Known limits

The piece is locked at v1; the boundaries below are deliberate. PRs that work *within* them are welcome.

## Hardware

- **ESP32-class boards only** for the BLE proxies. ESP8266 doesn't have a BLE radio. The reference is `esp32dev` (DevKit V1); modern alternatives like XIAO ESP32-S3 work with pin-map adjustments but aren't bundled
- **Hi-Link LD2450 only** for the radar. Other mmWave radars (e.g. Aqara FP300) report binary presence but NOT per-target X/Y position — unsuitable for the fine-grained zones + the Long Take visualisation companion piece. PIR sensors are useful as backup confirmation signals (the multi-signal-vote variant uses one) but not as the primary radar layer
- **One radar per room** for the reference deployment. Multi-radar fusion in a single large room is doable (treat each radar as a separate target stream) but not bundled here

## BLE / IRK

- **The IRK path requires resolvable BLE** from the tracked device. Most modern phones / watches broadcast resolvable IRK identities by default, but some health trackers, kids' watches, and very old devices don't. Workaround: iBeacon-broadcaster fallback (use a free Android app to broadcast a fixed UUID, fall back to Bermuda area only — loses the cryptographic identity protection)
- **You need to capture the IRK once** per device. Some manufacturers don't expose IRK during pairing — those devices can't be tracked via this path. Workarounds in `capture-irk.md`
- **Per-MAC-rotation latency** — the IRK resolution typically catches up to a new MAC within 10-30 seconds of rotation. Most use cases tolerate this; for very latency-sensitive automation (e.g. unlock-on-arrival), the GPS away override + nearest-MAC fallback bands of the committed_room sensor cover the gap

## Bermuda + Private BLE Device

- **Bermuda is a HACS integration**, not built-in to HA. If HACS goes away or Bermuda is unmaintained, you lose the trilateration distance + finer area resolution. The IRK path via Private BLE Device (built-in) survives independently
- **Bermuda calibration drifts** with significant furniture rearrangement. After a major reshuffle, expect to walk-test and adjust 1-2 boards' `ref_power`. See `calibrate.md`
- **Private BLE Device area resolution depends on Bermuda** in this stack. If you don't install Bermuda, the IRK device tracker still gives you home/not_home but the per-room area sensor won't populate

## Multi-person

- **Per-person presence requires per-person template binary sensors** — the radar can only tell you a body is in a room, not whose. The committed_room_multi_person template documents the AND pattern but you have to instantiate it per (person, room) pair
- **Two people in the same room with overlapping radar targets** — LD2450 reports them as separate targets but doesn't track identity across frames. The committed_room sensor handles this fine (both BLE areas resolve to the same room → both committed_room sensors say that room) but if you want to know which target is which person, you need additional logic (e.g. compare radar (x,y) to each person's last known position)
- **Guests** — visitors don't have IRKs registered. Radar fires but the AND with BLE area never matches. Result: `binary_sensor.<room>_<person>_present` stays off, committed_room falls back to the IRK path (which also says nothing for guests). The room shows a body in the radar entity but no committed person — arguably correct behaviour for a privacy-oriented system

## Pet vs human

- **Radar can't tell** a pet body from a human body. If your cat sits at your desk, the office_at_desk binary sensor fires
- **The "kitchen busier than the others" effect** is mostly dogs (see [haroldathome.com/the-long-take](https://haroldathome.com/the-long-take))
- **Workarounds**: weight thresholds (radar can't measure weight); height thresholds (LD2450 is 2D, no Z); time-of-day heuristics (the cat is on the desk only at night); these all work imperfectly

## Floor distinction

- **Radar is per-room; trilateration is rough on Z**. If you have an ESP32 board on the floor below detecting body activity, the system can be confused about which floor the person is on
- **Workaround**: use the `floor` attribute of the trilateration centroid sensor, OR rely on the IRK area which (with proper Bermuda calibration) gets per-room resolution that's effectively per-floor

## Privacy

- **The system runs locally** — no signals leave your network. But the IRK is a household secret; treat your `secrets.yaml` with the same care you'd give a password file
- **Anyone with physical access to your network** can read the BLE proxy outputs (the per-board RSSI sensors, etc) and from those infer presence. Lock down your network like any other smart-home setup
- **The IRK capture firmware is a pairing surface while running**. Reflash to normal duty as soon as you've finished captures (see `capture-irk.md`)

## Performance / cost

- **Battery**: each ESP32 board on permanent USB draws ~1-2W idle (~$2/year per board at typical electricity rates)
- **Network**: ~10-50 BLE advertisement messages/second from the boards to HA via WiFi. Negligible
- **HA load**: template sensor evaluations are ~1ms each. With 6 boards × 2 people × ~0.5 BLE updates/sec, ~6 evaluations/second per committed_room sensor — well within HA's budget
- **MariaDB recorder**: the per-board RSSI sensors and per-target radar X/Y/speed sensors generate a LOT of state events (8-10 readings/sec/sensor). Consider excluding the radar X/Y/speed entities from `recorder` if you don't need historical playback (the Long Take companion piece consumes them — keep them recorded if you want to use that)

## What's NOT in this repo

- **The Long Take visualisation** — that's a separate piece at [`alfiedennen/the-long-take`](https://github.com/alfiedennen/the-long-take). It consumes the LD2450 X/Y/speed data this paradigm produces
- **The Capacitor app for iBeacon broadcasting** — out of scope. Use a free Android app like "BLE Beacon" / "Beacon Tools" if you need a broadcaster for the trilateration path
- **Ready-made Lovelace dashboards** — your house, your dashboard. The committed_room sensor is the input; what you do with it is yours
- **Automation examples** — likewise. The point of this repo is the SENSOR; what you trigger from it (lights, heating, presence-aware speakers) is application-specific

## Tested against

- Home Assistant 2024.6 → 2026.5 (continuously through that range)
- ESPHome 2024.11.0+
- Bermuda 0.10.x
- Pixel Watch 3 + Pixel 9 Pro (tested as the broadcasting devices)
- Apple Watch + iPhone (tested as the broadcasting devices)
- HLK-LD2450 firmware v1.07.20H (the version that ships from Hi-Link as of mid-2026)
