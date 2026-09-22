# EcoRoad ES6 BLE Controller — AI Handoff Document

## Project File

`index.html`  
Single self-contained HTML file (~4310 lines). Pure Vanilla HTML5, CSS3, and modern JavaScript. No build system or external bundlers. Open directly in Google Chrome or Edge.

---

## What This Is

A Web Bluetooth (WebBLE) mobile companion app and diagnostic tuning console for the **EcoRoad ES6** electric scooter.  
The Android companion APK (`com.ecoroad.es6`) was reverse-engineered via JADX to extract the proprietary BLE protocol, telemetry frames, and controller commands. All communication is executed entirely in-browser over Web Bluetooth.

---

## BLE Connection Specifications

| Property                 | Value                                                      |
| ------------------------ | ---------------------------------------------------------- |
| Service UUID             | `0000ff60-0000-1000-8000-00805f9b34fb`                     |
| Write (No Response) UUID | `0000ff62-0000-1000-8000-00805f9b34fb`                     |
| Notify/Read UUID         | `0000ff61-0000-1000-8000-00805f9b34fb`                     |
| Device Name Filters      | `ES6-US`, prefix `ES6`, prefix `ECOROAD`, prefix `EcoRoad` |

---

## Connection Lifecycle & Startup Query Pipeline

- The header button toggles the link: `Connect` ⇄ `Disconnect` (`toggleConnection()` → `connectScooter()` / `disconnectDevice()`).
- First connect calls `navigator.bluetooth.requestDevice()` (user gesture) and caches the device in `bluetoothDevice`; every later connect reuses it, so reconnects need no gesture.
- First connect and auto-reconnect share one post-connect path, `finishConnection()`: resolve service/characteristics → `startNotifications()` → update UI → fire the startup handshake.
- On unexpected link loss (`gattserverdisconnected` without a user disconnect), the app shows a toast and retries `gatt.connect()` every **3 seconds** until it succeeds or the user disconnects.
- **Startup query pipeline**: immediately after every GATT connect, `executeStartupHandshake()` fires `0x60 → 0x61 → 0x62 → 0x63` sequentially with `PARAM 0x02`, `DATA [0x00]`, and a **120 ms** gap between writes (query CAR status, SN, speed limit, light state). Responses arrive on the notify characteristic, are decoded by `handleTelemetryUpdate`, and appear in the BLE Packet Logger. The `0x62` reply (two DATA bytes) is additionally parsed into the Governor card's live **readback** line — see §1 below.

---

## Safety Confirmations (Destructive Actions)

`confirmDestructive(message, confirmLabel, title)` returns a `Promise<boolean>` and shows a styled, app-themed modal (`#confirm-overlay`) instead of the native `confirm()` dialog.

- **Safe default**: focus starts on `Cancel`; pressing `Escape` or tapping the backdrop cancels; only one confirmation can be pending at a time (a second call resolves `false`); page scroll is locked while open.
- **Gated actions** (must confirm before any packet is sent):
  1. `unlockMaxSpeed()` — writes byte `120` (uncapped) to gears 11 and 3
  2. `applySpeedLimit()` — only when the target is **≥ 50 km/h** (lowering the cap stays one-tap)
  3. `sendSpeedLimit()` — raw governor byte from the advanced tuner
  4. `sendMileageReset()` — clears the trip odometer (irreversible data loss)
  5. `sendSingleMileageReset()` — clears the single-mileage counter (irreversible)
  6. `sendLockAction(3)` — clears the anti-theft lock password
  7. `sendCustomCommand()` — arbitrary framed TX from the Command Lab (saved shortcuts route through it, so they are covered too)
  8. `sendCustomGear()` — injects an unvalidated raw gear byte
- **Deliberately NOT gated** (must stay prompt-free, especially while riding): `sendGear()` presets, the Adaptive Auto Shifter, cruise / e-brakes / TCS / start-mode, lock & PIN unlock, lights & RGB, unit & brightness changes, and the startup handshake.

