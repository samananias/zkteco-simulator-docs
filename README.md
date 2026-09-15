# ZKTeco Simulator — Documentation Repository

This repository is the **long-term technical knowledge base** for the ZKTeco Simulator App.
It is deliberately separate from the application repository (`zkteco-simulator-app`):

- **This repo** explains *how and why* the system works — research, architecture, decisions, plans, troubleshooting.
- **The app repo** contains *the system itself* — source code, tests, run configuration.

## Scope

The system is being built in two horizons, both documented here:

1. **v1 — the simulator (Phases 1–5):** protocol-faithful ZKTeco terminal emulation on TCP :4370 + device-face UI, with triggered (not scanned) punches.
2. **Substitute extension (Phases 6–8):** real fingerprint enrollment and matching (camera capture first, USB-OTG scanner later), an encrypted biometric store, and an open REST surface — with the ZK protocol surface frozen so the BITS backend and base demo never depend on any of it.

Two items are **permanently out of scope**, with the reasons documented rather than deferred: using a phone's built-in/in-display fingerprint sensor as a capture device (platform isolation), and template interoperability with real ZKTeco firmware (proprietary formats). See `simulator-docs/status/known-limitations.md`.

## Entry point

All documentation lives under [`simulator-docs/`](./simulator-docs/README.md).

## Operating principles

1. **Living knowledge base** — every implementation phase updates the relevant docs in the same change (see `process/documentation-policy.md`).
2. **No duplication** — the app repo has README-level info only; everything explanatory lives here.
3. **Verified vs assumed** — technical claims are tagged:
   - `[V]` verified against a cited source (spec file, library source code, or the BITS backend code).
   - `[A]` assumption — plausible but pending empirical validation (with the step that will validate it).
4. **Decisions are immutable** — accepted ADRs are never edited; they are superseded by a new ADR.

## Related repositories / codebases

| Repository | Role |
|---|---|
| [`samananias/zkteco-simulator-app`](https://github.com/samananias/zkteco-simulator-app) (local: `../zkteco-simulator-app/`) | The simulator application — implements what is documented here. |
| [`avegabros/bits`](https://github.com/avegabros/bits) (local: `C:/bits/backend`) | The BITS attendance system (Node.js + node-zklib@1.3.0) that the simulator must serve **unmodified**. Its `src/shared/lib/zk-driver.ts` defines the exact client behavior the simulator is validated against. |
