# ADR-007: Device-face UI — vanilla HTML/CSS/JS, no framework

**Status:** accepted · **Date:** 2026-09-15

## Context
The UI must look like an early-2010s embedded touchscreen kiosk in a hardware bezel (brief §5): one screen, ~6 states, a WebSocket connection. Component libraries and virtual DOM solve problems this UI doesn't have, while their default styling fights the required aesthetic.

## Decision
Hand-written HTML/CSS/JS (no build step, no framework, one `ws` client). Served statically by the same process.

## Consequences
- (+) Total control over the kiosk aesthetic (fonts, bezel, glyphs) — the fidelity goal *is* the styling.
- (+) No build toolchain for the front end; instantly hackable during a demo.
- (−) No component reuse — acceptable at ~6 screens; if the UI grows a real menu system, revisit this ADR before adding a framework.
