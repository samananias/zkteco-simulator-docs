# ZKTeco Standalone Protocol — Verified Notes

**Status:** research basis for implementation. Every claim tagged `[V]` (verified against the cited source this project) or `[A]` (assumption + validation step). Primary sources:

- **Spec:** `adrobinoga/zk-protocol` — `protocol.md` + `sections/{realtime,ex_data,data-user,data-record,terminal}.md` (reverse-engineered from a real f19/TFT device; includes captured packets).
- **pyzk** (`fananimi/pyzk`): `zk/const.py`, `zk/user.py`, `zk/base.py` — most battle-tested independent implementation.
- **node-zklib 1.3.0** — the exact npm artifact installed in `C:/bits/backend/node_modules/node-zklib/` (our validation oracle).
- Real captured packet embedded in `zk-protocol/sections/realtime.md`.

---

## 1. TCP framing `[V]`

```
offset 0  magic      4 B   50 50 82 7D
offset 4  length     4 B   u32-LE: size of (8-byte header + data), i.e. everything after offset 8
offset 8  command    2 B   u16-LE  request command (client→device) or reply code (device→client)
offset 10 checksum   2 B   u16-LE  see §2
offset 12 sessionId  2 B   u16-LE
offset 14 replyNo    2 B   u16-LE
offset 16 data       n B   length − 8
```

Cross-checks: the spec's realtime example `50 50 82 7D 28 00 00 00 F4 01 AC 12 01 00 00 00 …` decodes as length 40, command 0x01F4 = 500 (`CMD_REG_EVENT`), event code 1 (`EF_ATTLOG`) in the session field, reply 0 — matches this table. node-zklib's `createTCPHeader` writes the payload length as u16 at prefix offset 4 (u32-equivalent since high half stays 0) `[V]`. **Consequence: a TCP "data event" is not necessarily one frame — the engine must buffer and reassemble frames across socket chunks.**

## 2. Checksum `[V] two variants — see ADR-005`

Both published implementations reduce the same sum mod 65535 (high-16 folding ≡ mod 65535) but differ in the final step:

- **Classic (spec / pyzk):** sum u16-LE words of the 8-byte header + data (checksum field zeroed; odd tail byte added alone), fold high 16 into low, then one's complement → `65535 − s`.
- **Oracle (node-zklib 1.3.0 `createChkSum`):** same accumulation with `%= 65535` each step, then `chksum = 65535 − s − 1` → `65534 − s`.

They differ by exactly 1. Which one real firmware expects is **unresolved without hardware** — but the oracle never validates reply checksums (`decodeTCPHeader` ignores the field) `[V]`, so for our purposes: **inbound: accept both variants (log which matched); outbound: classic by default, config-switchable.** No oracle client can break either way.

## 3. Session & reply-number rules `[V]`

- `CMD_CONNECT` (1000): client sends session 0 / reply 0; device replies `CMD_ACK_OK` (2000) with the **assigned session ID in the session field** (node-zklib reads it at offset 4 after header strip). Session ID is constant for the connection.
- Replies echo the request's reply number. Exception: realtime event packets use reply number 0 and put the event code in the session field.
- `CMD_DATA_RDY` → `CMD_PREPARE_DATA` → `CMD_DATA` → `CMD_ACK_OK` chunk sequence shares one reply number (spec §ex_data) — only relevant in chunked mode (ADR-004).
- `CMD_STATE_RRQ` (64) returns the state as the session-field value of the reply.

## 4. Command & event codes `[V] — pyzk const.py ≡ node-zklib 1.3.0 constants.js`

| Code | Name | Code | Name | Code | Name |
|---|---|---|---|---|---|
| 7 | CMD_DB_RRQ | 500 | CMD_REG_EVENT | 1101 | CMD_CHANGE_SPEED |
| 8 | CMD_USER_WRQ | 1000 | CMD_CONNECT | 1102 | CMD_AUTH |
| 9 | CMD_USERTEMP_RRQ | 1001 | CMD_EXIT | 1500 | CMD_PREPARE_DATA |
| 10 | CMD_USERTEMP_WRQ | 1002 | CMD_ENABLEDEVICE | 1501 | CMD_DATA |
| 11 | CMD_OPTIONS_RRQ | 1003 | CMD_DISABLEDEVICE | 1502 | CMD_FREE_DATA |
| 12 | CMD_OPTIONS_WRQ | 1004 | CMD_RESTART | 1503 | CMD_DATA_WRRQ |
| 13 | CMD_ATTLOG_RRQ | 1005 | CMD_POWEROFF | 1504 | CMD_DATA_RDY |
| 14 | CMD_CLEAR_DATA | 1006 | CMD_SLEEP | 2000 | CMD_ACK_OK |
| 15 | CMD_CLEAR_ATTLOG | 1007 | CMD_RESUME | 2001 | CMD_ACK_ERROR |
| 18 | CMD_DELETE_USER | 1013 | CMD_REFRESHDATA | 2002 | CMD_ACK_DATA |
| 19 | CMD_DELETE_USERTEMP | 1014 | CMD_REFRESHOPTION | 2005 | CMD_ACK_UNAUTH |
| 31 | CMD_UNLOCK | 1100 | CMD_GET_VERSION | 65535 | CMD_ACK_UNKNOWN |
| 50 | CMD_GET_FREE_SIZES | 61 | CMD_STARTENROLL | 87 | CMD_TMP_WRITE |
| 57 | CMD_ENABLE_CLOCK | 62 | CMD_CANCELCAPTURE | 119 | CMD_CHECKSUM_BUFFER |
| 66 | CMD_WRITE_LCD | 64 | CMD_STATE_RRQ | 134 | CMD_DEL_FPTMP |
| 67 | CMD_CLEAR_LCD | 201/202 | CMD_GET/SET_TIME | | |

