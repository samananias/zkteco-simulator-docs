# Development Roadmap

**Rule:** a phase is done only when its gate (testing) passes **and** the docs listed in its last column are updated (`../process/development-workflow.md`).

## Phase 0 — Foundations ✅ (this commit)
Repos established (`zkteco-simulator-app`, `docs`), research verified against spec + oracle sources, feasibility analyzed, architecture and docs foundation written, ADR-001…008 accepted.

## Phase 1 — Protocol core
- [ ] Project skeleton (`src/` layout per architecture docs), config module, logging
- [ ] Frame reader (reassembly), packet codec, checksum (both variants), session
- [ ] Handlers: CONNECT/AUTH/EXIT, GET_VERSION, GET_FREE_SIZES, OPTIONS_RRQ/WRQ, GET/SET_TIME (372-day codec), ENABLE/DISABLE, REFRESHDATA, FREE_DATA, ACK_UNKNOWN fallback
- [ ] In-memory store + seed data basics
- **Gate:** unit golden-byte tests pass; integration: oracle `createSocket + connect + getInfo + getTime` green
- **Docs:** ADR-005 outcome, change-history, traceability rows

## Phase 2 — Data layer
- [ ] Record codecs (72/40-B), direct `CMD_DATA` burst reads, chunked mode (config-gated) + its integration test
- [ ] USER_WRQ (72/73-B tolerant), DELETE_USER, DELETE_USERTEMP, CLEAR_ATTLOG, CLEAR_DATA, DB_RRQ/ATTLOG_RRQ styles
- [ ] Template flows: USERTEMP_RRQ probe/read, PREPARE_DATA→DATA→CHECKSUM_BUFFER→TMP_WRITE write flow
- **Gate:** oracle green for `getUsers / setUser / deleteUser / getAttendances / clearAttendanceLog / getFingerCount / getFingerTemplate / setFingerTemplate` call patterns
- **Docs:** data-model updates, change-history

## Phase 3 — Realtime + demo triggers
- [ ] REG_EVENT handling, EF_ATTLOG emission, per-session gating + in-flight queueing (ADR-006)
- [ ] REST API (`/api/punch`, state, users, attendance, lcd, reset), `scripts/punch.mjs`
- **Gate:** integration: registered oracle session receives correct EF_ATTLOG frame for a REST punch; unregistered session (backend pattern) receives none; event while `getAttendances` in flight never desyncs
- **Docs:** api-and-events, troubleshooting entries

## Phase 4 — Device-face UI
- [ ] Bezel + keypad shell, idle clock (device time), verify pass/fail overlays, disabled state, LCD mirror
- [ ] WS sync + reconnect; menu grid skeleton
- **Gate:** scripted e2e: punch from UI visible on device screen + in backend DB ≤ 30 s; backend `setUser` visible on UI; visual QA checklist (bezel, glyphs, colors)
- **Docs:** web-ui updates, screenshots for README

## Phase 5 — Portfolio polish
- [ ] Full seed week, optional SQLite store (config), README (why-a-simulator story), demo script + optional video, final docs pass
- **Gate:** full test suite green; fresh-clone setup walkthrough (`operations/setup.md`) executed verbatim
- **Docs:** change-history, known-limitations final review, future-work

## Milestones after v1 (non-committed)
UDP transport · pyzk cross-oracle CI · iFace theme · multi-device instances · Docker packaging — see `../status/future-work.md`.
