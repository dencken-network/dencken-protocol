# P-02 — LEDGER

**Layer:** Infrastructure
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/core/ledger.js`

\---

## Purpose

The ledger is the network's irrevocable memory. Every governance action, deliberation cycle, board decision, and system event is recorded here. Nothing is ever deleted or modified. The ledger's history is the network's identity over time.

The ledger sub-plane is one of three constitutional sub-planes. It differs from the knowledge store in one critical way: **ledger records are irrevocable**. The knowledge store can retract records via a signed notice — which itself enters the ledger. The ledger itself never retracts anything.

\---

## Storage

**SQLite** via `better-sqlite3` (synchronous, no async complexity).

**Fallback:** append-only JSONL file (`ledger.jsonl`) if SQLite unavailable.

**File location:** `data/ledger.db` — never committed to git. Added to `.gitignore`.

\---

## Schema

```sql
CREATE TABLE IF NOT EXISTS ledger (
  id TEXT PRIMARY KEY,
  created\\\_at TEXT NOT NULL,
  record\\\_type TEXT NOT NULL,
  brief\\\_version TEXT NOT NULL,
  field TEXT NOT NULL DEFAULT 'operational',
  audience TEXT NOT NULL DEFAULT 'internal',
  graph\\\_type TEXT NOT NULL DEFAULT 'task\\\_state',
  content\\\_hash TEXT NOT NULL,
  content\\\_encrypted TEXT NOT NULL,
  author\\\_pubkey TEXT NOT NULL,
  signature TEXT NOT NULL,
  prev\\\_hash TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending\\\_review',
  board\\\_note TEXT
);

CREATE INDEX IF NOT EXISTS idx\\\_record\\\_type ON ledger(record\\\_type);
CREATE INDEX IF NOT EXISTS idx\\\_status ON ledger(status);
CREATE INDEX IF NOT EXISTS idx\\\_field ON ledger(field);
CREATE INDEX IF NOT EXISTS idx\\\_created\\\_at ON ledger(created\\\_at);
```

\---

## Append procedure

```
1. Assemble record object (all fields)
2. SIGN record (P-01)
3. INSERT into ledger table
4. Append hash-only entry to ledger-public/log.jsonl
5. If INSERT fails: write to JSONL fallback, alert PULSE
```

The ledger will never UPDATE or DELETE. Any code path that attempts to modify an existing ledger record is a bug.

\---

## Public hash log

After every append, one line is written to `ledger-public/log.jsonl`:

```json
{"id":"uuid","created\\\_at":"ISO8601","record\\\_type":"cycle","content\\\_hash":"sha256-hex","prev\\\_hash":"sha256-hex","status":"pending\\\_review"}
```

This file contains **no content** — only hashes. It is safe to commit to `dencken-core` and serves as a public proof-of-activity trail. External parties can verify that records exist at specific times without seeing their content.

\---

## Encryption

Record content is encrypted with AES-256-GCM before storage:

```
key = HKDF(MASTER\\\_KEY, salt="ledger", length=32)
iv  = random 12 bytes
encrypted = AES-256-GCM.encrypt(plaintext, key, iv)
stored    = base64(iv + ciphertext + auth\\\_tag)
```

The `content\\\_hash` is computed from the **plaintext** before encryption. This allows integrity verification without decryption: if the hash matches a known value, the content is authentic.

\---

## Chain integrity

Each record contains `prev\\\_hash` = SHA-256 of the previous record's `content\\\_hash`. This forms a hash chain. Any tampering with a record invalidates all subsequent records.

**Verification:**

```
1. Read all records in insertion order
2. For each record N: verify prev\\\_hash(N) = content\\\_hash(N-1)
3. Verify signature on each record (P-01)
4. Any break in the chain = ledger has been tampered with
```

\---

## Reset prohibition

There is no reset, truncate, or delete path in the ledger module. A one-time setup script (`setup/init-ledger.js`) may create an empty database if `data/ledger.db` does not exist. It exits immediately if the database exists and contains any records.

Any PR that adds delete or truncate functionality to `ledger.js` will be rejected.

\---

## Record types

|record\_type|Description|
|-|-|
|`system`|Node bootstrap, setup complete, configuration changes|
|`cycle`|Deliberation cycle output|
|`pulse`|PULSE health check|
|`board\\\_action`|Promote, discard, board note|
|`knowledge`|Knowledge store entry reference|
|`retraction`|Signed retraction notice for a knowledge record|
|`constitution\\\_change`|Variable rule amendment|
|`agent\\\_snapshot`|Agent capability snapshot|

\---

*Dencken Network — P-02 LEDGER — dencken.net — August 2026*
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.