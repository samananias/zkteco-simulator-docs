# ADR-008: In-memory store default; SQLite behind an interface

**Status:** accepted · **Date:** 2026-09-15

## Context
The brief allows "in-memory for simplicity, or better-sqlite3 if persistence across restarts is wanted". Demo usage resets are actually *desirable* (fresh portfolio state).

## Decision
`DeviceStore` interface with an in-memory implementation as the default. A SQLite adapter (same interface, `better-sqlite3`) is a Phase-5 optional, enabled by config. Seed data loads into whichever store is active.

## Consequences
- (+) Zero native-module dependency for the core demo (no ABI/build risk on Windows).
- (+) Persistence, when wanted, is a config flip — the protocol engine cannot tell the difference.
- (−) In-memory mode loses state on restart (documented; seed re-applies automatically).