---

## Packet Frame Architecture

All TX packets adhere strictly to the controller framing protocol (mirrors APK `BleDataFactoryEco.d()`):

```
[0x1A] [0xA1] [CMD] [PARAM] [LEN] [DATA...] [CRC_HI] [CRC_LO] [0x1F] [0xF1]
```

- **Header**: `1A A1` (fixed magic bytes)
- **CMD**: command byte (from `BleEcoConstant`)
- **PARAM**: sub-parameter byte (usually `0x02`, sometimes `0x05`)
- **LEN**: `len(DATA)` — payload byte count
- **DATA**: variable-length parameter payload
- **CRC**: CRC-16/ARC over `[CMD, PARAM, LEN, DATA...]` (swapped high/low to match controller endianness)
- **Footer**: `1F F1` (fixed magic bytes)

### Hardware CRC Algorithm (CRC-16/ARC, polynomial `0xA001`)

```javascript
function runHardwareCRC16(data) {
  let crc = 0xffff;
  for (let i = 0; i < data.length; i++) {
    crc ^= data[i] & 0xff;
    for (let j = 0; j < 8; j++) {
      const lsb = crc & 1;
      crc >>>= 1;
      if (lsb) crc ^= 0xa001;
    }
  }
  const swapped = ((crc & 0xff00) >> 8) | ((crc & 0x00ff) << 8);
  return new Uint8Array([(swapped >> 8) & 0xff, swapped & 0xff]);
}
```

---

## Confirmed Hardware Discoveries

### 1. Speed Governor Firmware Offset (-29)

The motor controller firmware applies a fixed `-29` offset to speed values sent in command `0x36`:

- Sending `55` $\rightarrow$ results in **26 km/h** ($55 - 29$)
- Sending `65` $\rightarrow$ results in **36 km/h** ($65 - 29$)
- Sending `84` $\rightarrow$ sets the true **55 km/h** speed limit ($84 - 29 = 55$)
- Sending `94` $\rightarrow$ sets **65 km/h** ($94 - 29 = 65$)
- Sending `120` $\rightarrow$ completely **uncaps top speed** (maximum duty cycle)

`convertKmhToWire()` in `index.html` therefore applies **`Byte = Speed + 29` unconditionally** for every target in the 15–64 km/h range (the offset is *not* conditional on reaching 50 km/h), and returns the uncapped byte `120` only for the slider's 65 "MAX" position:

| Target (km/h) | Wire byte sent | Hex    |
| ------------- | -------------- | ------ |
| 15            | 44             | `0x2C` |
| 25            | 54             | `0x36` |
| 35            | 64             | `0x40` |
| 55            | 84             | `0x54` |
| 65 (MAX)      | 120            | `0x78` |

**0x62 readback (`SpeedLimitBimt`)**: the APK's RX parser (`method.txt`, `case 98`) reads **two** DATA bytes — `new SpeedLimitBimt(bytes[5], bytes[6])` — from the response to our `0x62` handshake query (sibling cases `97`/`99` answer `0x61` SN and `0x63` light, matching our pipeline). The Governor card now renders this live in `#gov-readback` and disambiguates the byte order empirically: wire-limit bytes are always ≥ 44 (15 km/h + 29) while gear bytes are ≤ 14, so the smaller side identifies the gear.

- **gear-first** (green) → confirms the app's TX format `[gear, wire]`.
- **limit-first** (orange) → would contradict it; report this result before changing any TX code.

The TX format itself remains `[gear, wire]` with `LEN 0x02`: the command is named `APP_SPEED_LIMIT_GEAR_ECO` ("speed limit **gear**"), every writer in `index.html` sends two bytes (`applySpeedLimit`, `unlockMaxSpeed`, profiles), and the −29 table above was CRC-verified against that layout. A hypothetical `[wireSpeed, 0x01]` write would make the controller read the speed value as a *gear index* and has no support in the decompile.

