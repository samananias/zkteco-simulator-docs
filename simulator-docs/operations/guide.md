# Live Gate Guide — Phase-7 + Phase-8 Human Validation (Camera-Only)

**Scope:** the human-only validation the automated suite (80/80 green) cannot do.
You start the simulator + matcher, connect BITS (now on your own domain), push real
BITS employees onto the simulated device, enroll a real finger from your phone camera,
punch from your phone, watch it land in BITS as a genuine `FINGERPRINT` punch,
exercise the Phase-8 admin/consent/retention flows, and record your quality judgement.

**Separation to keep in your head:**
- The **simulator** (TCP `4370` + HTTP + sidecar `28090`) runs on **your PC**.
- The **`/enroll` + `/scan` pages** are served by the simulator — your phone talks to
  the simulator, not to BITS.
- The **BITS backend** talks to the simulator over the ZK protocol (`4370`).
  The BITS frontend (your domain) only displays what its backend ingested.

Related: `setup.md` §3b (biometric quick-start) and §4b (BITS checklist),
`deployment.md` (deploy + key management), `troubleshooting.md`,
`../architecture/api-and-events.md` §4 (exact API v2 bodies),
`../architecture/capture-stations.md` §8 (multi-station limits).

---

## 0. What you need before starting

### 0.1 Software

- **Node ≥ 20** (`node -v`).
- **JDK 17+** (`java -version`) — the matcher sidecar will not boot without it.
- **Android platform-tools (`adb`)** — already at
  `C:\Users\Administrator\AppData\Local\Android\Sdk\platform-tools\adb.exe`
  (not on PATH — every command below uses the full path).
- Chrome on the phone.

### 0.2 Network — read this, it decides everything

- Your PC's LAN IP (last seen `192.168.100.8` — re-check with `ipconfig` on the day;
  DHCP can change it).
- Phone + PC on the **same Wi-Fi** *unless* you use `adb reverse` (USB), which bypasses
  Wi-Fi entirely — **this is the recommended path**.

### 0.3 The one concept that confuses everyone: camera needs a "secure context"

Phone browsers **refuse camera access on `http://<lan-ip>:<port>`**. They only allow it on
`http://localhost/...` (treated as secure) or `https://...` (real TLS). That is why we use
`adb reverse`: it makes the phone's own `localhost:3005` tunnel to the PC — no flags,
no HTTPS setup, no Wi-Fi dependency.

### 0.4 BITS-on-a-domain — what changes vs. localhost

- **BITS backend still on this same PC** (only the frontend moved to a domain):
  nothing changes. `ZK_HOST=127.0.0.1` still works.
- **BITS backend on the remote server** (same host as the domain): `127.0.0.1` is wrong
  (it would mean "the server itself"). Set `ZK_HOST` to an address where the server can
  reach your PC's port `4370` — easiest is keeping a local BITS backend for the gate
  test (production domain untouched); harder is a temporary router port-forward of `4370`
  + Windows Firewall opening. Close it after the test window — the ZK protocol has no TLS
  by design (NFR-07).
---
## 1. Prepare the simulator PC

### Step 1.1 — Kill strays

Old `node`/`java` processes hold ports `3000/4370/28090` and cause "address in use"
errors that look like real bugs.

```powershell
netstat -ano | findstr "LISTENING" | findstr ":3000 :4370 :28090"
```

If anything listens and you know it is stale, stop it before continuing.

### Step 1.2 — Confirm your PC's LAN IP

```powershell
ipconfig | findstr "IPv4"
```

Write it down (example: `192.168.100.8`). You need it for §0.4 if BITS is remote,
and for firewall prompts.

### Step 1.3 — Allow the firewall prompts (first run only)

When you start the stack, Windows prompts for **Node** and **Java**. Allow on
**private networks**. Do not allow public networks.

---
## 2. Prepare BITS

### Step 2.1 — Point the BITS backend at the simulator

Open `C:\antigravity-proj\zkteco-simulator\bits\.env` (the single file that serves
backend + frontend + prisma) and set:

```ini
ZK_HOST=127.0.0.1
ZK_PORT=4370
```

Backend on the same PC → `127.0.0.1` (correct for the gate). Backend on your domain
server → the reachable IP of this PC (see §0.4).

### Step 2.2 — Fix the Device row

BITS syncs with the IP stored in its **database `Device` row**, not just `.env`, and it
**skips devices with `isActive=false`**. Either:

