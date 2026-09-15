# ADR-010: Matching engine runs out-of-process (sidecar)

**Status:** **accepted** — confirmed by the Phase-6 spike; results recorded below `[V]` · **Date:** 2026-09-15 (spike recorded 2026-09-16)

## Context

A substitute must answer "who is this?" (1:N identify) and "is this really employee N?" (1:1 verify). That requires a **minutiae matcher**, not a blob comparison.

Research findings `[V]` (`../research/biometrics.md` §5):
- **No usable AFIS exists in the npm/Node ecosystem.** A search of published packages returns no maintained fingerprint *matcher* (the commonly-found `fingerprintjs` is *browser/device fingerprinting* — an unrelated false positive worth naming so nobody re-searches for it).
- The mature open-source options are **JVM-based** (SourceAFIS, Apache-2.0, consumes ISO/IEC 19794-2 and ANSI/INCITS 378) with ports/forks in other ecosystems of varying maturity; a C→WASM route exists but is research-grade `[A]`.
- Some cheap USB scanner modules can match **on-sensor** (the module returns a match/nomatch result instead of a template), which would remove the need for a host matcher but also removes template control and ties identity resolution to the sensor's own store `[A]`.

## Decision (accepted)

Run the matcher as a **separate local process behind a small HTTP/JSON service** — the "sidecar" — with a thin client in the Node app.

```text
Node app ── HTTP/JSON ──► matcher sidecar (JVM + SourceAFIS, localhost)
                          POST /templates   (add/remove template for uid,finger)
                          POST /identify    (probe template → ranked candidates)
                          POST /verify      (probe template + uid → score)
```

- Interface is **intentionally tiny** (three endpoints) so any engine — JVM, .NET, WASM, or a future pure-Node implementation — can satisfy it.
- Templates cross the boundary as ISO 19794-2 bytes only.
- The sidecar is **optional at boot**: the simulator runs perfectly with it absent (Phases 1–5 behaviour); biometric endpoints report 503 until it is up.
- Configuration: `ZK_MATCHER_URL`; absent/empty ⇒ biometric features disabled.

## Why not the alternatives

| Option | Rejected because |
|---|---|
| Pure-Node matcher | Nothing maintained exists `[V]`; writing a minutiae matcher from scratch is a research project, not a feature |
| C/C++ engine via WASM in-process | Research-grade builds, hard to debug, crashes take the protocol engine down with them `[A]` |
| On-sensor matching only | Loses template control/blobs (we need them for FR-8 wire compatibility and for BITS), couples identity to one vendor's module `[A]` |
| Cloud biometric API | Sends real biometrics off-device; contradicts the privacy posture (ADR-011) and the local-LAN deployment model |

## Consequences

- (+) Best available matching quality for free (Apache-2.0), in a language that has genuinely mature implementations.
- (+) Process isolation: a matcher crash or memory spike cannot corrupt or stall attendance serving — a property that matters because the protocol engine must keep serving BITS.
- (+) Engine is replaceable without touching device logic (the whole point of ADR-009's normalization).
- (−) Adds a **JVM runtime requirement** to the biometric deployment — mitigated by keeping the sidecar optional and documenting it as a Phase-6+ prerequisite, not a Phase-1–5 one.
- (−) Two runtimes to start for the full experience → a documented start script (`npm run dev:biometric`) is part of Phase 6.
- (−) Small latency per identify (HTTP + JVM) — irrelevant at demo scale; measured in the spike.

## Spike acceptance criteria (Phase 6 gate, must be recorded here)

1. Sidecar boots in < 10 s on the dev box and answers `/identify` for a single template in < 200 ms `[A]`.
2. **10 enrolled templates** (≥ 3 different fingers plus repeats) → correct uid identified in **10/10** probes with the impostor row rejected.
3. Enroll → identify → reject-then-accept cycle reproducible end to end from the app's REST layer.
4. Decision recorded by updating this ADR's status to `accepted` (or superseding it if the spike fails).

## Spike results (recorded 2026-09-16) — ALL CRITERIA MET

Method: sidecar (`sidecar/MatcherSidecar.java`, SourceAFIS 3.18 + fingerprintio 1.3.0, JDK 17) driven from a Node harness. Probe/enroll fixtures are **synthetic ISO 19794-2 templates** from the sidecar's `/synthetic` endpoint (36 deterministic minutiae per "finger" from a seed; probes = same seed rotated ±2…6°, ±1–3 px position jitter, ±10° angle noise, 10 % minutia drop-out) — synthetic by design per NFR-07, never real. The image-extract path (`POST /extract`) was verified mechanically on ridge images (real-image *matching quality* is a Phase-7 capture concern, handled there by the quality gate).

| Criterion | Requirement | Measured `[V]` |
|---|---|---|
| 1 | boots < 10 s | **3.86 s** cold (single-file source launcher incl. in-memory compile) |
| 1 | identify < 200 ms | **33–51 ms** steady-state with 10 templates enrolled (JVM warm); first calls after boot are JIT-warmup dominated (~1 s) |
| 2 | 10 templates, ≥ 3 fingers + repeats → 10/10 | **10/10 correct** (5 identities × fingers 0/1/2; probes rotated ±2…6° + jitter + 10 % drop) |
| 2 | impostor rejected | impostor probes score **≤ 2.1**; same-finger scores 62.6–324.9 vs cross-finger max **4.3** — ≈ 15× separation |
| 2 | 1:1 verify | same-finger **231.7** vs wrong-probe **0.236** |
| 3 | app REST layer cycle | **MET** — `test/integration/biometric.test.ts`: REST enroll (consent-gated) → REST identify match (`userId` resolved) → impostor `no-match` → remove → re-probe gone → purge leaves no ciphertext (sidecar candidate table verified empty) |
| 4 | decision recorded | this section; status → **accepted** |

Operating point: SourceAFIS threshold 40 separates the classes with a wide margin — configurable via `ZK_MATCH_THRESHOLD` (default null ⇒ 40); ambiguity guard via `ZK_MATCH_SEPARATION` (runner-up proximity, biometric-core §5). Negative images (noise / gradient / stripes) do not crash `/extract`; they yield small artifact templates, which is exactly why capture quality gating exists in Phase 7.

Final sidecar surface: `GET /health` · `GET /synthetic` (fixture generator) · `POST /extract` (image → ISO template) · `POST /templates` (enroll) · `POST /identify` · `POST /verify` · `POST /match` (engine consistency check) · `DELETE /templates/{uid}/{finger}`. All template payloads cross the boundary as **ISO 19794-2** (`FingerprintCompatibility` at the edge; SourceAFIS native format only inside the sidecar). ADR-010's "three endpoints" refers to the matching contract (`templates` / `identify` / `verify`); `/extract` is the capture-normalization entry point (ADR-009) and `/synthetic` the fixture generator.
