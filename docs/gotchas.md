# Gotchas — the trap list

Specific things that bit us during the May 2026 build. Save yourself the time.

## Hardware

### ESP32 DevKit V1 VIN pin trap

**Symptom**: Connect LD2450 5V to ESP32 VIN; LD2450 doesn't power up. Or worse, you fry the radar.

**Cause**: Most ESP32 DevKit V1 clones (the cheap 38-pin DOIT-style boards) DO NOT pass USB 5V through to the VIN pin. The board has an AMS1117 3.3V regulator with VIN as its INPUT pin — VIN is for *external* power going INTO the regulator, not USB power coming out. When you're powering the ESP32 via USB, the VIN pin is floating or near-zero, NOT 5V.

**Fix** — three options, in order of effort:
1. **Cleanest**: power the LD2450 from a separate 5V source (a USB → barrel-jack breakout, or a second USB port on a charger). Share GND with the ESP32 only
2. **Hacky but works**: solder a wire from the USB connector's 5V pad directly to a free pin (or the LD2450). Bypasses the on-board regulator entirely
3. **Modern alternative**: switch to a board that does pass through 5V on a real pin — XIAO ESP32-S3 (5V on the dedicated 5V pin), Lolin S3 Mini (same), Adafruit ESP32 Feather (USB 5V exposed on `USB` pin)

**Reference**: this trap cost ~3 hours during the Spring 2026 build. The original `presence-office.yaml` config in the haroldathome repo carries a comment noting the LD2450 plan was "deferred until S3 boards arrive" — that's because of this gotcha.

### LD2450 cable colour trap

**Symptom**: Wire the LD2450 by colour convention (red = 5V, black = GND). LD2450 doesn't power up, or worse, you reverse-feed it.

**Cause**: HLK ships the LD2450 with a 4-pin JST-XH connector + flying leads. The colour scheme is **NON-STANDARD**:

| HLK cable colour | Function |
|---|---|
| Red | 5V (this one matches convention) |
| Black | GND (this one matches) |
| Yellow | TX |
| White | RX |

This is mostly fine — but cheap clone replacement cables sometimes ship with COMPLETELY different colour conventions. Some ship with green = TX, blue = RX. Some ship with red on the wrong end of the connector.

**Fix**: BEFORE connecting, verify with a multimeter that the wire you think is 5V is connected to the LD2450 pin labelled `5V` (silkscreen on the board). The pin labels are tiny but they're there.

**Reference**: cost ~1 hour during the Spring 2026 build. One LD2450 was returned (DOA after misconnection); a multimeter check on the next one prevented a repeat.

### ESP-IDF + bluetooth_proxy errno=128 loop

**Symptom**: Flash an ESP32 with the BLE proxy firmware using ESP-IDF framework. Logs fill with `errno=128 (ENOSPC)` infinite loop within seconds of starting `bluetooth_proxy`. Board hangs / restarts.

**Cause**: A specific class of BLE advertisement frames (some Apple device frames seem to trigger it) hit a bug in ESP-IDF's bluetooth_proxy implementation that the Arduino-framework path doesn't exhibit.

**Fix**: Use Arduino framework, not ESP-IDF, for the BLE proxy boards. The example configs in this repo (`presence-room.example.yaml`) already do this.

**Note**: the IRK capture firmware (`irk-capture.example.yaml`) DOES use ESP-IDF, because the irk_capture external component requires it. This is fine — that firmware doesn't run bluetooth_proxy, only the BLE bonding for IRK exchange.

## Bermuda

### Bermuda area_id trap

**Symptom**: You assign each ESP32 board to a room area in HA. Bermuda still resolves areas randomly / inconsistently, with rooms it shouldn't reach claiming the device.

**Cause**: Bermuda doesn't actually use the HA-assigned area on the SCANNER. It uses the area assigned to the BLE device. The scanner-area is metadata; the device area is what determines area resolution.

**Fix**: For each tracked BLE device (your phone, watch), make sure it's assigned to a single Area in HA → Settings → Devices & Services → Bermuda → device → Area. Bermuda will then publish that device's `_area` sensor based on which scanner is reporting strongest signal — but the area name comes from the SCANNER's assignment, not the device's.

