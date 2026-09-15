# API & Events

## 1. HTTP surface (port from config, default 3000)

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Device-face UI (static) |
| GET | `/api/device/state` | Device snapshot: enabled, deviceName, deviceTime (ISO), counts (users/logs), lcd |
| POST | `/api/punch` | `{ userId, verifyType? 0/1/2, state? 0-5, forceFail? }` → creates attendance record, emits verify + EF_ATTLOG (if registered) |
| GET | `/api/users` | Users in device store |
| GET | `/api/attendance` | Attendance records (desc) |
| POST | `/api/lcd` | `{ lines: string[] }` mirror of `CMD_WRITE_LCD` |
| POST | `/api/reset` | Restore seed state (demo convenience) |

No auth on these endpoints by design — LAN demo tool (see `../operations/security.md`). Binding defaults to localhost + LAN interface, not 0.0.0.0-exposed cloud hosts.

## 2. WebSocket

Single namespace; messages documented in `web-ui.md` §3. Server pushes `state` on every store change (debounced), `verify`/`punch` on punches, `users` after any user mutation (protocol or REST), `lcd` on LCD commands.

## 3. Internal event bus

```text
DeviceStore.onChange(StoreEvent)
   ├─► protocol engine  → EF_ATTLOG frames to registered sessions (queued if in-flight)
   └─► web/ws           → JSON fan-out to UI clients
```
Event types: `punch`, `user-upserted`, `user-deleted`, `attendance-cleared`, `template-changed`, `lcd-changed`, `state-changed`, `option-changed`.
Rule: the store never knows about TCP or WS; subscribers adapt. This keeps the engine testable without sockets.

## 4. Biometric REST surface — API v2 (Phase 6+, additive)

Nothing above changes. These endpoints are **new** and only active when a matcher/stations are configured (otherwise 503 — NFR-12).

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/biometric/status` | Matcher reachable? stations available? enrolled count? — safe, no template data |
| POST | `/api/biometric/capture` | Multipart frame (camera station) → quality verdict + reasons, or a one-shot probe template handle |
| POST | `/api/biometric/enroll` | `{ uid, finger, samples }` → runs the N-sample ceremony, stores encrypted template; returns quality + audit info |
| POST | `/api/biometric/identify` | Probe (or capture handle) → ranked candidates + verdict; **accepted identify also creates the attendance record** (the substitute's core operation) |
| POST | `/api/biometric/verify` | `{ uid, finger, probe }` → 1:1 score/verdict |
| POST | `/api/biometric/purge` | Delete all templates for an employee (or `{}` for a full wipe — explicit confirmation flag required) |

Response rule (ADR-011): template bytes are **never** returned — only metadata, scores, verdicts and reasons.

### Sequence — the substitute's headline demo (Phase 7)

```text
📱 /scan page ── camera frame ──► POST /api/biometric/capture   (quality gate)
                                        │
                                        ▼ probe template
                                  POST /api/biometric/identify ──► matcher sidecar (1:N)
                                        │
                    match accepted? ────┼────────────────────────► no → fail screen, no record
                                        │ yes
                                        ▼
                               store.append(AttendanceRecord)   (verifyType = 1)
                                        ├─► WS verify(pass, "Maria Santos") → device screen green ✓
                                        └─► (session registered → EF_ATTLOG)
BITS syncScheduler (≤ 30 s) ─► getAttendances() ─► ingest → HR dashboard shows a genuinely matched punch
```

## 5. Sequence — the headline demo

```text
Web UI "scan" ─► POST /api/punch ─► store.append(AttendanceRecord)
                                     ├─► WS verify(pass, "Maria Santos")  → device screen shows green ✓
                                     └─► (session registered? → EF_ATTLOG frame)
BITS syncScheduler (≤30 s) ─► getAttendances() ─► CMD_DATA_WRRQ ─► CMD_DATA burst
                             ─► ingest into Postgres ─► HR dashboard shows the punch
```
With realtime: a `getRealTimeLogs`-style client receives the same punch as an `EF_ATTLOG` event within milliseconds — shown in Phase 3 tests.
