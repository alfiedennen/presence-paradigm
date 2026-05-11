# Redaction record

Transparent log of what was scrubbed when this code was extracted from
its original homes (`homeassistant/esphome/presence-*.yaml` for the
ESPHome firmwares, `homeassistant/config/configuration.yaml` for the
template sensors) to make it safe for public release.

The originals continue to live in the haroldathome.com private repo;
this repo is the generic-ised, shareable version.

## ESPHome configs (`esphome/*.example.yaml`)

| Original | Public version | Reason |
|---|---|---|
| `ssid: "HaroldHome"` (literal household WiFi name) | `ssid: !secret wifi_ssid` | Generic; user supplies their own via `secrets.yaml` |
| `password: !secret wifi_password` | unchanged | Already a secret reference; ship `secrets.example.yaml` template |
| `key: !secret presence_node_api_key` | unchanged | Same — secrets convention preserved |
| `password: !secret presence_node_ota_password` | unchanged | Same |
| `ibeacon_uuid: "923b5c66-a0c6-4826-8c26-66469ea0d3f4"` (Watch broadcaster UUID) | `ibeacon_uuid: "<watch-uuid>"` placeholder | UUID identifies a specific household's broadcasting device |
| `ibeacon_uuid: "80f55214-3945-421b-a129-037620b515a6"` (Phone broadcaster UUID) | `ibeacon_uuid: "<phone-uuid>"` placeholder | Same |
| Per-room `name:` and `friendly_name:` ("presence-office", etc) | Generic `presence-room` template with substitutions block at top | One file per room, user duplicates per room |
| Comments referencing `memory/feedback_*` paths in haroldathome private repo | Re-written generically; the relevant lessons are baked into `docs/gotchas.md` | Internal memory file paths aren't useful out of context |
| Comment "192.168.1.78 is the old DEVKIT-V1 board" (a host IP) | Removed entirely — generic VIN-trap commentary in `docs/gotchas.md` | Don't ship internal LAN topology |

## Template sensors (`home-assistant/*.yaml`)

| Original | Public version | Reason |
|---|---|---|
| Sensor name `Alfie Committed Room` | `Person A Committed Room` (with Alfie/Elena examples in case-study docs) | Generic by default; users substitute their own names — drop-in for any household |
| Entity IDs like `sensor.alfie_pixel_watch_private_ble_area` | Generic `sensor.person_a_watch_ble_area` | Same |
| Specific room labels `'alfies_office': 'Office'`, `'elenas_office': 'Office'` | Just `'office': 'Office'` | Two-office-collapse case is household-specific; documented as a pattern in `docs/template-sensor.md` |
| Entity prefix `device_tracker.alfie_phone_ble` | `device_tracker.person_a_phone_ble` | Same as above |
| Per-room radar binary sensors `binary_sensor.office_alfie_present` | `binary_sensor.office_person_a_present` | Same |
| Specific MAC addresses in nearest-MAC fallback (`E0:8C:FE:59:88:82`, etc) | Placeholder MACs (`AA:BB:CC:DD:EE:01`, etc) with explanatory comments | Don't ship household-specific board MACs |
| Trilateration board positions (`0.20, 7.35`, etc — actual house coordinates) | Kept as ILLUSTRATIVE example values, with comment "calibrate for your own house" | Coordinates are useless out of context but illustrate the format; users replace |
| Tunables documented inline (post-23:00 bedroom override, hallway transit threshold, etc) | Single-person variant ships the simple form; haroldathome private repo's deeper tuning lives in `docs/template-sensor.md` as patterns to consider | Reduces shipped YAML weight without losing pedagogical content |
| Bermuda iBeacon UUID-derived entity names (e.g. `sensor.bermuda_80f552143945421ba129037620b515a6_100_40004_area`) | Replaced with the cleaner Private BLE Device pattern (`sensor.<device>_area`) which doesn't surface UUIDs | Same — UUIDs are household-specific |
| Comments referencing `room.js v8.4`, `Nearness viewer`, `Standing Wave` (haroldathome-internal browser-side components) | Generalised — the template sensor doesn't depend on any of those | Out of scope for this repo |
| Specific FP300 entity IDs (`binary_sensor.0x54ef441001694ded_presence`) | Removed from the simple committed_room template; the multi-signal-vote variant is documented as a pattern in `docs/template-sensor.md` (didn't ship as a separate YAML to keep scope tight) | Zigbee hex names are device-specific; the pattern is portable |
| Specific WiFi-zone sensor (`sensor.alfie_zone` — RSSI-based, household-specific) | Removed from the shipped templates; documented as an OPTIONAL signal in `docs/template-sensor.md` | Custom router-side script (`/jffs/scripts/rssi-tracker.sh` on an ASUS router) — heavy lift to extract; out of scope |

## What was NOT shipped

- **Any actual IRK cryptographic keys.** These live in `secret_irks_2026_04_24.md` in the private repo's gitignored memory directory. The capture procedure is documented in `docs/capture-irk.md`; no actual key values exist anywhere in this repo
- **The full 6-room set of named ESPHome configs.** Per-room files would just be the same template with different `name:` substitutions — wasteful. Ship the template, document per-room usage in `esphome/README.md`
- **The deeply-tuned multi-signal vote variant** (`Alfie Fused Room` template sensor in the private repo). Uses an FP300 + a router-side WiFi RSSI script + post-23:00 bedroom default — too household-specific for a general repo. The tuning patterns are documented in `docs/template-sensor.md` for users who want to add their own
- **The raw RSSI tracker shell script** (lives on the household's ASUS router, pushes RSSI → HA via REST). Useful in some home topologies but not generally applicable; out of scope
- **Lovelace dashboards / automations consuming the committed_room sensor.** Application-specific; everyone's house wants different things
- **The `Alfie Position` trilateration centroid** is shipped (as `ble_trilateration.yaml`), but with the per-board coordinate tuples kept as ILLUSTRATIVE rather than redacted — they're the original household's measurements and useful as a format reference. Users replace them per their own house

## Verification before push

The `.githooks/pre-commit` hook scans the staged diff for the patterns
documented above plus generic secret-shape patterns. Bypass is
`--no-verify`; never used legitimately.

## Licence audit

- All code files in this repo: MIT (LICENSE)
- All documentation, prose, illustrative coordinate values: CC-BY-NC 4.0 (LICENSE-CONTENT)
- No third-party code is bundled. The IRK-capture firmware uses
  `DerekSeaman/irk-capture` as an external_component fetched at build
  time. The trilateration centroid sensor is original work
- All `!secret` references documented in `home-assistant/secrets.example.yaml` for users to fill in
