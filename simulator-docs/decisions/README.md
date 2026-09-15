# Architecture Decision Records

Accepted ADRs are **immutable** — to change a decision, write a new ADR that supersedes the old one and update links. Format:

```md
# ADR-NNN: Title
Status: accepted | superseded by ADR-MMM
Date: YYYY-MM-DD
## Context
## Decision
## Consequences (including negative)
```

## Index

| ADR | Title | Status |
|---|---|---|
| [ADR-001](./ADR-001-single-process-layered.md) | Single-process, four-layer Node.js app | accepted |
| [ADR-002](./ADR-002-typescript-engine.md) | TypeScript for the protocol engine | accepted |
| [ADR-003](./ADR-003-tcp-only.md) | TCP-only transport, UDP deferred | accepted |
| [ADR-004](./ADR-004-direct-data-burst.md) | Reads answered with direct `CMD_DATA` bursts; chunked flow config-gated | accepted |
| [ADR-005](./ADR-005-checksum-strategy.md) | Checksum: lenient inbound (both variants), classic outbound default | accepted |
| [ADR-006](./ADR-006-realtime-gating.md) | Realtime events: per-session registration + in-flight queueing | accepted |
| [ADR-007](./ADR-007-vanilla-web-ui.md) | Device-face UI: vanilla HTML/CSS/JS, no framework | accepted |
| [ADR-008](./ADR-008-in-memory-store.md) | In-memory store default; SQLite behind an interface | accepted |
| [ADR-009](./ADR-009-capture-source-abstraction.md) | Capture sources behind one pluggable station interface | accepted |
| [ADR-010](./ADR-010-matching-engine.md) | Matching engine out-of-process (sidecar); npm has no usable AFIS | **proposed** (Phase-6 spike) |
| [ADR-011](./ADR-011-biometric-data-protection.md) | Biometric data: encrypt at rest, consent-gated, deletable | accepted (applies from Phase 6) |

## Standing decision conventions

- The **pinned oracle** (`node-zklib@1.3.0` + patch, driven like `C:/bits/backend/src/shared/lib/zk-driver.ts`) is the compatibility criterion for every protocol decision.
- Protocol-visible behavior changes require a new ADR; internal refactors do not.
- An ADR may be created as **proposed** when research identifies the need but a spike must confirm the choice (example: ADR-010). It is referenced as proposed in the index and only marked accepted when its stated acceptance criteria are met.
- ADRs that add a capability but no protocol change (ADR-009, ADR-011) state explicitly that the ZK wire surface is untouched.
- Every ADR cites its research (`../research/*`) — no decisions from vibes.
