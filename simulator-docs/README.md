# ZKTeco Simulator App — Documentation Index

**Project in one paragraph:** a software simulator of a ZKTeco standalone biometric attendance terminal. It speaks the same binary protocol a real device speaks on TCP port 4370, so the existing BITS attendance backend (Node.js, `node-zklib@1.3.0`) connects **with zero code changes** — and it renders a bezel-framed web UI that visually mimics the physical device for portfolio demos. Biometric capture is simulated (a punch is triggered, not scanned); everything else is protocol-faithful.

**Current status:** Phase 0 complete — research, feasibility analysis, architecture and documentation foundation done. Implementation begins at Phase 1 (see `plan/roadmap.md`).

**Validation oracle (pinned):** `node-zklib@1.3.0` as installed in `C:/bits/backend` (plus its `patch-package` control-flow fix), driven exactly the way `C:/bits/backend/src/shared/lib/zk-driver.ts` drives it.

## Document map

| Folder | Files | Read when |
|---|---|---|
| `requirements/` | `00-original-brief.md` (archived spec, immutable), `functional.md`, `non-functional.md`, `traceability.md` | You need the current committed scope, or want to trace any brief item to its fate |
| `research/` | `zk-protocol-notes.md`, `client-libraries.md`, `feasibility-report.md` | You touch anything wire-level, or want to know why something is built a certain way |
| `architecture/` | `system-architecture.md`, `protocol-engine.md`, `data-model.md`, `web-ui.md`, `api-and-events.md` | You implement or review a module |
| `decisions/` | `README.md` (ADR index + template), `ADR-001` … `ADR-008` | You want the reasoning behind a choice, or need to make a new decision |
| `plan/` | `roadmap.md`, `revised-project-plan.md`, `testing-strategy.md` | You start a phase or write tests |
| `operations/` | `setup.md`, `configuration.md`, `troubleshooting.md`, `security.md` | You run, configure, or debug the system |
| `process/` | `development-workflow.md`, `documentation-policy.md` | You make any change |
| `status/` | `known-limitations.md`, `risks.md`, `change-history.md`, `future-work.md` | You wonder what the system *can't* do, what might bite us, or what changed |

## Quick facts (all verified — see `research/` for sources)

- Protocol: ZKTeco standalone binary protocol, TCP port 4370; frame = magic `50 50 82 7D` + u32-LE payload size + 8-byte header (command, checksum, session-id, reply-number) + data.
- Device "clock" uses a **372-day-year calendar** (31-day months) — seconds since 2000-01-01.
- The BITS backend polls attendance on a ~30 s scheduler and **does not** consume realtime event packets; events are still implemented (gated per session) for the web UI and future clients.
- The backend talks **TCP only** (it deliberately bypasses node-zklib's UDP fallback) and uses **no comm key**.
- Existing open-source ZKTeco simulators: effectively none — this project is prior-art-light by design.

## Conventions

- Claims tagged `[V]` (verified, source cited) or `[A]` (assumption + validation step).
- ADRs are immutable once accepted; supersede with a new one.
- `status/change-history.md` gets one entry per phase/decision; docs updates are part of Definition of Done (`process/development-workflow.md`).