Event flags: `EF_ATTLOG=1, EF_FINGER=2, EF_ENROLLUSER=4, EF_ENROLLFINGER=8, EF_BUTTON=16, EF_UNLOCK=32, EF_VERIFY=128, EF_FPFTR=256, EF_ALARM=512`. Privilege levels: 0 user, 2 enroller, 6 manager, 14 admin `[V] pyzk`.

## 5. Record layouts `[V]`

### User record — 72 bytes (TFT/TCP; node-zklib + backend driver agree)
| Offset | Size | Field | Encoding |
|---|---|---|---|
| 0 | 2 | uid | u16-LE internal index |
| 2 | 1 | role / privilege | 0 user · 14 admin |
| 3 | 8 | password | ascii, NUL-padded |
| 11 | 24 | name | ascii, NUL-terminated |
| 35 | 4 | card number | u32-LE |
| 48 | 9 | userId (visible PIN) | ascii, NUL-padded |

pyzk's *write* variant `repack73` prepends a `0x02` tag byte (73 B total); the BITS driver writes **plain 72 B with no tag**. The simulator's `USER_WRQ` parser must accept both (auto-detect by length/first byte).

### Attendance record — 40 bytes
| Offset | Size | Field | Values |
|---|---|---|---|
| 0 | 2 | userSn (internal) | u16-LE |
| 2 | 9 | deviceUserId | ascii |
| 11 | 15 | fixed | zeros |
| 26 | 1 | verifyType | 0 password · 1 fingerprint · 2 card |
| 27 | 4 | time | u32-LE, §6 codec |
| 31 | 1 | state | 0 in · 1 out · 2 break-out · 3 break-in · 4 OT-in · 5 OT-out |
| 32 | 8 | fixed | `00 00 00 00 FF 00 00 00` |

Dataset frame for reads: `[u32-LE total bytes of records][records…]` — node-zklib slices the 4-byte prefix, pyzk reads it as a length `[V]`.

### Realtime EF_ATTLOG event data — 32 bytes
userId ascii 9 B@0 · zeros 15 B@9 · verifyType u16-LE@24 · time 6 B@26 as raw bytes `20YY MM DD HH MM SS` (e.g. `12 06 19 11 29 05` = 2018-06-25 17:29:05 — example packet verified in spec).

## 6. Device time codec `[V] — identical in pyzk, node-zklib, and the BITS driver`
```
seconds = ((year-2000)*12*31 + (month-1)*31 + (day-1)) * 86400
        + (hour*60+minute)*60 + second
```
⚠ It is **not** unix-like: months are 31 days, years 12 months (372 days). `GET_TIME`/`SET_TIME` carry u32-LE in this form; realtime events carry the 6-byte raw form.

## 7. Large-data flows `[V]`

**Small read (oracle primary):** `CMD_DATA_WRRQ` → device answers immediately with `CMD_DATA` frame(s) = dataset (§5), all sent back-to-back (client settles on 1 s of silence).

**Chunked read (spec; optional, ADR-004):**
```
> CMD_DISABLEDEVICE → ACK_OK
> CMD_DATA_WRRQ(payload) → ACK_OK + data-stat(00 + u32 size @1 + …)
> CMD_DATA_RDY(start u32, size u32)   ×N chunks of 65472
  ← CMD_PREPARE_DATA → CMD_DATA(chunk) → CMD_ACK_OK   (same reply number)
> CMD_FREE_DATA → ACK_OK → CMD_ENABLEDEVICE → ACK_OK
```

**Client→device template write (BITS driver's exact sequence):**
`DISABLEDEVICE[00000000]` → `DELETE_USERTEMP[uid,fid]` → `PREPARE_DATA[size u16, 00 00]` → `CMD_DATA[blob]` → `CHECKSUM_BUFFER['']` → `TMP_WRITE[uid u16, fid u8, flag u8, size u16]` → `FREE_DATA` → `REFRESHDATA` → `ENABLEDEVICE`.

## 8. Authentication (comm key) `[V]`
`makeCommKey`: bit-reverse the 32-bit key, add session ID, pack LE, XOR `ZKSO`, swap 16-bit halves, XOR each byte with `ticks` (50) — byte 2 replaced by the tick value. Identical in pyzk and node-zklib. Only relevant if a comm key is configured (backend: **none**).

## 9. Open items carried into Phase 1
- `[A]` Chunked-mode 8-byte per-chunk sub-header composition (only if ADR-004 chunk mode is exercised).
- `[A]` Exact options keys a future client might require beyond the backend's set (non-blocking: unknown keys return `ACK_UNKNOWN` + empty value).
- `[A]` pyzk end-to-end behavior as a *second* oracle (its `CMD_DB_RRQ`/`CMD_ATTLOG_RRQ` read style is implemented per spec but not yet integration-tested).

