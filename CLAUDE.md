# DIVE COMPUTER COMPANION — Claude Code Context

## Project

DIY dive computer companion PWA hosted at **depthcompanion.com**. Communicates with an ESP32-based dive computer over Web Bluetooth (Nordic UART profile). Tested in Chrome on HTTPS only (Web Bluetooth requires secure context).

## Architecture

**Single `index.html` file** — all HTML, CSS, and JS inline. No build step, no dependencies, no npm, no bundler. Keep it that way unless the user explicitly asks to split.

Fonts loaded from Google Fonts CDN: `Share Tech Mono` (monospace displays) and `Barlow Condensed` (UI chrome).

## Theme

Deep navy palette via CSS custom properties:

| Variable | Value | Role |
|---|---|---|
| `--bg` | `#011a3d` | Page background |
| `--bg2` | `#022a5e` | Card/panel background |
| `--bg3` | `#023e8a` | Input/slider track background |
| `--border` | `#0077b6` | All borders |
| `--cyan` | `#90e0ef` | Primary accent, active values |
| `--text` | `#caf0f8` | Default text |
| `--dim` | `#90e0ef` | Secondary text (same as cyan) |
| `--muted` | `#0077b6` | Muted labels (same as border) |
| `--green` | `#3dffa0` | Success / NDL OK / connected |
| `--yellow` | `#ffd060` | Warning / active state / scanning |
| `--red` | `#ff5a4a` | Error / DECO / recording |
| `--ocean` | `#0077b6` | Logo accent |

Header title: `DIVE` (`.logo-dive`, `--text`), `COMPUTER` (`.logo-computer`, `--cyan`), `COMPANION` (`.logo-companion`, `--ocean`).

Inline styles and `<style>` block both used — match what's already there for the element you're editing.

## App Structure — Three Tabs

The default active tab is **LOGS** (`panel-logs` has `.active` on page load).

### LOGS Tab (`#panel-logs`)

- **Dive list** (`#diveList`) — cards rendered newest-first by `renderDiveList()`. Empty state shows `#diveEmpty`.
- **Dive count** (`#diveCount`) — e.g. "3 dives".
- Clicking a dive card calls `openDetail(dive)` → opens the **detail overlay**.

**Detail overlay** (`#detailOverlay`, `position:fixed`) — full-screen overlay with:
- Back button → `closeDetail()`
- Stats grid (`#detailStats`) — 6 cells: Duration, Max Depth, Avg Depth, GF Low, GF High, Samples
- Depth profile canvas (`#profileCanvas`) drawn by `drawProfile(dive)`
- Sample table (`#sampleBody`) — one row per sample (time, depth, GF)
- CSV download button → `downloadDive()`

### SIM Tab (`#panel-sim`)

- **Recording pill** (`#recPill`) — shows `REC · SIM · MM:SS` or `REC · LIVE · MM:SS` when a dive is active.
- **Depth display** (`#depthValue`) — large 96px number, color-coded: default cyan, `warning` (yellow) ≥20m, `danger` (red) ≥40m.
- **Depth Override card**:
  - Slider `#depthSlider` (0–60m, step 0.5) → `oninput="onDepthSlider(this.value)"`
  - Readout `#depthReadout`
  - SEND DEPTH button (`#sendBtn`) → `sendDepth()`
  - USE SENSOR button (`#sensorBtn`) → `sendSensor()`
  - Status bar `#depthStatus`
- **Live from Device panel** (`#livePanel`, hidden until connected):
  - Shows: `#liveDepth`, `#liveMaxDepth`, `#liveNDL`, `#liveCeiling`, `#liveElapsed`, `#liveDecoFlag`
  - `#liveDecoFlag` shows `NDL OK` (green) or `DECO` (red, blinking)
- **Sim Depth Profile graph** (`#simProfileCanvas`) — canvas drawn by `drawSimProfile()`, records via `simProfileRecord()`.

### SETTINGS Tab (`#panel-settings`)

**Gradient Factors card**:
- GF Low slider `#gfLowSlider` (10–100, step 5) + readout `#gfLowReadout`
- GF High slider `#gfHighSlider` + readout `#gfHighReadout`
- Validation: gfLow is clamped ≤ gfHigh in `onGFSlider()`
- Preset buttons: 100/100, 85/85, 70/85, 40/85, 30/70 → `setGFPreset(lo, hi)`
- SEND GF TO DEVICE (`#sendGFBtn`) → `sendGF()` → sends `GF:lo/hi`
- Status bar `#gfStatus`

