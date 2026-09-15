# Functional Requirements

**v1.0 — Phase 0.** Derived from the original brief (`00-original-brief.md`) and revised per `../research/feasibility-report.md`. Item-by-item mapping: `traceability.md`.

**Global acceptance criterion:** unless stated otherwise, every requirement is validated by running the *unmodified* pinned oracle — `node-zklib@1.3.0` (patched) as used by `C:/bits/backend/src/shared/lib/zk-driver.ts` — against the simulator.

---

## FR-1 — Protocol engine (TCP :4370)
Raw TCP server (Node `net`) speaking the ZKTeco standalone binary protocol. Frame = magic `50 50 82 7D` + u32-LE payload size + 8-byte header (command u16, checksum u16, session-id u16, reply-number u16, all LE) + data. Handles TCP stream fragmentation/reassembly. Lenient inbound checksum (accept oracle and classic variants, log mismatches); outbound checksum configurable (`classic` default, `oracle` variant available).

## FR-2 — Session lifecycle
`CMD_CONNECT` (1000) → reply `CMD_ACK_OK` (2000) carrying a newly assigned session ID in the session field. `CMD_EXIT` (1001) → ACK + session teardown. Sessions must tolerate the backend's pattern of **frequent short-lived connections** (connect/disconnect per operation batch) and parallel sessions. Comm-key auth optional via config (default **off** — backend uses none): when on, first connect replies `CMD_ACK_UNAUTH` (2005) and `CMD_AUTH` (1102) is verified with the makeCommKey algorithm before the session opens.

## FR-3 — Device information
- `CMD_GET_VERSION` (1100) → firmware string, e.g. `Ver 6.81  Apr 28 2015\0`.
- `CMD_GET_FREE_SIZES` (50) → reply buffer ≥ 76 B; u32-LE `userCounts`@24, `logCounts`@40, `logCapacity`@72 (offsets into the buffer *after* TCP-header strip — these exact offsets are what `getInfo()` reads).
- `CMD_OPTIONS_RRQ` (11) → key lookup in a device-parameter store (`~SerialNumber`, `~DeviceName`, `~Platform`, `~OS`, `~ZKFPVersion`, `~PIN2Width`, `FingerFunOn`, `FaceFunOn`, …); reply `CMD_ACK_OK` + `value\0`.
- `CMD_OPTIONS_WRQ` (12) → store/ACK (`SDKBuild=1\0` arrives right after connect).

## FR-4 — Device clock (372-day-year codec)
`CMD_GET_TIME` (201) → u32-LE; `CMD_SET_TIME` (202) → parse u32-LE. Encoding (matches the BITS driver and node-zklib exactly):
`seconds = ((year-2000)*12*31 + (month-1)*31 + (day-1)) * 86400 + (hour*60+minute)*60 + second`.
Decode reverses with 31-day months and 12-month years. Round-trip validated against values packed by the driver.

## FR-5 — Device state & housekeeping
`CMD_ENABLEDEVICE` (1002) / `CMD_DISABLEDEVICE` (1003, payload `00 00 00 00`) toggle device state (shown on web UI; reads are allowed regardless). `CMD_REFRESHDATA` (1013), `CMD_FREE_DATA` (1502) → always `CMD_ACK_OK` (called constantly around bulk reads). Cosmetic ACKs: `CMD_RESTART`/`CMD_POWEROFF`/`CMD_SLEEP`/`CMD_RESUME`, `CMD_TESTVOICE`, `CMD_CANCELCAPTURE` (62).

## FR-6 — User management
- **Read:** reply to `CMD_DATA_WRRQ` (1503) with the users payload (`01 09 00 05 00 00 00 00 00 00 00`) by sending a **direct `CMD_DATA` (1501) burst**: frames containing `[u32-LE total byte size][72-byte user records…]`, sent back-to-back. Also accept pyzk-style `CMD_DB_RRQ` (7).
- **User record (72 B):** uid u16@0 · role u8@2 (0 user / 14 admin) · password 8 B@3 · name 24 B@11 (NUL-terminated) · cardno u32@35 · rest reserved · userId (visible PIN) 9 B ascii@48.
- **Write:** `CMD_USER_WRQ` (8) — parse the 72-byte layout exactly as the BITS driver builds it; also tolerate pyzk's 73-byte tagged variant (leading `0x02` byte). Stored users are visible in subsequent reads and the web UI.
- **Delete:** `CMD_DELETE_USER` (18, payload uid u16-LE); `CMD_DELETE_USERTEMP` (19, payload uid u16 + finger u8); `CMD_DEL_FPTMP` (134).
- **Clear:** `CMD_CLEAR_DATA` (14, payload type u8: 1=attlog, 2=templates, 5=users).

