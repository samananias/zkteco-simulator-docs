# ADR-004: Reads answered with direct `CMD_DATA` bursts; chunked flow config-gated

**Status:** accepted · **Date:** 2026-09-15

## Context
The spec (and the brief) describe large reads as the `CMD_DATA_RDY → CMD_PREPARE_DATA → CMD_DATA → CMD_ACK_OK` chunked dance. But the pinned oracle (node-zklib 1.3.0) resolves `readWithBuffer` on the **direct `CMD_DATA` reply path**: it accumulates frames for 1 s of silence, then slices a 4-byte size prefix and iterates records `[V]`. Demo datasets are a few KB.

## Decision
Default: answer `CMD_DATA_WRRQ` (and pyzk-style `CMD_DB_RRQ`/`CMD_ATTLOG_RRQ`) with the complete dataset in one back-to-back `CMD_DATA` burst. Also implement the spec's chunked conversation, enabled by config, for spec conformance and future clients (its 8-byte per-chunk sub-header is validated by an integration test before the mode is exposed).

## Consequences
- (+) Deterministic behavior with the backend; no dependency on the partially-undocumented chunk header.
- (+) Chunked mode preserves the "indistinguishable from a real device" ambition for large datasets and other clients.
- (−) Two read paths to test (mitigated: shared dataset serializer, separate transport strategies).
- (−) Datasets > ~65 KB require chunked mode to be enabled (demo data stays far below).
