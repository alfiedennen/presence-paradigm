# The committed_room template sensor

Walks through the fusion logic in `home-assistant/committed_room_single_person.yaml` block by block. The same pattern applies to the multi-person variant, just instantiated twice.

## What the sensor produces

A single string-valued sensor:

```
sensor.person_a_committed_room → "Office" | "Kitchen" | "Library" | "Living Room"
                                | "Bedroom" | "Hallway" | "away" | "unknown"
```

Updates instantly when any input changes, plus a 15-second heartbeat to catch missed updates.

## The four-priority cascade

The state template runs through four priority bands, top to bottom:

```
1. RADAR OVERRIDES        ← highest: ground truth, beats everything
2. BLE-IRK area           ← Private BLE Device's resolved area
3. Nearest-MAC fallback   ← when IRK area hasn't caught up yet
4. Hold-last + away override ← last-resort behaviour for silent BLE
```

Whichever band first produces a real answer wins.

## Band 1 — Radar overrides

```jinja
{%- if is_state('binary_sensor.office_person_a_at_desk', 'on') -%}Office
{%- elif is_state('binary_sensor.kitchen_person_a_present', 'on') -%}Kitchen
{%- elif is_state('binary_sensor.library_person_a_present', 'on') -%}Library
{%- elif is_state('binary_sensor.living_room_person_a_present', 'on') -%}Living Room
{%- elif is_state('binary_sensor.office_person_a_present', 'on') -%}Office
{%- else -%}
  ... (fall through to band 2)
```

Each `<room>_<person>_present` is a template binary sensor that AND's the room's radar with the person's BLE area (see `deploy-radar.md`):

```yaml
office_person_a_present:
  value_template: >-
    {{ is_state('binary_sensor.presence_office_radar_presence', 'on')
       and (states('sensor.person_a_phone_ble_area') | lower == 'office'
            or states('sensor.person_a_watch_ble_area') | lower == 'office') }}
```

Why the AND: the radar alone can't tell *who's* in the room. The BLE alone takes 5-30 seconds to commit to a new area. AND-ing them gives instant + identity-confirmed presence.

`office_person_a_at_desk` is a sub-zone of the office radar — fires only when the person's `(x, y)` is inside the desk polygon. Sub-zone first, generic room presence after.

`delay_off: minutes: 2` on the underlying binary sensors prevents flicker from transient radar gaps (the LD2450 sometimes loses a stationary target for a few seconds while it re-acquires).

## Band 2 — BLE-IRK area resolution

When no radar fires, fall back to the Private BLE Device integration's resolved area:

```jinja
{%- set ba = {
  'kitchen':       'Kitchen',
  'living_room':   'Living Room',
  'library':       'Library',
  'bedroom':       'Bedroom',
  'front_hallway': 'Hallway',
  'office':        'Office',
} -%}
{%- set watch_area = (states('sensor.person_a_watch_ble_area') or '') | lower | replace(' ', '_') -%}
{%- set phone_area = (states('sensor.person_a_phone_ble_area') or '') | lower | replace(' ', '_') -%}
{%- set watch_via_ble = ba.get(watch_area) -%}
{%- set phone_via_ble = ba.get(phone_area) -%}
```

The `ba` map normalises HA-area-naming (lowercase, underscored — what Bermuda spits out) to your house's display labels. Multiple BLE areas can collapse to the same display room (e.g. if you have two offices and want them both to read as "Office", map both keys to the same value).

Watch comes first because in practice **wrist-worn devices are more reliable for room-level area than pocket-worn phones**. The watch is consistently broadcasting; the phone might be in a coat downstairs while the person is in bed upstairs.

## Band 3 — Nearest-MAC fallback

When even the IRK area resolution hasn't caught up — e.g. just after a MAC rotation event — fall back to "the strongest-RSSI ESP32 board's home room":

```jinja
{%- set m = {
  'AA:BB:CC:DD:EE:01': 'Living Room',
  'AA:BB:CC:DD:EE:02': 'Office',
  ...
} -%}
{%- set phone_src = state_attr('device_tracker.person_a_phone_ble','source') -%}
{%- set watch_src = state_attr('device_tracker.person_a_watch_ble','source') -%}
{%- set watch_via_mac = m.get(watch_src) -%}
{%- set phone_via_mac = m.get(phone_src) -%}
```

The Private BLE Device device_tracker exposes a `source` attribute = the MAC of the ESP32 board currently seeing the strongest signal. Map each board's MAC to that board's home room.

Get each board's MAC from HA → Settings → Devices → ESPHome → `<board>` → "About" (or `esphome logs <config>.yaml` and look for the WiFi-connected log line).

This band is a coarse hint, not a full area resolution — if your boards are calibrated well it might be off by one room (e.g. the strongest board for a person standing in the hallway is the kitchen one because of geometry). But it's better than `unknown` while the IRK resolution catches up.

