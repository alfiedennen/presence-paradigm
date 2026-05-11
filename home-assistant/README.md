# home-assistant/

Reference Home Assistant template sensors for the presence mesh.

## Files

| File | Purpose |
|---|---|
| `secrets.example.yaml` | Reference template for the `!secret` keys ESPHome and (optionally) HA need |
| `committed_room_single_person.yaml` | Stable, debounced "which room is the person in?" sensor for a single-person household. Fuses radar overrides + BLE-IRK area + nearest-MAC fallback + GPS away override |
| `committed_room_multi_person.yaml` | The two-person variant. Independent fusion per person; per-person room presence requires per-person template binary sensors (pattern documented inline) |
| `ble_trilateration.yaml` | OPTIONAL — continuous 2D position sensor (smooth live-map shape) computed via inverse-square-weighted RSSI across all your ESP32 boards |

## How to wire these into HA

If you use a single `configuration.yaml`, paste the contents under your existing `template:` block.

If you use the [packages](https://www.home-assistant.io/docs/configuration/packages/) feature (recommended for anything non-trivial), drop these files into `homeassistant/packages/presence/` and add this to `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

After dropping these in, edit them in-place to match your own:

- Entity IDs (replace `person_a_phone_ble`, `person_a_watch_ble`, etc. with your own Private BLE Device entity names — they're auto-generated when you add devices via the integration)
- Room labels in the `ba` map
- ESP32 board MACs in the `m` map (look them up in HA → Settings → Devices → ESPHome → `<board>` → "About")
- Per-room radar binary sensor names (your room layout decides which apply)
- Person entity (`person.person_a` → your HA Person entity name)

## Required dependencies

- **Home Assistant 2024.6+** with `template:` (built-in)
- **Private BLE Device integration** (built-in) — provides `sensor.<device>_<irk>_area` and `device_tracker.<device>_<irk>` entities once you've added each device's IRK
- **Bermuda HACS integration** (optional but recommended) — provides finer-grained area resolution + trilateration distance, on top of the IRK identity. Gives the BLE proxies somewhere to feed; without it you'd need a different way to map RSSI → area
- **ESPHome** (built-in or add-on) — for the BLE proxies + LD2450 radars

See `../docs/the-paradigm.md` for the architectural relationship between Private BLE Device, Bermuda, ESPHome, and the LD2450 radars.
