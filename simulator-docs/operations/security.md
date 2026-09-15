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
| Fingerprint blobs | Synthetic — no biometric data of any person exists in this project. |
| Supply chain | Runtime dependencies kept minimal (`ws` only planned); the pinned oracle is a devDependency for tests, never shipped to users of the simulator. |
| Repo hygiene | `.env` files and SQLite data dirs are git-ignored; the docs repo must never contain internal credentials. |

## Threat-model notes (short)
- Attacker on the LAN can impersonate the device or the backend (plaintext) — same as with real hardware; out of scope to fix.
- The simulator trusts its HTTP clients (can trigger punches/reset). That's the feature; don't expose it.
