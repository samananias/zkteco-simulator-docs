# ADR-011: Biometric data protection (encrypt at rest, consent-gated, deletable)

**Status:** accepted · **Date:** 2026-09-15 · **Applies from:** Phase 6 (no real biometrics before that)

## Context

Until Phase 6, "fingerprints" in this project are synthetic blobs — no person's biometrics exist anywhere (`../operations/security.md`). Phases 6+ enrol **real** fingers, which turns templates into sensitive personal information under data-protection law (Research: `../research/biometrics.md` §6 — Philippine Data Privacy Act RA 10173 classification of biometrics as sensitive personal information; GDPR Art. 9 where applicable). The project must therefore have its protection rules decided **before** the first real template is stored, not retrofitted.

## Decision

1. **Store templates, never images.** Fingerprint images live only in memory during capture/quality-check and are discarded as soon as a template (ISO 19794-2) is extracted.
2. **Encrypt at rest.** Templates are stored as ciphertext (AES-256-GCM) in the biometric store. The key comes from `ZK_BIOMETRIC_KEY` (env/config, never committed); a dev-only key may be auto-generated with a loud warning. Plaintext templates never touch disk, logs, or the UI.
3. **No plaintext in logs or API responses.** Wire-level logging (NFR-09) must redact template payloads (`<TEMPLATE n bytes>`) — a real requirement because the protocol engine legitimately handles those bytes for FR-8.
4. **Consent-gated enrolment.** Enrolment is an explicit, logged action; the capture UI states what is captured and why. No bulk/silent enrolment. Demo seeds use **synthetic** templates only.
5. **Deletion is a first-class operation.** Deleting an employee deletes their templates (`CMD_DELETE_USERTEMP` / `CMD_CLEAR_DATA` type 2 map to it); a `POST /api/biometric/purge` exists for a full wipe. Retention rule: templates exist only while the employee record does.
6. **Repo hygiene is absolute.** No real template, image, or key may ever be committed to either repository. `.gitignore` covers biometric data dirs and key files; the docs repo contains no biometric data by definition.
7. **Thresholds and quality scores are configurable, never hard-coded** — different deployments must be able to tighten strictness without code changes.

## Consequences

- (+) The project can honestly state its data posture in the README/UI — a differentiator versus "toy biometrics" projects that store plaintext.
- (+) Key management is simple and explicit: one env var; rotating it is documented (invalidates stored templates — acceptable and stated).
- (+) A `purge` path means a demo can be fully reversed, which matters when real people volunteer fingers for a portfolio demo.
- (−) Encryption adds a small amount of code and one config requirement to Phase 6 — accepted, because retrofitting encryption over an existing template store is strictly worse.
- (−) Losing `ZK_BIOMETRIC_KEY` makes templates unrecoverable. Documented, not mitigated: re-enrolment is the recovery path (this is standard and defensible for a terminal).
- (−) Redaction of template bytes in the packet log slightly reduces wire-debug fidelity for template commands; hex logging can be explicitly re-enabled for a single debugging session, with the tradeoff stated in the troubleshooting doc.