### 2. 7-Segment Dashboard Hex Gear Profiles

The scooter's dashboard 7-segment display shows `(gearByte + 1)` in hexadecimal:

- **Bytes 0, 1, 2, 3**: Standard riding modes showing **1**, **2**, **3**, **4** on dash (Eco, Drive, Turbo, Sport)
- **Byte 9**: Dash displays **A** (Eco profile, 26 km/h)
- **Byte 10**: Dash displays **b** (Mid profile, 37 km/h)
- **Byte 11**: Dash displays **C** (Sport profile, 50 km/h 🔥)
- **Byte 14**: Dash displays **F** (Crawl mode, 16 km/h)
- Modes `A`, `b`, `C` provide direct secondary firmware speed profiles accessible via one-touch buttons.

**Gear label registry**: `index.html` keeps one `GEAR_REGISTRY` object keyed by the raw **gear byte** (0–3 = standard gears, 9 = A, 10 = b, 11 = C, 14 = F, plus extended bytes 4/7/8/12/13), with `getGearLabel(gearByte)` as the only label source. The telemetry decoder (`bytes[15] & 0x0f`), the `sendGear()` button badge, and the Adaptive Auto Shifter status text all read from it, so labels can never drift apart. Unknown bytes fall back to `Raw Gear (n)`.

### 3. Lights & RGB Sequence

- **Headlight ON**: Sends CMD `0x33` on `PARAM 0x05` with `LEN 0x01`, data `[0x00]` (`1A-A1-33-05-01-00-1E-F1-1F-F1`). The packet bytes are authoritative: CRC `1E F1` is the CRC-16/ARC of `33 05 01 00`, so the payload really is the single byte `0x00` (an earlier note claiming `[0x01, 0x01]` contradicted its own packet and has been corrected).
- **All Lights OFF**: Sends 6-byte zero array `[0, 0, 0, 0, 0, 0]` (`1A-A1-33-02-06-00-00-00-00-00-00-AD-D8-1F-F1`).
- **RGB Color Sequence**: Physical scooter firmware requires lights to be turned ON first before setting custom RGB color; the app automatically executes this sequence ("Arm & Apply RGB").

---

## RX Telemetry Decoders

The controller broadcasts two telemetry frames continuously (~200ms interval):

### Frame `0x20` — Meter1ECO (27 bytes)

Decoded in `handleTelemetryUpdate`:

- `[5]` bit 0 $\rightarrow$ Headlight active indicator
- `[8]` $\rightarrow$ Battery charge percentage (0–100%)
- `[9]` (low 8b) + `[13]` bits 0-1 (high 2b) $\rightarrow$ Live speed in 0.1 km/h: `((bytes[13] & 0x03) << 8 | bytes[9]) * 0.1`
- `[12]` $\rightarrow$ Active speed limit in km/h
- `[15]` bits 0-3 $\rightarrow$ Active gear byte:
  - `0` = Gear 1 (Eco · 26)
  - `1` = Gear 2 (Drive · 35)
  - `2` = Gear 3 (Turbo · 45)
  - `3` = Gear 4 (Sport · 50)
  - `9` = Mode A (26)
  - `10` = Mode b (37)
  - `11` = Mode C (50 🔥)
  - `14` = Mode F (16)
- `[15]` bit 3 $\rightarrow$ Speed unit (0 = km/h, 1 = mph)
- `[16]` bit 6 $\rightarrow$ Cruise control state (1 = Active, 0 = Off)
- `[16]` bit 7 $\rightarrow$ Throttle mode (0 = Zero-Start, 1 = Kick-Start)

### Frame `0x21` — Meter2ECO (25 bytes)

