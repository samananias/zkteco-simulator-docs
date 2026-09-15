# Protocol Engine Design

Implementation detail for `src/transport/` + `src/protocol/`. Wire facts: `../research/zk-protocol-notes.md`.

## 1. Frame lifecycle

1. **Reassembly:** TCP has no message boundaries — `TcpFrameReader` accumulates bytes, waits for ≥ 8 B magic+length, then for `8 + payloadSize` total bytes, and emits complete frames. A partial trailing buffer is kept; garbage before a magic is logged and skipped.
2. **Parse:** header fields (command, checksum, session, reply) + data. Checksum verified leniently: compute both classic and oracle variants of the incoming packet; accept if either matches (log which). `[V]` justification — the two published variants differ by 1 and the oracle emits its own variant.
3. **Dispatch:** `Session` (per socket) holds sessionId, registered event flags, in-flight flag, event queue. `CommandDispatcher` maps command → handler; handler returns `{ code, data }` (reply frame is built by the engine, echoing the request's reply number).
4. **Build:** reply frame with outbound checksum (config: `classic` default, `oracle` variant available — ADR-005).

## 2. Command handler map (Phase 1–3)

| Group | Commands | Notes |
|---|---|---|
| session | CONNECT (1000) · EXIT (1001) · AUTH (1102) | CONNECT assigns session id into reply; AUTH only when comm key enabled |
| info | GET_VERSION (1100) · GET_FREE_SIZES (50) · OPTIONS_RRQ (11) · OPTIONS_WRQ (12) · GET/SET_TIME (201/202) · STATE_RRQ (64) | free-sizes buffer ≥ 76 B (u32 @24/40/72); time codec module |
| state | ENABLEDEVICE (1002) · DISABLEDEVICE (1003) · RESTART/POWEROFF/SLEEP/RESUME · REFRESHDATA (1013) · FREE_DATA (1502) · CANCELCAPTURE (62) · TESTVOICE | all ACK_OK; state reflected on UI |
| users | USER_WRQ (8) · DELETE_USER (18) · DB_RRQ (7) | USER_WRQ auto-detects 72-B (backend) vs 73-B (pyzk tag) |
| attendance | CLEAR_ATTLOG (15) · CLEAR_DATA (14) · ATTLOG_RRQ (13) | |
| reads | DATA_WRRQ (1503) — payload `0109…` users, `010d…` attendance | direct `CMD_DATA` burst (ADR-004 default); chunked mode behind config |
| templates | USERTEMP_RRQ (9) · USERTEMP_WRQ (10) · DELETE_USERTEMP (19) · DEL_FPTMP (134) · PREPARE_DATA (1500) · DATA (1501, client→device) · CHECKSUM_BUFFER (119) · TMP_WRITE (87) · STARTENROLL (61) | write flow per research §7; pending-buffer state on Session |
| events | REG_EVENT (500) | per-session flags; payload `01 00 00 00` enables EF_ATTLOG |
| lcd | WRITE_LCD (66) · CLEAR_LCD (67) | payload = text rows; mirrored to UI |
| cosmetic | UNLOCK (31) etc. | ACK_OK + UI effect |

Unknown command → `CMD_ACK_UNKNOWN` (65535) + warn log (never disconnect).

## 3. Read-path state machine (oracle-critical)

```text
CMD_DATA_WRRQ(payload) ──► classify payload (users | attendance | unknown)
   ──► store.snapshot() → dataset = [u32 size][records…]
   ──► set in-flight = true
   ──► write CMD_DATA frame(s): one frame if ≤ 65 000 B, else back-to-back frames
   ──► after last byte flushed: keep in-flight ~ quiet-window, then false
```
Rationale: node-zklib 1.3.0 resolves on the *first* reply packet for non-`CMD_DATA` commands and on **1 s of socket silence** for `CMD_DATA` reads, accumulating everything `[V]`. A single burst inside that window is the deterministic path. Chunked mode (ADR-004) is a config-gated alternate implementing the spec conversation exactly.

## 4. Realtime event emission (ADR-006)

- Only sessions with `EF_ATTLOG` registered receive events (registration = `CMD_REG_EVENT` with bit0). The BITS backend never registers → zero unsolicited traffic to the real system.
- Punch pipeline: store append → internal event → for each registered session: if `in-flight` → queue; else write event frame (command 500, session-field = 1, reply 0, 32-B data). Queued events flush when in-flight clears.
- Rationale: on one TCP stream, an unsolicited packet arriving while a client awaits a command reply would be consumed as that reply by 1.3.0's `writeMessage` `[V]`. Gating + queueing makes desync impossible without client cooperation.

## 5. Session lifecycle

- `CONNECT`: session id = deterministic counter (config base), reply carries it in the session field `[V]` (node-zklib reads offset 4).
- Short-lived connections are the norm (backend connects per batch) — no state may be assumed to persist across sessions except the shared store.
- Comm key enabled (optional): first `CONNECT` → `CMD_ACK_UNAUTH`; expect `CMD_AUTH` with makeCommKey(key, sessionId); verify; only then ACK. Disabled by default (backend sends none).

## 6. Observability

Per-session log lines: `→ CMD_CONNECT(len) … ← ACK_OK(8)`; `LOG_LEVEL=debug` adds hex dumps; integration tests assert on parsed model objects, not logs.
