# Deployment & Run Guide (Phase 8)

For running the substitute beyond a laptop demo: one machine on a trusted office LAN
where people actually enrol fingers and punch for real.

**Scope reminder.** The simulator is a LAN tool. Its ZK protocol is plaintext with
weak auth, and its own HTTP surface has **no auth by design** (`security.md`) —
neither may ever be internet-exposed. "Deployment" here means *a machine on the
office network*, not a cloud service.

## 1. Topology

```text
  📱 capture stations            🖥 the substitute host                 ☁ BITS
  (phone/laptop browsers)        ┌──────────────────────────────
  /enroll  /scan  /biometric ───►│ simulator  :3000 HTTP/WS/UI  │
                                 │            :4370 ZK protocol │◄── node-zklib
                                 │ matcher sidecar :28090 (JVM) │    (30 s sync)
                                 │ data/biometric/  (encrypted) │
                                 └──────────────────────────────┘
```

Two runtimes are required for real biometrics; both start with one command
(`npm run dev:biometric`, or as services in §6). Without the sidecar the simulator
still runs the full v1 demo and every biometric endpoint answers `503` — never an
auto-accept (NFR-12).

## 2. Requirements

| | |
|---|---|
| Node.js | ≥ 20 (dev box runs 24) |
| JVM | **JDK 17+** on `PATH` (`java -version`) — the matcher sidecar is Java (ADR-010) |
| Ports | `4370` (ZK protocol), `3000` (HTTP/WS), `28090` (sidecar, loopback only) |
| Disk | trivial: one AES-GCM file per enrolled finger (~1 KB each) + consent JSON |
| Network | trusted LAN only; browsers used as capture stations need to reach `:3000` |

## 3. Install & configure

```bash
git clone <app repo> && cd zkteco-simulator-app
npm ci                 # or npm install
npm run build          # optional: dist/ for `npm start`
```

Configuration is env/CLI only (`configuration.md` is the full table). The values that
matter for a real deployment:

```bash
ZK_BIOMETRIC_KEY=<64 hex chars>      # REQUIRED — see §4
ZK_HTTP_PORT=3005                    # avoid clashing with BITS' Next.js dev server on 3000
ZK_STATION_ID=front-door-1           # station attribution for consent/enrollment records
ZK_RETENTION_DAYS=0                  # 0 = keep while the employee record exists (ADR-011 §5 default)
ZK_CONSENT_VERSION=v1                # statement version collected against (must exist in consent-text.ts)
ZK_MATCHER_URL=http://127.0.0.1:28090
```

## 4. Key management (the one unrecoverable thing)

Templates are AES-256-GCM ciphertext; `ZK_BIOMETRIC_KEY` is the only way back in.

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

- Store it in the service environment (systemd `EnvironmentFile` / NSSM app parameters),
  **not** in the repo — `.env` and biometric data dirs are git-ignored, and no key or
  template may ever be committed (ADR-011 §6).
- **Without a key** the simulator generates a process-session key and warns loudly:
  everything still works until restart, then the ciphertext is unreadable. That is the
  safe default, not a bug.
