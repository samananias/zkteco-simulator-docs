# ADR-002: TypeScript for the protocol engine

**Status:** accepted · **Date:** 2026-09-15

## Context
The core difficulty of this project is byte-exact record structs (72-B users, 40-B attendance, 32-B events) and a two-variant checksum. The original brief suggested plain Node.js; the BITS backend is already TypeScript, so team familiarity exists.

## Decision
Write the engine in TypeScript (strict mode), executed in dev via `tsx`, built with `tsc`. Record codecs defined next to typed interfaces with byte-offset constants co-located in the same module. `node:test` as the runner.

## Consequences
- (+) Literal/union types encode protocol constraints (`verifyType: 0|1|2`), decoders and encoders share one type — drift between "what we send" and "what the client parses" is caught at build time.
- (+) Matches the backend's ecosystem (same language family, shared mental model).
- (−) A compile step and `tsx` dev dependency. Rejected plain JS for this reason and this reason only; the cost is one `npm run build`.
- (−) `Buffer`-heavy code still needs golden-byte unit tests — types do not verify wire layout; the test suite (testing-strategy.md) covers that.
