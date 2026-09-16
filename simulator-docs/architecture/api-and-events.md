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

## 4. Biometric REST surface — API v2 (Phase 6–7, additive)

Nothing above changes. These endpoints are **new** and only active when a matcher/stations are configured (otherwise 503 — NFR-12). **Implemented 2026-09-16 (Phase 6–7).**

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/biometric/status` | Matcher reachable? enrolled count? threshold/separation/sample config — no template data |
| POST | `/api/biometric/capture` | **Raw image body** (octet-stream: JPEG/PNG/BMP) → sidecar extraction + measurement → quality verdict; reject = 409 + actionable reason (`too-dark` / `too-blurry` / `partial-finger`); accept = `{templateB64, quality, score}` |
| POST | `/api/biometric/enroll` | `{ userId, finger, consent, samples: [templateB64 × N] }` → N-sample ceremony (every pair must clear the accept threshold on the engine's consistency check), then the encrypted triple-write; single-template form (`templateB64`) = the import station |
| POST | `/api/biometric/identify` | Probe template → `{decision: match\|no-match\|ambiguous\|not-enrolled, uid?, userId?, score, runnerUpScore?}` — **decision only, no side effects** |
| POST | `/api/biometric/verify` | `{ userId, finger, templateB64 }` → 1:1 `{decision, score}` |
| POST | `/api/biometric/punch` | **The punch controller** (biometric-core §2): identify → on match, `AttendanceRecord` (verifyType=1) through the same store/event-bus path as `/api/punch` → device face + EF_ATTLOG + BITS; `no-match`/`ambiguous` → fail, no record; sidecar down → 503, never auto-accept |
| POST | `/api/biometric/remove` | Delete one enrollment (`{userId, finger}`) — encrypted copy + wire copy + matcher |
| POST | `/api/biometric/purge` | Full wipe — requires `{confirm: true}`; removes ciphertext, matcher table, and FR-8 ISO wire copies |

Documented deviations from the pre-implementation draft (2026-09-16):
- **Capture body is raw octet-stream**, not multipart — one code path, consistent with the rest of the surface.
- **`/identify` stays side-effect-free**; the record-creation composition lives in `/punch` (the punch controller), keeping decision and write separable.
- **`/capture` returns the just-extracted template to the presenting client.** The stateless ceremony (page holds the samples) needs it. This is a narrow, documented exception to the "no template bytes in API responses" rule: only the client that just presented the finger receives that finger's template, nothing is logged, and the wire is the same untrusted LAN HTTP as the ZK protocol itself (FR-8 carries templates in plaintext too). Server-held ceremony state would remove the round-trip — a Phase-8 hardening option (future-work).

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
