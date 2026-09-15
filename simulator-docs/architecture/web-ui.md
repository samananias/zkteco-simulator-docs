# Web UI Design — "Device Face"

Goal per brief §5: a viewer should think *hardware*, not *website*. This is a deliberate kiosk/embedded-hardware aesthetic — explicitly **not** a SaaS dashboard. Vanilla HTML/CSS/JS, no framework (ADR-007).

## 1. Composition

```text
┌──────────────────────────────┐
│  bezel (dark rounded frame,  │   ← subtle metallic gradient, brand text
│  speaker grille + keypad     │
│  graphic below the screen)   │
│  ┌────────────────────────┐  │
│  │ status bar  ▂▄▆ ✱ 🔒   │  │   ← monochrome glyphs: network, alarm, lock
│  │                        │  │
│  │      07:42:15          │  │   ← large bold digits, device time
│  │     2026-09-15 Tue     │  │
│  │   ACME Corp — Device 1 │  │   ← device name from options store
│  │  "Please place finger" │  │   ← rotating prompt
│  │                        │  │
│  │  (verify overlay)      │  │   ← PASS: green check + name + "Thank you!"
│  │                        │  │      FAIL: red X + "Please try again"
│  └────────────────────────┘  │
│  [1][2][3] [MENU] [ESC] [OK] │   ← decorative keypad (light click animation)
└──────────────────────────────┘
```

## 2. Screens / states

| State | Trigger | Behavior |
|---|---|---|
| `idle` | boot, after verify | Clock ticks from **device time** (372-day codec), prompt cycles |
| `verify-pass` | punch with known user (REST/WS/or backend-visible record) | Overlay 2 s: green ✓, name + userId, "Thank you!" → idle |
| `verify-fail` | punch with unknown user or forced fail | Overlay 2 s: red ✗, "Please try again" → idle |
| `disabled` | `CMD_DISABLEDEVICE` from backend | Banner "Device disabled by admin" |
| `menu` | tap MENU (or keypad) | Icon-grid skeleton (User Mgmt, Attendance Search, Data Mgmt, System Settings) — visual only in v1 |
| `lcd` | `CMD_WRITE_LCD`/`CLEAR_LCD` | Backend-pushed text replaces the prompt line |

## 3. Live sync protocol (WebSocket, JSON)

Server → client:
```json
{ "type": "state",    "state": { "enabled": true, "deviceName": "...", "deviceTime": "ISO" } }
{ "type": "verify",   "result": "pass" | "fail", "user": { "name": "...", "userId": "3" }, "verifyType": 1, "at": "ISO" }
{ "type": "punch",    "record": { "deviceUserId": "3", "state": 0, "at": "ISO" } }
{ "type": "users",    "users": [ ... ] }          // after backend writes
{ "type": "lcd",      "lines": ["...", "..."] }
```
Client → server: `{ "type": "punch", "userId": "3", "verifyType": 1, "state": 0, "forceFail": false }` and `{ "type": "subscribe" }`.
Reconnect with exponential backoff; on connect the client pulls a full REST snapshot (`/api/device/state`, `/api/users`).

## 4. Typography / color cues (from brief §5)

- Idle: dark navy background, white/cyan text, bold segmented-feel sans-serif digits.
- Status glyphs: small monochrome shapes, top-right.
- Verify overlays: high-contrast full-screen green/red — 2 s auto-revert.
- Utilitarian, slightly "early-2010s touchscreen kiosk" — square icon tiles in the menu, minimal shadows, no modern glassmorphism.
- The bezel + keypad graphic is the single strongest "this is a device" cue — do not ship a borderless page.

## 5. Biometric screens (Phase 6–7 — additive pages, same device aesthetic)

The device face above is **unchanged**; the biometric UI is a separate set of pages so the kiosk illusion is never diluted by admin chrome.

| Page | Purpose | Notes |
|---|---|---|
| `/enroll` | Employee enrollment via the camera station | Consent statement first, framing overlay, per-sample quality verdict with actionable reasons, progress (sample 2 of 3), success confirmation |
| `/scan` | Punch / identify station | Big "Place finger" guidance, capture → verdict; on pass shows the employee name (mirrors the device's green ✓); on fail shows the reason and offers retry |
| `/biometric` | Admin view | Enrolled employees (metadata only), template counts, station/matcher status, purge action with explicit confirmation |

Rules carried from ADR-011: no fingerprint image is ever displayed after capture (not even to the enrollee), no template bytes in the UI or DOM, and the consent text is not dismissible-without-reading for enrollment. On `identify` the device face's `verify-pass`/`verify-fail` states are triggered by the **real** matcher verdict — the same WebSocket messages as before, so no UI rework is needed (Phase 6 reuses this file's §2 state table verbatim).

If the matcher is unreachable, `/scan` shows a distinct "identification service unavailable" state — deliberately not the red "try again" fail overlay, because those mean different things (research Q: never confuse "I don't know you" with "I can't check right now").

## 6. Accessibility & practical notes

- The UI is a prop: auto-refresh clock, no interactive focus traps; keyboard shortcuts (1–9, M, Esc, Enter) mirror the keypad for screencasts.
- A small "SIMULATOR" watermark in a corner of the bezel (not the screen) keeps portfolio screenshots honest without breaking immersion.
