# Capturing IRKs

The Identity Resolving Key (IRK) is what lets Home Assistant resolve a device's rotating BLE addresses back to a stable identity. Each tracked device has one. You capture it once, paste it into HA's Private BLE Device integration, and the device is then trackable forever — across MAC rotation, deep sleep, firmware updates, anything.

## Security warnings — read first

The IRK is a **cryptographic secret**. It's the key that defeats the BLE MAC-rotation privacy feature. Treat it accordingly.

- **Never commit IRK values to a public repository.** Or any repository, ideally — even private repos drift through backups, AI sessions, screenshots
- **Capture only your own household's devices.** Capturing someone else's IRK from across the street is wiretapping
- **Reflash the IRK-capture board back to its normal firmware after capture.** Leaving an IRK-capture firmware running in your house is a needless pairing surface
- **Storage**: keep IRKs in your HA's `secrets.yaml` (which is in `.gitignore` by default), or in a password manager
- **Rotation**: there's no easy way to rotate an IRK. Once captured, treat it as a long-lived credential — you have to factory-reset the device to re-roll it

## Procedure (Apple Watch)

The Apple Watch is the easiest device to capture. Here's the flow:

1. Take any spare ESP32 (or borrow a presence-mesh board temporarily)
2. Flash the IRK capture firmware:
   ```sh
   esphome run esphome/irk-capture.example.yaml
   ```
   (USB connection required for first flash)
3. Wait for the board to come up. In HA → Settings → Devices & Services → ESPHome → "irk-capture" you should see a `select.ble_profile` entity
4. Set `select.ble_profile` to **"Heart Sensor"** (default; the firmware sets this on boot)
5. On the Apple Watch:
   - Health app → top-right user icon → Health Devices → "Add Bluetooth Device"
   - The watch finds "irk-capture" advertised as a heart sensor
   - Tap to pair (no PIN prompt)
6. The pairing handshake exchanges the IRK. In HA, watch the `text_sensor.last_irk` entity update
7. Note down the IRK value (32-character hex string)
8. Note down the `text_sensor.last_address` value too — this is the device's static identity address (not the rotating BLE address — the original pre-rotation address)

You now have one IRK. Repeat per device.

## Procedure (iPhone)

Same as Apple Watch — iPhone uses the same Health app's "Add Bluetooth Device" flow with the Heart Sensor profile.

## Procedure (Android phone — Pixel etc)

Android pairs slightly differently:

1. On the IRK capture board, set `select.ble_profile` to **"Generic Access"** (not Heart Sensor — Android doesn't have a one-tap Health flow like iOS)
2. On the Android phone: Settings → Connected devices → Pair new device
3. The phone shows "irk-capture" in its discovery list
4. Tap to pair. You'll see a PIN prompt — enter `123456` (or whatever the irk-capture firmware logs to the ESPHome console; default is no PIN with Generic Access on most builds)
5. Watch `text_sensor.last_irk` for the captured value

If pairing fails, check the ESPHome console (`esphome logs irk-capture.yaml`) for diagnostic output — the upstream `DerekSeaman/irk-capture` README also has troubleshooting.

## Procedure (Android Wear / Pixel Watch)

Same as Android phone — pair via the watch's Bluetooth settings. Pixel Watch may need to be temporarily disconnected from its companion phone for the pairing to find the irk-capture board.

## After capture — reflash to normal duty

Once you have all the IRKs you want, **reflash the board back to its normal presence-room firmware**. Don't leave irk-capture running.

```sh
esphome run esphome/presence-room.example.yaml
```

(Edit the substitutions block to set the room name first.)

## Add IRKs to Home Assistant

1. Settings → Devices & Services → "Add Integration" → "Private BLE Device"
2. Paste the IRK (32-character hex)
3. Optionally name the device (e.g. "Person A's Pixel Watch")
4. HA creates:
   - `device_tracker.<device_name>` — home/not_home + GPS-equivalent state
   - `sensor.<device_name>_area` — area resolution (only populated once Bermuda has gathered enough samples to map the device's signal to your areas)
   - `sensor.<device_name>_distance` — calibrated distance from the strongest scanner

Repeat per IRK. Now your committed_room template sensor has the inputs it needs.

## What to do if a device's IRK can't be captured

Some devices' BLE stacks don't expose their IRK during pairing — older Android, some kids' watches, some health trackers. Workarounds:

- **iBeacon broadcaster fallback**: if the device can't broadcast a resolvable IRK identity but CAN broadcast iBeacon (some apps allow this), use iBeacon + Bermuda for area resolution. This is what the BLE proxy's `ble_rssi` sensors are for. You lose the cryptographic identity protection but gain functional tracking
- **OS-level BLE address access**: NOT recommended — defeats the privacy feature your platform vendor specifically added
- **Just use radar**: skip BLE entirely for this person, rely on radar presence + per-room defaults. Loses identity (anyone in the kitchen counts), but works

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `text_sensor.last_irk` stays blank after pairing | Pairing didn't complete the security handshake — usually because the device thinks it's already paired | Forget the device on the phone/watch first, then re-pair |
| iOS won't show "irk-capture" in the Heart Sensor list | Profile set to Generic Access instead of Heart Sensor | Set `select.ble_profile` → "Heart Sensor" in HA, wait 5 seconds for the firmware to advertise the new profile, retry |
| ESP-IDF build fails on `esphome run` | Toolchain version mismatch | Update to ESPHome 2024.11.0+ — earlier versions may have IDF version issues with the irk-capture external component |
| Captured an IRK but Private BLE Device shows no area for the device | Bermuda hasn't seen enough broadcasts from the device yet | Carry the device around the house for 30 minutes, the area sensor will populate |

## Reference

- Upstream IRK capture firmware: [DerekSeaman/irk-capture](https://github.com/DerekSeaman/irk-capture)
- Home Assistant Private BLE Device integration: built-in, no docs link needed
- Why IRK matters (Bluetooth SIG paper): "Resolving Random Private Addresses" in the BLE Core Specification
