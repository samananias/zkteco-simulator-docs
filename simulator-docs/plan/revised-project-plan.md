# Revised Project Plan

The original brief, run through the feasibility filter, expressed as buildable work. Full analysis: `../research/feasibility-report.md`; requirement text: `../requirements/functional.md`; status tracking: `../requirements/traceability.md` + `../status/change-history.md`.

## Original requirement → final implementation map

| # | Original (brief §) | Feasibility | Modification | Final implementation | Depends on | Phase | Testing | Docs touched |
|---|---|---|---|---|---|---|---|---|
| 1 | TCP server :4370, packet parser/builder, checksum, session handshake (3.1–3.3, 6-P1) | FF | Checksum strategy (ADR-005); frame reassembly across TCP segments | `protocol/packet.ts`, `transport/tcp-server.ts`, golden-byte tests | — | 1 | Unit + oracle connect/getInfo | protocol-engine, ADR-005 |
| 2 | CONNECT/AUTH/EXIT + comm key (3.4, 8-Q3) | FF | AUTH optional (backend has no key); 1.3.0 has no UNAUTH retry path → never send UNAUTH to oracle unless key configured | session handlers + makeCommKey module | #1 | 1 | Unit (key hash vectors) + integration | client-libraries §1.3, FR-2 |
| 3 | Version / free sizes / time / options (3.4) | FF | getInfo corrected to FREE_SIZES@24/40/72; **372-day time codec** added | info handlers + `protocol/time.ts` | #1 | 1 | Round-trip codec vectors vs driver values | zk-protocol-notes §4–6 |
| 4 | Seeded store (6-P1) | FF | Time codec applied to seeds | `device/store.ts` + `seed/` | #1 | 1–2 | Store unit tests | data-model |
| 5 | User/attendance reads w/ chunked flow (3.4, 6-P1) | FM | Direct `CMD_DATA` burst default (oracle); chunked behind config; both `DATA_WRRQ` and `DB_RRQ`/`ATTLOG_RRQ` styles | read-path state machine | #1–4 | 2 | Oracle `getUsers/getAttendances` round-trips; chunk-mode test | protocol-engine §3, ADR-004 |
| 6 | USER_WRQ / DELETE_USER two-way sync (6-P2) | FF | Accept backend's tag-less 72-B record and pyzk's 73-B variant | user handlers | #4 | 2 | Oracle `setUser/deleteUser` → re-read | client-libraries §2.2 |
| 7 | Template placeholders (3.4) | FM | Envelope pinned to driver logic (6-B entry header, probe-by-error) | template handlers | #5 | 2–3 | Oracle `getFingerCount/getFingerTemplate/setFingerTemplate` | FR-8 |
| 8 | Realtime events (6-P2) | FF | Per-session registration + in-flight queueing (ADR-006) | event module | #5 | 3 | Registered/unregistered/in-flight matrix | ADR-006, api-and-events |
| 9 | Punch trigger (6-P2) | FF | REST + CLI shapes fixed | `web/routes`, `scripts/punch.mjs` | #4 | 3 | API tests + e2e | api-and-events |
| 10 | Web device face + WS + bezel (5, 6-P3) | FF | Menu reduced to visual skeleton | `public/`, `web/ws.ts` | #8 | 4 | e2e punch→screen→DB; visual QA checklist | web-ui |
| 11 | LCD mirroring (3.4) | FF | — | lcd handlers + WS | #10 | 4 | Manual + e2e | web-ui §2 |
| 12 | SQLite persistence (7) | FF | Behind `DeviceStore` interface | `device/sqlite-store.ts` | #4 | 5 | Store adapter tests | data-model §5, ADR-008 |
| 13 | Seed week + README + demo video (6-P4) | FF | README lives in app repo; "why" story in docs | seed profiles, app README | all | 5 | Fresh-clone walkthrough | change-history |
| 14 | UDP support (8-Q2) | TR | **Dropped from v1** — backend is TCP-only `[V]` | — (future work) | — | — | — | ADR-003, future-work |
| 15 | "Indistinguishable from real device" (§1) | FM | Reframed honestly; deviations listed | NFR-08 + known-limitations | — | — | — | known-limitations |
| 16 | *(owner direction 2026-09-15)* Use phone's built-in / in-display biometric sensor for capture | **NF** — TEE/Secure Enclave isolation; no capture or enrollment API exists `[V]` | Ruled out permanently; nearest intent preserved via camera/USB capture | ADR-009 capture stations (camera first, scanner later) | — | — | Documented boundary (no test possible) | biometrics research §2, known-limitations |
| 17 | *(owner direction)* Real fingerprint enrollment + matching → "ZKTeco substitute" | **FM** — ISO/IEC 19794-2 templates + out-of-process matcher; no usable Node AFIS exists `[V]` | Matcher isolated behind a 3-endpoint HTTP/JSON sidecar; capture normalized to one format; ZK wire surface frozen | `biometric/` module (enroll, identify, verify), ADR-009/010/011, REST v2 | #4 (store), ADR-010 spike | 6 | Spike criteria (10/10 identify, `< 200 ms`); oracle template call patterns unchanged | biometric-core, ADR-010, ADR-009 |
| 18 | Camera-based touchless capture ("zero hardware") | **FM** — genuinely functional, demo-grade quality (lighting/pose sensitive) | Quality gate + actionable rejection reasons; quality bar documented, not hidden | `capture/camera-station.ts` + `/enroll`,`/scan` PWA pages | #17 | 7 | Scripted UX checks; human-judged quality assessment recorded | capture-stations, biometrics research §3, known-limitations |
| 19 | Android + USB-OTG fingerprint scanner | **DE** `[A]` — needs a purchased module (≈₱1–2.5k) and per-module integration | Deferred to a second adapter; nothing upstream depends on it. **Owner decision 2026-09-16:** deferred until hardware allowance; Phase 7/8 proceed camera-only | `capture/otg-station.ts` + small Android client (design in capture-stations §3) | #17, hardware | future | Module returns data usable by the matcher (1 module validated) | capture-stations §3, biometrics research §3 |
| 20 | Biometric data protection (privacy/legal, not in brief) | **FF** (obligation, not a feature) | Encrypt at rest (AES-256-GCM), consent-gated enrollment, no images persisted, logs/API redacted, delete/purge paths | `BiometricStore`, key config, NFR-10/11 | #17 | 6 | Ciphertext-at-rest assertion; leak tests; purge test | ADR-011, security, NFR-10/11 |

## Resolved open questions (brief §8)
1. **Library:** `node-zklib@1.3.0` from npm (GitHub master is an unpublished rewrite — do not use as reference) `[V]`.
2. **TCP/UDP:** TCP only (driver bypasses UDP fallback) `[V]`.
3. **Comm key:** none; AUTH optional, default off `[V]`.
4. **Visual depth:** idle + verify screens are the portfolio core; menu skeleton later (roadmap Phase 4).

## Dependency graph (build order)
```text
codec/session (#1) → info+time (#3) → store/seed (#4) → reads (#5) → users (#6) → templates (#7)
                                                                    ↘ events (#8) → punch API (#9) → UI (#10) → LCD (#11)
store (#4) ────────────────────────────────────────────────────────► SQLite (#12) ──► polish (#13)
```
