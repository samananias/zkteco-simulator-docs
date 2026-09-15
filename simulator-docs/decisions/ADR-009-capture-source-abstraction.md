# ADR-009: Capture-source abstraction — biometrics enter as pluggable stations

**Status:** accepted · **Date:** 2026-09-15

## Context

The substitute ambition requires enrolling real fingerprints and matching them at punch time. Research (`../research/biometrics.md`) established that **phone built-in and in-display sensors are unusable** (`[V]` platform isolation — no capture or enrollment API), while three legitimate sources exist with very different cost/quality profiles: camera touchless (₱0, demo-grade), USB-OTG scanner (~₱1–2.5k, real quality), PC scanner (~₱1–3k, convenient). Hardware availability is not guaranteed at any given moment, and a demo must never be blocked by a missing scanner.

## Decision

All biometric input enters the system through a single **capture-station interface**; nothing downstream knows which station produced the sample.

```ts
interface CaptureStation {                    // one adapter per source
  id: string;                                 // 'camera-pwa' | 'otg-scanner' | 'pc-scanner' | 'import'
  capabilities(): { images: boolean; minutiae: boolean; quality: 'demo' | 'sensor' };
  enroll(uid: number, finger: number): Promise<CaptureSample[]>;   // N samples
  captureProbe(): Promise<CaptureSample>;                          // one presentation
}
interface CaptureSample {
  stationId: string;
  kind: 'image' | 'minutiae';
  bytes: Buffer;
  quality?: number;
  capturedAt: Date;
}
```

- The **camera station ships first** (Phase 7) because it needs no hardware and no app store.
- The **USB-OTG station is a second adapter**, added without touching enrollment, matching, or storage.
- Every sample is normalized at the boundary: images are converted to ISO 19794-2 templates **before** leaving the station adapter, so the core only ever sees one format.
- Stations may be **remote** (a phone browser talking HTTP to the core) or local (a USB device on the host).

## Consequences

- (+) A missing scanner cannot block development or a demo — the camera station is always available.
- (+) Adding a capture technology is an adapter, never a refactor; a future NFC/face station slots in the same way.
- (+) The substitute's "real proof" is hardware-independent: any station produces a template the matcher can use.
- (−) Two capture paths must be documented and tested (camera + one scanner), and the camera path carries an explicit quality caveat — acceptable, because the abstraction keeps the caveat local to one adapter.
- (−) Station identity must be recorded on captures (`source` on `BiometricTemplate`) so a demo can show which route produced what — small bookkeeping cost, real audit value.
