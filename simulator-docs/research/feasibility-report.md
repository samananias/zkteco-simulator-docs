# Feasibility Report

**Date:** Phase 0 · **Method:** spec + source verification (`zk-protocol-notes.md`, `client-libraries.md`) against the pinned oracle. Categories: **FF** Fully Feasible · **FM** Feasible With Modification · **TR** Technically Risky · **DE** Dependent on External · **NF** Not Feasible as Proposed.

| # | Requirement (brief) | Verdict | Evidence / reason | Revision | Priority |
|---|---|---|---|---|---|
| 1 | TCP server :4370 speaking the exact wire protocol | **FF** | Framing verified against a real captured packet in the spec; three independent implementations agree | None | High |
| 2 | Checksum + session/reply handshake | **FF** (one open question) | Both variants computed and compared; oracle does not validate reply checksums `[V]` | Accept both inbound; outbound classic w/ config switch (ADR-005); Phase-1 golden-bytes test | High |
| 3 | Connect / auth (comm key) / exit | **FF** | makeCommKey verified identical in pyzk + node-zklib; backend uses **no** comm key `[V]` | AUTH optional, default off | High |
| 4 | Version / capacity / time / options | **FF** | Free-sizes offsets (24/40/72) verified; **372-day-year time codec** discovered in oracle source — brief omitted it | Implement codec exactly; options store | High |
| 5 | getUsers()/getAttendances() with chunked transfer | **FM** | Oracle's primary read path is a direct `CMD_DATA` burst (not `CMD_DB_RRQ`/`CMD_ATTLOG_RRQ` replies as the brief implies); layouts fully verified | Serve both request styles; direct burst default, chunked optional (ADR-004) | High |
| 6 | Realtime events (REG_EVENT + EF_ATTLOG) | **FF** | node-zklib 1.3.0 `getRealTimeLogs` verified end-to-end; 32-B event record satisfies it; backend never registers → no interference | Per-session gating + in-flight queueing (ADR-006) | High |
| 7 | On-demand punch trigger (REST/CLI) | **FF** | Pure application layer | — | High |
| 8 | Two-way user sync (backend `setUser` → visible on device) | **FF** | Backend sends a plain 72-B `USER_WRQ` record (no tag byte) — pinned from driver source | Accept 72-B and pyzk 73-B variants | Medium |
| 9 | Fingerprint template placeholders | **FM** | Templates are opaque blobs; the driver's probe/read logic (>100 B, 6-B entry header) is pinned | Synthetic ≥500 B blobs with correct envelope in v1; **extension #21** replaces the bytes with real ISO templates without touching the wire shape | Medium |
| 10 | Cosmetic commands (UNLOCK, LCD, RESTART…) | **FF** | Single-ACK handlers; LCD drives the web UI | — | Low |
| 11 | Device-face web UI + WebSocket sync | **FF** | Pure frontend; no hardware dependency | Kiosk aesthetic per brief §5 | High |
| 12 | SQLite persistence | **FF** | `better-sqlite3` prebuilds for current Node; behind store interface | In-memory default | Medium |
| 13 | UDP transport | **TR** | Backend is **TCP-only by design** (bypasses UDP fallback) `[V]`; UDP uses different record sizes (28/16/8 B) | Out of scope v1 (ADR-003) | Low |
| 14 | "Feels indistinguishable from a real device" | **FM** (scope) | No biometric capture; protocol edge behaviors approximated; single-socket command/event race is protocol-inherent | Reframe: protocol-compatible at command level + visual replica; deviations documented | Medium |
| 15 | Zero backend changes | **DE** (resolved) | Backend pinned: node-zklib@1.3.0 + patch, TCP-only wrapper, no comm key `[V]` | Oracle tests replicate the driver's exact call patterns | High |
| 16 | Reuse an existing simulator | **NF** (none exists) | GitHub search: only trivial repos (0★); no viable prior art | Build from spec — also the stronger portfolio story | — |

## Biometric extension — feasibility roll-up (2026-09-15)