- BITS UI → Devices → edit "Main Entrance Biometric" → IP `127.0.0.1`, port `4370` →
  Save → press **"Test Connection"** (this flips `isActive=true`), or
- SQL:

```sql
UPDATE "Device" SET ip='127.0.0.1', port=4370, "isActive"=true, "syncEnabled"=true WHERE id=1;
```

**Verify:** BITS backend log or topbar shows the device **online** once the simulator
is up (§3).

### Step 2.3 — Know your zkIds (this is the trap)

BITS attributes a punch with
`findUnique({ where: { zkId: parseInt(log.deviceUserId) } })`
and **silently skips** unknown IDs (`"Skipping unknown zkId — not in database"`).

Your BITS employees are **zkId 2,3,4,5,6**. The simulator's default seed is
**userId 1001–1005**. Those number spaces do not overlap — punches from seed users are
dropped without error.

**Consequence:** run the simulator with `--zk-seed empty` (§3) so BITS owns the uid
slots, then push employees (§4). Do not enroll `1001–1005` for the gate test.

### Step 2.4 — (Optional but wise) Back up BITS

The gate writes **real attendance rows**. Your employees are dummies so risk is low,
but a dump is cheap:

```powershell
pg_dump db_bits > backup-before-gate.sql
```


---
## 3. Start all runtimes

Three terminals. Keep them open — their logs are your observability.

### Step 3.1 — Terminal 1: simulator + matcher sidecar

```powershell
cd C:\antigravity-proj\zkteco-simulator\zkteco-simulator-app
npm run dev:biometric -- --zk-http-port 3005 --zk-seed empty
```

Why these flags:

- `--zk-seed empty` → empty device so the BITS employee-push owns uid slots 2–6
  (avoids the UID-conflict refusal).
