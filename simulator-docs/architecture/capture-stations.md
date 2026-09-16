# Capture Stations (Phase 7+)

**Status:** camera station, import station and fixture testing implemented in Phase 7 (2026-09-16). The USB-OTG adapter is **deferred** (§3, owner decision — allowance-gated). A capture station is whatever turns a physical finger into a `CaptureSample` the biometric core can consume; every member is optional.

## 1. The contract (ADR-009)

```ts
interface CaptureStation {
  id: 'camera-pwa' | 'otg-scanner' | 'pc-scanner' | 'import';
  capabilities(): { images: boolean; minutiae: boolean; quality: 'demo' | 'sensor' };
  enroll(uid: number, finger: number): Promise<CaptureSample[]>;  // N samples (default 3)
  captureProbe(): Promise<CaptureSample>;                         // one presentation
}
```

Normalization rule: **every station returns ISO 19794-2 bytes** (or an image that the enrollment service converts *before* leaving the station adapter). Nothing downstream sees `kind: 'image'`.

## 2. Camera station (ships first — plan A)

**Why first:** zero hardware, zero app-store friction, works on any phone or laptop with a browser, and it is the only station that can be demonstrated on a moment's notice.

```text
📱 PWA page (/enroll, /scan)                🖥 Simulator core
getUserMedia → live preview                 implemented (Phase 7)
  guidance overlay (finger box, level)      ┌────────────────────────────────────┐
  ├─ capture button → JPEG frame ──────────►│ POST /api/biometric/capture        │
  │                                         │  (raw octet-stream)                │
  │                                         │  extraction  (sidecar /extract:    │
  │                                         │   segmentation → enhancement →     │
  │                                         │   minutiae → ISO 19794-2 +         │
  │                                         │   brightness/sharpness/coverage)   │
  │                                         │  quality gate  (app-side policy:   │
  │                                         │   configurable thresholds)         │
  ◄──────────── quality feedback + retry ───┤  (409 + reason on reject)          │
  │                                         └────────────────────────────────────┘
```

**Capture protocol (UX):**
1. Employee places a finger on a plain high-contrast surface (a white sheet of paper is an explicitly supported "device").
2. The overlay guides framing; the page captures a still frame (avoid motion blur by freezing after a steady-state delay).
3. Server evaluates quality. Rejections return **actionable reasons** (`too-dark`, `too-blurry`, `partial-finger`) so the user can fix it — a bare "failed" is not acceptable UX.
4. Enrollment repeats this N times (default 3) and requires the samples to be mutually consistent before storing.
5. Probe (punch) is a single capture; the resulting template goes straight to identify — **no storage**.

**Honest quality bar** (also in `../status/known-limitations.md`): a camera capture is visibly below a capacitive sensor. Good enough for a portfolio demo and for genuine function at small scale; sensitive to lighting and pose. This is a *documented trade-off of the zero-hardware route*, not a bug to fix later.

## 3. USB-OTG scanner station (deferred — owner decision 2026-09-16, allowance-gated)

`[A]` overall design; specifics validated when a module is obtained. Deferred out of Phase 7 — the camera station ships alone; this adapter plugs into the same contract later with zero core changes (ADR-009's value demonstrated by exactly that).

```text
📱 Android app (or Node CLI on a PC)
  USB Host API → USB-serial bridge (CP2102/CH340/FTDI)
      → sensor module (ZFM-20 / R305 / R307 / R30x class)
      → raw image or on-module template
      → normalized to ISO 19794-2 → POST to core (same endpoint as the camera)
```

- The station reuses the **same HTTP contract** as the camera → the core never learns the difference (ADR-009's value demonstrated).
- Two integration options exist: take the **image** from the module and extract minutiae host-side (keeps one matcher), or take the **on-module template** (vendor-format; would need conversion, which we rejected in ADR-010).
- Validation step: obtain one module, capture a finger, confirm the returned data is usable by the matcher. **Nothing in Phases 1–6 depends on this outcome.**

## 4. Import station (seeding/demo convenience)

Loads ISO templates from files to seed a demo without a live capture session. Purpose: let a fresh clone show a working biometric flow immediately. Imported templates are clearly marked (`source: 'imported'`) so a demo never misrepresents their origin.

## 5. Choosing a station

| Situation | Use |
|---|---|
| Demo on someone else's laptop / no hardware | **camera-pwa** |
| Portfolio recording, showing "real biometrics" | **otg-scanner** if available, else camera |
| Seeding a fresh clone | **import** |
| Automated test of enrollment/identify logic | **import** or a synthetic fixture station (tests never require a camera) |

## 6. Failure & UX flows (shared)

| Condition | Response |
|---|---|
| No camera permission / device | Page explains and offers the import path; core is unaffected |
| Not enrolled | Punch is rejected with "not enrolled" + an enrolment link |
| Matcher offline | Punch returns 503; UI states the service is unavailable. **No auto-accept, ever** |
| Repeated low quality | After N attempts, offer the import/legacy trigger path so a demo can continue |
| Multiple employees claim one capture | Ambiguity rejection (see `biometric-core.md` §5) |

## 7. Testability

A **fixture capture station** (reads pre-made ISO templates from `test/fixtures/`) implements `CaptureStation` and is used by every automated test of enrollment/identify. Hardware and camera are never in a test's dependency path — the important consequence of putting capture behind ADR-009's interface.
