# Whole-Home Presence "Radar" with Home Assistant

Track people room-by-room — and watch live dots move around your floorplan — using cheap hardware: BLE for *who*, mmWave (LD2450) for *where*. Total hardware cost for a 6-room build: **under $100** (assuming you already run Home Assistant and have a couple of ESP32-based BLE proxies).

This documents a real build, end to end: BLE trilateration → per-room occupancy fusion → LD2450 mmWave nodes → a floorplan dashboard with moving position dots.

## Architecture

```
Phone (iBeacon via HA Companion app)
   └─► BLE proxies (ESPHome / HA Voice PE) ─► Bermuda (room-level "who/where")
                                                  │
Phone activity sensor (walking/still) ────────────┤
Media players, cameras, door sensors ─────────────┼─► Area Occupancy Detection
LD2450 mmWave nodes (ESPHome, per room) ──────────┘        (per-room probability)
   └─► target X/Y coordinates ─► FloorCast ─► live dots on the floorplan
```

Layered approach, each layer useful on its own:
1. **Bermuda BLE trilateration** (HACS) — which room each *person's phone* is in. Works with any ESPHome bluetooth_proxy; Home Assistant Voice PE units work out of the box.
2. **Area Occupancy Detection** (HACS, Hankanman) — Bayesian fusion of BLE presence + media players + cameras + anything else into a per-room occupancy probability.
3. **LD2450 mmWave nodes** — 24GHz radar that reports X/Y coordinates for up to 3 targets per room, built from a $5–7 module + a $5 ESP32.
4. **FloorCast** (HACS) — maps LD2450 target coordinates onto your floorplan for the "radar screen" view.
5. A simple **picture-elements dashboard** — per-room presence icons (blue = still, orange = walking) that works before the radar hardware even arrives.

## Bill of materials (6 rooms)

