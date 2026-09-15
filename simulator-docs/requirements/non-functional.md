# Non-Functional Requirements

**v1.0 — Phase 0.**

| ID | Requirement | Rationale / measure |
|---|---|---|
| NFR-01 | **Client compatibility** — the BITS backend connects and operates with zero source changes. All protocol features are validated against the pinned oracle before "done". | The project's core promise. Gate: integration test suite (see `../plan/testing-strategy.md`). |
| NFR-02 | **Robustness** — malformed/unknown packets are logged and answered with `CMD_ACK_UNKNOWN` (65535), never crash the process or poison the session; abrupt disconnects clean up session state; reconnect storms (backend connects per batch) are handled. | Real client code is defensive but quirky; the simulator must be the stable side. |
| NFR-03 | **Performance** — command replies < 10 ms locally; dataset reads for ≤ 10 000 records complete within the client's 1 s quiescence window (send the whole dataset as one burst). | Demo and test loads are small; node-zklib resolves reads after 1 s of socket silence. |
| NFR-04 | **Portability** — runs on the Windows dev box and any Node ≥ 20 host; no admin rights; all ports configurable (4370 TCP, HTTP, WS). Caveat: Windows may reserve port ranges (Hyper-V) — startup check + clear error (see `../operations/troubleshooting.md`). | Reproducible demos on any laptop. |
| NFR-05 | **Maintainability** — layered modules (transport / protocol / device / web), typed record structs with byte-offset constants co-located with decoders, ADR process, docs updated with every change. | Byte-layout code rots silently; structure is the countermeasure. |
| NFR-06 | **Testability** — unit tests (codec, checksum, records, time codec with golden byte vectors), integration tests (pinned oracle client), one scripted end-to-end demo flow. | Validation by construction, per the original brief. |
| NFR-07 | **Security posture** — LAN/localhost tool only; the ZK protocol has no TLS and weak auth (documented); demo data contains **no real personal data**; comm key supported but optional; HTTP/WS interface must not be exposed publicly. | Honest about what the underlying protocol does and doesn't protect. |
| NFR-08 | **Honest scope** — the simulator never claims real biometric verification; limitations are documented in `../status/known-limitations.md` and surfaced in the project README. | The portfolio value is protocol engineering, not pretending to be hardware. |
| NFR-09 | **Observability** — structured per-session packet log (command in/reply out, hex on demand) to make integration debugging tractable; log level configurable. | Protocol bugs are invisible without wire-level logging. |
