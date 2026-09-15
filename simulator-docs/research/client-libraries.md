# Client Libraries & Backend Analysis

**The validation oracle, pinned:** `node-zklib@1.3.0` **as installed in `C:/bits/backend/node_modules/`**, plus the repo's `patches/node-zklib+1.3.0.patch`, exercised exactly the way `C:/bits/backend/src/shared/lib/zk-driver.ts` uses it. All claims below were read from that artifact `[V]`.

---

## 1. node-zklib 1.3.0 internals (oracle ground truth)

### 1.1 Version warning
npm's latest `node-zklib` is **1.3.0 (published 2020-02-14)**. GitHub `master` (caobo171) contains a materially different, **unpublished** rewrite (`PACKET_SIZES`, `decodeUserData72` differences, `isEventPacketTCP`, always-chunked reads). **Any documentation based on GitHub master does not describe our oracle.** This project validates against 1.3.0.

### 1.2 Record decoders (`utils.js`) — the byte layouts the oracle expects
- `decodeUserData72`: uid u16@0 · role u8@2 · password ascii 8 B@3 · name ascii@11 (NUL-terminated within the remainder) · cardno u32@35 · userId ascii 9 B@48.
- `decodeRecordData40`: userSn u16@0 · **deviceUserId** ascii 9 B@2 · recordTime = `parseTimeToDate(readUInt32LE(27))`. (Backend maps `status = record.state || 0` — the state byte is not decoded by 1.3.0.)
- `decodeRecordRealTimeLog52`: strips TCP prefix, skips the 8-B ZK header, userId ascii 9 B@0, attTime from bytes 26–31 as raw `2000+yy, mm, dd, hh, mm, ss`.
- `parseTimeToDate`: **372-day-year decode** — day = t%31+1, month = t%12, year = t/12+2000 `[V]`.
- `createChkSum`: `s %= 65535` per word; final `65535 − s − 1` (see `zk-protocol-notes.md` §2).

### 1.3 TCP behaviors (`zklibtcp.js`)
- **connect():** `CMD_CONNECT` → resolves if *any* reply arrives (1.3.0 does **not** check `CMD_ACK_OK` here); reads session ID from reply offset 4. Timeouts: 2 s for connect/exit writes, user timeout (30 s in the backend) otherwise. **No comm-key path exists in 1.3.0's connect** — do not enable `CMD_ACK_UNAUTH` for this client.
- **executeCmd():** increments replyId (0 reset on connect; `createTCPHeader` additionally pre-increments when writing the packet — sloppiness that is harmless because the device side echoes), resolves with the *raw* reply payload (header+data after TCP strip), never validates the reply code.
- **Reads (getUsers/getAttendances):** `freeData()` → `readWithBuffer(payload)` → `freeData()`. `readWithBuffer` sends `CMD_DATA_WRRQ` and:
  - **Direct path (primary):** if the reply command is `CMD_DATA`, waits for 1 s of socket silence, then parses the *accumulated* buffer as `[u32 size][records…]` (it slices off 4 bytes before iterating records). → **Our simulator replies with one back-to-back `CMD_DATA` burst containing the entire dataset.**
  - **Chunked path (optional, ADR-004):** reply `CMD_ACK_OK`/`CMD_PREPARE_DATA` with data-stat (`00` + u32 size @1), then client sends `CMD_DATA_RDY` per 65472-byte chunk; each chunk frame's data carries an 8-byte sub-header before chunk bytes `[A — validate in Phase 1 if chunked mode is enabled]`.
  - Event packets are skipped by `checkNotEventTCP` inside read accumulation — but **not** in `writeMessage` (single-packet commands): another reason events are gated per registration + queued in-flight (FR-9).
- **getInfo():** `CMD_GET_FREE_SIZES`; reads u32 at reply offsets 24 / 40 / 72 → reply buffer must be ≥ 76 B with userCounts@24, logCounts@40, logCapacity@72.
- **getRealTimeLogs():** sends `CMD_REG_EVENT` with `[01 00 00 00]`, then classifies any packet with command 500 + event-code field 1 as an event and calls `decodeRecordRealTimeLog52`. Requires event data ≥ 32 B in the layout of FR-9.
- **freeData()/disableDevice()/enableDevice()/clearAttendanceLog():** single `executeCmd` wrappers; `DISABLEDEVICE` carries `[00 00 00 00]`.
- **Patch (`patches/node-zklib+1.3.0.patch`):** adds `return` after `reject()` in two error paths (prevents double-settled promises). Zero protocol impact `[V]`.