**Device card** (single card containing all device rows):
- **DEVICE NAME** — `#devNameVal` shows current name, RENAME button `#sendNameBtn` → `openNameEdit()`
  - Inline edit row `#nameEditRow` (hidden by default): input `#nameInput`, SEND `#sendNameConfirm` → `sendName()`, ✕ → `closeNameEdit()`
  - `onNameInput()` gates SEND on non-empty + connected
  - `nameInput.dataset.userEdited` flag prevents firmware STATUS from clobbering in-progress edits
- **FIRMWARE** — `#fwVer` (populated from `d.fw` in STATUS), UPDATE button `#fwCheckBtn` → `checkFW()` (stub, no OTA yet)
- **TIME SYNC** — live clock `#localClock` (ticks every 1s from `startClock()`), `#syncedCheck` (✓ shown after sync), time input `#timeInput` (type=time), SYNC button `#timeSyncBtn` → `doTimeSync()` → `syncTime()` → sends `TIME:HH:MM:SS`
- **ROTATION** — `#rotateVal` (NORMAL/FLIPPED), FLIP button `#rotateBtn` → `onRotateClick()` → sends `ROTATE:0` or `ROTATE:2`. Firmware uses TFT_eSPI rotation 1=normal, 3=flipped; `d.rotation` in STATUS reflects the firmware value.
- **SCREEN** — `#screenVal` (DIVE/TIME), SLEEP/WAKE button `#screenBtn` → `onScreenClick()` → sends `SCREEN:DIVE` or `SCREEN:TIME`. Button label is the *action* (SLEEP when current=DIVE, WAKE when current=TIME).
- **BRIGHTNESS** — slider `#brightSlider` (0–100, step 5), readout `#brightVal`; 50ms throttle on drag + guaranteed final send on release.

**Debug Log card** — collapsed by default, `toggleDebug()` toggles `.open` on `#debugLog`. Entries added by `dbgLog(type, msg)` — types: `ok` (green), `warn` (yellow), `err` (red), `info` (cyan).

## BLE Protocol

**Nordic UART Service (NUS)**:
- Service UUID: `6e400001-b5a3-f393-e0a9-e50e24dcca9e`
- RX (app→device): `6e400002-b5a3-f393-e0a9-e50e24dcca9e`
- TX (device→app, notifications): `6e400003-b5a3-f393-e0a9-e50e24dcca9e`

**Advertisement filter**: `{name: 'DiveComputer'}` — hardcoded on device, never changes.

**Commands sent to device** (via `send(str)` → `rxChar.writeValueWithResponse`):

| Command | Description |
|---|---|
| `STATUS` | Request full status packet |
| `DEPTH:x.x` | Override depth (metres, one decimal) |
| `DEPTH:SENSOR` | Return to physical pressure sensor |
| `GF:lo/hi` | Set gradient factors (slash separator) |
| `TIME:HH:MM:SS` | Sync RTC |
| `NAME:str` | Rename device (max 20 chars) |
| `BRIGHTNESS:n` | Set brightness 0–100 |
| `ROTATE:n` | `0` = normal, `2` = flipped |
| `SCREEN:DIVE` | Full dive UI |
| `SCREEN:TIME` | Sleep clock mode |

**Status packet from device** (JSON, received via TX notifications):

```json
{
  "depth": 12.3,
  "ndl": 45,
  "ceiling": 0.0,
  "deco": false,
  "elapsed": 180,
  "maxDepth": 15.0,
  "gfLow": 40,
  "gfHigh": 85,
  "fw": "1.0.0",
  "deviceName": "DIY DIVE",
  "brightness": 80,
  "rotation": 1,
  "screen": "DIVE"
}
```

Parsed by `onNotify(event)`. Any field may be absent — all checks use `!== undefined`.

## Key State

```js
// BLE
let device, rxChar, txChar, connected;

// Dive/sim
let currentDepth;   // metres, float
let gfLow, gfHigh; // 10–100
let isSim;          // true = depth overridden by slider; false = sensor mode

// Dive recording (session memory only — never localStorage)
let dives;          // completed dive objects
let activeDive;     // { num, mode, startTs, startDate, samples, maxDepth, currentDepth, gfLow, gfHigh }
let recTimer;       // setInterval for rec pill display
let sampleInterval; // setInterval every 5s

// Detail view
let viewingDive;

// Device settings (mirrors what firmware reports)
let currentRotation; // 1 = normal, 3 = flipped (firmware values)
let currentScreen;   // 'DIVE' | 'TIME'

// BLE depth throttle
let bleDepthBusy, bleDepthPending, bleDepthLastSent;
const BLE_DEPTH_MIN_INTERVAL = 150; // ms

// Brightness throttle
let brightSendTimer, brightLastSent;

// Sim profile graph
const SIM_PROFILE = [];  // {t: seconds, d: metres}
let simProfileStart;     // performance.now() at first point
let simGraphTimer;       // 1s tick

// STATUS poll
let liveTimer;           // 2s interval, sends 'STATUS'
```

