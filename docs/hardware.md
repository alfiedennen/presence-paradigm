# Hardware

What to buy. Roughly £40-£60 per room with everything; cheaper if you skip radar in some rooms.

## Per-room

### ESP32 BLE proxy — one per room (mandatory)

| Item | Specifics | Approx cost | Notes |
|---|---|---|---|
| **ESP32 DevKit V1** (or NodeMCU-32S clone) | 38-pin, classic ESP32 (not S2/S3/C3 unless you adapt the pin map) | ~£6 each | Available on Amazon, AliExpress, distributor sites. The reference firmware is calibrated against this board |
| **USB-A to micro-USB cable** + 5V power supply | Anything ≥1A | from your drawer | Per board, runs on a permanent USB feed |

### LD2450 mmWave radar — one per room you want fine-grained presence in (optional)

| Item | Specifics | Approx cost | Notes |
|---|---|---|---|
| **HLK-LD2450** | Hi-Link 24 GHz mmWave radar, 3-target X/Y/speed at 10 Hz | ~£9 | The component this whole stack is calibrated against |
| **Jumper wires** | 4 female-to-female, ~10 cm | pennies | 5V, GND, TX, RX |
| **Mounting**: a small plastic enclosure or just hot glue | from your drawer | n/a | The LD2450 is happiest mounted on a wall or shelf with a clear view of the room — see `deploy-radar.md` for placement |

**Power gotcha** — the LD2450 needs 5V. Most ESP32 DevKit V1 clones don't pass USB 5V through to the VIN pin (they only feed VIN to the 3.3V regulator's input). You CAN'T just wire LD2450 5V to the ESP32 VIN. See `gotchas.md` → "ESP32 DevKit V1 VIN pin trap" for the three workarounds.

**Cable colour gotcha** — the LD2450 ships with a non-standard wiring colour scheme (NOT "red = 5V, black = GND"). Read `gotchas.md` → "LD2450 cable colour trap" before connecting or you risk frying the radar.

## Modern alternative — XIAO ESP32-S3

If you'd rather use a more modern board, the **XIAO ESP32-S3** (~£8) is a good upgrade path:
- Tiny (21 × 17.5 mm), trivial to wall-mount discreetly
- USB-C
- 5V is exposed on a real pin — sidesteps the VIN gotcha entirely
- More RAM, faster CPU

Trade-off: you'll need to adapt the `esp32:` block and the UART pin numbers in the radar config (XIAO-S3 default UART pins differ from DevKit V1's `GPIO16/GPIO17`).

For a fresh build, XIAO-S3 is what we'd use today. The references in this repo target DevKit V1 because that's what the household ran during the Spring 2026 build, and the gotchas are best-documented against that board.

## Per-household — IRK capture (one-time)

You need ONE ESP32 with the IRK capture firmware temporarily — flash, capture IRKs from each device, then reflash back to the BLE proxy firmware. Use any one of the boards you already bought; no extra hardware needed.

(See `capture-irk.md` for the full procedure.)

## Per-inhabitant — devices that broadcast resolvable BLE

The IRK path needs each tracked person to carry a BLE-broadcasting device — typically:

- **Modern Android phone** (Pixel 6+, recent Samsung etc.) — broadcasts a resolvable identity by default
- **Apple Watch** (recent gen) — pair with the IRK-capture firmware via Heart Sensor profile
- **iPhone** (recent iOS) — same; pair via Health Devices

Older devices may not broadcast a resolvable identity. The IRK path works with whatever broadcasts a resolvable BLE address; if your device doesn't, you fall back to the iBeacon-broadcaster route (use a free Android app like "BLE Beacon" to broadcast a fixed UUID, then track via Bermuda only).

## Optional add-ons

| Item | What for |
|---|---|
| **Aqara FP300** (PS-S04D) or similar passive-presence sensor | Backup human-confirmation signal; the multi-signal vote variant of the template sensor uses one in the kitchen and one in the hallway |
| **Hi-Link USB-RS485 dongle** | If you want to flash LD2450 firmware updates or change radar config (zone polygons can also be set via the dongle's Hi-Link tool) — not strictly needed; ESPHome can drive the radar without it |
| **Spare ESP32** | Useful for testing — you can run irk-capture on it without disrupting your active BLE proxies |

## Total cost — a six-room household

Reference: the haroldathome setup is six BLE proxies + four LD2450 radars in the four rooms with most activity:

| Item | Qty | Unit | Total |
|---|---|---|---|
| ESP32 DevKit V1 | 6 | £6 | £36 |
| HLK-LD2450 | 4 | £9 | £36 |
| Jumper wires + plugs | a kit | £8 | £8 |
| 5V USB supplies | 6 | £4 | £24 |
| **Total** | | | **~£100** |

Plus a few weekend hours setting it up, calibrating the boards, capturing IRKs, and tuning the template sensor.

## What you don't need

- **A coordinator hub.** All six ESP32s talk to HA directly via WiFi.
- **A separate HA instance.** Runs on whatever HA install you already have.
- **A subscription.** The whole stack is local, MIT-licensed, no cloud dependencies.
- **A PoE switch.** USB power per board is fine.
- **Mains wiring.** Each board is a USB-powered fingertip-sized device that can sit on a shelf.
