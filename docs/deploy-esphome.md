# Deploy the BLE proxies

End-to-end recipe to take this code from `git clone` to one ESP32 per room running as a `bluetooth_proxy` for HA.

## Prerequisites

- Home Assistant 2024.6+ with the **ESPHome add-on** (or a standalone ESPHome install)
- The hardware in `hardware.md` — ESP32 DevKit V1 boards
- Your WiFi network reachable from where the boards will sit
- A USB-A to micro-USB cable for first flash (subsequent updates go via OTA over WiFi)

## Step 1 — set your secrets

In ESPHome's `secrets.yaml` (in HA → ESPHome add-on → top-right → "Secrets" or your equivalent file location), add:

```yaml
wifi_ssid: "your-wifi-network-name"
wifi_password: "your-wifi-password"
presence_node_api_key: "<32-byte base64 — generate with `openssl rand -base64 32`>"
presence_node_ota_password: "<a strong random password — also used for AP fallback>"
```

A reference template is in `../home-assistant/secrets.example.yaml`.

## Step 2 — copy the BLE proxy template

Copy `../esphome/presence-room.example.yaml` into your ESPHome configs directory as `presence-<roomname>.yaml`. Repeat per room.

For each, set the `substitutions` block at the top:

```yaml
substitutions:
  name: presence-office
  friendly_name: "Presence Node — Office"
```

Each board needs a unique `name`. The `friendly_name` is what you'll see in HA's device list.

## Step 3 — fill in iBeacon UUIDs (optional)

The `ble_rssi` sensors at the bottom of the config have placeholder UUIDs (`<watch-uuid>`, `<phone-uuid>`). These are for the OPTIONAL trilateration path — they expose per-board RSSI for one or two specific iBeacon-broadcasting devices.

If you don't have any iBeacon broadcasters yet:
- **Skip the `ble_rssi` sensors entirely** for now (just delete those two blocks). The IRK / Private BLE Device path doesn't need them.
- **Or set up a broadcaster** later: a free Android app like "BLE Beacon" or "Beacon Tools" can broadcast iBeacon for testing without writing any client code. Generate a UUID per device (`uuidgen` or any online generator), set the major/minor to anything (we use 100/40004), put the UUID in your config.

## Step 4 — flash via USB (first time)

In ESPHome dashboard:
1. Click "Install" on your `presence-office.yaml`
2. Choose "Plug into the computer running ESPHome Dashboard" (HA add-on path) or "Manual download" + flash via `esphome run` from CLI
3. Plug the ESP32 into USB
4. Wait for the upload (~30 seconds + ~30 seconds for first boot + WiFi join)

Once the board's connected to WiFi and reporting to HA, future updates go via OTA — no USB needed.

## Step 5 — repeat per room

One board per room. Walk around your house with the laptop, plug in each board, flash, install in its final location.

Tip: physically label each board (a Sharpie on the case is fine) with which room it goes in — once they're all flashed and look identical it's hard to tell apart.

## Step 6 — verify in HA

In HA → Settings → Devices & Services → ESPHome:
- Each board should appear with the friendly name you set
- Click into one — you should see entities for `Uptime`, `WiFi Signal`, plus your `Watch RSSI` and `Phone RSSI` if you set up beacons

In HA → Settings → Devices & Services → Bluetooth (built-in):
- Once your boards are running as `bluetooth_proxy: active`, they appear as "Bluetooth scanners"
- HA's Bluetooth integration auto-discovers them

## Step 7 — install Bermuda HACS

[Bermuda](https://github.com/agittins/bermuda) is the integration that takes the per-board RSSI signals from your BLE proxies and produces room-level area sensors via trilateration.

1. In HA → HACS → Integrations → "+" → search for "Bermuda BLE Trilateration"
2. Install
3. Restart HA
4. Settings → Devices & Services → "Add Integration" → "Bermuda BLE Trilateration"
5. Bermuda picks up your BLE proxies automatically

You'll need to:
- Set up an Area for each room in HA (Settings → Areas & Zones) and assign each ESPHome board to its room
- Wait a few hours for Bermuda to gather calibration data
- Tune per-board reference power offsets (see `calibrate.md`)

## Step 8 — install Private BLE Device

This is built-in to HA — no add-on required:

1. Settings → Devices & Services → "Add Integration" → "Private BLE Device"
2. Paste each tracked device's IRK (see `capture-irk.md` for capturing them)
3. HA creates a `device_tracker` and `sensor.<device>_<irk>_area` for each

Once you have Bermuda + Private BLE Device working, you have the per-room IRK identity layer running. Add radars (`deploy-radar.md`) for ground-truth body presence in the rooms you care about.

## Step 9 — wire up the template sensor

See `../home-assistant/committed_room_single_person.yaml` (or `_multi_person.yaml`) and `template-sensor.md` for the fusion layer that turns Bermuda + Private BLE Device + radar into a single committed room per person.

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| Boards keep falling off WiFi | Board is right at the edge of WiFi reception, OR your router rotates DHCP leases on the boards | Move the board closer / set static DHCP reservations on your router for each board's MAC |
| ESP-IDF errno=128 spam in logs | Some dev-board MAC frames hit a known bug in ESP-IDF + `bluetooth_proxy` | Use the Arduino framework, not ESP-IDF (the example configs already do this) |
| Bermuda shows everyone in one room regardless of where they are | RSSI calibration — `ref_power` offsets per board not set | See `calibrate.md` |
| Private BLE Device shows the device but never resolves an area | Bermuda hasn't tagged that device's broadcasts to a room yet | Walk around the house with the device for ~30 minutes, gives Bermuda enough samples |

## Next steps

- `deploy-radar.md` — add LD2450 to a room
- `capture-irk.md` — capture IRKs for each inhabitant
- `template-sensor.md` — wire up the committed_room sensor
- `calibrate.md` — tune Bermuda + the radar mapping
