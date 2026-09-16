# Development Roadmap

**Rule:** a phase is done only when its gate (testing) passes **and** the docs listed in its last column are updated (`../process/development-workflow.md`).

## Phase 0 — Foundations ✅ (this commit)
Repos established (`zkteco-simulator-app`, `docs`), research verified against spec + oracle sources, feasibility analyzed, architecture and docs foundation written, ADR-001…008 accepted.

## Phase 1 — Protocol core
- [x] Project skeleton (`src/` layout per architecture docs), config module, logging
- [x] Frame reader (reassembly), packet codec, checksum (both variants), session
- [x] Handlers: CONNECT/AUTH/EXIT, GET_VERSION, GET_FREE_SIZES, OPTIONS_RRQ/WRQ, GET/SET_TIME (372-day codec), ENABLE/DISABLE, REFRESHDATA, FREE_DATA, ACK_UNKNOWN fallback
- [x] In-memory store + seed data basics
- **Gate:** unit golden-byte tests pass; integration: oracle `createSocket + connect + getInfo + getTime` green
- **Docs:** ADR-005 outcome, change-history, traceability rows

## Phase 2 — Data layer
- [x] Record codecs (72/40-B), direct `CMD_DATA` burst reads, chunked mode (config-gated) + its integration test
- [x] USER_WRQ (72/73-B tolerant), DELETE_USER, DELETE_USERTEMP, CLEAR_ATTLOG, CLEAR_DATA, DB_RRQ/ATTLOG_RRQ styles
- [x] Template flows: USERTEMP_RRQ probe/read, PREPARE_DATA→DATA→CHECKSUM_BUFFER→TMP_WRITE write flow
- **Gate:** oracle green for `getUsers / setUser / deleteUser / getAttendances / clearAttendanceLog / getFingerCount / getFingerTemplate / setFingerTemplate` call patterns
- **Docs:** data-model updates, change-history

## Phase 3 — Realtime + demo triggers
- [x] REG_EVENT handling, EF_ATTLOG emission, per-session gating + in-flight queueing (ADR-006)
- [x] REST API (`/api/punch`, state, users, attendance, lcd, reset), `scripts/punch.mjs`
- **Gate:** integration: registered oracle session receives correct EF_ATTLOG frame for a REST punch; unregistered session (backend pattern) receives none; event while `getAttendances` in flight never desyncs
- **Docs:** api-and-events, troubleshooting entries

## Phase 4 — Device-face UI
- [x] Bezel + keypad shell, idle clock (device time), verify pass/fail overlays, disabled state, LCD mirror
- [x] WS sync + reconnect; menu grid skeleton
- **Gate:** scripted e2e: punch from UI visible on device screen + in backend DB ≤ 30 s; backend `setUser` visible on UI; visual QA checklist (bezel, glyphs, colors)
- **Docs:** web-ui updates, screenshots for README

## Phase 5 — Portfolio polish
- [x] Full seed week, optional SQLite store (config), README (why-a-simulator story), demo script + optional video, final docs pass
- **Gate:** full test suite green; fresh-clone setup walkthrough (`operations/setup.md`) executed verbatim — **MET 2026-09-15: 25/25 tests green** (11 golden-byte unit + 11 oracle integration + 3 sqlite gate); CLI boot smoke-verified (seed 5 users / 40 records / 10 templates, UI 200)
- **Docs:** change-history, known-limitations final review, future-work

## Milestones after v1 (non-committed)
UDP transport · pyzk cross-oracle CI · iFace theme · multi-device instances · Docker packaging — see `../status/future-work.md`.

---

## Scope extension — ZKTeco substitute (decided 2026-09-15)

Owner direction: the project should not stop at *simulating* a terminal but become a **usable ZKTeco substitute** — real fingerprint enrollment and real matching, with the ZK protocol surface frozen so BITS keeps working unmodified. Research: `../research/biometrics.md`. Design: `../architecture/biometric-core.md`, `capture-stations.md`.

| Phase | Content | Gate | Docs |
|---|---|---|---|
| **6 — Biometric core** | ADR-010 spike (matcher sidecar) → `BiometricStore` with AES-256-GCM at rest (ADR-011) → enrollment service (N samples + quality gate) → 1:N identify + 1:1 verify → REST API v2 (FR-14…FR-15, FR-17, FR-18) | Spike criteria in ADR-010 met (10/10 identify, `< 200 ms`); enroll → read template back via oracle → delete → probe error (FR-8 unchanged); no plaintext template in log/API output — **implemented 2026-09-16**: spike 4/4 criteria (boot 3.86 s, identify 33–51 ms, 10/10 + impostor ≤ 2.1 vs ≥ 62.6), encrypted store + service + REST v2 live, 43/43 tests (13 biometric unit + 5 biometric integration incl. ciphertext-at-rest & purge-or-nothing); the N-sample capture ceremony itself lands with Phase 7 stations | biometric-core, ADR-010 (→accepted), ADR-011, configuration, security, change-history ✅ |
| **7 — Capture stations** | Camera PWA station (FR-16.1); import + fixture stations for seeding/tests — *Android + USB-OTG scanner adapter deferred (owner decision 2026-09-16, allowance-gated; see future-work)* | Live enroll from the camera station → punch identified on the device-face UI **and** ingested by BITS; quality gate rejects poor captures with actionable reasons; scripted UX checks; human-judged quality assessment recorded — **implemented 2026-09-16 (automated)**: capture endpoint + quality gate (measurement sidecar-side, policy app-side, thresholds env-tunable), N-sample ceremony with pairwise consistency via the engine's `/match`, `POST /api/biometric/punch` (identify → AttendanceRecord verifyType=1 → EF_ATTLOG + device face), `/enroll` + `/scan` PWA pages, fixture tests; 59/59 tests (16 new: 7 quality unit, 3 ceremony unit, 6 capture integration incl. oracle EF_ATTLOG); **remaining for the gate:** owner-run live-camera UX check + human-judged quality assessment | capture-stations, biometrics research, known-limitations, troubleshooting ✅ |
| **8 — Substitute hardening** | API v2 docs, consent/retention flows, multi-station operation, deployment/run guide, ADR-010 acceptance finalized | BITS runs a full workday against the substitute with real punches; `purge` leaves no ciphertext; docs pass complete | api-and-events, security, setup, future-work, change-history |

**Ordering rule:** Phases 6–8 never start before Phase 1–5 gates are green. Phases 1–5 are the foundation a substitute needs (the frozen protocol surface is what makes it a *substitute* rather than a separate product); the two forward-compatibility hooks (`FpTemplate.format`, `BiometricStore` interface) are the only Phase-1–5 concessions, and they cost nothing.
