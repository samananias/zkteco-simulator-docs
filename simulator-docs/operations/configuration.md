# Configuration

Single source: environment variables + optional CLI flags (12-factor style, no config file needed for v1). Defaults chosen so **zero configuration runs the full demo**.

| Variable | Default | Meaning |
|---|---|---|
| `ZK_TCP_PORT` | `4370` | Protocol engine TCP port (the "device" port) |
| `ZK_HTTP_PORT` | `3000` | Web UI + REST + WebSocket port |
| `ZK_HTTP_HOST` | `0.0.0.0` | HTTP bind (use `127.0.0.1` for private demos) |
| `ZK_COMM_KEY` | `0` | Comm key for `CMD_AUTH`; `0` = disabled (backend default) |
| `ZK_CHECKSUM_VARIANT` | `classic` | Outbound checksum variant: `classic` \| `oracle` (ADR-005) |
| `ZK_CHECKSUM_STRICT` | `false` | Reject inbound packets failing the selected variant (default: lenient, log-only) |
| `ZK_READ_MODE` | `direct` | `direct` = CMD_DATA burst · `chunked` = spec conversation (ADR-004) |
| `ZK_DEVICE_NAME` | `ACME HQ — Main Door` | `~DeviceName` option / UI title |
| `ZK_SERIAL` | `CJ7X202660399` | `~SerialNumber` option (plausible-looking) |
| `ZK_TIME_OFFSET_MIN` | `0` | Device clock offset from host clock (minutes) — demos "wrong clock" scenarios |
| `ZK_SEED` | `standard` | `standard` \| `empty` — seed profile |
| `ZK_STORE` | `memory` | `memory` \| `sqlite` (Phase 5) |
| `ZK_SQLITE_PATH` | `./data/device.sqlite` | SQLite file when `ZK_STORE=sqlite` |
| `ZK_LOG_LEVEL` | `info` | `debug` adds per-frame hex dumps (NFR-09) |
| `ZK_MATCHER_URL` | *(empty)* | Matcher sidecar base URL (`http://127.0.0.1:28090` — the sidecar's default port); empty ⇒ biometric features disabled (NFR-12) |
| `ZK_BIOMETRIC_KEY` | *(empty)* | AES-256-GCM key (32 bytes: hex/base64/raw) for biometric templates at rest (ADR-011). Empty in dev ⇒ auto-generated **session** key with a loud warning — ciphertext becomes unreadable on restart; **never** committed or logged. Required in any real deployment |
| `ZK_BIOMETRIC_DATA` | `./data/biometric` | Directory for encrypted template storage (git-ignored). CLI flag: `--zk-biometric-data-dir` |
| `ZK_MATCH_THRESHOLD` | `40` | Accept threshold for 1:N identify (SourceAFIS operating point validated by the ADR-010 spike: same-finger ≥ 62.6 vs cross-finger ≤ 4.3) |
| `ZK_MATCH_SEPARATION` | `20` | Minimum score gap between best and runner-up candidate required to accept; a closer runner-up ⇒ `ambiguous` rejection (never silently pick) |
| `ZK_CAPTURE_SAMPLES` | `3` | Samples required per enrollment (FR-14) |

## Precedence & validation
CLI flags (e.g. `--zk-tcp-port 4371`) override env; invalid values fail fast at boot with the expected type/range in the message. All ports pre-flighted (bind check) before the "ready" log line.

## Backend-side configuration (recall)
The backend needs no changes: `ZK_HOST=<simulator ip>`, `ZK_PORT=4370` (or per-device rows). It runs its own ~30 s sync scheduler; no comm key.
