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

## Standing decision conventions

- The **pinned oracle** (`node-zklib@1.3.0` + patch, driven like `C:/bits/backend/src/shared/lib/zk-driver.ts`) is the compatibility criterion for every protocol decision.
- Protocol-visible behavior changes require a new ADR; internal refactors do not.
- Every ADR cites its research (`../research/*`) — no decisions from vibes.