- `[5:6]` $\rightarrow$ Battery voltage in 0.1V (little-endian: `((bytes[6] << 8) | bytes[5]) * 0.1`)
- `[7:8]` $\rightarrow$ Battery discharge current in 0.01A (little-endian)
- `[9:10]` $\rightarrow$ **Phase current** in 0.01A (little-endian) — present in `method.txt`, not yet surfaced in the UI
- `[11:12]` $\rightarrow$ Speed in 0.1 km/h (little-endian) — redundant with Frame `0x20`, currently unused
- `[13:14]` $\rightarrow$ Total odometer in km (little-endian: `(bytes[14] << 8) | bytes[13]`)
- `[15]` $\rightarrow$ Controller temperature in °C
- `[16]` $\rightarrow$ field ×0.1 (unused)
- `[20]` $\rightarrow$ Active speed-gear bitmask (`SpeedGearsBRse`, reversed bit string) — unused; live gear comes from Frame `0x20[15]`

> ⚠️ **Notation gotcha**: the literals in `method.txt` are **decimal** — `32` = `0x20` (Meter1), `33` = `0x21` (Meter2), `48` = `0x30`, `97/98/99` = `0x61/0x62/0x63` (query responses), `102` = `0x66` (version frames). "Frame 33" and "Frame `0x33`" are *not* the same frame (`0x33` = 51 decimal = the TX headlight command).

---

## App UI Architecture (Mobile Companion App)

The interface is built as a native companion mobile app (`.app-shell` with max 440px width, responsive on all devices):

### 1. Sticky Header

- Model brand badge: **EcoRoad ES6 Pro Companion**
- Live Battery Pill: `🔋 --%` (synced to live telemetry)
- Connection status badge: `● Live` / `● Disc.`
- Quick action: `Connect` ⇄ `Disconnect` (header button toggles the link)

### 2. Bottom Navigation Bar (4-Tab Layout)

#### 🏎️ Tab 1: Ride

- **Hero Speedometer**: SVG circular arc gauge (0–65 km/h range) with gradient stroke (Green $\rightarrow$ Cyan $\rightarrow$ Amber), dynamic `stroke-dashoffset` animation, large speed digits (`Outfit 56px`), and active gear badge. The gauge container was enlarged to 252×210 px (was 220×180) and the center text block moved to `top: 88px` so the digits no longer clip into the arc stroke.
- **Battery State of Charge**: Colored live progress bar (green $\rightarrow$ orange $\rightarrow$ red).
- **Sub-Metrics Tiles**: Voltage, Controller Temp, **live Power (W)**, Speed Governor, Cruise, and Odometer.
- **Riding Profiles** (compact card at the top of the Ride tab — slim header row instead of a full card-header, sequence details collapsed in a `<details>` drawer, status line hidden until a profile runs; 150 ms between writes):
  - `🚗 Commute · Legal 25` → `0x36` limit byte 54 on gear 1 → `sendGear(1)` → `sendStartSpeed(1)` (kick-start) → `sendTCS(1)`. No confirmation needed (it only *lowers* the cap).
  - `🏁 Private Track · Uncapped` → reuses the safety-gated `unlockMaxSpeed()` (returns `false` if the user declines, which aborts the rest of the profile) → `sendGear(11)` → `sendStartSpeed(1)` → `sendTCS(0)`.
  - Status pill: `Manual` / `APPLYING…` / profile name / `INCOMPLETE`.
