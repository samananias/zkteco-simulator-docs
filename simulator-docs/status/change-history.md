# Change History

Reverse-chronological. One entry per phase, decision, or significant fix (see `../process/documentation-policy.md`).

---

## 2026-09-16 — Phase 6 started: ADR-010 matcher spike passed (sidecar accepted)

**What**
- Built the matcher sidecar (app repo `sidecar/MatcherSidecar.java`): SourceAFIS 3.18 + fingerprintio 1.3.0 behind a tiny localhost HTTP API; pinned jars vendored (`sidecar/lib/`, 27 MB, offline-safe for demos). Optional at boot per ADR-010 / NFR-12.
- Spike (Node harness, throwaway): 10 enrolled synthetic-template candidates across 5 identities / fingers 0–2 → **10/10 identify correct** with probes rotated ±2–6°, ±1–3 px position jitter, ±10° angle noise, 10 % minutia drop; same-finger scores 62.6–324.9 vs cross-finger max **4.3** (≈15× margin); impostor probes ≤ **2.1**; 1:1 verify **231.7** vs wrong-probe **0.236**; identify latency **33–51 ms** steady-state (< 200 ms criterion); cold boot **3.86 s** (< 10 s criterion).
- Template wire format pinned to **ISO 19794-2** (`FingerprintCompatibility` at the sidecar edge; SourceAFIS native only internally). `GET /synthetic` produces deterministic synthetic minutiae templates (NFR-07 — never real); the Phase-6 integration tests reuse it.

**Key discoveries (sidecar / SourceAFIS 3.18)**
- `new FingerprintTemplate(byte[])` parses **only** SourceAFIS's native format — ISO/ANSI must go through `FingerprintCompatibility.importTemplate()`; export via `exportTemplates(TemplateFormat.ISO_19794_2_2005, …)`.
- fingerprintio's minutia `angle` is the ISO 2°-units unsigned byte (degrees ÷ 2), not degrees.
- SourceAFIS 3.18 deprecated the fluent `new FingerprintImage().dpi().decode()` — use `new FingerprintImage(image, new FingerprintImageOptions().dpi(dpi))`.
- Synthetic ridge-field images (pure sinusoid patterns) extract ~18 mostly mask-boundary minutiae and do not survive rotation — image matching *quality* belongs to Phase 7 (capture quality gate); the spike validates the matcher on controlled synthetic minutiae instead. Recorded honestly in ADR-010.

**Decisions:** ADR-010 status → **accepted** (spike results recorded in the ADR; criterion 3 — app REST-layer cycle — recorded with the Phase-6 integration test before the gate closes).
**Next:** Phase 6 app layer — encrypted `BiometricStore` (ADR-011), matcher client, enrollment/identify/verify services, REST v2.

## 2026-09-15 — Phases 1–5 implemented; oracle gates green

**What**
- Full simulator implemented (app repo `2f71368`): protocol engine (TCP :4370), frame/checksum/time codecs, record codecs, in-memory + sqlite stores, seed profiles, realtime `EF_ATTLOG` with per-session gating, REST + WebSocket layer, device-face UI (bezel, clock, verify overlays, keypad, LCD), punch CLI.
- **25/25 tests green:** 11 golden-byte unit (checksum verified against an independent reference implementation), 11 oracle integration (real `node-zklib@1.3.0`: connect → getInfo → getUsers → getAttendances → GET_TIME → driver-style 72-B user write/delete → full FR-8 template write→read-back→empty-probe→delete → realtime event on REST punch → CLEAR_ATTLOG → UI smoke → ACK_UNKNOWN), 3 sqlite gate. CLI boot smoke-verified.

