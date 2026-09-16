# ZKTeco Simulator App — Documentation Index

**Project in one paragraph:** a software simulator of a ZKTeco standalone biometric attendance terminal. It speaks the same binary protocol a real device speaks on TCP port 4370, so the existing BITS attendance backend (Node.js, `node-zklib@1.3.0`) connects **with zero code changes** — and it renders a bezel-framed web UI that visually mimics the physical device for portfolio demos. In v1, biometric capture is simulated (a punch is triggered, not scanned); everything else is protocol-faithful.

**Planned scope extension (decided 2026-09-15):** the project is intended to become a **usable ZKTeco substitute** — real fingerprint enrollment and real matching (camera capture first, USB-OTG scanner later), while the ZK protocol surface stays frozen so BITS keeps working unmodified. Phases 6–8; research in `research/biometrics.md`, design in `architecture/biometric-core.md`. **Phases 1–5 are unchanged and come first.**

**Current status:** Phase 0 complete — research (protocol + biometrics), feasibility analysis, architecture and documentation foundation done. Implementation begins at Phase 1 (see `plan/roadmap.md`).

**Validation oracle (pinned):** `node-zklib@1.3.0` as installed in `C:/bits/backend` (plus its `patch-package` control-flow fix), driven exactly the way `C:/bits/backend/src/shared/lib/zk-driver.ts` drives it.

**Companion application repository:** [`samananias/zkteco-simulator-app`](https://github.com/samananias/zkteco-simulator-app) — the simulator this knowledge base documents.

## Document map

| Folder | Files | Read when |
|---|---|---|
| `requirements/` | `00-original-brief.md` (archived spec, immutable), `functional.md`, `non-functional.md`, `traceability.md` | You need the current committed scope, or want to trace any brief item to its fate |
| `research/` | `zk-protocol-notes.md`, `client-libraries.md`, `feasibility-report.md`, `biometrics.md` | You touch anything wire-level, or want to know why something is built a certain way (or whether a biometric idea is actually possible) |
| `architecture/` | `system-architecture.md`, `protocol-engine.md`, `data-model.md`, `web-ui.md`, `api-and-events.md`, `biometric-core.md`, `capture-stations.md` | You implement or review a module |
| `decisions/` | `README.md` (ADR index + template), `ADR-001` … `ADR-011` | You want the reasoning behind a choice, or need to make a new decision |
| `plan/` | `roadmap.md`, `revised-project-plan.md`, `testing-strategy.md` | You start a phase or write tests |
| `operations/` | `setup.md`, `configuration.md`, `troubleshooting.md`, `security.md`, `deployment.md`, `guide.md` (live-gate walkthrough) | You run, configure, or debug the system |
| `process/` | `development-workflow.md`, `documentation-policy.md` | You make any change |
| `status/` | `known-limitations.md`, `risks.md`, `change-history.md`, `future-work.md` | You wonder what the system *can't* do, what might bite us, or what changed |

## Quick facts (all verified — see `research/` for sources)

- Protocol: ZKTeco standalone binary protocol, TCP port 4370; frame = magic `50 50 82 7D` + u32-LE payload size + 8-byte header (command, checksum, session-id, reply-number) + data.
- Device "clock" uses a **372-day-year calendar** (31-day months) — seconds since 2000-01-01.
- The BITS backend polls attendance on a ~30 s scheduler and **does not** consume realtime event packets; events are still implemented (gated per session) for the web UI and future clients.
- The backend talks **TCP only** (it deliberately bypasses node-zklib's UDP fallback) and uses **no comm key**.
- Existing open-source ZKTeco simulators: effectively none — this project is prior-art-light by design.
- Biometrics (Phase 6+): phone built-in/in-display sensors are **unusable by platform design** (TEE/Secure Enclave — no capture or enrollment API) `[V]`; capture is camera (demo-grade, zero hardware) or USB-OTG scanner (real quality) — `research/biometrics.md`.
- No usable fingerprint **matcher** exists in the npm ecosystem `[V]`; matching is isolated in an optional out-of-process sidecar (ADR-010), and our templates are ISO 19794-2 — **not** interchangeable with ZKTeco's proprietary firmware formats.

## Conventions

- Claims tagged `[V]` (verified, source cited) or `[A]` (assumption + validation step).
- ADRs are immutable once accepted; supersede with a new one.
- `status/change-history.md` gets one entry per phase/decision; docs updates are part of Definition of Done (`process/development-workflow.md`).