## Band 4 — Hold-last + GPS away override

```jinja
{%- set previous = this.state if this is not none else 'unknown' -%}
{%- set rooms = ['Living Room','Office','Kitchen','Bedroom','Hallway','Library'] -%}
{%- set non_transit = ['Living Room','Kitchen','Bedroom','Library','Office'] -%}
{%- set seconds_in_state = (now() - this.last_changed).total_seconds() if this is not none else 999 -%}
{%- set person_home = is_state('person.person_a', 'home') -%}
{%- set tracker = 'home' if (watch_tracker == 'home' or phone_tracker == 'home') else 'not_home' -%}

{#- Both BLE silent AND GPS says away → genuinely away -#}
{%- if tracker == 'not_home' and not person_home -%}
away

{#- BLE silent but GPS says home → hold last room -#}
{%- elif tracker == 'not_home' -%}
  {%- if previous in rooms -%}{{ previous }}
  {%- else -%}unknown
  {%- endif -%}

{#- We have a candidate from BLE → use it, but suppress hallway transits -#}
{%- elif candidate -%}
  {%- if candidate == 'Hallway'
        and previous in non_transit
        and seconds_in_state < 30 -%}
    {{ previous }}
  {%- else -%}
    {{ candidate }}
  {%- endif -%}

{%- else -%}
unknown
{%- endif -%}
```

Three behaviours encoded:

**Genuinely away**: both BLE goes silent AND the HA `person` entity (which incorporates GPS / wifi presence / etc) says not home → `"away"`.

**BLE silent, person home**: BLE proxies briefly lose sight of the device (deep sleep, RSSI dip behind a wall) but GPS still has them → hold the last known room. Prevents flicker to `"unknown"` while the person is just sitting still in the bedroom and their watch went to sleep.

**Hallway-transit suppression**: the hallway is the connector room — anyone walking from kitchen to bedroom passes through it for 5-15 seconds. Without suppression, the committed_room would briefly flick to `"Hallway"` mid-transit, which makes downstream automations fire spuriously (e.g. a "turn off kitchen lights when room changes" automation would fire on the transit, not the destination). The suppression: if the candidate is "Hallway" and the previous committed room was a real room and we've only been in the hallway for < 30 seconds, hold the previous room. Once the person actually settles in the hallway for > 30 seconds (e.g. talking to someone at the door), the sensor commits to "Hallway" properly.

## Tunables

A few constants you might tweak per household:

| Tunable | Default | When to change |
|---|---|---|
| Hallway transit threshold | 30 seconds | Lower if your hallway is small + transits are quick (10-15s); higher if your hallway is sometimes a "wait at the door" zone |
| Heartbeat interval | 15 seconds | Lower (5s) for faster reaction at the cost of more sensor evaluations; higher (60s) for less HA load if the trigger-on-state-change path is reliable |
| `delay_off` on radar binary sensors | 2 minutes | Lower if you have a multi-person household and want presence to update faster when someone leaves; higher if you have a single inhabitant who likes to sit very still |
| `non_transit` list | `['Living Room','Kitchen','Bedroom','Library','Office']` | Add any other rooms that count as "real" (not transit) — bathrooms, dining rooms, etc |

## Multi-person variant

`committed_room_multi_person.yaml` is structurally identical, just instantiated twice for `Person A` and `Person B`. Each person needs:

- Their own IRKs registered with Private BLE Device
- Their own per-room `<room>_<person>_present` template binary sensors (one per room with radar × person count)
- Their own HA `person` entity for the GPS away override

The two committed_room sensors are independent — Person A's room doesn't influence Person B's calculation, and vice versa.

## Debugging

Use the `attributes` block on the template sensor to expose the raw inputs:

```yaml
attributes:
  watch_ble_area: "{{ states('sensor.person_a_watch_ble_area') }}"
  phone_ble_area: "{{ states('sensor.person_a_phone_ble_area') }}"
  ...
```

In HA → Developer Tools → States, look up the sensor and you can see the underlying signals at the time the state was computed. Useful when the committed room "flickered" to something unexpected and you want to know which input drove it.

## Performance

The sensor evaluates on every state change of any input + a 15-second heartbeat. On a typical household with 2 people × 4 inputs each = 8 BLE entities + 5 radar entities = ~150 evaluations per minute when active, near-zero when everyone's still.

HA template sensors are cheap (~1ms each) so this is well within budget. If you're seeing perf issues, your template is doing too much work — strip the nearest-MAC fallback (band 3) which has the most string-handling overhead.

## See also

- `the-paradigm.md` — the architectural argument
- `calibrate.md` — getting the BLE areas to actually resolve correctly
- `gotchas.md` — the trap list
