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

## Deliberately not planned
- TLS/auth on the simulator's own HTTP API (demo tool; see security.md).
- Real biometric matching (impossible — see known-limitations #1).
- Cloud deployment of the protocol engine (plaintext LAN protocol).
