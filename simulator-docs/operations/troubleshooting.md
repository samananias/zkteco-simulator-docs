# Troubleshooting

Symptom → cause → fix. Add entries as real problems are met (this file grows with experience — that's its job).

## Connection / transport

| Symptom | Likely cause | Fix |
|---|---|---|
| Backend logs `TIMEOUT_ON_WRITING_MESSAGE` / `NO_REPLY_ON_CMD_CONNECT` | Simulator not running; wrong IP/port; firewall blocking 4370; Hyper-V reserved the port range | Check `npm run dev` ready line; `Test-NetConnection <host> -Port 4370`; change `ZK_TCP_PORT` (Windows may reserve ranges — see boot pre-flight error) |
| `EADDRINUSE` on 4370 | Another simulator instance (or real device tool) bound | Stop it, or set `ZK_TCP_PORT`; the pre-flight message names the conflicting process on Windows |
| Connects, then immediately disconnects | Backend connected to a *real* device elsewhere (stale `ZK_HOST`) or UDP fallback fired because TCP was refused | Ensure TCP reachable; the backend's driver is TCP-only — check its logs for `[ZKDriver] Connected` |
| Works locally, not from another machine | Firewall profile / binding | `ZK_HTTP_HOST` + allow 4370/tcp inbound on private networks |

## Protocol / data

| Symptom | Likely cause | Fix |
|---|---|---|
| `getUsers` returns users with garbage names | Consumer expects a different name encoding | Names are ascii NUL-terminated at offset 11 (72-B record) — check consumer version; 1.3.0 and backend verified compatible `[V]` |
| `getInfo` counts look wrong | Reply buffer shorter than 76 B or offsets shifted | Offsets are 24/40/72 in the post-TCP-strip buffer — compare against `zk-protocol-notes.md` §4 |
| Timestamps off by hours/days | Time codec misuse — 372-day calendar is not unix time | Use `protocol/time.ts` both directions; check `ZK_TIME_OFFSET_MIN`; backend drives PHT via `setTime` |
| Attendance sync sees old/no records | Backend caches or `CLEAR_ATTLOG` was issued by a script | Backend polls the full log each cycle; verify with `GET /api/attendance` on the simulator |
| Fingerprint probe counts 0 for everyone | Templates not seeded / probe replies `ACK_OK` instead of data-or-error | Enrolled slot must answer `CMD_DATA` with 6-B entry header; empty slot must answer `CMD_ACK_ERROR` (driver counts `length > 8`) |
| Client hangs after a large read | Dataset burst exceeded the client's 1 s quiet-window assumptions or chunked mode mismatch | Keep bursts back-to-back; try `ZK_READ_MODE=chunked`; enable `ZK_LOG_LEVEL=debug` and compare frame timings |

## Realtime events

| Symptom | Likely cause | Fix |
|---|---|---|
| `getRealTimeLogs` callback never fires | Session never registered (`CMD_REG_EVENT` not sent) or punch raised while exchange in flight (queued) | Register first; events flush on quiet by design (ADR-006) |
| Commands randomly fail while event client attached | Client consumed an event as a command reply (client-side multiplexing limitation) | Client must filter event frames (node-zklib master does; 1.3.0 only in read paths) — or keep event client and command client on separate sessions |

## Web UI

| Symptom | Likely cause | Fix |
|---|---|---|
| UI shows "disconnected" | WS down (server restarted) | Auto-reconnect with backoff; refresh for snapshot |
| Clock drifts from backend expectation | Backend expects PHT; device clock is simulator host time ± `ZK_TIME_OFFSET_MIN` | Backend's `setTime` aligns it; or set the offset explicitly |

## Debugging workflow
1. `ZK_LOG_LEVEL=debug` → per-frame hex lines.
2. Reproduce with the L2 integration test closest to the symptom (all oracle call patterns are scripted there).
3. Capture the failing exchange (log excerpt) into the issue/notes; if it reveals a wire-level fact, update `../research/zk-protocol-notes.md` and the relevant ADR.
