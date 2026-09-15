# Biometric Research — Capture, Templates, Matching

**Date:** 2026-09-15 · **Status:** complete for planning; two items flagged `[A]` awaiting a Phase-6 spike.

The original brief treats fingerprints as *placeholder blobs* (FR-8) and states the simulator "never claims real biometric verification". Owner direction on 2026-09-15 upgrades the ambition: the project should be able to **enroll real fingerprints and match them** so the app is not only a simulator but a **ZKTeco substitute**. This document records what that can honestly mean, what is impossible, and the evidence for each verdict. Feasibility roll-up: see the biometric section of `feasibility-report.md`; decisions: ADR-009, ADR-010, ADR-011.

## 1. Questions this research answers

1. Can we read raw fingerprint data out of a phone's built-in sensor (classic or in-display)?
2. Can we enrol a fingerprint programmatically on a phone?
3. Can a phone/PC act as a real capture station — with zero extra hardware, or with a cheap scanner?
4. What is actually inside a ZKTeco fingerprint template, and is it portable?
5. What matching engine can a Node.js service use? Is there one in the npm ecosystem?
6. What privacy obligations apply once real biometrics are stored?

## 2. Finding 1 — Phone built-in sensors cannot be used for capture (verdict: **not feasible, by platform design**) `[V]`

| Claim | Evidence |
|---|---|
| Apps can only invoke a **system dialog** (`BiometricPrompt` on Android, `LAContext`/`evaluatePolicy` on iOS); the API returns a **boolean result** (plus an optional `CryptoObject`), never image or template data. | Android API reference for `BiometricPrompt`/`BiometricManager` and Apple `LocalAuthentication` documentation (read 2026-09). The published API surface contains no capture, image, template, or feature-vector accessor. |
| There is **no public enrollment API** for apps on either platform. Enrollment is a Settings/system flow owned by the platform. | Same API references — only authentication entry points are exposed. |
| **In-display / under-screen sensors are the same story**, not a loophole. | Optical and ultrasonic FOD sensors are driven by the trusted stack; the sensor → match → boolean path stays inside the secure environment. Android's biometric HAL is owned by the system fingerprint service; there is no app-facing raw-capture path. |
| iOS is stricter (Secure Enclave; Touch ID/Face ID data never leaves it). | Apple platform security documentation. |
| The *reason* is isolation mandated at the platform level (biometric data is not accessible outside the trusted environment). | `[A]` — design intent is well documented across Android security material; if a formal citation is ever needed, quote the Android CDD's biometric section at that time (validation step: read the CDD section in force, ~1 h). |

**Why this is not an effort problem:** the restriction is the security feature. Even a device with a broken/absent OS restriction would still be *the owner's* finger, not the employee's — identity mapping is a second, independent impossibility.

**Research-level bypasses** (sensor-bus manipulation, rooted devices, physical access) exist in academic literature but require root/physical access, are device-specific, and would not be a legitimate or portable application path. Explicitly rejected.

**Consequence:** the built-in-sensor route is struck from scope permanently — recorded in `../status/known-limitations.md`. This is the one thing in the substitute ambition that is *fundamentally* impossible rather than merely difficult.
## 3. Finding 2 — Capture routes that *do* work

| Route | How it works | Quality | Hardware cost | Verdict |
|---|---|---|---|---|
| **Phone/PC camera, touchless** ★ chosen first | Employee presses a finger on a plain high-contrast surface; the camera photographs it; software segments the ridge pattern and extracts minutiae → ISO template | **Demo-grade: genuinely functional, clearly below a capacitive sensor** (lighting/pose sensitive; needs a quality gate) | ₱0 | Feasible — ships as a web page, no app-store release |
| **Android + USB-OTG fingerprint scanner** | Cheap sensor module (ZFM-20 / R305 / R307 / R30x class) plus a USB-serial bridge (CP2102/CH340/FTDI) driven through the Android USB Host API; or a vendor reader (SecuGen, Futronic) with its own Android SDK | **Real sensor quality** (contact, high resolution) | ~₱1 000–2 500 | Feasible `[A]` — needs one module obtained to validate integration (Phase 7) |
| **PC-connected USB scanner** | Same idea on desktop via Node serial/USB | Real sensor quality | ~₱1 000–3 000 | Feasible `[A]` — convenience option, not required |
| Phone built-in / in-display sensor | — | — | — | **Not feasible** (§2) |
| Import an existing ISO template file | Load templates produced elsewhere, for seeding | Depends on source | — | Feasible — useful for demo seeding without a live capture |
| A ZKTeco device's own template | `CMD_USERTEMP_RRQ` returns a blob | Real, but proprietary format | needs a real terminal | **Store/copy yes, match no** (§4) |

Both `[A]` rows share one validation step, scheduled in Phase 7: obtain one module, capture a finger, confirm the returned data is usable by the matcher. **Nothing in Phases 1–6 depends on the outcome** — the camera route is plan A.

## 4. Finding 3 — What a ZKTeco template is, and its portability

| Claim | Confidence |
|---|---|
| A template is a **minutiae/feature blob, not an image** — compact (hundreds of bytes to a few KB), which is exactly why the BITS driver uses a `> 100 B` heuristic to tell "real template" from "empty slot". | `[V]` — driver logic, `client-libraries.md` §2.2 |
| Templates live **per `(uid, finger)`** on the device and are fully **readable/writable over the protocol** (`CMD_USERTEMP_RRQ` + write flow, FR-8). Our simulator must therefore treat template bytes as **opaque** — store and return them verbatim. | `[V]` — `zk-protocol-notes.md` §7, driver code |
| Matching a ZKTeco template requires **the same algorithm family as that firmware** (ZKFinger-family templates, versioned by algorithm generation). ZKTeco publishes no converter between these formats. | `[A]` — community/industry understanding; validating it would need a real terminal plus vendor documentation, which is out of scope. Treated as a **permanent documented boundary**, not a to-do. |
| ISO/IEC 19794-2 (fingerprint minutiae data) is the published, interoperable alternative; SourceAFIS consumes it directly. | `[V]` — SourceAFIS project documentation/GitHub |

