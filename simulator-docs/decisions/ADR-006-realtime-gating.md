# ADR-006: Realtime events — per-session registration + in-flight queueing

**Status:** accepted · **Date:** 2026-09-15

## Context
Realtime events (`CMD_REG_EVENT` → unsolicited `EF_ATTLOG` frames) make the demo compelling. But the protocol multiplexes commands and events over one TCP stream, and the oracle's `writeMessage` resolves on the **first** arriving packet — an event interleaved with a command exchange would be consumed as that command's reply `[V]`. Meanwhile the BITS backend never registers for events (its poller uses `getAttendances`) `[V]`.

## Decision
Events are emitted **only** to sessions that explicitly registered via `CMD_REG_EVENT`. If a registered session has a request/response exchange in flight, events are queued and flushed when the socket goes quiet. The punch→DB path never relies on events (the backend polls).

## Consequences
- (+) Zero desync risk for the unmodified backend; safe for event-consuming clients (they registered, so they expect frames).
- (+) The web UI gets punches instantly via WebSocket regardless of event gating.
- (−) An event-consuming client that simultaneously issues commands may see delayed (not lost) events — inherent to the single-stream protocol; documented in known-limitations.
