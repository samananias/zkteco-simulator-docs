# Change History

Reverse-chronological. One entry per phase, decision, or significant fix (see `../process/documentation-policy.md`).

---

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
