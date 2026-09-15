# Documentation Policy

How this knowledge base stays alive without becoming bureaucracy.

## Principles
1. **Docs evolve with code** — the same change updates both; stale docs are bugs.
2. **One home per fact** — no duplication between repos; the app README links here for every "why".
3. **Claim discipline** — technical statements carry `[V]` (with source) or `[A]` (with validation step). Promoting `[A]` → `[V]` is a normal, celebrated edit.
4. **Brevity with structure** — tables > prose; every doc answers a question someone will actually ask.

## What triggers which update

| Event | Required updates |
|---|---|
| New/changed wire behavior | `research/zk-protocol-notes.md`, relevant ADR (new if decision changed), `requirements/functional.md`, `requirements/traceability.md` row |
| New dependency or module boundary | ADR + `architecture/system-architecture.md` |
| A phase completes | `plan/roadmap.md` checkboxes, `status/change-history.md`, traceability statuses |
| A bug with a non-obvious cause | `operations/troubleshooting.md` entry (+ regression test) |
| A discovered limitation | `status/known-limitations.md` |
| A discovered risk | `status/risks.md` |
| Oracle/backend changes (upgrade, new library) | `research/client-libraries.md` + re-pin note in change-history |

## ADR lifecycle
`proposed → accepted → (superseded by ADR-MMM)`. Accepted ADRs are never edited. Template and index: `../decisions/README.md`.

## Change history
Reverse-chronological, one entry per phase/decision/fix: date, phase, what, why, docs touched. New entries at the top.

## Style rules
- Markdown, relative links, no generated TOCs (index doc serves that role).
- Code identifiers in backticks; byte offsets as `field size@offset` tables.
- Every research doc lists its sources with dates (sources rot — the date tells readers how much to trust).
- Delete docs that no longer answer a live question — the archive of *reasoning* lives in ADRs and change-history, not in zombie pages.