Owner direction: evolve the simulator into a **ZKTeco substitute** (real enrollment + real matching). Full evidence: `biometrics.md`. Categories as above.

| # | Requirement (extension) | Verdict | Evidence / reason | Revision | Priority |
|---|---|---|---|---|---|
| 17 | Capture fingerprints with a phone's **built-in / in-display** sensor | **NF** | Platform isolation (TEE / Secure Enclave): `BiometricPrompt`/`LAContext` expose only a boolean result; no capture API, no app-facing enrollment API `[V]` | Permanently ruled out; intent preserved via camera / USB scanner | — |
| 18 | **Camera-based touchless capture** (zero hardware) | **FM** | Images are fully exposed to apps (`getUserMedia`); minutiae extraction from a photo is standard practice, quality is lighting/pose sensitive `[A]` | Quality gate + actionable rejection reasons; ship as a PWA; quality bar documented, not hidden | High |
| 19 | **USB-OTG fingerprint scanner** capture | **DE** | Cheap ZFM/R30x-class modules + USB-serial bridges work via Android USB Host; needs a purchased module and per-module validation `[A]` | Second `CaptureStation` adapter; nothing upstream depends on it | Medium |
| 20 | **Matching engine** (1:N identify, 1:1 verify) | **FM** | **No usable matcher exists in npm** `[V]`; mature option is JVM SourceAFIS (Apache-2.0, ISO 19794-2/ANSI 378) `[V]`; WASM ports are research-grade `[A]` | Run out-of-process behind a 3-endpoint HTTP/JSON interface (ADR-010, **proposed** pending spike) | High |
| 21 | Real templates over the ZK protocol (FR-8 wire shape) | **FF** | The protocol treats templates as **opaque blobs** — the backend already reads/writes/deletes them `[V]` | Keep bytes opaque; add `format` field; wire surface frozen | High |
| 22 | Template interop with **real ZKTeco hardware** | **NF** | Proprietary ZKFinger-family formats, no published converter `[A]` | Permanent documented boundary; "substitute" means within our own ecosystem | — |
| 23 | Store biometrics per employee | **FF** | Ordinary keyed storage; BITS already keyed by `uid`/`userId` | Encrypted at rest (ADR-011), consent-gated, purge path | High |
| 24 | Privacy/legal compliance for real biometrics | **FF** (obligation) | Biometrics are sensitive personal information (RA 10173; GDPR Art. 9 where applicable) `[V]` | NFR-10/NFR-11 + ADR-011: consent, encryption, minimisation (no images), deletion, repudiation of cloud matchers | High |
| 25 | Open REST surface so non-ZK systems can integrate | **FF** | Pure application layer | API v2 additive to v1 (FR-18) | Medium |

**Sequencing verdict:** none of the above invalidates Phases 1–5. The only concessions are two zero-cost seams (`FpTemplate.format`, `BiometricStore` interface) plus the standing rule that the protocol surface stays frozen. See `../plan/roadmap.md` Phases 6–8.

## Uncertainty register (explicit, per instructions)

| Uncertainty | Why it can't be settled on paper | Validation path |
|---|---|---|
| Which checksum variant real firmware expects | Diverges by 1 between two widely-used implementations; no device available | Non-blocking for oracle (it never validates) `[V]`; golden-bytes test records oracle-emitted checksums; real-device check is a future-work item |
| Chunked-read 8-B per-chunk sub-header | Only observed indirectly in oracle accumulation logic | Phase-1 integration test if ADR-004 chunk mode is enabled |
| pyzk as second oracle | Its read style (`CMD_DB_RRQ`/`CMD_ATTLOG_RRQ`) is implemented per spec, untested | Optional Phase-1/3 cross-check |
| 1.3.0 vs GitHub-master divergence | Master is unpublished; behavior differs | Resolved by pinning 1.3.0 `[V]`; revisit only if the backend upgrades |

**Bottom line:** no brief requirement is fundamentally incompatible; the simulator is realistic, with the revisions above. Highest-risk items (#2, #5) are mitigated by making the *pinned oracle* the acceptance criterion.
