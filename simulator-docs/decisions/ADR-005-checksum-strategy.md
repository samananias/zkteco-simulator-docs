# ADR-005: Checksum — lenient inbound (both variants), classic outbound default

**Status:** accepted · **Date:** 2026-09-15

## Context
Two published implementations of the ZK checksum disagree by exactly 1 in the final step: classic (spec/pyzk) = one's complement of the mod-65535 word sum (`65535 − s`); the pinned oracle (node-zklib 1.3.0) emits `65534 − s` `[V]`. Which one real firmware expects cannot be settled without hardware. Critically, the oracle **never validates reply checksums** (`decodeTCPHeader` ignores the field) `[V]`.

## Decision
1. Inbound: compute both variants; accept the packet if either matches; log which matched (never reject a client packet on checksum mismatch alone — log-only at `warn`).
2. Outbound: emit the **classic** variant by default (spec-conformant, satisfies any validating client such as pyzk); `ZK_CHECKSUM_VARIANT=oracle` config switch for experiments.

## Consequences
- (+) Impossible to break the oracle interop regardless of variant resolution; spec clients still get spec frames.
- (−) Slightly lenient server posture (malformed frames pass with a warning) — mitigated by NFR-09 logging and a `strict` config flag that enforces the chosen variant if ever needed.
- (→) Real-device verification of the "true" variant recorded as future work; the answer changes one config default, not the architecture.
