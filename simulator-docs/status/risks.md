# Risks & Mitigations

| # | Risk | Likelihood | Impact | Mitigation | Status |
|---|---|---|---|---|---|
| R1 | Checksum variant mismatch with a strict client (real device firmware unknown) | Medium | Low (oracle unaffected `[V]`) | Accept-both inbound; classic outbound; config switch (ADR-005); golden-bytes test records oracle-emitted variant | Mitigated |
| R2 | node-zklib upgrade changes parsing (backend moves to master/zklib-ts) | Low | Medium | Re-pin oracle, re-run L2 suite; layouts documented for fast diff (`client-libraries.md`) | Watched |
| R3 | Event/command desync if a client pipelines commands while registered for events | Low | Medium | ADR-006 gating/queueing; desync-matrix integration test; troubleshooting entry | Mitigated |
| R4 | Windows port reservation blocks 4370 | Low | Low (config change) | Boot pre-flight with readable error; configurable ports | Mitigated |
| R5 | `better-sqlite3` ABI mismatch on a future Node upgrade | Medium | Low (Phase 5, optional) | In-memory default; adapter isolated; upgrade only when needed | Deferred |
| R6 | Scope creep on UI fidelity eats protocol time | Medium | Medium | Roadmap locks protocol-first ordering; menu is visual-only in v1 (ADR-007) | Managed |
| R7 | 1.3.0 quirks (no reply-code checks, replyId double-increment) mask simulator bugs in oracle tests | Medium | Medium | L1 golden-byte tests assert byte truth independently of client leniency; strict checksum flag exists | Managed |
| R8 | Docs rot as implementation proceeds | Medium | Medium | Docs-in-DoD (workflow), change-history habit, documentation-policy triggers | Process |
| R9 | Backend behavior differs from driver source (env-specific patches/config) | Low | High | Phase-1 gate runs against the *actual* backend end-to-end before Phase 2 starts | Planned |
| R10 | Camera-captured templates are too poor for reliable identification → the "real biometrics" demo falls flat | Medium | Medium | Phase-6 spike answers it *before* Phase 7 UI work (research Q1/Q4); quality gate with actionable feedback; USB-OTG scanner as the quality fallback (R11) | Monitored |
| R11 | USB-OTG module integration fails or is device-specific (Android vendor quirks, USB-serial drivers) | Medium | Low (camera path remains) | Camera station is plan A and never blocked; module validation is an isolated Phase-7 step; ADR-009 keeps the failure local to one adapter | Accepted |
| R12 | Real biometrics create a privacy/legal incident (stored plaintext, retained too long, committed by accident) | Low | **High** | ADR-011 decided *before* the first real template: encryption at rest, redaction in logs/API, no images persisted, delete/purge paths, `.gitignore` for data dirs and keys; demo/repo data stays synthetic | Planned (Phase 6 gate) |
| R13 | JVM sidecar adds operational friction (two runtimes to start; demo machines without a JRE) | Medium | Low | Sidecar optional at boot; protocol suite independent of it; documented single-command start; decision recorded in ADR-010 with alternatives | Managed |
| R14 | Scope creep: substitute ambitions delay the portfolio-critical simulator (Phases 1–5) | Medium | High | Hard ordering rule in `roadmap.md` — Phases 6–8 cannot start before Phase 1–5 gates are green; only two zero-cost forward hooks are allowed into Phases 1–5 | Enforced |
| R15 | Templates leak into packet logs (the engine legitimately handles template bytes for FR-8) | Medium | Medium | Redaction by default (`<TEMPLATE n bytes>`); hex re-enable is an explicit debugging action documented in troubleshooting; leak test in L5 | Planned |

## Early-warning signals
- L2 test flakiness under parallel sessions → revisit session cleanup.
- Backend team reports "device behaves differently than before" after simulator changes → check traceability + ADRs for the changed behavior.
- Golden-byte test edits without a research-doc update → process violation, fix the process.
