# ADR-001: Single-process, four-layer Node.js app

**Status:** accepted · **Date:** 2026-09-15

## Context
The simulator must host a binary TCP protocol server, a small HTTP/WS API and a static web page, sharing one mutable "device state". Expected load: one backend client polling every ~30 s plus a demo browser.

## Decision
One Node.js process with four strictly layered modules (transport → protocol → device → web) communicating through the store + an internal event bus. No external services.

## Consequences
- (+) Trivial to run (one command), trivial to reason about, no IPC/serialization.
- (+) The store/event bus seam lets protocol and UI be tested independently (unit tests without sockets).
- (−) A crash takes down both protocol and UI — accepted: robustness NFR-02 + watchdog-friendly single command (restart is instant by design).
- (−) If real multi-device simulation or heavy load ever materializes, layers can be split without rewriting the protocol core.
