# Deploy an LD2450 mmWave radar

End-to-end recipe to add a Hi-Link LD2450 to one of your rooms, wired into an existing ESP32 BLE proxy.

## When to add radar to a room

Not every room needs radar. The IRK identity layer alone tells you *which person is in which room*; radar adds *ground-truth body presence*, which:

- Tightens the room transition (fires within 100ms of someone entering vs the IRK path's 5-30 seconds)
- Gives you sub-room zone granularity ("at desk", "on sofa", "in doorway")
- Disambiguates pet vs human less well than you'd hope (radar can't tell), but adds confidence on multi-body rooms
- Powers visualisations like the Long Take ([haroldathome.com/the-long-take](https://haroldathome.com/the-long-take))

Practical recommendation: radar in your office, kitchen, living room, library. Skip radar in bathrooms (privacy, plus FP300 with PIR is enough), bedrooms (you don't need fine-grained tracking of where in the bed someone is), and corridors (presence is transient by definition).

## Hardware

Per radar:
- 1× HLK-LD2450 (~£9)
- 4× female-female jumper wires
- A small enclosure or just hot glue
- That room's existing ESP32 board (we wire the radar in alongside the BLE proxy)

## ⚠ Two gotchas BEFORE you wire

1. **Cable colour trap** — see `gotchas.md` → "LD2450 cable colour trap". The LD2450 ships with a NON-STANDARD wiring colour scheme. Verify against the datasheet before connecting; getting it wrong can fry the radar.

2. **VIN pin trap** — see `gotchas.md` → "ESP32 DevKit V1 VIN pin trap". On most DevKit V1 clones, the VIN pin doesn't pass through USB 5V. The LD2450 needs 5V. You CAN'T just wire LD2450 5V to the ESP32 VIN. Three workarounds:
   - Power the LD2450 from a separate USB→5V breakout
   - Tap directly into the USB connector's 5V pin on the board
   - Use a board that does pass through 5V (XIAO ESP32-S3, Lolin S3 Mini)

Read both before connecting.

## Wiring

LD2450 pins → ESP32 pins (with the VIN gotcha resolved):

| LD2450 | ESP32 (DevKit V1) | Note |
|---|---|---|
| 5V | (separate 5V source per gotcha #2) | LD2450 power |
| GND | GND | shared ground with ESP32 |
| TX | GPIO16 (RX) | radar TX → ESP32 RX |
| RX | GPIO17 (TX) | radar RX → ESP32 TX |

(GPIO16/17 are the safe defaults on classic ESP32; swap if your specific dev board reserves them. The reference config uses these.)

## Software — flash the radar firmware

Replace the BLE-only `presence-room.example.yaml` for that room with `presence-room-with-radar.example.yaml`:

```sh
cp esphome/presence-room-with-radar.example.yaml \
   /path/to/your/esphome/configs/presence-office.yaml
```

Edit `substitutions:`:
```yaml
substitutions:
  name: presence-office
  friendly_name: "Presence Node — Office"
```

Install via ESPHome dashboard. The first install adds the LD2450 external component (downloads from GitHub); subsequent flashes are quick.

## Mounting

Where to mount matters a lot. The LD2450 has a roughly 60° horizontal × 40° vertical field of view. Mount it:

- **At ~1.5–2.0m height** (mid-chest for an adult standing). Higher than head-height misses people standing up; floor-level is dominated by furniture clutter
- **In a corner of the room** with the antenna face pointing into the room. Avoid pointing at large reflective surfaces (TVs, mirrors, glass) — they cause phantom targets
- **Away from running fans and HVAC** — the moving blades fire as targets
- **Away from the doorway** unless you specifically want doorway-transit detection (then mount it pointing AT the doorway)

The radar's coordinate frame is `+X = right of the antenna, +Y = forward (away from antenna face)`. You'll translate this to your house frame in `calibrate.md`.

## Verify in HA

After flashing, the new entities appear under the ESPHome device:

| Entity | What |
|---|---|
| `binary_sensor.presence_office_radar_presence` | Any body in the field of view (smoothed by ESPHome) |
| `binary_sensor.presence_office_radar_moving` | Moving body |
| `binary_sensor.presence_office_radar_still` | Still body |
| `sensor.presence_office_target_count` | 0–3 |
| `sensor.presence_office_t1_x`, `_t1_y`, `_t1_speed` | Target 1 X/Y/speed (mm, mm, mm/s) |
| (same for `_t2_*` and `_t3_*`) | Targets 2 and 3 |

Test by walking through the room and watching the entities update in HA → Developer Tools → States.

## Tune the polygon zones (optional)

For sub-room zones ("at desk", "on sofa"), define polygon coordinates in the radar config:

```yaml
zone:
  - name: "Desk"
    polygon:
      - { x: -1500, y:  200 }
      - { x:  -300, y:  200 }
      - { x:  -300, y: 1400 }
      - { x: -1500, y: 1400 }
```

X and Y are in millimetres in the radar's coordinate frame. To derive these for your space:
1. Sit in the position you care about (e.g. at your desk)
2. Watch `sensor.presence_office_t1_x` and `_t1_y` for ~30 seconds — you'll see your X/Y oscillate within a small range
3. Take that range, expand by ~200mm padding, that's your zone polygon

A polygon fires `binary_sensor.<zone>_present` whenever any target's position lands inside it.

## Per-person presence — combine radar + IRK area

The radar can tell you *a body* is in the office. To say *which person*, AND-it with the BLE area:

```yaml
binary_sensor:
  - platform: template
    sensors:
      office_person_a_present:
        friendly_name: "Office — Person A present"
        delay_off:
          minutes: 2
        value_template: >-
          {{ is_state('binary_sensor.presence_office_radar_presence', 'on')
             and (states('sensor.person_a_phone_ble_area') | lower == 'office'
                  or states('sensor.person_a_watch_ble_area') | lower == 'office') }}
```

This is the per-room input that the committed-room template sensor uses for its highest-priority "RADAR OVERRIDE" branch. See `template-sensor.md`.

## Performance

- LD2450 native rate: 10 Hz
- ESP32 → HA via WiFi: tested fine at 1 Hz for fusion sensors, 250ms for visualisations
- Long-term storage: see `haroldathome.com/the-long-take` for one approach (decimate to ~30k points per 24h per room, write JSON to disk every 5 min, render via Three.js)

## Next

- `template-sensor.md` — wire your radar binary sensors into the committed_room fusion
- `calibrate.md` — translate radar coordinates to house coordinates
- `gotchas.md` — the trap list including the cable + VIN ones referenced above
