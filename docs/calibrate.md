# Calibrate

Calibration is what takes the system from "Bermuda thinks everyone's in the kitchen all the time" to "the right room reliably". Allow ~1 evening for the initial pass and ~2 weeks of small tweaks to settle.

## Bermuda — per-board reference power

Bermuda's trilateration depends on each ESP32 board having an accurate `ref_power` — the RSSI a tracked beacon would read at exactly 1 metre from the board. Without this, the distance estimates are skewed and area resolution stays bad.

### Procedure

1. Take a tracked device (your phone, your watch — anything with an IRK already in HA's Private BLE Device integration)
2. Stand exactly 1 metre from one ESP32 board, device held at chest height
3. Watch HA → Developer Tools → States → search for `sensor.<device>_<irk>_distance` (or the per-board RSSI sensor for the device if you have iBeacon broadcasters set up)
4. The RSSI reading at that moment is your `ref_power` for that board
5. Repeat per board

### Set in Bermuda

In HA → Settings → Devices & Services → Bermuda → Configure → Scanner:

For each scanner (= each ESP32 board), set its `ref_power` offset. The reference power applies *per scanner* — different boards in different locations / orientations will have slightly different values.

A typical range for a phone-class iBeacon is `-58 dBm to -68 dBm` at 1m. If your readings are way outside this, the device might be transmitting at unusual power, OR the board has been mounted somewhere with unusual signal characteristics (e.g. inside a metal box).

### Iteration

After the initial pass, walk through the house with the device for ~30 minutes. Watch the per-room area assignments update. If one room consistently mis-attributes (e.g. people in the kitchen show up as in the dining room), tweak the kitchen board's `ref_power` down by 2-3 dB to make its claimed distances longer. If a room never wins (e.g. the office is "too far away"), tweak its `ref_power` up.

## Path-loss exponent N

The trilateration centroid sensor (`ble_trilateration.yaml`) uses a default path-loss exponent of `N = 2.5`, suitable for a typical home with normal walls and furniture.

If your house has unusual signal characteristics:
- **Thicker walls** (Victorian solid brick, stone) → bump N to 2.7-3.0
- **More open plan** (modern apartment, fewer interior walls) → drop N to 2.0-2.2
- **Lots of metal** (steel-frame conversion) → bump N higher (3.0+) and accept that your trilateration will be noisy

You can measure N empirically — take readings at 1m, 2m, 5m, plot RSSI vs log(distance), the slope gives you `-10 × N`. Most people don't bother and just use 2.5.

## Coordinate frame for the trilateration sensor

`ble_trilateration.yaml` ships with placeholder `(x, y, floor)` tuples per board. For your house:

1. Pick an origin. Convention: SW corner of the ground floor, x increasing east, y increasing north
2. Measure each board's position. Use a tape measure or floorplan
3. Pick a floor index per board. Convention: 0 = ground, 1 = first, 2 = second, -1 = basement
4. Fill in the `boards` tuple in `ble_trilateration.yaml`

If your house has irregular geometry (extension, single-storey wing), the trilateration is still useful — just place the boards' coordinates accurately and the centroid will be approximately right.

## LD2450 radar — coordinate frame translation

The LD2450 reports targets in its own frame:
- `+X` = right of the antenna face
- `+Y` = forward (away from antenna face)

Translation to your house frame depends on:
- Where on the wall the radar is mounted
- What direction the antenna face points

Procedure for one radar:

1. Note the radar's mounting position in house coordinates (`origin_x`, `origin_y` — measure from your house origin)
2. Note the rotation angle of the antenna face (e.g. 0° = pointing east, 90° = pointing north)
3. To translate radar `(rx, ry)` to house `(hx, hy)`:
   ```
   hx = origin_x + rx*cos(angle) - ry*sin(angle)
   hy = origin_y + rx*sin(angle) + ry*cos(angle)
   ```

For most use cases (per-room presence, polygon zones for "at desk") you don't actually NEED house-frame coordinates — the radar's own frame is fine. House-frame coordinates matter only if you want to render targets on a unified house map (like the live-map use case in [haroldathome.com/the-long-take](https://haroldathome.com/the-long-take)).

## Polygon zones inside the radar's frame

For sub-room zones ("at desk", "on sofa"), you stay in the radar's frame:

1. Sit in the position you want to define a zone for (e.g. at your desk)
2. Watch `sensor.presence_office_t1_x` and `_t1_y` for ~30 seconds
3. Note the X/Y range
4. Add ~200mm padding on each side
5. Define the polygon in `presence-room-with-radar.example.yaml`'s `zone:` block

Example: if X varies from -1500 to -800 mm and Y from 1000 to 1300 mm while you're at the desk:

```yaml
zone:
  - name: "Desk"
    polygon:
      - { x: -1700, y:  800 }
      - { x:  -600, y:  800 }
      - { x:  -600, y: 1500 }
      - { x: -1700, y: 1500 }
```

Each polygon fires `binary_sensor.<zone>_present` whenever any target's position is inside it.

## Tuning the per-room <person>_present binary sensors

These derive per-person presence by AND-ing radar with that person's BLE area. The `delay_off` is a critical tunable:

```yaml
office_person_a_present:
  delay_off:
    minutes: 2
  value_template: >-
    {{ is_state('binary_sensor.presence_office_radar_presence', 'on')
       and (states('sensor.person_a_phone_ble_area') | lower == 'office'
            or states('sensor.person_a_watch_ble_area') | lower == 'office') }}
```

`delay_off: 2 minutes` means: when the AND condition becomes false, wait 2 minutes before flipping the binary sensor off. Reasons:

- **Radar dropouts**: LD2450 occasionally loses a stationary target for 5-10 seconds while it re-acquires. Without delay_off, the sensor would flicker
- **BLE dropouts**: Watch goes briefly unreachable (1-3 seconds is common), area sensor becomes "unavailable" momentarily, AND-condition false, sensor flickers
- **Person leaves the room briefly**: walk to the door for 30 seconds to grab a thing, come back. Without delay_off, downstream automations see leave + return rapid-fire

Tune up if you have a multi-person household and want fast updates when someone genuinely leaves; tune down if you have a single inhabitant who likes to sit very still.

## Verification — the walk test

The single best way to verify your calibration is a **timed walk test**.

1. Start with HA → Developer Tools → States open, looking at `sensor.person_a_committed_room`
2. Walk slowly through every room: kitchen → dining → living room → hallway → office → bedroom → back through hallway → library → kitchen
3. At each room, pause for 1 minute, note the committed_room state
4. Note any rooms where the sensor:
   - Stays on the previous room (BLE area not updating fast enough — check Bermuda calibration)
   - Jumps to a wrong room (a board's `ref_power` is off — adjust it)
   - Goes to "unknown" (BLE proxies aren't reaching, OR the IRK isn't broadcasting from the device — check the device's BLE settings)

A well-calibrated system updates within 5-10 seconds of room entry, with no spurious mid-transit jumps to "Hallway".

## When to recalibrate

- New ESP32 board added or moved → recalibrate that board's `ref_power`
- Major furniture rearrangement (RF environment changes) → re-walk the test, adjust 1-2 boards as needed
- New device added (new phone, new watch) → capture the new IRK, add to Private BLE Device, walk-test
- Bermuda integration update → check release notes; calibration usually survives but worth a quick walk-test
- Seasonal change (windows open vs closed, doors regularly closed in winter) → minor drift; usually self-corrects

## See also

- [Bermuda upstream calibration docs](https://github.com/agittins/bermuda) — the canonical reference for Bermuda-specific tuning
- `template-sensor.md` — what the calibrated inputs feed into
- `gotchas.md` — common calibration traps
