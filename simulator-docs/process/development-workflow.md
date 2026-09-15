# Development Workflow

## Feature cycle (the happy path)

```text
Requirement ─► Research ─► Feasibility check ─► Architecture/Design ─► Documentation
     ─► Implementation ─► Testing ─► Validation ─► Documentation update ─► Next feature
```

Concrete rules per step:
- **Research** before touching wire-level code: check `../research/zk-protocol-notes.md` first; extend it with the newly verified fact and its source.
- **Feasibility check:** anything new gets a row in `../research/feasibility-report.md` + `../requirements/traceability.md` before implementation.
- **Design:** protocol-visible behavior changes require an ADR *before* the code merges.
- **Implementation:** one layer at a time (transport → protocol → device → web); no cross-layer shortcuts.
- **Testing:** the phase gate in `../plan/roadmap.md`; the pinned oracle decides compatibility.
- **Documentation update:** part of the same change — docs lag is unfinished work.

## Problem cycle (when reality disagrees)

```text
Problem ─► Investigate ─► Identify root cause ─► Research ─► Choose solution
        ─► Document decision (ADR or troubleshooting entry) ─► Implement fix ─► Test ─► Update docs
```

Root-cause discipline: wire-level bugs get the exchange captured in hex (`ZK_LOG_LEVEL=debug`) and attached to the investigation notes; the fix lands with a regression test in the L1/L2 suite (`../plan/testing-strategy.md`).

## Definition of Done (any change)
1. Code complete, layered correctly.
2. L1 tests updated/passing; L2 oracle test for anything protocol-visible.
3. Docs updated: traceability row, relevant ADR/research/architecture page, `../status/change-history.md` entry.
4. No new runtime dependency without an ADR.

## Branching & commits
- Trunk-based: `main` is always demo-ready. Short-lived `feature/<topic>` branches fine; rebase before merge.
- Conventional, imperative messages (`feat: user read burst`, `fix: 372-day decode for month 12`, `docs: ADR-005 outcome`).
- Both repos commit together when a change spans docs + code.

## Where things live
| Artifact | Location |
|---|---|
| Application code/tests | `zkteco-simulator-app/` |
| Why-it-works knowledge | `docs/simulator-docs/research`, `architecture`, `decisions` |
| What-to-build knowledge | `docs/simulator-docs/requirements`, `plan` |
| Running/troubleshooting | `docs/simulator-docs/operations` |
| History & limits | `docs/simulator-docs/status` |