- **Rotating** the key invalidates every stored template — employees re-enrol.
- **Losing** the key is unrecoverable by design (known-limitations #13); the backup in
  §5 is the mitigation.

## 5. Backup & restore

Two things must travel together: **the key** and **the data directory** (`ZK_BIOMETRIC_DATA`,
default `./data/biometric`).

```text
data/biometric/
├── <uid>-<finger>.bio        # AES-256-GCM ciphertext (iv|tag|ct) — useless without the key
└── consent/
    └── <userId>.consent.json # plaintext accountability records: who consented, to which
                              # statement version + hash, when, from which station
```

```bash
# backup
tar czf biometric-$(date +%F).tgz data/biometric
# restore (same key!)
tar xzf biometric-2026-09-16.tgz
```

Consent records are deliberately **plaintext** (they contain no biometrics): the
accountability evidence must stay readable even if the key is lost. Restoring them
alone restores the audit trail, not the ability to match.

## 6. HTTPS for camera stations (why you probably need it)

Browser camera access (`getUserMedia`) requires a **secure context**: `localhost` or
HTTPS. A page served from `http://192.168.x.x:3000` is *not* secure, so phones will
refuse to open the camera. Three ways out, in order of preference:

1. **Reverse proxy with TLS** (production): terminate HTTPS in front of `:3000`.
   ```caddy
   terminals.office.lan {
     reverse_proxy 127.0.0.1:3000   # WebSocket upgrades for /ws pass through transparently
   }
   ```
   With an internal or real certificate, capture pages then work from any phone on the
   LAN. Point the proxy at the HTTP port only — never expose `:4370`.
2. **Dev/demo path — `adb reverse`** (no TLS; phone must be USB-connected):
   ```bash
   adb reverse tcp:3005 tcp:3005      # phone's localhost:3005 → this machine's :3005
   ```
   Open `http://localhost:3005/enroll` **on the phone** — `localhost` is a secure
   context, so the camera works and traffic never crosses Wi-Fi.
3. **Chrome flag** (`chrome://flags/#unsafely-treat-insecure-origin-as-secure`) —
   demo-only convenience; you are lowering a browser protection deliberately.
   iPhone/iPad have no equivalent: use 1 or 2.

## 7. Run as a service

**Linux (systemd)** — one unit each:

```ini
# /etc/systemd/system/zkteco-simulator.service
[Unit]
Description=ZKTeco substitute simulator
After=network.target

[Service]
WorkingDirectory=/opt/zkteco-simulator-app
EnvironmentFile=/etc/zkteco-simulator.env      # ZK_BIOMETRIC_KEY etc.
ExecStart=/usr/bin/node dist/index.js
Restart=on-failure
User=zksim

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/zkteco-matcher.service
[Service]
WorkingDirectory=/opt/zkteco-simulator-app
ExecStart=/usr/bin/java -cp sidecar/lib/* sidecar/MatcherSidecar.java 28090
Restart=on-failure
User=zksim
```

**Windows (NSSM)**:

```powershell
nssm install ZkMatcher java "-cp" "sidecar\lib\*" "sidecar\MatcherSidecar.java" "28090"
nssm set ZkMatcher AppDirectory C:\antigravity-proj\zkteco-simulator\zkteco-simulator-app
nssm install ZkSimulator node dist\index.js
nssm set ZkSimulator AppDirectory C:\antigravity-proj\zkteco-simulator\zkteco-simulator-app
nssm set ZkSimulator AppEnvironmentExtra ZK_BIOMETRIC_KEY=... ZK_MATCHER_URL=http://127.0.0.1:28090
```

Order matters at boot: start the sidecar first. If it starts later, nothing is lost —
`reconcile()` rebuilds the matcher's in-memory candidate table from the encrypted store
on every simulator boot.

## 8. Ports & firewall

| Port | Bind | Who needs it |
|---|---|---|
| `4370` | LAN interface | **BITS only** (node-zklib). Never beyond the LAN |
| `3000` / `3005` | LAN / reverse proxy | capture stations (phones), device-face UI, admins |
| `28090` | `127.0.0.1` | the simulator only — keep it loopback |

Windows Firewall: allow Node and Java on **private** networks only. Collisions to check
first: **BITS' Next.js dev server also defaults to `3000`** — run the simulator on
`3005` (`ZK_HTTP_PORT=3005`) when both share a machine. `4370` is occasionally inside a
Hyper-V reserved range; startup pre-flights it and fails with a readable message
(`troubleshooting.md`).

## 9. Upgrades

```bash
git pull && npm ci && npm run build && sudo systemctl restart zkteco-matcher zkteco-simulator
```

At boot the simulator runs a **reconcile** (drop enrollments whose employee or FR-8 wire
copy no longer exists; repopulate the sidecar's candidate table) and a **retention
sweep** (§3). Both are logged, e.g. `INFO biometric reconcile synced matcher synced=5`.
There is no data migration to perform: templates are opaque ciphertext and consent
records are plain JSON.

## 10. Health checks

| Check | Command | Healthy |
|---|---|---|
| Sidecar | `curl -s localhost:28090/health` | `{"ok":true,"engine":"…","templates":N}` |
| Subsystem state | `curl -s localhost:3005/api/biometric/status` | `matcherUp:true`, counts as expected |
| Admin view | open `/biometric` | matcher badge green, rosters listed |
| Wire to BITS | BITS device indicator | `getInfo`, users and the ≤ 30 s attendance sync flowing |

What to watch in the logs: `biometric punch accepted … station=` (attribution),
`biometric punch rejected decision=no-match|ambiguous` (someone failing to present a
finger — never a silent accept), `WARN orphaned biometric template removed` (an employee
was deleted while enrolled), `WARN ZK_BIOMETRIC_KEY not set` (nothing survives a restart).

## 11. Multi-station operation (limits)

Several capture stations run against one simulator — they are browsers hitting the same
REST surface.

- **Attribution:** each station stamps its identity into consent and enrollment records.
  Set `ZK_STATION_ID` per host, or append `?station=<name>` to `/enroll` and `/scan`
  (`http://host:3005/scan?station=lobby-tablet`). `/biometric` shows which station
  enrolled what.
- **Concurrency:** requests are independent; two stations can capture and identify at the
  same time (covered by a concurrency integration test). There is **one shared matcher
  and one shared candidate table**, so throughput is bounded by the sidecar — right for
  demo/small-office volumes, not a cluster.
- **No station registry:** a station is simply "whatever client posts"; nothing is
  registered or heartbeated. A registry would need its own ADR and is listed as future
  work — the lean attribution above is a deliberate choice, not an oversight.

## 12. Deployment checklist

- [ ] `ZK_BIOMETRIC_KEY` set, backed up, and **not** in the repo
- [ ] `data/biometric/` included in the backup set (and git-ignored)
- [ ] `4370` reachable **only** from the BITS host
- [ ] `28090` bound to loopback; the HTTP port behind TLS or LAN-only
- [ ] capture-station pages load in a **secure context** (§6) and the camera opens
- [ ] `ZK_STATION_ID` set per station; `/biometric` shows the stations you expect
- [ ] retention policy chosen (`ZK_RETENTION_DAYS`; `0` = the ADR-011 default) and communicated
- [ ] consent statement version matches what employees are shown (`ZK_CONSENT_VERSION`)
- [ ] the admin knows the two irreversible actions: **withdraw** (one employee) and **purge** (everything)