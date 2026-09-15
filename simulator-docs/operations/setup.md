# Setup & Deployment

## 1. Prerequisites
- Node.js ≥ 20 (dev machine currently runs v24) · npm ≥ 10 · git
- The BITS backend only if you want the full end-to-end demo (`C:/bits/backend`)

## 2. Repositories
```bash
# both live side by side in the workspace
zkteco-simulator/
├── zkteco-simulator-app/   # the simulator (this is what you run)
└── docs/                   # this documentation repo (nothing to install)
```

## 3. Install & run (app repo)
```bash
cd zkteco-simulator-app
npm install
npm run dev            # protocol engine :4370 + HTTP/WS :3000 + seed data
# production-ish:
npm run build && npm start
```

### 3b. Full biometric experience (Phase 6+) — optional
Requires **JDK 17+** (`java` on PATH). Everything else installs with `npm install`.
```bash
npm run dev:biometric
```
This starts the matcher sidecar (default `127.0.0.1:28090`, override with `ZK_MATCHER_PORT`) and the simulator with `ZK_MATCHER_URL` wired to it — one command, both runtimes, Ctrl+C stops both. Without it, the simulator behaves exactly as in §3 (biometric endpoints answer 503 — NFR-12, never auto-accept).

For persistence across restarts set a stable key before enrolling real templates:
```bash
# 32 bytes — hex (any generator, e.g. node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")
export ZK_BIOMETRIC_KEY=<64 hex chars>
```
Without it a process-session key is used with a loud warning: enrolled templates become unreadable after restart (re-enrollment is the recovery path — ADR-011). Encrypted records live under `./data/biometric/` (git-ignored).

## 4. Point the BITS backend at the simulator
The backend resolves devices from `ZK_HOST`/`ZK_PORT` env or per-device rows in its DB.
- Simplest: set `ZK_HOST=127.0.0.1` (or your LAN IP for device-on-another-machine demos) and keep port `4370`.
- Or edit the device record's IP in the BITS admin UI to the simulator host's IP — no code changes, as designed.
- Verify from the BITS side: device status endpoint / topbar indicator turns active; `getInfo`, users, and the ~30 s attendance sync begin to flow.

## 5. Windows notes
- Port 4370 is not a standard Windows reservation, but Hyper-V/dynamic ranges occasionally reserve ranges — the simulator pre-flights the port and fails with a readable message (see `troubleshooting.md`).
- No admin rights required. Firewall prompt on first run: allow on **private** networks only.

## 6. Fresh-machine checklist (Phase 5 gate)
`operations/setup.md` is the canonical walkthrough; the Phase-5 exit criterion is that a clean clone → demo works by following it verbatim.

## 7. Deployment shapes
- **Laptop demo (primary):** simulator + backend + DB on one machine; punch from the web UI, watch the dashboard update.
- **LAN demo:** simulator on a laptop visible as `192.168.x.x:4370`; backend elsewhere — indistinguishable from a real device on the network.
- **Server:** not a target for v1 (no auth, no TLS on the wire protocol — see `security.md`). Docker packaging listed as future work.
