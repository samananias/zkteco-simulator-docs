# Security Considerations

Written for a portfolio/demo tool — honest about what is and isn't protected.

## What the underlying protocol gives you (and doesn't)
- **No TLS, no integrity:** the ZK standalone protocol is plaintext binary over TCP; the checksum is error-detection, not a MAC. Anyone on the network segment can read or inject packets. `[V]` (wire format has no crypto anywhere)
- **Weak auth:** optional comm key (a 4-byte hashed value, no salt, session-bound). It stops casual connections, not an intentional attacker. The BITS backend runs **without** one.
- Implication: position the simulator like you would a real terminal — on a trusted LAN, never directly internet-exposed.

## Simulator-specific posture
| Area | Decision |
|---|---|
| Binding | HTTP/WS default `0.0.0.0` for LAN demos; use `ZK_HTTP_HOST=127.0.0.1` for private runs. Never port-forward to the internet. |
| REST/WS auth | None by design (demo tool). If ever deployed beyond a trusted LAN, put it behind a reverse proxy with auth — do not build auth into the simulator. |
| Data | Seed employees are fictional; **no real personal data** may be committed (NFR-07). Real punches ingested via the backend stay in the backend's DB — the simulator only stores device-level records. |
| Comm key | Supported (`ZK_COMM_KEY`) for demonstrating the auth flow; default off to match the backend. |
| Fingerprint blobs | Synthetic in v1 — **no biometric data of any person exists in this project's repo or demo seeds**. From Phase 6, real templates may exist and are handled under ADR-011 (encrypted at rest, consented, deletable, redacted from logs/API); images are never persisted. |
| Supply chain | Runtime dependencies kept minimal (`ws` only planned); the pinned oracle is a devDependency for tests, never shipped to users of the simulator. |
| Repo hygiene | `.env` files and SQLite data dirs are git-ignored; the docs repo must never contain internal credentials. |

## Biometric data (Phase 6+ — the one area with real-world obligations)

When real fingerprints are enrolled, templates become **sensitive personal information** (ADR-011; `../research/biometrics.md` §6). Rules:

| Rule | Implementation |
|---|---|
| Consent before first capture | Explicit, logged enrollment action with a stated purpose; the capture page says what is captured and why. No silent or bulk enrollment |
| Templates, never images | Images exist only in memory during capture/quality-check, then are discarded |
| Encrypted at rest | AES-256-GCM; key from `ZK_BIOMETRIC_KEY`; key never committed or logged; data dir git-ignored |
| Redaction | Packet logs and API responses show `<TEMPLATE n bytes>`, never template bytes. Hex re-enable is an explicit debugging action (tradeoff documented) |
| Deletable | Employee delete, `CMD_DELETE_USERTEMP`, `CMD_CLEAR_DATA` (templates) and `POST /api/biometric/purge` remove ciphertext |
| Reversible demo | `purge` restores a clean state — important when real volunteers provide fingers for a demo |
| Repo hygiene | No template, image, or key may ever be committed to either repository; seeds remain synthetic |

Honest limits: no liveness/anti-spoofing (a fingerprint photograph can pass a camera station — known-limitations #17), and integrity of stored templates relies on the filesystem, not on tamper-evident hardware — a real terminal would have the same class of exposure, but we do not pretend to solve it.

## Threat-model notes (short)
- Attacker on the LAN can impersonate the device or the backend (plaintext) — same as with real hardware; out of scope to fix.
- The simulator trusts its HTTP clients (can trigger punches/reset). That's the feature; don't expose it.
