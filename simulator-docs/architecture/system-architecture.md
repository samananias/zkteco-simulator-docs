# System Architecture

## 1. Shape

**One Node.js process, four layers, one shared state store.** No microservices, no broker, no DB server (SQLite optional file). Chosen for demo reliability, testability, and zero operational surface — every part earns its place (see ADR-001).

```text
                        BITS backend (unmodified)
                        node-zklib@1.3.0 → ZKLibTCP
                                 │ TCP :4370
┌────────────────────────────────▼────────────────────────────────┐
│ zkteco-simulator-app (single process)                           │
│                                                                 │
│  Transport        net.Server → frame reassembly → Session       │
│  Protocol engine  codec (checksum, structs) → Command dispatcher│
│                       │ reads/writes                            │
│  Device state     DeviceStore (in-memory | SQLite adapter)      │
│                   DeviceClock (372-day codec) · DeviceOptions   │
│                       │ domain events (punch, user, lcd…)       │
│  Web/API          HTTP static + REST  +  WebSocket (ws)         │
│                       │                                         │
│  Device-face UI   public/ — bezel kiosk replica (vanilla JS)    │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Modules & responsibilities

| Module (src/) | Responsibility | Key docs |
|---|---|---|
| `transport/tcp-server.ts` | TCP :4370, byte-stream → frames, per-socket Session (id, registered event flags, in-flight flag, event queue) | protocol-engine.md |
| `protocol/packet.ts` | Frame build/parse, checksum (two variants), validation | protocol-engine.md |
| `protocol/records.ts` | 72-B user, 40-B attendance, 32-B event codecs (offsets co-located with decoders) | data-model.md |
| `protocol/time.ts` | 372-day-year clock codec | zk-protocol-notes.md §6 |
| `protocol/commands/*` | One handler module per command group (session, info, users, attendance, templates, events, cosmetic) | protocol-engine.md |
| `device/store.ts` | `DeviceStore` interface + in-memory impl; entities Users / Attendance / Templates / Options / Lcd | data-model.md |
| `device/sqlite-store.ts` | Optional persistence adapter (Phase 5) | data-model.md |
| `device/events.ts` | Internal event bus: punch/user/lcd/state → protocol emission + WS broadcast | api-and-events.md |
| `web/http.ts` + `web/routes/*` | Static UI, REST API (punch, state, users, attendance, lcd) | api-and-events.md |
| `web/ws.ts` | WebSocket server + fan-out of state snapshots and live events | web-ui.md |
| `seed/demo-data.ts` | Employees, a week of attendance, placeholder templates | data-model.md |
| `config.ts` | Env/CLI config (ports, comm key, checksum variant, seed profile, log level) | ../operations/configuration.md |

## 3. Data flow — the two demo directions

**Punch from web UI → backend DB:**
UI "scan" → REST `/api/punch` → store appends AttendanceRecord → event bus → (a) WS: verification screen + updated state; (b) protocol engine: `EF_ATTLOG` to registered sessions (queued if in-flight). BITS backend picks it up on its ~30 s poll (`getAttendances`) and ingests into Postgres; realtime events additionally available to any registering client.

**Backend action → web UI:**
backend `setUser` → `CMD_USER_WRQ` → store update → event bus → WS: user list/UI reflects; next backend read returns the user (two-way sync).

## 4. Technology (ADR-002 justification)

| Choice | Rationale | Rejected alternative |
|---|---|---|
| Node.js + TypeScript | Byte-layout code benefits from literal types/struct types; same runtime as the oracle so tests run in-process | Plain JS (viable; ADR-002 records the tradeoff) |
| `net` + `http` + `ws` only | The protocol is plain TCP; no framework needed for one static page + 6 endpoints | Express/React — unnecessary weight here |
| `node:test` + `tsx` | Built-in runner, zero config; `tsx` for TS execution | Jest (config overhead for a small suite) |
| `node-zklib@1.3.0` (exact) as devDependency | The acceptance oracle — tests import the backend's actual client | Testing against spec prose only |
| in-memory store default | Demo-first; SQLite only when persistence is wanted | Postgres/etc. — no justification |

## 5. Error handling principles

- Unknown/malformed packets → log (hex) + `CMD_ACK_UNKNOWN`; session survives.
- Abrupt disconnects → session cleanup; event queues dropped.
- Port conflicts → pre-flight check with a human-readable error (Windows reserved ranges).
- Never throw across the protocol boundary: every command handler returns a reply.

## 6. Repository relationship

`zkteco-simulator-app` contains this system; everything above (and below) is documented in this docs repo. The app README links here for any "why" question.
