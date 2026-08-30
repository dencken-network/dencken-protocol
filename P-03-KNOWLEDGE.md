# P-03 — KNOWLEDGE

**Layer:** Infrastructure
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/core/knowledge.js`

\---

## Purpose

The knowledge store is the network's active living memory. Unlike the ledger (irrevocable), knowledge records can be retracted by their author via a signed retraction notice — which itself enters the ledger permanently.

The knowledge store holds what the network has learned: cycle outputs, insights, agent snapshots, manifest versions, promoted deliberation results. It grows over time and shapes inference by providing context to agents at cycle time.

\---

## Storage model

**Two-part structure:**

1. **Content store:** encrypted files on disk, named by content hash

   * Path: `knowledge/{sha256}.enc`
   * Format: AES-256-GCM encrypted JSON
   * Never modified after creation
2. **Metadata index:** SQLite table for fast querying

   * Path: `data/knowledge.db`
   * Contains: hash, type, field, audience, graph\_type, status, timestamps
   * Never contains decrypted content

\---

## Schema (metadata index)

```sql
CREATE TABLE IF NOT EXISTS knowledge\\\_index (
  hash TEXT PRIMARY KEY,
  created\\\_at TEXT NOT NULL,
  record\\\_type TEXT NOT NULL,
  field TEXT NOT NULL DEFAULT 'operational',
  audience TEXT NOT NULL DEFAULT 'internal',
  graph\\\_type TEXT NOT NULL DEFAULT 'knowledge',
  author\\\_pubkey TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending\\\_review',
  retracted INTEGER NOT NULL DEFAULT 0,
  retraction\\\_ledger\\\_id TEXT,
  ledger\\\_ref TEXT,
  size\\\_bytes INTEGER,
  tags TEXT
);

CREATE INDEX IF NOT EXISTS idx\\\_k\\\_field ON knowledge\\\_index(field);
CREATE INDEX IF NOT EXISTS idx\\\_k\\\_audience ON knowledge\\\_index(audience);
CREATE INDEX IF NOT EXISTS idx\\\_k\\\_graph ON knowledge\\\_index(graph\\\_type);
CREATE INDEX IF NOT EXISTS idx\\\_k\\\_status ON knowledge\\\_index(status);
CREATE INDEX IF NOT EXISTS idx\\\_k\\\_retracted ON knowledge\\\_index(retracted);
```

\---

## Store procedure

```
1. Compute content\\\_hash = SHA-256(plaintext JSON)
2. Encrypt content (AES-256-GCM)
3. Write encrypted file to knowledge/{content\\\_hash}.enc
4. Insert metadata row into knowledge\\\_index
5. Write ledger reference record (record\\\_type: knowledge)
6. Return content\\\_hash as the record's permanent address
```

\---

## Fetch procedure

```
1. Look up hash in knowledge\\\_index
2. Check audience tag against requesting context
3. Check retracted = 0
4. Read knowledge/{hash}.enc from disk
5. Decrypt and return
```

Permission error — not "not found" — for audience mismatches.

\---

## Retraction procedure

```
1. Author signs retraction notice: { hash, reason, retracted\\\_at }
2. Retraction notice appended to ledger (record\\\_type: retraction)
3. knowledge\\\_index: retracted=1, retraction\\\_ledger\\\_id set
4. Encrypted file remains on disk — never deleted
5. Queries surface retraction status alongside metadata
```

\---

## Four graph types

|graph\_type|Description|Query|
|-|-|-|
|`policy`|Constitutional sub-plane|Loaded at boot, memory cached|
|`knowledge`|Stable accumulated truth|Field-scoped SQL at inference|
|`task\\\_state`|Execution context|Recent records only|
|`episodic`|Session working memory|RAM only — never written here|

\---

## What the knowledge store does NOT contain

* The constitution (Constitutional sub-plane, encrypted flat file)
* Ledger records (Ledger sub-plane, data/ledger.db)
* Session working memory (RAM only)
* API keys or private keys (Private bucket only)

\---

*Dencken Network — P-03 KNOWLEDGE — dencken.net — August 2026*
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.