- `--zk-http-port 3005` → frees `3000` (BITS frontend's Next.js default).

**Expected startup sequence:**

1. `[sidecar] MATCHER_SIDECAR_READY port=28090` (within ~10 s)
2. `[dev-biometric] sidecar ready`
3. Simulator log line with `biometric subsystem enabled`

**Verify right now:**

```powershell
curl http://127.0.0.1:3005/api/biometric/status
```

Expect JSON with `"enabled": true, "matcherUp": true`. If `matcherUp` is false the
sidecar did not boot — check `java -version` and port `28090`. `Ctrl+C` later stops
**both** runtimes together.

### Step 3.2 — Terminal 2: BITS backend

```powershell
cd C:\antigravity-proj\zkteco-simulator\bits\backend
npm run dev
```

First run after the move may need `npm install; npx prisma generate` (postinstall runs
patch-package). Fast alternative: copy `node_modules` from the old `C:\bits\backend`
if it still exists.

**Verify:** `curl http://localhost:3001/api/health` → OK.

### Step 3.3 — Terminal 3: BITS frontend (only if testing locally)

```powershell
cd C:\antigravity-proj\zkteco-simulator\bits\frontend
npm run dev
```

If you only use the deployed domain frontend, skip this — but the backend
(Terminal 2) is still mandatory.

---
## 4. Push BITS employees onto the simulated device

### Step 4.1 — Sync employees BITS → device

```powershell
cd C:\antigravity-proj\zkteco-simulator\bits\backend
npm run sync-employees
```

(or `POST /api/employees/sync-to-device`).

**Expect:** all 5 listed (`Admin User` zkId=2 … `sss asdasd` zkId=6 — all ACTIVE) with
success. Our engine accepts BITS's hand-rolled 72-byte user record, so this should
just work.

### Step 4.2 — Confirm on the simulator side

```powershell
curl http://127.0.0.1:3005/api/users
```

**Expect:** 5 users with `userId` `"2"`…`"6"` and the BITS names. If you see
`1001–1005`, you forgot `--zk-seed empty` — restart Terminal 1 with it.

### Step 4.3 — If sync reports UID conflicts

`UID conflict: slot UID=2 occupied by userId="1002"` means the standard seed is loaded.
There is no fix except restarting with `--zk-seed empty`. Do not delete-and-retry one
by one.

---
## 5. Connect the phone (USB reverse — recommended)

### Step 5.1 — Confirm the phone is visible

```powershell
$ADB = "C:\Users\Administrator\AppData\Local\Android\Sdk\platform-tools\adb.exe"
& $ADB devices
```

**Expect:** `10AD9T0XG1000YA  device`. If `unauthorized`, accept the RSA prompt on the
phone and retry. If empty, check cable + USB debugging + USB mode (PTP/file transfer,
not charge-only).

### Step 5.2 — Open the tunnel

```powershell
& $ADB reverse tcp:3005 tcp:3005
```

This maps the phone's `localhost:3005` → PC's `:3005`.

### Step 5.3 — Open the enroll page on the phone

In the **phone's Chrome**, open:

```text
http://localhost:3005/enroll
```

Because it is `localhost`, Chrome grants camera access with a normal permission prompt.
No `chrome://flags` hack needed.

**Fallbacks (only if USB is impossible):**

- **PC webcam:** open `http://localhost:3005/enroll` on the PC itself. `localhost` is
  secure so it works if the PC has a camera.
- **Wi-Fi + Chrome flag:** on the phone visit
  `chrome://flags/#unsafely-treat-insecure-origin-as-secure`, add
  `http://<pc-lan-ip>:3005`, relaunch. Demo-only.
- **iPhone:** no practical insecure-camera exception — use the PC webcam path.

---
## 6. Live enrollment — the UX check (FR-14)

### Step 6.1 — Set up the scene

- Plain **white paper** as background.
- Even, bright light. Avoid backlight and harsh shadow across the fingertip.
- Pick **Admin User (2)** — a BITS-pushed employee, never a seed user.
- Pick a finger (0 = conventionally thumb/right-index — just be consistent).
- **Tick the consent box** (required by ADR-011 — the request is rejected without it).

### Step 6.2 — Capture 3 samples

Press START ENROLLMENT → allow camera → fill the dashed oval with the fingertip →
CAPTURE SAMPLE. **Lift and reposition slightly** between the 3 captures — this is the
ceremony; it proves the samples are mutually consistent, not three copies of one frame.

**Expect per sample:** `✓ sample accepted (quality NN)`, then a success screen.

### Step 6.3 — Deliberately provoke two rejections (this is data, not failure)

1. Finger **half out of frame** → expect `partial-finger`.
2. **Dim the light / half-cover the lens** → expect `too-dark`.
3. (Optional) fast motion / out of focus → `too-blurry`.

Write down **every rejection and its reason** — §9's template asks for counts.

### Step 6.4 — If every sample is rejected

- Move to brighter, diffuse light first (the most common cause).
- Fill more of the oval (coverage gate).
- Hold still ~0.5 s before tapping capture (sharpness gate).
- If normal indoor light keeps saying `too-dark`, note it — a candidate threshold
  change (`ZK_QUALITY_MIN_BRIGHTNESS`, default 60).

### Step 6.5 — Confirm server-side (Phase-8 roster)

On the PC:

```powershell
curl http://127.0.0.1:3005/api/biometric/enrollments
```

**Expect:** your enrollment listed with `userId:"2"`, `source:"camera"`,
`station:"camera-1"`, quality, timestamp — and **no `templateB64` field anywhere**.
Absence of template bytes in rosters is a privacy requirement (ADR-011), not an
omission.

Also open `http://localhost:3005/biometric` (the Phase-8 admin view): status card +
enrollment row + consent row should all be visible.

---
## 7. Punch — identify (FR-15)

### Step 7.1 — The happy path

On the phone open `http://localhost:3005/scan` → place the **same finger** →
CAPTURE & PUNCH.

**Expect:** green **✔ + "Admin User" + score** (real SourceAFIS scores land in the
hundreds for same-finger, single digits for impostors).

### Step 7.2 — The negative control (do not skip)

Present a **different, un-enrolled finger** → red **✘ "Not recognized"** — and no
attendance record is created. A biometric that cannot say "no" is a button, not
a matcher.

### Step 7.3 — Watch the device face mirror it

Keep `http://localhost:3005/` open on the PC while punching from the phone. The
**same green/red overlay** appears live — the WebSocket fan-out (`verify`/`punch`
events) working.

### Step 7.4 — If "Not recognized" on the right finger

- You are presenting a different finger than enrolled (most common — finger 0 vs
  finger 1).
- Or lighting/pose changed drastically — recapture with the §6.1 setup.
- Ground truth: check `/api/biometric/status` — `matcherUp` must be true; sidecar
  down → 503 "service unavailable", and the server **never auto-accepts** (NFR-12).

---
## 8. Watch BITS ingest it (the headline proof)

### Step 8.1 — What to watch

BITS backend console (Terminal 2). Within ~30 s of a punch you should see, in order:

```text
[ZK] "Main Entrance Biometric" watermark: …
[ZK] Syncing device … at 127.0.0.1:4370
[ZK] Fetched N total logs, filtered to M logs newer than watermark.
```

### Step 8.2 — Punch 2–3 times, then check BITS

- BITS UI (your domain or local `:3000`): topbar device indicator **online**;
  **Attendance** page shows the punches as **FINGERPRINT** (not password/generic) —
  that `verifyType=1` → `FINGERPRINT` mapping is the proof the punch was genuinely
  matched.
- If the UI is slow, SQL ground truth:

```sql
SELECT timestamp, "employeeId", metadata FROM "AttendanceLog" ORDER BY timestamp DESC LIMIT 5;
```

### Step 8.3 — Failure dictionary (you will likely meet one of these)

| What you see | What it means | Fix |
|---|---|---|
| `[ZK] Skipping unknown zkId 1002` | Punched a seeded user, not a BITS employee | Enroll/punch userId 2–6; confirm `--zk-seed empty` |
| `Skipping … device is currently offline` / `sync is disabled` | Device row `isActive=false` / `syncEnabled=false` | §2.2 — "Test Connection" |
| `UID conflict: slot UID=2 occupied by userId="1002"` during push | Standard seed still loaded | Restart simulator with `--zk-seed empty` |
| Topbar offline but backend log shows sync lines | `.env` `ZK_HOST` ≠ Device-row IP | Make them agree (both `127.0.0.1` locally) |
| Punches on simulator (`/api/attendance`) but never in BITS | Watermark/scheduler lag or wrong device IP | Wait 30 s, check §8.1 lines, recheck §2.2 |

---
## 9. Phase-8 admin flows (consent, withdraw, purge)

### Step 9.1 — Consent roster

```powershell
curl http://127.0.0.1:3005/api/biometric/consents
```

Each enrollment wrote a **plaintext JSON** consent record (userId, statement
version+hash, purpose `attendance-identification`, method, station, timestamp).
Plaintext is deliberate — consents contain no biometrics; only templates are
AES-256-GCM ciphertext.

### Step 9.2 — Withdraw one employee (employee-initiated delete)

`POST /api/biometric/withdraw` `{"userId":"2"}` (or the button on `/biometric`).
This removes **all three copies** (encrypted file + wire record + sidecar candidate)
and marks the consent withdrawn. Re-punching that finger must now return `no-match`.

### Step 9.3 — Full purge (end of session, ADR-011 §5)

On `/biometric`, type `PURGE` to confirm (typed confirmation is intentional — no-auth
LAN surface). Or:

```powershell
curl -X POST http://127.0.0.1:3005/api/biometric/purge -H "content-type: application/json" -d '{"confirm":true}'
```

**Expect:** `{ok:true, purged:N}`, status shows
`enrolled=0 matcherTemplates=0 consents=0`, and the data dir holds **zero files**.
Purge clears consent records too — full wipe means full wipe.

---
## 10. Record your human-judged quality assessment

Camera frames are **never stored** (ADR-011 §1 — they exist only inside the capture
call), so no log can substitute for your judgement. Fill this in while testing and
paste it back so it can be recorded in `capture-stations.md`:

```text
Device & camera used:
Access path (USB reverse / PC webcam / Wi-Fi+flag):
Lighting tried (bright / indoor / dim) -> accept rate each:
Distance & pose that worked / failed:
Rejections observed (too-dark / too-blurry / partial-finger — counts):
Punch success rate (accepted / total attempts):
Negative control result (un-enrolled finger rejected?):
BITS ingest confirmed (FINGERPRINT rows visible? watermark lines seen?):
Quality vs a capacitive sensor (2 honest sentences):
Suggested threshold changes (e.g. normal indoor triggers too-dark ->
  propose a new ZK_QUALITY_MIN_BRIGHTNESS value, default 60):
Multi-station note (did station "camera-1" attribution show correctly?):
```

---
## 11. Wrap-up

1. **Purge** (§9.3) if the enrolled finger was a real one — or keep it knowingly
   (AES-256-GCM at rest, `data/` is git-ignored, never committed).
2. `Ctrl+C` all three terminals (Terminal 1 stops sidecar + simulator together).
3. Confirm silence: ports `3005/4370/28090` free, no stray `node`/`java`.
4. **Never paste finger images, templates, or `.bio` contents** into chat, tickets, or
   either repo — there should be none to paste (nothing is stored), which is exactly
   why §10 has to come from you.