**Consequence, stated plainly:** our engine enrols **ISO 19794-2** templates. A real ZKTeco terminal cannot match those, and we cannot match a real terminal's templates. "Substitute" is therefore scoped honestly as **a real biometric terminal within our own ecosystem (capture stations ↔ simulator ↔ BITS)** — not a template donor to physical hardware. Recorded in `../status/known-limitations.md`.
## 5. Finding 4 — Matching-engine landscape

Answer "who is this finger?" (1:N) or "is this employee N?" (1:1) requires a **minutiae matcher**. Search results (npm registry + GitHub project search, 2026-09):

| Option | Verdict | Notes |
|---|---|---|
| **SourceAFIS (JVM, Apache-2.0)** ★ recommended | `[V]` mature, actively ported, consumes ISO/IEC 19794-2 and ANSI/INCITS 378 | The default choice behind ADR-010's sidecar interface |
| .NET port of SourceAFIS | `[A]` exists, less mature | Viable alternative if a .NET runtime is already present |
| C/C++ engine compiled to WASM | `[A]` research-grade builds | Rejected: hard to debug, crash takes the protocol engine down |
| **On-sensor matching** (module returns match/nomatch) | `[A]` simpler integration, but loses template control | Rejected as the *primary* design: we must keep template **blobs** for FR-8 wire compatibility (BITS reads/writes them) |
| Cloud biometric API | `[A]` would work technically | Rejected: sends real biometrics off-device, contradicts ADR-011 and the local-LAN deployment model |
| **Pure Node.js / npm matcher** | **`[V]` does not exist** | A registry search finds no maintained fingerprint *matcher*. Note the trap: `fingerprintjs` is **browser/device fingerprinting**, a completely unrelated package — recorded so nobody re-searches it. |

**Consequence:** the matcher will be a non-Node component. This is exactly why ADR-010 (see `../decisions/ADR-010-matching-engine.md`) isolates it behind a three-endpoint HTTP/JSON interface, and why ADR-009 normalizes every capture source to one template format before it reaches the core.

## 6. Finding 5 — Privacy and legal obligations (new, non-optional)

Real fingerprints are **sensitive personal information**, not ordinary demo data. The moment we store a real template, the project acquires obligations it did not have while blobs were synthetic `[V]` (Philippine Data Privacy Act, RA 10173 classifications; GDPR Art. 9 if ever EU-facing — general framing, not legal advice):

| Obligation | Consequence for this project |
|---|---|
| **Consent** — explicit, informed, purpose-bound | Enrolment must be an explicit action with a stated purpose; a written consent step before the first capture |
| **Purpose limitation** | Templates are used for attendance identification only — never for anything else, never shared |
| **Security** | Encryption at rest + key management (ADR-011); no plaintext templates in logs, backups, or the UI |
| **Retention & deletion** | A delete path per employee; templates die with the employee record; demo datasets contain **no real person's biometrics** |
| **Transparency** | A visible statement in the UI/README about what is captured and stored |
| **Data minimisation** | Store **templates**, not fingerprint images — images are deleted after extraction |

**This changes the project's data posture:** NFR-07 currently says "no real personal data" is ever committed. That stays true for the repository, but must be restated as *"no real personal data in the repo; real biometrics, when used, are encrypted, consented, and never committed"* — see `../requirements/non-functional.md` (NFR-10/NFR-11) and `../operations/security.md`.

## 7. Consequences for the architecture we are about to build

1. **Nothing in Phases 1–5 blocks on this research.** The protocol engine, store, event bus and UI are unchanged; biometrics enter through a new upstream layer.
2. **Two small forward-compatibility hooks** must exist from the start (cheap now, expensive later):
   - `FpTemplate.format` (`synthetic` | `iso19794-2`) — see `../architecture/data-model.md`.
   - `BiometricStore` as an interface alongside `DeviceStore`, even while its only implementation is the synthetic one.
3. **`CMD_STARTENROLL` becomes meaningful** — it can be wired to the enrollment service instead of only ACKing (currently known-limitation #6).
4. **A new REST surface is added, not a changed one** — API v1 (FR-10) stays byte-for-byte as documented; biometric endpoints are API v2 (FR-18). No breaking changes for BITS or the demo scripts.

## 8. Open questions and how they will be closed

| # | Question | Closing step | When |
|---|---|---|---|
| Q1 | Does the camera route really yield matcher-usable templates from a phone photo? | Phase-6 spike: 10 fingers captured by camera → enroll → 1:N identify round-trip | Phase 6 |
| Q2 | Sidecar latency/robustness under demo load | ADR-010 spike acceptance criteria (`< 200 ms`, 10/10) | Phase 6 |
| Q3 | Actual cost/behaviour of a specific USB-OTG module | Buy one (~₱1–2.5k), capture, confirm data usable | Phase 7 |
| Q4 | Practical false-accept/false-reject thresholds for demo use | Tune on the spike dataset; record thresholds in the configuration doc (not hard-coded) | Phase 6 |
| Q5 | Do we need NFIQ-style quality scoring, or is a simple blur/contrast gate enough? | Start simple; add only if the spike shows camera rejects are too noisy | Phase 6 |
| Q6 | Whether a real terminal could ever accept our templates | Would require vendor documentation or reverse engineering — **explicitly not pursued** | never (boundary documented) |

