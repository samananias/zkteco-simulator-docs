# Requirements Traceability

Maps every requirement of the original brief to its feasibility verdict, current requirement, development phase and status. This is the anti-drift document: when implementation deviates, update the row and `../status/change-history.md`.

Legend — Feasibility: **FF** Fully Feasible · **FM** Feasible With Modification · **TR** Technically Risky · **DE** Dependent on External (hardware/software) · **NF** Not Feasible as Proposed. Status: ⬜ pending · 🔨 Phase n · ✅ done · **(ruled out)** = permanently excluded, boundary documented.

**Scope extension (2026-09-15):** rows below the brief-derived set come from owner direction to evolve the simulator into a **ZKTeco substitute** (real biometrics). Evidence: `../research/biometrics.md`; design: `../architecture/biometric-core.md`; decisions: ADR-009…ADR-011.

| Brief item (§) | Verdict | Current requirement | Phase | Status |
|---|---|---|---|---|
| TCP server :4370, raw sockets (3, 4) | FF | FR-1 | 1 | ⬜ |
| Packet codec + checksum + session/reply (3.1–3.3) | FF (checksum variant open → ADR-005) | FR-1, FR-2 | 1 | ⬜ |
| CONNECT / AUTH / EXIT (3.4) | FF; AUTH optional (backend uses no comm key) | FR-2 | 1 | ⬜ |
| GET_VERSION / FREE_SIZES / TIME / OPTIONS (3.4) | FF; brief's "version feeds getInfo()" corrected — getInfo uses FREE_SIZES + options | FR-3, FR-4 | 1 | ⬜ |
| Seeded store of users + logs (Phase 1) | FF; time codec quirk added (372-day year) | FR-13, FR-12 | 1–2 | ⬜ |
| User/attendance reads with chunked flow (3.4) | FM — oracle's primary path is direct `CMD_DATA` bursts; chunked flow kept as option (ADR-004) | FR-6, FR-7 | 2 | ⬜ |
| ENABLE/DISABLEDEVICE (Phase 1) | FF | FR-5 | 1 | ⬜ |
| End-to-end validation vs real backend (Phase 1) | FF (oracle = backend's exact lib version) | NFR-01 | 1–3 | ⬜ |
| Realtime events (REG_EVENT + EF_ATTLOG) (Phase 2) | FF; per-session gating + in-flight queueing added | FR-9 | 3 | ⬜ |
| Punch trigger (HTTP/CLI) (Phase 2) | FF | FR-10 | 3 | ⬜ |
| USER_WRQ / DELETE_USER two-way sync (Phase 2) | FF; 72-B layout pinned to backend driver (no tag byte) | FR-6 | 2 | ⬜ |
| Web device-face UI + WS sync (Phase 3) | FF | FR-11 | 4 | ⬜ |
| Bezel presentation (5, Phase 3) | FF | FR-11 | 4 | ⬜ |
| Realistic seed data (Phase 4) | FF | FR-13 | 5 | ⬜ |
| README + demo script/video (Phase 4) | FF | — (app repo README; video manual) | 5 | ⬜ |
| SQLite persistence (7) | FF (behind store interface) | FR-12 | 5 | ⬜ |
| UDP support ("if needed", 8-Q2) | TR — backend is TCP-only by design; UDP record sizes differ | Out of scope v1 | — | ⬜ |
| Fingerprint template placeholder blobs (3.4) | FM — synthetic blobs, structured per driver's read/probe code | FR-8 | 2–3 | ⬜ |
| Cosmetic commands: UNLOCK/LCD/RESTART… (3.4) | FF | FR-5, FR-11 | 1, 4 | ⬜ |
| Backend library/version pin (8-Q1) | Resolved — `node-zklib@1.3.0` (npm) + `patch-package` fix, TCP-only wrapper | NFR-01 | 1 | ✅ researched |
| Comm key configured? (8-Q3) | Resolved — none; AUTH optional (default off) | FR-2 | 1 | ✅ researched |
| Depth of visual fidelity (8-Q4) | Idle + verify screens first; menu skeleton later | FR-11 | 4 | ⬜ |
| "Indistinguishable from real device" (1) | FM — reframed honestly: protocol-compatible at command level + visual replica | NFR-08, limitations doc | — | ⬜ |
| Phase 3 menu system (6-Phase 3) | Partially in scope — icon-grid skeleton, not full menu logic | FR-11 | 4 | ⬜ |
| Phone built-in / in-display sensor as capture device (owner direction 2026-09-15) | **NF** — platform isolation (TEE / Secure Enclave): no raw capture API, no app-facing enrollment API `[V]` | — (permanent boundary, documented) | — | (ruled out) |
| Real enrollment + matching (simulator → ZKTeco substitute) | **FM** — requires capture station + ISO templates + out-of-process matcher (no Node AFIS exists `[V]`) | FR-14, FR-15 | 6 | ✅ core (2026-09-16): REST enrollment, 1:N identify, 1:1 verify live behind ADR-010 sidecar; N-sample capture ceremony completes with capture stations (Phase 7) |
| Camera touchless capture (zero hardware) | **FF for demo-grade quality** (quality bar documented, not hidden) | FR-16 | 7 | ⬜ |
| Android + USB-OTG scanner capture | **DE** `[A]` — depends on a purchased module + integration validation | FR-16 | 7 | ⬜ |
| Encrypted biometric storage + consent/retention | **FF** | FR-17, NFR-10, NFR-11 | 6 | ✅ (2026-09-16): AES-256-GCM store, consent-gated enroll, ciphertext-at-rest + purge tests green |
| Open REST surface for non-ZK integrators | **FF** | FR-18 | 6 | ✅ core (2026-09-16): `/api/biometric/status·enroll·identify·verify·remove·purge`; `/capture` lands with capture stations (Phase 7) |
| Template interop with real ZKTeco hardware | **NF** — proprietary ZKFinger-family formats, no public converter `[A]` | — (permanent boundary) | — | (ruled out) |