- **Battery Health & Range card**: pack voltage (+ trip min–max voltage curve), current draw, SoC, remaining Wh (`Ah × V × SoC`), avg Wh/km, and `Est. Range = Remaining Wh ÷ Avg Wh/km`. **Pack capacity is a user-editable Ah input** (default 13 Ah — the actual pack size — persisted in `ecoroad-capacity-ah`). The estimate appears only after >0.2 km of riding.
- **Trip Computer card**: peak speed, average speed, ride time, distance, energy consumed (Wh), peak power. Integrates Frame 0x20 speed and Frame 0x21 `V×I` at telemetry rate (dt > 5 s gaps are discarded so a backgrounded tab can't skew stats), persists to `ecoroad-trip-stats` in localStorage (saved every ~5 s and on `pagehide`/`visibilitychange`), and has a **confirmation-gated** "↺ Reset Trip" button that touches local stats only — never the scooter's odometer.
- **Riding Modes**:
  - 4 Main Gears: 1 Eco (26), 2 Drive (35), 3 Turbo (45), 4 Sport (50).
  - Secret Letter Modes: Mode A (26), Mode b (37), Mode C (50 🔥).
  - Extended Modes Drawer: Modes 5, 8, 9, d, E, F, and custom byte injector.
- **Adaptive Auto Shifter**:
  - Virtual automatic transmission simulation.
  - Monitors live speed from Frame `0x20` and automatically upshifts as you reach current gear top speed:
    - **1 $\rightarrow$ 2**: at 15 km/h
    - **2 $\rightarrow$ 3**: at 25 km/h
    - **3 $\rightarrow$ 4 (or Mode C)**: at 35 km/h
  - Downshifts with hysteresis (11, 22, 32 km/h) to prevent gear hunting.
  - 1.4-second debounce cooldown prevents controller packet flooding.
- **Speed Governor**:
  - Target slider (15 to 65+ km/h).
  - "🚀 Set Speed Limit" (with `-29` offset calculation).
  - "⚡ Unlock Full 55+ km/h" button (sends uncapped byte `120`).
  - Quick Presets: 15, 25, 35, 55 km/h.
  - Raw Byte & Target Gear selector drawer.
- **Ride Dynamics**:
  - Zero-Start vs Kick-Start (`0x39`).
  - Cruise Control ON / OFF (`0x38`).
  - Electronic Braking (Soft / Medium / Strong) (`0x37`).
  - Traction Control TCS ON / OFF (`0x52`).

#### 💡 Tab 2: Lights

- Headlight toggle (ON / All OFF).
- RGB Underglow Deck Illumination:
  - Real-time color preview box & native OS color picker.
  - 8 quick preset swatches.
  - Individual R, G, B and brightness sliders.
  - Animation modes: Static, Breathe, Rainbow.
  - "🎨 Arm & Apply RGB" multi-packet sequence.

#### 🔒 Tab 3: Security

- Digital Anti-Theft Lock:
  - 4-digit PIN input with default `0000`.
  - Action buttons: Lock Scooter, Unlock with PIN, Reset/Clear Password, Unlock Raw Zeroes.
  - Real-time response decoder (handles error `0x89` invalid PIN).
  - Anti-theft wheel resistance overview.

#### ⚙️ Tab 4: Lab & Diagnostics

- **Cockpit & Units**: Metric (km/h) vs Imperial (mph), Gauge brightness (25–100%), Trip & Single mileage reset.
- **Hardware Diagnostics card**: reads the fault byte (Frame 0x20 `bytes[7]`, APK `FaultInfoEco`, `0x00` = OK) and controller temperature (Frame 0x21 `bytes[15]`). Renders a green **✓ Systems Normal** row when clean, or amber warning rows for a non-zero fault code and/or over-temperature at ≥ 80 °C. Note: `bytes[10]`/`bytes[11]` are *speed* fields per `method.txt`, not fault bits — do not read a bitmask there.
- **Custom Command Builder**: Raw CMD, PARAM, and DATA hex inputs with shortcut save/load/delete.
- **Packet Traffic Logger**: Real-time timestamped BLE TX/RX inspector with one-click copy, clear, and heartbeat silence filter.

---

## Tech Stack

- Pure HTML5 + Vanilla JS + CSS3
- Web Bluetooth API (`navigator.bluetooth`)
- Google Fonts: Inter & Outfit
- Runs in Google Chrome or Edge on Windows, Mac, Android, and Linux.
