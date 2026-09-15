# Known Limitations

Honest scope statement — what this simulator is and is not. Updated whenever a new boundary is discovered.

## Fundamental (by design)
1. **No biometric capture in v1.** A "scan" is a triggered event (UI tap / REST call / CLI); the simulator emits the *result* a real device would, from synthetic templates. **Superseded from Phase 6** — real capture and real matching arrive via `../architecture/biometric-core.md`; the boundaries that remain are listed in §Biometric boundaries below.
2. **Not bit-for-bit hardware behavior.** Error/retry edge behaviors (malformed sequence handling, device-specific timing quirks, `CMD_ACK_RETRY` semantics) are approximated from spec + client expectations, not observed from real firmware.
3. **Single-socket multiplexing:** events and commands share one TCP stream (protocol-inherent). Events are gated/queued (ADR-006); an event-consuming client that also pipelines commands may see delayed events.

## Protocol coverage (v1)
4. **TCP only** (ADR-003). pyzk `force_udp=True` and other UDP clients will not connect.
5. **Command set = backend inventory + spec basics** (`../research/client-libraries.md` §2.2). Unimplemented commands (SMS, Mifare, operation logs, user groups/timezones, access-control modules) reply `CMD_ACK_UNKNOWN` — tolerated by clients, but a client *requiring* those features would notice.
6. **`CMD_STARTENROLL` acks but does not perform a real multi-step enrollment ceremony** (optionally simulated in a later phase).
7. **Options store is shallow** — keys the backend/clients read are populated; exotic device parameters return empty values.

## Fidelity
8. **Clock:** the 372-day calendar is lossy for real dates (months > 31 days don't exist in it); round-trips are exact, arithmetic on wire values is not wall-clock time.
9. **Fingerprint blobs** are structurally plausible but synthetic (v1); a real system could not match them. From Phase 6 templates become **real ISO/IEC 19794-2**, but see §Biometric boundaries — they still do not interoperate with ZKTeco firmware.
10. **UI is a replica of the *class*** of TFT devices (iFace/K30-style), not a pixel trace of one specific model.

## Operational
11. **No auth/TLS anywhere** — trusted-LAN tool only (`../operations/security.md`).
12. **Single device instance** per process; multi-device needs multiple instances (future work).
13. **Biometric templates are key-encrypted per deployment** (ADR-011); losing `ZK_BIOMETRIC_KEY` makes them unrecoverable — re-enrolment is the only recovery path.

## Biometric boundaries (permanent — apply from Phase 6)

14. **Phone built-in / in-display sensors are unusable as capture devices.** Platform isolation (TEE / Secure Enclave): there is no API for raw capture and no app-facing enrollment API; the sensor → match → boolean path never leaves the secure environment. Camera or an external scanner only — this is a design guarantee, not a gap we can close `[V]` (`../research/biometrics.md` §2).
15. **Templates are not interchangeable with real ZKTeco hardware.** Our engine enrols **ISO/IEC 19794-2**; ZKTeco firmware uses proprietary ZKFinger-family formats with no published converter. A real terminal cannot match our templates, and we cannot match its templates (we *can* store and replay them opaquely — FR-8 is unchanged). "Substitute" therefore means **a real biometric terminal within our own ecosystem** (capture stations ↔ simulator ↔ BITS), not a template donor to physical hardware.
16. **Camera capture quality is demo-grade.** Lighting/pose sensitive and clearly below a capacitive sensor; the quality gate rejects bad captures rather than accepting them. Suitable for demos and small-scale genuine use, not high-assurance deployments.
17. **No liveness / anti-spoofing.** A photograph of a fingerprint could pass a camera station. Not claimed otherwise.
18. **The matcher is a separate runtime** (ADR-010) — the biometric deployment needs a JVM; the protocol engine, BITS compatibility and the base demo never depend on it existing.

## How we compensate
Deviations that matter are compensated by the oracle gate (anything the backend does is exact), the seed data (demos look real), and this document (nothing is silently fake).
