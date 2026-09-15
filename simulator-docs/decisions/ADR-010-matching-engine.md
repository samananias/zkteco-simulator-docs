# ADR-010: Matching engine runs out-of-process (proposed sidecar)

**Status:** **proposed** — decision to be confirmed by a Phase-6 spike; this ADR is not accepted until the spike results are recorded here `[A]` · **Date:** 2026-09-15

## Context

A substitute must answer "who is this?" (1:N identify) and "is this really employee N?" (1:1 verify). That requires a **minutiae matcher**, not a blob comparison.

Research findings `[V]` (`../research/biometrics.md` §5):
- **No usable AFIS exists in the npm/Node ecosystem.** A search of published packages returns no maintained fingerprint *matcher* (the commonly-found `fingerprintjs` is *browser/device fingerprinting* — an unrelated false positive worth naming so nobody re-searches for it).
- The mature open-source options are **JVM-based** (SourceAFIS, Apache-2.0, consumes ISO/IEC 19794-2 and ANSI/INCITS 378) with ports/forks in other ecosystems of varying maturity; a C→WASM route exists but is research-grade `[A]`.
- Some cheap USB scanner modules can match **on-sensor** (the module returns a match/nomatch result instead of a template), which would remove the need for a host matcher but also removes template control and ties identity resolution to the sensor's own store `[A]`.

## Decision (proposed)

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
