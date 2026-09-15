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

## Early-warning signals
- L2 test flakiness under parallel sessions → revisit session cleanup.
- Backend team reports "device behaves differently than before" after simulator changes → check traceability + ADRs for the changed behavior.
- Golden-byte test edits without a research-doc update → process violation, fix the process.