**Key discoveries (now in `../research/zk-protocol-notes.md` §10)**
- The oracle's read path resolves on the *first non-event chunk* — a leading ACK before the dataset deadlocks it. Direct-burst = one `CMD_DATA` frame, nothing else.
- `executeCmd` resolves the raw frame buffer → `getInfo` capacity offsets are frame-relative (payload 16/32/64).
- The chunked collector's per-chunk 8-byte sub-header arithmetic is now known (was open item).
- Seed employee IDs are `1001`–`1005` (RFC: matches the backend's realistic userId space).

**Docs updated:** protocol-engine read-path (verified rule), roadmap Phases 1–5 checked with evidence, traceability rows 1–16 ✅.

**Next:** Phases 6–8 (biometric substitute) per plan; Phase 6 starts with the ADR-010 matcher spike.

## 2026-09-15 — Scope extension: simulator → ZKTeco substitute (biometrics planned)

**What**
- Owner direction: the project should become a **usable ZKTeco substitute**, not only a simulator — real fingerprint enrollment and real matching — while keeping the ZK protocol surface frozen so BITS keeps working unmodified.
- Researched the feasibility of real biometrics end to end and recorded it in `../research/biometrics.md` (new): phone built-in and in-display sensors are **unusable by platform design** (TEE/Secure Enclave; no capture or enrollment API) `[V]`; the workable routes are **camera touchless capture** (zero hardware, demo-grade) and **USB-OTG scanners** (~₱1–2.5k, real quality); ZKTeco templates are proprietary minutiae blobs that can be **stored/copied but not matched** by our engine (`[A]`); **no usable fingerprint matcher exists in the npm ecosystem** `[V]`, so matching is isolated out-of-process.
- Added architecture: `../architecture/biometric-core.md`, `../architecture/capture-stations.md`.
- Added decisions: **ADR-009** capture-source abstraction (accepted), **ADR-010** matching engine out-of-process (**proposed** — Phase-6 spike), **ADR-011** biometric data protection (accepted, applies from Phase 6).
- Extended requirements: **FR-14…FR-18** (enrollment, identify/verify, capture stations, encrypted store, REST v2), **NFR-10…NFR-12** (privacy, encryption at rest, graceful degradation), traceability rows for the substitute scope (including the two permanently ruled-out items).
- Extended plans: roadmap **Phases 6–8**, revised-project-plan rows 16–20, testing-strategy **L5**, configuration vars (matcher URL, biometric key, thresholds), security posture for biometric data, risks **R10–R15**, known-limitations **#14–#18**, future-work biometric backlog.

**Why this is a scope *extension*, not a rewrite**
- Phases 1–5 are unchanged and remain the prerequisite: the frozen protocol surface is precisely what makes the result a *substitute* rather than a separate product.
- Only two zero-cost forward-compatibility hooks enter Phases 1–5: `FpTemplate.format` (`synthetic` \| `iso19794-2`) and `BiometricStore` as an interface alongside `DeviceStore`.
- The base demo and BITS compatibility never depend on any biometric component (NFR-12); the matcher sidecar is optional at boot.

**Permanently ruled out (documented, not deferred)**
- Phone built-in / in-display sensor as a capture device (platform isolation).
- Template interoperability with real ZKTeco firmware (proprietary formats).

**Next:** Phase 1 — protocol core (unchanged; see `../plan/roadmap.md`).

## 2026-09-15 — Repositories published

Both repositories published publicly on GitHub under the `samananias` organization:
[`zkteco-simulator-app`](https://github.com/samananias/zkteco-simulator-app) and
[`zkteco-simulator-docs`](https://github.com/samananias/zkteco-simulator-docs) (this repository).
Cross-links between the two READMEs updated to the published URLs; `.gitattributes` added to both.
From now on, every commit to `main` is published with a plain `git push`.

## 2026-09-15 — Phase 0: Foundations complete

**What**
- Analyzed the original brief; researched the ZKTeco standalone protocol against three independent sources (`adrobinoga/zk-protocol` spec, `pyzk`, `node-zklib@1.3.0` as installed in the BITS backend).
- **Pinned the validation oracle:** `node-zklib@1.3.0` (npm artifact, + backend's `patch-package` control-flow fix), driven exclusively over TCP by `C:/bits/backend/src/shared/lib/zk-driver.ts`; no comm key; realtime events unused by the backend; ~30 s polling scheduler.
- Established both repositories (`zkteco-simulator-app`, `docs` with `simulator-docs/`).
- Wrote the documentation foundation: requirements (functional/non-functional/traceability + archived brief), research (protocol notes, client libraries, feasibility report), architecture (system, engine, data model, web UI, API), decisions (ADR-001…008), plans (roadmap, revised plan, testing strategy), operations (setup, configuration, troubleshooting, security), process (workflow, documentation policy), status (limitations, risks, history, future work).

**Key findings recorded** (details in `../research/`)
- Wire format verified against a real captured packet; two checksum variants identified (±1) and neutralized by ADR-005.
- **Device time uses a 372-day calendar** (31-day months) — brief omission, would have corrupted all timestamps.
- Brief's read-command expectation (`CMD_DB_RRQ`/`CMD_ATTLOG_RRQ`) corrected: the oracle actually reads via `CMD_DATA_WRRQ` → direct `CMD_DATA` bursts; chunked flow kept behind config (ADR-004).
- Backend writes users as plain 72-byte records (no pyzk tag byte); template probe/read/write envelopes pinned to the driver's logic.
- No usable existing simulator exists in open source (search performed) — building our own is justified and is the stronger portfolio story.

**Decisions:** ADR-001…ADR-008 accepted (single process; TypeScript; TCP-only; direct-burst reads; checksum strategy; realtime gating; vanilla UI; memory store).
**Next:** Phase 1 — protocol core (see `../plan/roadmap.md`).