## Dive Recording

- **Threshold**: `DIVE_THRESHOLD = 0.5` metres
- `checkDiveRecording()` called on every depth change (slider or incoming STATUS when `!isSim`)
- Samples recorded every **5 seconds** via `sampleInterval`; t=0 sample captured immediately at dive start
- Completed dive object: `{ num, mode ('sim'|'live'), date, durationSec, maxDepth, avgDepth, gfLow, gfHigh, samples:[{t, depth, gfLow, gfHigh}] }`
- **Session memory only** — `dives[]` lives in JS, never touches localStorage
- CSV export via `downloadDive()` → `dive_001_sim.csv` format

## Key Functions

| Function | Purpose |
|---|---|
| `toggleBLE()` | Connect/disconnect; on connect: syncs time, sends STATUS, starts live poll |
| `onDisconnect()` | Cleans up state, resets UI, stops live poll |
| `send(str)` | Raw BLE write, logs to debug, returns bool |
| `onDepthSlider(val)` | Updates display + dive recording + queues BLE send |
| `scheduleBLEDepth()` | Computes wait time from last send, fires `sendDepthBLE` |
| `sendDepthBLE(depth)` | Async BLE send with busy-flag + pending flush |
| `sendDepth()` | SEND DEPTH button — immediate send of current slider value |
| `sendSensor()` | USE SENSOR button — sends `DEPTH:SENSOR` |
| `onGFSlider()` | Reads sliders, enforces lo≤hi, updates readouts |
| `setGFPreset(lo,hi)` | Sets sliders programmatically |
| `sendGF()` | Sends `GF:lo/hi` |
| `syncTime()` | Uses manual input or current time, sends `TIME:HH:MM:SS` |
| `onBrightInput()` | 50ms throttle during drag |
| `onBrightChange()` | Guaranteed final send on release |
| `onRotateClick()` | Toggles rotation, sends `ROTATE:0` or `ROTATE:2` |
| `onScreenClick()` | Toggles screen mode, sends `SCREEN:DIVE` or `SCREEN:TIME` |
| `sendName()` | Sends `NAME:str`, mirrors to `#devNameVal` |
| `dbgLog(type, msg)` | Prepends entry to `#debugLog` |
| `checkDiveRecording()` | start/end dive based on depth threshold |
| `startDive()` | Initializes `activeDive`, starts timers, records t=0 sample |
| `endDive()` | Finalizes dive, pushes to `dives[]`, calls `renderDiveList()` |
| `renderDiveList()` | Rebuilds `#diveList` DOM, newest-first |
| `openDetail(dive)` | Populates and shows detail overlay |
| `drawProfile(dive)` | Canvas depth profile in detail overlay |
| `onNotify(event)` | Parses JSON STATUS packet, updates all live fields |
| `startLiveData()` | Begins 2s STATUS poll, shows `#livePanel` |
| `stopLiveData()` | Clears poll, resets live cells to `—` |
| `simProfileRecord()` | Records point to `SIM_PROFILE`, starts 1s ticker if first call |
| `drawSimProfile()` | Full canvas redraw of sim depth graph (fixed 440×160 logical px) |
| `switchTab(name)` | Toggles `.active` on panels and tab buttons |
| `toast(msg, type)` | 2.5s toast notification (success/error/plain) |
| `startClock()` | 1s tick updating `#localClock` |

## Key Constraints — Do Not Break

- **Dive logs are session memory only.** `dives[]` is never written to localStorage.
- **localStorage** is acceptable only for UI preference flags (e.g. advanced mode toggles).
- **Depth slider sends on every `oninput`** — no debounce on the slider itself. The 150ms throttle lives in the BLE layer (`scheduleBLEDepth` / `sendDepthBLE`).
- **BLE advertisement name `'DiveComputer'` is hardcoded on the device** — never change the filter in `requestDevice`.
- **Brightness sends both during drag (50ms throttle) and on release** — don't collapse these to one path.
- **`nameInput.dataset.userEdited`** prevents firmware STATUS from overwriting a name the user is actively typing.

## Styling Conventions

- Use `Share Tech Mono` for all numeric readouts, labels, monospace UI.
- Use `Barlow Condensed` for buttons and body text.
- Match existing style pattern for the element type — don't introduce new CSS patterns unless needed.
- Single `index.html` — keep everything inline unless user explicitly asks to split.