OK, this is confusing. The model is:
- Each scanner (ESP32 board) has an `Area` in HA — this should be the room the board is in
- Each tracked device has an `Area` in HA — this is just metadata, doesn't affect resolution
- Bermuda's `sensor.<device>_area` reports the AREA NAME of the strongest-RSSI scanner

So the scanner areas are what matter. Make sure each board's `Area` in HA matches the room it's physically in.

**Reference**: cost ~2 hours during the Spring 2026 build before realising the area assignment had to be on the SCANNERS, not the devices.

### Bermuda calibration "everyone is in one room"

**Symptom**: Bermuda says everyone is in the kitchen, regardless of where they actually are.

**Cause**: One scanner has a much higher RSSI than the others (maybe it's centrally located, maybe its `ref_power` calibration is off). The strongest-RSSI-scanner-wins logic always picks it.

**Fix**: Per-scanner `ref_power` calibration — see `calibrate.md`. Adjust the dominant scanner DOWN until the area assignments correlate with where the device actually is.

## Home Assistant

### `device_tracker.<device>_<irk>` source attribute is null

**Symptom**: Your committed_room template sensor's nearest-MAC fallback (band 3) never fires because `state_attr('device_tracker.X', 'source')` returns None.

**Cause**: The `source` attribute on Private BLE Device's tracker is only populated when there's an active resolution. If the device hasn't broadcast in the last few seconds, `source` is null.

**Fix**: This is mostly a wait-it-out — the source attribute populates once the device's next broadcast lands on a board. If you're seeing this regularly (more than a few times an hour), it suggests your BLE proxies aren't picking up enough broadcasts — probably one or more boards are out of range of the device.

### Template sensor "TypeError: 'NoneType' object is not subscriptable"

**Symptom**: When the committed_room template runs and one of the source entities is `unavailable`, you get template errors in HA's logs.

**Cause**: Jinja's `is_state` and `states()` handle `unavailable` gracefully but indexing into them doesn't. The template uses `(states('sensor.X') or '') | lower` to neutralise this — make sure you preserve those `or ''` defensive guards in your customisations.

### Multiple `template:` blocks in `configuration.yaml`

**Symptom**: HA refuses to load `configuration.yaml` with "Duplicate key 'template'".

**Cause**: YAML doesn't allow duplicate top-level keys. If you `!include` multiple files that each have `template:` at the top level, they collide.

**Fix**: Either combine them into one `template:` list, OR use the `packages` feature (`homeassistant.packages: !include_dir_named packages`) which merges them properly.

The reference template-sensor files in `home-assistant/` are written assuming you'll either paste them under your existing `template:` block, or drop them in the `packages/` directory.

## IRK capture

### `text_sensor.last_irk` blank after pairing

**Symptom**: You pair your device with the irk-capture board. Pairing appears to complete on the device side. `text_sensor.last_irk` stays blank.

**Cause**: The pairing handshake didn't complete the security exchange. Most often: the device thinks it's already paired (cached pairing from a previous attempt).

**Fix**: On the device, "Forget" the irk-capture pairing first. Then re-pair from scratch.

### iOS won't show "irk-capture" in the Heart Sensor list

**Symptom**: You set the irk-capture board's profile to "Heart Sensor" but iOS Health → Add Bluetooth Device doesn't show it.

**Cause**: The board's BLE profile change takes a few seconds to propagate after the `select.set` in HA. iOS scanned the BLE airwaves while the board was still advertising as a Generic Access device.

**Fix**: After setting the profile in HA, wait 5-10 seconds before scanning from iOS. If still failing, restart the irk-capture board (re-flash, OR power-cycle, OR use the `Restart` button HA exposes for the device).

## See also

- `deploy-esphome.md` — main deployment recipe (links here from the gotchas section)
- `deploy-radar.md` — radar deployment (calls out the cable + VIN traps in line)
- `capture-irk.md` — IRK capture procedure
- `calibrate.md` — Bermuda + room-classification tuning
