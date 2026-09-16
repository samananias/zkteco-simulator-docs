# Future Work

Ideas and deferred items — none are commitments. Promote an item by giving it roadmap phases + ADRs.

## Protocol / compatibility
- **UDP transport** — full `dgram` implementation with the 28/16/8-B UDP record variants; only if a UDP-first client matters (ADR-003).
- **Real-device validation session** — borrow any ZKTeco terminal once; resolve the checksum-variant question empirically and confirm edge behaviors (R1 closure).
- **Second-oracle CI** — run pyzk + zklib-ts integration suites alongside node-zklib for stronger "any client works" claims.
- **Chunked mode by default for big datasets** — if demo data ever exceeds ~64 KB (ADR-004).

## Device simulation depth
- **Enrollment ceremony simulation** — multi-step `CMD_STARTENROLL` → `EF_ENROLLFINGER` event sequence so backend-driven enrollment demos fully light up.
- **Failure injection** — config/REST knobs to simulate device errors (timeout before reply, `CMD_ACK_ERROR` storms, half-open sockets) for backend resilience testing.
- **Multi-device instances** — one process, N simulated devices (multi-terminal dashboards).
- **Operation log (`CMD_OPLOG_RRQ`)** — cosmetic admin-action records.

## UI
- **Full menu system** — working sub-screens (user list, attendance search) rendered from the store (ADR-007 revisit if this grows).
- **iFace theme** — second bezel/screen skin to match face-recognition terminals.
- **Screenshot/video capture button** — one-click portfolio assets.

## Operations
- **Docker image** + compose file alongside the BITS stack.
- **GitHub Actions CI** (test matrix; see testing-strategy §3).
- **Packaged releases** (single-binary via `node --experimental-sea` or `pkg`) for share-anywhere demos.

## Biometrics (Phase 6+ — planned, not v1)
- **Android + USB-OTG scanner station** — *deferred by owner decision 2026-09-16 until hardware allowance* (module ≈₱1–2.5k); design preserved in `../architecture/capture-stations.md` §3 — it plugs into the same capture HTTP contract with zero core changes when funded (that interchangeability is ADR-009's demonstrated value).
- **Enrollment ceremony over the protocol** — wire `CMD_STARTENROLL` (FR-8, currently ACK-only) to the real enrollment service and emit the `EF_ENROLLFINGER` event sequence, so backend-driven enrollment demos work end to end without any BITS change.
- **PC-connected USB scanner station** — third `CaptureStation` adapter (research §3); convenient for desktop demos.
- **Multi-finger / multi-modal** — more than one finger per employee; eventual face capture on terminals that have it (a separate research effort; cameras are far more accessible than fingerprint sensors).
- **Liveness / anti-spoofing** — if the substitute is ever used with real employees at any scale, presentation-attack detection becomes a genuine requirement (currently an explicit non-goal, known-limitations #17).
- **Template export/import (ISO 19794-2)** — portable backup so a deployment can migrate between machines without re-enrolment.
- **OpenAPI spec for REST v2** — machine-readable contract for non-ZK integrators.

## Deliberately not planned
- TLS/auth on the simulator's own HTTP API (demo tool; see security.md).
- **Built-in / in-display phone sensor capture** (platform-impossible — known-limitations #14).
- **Template interop with real ZKTeco firmware** (proprietary formats, no published converter — known-limitations #15).
- **Cloud biometric APIs** (sends templates off-device; ADR-010/ADR-011).
- Cloud deployment of the protocol engine (plaintext LAN protocol).
