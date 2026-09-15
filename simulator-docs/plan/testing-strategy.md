# Testing Strategy

**Principle:** the simulator is correct when the *unmodified pinned oracle* works against it — validation by construction, exactly as the original brief proposed. Everything below serves that principle.

## 1. Test layers

### L1 — Unit (no sockets, `node:test`)
- **Codec golden bytes:** hand-computed vectors for frame build/parse (magic, LE fields), both checksum variants on known inputs (including the realtime example packet from the spec, whose checksum byte is printed in the spec `[V]`).
- **Record round-trips:** User → 72 B → User (byte-exact buffer assertions against layouts in `zk-protocol-notes.md` §5); attendance 40-B including the `FF` tail; event 32-B with the spec's decoded example.
- **Time codec:** vector table from the BITS driver's formula; round-trip ±0 s; boundary dates (Dec 31, Feb 30(!), leap-day handling of the fictitious calendar documented).
- **Store:** CRUD, uid reuse after delete, seed idempotency.
- **Command handlers:** pure request→reply assertions with a fake session.

### L2 — Integration (real oracle client, real sockets)
Dev-dependency `node-zklib@1.3.0` **pinned exact** (same artifact as `C:/bits/backend`, patch applied or not — the patch is control-flow only). Tests boot the simulator on an ephemeral port and replay the backend's call patterns:
1. `createSocket → connect → getInfo → getTime → disconnect` (Phase 1 gate)
2. `getUsers → setUser → getUsers → deleteUser → clearAttendanceLog → getAttendances` (Phase 2 gate)
3. `getFingerCount → getFingerTemplate → setFingerTemplate` (Phase 2–3 gate)
4. `getRealTimeLogs` + REST punch → event received, layout decoded (Phase 3 gate)
5. **Desync matrix:** registered session + concurrent `getAttendances`; unregistered session receives zero unsolicited frames (Phase 3 gate)
6. Chunked-mode read (config on) vs direct burst (ADR-004)
7. Robustness: garbage bytes, truncated frames, unknown commands → session still usable afterwards

### L3 — End-to-end (scripted, no test framework)
`npm run demo` boots the simulator, fires punches via `scripts/punch.mjs`, and the BITS backend (optional, via `ZK_HOST=127.0.0.1`) ingests them within one poll interval — the literal portfolio demo, runnable on demand.

### L4 — Manual visual QA (Phase 4)
Bezel/glyph/typography checklist in `web-ui.md`; screenshots archived in the app repo.

### L5 — Biometric (Phase 6–8; never depends on hardware)
- **Matcher spike bench** (ADR-010 criteria): 10 enrolled templates → 10/10 correct identify; impostor rejected; latency recorded.
- **Enrollment unit tests** via the **fixture capture station** (pre-made ISO templates in `test/fixtures/`): N-sample consistency gate, quality rejections (`too-dark`, `too-blurry`, `partial-finger`), image never persisted.
- **Store tests:** ciphertext at rest (assert the raw bytes on disk/`Buffer` are not the plaintext template), key-missing behaviour, delete/purge leaves nothing.
- **Leak tests:** packet log and API responses contain `<TEMPLATE n bytes>` redactions, never template bytes.
- **Degradation tests (NFR-12):** with the matcher and all stations absent — protocol suite still green, biometric endpoints return 503, and **no punch is ever auto-accepted**.
- **Protocol non-regression (the critical one):** with real stored templates, the oracle's `getFingerCount` / `getFingerTemplate` / `setFingerTemplate` / `deleteUserTemplate` call patterns still pass byte-for-byte against FR-8.
- **Camera station (Phase 7):** scripted browser check for permission-denied and quality-rejection UX; manual quality assessment on real captures is a *documented, human-judged* step (not an automated pass/fail).

## 2. Running
```bash
npm install            # installs pinned oracle as devDependency
npm test               # L1 + L2
npm run test:chunked   # L2 subset with chunked mode enabled
npm run demo           # L3
```

## 3. CI (later, optional)
GitHub Actions: node 20/22 matrix, `npm ci && npm test`. Protocol tests are hermetic (ephemeral ports) so CI needs nothing else.

## 4. What we deliberately do NOT test
Pixel-perfect device firmware behavior we cannot observe (no hardware) — those are documented as approximations in `../status/known-limitations.md`, not hidden behind green checks.