## FR-7 — Attendance log
- **Read:** reply to `CMD_DATA_WRRQ` (`01 0d 00 00 00 00 00 00 00 00 00`) with direct `CMD_DATA` burst: `[u32-LE size][40-byte records…]`. Also accept `CMD_ATTLOG_RRQ` (13).
- **Record (40 B):** userSn u16@0 · deviceUserId 9 B ascii@2 · zeros 15 B@11 · verifyType u8@26 (0 password / 1 fingerprint / 2 card) · time u32-LE@27 (FR-4 codec) · state u8@31 (0 check-in / 1 check-out / 2 break-out / 3 break-in / 4 overtime-in / 5 overtime-out) · tail 8 B@32 = `00 00 00 00 FF 00 00 00`.
- **Clear:** `CMD_CLEAR_ATTLOG` (15).
- Backend mapping note: node-zklib 1.3.0 exposes records as `{ userSn, deviceUserId, recordTime, ip }`; the BITS layer maps `status = record.state || 0`.

## FR-8 — Fingerprint templates (placeholder blobs)
No real biometrics exist; blobs are synthetic but structurally plausible (≥ 500 B, `SS21`-style header) so downstream code paths behave realistically.
- **Read:** `CMD_USERTEMP_RRQ` (9, payload uid u16 + finger u8) → enrolled slot: single `CMD_DATA` frame, data = `[entrySize u16][uid u16][fid u8][flag u8=1][template bytes…]`; empty slot: `CMD_ACK_ERROR` (2001) with no data (the BITS driver probes finger counts this way).
- **Write:** client-initiated flow → `CMD_PREPARE_DATA` (1500, payload `[size u16][00 00]`) → `CMD_DATA` (blob) → `CMD_CHECKSUM_BUFFER` (119) → `CMD_TMP_WRITE` (87, payload `[uid u16][fid u8][flag u8][size u16]`) → `CMD_FREE_DATA`. The simulator stores the blob for that (uid, finger).
- **Enroll trigger:** `CMD_STARTENROLL` (61, payload `[userId 24 B ascii][finger i8][overwrite flag i8]`) → ACK (optionally emit an `EF_ENROLLFINGER` event sequence in a later phase).

## FR-9 — Realtime events
`CMD_REG_EVENT` (500, payload flags u32; bit0 `EF_ATTLOG`) enables events **per session**; `00000000` disables. Event frame: command field = 500, session field = event code (`EF_ATTLOG` = 1), reply number = 0; data = 32-B record: userId 9 B ascii@0 · zeros 15 B@9 · verifyType u16-LE@24 · time 6 B@26 (raw bytes `20YY MM DD HH MM SS`).
**Emission safety rules:** only to sessions that registered; events raised while a request/response exchange is in flight are queued and flushed when the socket goes quiet. (The BITS backend never registers → it never receives unsolicited packets.)

## FR-10 — Demo triggers
REST: `POST /api/punch` `{ userId, verifyType?, state? }` (creates an attendance record + screen verification event), `GET /api/device/state`, `POST /api/lcd`, `GET /api/users`, `GET /api/attendance`. CLI: `scripts/punch.mjs` for demos without the UI.

## FR-11 — Device-face web UI
Bezel-framed kiosk replica per brief §5: idle clock (device time), device name, status-bar glyphs; green pass / red fail verification screens; icon-grid menu skeleton; hardware keypad graphic. Live two-way WebSocket sync with engine state (punch from UI → protocol-visible immediately; backend writes → screen reflects). LCD commands (`CMD_WRITE_LCD` 66 / `CMD_CLEAR_LCD` 67) mirror text onto the screen.

## FR-12 — Persistence
In-memory store default. Optional SQLite (`better-sqlite3`) behind the `DeviceStore` interface for restart persistence; schema sketched in `../architecture/data-model.md`.

## FR-13 — Seed data
Named fake employees (uid/userId/role/card), a week of varied attendance (on-time / late / absent), and enrolled placeholder fingerprints. Seeding shares the FR-4 time codec.

## Explicitly out of scope (v1)
UDP transport, iFace-specific screens, access-control modules (timezones/groups/anti-passback), SMS/Mifare/operation-log commands. Unimplemented commands reply `CMD_ACK_UNKNOWN` (65535) — clients tolerate it; see `../status/known-limitations.md`.

