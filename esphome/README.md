# esphome/

ESPHome firmwares for the presence mesh.

## Files

| File | Purpose |
|---|---|
| `presence-room.example.yaml` | BLE-only proxy. Drop one per room you want presence detection in. Acts as `bluetooth_proxy` for HA (so the IRK + Bermuda integrations can use it) AND exposes per-beacon RSSI sensors directly |
| `presence-room-with-radar.example.yaml` | BLE proxy + LD2450 mmWave radar combined. Use this in rooms where you want fine-grained body-position presence (per-target X/Y/speed at 10 Hz) on top of the BLE-IRK identity |
| `irk-capture.example.yaml` | Temporary single-purpose firmware. Flash onto a spare ESP32 to capture IRKs from your phones / watches, then reflash back to the normal presence-room firmware |

## Workflow

1. **Buy hardware** — see `../docs/hardware.md`
2. **Capture IRKs first** — flash `irk-capture.example.yaml` onto one ESP32, capture each device's IRK, paste into HA's Private BLE Device integration. See `../docs/capture-irk.md`
3. **Reflash with the BLE proxy** — same ESP32, swap to `presence-room.example.yaml`. Repeat per room (one ESP32 per room)
4. **Add radar to interesting rooms** — for rooms where you want fine-grained presence (your office, the kitchen, the living room), use `presence-room-with-radar.example.yaml` instead. See `../docs/deploy-radar.md`
5. **Wire up the template sensor** — see `../home-assistant/`

## Required `!secret` keys

The configs reference four keys via `!secret`. Define them in your ESPHome
add-on's `secrets.yaml` (or the equivalent if you're using a standalone
ESPHome install):

```yaml
wifi_ssid: "your-wifi-name"
wifi_password: "your-wifi-password"
presence_node_api_key: "<32-byte base64 key — generate one with `openssl rand -base64 32`>"
presence_node_ota_password: "<a strong password for OTA updates + AP fallback>"
```

A reference template is in `../home-assistant/secrets.example.yaml`.

## Adapt per room

For each room, set the `substitutions` block at the top:

```yaml
substitutions:
  name: presence-office          # short identifier — appears in HA + URL
  friendly_name: "Presence Node — Office"
```

Each board needs a unique `name`. The `friendly_name` is what you'll see in
HA's device list.

## ESP32 board choice

These configs are written against `esp32dev` (the classic 38-pin DevKit V1
clone — ~£6 each, available everywhere). For modern alternatives (XIAO
ESP32-S3, Lolin S3 Mini) change the `esp32:` block per ESPHome's docs and
account for the different pin numbering on the LD2450 UART variant.

See `../docs/hardware.md` for board recommendations and the ESP32 DevKit V1
VIN-pin gotcha (relevant to the radar variant).