| Item | Qty | ~Price | Notes |
|---|---|---|---|
| [HLK-LD2450-style radar module](https://www.amazon.com/dp/B0GV3X75WF) | 6 | ~$8–10 ea (2/4-packs cheaper) | Must be LD2450 — the LD2410 has **no X/Y output**. LD2450-style clones (e.g. HLIH V1.1) work too |
| [ESP32-DevKitC-32 (USB-C)](https://www.amazon.com/dp/B0DSZBH9N9) | 6 | ~$5 ea in 6-packs | Any ESP32 with a free hardware UART works; 38-pin CP2102 boards (NodeMCU-32S style) are fine. On Windows, CP2102 boards need the Silicon Labs driver |
| [USB-to-TTL serial adapter (FTDI)](https://www.amazon.com/dp/B07BBPX8B8) | 1 | ~$10 | Optional, for debugging a silent radar (see Troubleshooting) |
| Dupont jumper wires, 5V USB chargers | — | spare-drawer | 4 wires + 1 charger per node |

BLE layer (optional but recommended): 2+ ESPHome bluetooth proxies placed at opposite ends of the house. Home Assistant Voice PE, ESP32 flashed with the ready-made `bluetooth-proxy` project, or any existing ESPHome node with BT enabled.

## Part 1 — BLE: who is in which room

1. Install **Bermuda BLE Trilateration** via HACS; restart.
2. On each phone: HA Companion app → Settings → Companion App → *BLE Transmitter* → enable (this broadcasts an iBeacon). Also enable the *Detected Activity* sensor (Android) for walking/still state.
3. Bermuda → Configure → Select Devices → pick the phone's **iBeacon** entry (a stable UUID), *not* the randomized-MAC entries of the same phone.
4. Assign each BLE proxy device to its HA Area. Bermuda now produces an `_area` sensor per phone.
5. Calibrate: Bermuda → Configure → Calibration. Stand 1m from a proxy, tune reference power until the distance sensor reads ≈1m; then tune attenuation at 3–5m.

Template sensors turn the area sensor into per-room binary sensors (see [`template_presence.yaml`](template_presence.yaml)): one `occupancy` sensor per room plus a `walking` sensor from the phone's activity.

## Part 2 — Area Occupancy Detection

Install **Area Occupancy Detection** via HACS. One config entry manages all areas. Per area, feed it:
- Motion: the BLE room sensor from Part 1 (and later, the radar's presence sensor)
- Media: the room's TV/speakers (`playing`, optionally `paused` as active states)
- Cameras: e.g. Frigate person-detection binary sensors
- Doors/windows, power plugs — whatever you have

You get `binary_sensor.<room>_occupancy_status` + a probability sensor per room.

**Gotcha from this build:** a game console's `media_player` can stick in `playing` forever and pin a room occupied. If a room never clears, check its media players' states first and remove the liar.

## Part 3 — LD2450 mmWave nodes

### Wiring (per node)

| LD2450 | ESP32 |
|---|---|
| 5V | 5V / VIN |
| GND | GND |
| TX | GPIO16 |
| RX | GPIO17 |

On the ESP32 dev board, 5V and GND are the last two pins of the row at the USB end (side by side).

TX/RX **crossed**. The LD2450's UART is 3.3V logic — direct connection, no level shifter. No data later = the pair is swapped; swapping is harmless, just fix it.

### Firmware

ESPHome has a **native `ld2450` component** — no external packages needed. Full config: [`esphome/presence-room.yaml`](esphome/presence-room.yaml). The essentials:

```yaml
logger:
  baud_rate: 0        # frees UART logging; good hygiene with high-baud radar

uart:
  id: uart_ld2450
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 256000   # LD2450 is fixed at 256000
  parity: NONE
  stop_bits: 1

ld2450:
  id: radar
  uart_id: uart_ld2450
```

Leave `Radar Bluetooth` off and turn `Multi Target Tracking` on.

Expose per-target `x`, `y`, `speed`, `distance` sensors (3 targets), the presence/moving/still binary sensors, a `multi_target` switch, and `presence_timeout`. Copy the file once per room, change only the device name.

Flash each board once over USB from the ESPHome dashboard; everything after that is OTA. Label the boards physically — six identical DevKitCs are indistinguishable once flashed.

### Build and flash

1. Install the **ESPHome** add-on in Home Assistant (or `pip install esphome`).
2. Add `wifi_ssid` and `wifi_password` to ESPHome's `secrets.yaml`.
3. Copy `esphome/presence-room.yaml` to `presence-<room>.yaml` and change `name`/`friendly_name`.
4. First flash over USB: connect the board, then in the ESPHome dashboard choose *Install* > *Plug into this computer* (or `esphome run presence-<room>.yaml`). If the upload fails to connect, hold the board's BOOT button while it starts.
5. After the board joins WiFi, Home Assistant offers it under *Settings > Devices & Services*. Select *Add* and assign it to the room's Area.
6. Later updates go over WiFi (OTA) from the dashboard.

### Troubleshooting a silent radar

The radar streams frames continuously once powered, so no data means a wiring or power problem. In this build the cause was a poor 5V contact, not swapped TX/RX.

1. Re-seat 5V and GND, and check continuity at both ends of each wire.
2. Loopback test the ESP32 UART: jumper GPIO16 to GPIO17 and confirm it echoes.
3. Test the radar alone with a USB-to-TTL adapter at 256000 baud 8N1: you should see frames beginning `AA FF 03 00` and ending `55 CC`.
4. If the radar works alone and the ESP32 UART loops back, the fault is in the wiring or power between them.

### Mounting

- Wall-mount at **1.4–1.8m**, radar face vertical, pointed into the room's main living zone. FOV ±60°, ~6m range.
- Avoid aiming at ceiling fans, swaying plants, or curtains near vents (ghost targets), and remember it sees *through drywall*.
- **Record per room:** mounting wall, height, and facing direction — FloorCast needs sensor position + angle to translate radar X/Y into house coordinates.
- Ghost-target cleanup: the LD2450's Bluetooth + the HLKRadarTool phone app lets you draw exclusion zones; the config exposes a switch to enable/disable the radar's BT.

## Part 4 — FloorCast: the radar view

1. Install **FloorCast** via HACS, add the integration.
2. Give it your floorplan image, trace your rooms.
3. Add each LD2450 node as a source: select its target sensors, set its position on the plan and facing angle (from your mounting notes).
4. Add the FloorCast card to a dashboard. Walk each room; if your dot moves mirror-image, the sensor's configured angle is 180° off.

## Part 5 — Simple dashboard (works without radar)

A `picture-elements` card over the floorplan image with two conditional icons per room — shown when the room's occupancy sensor is on, colored by the walking sensor:

```yaml
type: picture-elements
image: /local/floorplan.jpg
elements:
  - type: conditional
    conditions:
      - entity: binary_sensor.kitchen_occupancy_status
        state: "on"
      - entity: binary_sensor.person_walking
        state: "off"
    elements:
      - type: icon
        icon: mdi:account
        style: {left: "30%", top: "25%", color: "#03a9f4", "--mdc-icon-size": "36px"}
  # …repeat with mdi:walk / orange for the walking state, then per room
```

Floorplan image: draw one free at floorplanner.com, or scan your home with the CubiCasa app.

## Lessons learned

- **LD2410 ≠ LD2450.** Only the LD2450 outputs target coordinates. For the radar view, nothing else at this price point does.
- Pre-assembled LD2450 units (Apollo MTR-1, Everything Presence Lite) are great but only sold by maker stores; bare modules + ESP32s from mainstream retailers cost roughly half.
- BLE alone gets solid *room-level* presence for whoever carries a phone; mmWave adds position and covers phone-less guests. The layers are complementary, not redundant.
- Bayesian fusion (Area Occupancy Detection) beats any single sensor — and makes single-sensor failures (stuck media player) visible instead of fatal.

## License

MIT
