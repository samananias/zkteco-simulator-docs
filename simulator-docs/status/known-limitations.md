# Known Limitations

Honest scope statement — what this simulator is and is not. Updated whenever a new boundary is discovered.

## Fundamental (by design)
1. **No biometric capture.** A "scan" is a triggered event (UI tap / REST call / CLI). The simulator emits the *result* a real device would, not a capture. It will never match on a fingerprint image.
2. **Not bit-for-bit hardware behavior.** Error/retry edge behaviors (malformed sequence handling, device-specific timing quirks, `CMD_ACK_RETRY` semantics) are approximated from spec + client expectations, not observed from real firmware.
3. **Single-socket multiplexing:** events and commands share one TCP stream (protocol-inherent). Events are gated/queued (ADR-006); an event-consuming client that also pipelines commands may see delayed events.

## Protocol coverage (v1)
4. **TCP only** (ADR-003). pyzk `force_udp=True` and other UDP clients will not connect.
5. **Command set = backend inventory + spec basics** (`../research/client-libraries.md` §2.2). Unimplemented commands (SMS, Mifare, operation logs, user groups/timezones, access-control modules) reply `CMD_ACK_UNKNOWN` — tolerated by clients, but a client *requiring* those features would notice.
6. **`CMD_STARTENROLL` acks but does not perform a real multi-step enrollment ceremony** (optionally simulated in a later phase).
7. **Options store is shallow** — keys the backend/clients read are populated; exotic device parameters return empty values.

## Fidelity
8. **Clock:** the 372-day calendar is lossy for real dates (months > 31 days don't exist in it); round-trips are exact, arithmetic on wire values is not wall-clock time.
9. **Fingerprint blobs** are structurally plausible but are not real templates; converting them to a format another system could match would fail (by design).
10. **UI is a replica of the *class*** of TFT devices (iFace/K30-style), not a pixel trace of one specific model.

## Operational
11. **No auth/TLS anywhere** — trusted-LAN tool only (`../operations/security.md`).
12. **Single device instance** per process; multi-device needs multiple instances (future work).

## How we compensate
Deviations that matter are compensated by the oracle gate (anything the backend does is exact), the seed data (demos look real), and this document (nothing is silently fake).
