# ADR-003: TCP-only transport, UDP deferred

**Status:** accepted · **Date:** 2026-09-15

## Context
The ZK protocol exists over TCP and UDP on 4370. The brief said "plus `dgram` if needed". Research showed: the BITS backend instantiates `ZKLibTCP` directly and deliberately bypasses node-zklib's UDP fallback `[V]`; UDP mode uses different record sizes (28/16/8 B) and datagram-level retransmission quirks.

## Decision
Implement TCP only. UDP support is out of scope for v1; revisit only if a concrete UDP client appears (tracked in `../status/future-work.md`).

## Consequences
- (+) Halves transport work; avoids the undocumented 28/16/8-B UDP record variants entirely.
- (+) Zero risk to the demo: the oracle never speaks UDP to a reachable TCP server (fallback only triggers on `ECONNREFUSED`).
- (−) A UDP client (e.g. pyzk with `force_udp=True`) will not work — acceptable; documented in known-limitations.
