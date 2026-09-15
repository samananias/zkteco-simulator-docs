> **ARCHIVED ORIGINAL INPUT — HISTORICAL DOCUMENT.**
> This is the initial project brief, preserved verbatim as the origin of the project. It is **superseded** by
> `functional.md` / `non-functional.md` (current requirements) and `../research/feasibility-report.md`
> (what changed and why). Do **not** edit this file; make changes in the current docs and record them in
> `../status/change-history.md`. Item-level mapping: `traceability.md`.

---

# ZKTeco Device Simulator — Project Brief & AI Build Context

**Purpose of this document:** This is the single source of truth to hand to an AI coding assistant (or a new team member) to start, continue, or finish this project. It explains *why* the project exists, the *research basis* for every technical decision, the *protocol* the simulator must speak, the *features* required, and the *visual/UX fidelity* needed so that connecting to the simulator feels indistinguishable from connecting to a real ZKTeco terminal.

---

## 1. Project Context

We built an attendance/HR system during our OJT internship that talks to a physical ZKTeco biometric terminal (fingerprint/RFID attendance device) over the local network. We no longer have access to the physical device, but we want to showcase the working system in our portfolio.

**Goal:** Build a software simulator that:
1. Sits on the network (or `localhost`) and speaks the *exact same wire protocol* a real ZKTeco terminal speaks, so our existing Node.js backend can connect to it with zero code changes (just point it at the simulator's IP/port instead of a real device).
2. Optionally exposes a **web UI that visually mimics the physical device's on-screen interface** (clock screen, verification success/fail screens, menu icons) — so a demo/portfolio viewer sees something that looks and behaves like the real hardware, not just a bare API.

This is a well-established practice — protocol/device simulators are commonly built for testing and demos when hardware isn't available (e.g., Modbus/MQTT industrial device simulators, IoT simulators, etc.). It is realistic here because the ZKTeco protocol has been independently reverse-engineered and documented, and multiple open-source client libraries (Python, Go, Rust, PHP, Node.js) all implement it consistently — giving us solid ground truth to build against.

---

## 2. Research Basis (sources to build from)

| Source | What it gives us |
|---|---|
| `adrobinoga/zk-protocol` (GitHub) | The most complete, independently reverse-engineered spec of the ZKTeco standalone-device protocol: packet structure, command/reply codes, checksum algorithm, session handling, realtime event codes. Written by analyzing real traffic against ZKTeco's TFT-series devices. |
| `caobo171/node-zklib` (npm: `node-zklib`) | The most widely used **Node.js client library** for ZKTeco devices. Since our backend is Node.js, this is very likely the library (or a fork of it, e.g. `zklib-ts`) our existing system uses. It implements `getInfo()`, `getUsers()`, `getAttendances()`, `setUser()`, `executeCmd()`, and more — this tells us exactly which commands the simulator must respond to. |
| `fananimi/pyzk` (Python) | Reference implementation cross-checked against the protocol spec; useful for validating our simulator's responses are correct if we prototype in Python before porting logic to Node. |
| ZKTeco official user manuals (iFace series, K30, SF300, UA-150, etc.) | Describe the actual **on-device screens**: main clock/status screen, verification result screens, menu structure (User Management, Department, Shift Settings, Reports, System Settings), status bar icons (network, alarm, battery). This is our visual reference. |

**Important limitation to document honestly:** No amount of software can simulate an actual fingerprint/face scan. The simulator fakes the *result* of a scan (a successful/failed verification event), not the biometric capture hardware itself. For a demo, this is presented as: tap a virtual "finger" button or select a simulated employee card, and the simulator emits the same data a real scan would produce.

---

## 3. Protocol Specification to Implement

The classic ZKTeco standalone protocol runs over **TCP (and UDP) port 4370**. It is a binary request/reply protocol. This is the protocol `node-zklib` and equivalents speak, and the one our backend already uses (confirmed: our system connects to a device IP on port 4370 and pulls data).

### 3.1 Packet structure

Every packet:

| Field | Size | Notes |
|---|---|---|
| Start marker | 4 bytes | Fixed value `50 50 82 7D` |
| Payload size | 4 bytes | Little-endian integer |
| Payload | variable | See below |

Payload (regular packets):

| Field | Size | Notes |
|---|---|---|
| Command ID / reply code | 2 bytes | Little-endian |
| Checksum | 2 bytes | Little-endian, see algorithm below |
| Session ID | 2 bytes | Assigned by the "device" (our simulator) on connect, constant for the life of the connection |
| Reply number | 2 bytes | Increments per request/response pair, starts at 0 |
| Data | remainder | Command-specific |

**Realtime/event packets** (sent unprompted by the device to report live events) reuse the same shape but: command ID is always the "register event" code, the session-ID field is repurposed to carry an *event code*, and reply number is always zero.

### 3.2 Checksum algorithm

1. Treat the payload (excluding the checksum field itself) as a sequence of little-endian 16-bit words; pad with a zero byte if the length is odd.
2. Sum all the 16-bit words into a 32-bit accumulator.
3. Fold the high 16 bits into the low 16 bits (add them together).
4. Take the one's complement (bitwise NOT) of the result — that's the 16-bit checksum.

This must be implemented **exactly** (in both directions — validating incoming packets and generating outgoing ones), or real client libraries will silently reject the simulator's responses.

### 3.3 Session & reply-number handshake

- On `CMD_CONNECT`, the simulator generates and returns a session ID. It stays constant until disconnect.
- Reply numbers start at 0 and increment in lockstep between client and "device" for every request/response pair. Realtime event packets don't participate in this counter.

### 3.4 Core commands the simulator must handle

Grouped by what our Node backend actually needs (per `node-zklib`'s public API), in priority order:

**Must-have (core demo flow):**
- `CMD_CONNECT` / `CMD_EXIT` — begin/end session
- `CMD_AUTH` — authenticate session with a comm key (if our system sets a password)
- `CMD_GET_VERSION` — firmware/version string (feeds `getInfo()`)
- `CMD_GET_FREE_SIZES` — device capacity/status numbers (user count, log count, storage)
- `CMD_GET_TIME` / `CMD_SET_TIME` — device clock
- `CMD_USER_WRQ` / `CMD_DB_RRQ` (user table read) — list/add/edit users, backs `getUsers()` / `setUser()`
- `CMD_ATTLOG_RRQ` — read attendance log, backs `getAttendances()`
- `CMD_CLEAR_ATTLOG` — clear attendance log
- `CMD_ENABLEDEVICE` / `CMD_DISABLEDEVICE` — device state toggles used around bulk reads
- `CMD_REG_EVENT` + realtime event stream (`EF_ATTLOG`, `EF_VERIFY`) — **this is what makes a live demo compelling**: trigger a fake punch and watch it appear in the real system in real time

**Nice-to-have (rounds out the "full device" feel):**
- `CMD_USERTEMP_RRQ` / `CMD_USERTEMP_WRQ` — fingerprint template read/write (return placeholder binary blobs, not real biometric data)
- `CMD_DELETE_USER` / `CMD_DELETE_USERTEMP`
- `CMD_UNLOCK` — simulated door relay trigger
- `CMD_RESTART` / `CMD_POWEROFF` / `CMD_SLEEP` / `CMD_RESUME` — device state, purely cosmetic in the simulator
- `CMD_WRITE_LCD` / `CMD_CLEAR_LCD` — if our system ever pushes text to the device screen, mirror it in the web UI

**Exchange-of-large-data flow:** for commands that return more data than fits in one packet (e.g., full user table, large attendance logs), the real protocol uses a `CMD_PREPARE_DATA` → `CMD_DATA` (chunked) → `CMD_ACK_OK` handshake. This must be implemented for any dataset over ~1KB or client libraries will hang waiting for continuation packets.

### 3.5 Data record formats

- **User record:** UID (internal index), user ID/PIN (external ID), name, privilege level (0 = normal, 14 = admin — exact values per spec), password, card number, group.
- **Attendance record:** user ID/PIN, timestamp, verification method (fingerprint / card / password / face), in/out state, work code.

These must be packed into the exact fixed-width binary struct layout the client library expects (byte offsets matter — this is the #1 source of subtle bugs when reimplementing this protocol, so structs should be validated against `node-zklib`'s parsing code directly, not just against the spec prose).

---

## 4. System Architecture

```
┌─────────────────────────────┐      TCP 4370       ┌───────────────────────────┐
│  Our existing HR/Attendance │◄────────────────────►│   ZKTeco Simulator         │
│  system (Node.js backend)   │  (unmodified client)  │   ┌─────────────────────┐ │
│  — already exists, no       │                        │   │ Protocol Engine     │ │
│  changes needed             │                        │   │ (packet parse/build,│ │
└─────────────────────────────┘                        │   │  checksum, sessions)│ │
                                                          │   └──────────┬──────────┘ │
                                                          │              │            │
                                                          │   ┌──────────▼──────────┐ │
                                                          │   │ In-memory / SQLite  │ │
                                                          │   │ "device state" store│ │
                                                          │   │ (users, logs, config)│ │
                                                          │   └──────────┬──────────┘ │
                                                          │              │            │
                                                          │   ┌──────────▼──────────┐ │
                                                          │   │ Web UI (device face)│ │
                                                          │   │ served separately,  │ │
                                                          │   │ WebSocket-linked to │ │
                                                          │   │ device state store  │ │
                                                          │   └─────────────────────┘ │
                                                          └───────────────────────────┘
```

Two layers, intentionally decoupled:

1. **Protocol engine** — a raw Node `net.Server` (plus `dgram` for UDP if needed) that real client libraries connect to on port 4370. This is invisible to a demo viewer; it's what makes the existing backend "just work" unmodified.
2. **Visual front-end** — a separate web app (served on a normal HTTP port) that renders a **pixel-styled replica of a physical ZKTeco terminal's screen**, driven by the same underlying device-state store. This is what a portfolio viewer actually looks at during a demo.

Both layers read/write the same shared state (in-memory object or a small SQLite file) so that a "punch" triggered from the visual UI is what the protocol engine reports back to the real backend via `CMD_ATTLOG_RRQ` / realtime events — and vice versa, admin actions from the real backend (like adding a user) should reflect on the simulated screen.

---

## 5. Visual / UX Fidelity Requirements

Based on ZKTeco's official manuals (iFace series, K30, SF300, UA-series terminals), the physical device UI to replicate has these recurring characteristics:

- **Idle/main screen:** large digital clock (HH:MM:SS + date), company/device name, small status-bar icons (network connectivity, alarm/tamper state, battery if applicable), and a prompt like "Please place finger / swipe card / enter password."
- **Verification screens:** on a scan, the screen briefly switches to a large, high-contrast **pass** (green checkmark + user name/ID + "Thank you!") or **fail** (red X + "Please try again") state, then reverts to idle after a couple seconds.
- **Menu system:** icon-grid style menu (not a scrolling list) for functions like User Management, Attendance Search, System Settings, Data Management — TFT-series devices use a color touchscreen with square icon tiles, similar to an early-2010s embedded touchscreen kiosk aesthetic, not a modern flat-design mobile app.
- **Color/typography cues:** dark or navy backgrounds with white/cyan text on the idle screen; bold sans-serif digits for the clock; status icons rendered as small monochrome glyphs in a top status bar — deliberately *utilitarian*, not decorative.
- **Form factor:** should be presented in the browser inside a **device bezel frame** (rounded rectangle "hardware" border, small physical-looking speaker grille/keypad graphic below the screen) rather than a borderless web page — this single detail does more than anything else to sell the "this is a device, not a website" illusion in a portfolio screenshot or video.

The `frontend-design` guidance should be consulted when actually building this UI to keep the styling intentional rather than defaulting to generic component-library looks — the target here is a **kiosk/embedded-hardware aesthetic**, which is a distinct design language from a typical SaaS dashboard.

---

## 6. Feature Checklist

### Phase 1 — Protocol core (makes the existing backend work unmodified)
- [ ] TCP server on port 4370, raw socket handling
- [ ] Packet parser/builder with correct checksum implementation
- [ ] Session ID + reply-number handshake
- [ ] `CMD_CONNECT`, `CMD_AUTH`, `CMD_EXIT`
- [ ] `CMD_GET_VERSION`, `CMD_GET_FREE_SIZES`, `CMD_GET_TIME`/`CMD_SET_TIME`
- [ ] Seeded in-memory (or SQLite) store: fake employees + fake attendance history
- [ ] `CMD_DB_RRQ` / user read, `CMD_ATTLOG_RRQ` / attendance read, with chunked large-data flow
- [ ] `CMD_ENABLEDEVICE` / `CMD_DISABLEDEVICE`
- [ ] Validate end-to-end against the real backend (point it at the simulator, confirm `getInfo()`, `getUsers()`, `getAttendances()` return sane data)

### Phase 2 — Live demo behavior
- [ ] Realtime event packets (`CMD_REG_EVENT`) so a "punch" shows up in the real backend without polling
- [ ] A trigger mechanism (small local HTTP endpoint or CLI) to simulate a punch on demand during a live demo
- [ ] `CMD_USER_WRQ` (add/edit user) and `CMD_DELETE_USER` so the demo can show two-way sync (add a user in the real system → confirm it can be "read back" from the simulator)

### Phase 3 — Visual device face
- [ ] Web UI styled as a physical device screen (idle clock, verify pass/fail states, menu grid) per Section 5
- [ ] WebSocket (or polling) link from UI to the shared device-state store, so triggering a punch from the UI reflects instantly in the real backend, and vice versa
- [ ] Bezel/hardware-frame presentation for screenshots/video

### Phase 4 — Portfolio polish
- [ ] Seed realistic demo data (a handful of named fake employees, a week of varied attendance patterns — on-time, late, absent)
- [ ] README documenting the simulator as a deliberate engineering choice (explains *why* a simulator was built, which doubles as a talking point about protocol-level problem solving)
- [ ] Short demo script/video: show the "device" idle screen, trigger a punch, cut to the real backend dashboard updating live

---

## 7. Recommended Tech Stack

- **Simulator core:** Node.js (`net` + `dgram` built-in modules, no need for `node-zklib` itself since we're building the *server* side — but keep a copy of its source open as the ground-truth reference for exact byte layouts).
- **State store:** in-memory objects for simplicity, or `better-sqlite3` if persistence across restarts is wanted for the portfolio demo.
- **Visual front-end:** plain HTML/CSS/JS or a small React app; WebSocket (`ws` package) for live sync with the protocol engine.
- **Testing:** run the real existing backend against the simulator as the primary test harness — if `getInfo()`, `getUsers()`, and `getAttendances()` return correct-looking data through the unmodified client library, the protocol implementation is validated by construction.

---

## 8. Open Questions to Resolve Before/During Build

1. Confirm which exact npm package (and version) the existing backend imports — `node-zklib`, `zklib-ts`, or a fork — so the simulator is validated against that library's actual parsing code, not just the general spec.
2. Confirm whether the backend uses TCP, UDP, or both (`node-zklib` supports both with fallback).
3. Confirm whether a comm key / password is configured on the backend's device connection (`CMD_AUTH` handling depends on this).
4. Decide how "deep" Phase 3 visual fidelity needs to go for the portfolio goal — a convincing idle + verify screen may be enough; a full touchscreen menu system is a bigger, lower-priority undertaking.

---

## 9. Why This Is a Good Portfolio Story

Framing this correctly matters as much as building it: the interesting engineering work here isn't "we couldn't get a device so we faked one" — it's "we reverse-engineered and reimplemented a proprietary hardware protocol from the wire format up, byte-for-byte, well enough that our own production system can't tell the difference." That's a legitimately strong systems-programming/protocol-engineering story for a portfolio, independent of the original attendance-system project itself.
