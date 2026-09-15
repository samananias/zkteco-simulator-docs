# Change History

Reverse-chronological. One entry per phase, decision, or significant fix (see `../process/documentation-policy.md`).

---

## 2026-09-15 — Phase 0: Foundations complete

**What**
- Analyzed the original brief; researched the ZKTeco standalone protocol against three independent sources (`adrobinoga/zk-protocol` spec, `pyzk`, `node-zklib@1.3.0` as installed in the BITS backend).
- **Pinned the validation oracle:** `node-zklib@1.3.0` (npm artifact, + backend's `patch-package` control-flow fix), driven exclusively over TCP by `C:/bits/backend/src/shared/lib/zk-driver.ts`; no comm key; realtime events unused by the backend; ~30 s polling scheduler.
- Established both repositories (`zkteco-simulator-app`, `docs` with `simulator-docs/`).
- Wrote the documentation foundation: requirements (functional/non-functional/traceability + archived brief), research (protocol notes, client libraries, feasibility report), architecture (system, engine, data model, web UI, API), decisions (ADR-001…008), plans (roadmap, revised plan, testing strategy), operations (setup, configuration, troubleshooting, security), process (workflow, documentation policy), status (limitations, risks, history, future work).

**Key findings recorded** (details in `../research/`)
- Wire format verified against a real captured packet; two checksum variants identified (±1) and neutralized by ADR-005.
- **Device time uses a 372-day calendar** (31-day months) — brief omission, would have corrupted all timestamps.
- Brief's read-command expectation (`CMD_DB_RRQ`/`CMD_ATTLOG_RRQ`) corrected: the oracle actually reads via `CMD_DATA_WRRQ` → direct `CMD_DATA` bursts; chunked flow kept behind config (ADR-004).
- Backend writes users as plain 72-byte records (no pyzk tag byte); template probe/read/write envelopes pinned to the driver's logic.
- No usable existing simulator exists in open source (search performed) — building our own is justified and is the stronger portfolio story.

**Decisions:** ADR-001…ADR-008 accepted (single process; TypeScript; TCP-only; direct-burst reads; checksum strategy; realtime gating; vanilla UI; memory store).
**Next:** Phase 1 — protocol core (see `../plan/roadmap.md`).
