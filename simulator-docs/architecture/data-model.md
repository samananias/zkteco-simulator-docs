# Data Model

## 1. Entities (in-memory store, mirrored later by SQLite)

```ts
interface User {            // maps 1:1 to the 72-byte wire record
  uid: number;              // u16 internal index — unique, reused after delete
  userId: string;           // visible PIN (≤ 9 ascii chars) — backend's badge/zkId
  name: string;             // ≤ 24 ascii
  privilege: 0 | 2 | 6 | 14;
  password: string;         // ≤ 8 ascii
  cardno: number;           // u32
  groupId?: number;         // reserved bytes; kept for completeness
}

interface AttendanceRecord {
  seq: number;              // internal monotonic id (not on the wire)
  userSn: number;           // u16 snapshot of uid at record time
  deviceUserId: string;     // what the backend keys on
  timestamp: Date;          // converted via the 372-day codec both ways
  verifyType: 0|1|2;        // password | fingerprint | card
  state: 0|1|2|3|4|5;       // in/out/break-out/break-in/OT-in/OT-out
}

interface FpTemplate { uid: number; finger: 0-9; flag: 1; blob: Buffer; } // synthetic ≥ 500 B
interface DeviceOptions { [key: string]: string }   // ~SerialNumber, ~DeviceName, ~Platform, ~OS, ~ZKFPVersion, ~PIN2Width, FingerFunOn, FaceFunOn, SDKBuild…
interface LcdState { lines: string[]; until?: Date }
interface DeviceState { enabled: boolean; locked: boolean; }
```

## 2. Store interface (swappable per ADR-008)

```ts
interface DeviceStore {
  // users
  listUsers(): User[]; getUserByUid(uid): User | null; getUserByUserId(id): User | null;
  upsertUser(u: User): void; deleteUser(uid): void; clearUsers(): void;
  // attendance
  listAttendance(): AttendanceRecord[]; appendAttendance(r): AttendanceRecord;
  clearAttendance(): void; clearAll(): void;
  // templates
  listTemplates(uid): FpTemplate[]; getTemplate(uid, finger): FpTemplate | null;
  setTemplate(t: FpTemplate): void; deleteTemplate(uid, finger): void;
  // options / lcd / state
  getOptions(): DeviceOptions; setOption(k, v): void;
  getLcd(): LcdState; setLcd(s: LcdState): void;
  getState(): DeviceState; setState(s: Partial<DeviceState>): void;
  onChange(cb: (evt: StoreEvent) => void): () => void;   // feeds the event bus
}
```

## 3. Wire ↔ model mapping highlights

- `uid` vs `userId`: the protocol uses both (uid = internal index, userId = visible PIN); the BITS backend treats `userId` as the employee's `zkId`. Seed data keeps them numerically aligned but distinct in type.
- Timestamps: stored as real `Date`; encoded/decoded only at the wire boundary (single codec module prevents drift — the 372-day calendar is easy to get subtly wrong).
- Attendance state: node-zklib 1.3.0 does not decode the state byte (backend defaults to 0) — the simulator still emits correct states for future clients.

## 4. Seed data (FR-13)

- ~8 named employees (uid 1–8, userId matching), one admin (role 14).
- A week of attendance (Mon–Sat): on-time check-ins, some late, one absent, check-outs in the evening — timestamps generated through the codec so device reads and UI agree.
- Fingerprints: finger 0 (right index) enrolled for most employees with 512-B synthetic `SS21…` blobs; one employee left empty to exercise the "no template" probe path.
- ~300 attendance records total — big enough to cross a couple of TCP segments, far below any performance concern.

## 5. SQLite schema (Phase 5, optional)

```sql
CREATE TABLE users        (uid INTEGER PRIMARY KEY, user_id TEXT NOT NULL, name TEXT, privilege INTEGER, password TEXT, cardno INTEGER);
CREATE TABLE attendance   (seq INTEGER PRIMARY KEY AUTOINCREMENT, user_sn INTEGER, device_user_id TEXT, ts INTEGER, verify_type INTEGER, state INTEGER);
CREATE TABLE templates    (uid INTEGER, finger INTEGER, flag INTEGER, blob BLOB, PRIMARY KEY (uid, finger));
CREATE TABLE options      (key TEXT PRIMARY KEY, value TEXT);
```
Same `DeviceStore` interface; file path via config; disabled by default.
