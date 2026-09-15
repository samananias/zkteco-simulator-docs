# Biometric Core (Phase 6+)

**Status:** core implemented in Phase 6 (matcher sidecar, encrypted store, enrollment/identify/verify service, REST v2 — see ADR-010 spike results); capture *stations* are Phase 7, so live capture still enters through the REST surface. This document defines the design the implementation follows.

**Reading order:** `../research/biometrics.md` (what is possible) → `../decisions/ADR-009*.md`, `ADR-010*.md`, `ADR-011*.md` → this document → `capture-stations.md`.

## 1. What "ZKTeco substitute" means precisely

| Dimension | v1 (Phases 1–5): simulator | Phase 6+: substitute |
|---|---|---|
| Attendance recorded | UI/REST/CLI trigger (`forceFail` can fake a reject) | **Real capture → real template → real 1:N identify** |
| Verify type in records | `verifyType=1` cosmetic | `1` genuinely earned by a match |
| Templates | Synthetic `SS21`-style blobs (FR-8) | **Real ISO/IEC 19794-2 minutiae templates** from real fingers |
| Interoperability | Protocol-compatible with ZK clients | Same, **plus** an open REST surface any system can integrate |
| Boundary that remains | — | Templates are *not* matchable by real ZKTeco firmware (proprietary minutiae format) |

The **ZK protocol surface stays frozen** across that transition: BITS keeps working with zero code changes, and the punches it ingests are now genuinely matched. That is the point of the sequencing — the substitute is a *capability addition*, not a rewrite.

## 2. Position in the system

```text
capture station (see capture-stations.md)
        │  frames / samples (HTTP multipart or local device)
        ▼
┌──────────────────────────────────────────────────────────────┐
│ Enrollment service                                           │
│  1. collect N samples (default 3) per finger                 │
│  2. quality gate per sample (reject blurry/partial)          │
│  3. extract minutiae → ISO/IEC 19794-2 template              │
│  4. cross-check the samples' templates against each other    │
│  5. store encrypted (ADR-011) + audit entry                  │
└──────────────────────────────────────────────────────────────┘
        ▼
┌──────────────────────────────────────────────────────────────┐
│ BiometricStore (behind an interface, like DeviceStore)       │
│  templateRef, uid, finger, format, quality, createdAt        │
│  → ciphertext at rest; no plaintext minutiae on disk         │
└──────────────────────────────────────────────────────────────┘
        ▲
        │ 1:N search (probe template → candidate list)
┌──────────────────────────────────────────────────────────────┐
│ Matching engine (ADR-010: out-of-process, HTTP/JSON)         │
│  identify(probe) → ranked candidates · verify(probe, uid)    │
└──────────────────────────────────────────────────────────────┘
        ▲
        │ probe template
   Punch controller: capture → extract → identify → record
        │
        ─► DeviceStore.appendAttendance({ verifyType: 1, … })
                 ├─► event bus → EF_ATTLOG to registered sessions (FR-9)
                 └─► WS → device-face UI shows the *employee's name*
```
## 3. Design rules (why the seams are where they are)

1. **The protocol engine never learns about biometrics.** It receives an `AttendanceRecord`, exactly as it does today. Biometrics enter upstream, through the punch controller.
2. **Matching runs out-of-process** (ADR-010). A JVM AFIS is the mature option; keeping it behind HTTP/JSON means a future pure-Node/WASM engine can replace it without touching device logic — and the protocol engine's test suite never needs a Java runtime.
3. **Templates are opaque blobs to everything except the matcher.** This mirrors how real firmware treats them and keeps the store's contract identical to FR-8's, just with real bytes.
4. **The enrollment ceremony has the same shape as the device's** — capture N times, quality-check, store — so `CMD_STARTENROLL` (FR-8, currently ACK-only) can later be wired to this same service instead of staying cosmetic. A backend-driven enrollment demo then works end to end.
5. **Nothing in Phases 1–5 may assume synthetic blobs are permanent.** `FpTemplate` carries a `format` field (`synthetic` | `iso19794-2`) from the start, so no data migration is ever needed.

## 4. Data model delta (extends `data-model.md`)

```ts
interface BiometricTemplate {
  templateRef: string;          // opaque key; ciphertext lives in the biometric store
  uid: number;                  // same identity as the wire record
  finger: 0 | 1 | 2 | 3 | 4;    // ZKTeco finger numbering runs 0–9; v1 uses 0–4
  format: 'iso19794-2' | 'synthetic';
  quality: number;              // 0–100 from the quality gate
  createdAt: Date;
  source: 'camera' | 'usb-otg' | 'pc-scanner' | 'imported';
}

interface IdentifyResult {
  matched: boolean;
  uid?: number;                 // absent when no candidate clears the threshold
  userId?: string;
  score: number;                // matcher score (engine-specific scale)
  runnerUpScore?: number;       // separation = confidence evidence, not just top score
  candidates?: { uid: number; score: number }[];
}
```

Wire mapping: `uid`/`finger` are the same coordinates FR-8 uses, so `CMD_USERTEMP_RRQ` answers with a real template once one exists — and the BITS backend's `getFingerTemplate` path starts returning real data with **no code change**.

## 5. Failure modes and their handling (defined now, implemented in Phase 6)

| Failure | Handling |
|---|---|
| No candidate clears the accept threshold | Punch **rejected**: red fail screen, no attendance record. Matches real device behaviour. |
| Ambiguous (runner-up too close to winner) | Reject as ambiguous; require a second presentation (retry counter on the UI). Never silently pick. |
| Matcher unavailable (sidecar down) | HTTP 503 with a clear message; UI shows "service unavailable". **Never** fall back to auto-accept — that would corrupt attendance data. |
| Capture quality too low | Enrollment refuses the sample with actionable feedback (lighting / blur / coverage). |
| Employee has no enrolled template | Rejected with "not enrolled"; enrollment path offered. |

## 6. What this deliberately is not

- Not parity with a capacitive sensor (see `../status/known-limitations.md`).
- Not multi-finger or multi-modal in the first cut — one finger per employee is enough to be real.
- Not liveness/anti-spoofing: a good photograph of a fingerprint could pass. Documented, not hidden.
- Not a replacement for BITS: the backend stays the system of record for attendance; this core adds the terminal's front door.