---

## 2. The BITS backend — how the oracle is actually driven `[V] from src/`

Location: `C:/bits/backend` (Express + Prisma/Postgres + TS). Device access is centralized in **`src/shared/lib/zk-driver.ts`**.

### 2.1 Connection profile
- `new ZKLibTCP(ip, port, 30000)` — **TCP only** (deliberately bypasses ZKLib's UDP fallback; sets `connectionType='tcp'` manually). No comm key. Default `ZK_HOST=192.168.1.201`, port 4370; devices also stored in the DB with per-device ip/port.
- Frequent short-lived sessions: driver connects and disconnects around operation batches; a driver pool (`getDriver(ip, port)`) serves the sync queue.

### 2.2 Operation inventory (commands the simulator must serve)
| Driver method | Wire behavior |
|---|---|
| `connect` / `disconnect` | `CMD_CONNECT` / `CMD_EXIT` |
| `getInfo` | `CMD_GET_FREE_SIZES` (reads offsets 24/40/72) |
| `getTime` / `setTime` | `CMD_GET_TIME`; `CMD_SET_TIME` + u32-LE 372-day encoding (PHT +8 h applied by the driver) → then `CMD_REFRESHDATA` |
| `getUsers` | `freeData` → `CMD_DATA_WRRQ(users payload)` → `CMD_DATA` burst → `freeData`; maps `uid, userId, name, password, role, cardno` |
| `setUser` | manual 72-B `CMD_USER_WRQ` record (uid@0, role@2, password@3, name@11, cardno@35, userId@48 — **no tag byte**) → `CMD_REFRESHDATA` |
| `deleteUser` | `CMD_DELETE_USER` [uid u16] |
| `getLogs` | `CMD_ATTLOG_RRQ`-equivalent: `CMD_DATA_WRRQ(att payload)` → 40-B records → `{deviceUserId, recordTime, status}` |
| `clearAttendanceLogs` | `CMD_CLEAR_ATTLOG` |
| `getFingerCount` | probes `CMD_USERTEMP_RRQ[uid u16, finger u8]` ×10; counts replies with data beyond the 8-B header; errors = empty slot |
| `getFingerTemplate` / `readAllFingerprintTemplates` | `CMD_USERTEMP_RRQ`; strips a 6-B entry header `[size u16][uid u16][fid u8][flag u8]`; treats >100 B as a real template |
| `setFingerTemplate` | full write flow (protocol notes §7): DISABLE → DEL_TMP → PREPARE_DATA → DATA → CHECKSUM_BUFFER → TMP_WRITE → FREE → REFRESH → ENABLE |
| `startEnrollment` | `CMD_CANCELCAPTURE` then `CMD_STARTENROLL` with `[userId 24 B ascii][finger i8][flag i8=1]` |

### 2.3 Data pipeline (what a punch has to survive)
- `syncZkData` (`modules/devices/zk/zk-sync.service.ts`) calls `zk.getLogs()` and ingests records into Postgres.
- **`modules/system/syncScheduler.ts`** polls on a ~30 s interval (configurable, shift-aware PEAK/OFF-PEAK modes) + 5-min orphan recovery; manual sync via `POST /attendance/sync`.
- **Realtime events are not consumed** (no `getRealTimeLogs` anywhere in src) → events are for the web UI/future clients only, which is why per-session gating (FR-9) is safe.
- Attendance records are stored/interpreted in PHT (`Asia/Manila`); the device clock is driven to PHT via `setTime`.

### 2.4 Other libraries
- `zklib-js@^1.3.5` is declared in backend `package.json` with only a `declare module 'zklib-js'` shim — **no runtime import found** in `src/` (legacy dependency) `[V]`.
- `zklib-ts` (npm, actively maintained TS rewrite) — alternative oracle for cross-checks; not used by the backend.
- `pyzk` — second independent implementation; used to justify spec-level checksum/flows; optional Phase-1 cross-check.

### 2.5 Consequence for the simulator
Command coverage is defined by §2.2 — 17 distinct commands. Everything else is ACK_UNKNOWN-tolerated. The oracle's 30-s polling plus short-lived sessions means the simulator's worst-case load is trivial, but session hygiene and correct `freeData` handling are exercised constantly.

