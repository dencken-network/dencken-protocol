# Dencken Network — Protocol Index
**Repository:** dencken-network/dencken-protocol
**Version:** v0.1 — August 2026

> "Built by what it builds."

Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.

---

## Protocol Index

| # | Protocol | Layer | Status |
|---|---|---|---|
| P-01 | SIGN — Record Signing | Infrastructure | Draft |
| P-02 | LEDGER — Append-Only State | Infrastructure | Draft |
| P-03 | KNOWLEDGE — Content-Addressed Store | Infrastructure | Draft |
| P-04 | PULSE — Health Heartbeat | Operational | Draft |
| P-05 | CYCLE — Deliberation Cycle | Constitutional | Draft |
| P-06 | CLASSIFY — Knowledge Routing | Constitutional | Draft |
| P-07 | SYNC — Node Ledger Sync | Infrastructure | Stub |
| P-08 | CAST — Signed Broadcast | Infrastructure | Stub |
| P-09 | FETCH — Content Retrieval | Infrastructure | Stub |
| P-10 | DISPATCH — ACL Command Flow | Constitutional | Stub |
| P-11 | VOTE — Governance Voting | Constitutional | Stub |
| P-12 | REFLECT — Evolution System | Constitutional | Stub |

---

## Architecture documents

- ARCHITECTURE — Three-plane overview
- KNOWLEDGE-GRAPH — Four graph types and classification
- NODE-TYPES — Server, device, embed nodes
- NETWORK-STRUCTURE — Three network tracks

---

*Dencken Network — dencken.net — CP Müller / Oddsized — August 2026*

---

# P-01 — SIGN
**Layer:** Infrastructure
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/core/crypto.js`, `src/core/records.js`

---

## Purpose

Every record produced by a Dencken node must be signed before it leaves the Node-Local bucket. SIGN is the foundational integrity guarantee: it makes every record attributable to a specific node and tamper-evident.

Without SIGN, the ledger is a log. With SIGN, the ledger is a constitutional record.

---

## Algorithm

**Ed25519** — Edwards-curve Digital Signature Algorithm over Curve25519.

Chosen because:
- Fast verification (critical at retrieval time)
- Small signature size (64 bytes)
- No parameter choices that can be misconfigured
- Native in Node.js `crypto` module (no external dependency)

---

## Record structure

Every signed record contains exactly these fields:

```json
{
  "id": "uuid-v4",
  "created_at": "ISO8601 UTC",
  "record_type": "cycle | pulse | board_action | system | knowledge | retraction",
  "brief_version": "1.0.0",
  "field": "constitution | governance | operational | learning | self_reflection | application | enterprise | mesh | avatar | spells",
  "audience": "node-private | internal | board-only | public",
  "graph_type": "policy | knowledge | task_state | episodic",
  "content_hash": "sha256-hex",
  "content_encrypted": "base64-aes-256-gcm",
  "author_pubkey": "hex",
  "signature": "base64-ed25519",
  "prev_hash": "sha256-hex",
  "status": "pending_review | promoted | discarded",
  "board_note": "string | null"
}
```

---

## Signing procedure

```
1. Assemble plaintext content (JSON string, deterministic key order)
2. Compute content_hash = SHA-256(plaintext)
3. Encrypt plaintext → content_encrypted (AES-256-GCM, see P-02)
4. Compute prev_hash = SHA-256(previous ledger entry's content_hash)
   — Genesis record: prev_hash = SHA-256("GENESIS")
5. Assemble signing payload:
   payload = content_hash + "|" + prev_hash + "|" + created_at + "|" + record_type
6. signature = Ed25519.sign(payload, node_private_key)
7. Record is complete. Move from Pending bucket to Ledger.
```

---

## Verification procedure

```
1. Recompute payload from record fields
2. Ed25519.verify(payload, signature, author_pubkey)
3. Verify content_hash matches SHA-256 of decrypted content
4. Verify prev_hash matches previous ledger entry
5. All three must pass. Any failure = record invalid.
```

---

## Key management

- **Private key:** Ed25519, generated on first boot, stored in `.env` as `NODE_PRIVATE_KEY` (PEM). Never written to disk in plaintext. Never logged.
- **Public key:** stored in `config/node-identity.pub`. Safe to commit to `dencken-core`. Used by peers and external verifiers.
- **Key recovery:** Shamir Secret Sharing, 3-of-5 shares. Shares distributed to trusted witnesses. See P-07 (SYNC) for multi-node key trust.

---

## What SIGN does NOT cover

- Encryption of content (see P-02 LEDGER for AES-256-GCM)
- Routing decisions (see P-06 CLASSIFY)
- Audience enforcement (see P-09 FETCH)

---

## Implementation notes

```javascript
// src/core/crypto.js — sign a payload
import { createSign } from 'crypto';

export function signPayload(payload, privateKeyPem) {
  const sign = createSign('Ed25519');
  sign.update(payload);
  return sign.sign(privateKeyPem, 'base64');
}

export function verifySignature(payload, signatureB64, publicKeyPem) {
  const verify = createVerify('Ed25519');
  verify.update(payload);
  return verify.verify(publicKeyPem, signatureB64, 'base64');
}
```

---

*Dencken Network — P-01 SIGN — dencken.net — August 2026*
# P-02 — LEDGER
**Layer:** Infrastructure
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/core/ledger.js`

---

## Purpose

The ledger is the network's irrevocable memory. Every governance action, deliberation cycle, board decision, and system event is recorded here. Nothing is ever deleted or modified. The ledger's history is the network's identity over time.

The ledger sub-plane is one of three constitutional sub-planes. It differs from the knowledge store in one critical way: **ledger records are irrevocable**. The knowledge store can retract records via a signed notice — which itself enters the ledger. The ledger itself never retracts anything.

---

## Storage

**SQLite** via `better-sqlite3` (synchronous, no async complexity).

**Fallback:** append-only JSONL file (`ledger.jsonl`) if SQLite unavailable.

**File location:** `data/ledger.db` — never committed to git. Added to `.gitignore`.

---

## Schema

```sql
CREATE TABLE IF NOT EXISTS ledger (
  id TEXT PRIMARY KEY,
  created_at TEXT NOT NULL,
  record_type TEXT NOT NULL,
  brief_version TEXT NOT NULL,
  field TEXT NOT NULL DEFAULT 'operational',
  audience TEXT NOT NULL DEFAULT 'internal',
  graph_type TEXT NOT NULL DEFAULT 'task_state',
  content_hash TEXT NOT NULL,
  content_encrypted TEXT NOT NULL,
  author_pubkey TEXT NOT NULL,
  signature TEXT NOT NULL,
  prev_hash TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending_review',
  board_note TEXT
);

CREATE INDEX IF NOT EXISTS idx_record_type ON ledger(record_type);
CREATE INDEX IF NOT EXISTS idx_status ON ledger(status);
CREATE INDEX IF NOT EXISTS idx_field ON ledger(field);
CREATE INDEX IF NOT EXISTS idx_created_at ON ledger(created_at);
```

---

## Append procedure

```
1. Assemble record object (all fields)
2. SIGN record (P-01)
3. INSERT into ledger table
4. Append hash-only entry to ledger-public/log.jsonl
5. If INSERT fails: write to JSONL fallback, alert PULSE
```

The ledger will never UPDATE or DELETE. Any code path that attempts to modify an existing ledger record is a bug.

---

## Public hash log

After every append, one line is written to `ledger-public/log.jsonl`:

```json
{"id":"uuid","created_at":"ISO8601","record_type":"cycle","content_hash":"sha256-hex","prev_hash":"sha256-hex","status":"pending_review"}
```

This file contains **no content** — only hashes. It is safe to commit to `dencken-core` and serves as a public proof-of-activity trail. External parties can verify that records exist at specific times without seeing their content.

---

## Encryption

Record content is encrypted with AES-256-GCM before storage:

```
key = HKDF(MASTER_KEY, salt="ledger", length=32)
iv  = random 12 bytes
encrypted = AES-256-GCM.encrypt(plaintext, key, iv)
stored    = base64(iv + ciphertext + auth_tag)
```

The `content_hash` is computed from the **plaintext** before encryption. This allows integrity verification without decryption: if the hash matches a known value, the content is authentic.

---

## Chain integrity

Each record contains `prev_hash` = SHA-256 of the previous record's `content_hash`. This forms a hash chain. Any tampering with a record invalidates all subsequent records.

**Verification:**

```
1. Read all records in insertion order
2. For each record N: verify prev_hash(N) = content_hash(N-1)
3. Verify signature on each record (P-01)
4. Any break in the chain = ledger has been tampered with
```

---

## Reset prohibition

There is no reset, truncate, or delete path in the ledger module. A one-time setup script (`setup/init-ledger.js`) may create an empty database if `data/ledger.db` does not exist. It exits immediately if the database exists and contains any records.

Any PR that adds delete or truncate functionality to `ledger.js` will be rejected.

---

## Record types

| record_type | Description |
|---|---|
| `system` | Node bootstrap, setup complete, configuration changes |
| `cycle` | Deliberation cycle output |
| `pulse` | PULSE health check |
| `board_action` | Promote, discard, board note |
| `knowledge` | Knowledge store entry reference |
| `retraction` | Signed retraction notice for a knowledge record |
| `constitution_change` | Variable rule amendment |
| `agent_snapshot` | Agent capability snapshot |

---

*Dencken Network — P-02 LEDGER — dencken.net — August 2026*
# P-03 — KNOWLEDGE
**Layer:** Infrastructure
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/core/knowledge.js`

---

## Purpose

The knowledge store is the network's active living memory. Unlike the ledger (irrevocable), knowledge records can be retracted by their author via a signed retraction notice — which itself enters the ledger permanently.

The knowledge store holds what the network has learned: cycle outputs, insights, agent snapshots, manifest versions, promoted deliberation results. It grows over time and shapes inference by providing context to agents at cycle time.

---

## Storage model

**Two-part structure:**

1. **Content store:** encrypted files on disk, named by content hash
   - Path: `knowledge/{sha256}.enc`
   - Format: AES-256-GCM encrypted JSON
   - Never modified after creation

2. **Metadata index:** SQLite table for fast querying
   - Path: `data/knowledge.db`
   - Contains: hash, type, field, audience, graph_type, status, timestamps
   - Never contains decrypted content

---

## Schema (metadata index)

```sql
CREATE TABLE IF NOT EXISTS knowledge_index (
  hash TEXT PRIMARY KEY,
  created_at TEXT NOT NULL,
  record_type TEXT NOT NULL,
  field TEXT NOT NULL DEFAULT 'operational',
  audience TEXT NOT NULL DEFAULT 'internal',
  graph_type TEXT NOT NULL DEFAULT 'knowledge',
  author_pubkey TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending_review',
  retracted INTEGER NOT NULL DEFAULT 0,
  retraction_ledger_id TEXT,
  ledger_ref TEXT,
  size_bytes INTEGER,
  tags TEXT
);

CREATE INDEX IF NOT EXISTS idx_k_field ON knowledge_index(field);
CREATE INDEX IF NOT EXISTS idx_k_audience ON knowledge_index(audience);
CREATE INDEX IF NOT EXISTS idx_k_graph ON knowledge_index(graph_type);
CREATE INDEX IF NOT EXISTS idx_k_status ON knowledge_index(status);
CREATE INDEX IF NOT EXISTS idx_k_retracted ON knowledge_index(retracted);
```

---

## Store procedure

```
1. Compute content_hash = SHA-256(plaintext JSON)
2. Encrypt content (AES-256-GCM)
3. Write encrypted file to knowledge/{content_hash}.enc
4. Insert metadata row into knowledge_index
5. Write ledger reference record (record_type: knowledge)
6. Return content_hash as the record's permanent address
```

---

## Fetch procedure

```
1. Look up hash in knowledge_index
2. Check audience tag against requesting context
3. Check retracted = 0
4. Read knowledge/{hash}.enc from disk
5. Decrypt and return
```

Permission error — not "not found" — for audience mismatches.

---

## Retraction procedure

```
1. Author signs retraction notice: { hash, reason, retracted_at }
2. Retraction notice appended to ledger (record_type: retraction)
3. knowledge_index: retracted=1, retraction_ledger_id set
4. Encrypted file remains on disk — never deleted
5. Queries surface retraction status alongside metadata
```

---

## Four graph types

| graph_type | Description | Query |
|---|---|---|
| `policy` | Constitutional sub-plane | Loaded at boot, memory cached |
| `knowledge` | Stable accumulated truth | Field-scoped SQL at inference |
| `task_state` | Execution context | Recent records only |
| `episodic` | Session working memory | RAM only — never written here |

---

## What the knowledge store does NOT contain

- The constitution (Constitutional sub-plane, encrypted flat file)
- Ledger records (Ledger sub-plane, data/ledger.db)
- Session working memory (RAM only)
- API keys or private keys (Private bucket only)

---

*Dencken Network — P-03 KNOWLEDGE — dencken.net — August 2026*
# P-04 — PULSE
**Layer:** Operational
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/layers/pulse.js`

---

## Purpose

PULSE is the network's always-on health heartbeat. It runs independently of the deliberation cycle on a fixed interval, monitors the node's operational state, and writes a signed health record to the ledger after every check.

PULSE is the first OFFICERS layer active in the prototype. It is the diagnostic foundation all other layers depend on. Without PULSE, the network cannot detect degradation, provider failure, ledger drift, or task queue stagnation.

---

## Trigger

- **Interval:** `PULSE_INTERVAL_MS` from `.env` (default: 30000ms / 30 seconds)
- **Runs independently:** PULSE does not block or wait for the deliberation cycle
- **Always-on:** starts at boot, runs until process exits

---

## What PULSE checks

Each PULSE run collects the following:

| Check | Source | Failure condition |
|---|---|---|
| Ledger height | `ledger.js — getHeight()` | Cannot read ledger |
| Last cycle timestamp | `ledger.js — getLastByType('cycle')` | No cycle in >24h |
| Task queue depth | `taskqueue.js — list()` | Queue depth > 50 |
| Agent API status | lightweight probe per active agent | HTTP error or timeout |
| Constitution loaded | `constitution.js — isLoaded()` | Not loaded |
| Node identity | `identity.js — isReady()` | Keypair missing |

---

## PULSE record

Every PULSE run writes one record to the ledger:

```json
{
  "record_type": "pulse",
  "content": {
    "node_id": "dencken-server-node-0",
    "ledger_height": 47,
    "last_cycle_at": "2026-08-15T06:00:00Z",
    "task_queue_depth": 3,
    "constitution_loaded": true,
    "identity_ready": true,
    "agent_statuses": {
      "agent-alpha": "ok",
      "agent-beta": "ok",
      "agent-gamma": "degraded",
      "agent-delta": "unknown"
    },
    "overall": "degraded"
  },
  "audience": "internal",
  "graph_type": "task_state",
  "status": "promoted"
}
```

PULSE records are always `status: promoted` — they skip board review. They are operational telemetry, not deliberation output.

---

## Overall status logic

```
overall = "ok"       if all checks pass
overall = "degraded" if any agent is degraded or unreachable,
                     or last cycle > 24h ago,
                     or task queue depth > 50
overall = "critical" if ledger unreadable,
                     or constitution not loaded,
                     or identity not ready
```

On `critical`: PULSE logs to console with timestamp and continues. It does not halt the process. Critical state is visible in the board interface via the ledger.

---

## Agent health probe

A lightweight probe — not a full API call:

```javascript
// For OpenRouter / Groq (OpenAI-compatible):
// HEAD or GET to the provider's base URL with a 3s timeout
// If response < 400: ok
// If timeout or error: degraded
// If never probed this session: unknown

async function probeAgent(agent) {
  const endpoints = {
    openrouter: 'https://openrouter.ai/api/v1/models',
    groq: 'https://api.groq.com/openai/v1/models'
  };
  try {
    const res = await fetch(endpoints[agent.provider], {
      method: 'GET',
      signal: AbortSignal.timeout(3000),
      headers: { Authorization: `Bearer ${getApiKey(agent.provider)}` }
    });
    return res.ok ? 'ok' : 'degraded';
  } catch {
    return 'degraded';
  }
}
```

---

## Integration in app.js

```javascript
import { runPulse } from './src/layers/pulse.js';
import { simulateDeliberationCycle } from './src/agents/cycle.js';
import cron from 'node-cron';

// PULSE — independent interval
setInterval(async () => {
  await runPulse();
}, parseInt(process.env.PULSE_INTERVAL_MS) || 30000);

// Deliberation cycle — cron schedule
cron.schedule(process.env.CYCLE_SCHEDULE || '0 */6 * * *', async () => {
  await simulateDeliberationCycle();
});
```

---

## Future extensions (do not implement yet)

- **Network PULSE:** when multiple nodes exist, each node broadcasts its PULSE record to peers via CAST (P-08). Other nodes maintain a peer health map.
- **SHIELD trigger:** PULSE `critical` status triggers SHIELD layer response (P-10 DISPATCH).
- **Drift detection:** PULSE compares ledger height against expected growth rate and flags stagnation to ETHER.

---

## Folder stub

```
src/layers/
  pulse.js        ← implement now
  ether.js        ← stub only, Phase 2
  shield.js       ← stub only, Phase 2
  mission.js      ← stub only, Phase 2
  task.js         ← stub only, Phase 2
  field.js        ← stub only, Phase 2
```

---

*Dencken Network — P-04 PULSE — dencken.net — August 2026*
# P-05 — CYCLE
**Layer:** Constitutional
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/agents/cycle.js`

---

## Purpose

CYCLE is the deliberation engine — the core operational loop of the Dencken Network. It takes a topic from the task queue, assembles a constitutional context, invokes a sequence of AI agents, records the deliberation in the ledger, and produces a board_review object for human oversight.

Every meaningful thing the network produces comes from a CYCLE run. It is the mechanism through which the constitution becomes action, and through which the network accumulates knowledge over time.

---

## Trigger

- **Scheduled:** via cron (`CYCLE_SCHEDULE` in `.env`, default `0 */6 * * *`)
- **Manual:** `POST /cycle/run` from board interface
- **Triggered:** by board promotion with `follow_up_task` (via taskqueue.js)

---

## Cycle structure

```
CYCLE RUN

1. LOAD
   - Load constitutional brief from constitution.js
     (decrypted at runtime, never logged)
   - Load next task from taskqueue.js
     (falls back to config/tasks.json if queue empty)
   - Select active agents from pool.js
     (minimum 2: one initiator, at least one respondent)

2. ASSEMBLE CONTEXT
   - system_prompt = system_base + constitutional_dispositions
                   + role_prompt (varies by agent role)
   - user_message  = [recent promoted knowledge if available]
                   + "---" + cycle topic
   - Knowledge context: last 5 promoted records, field-matched
     to task.field (see P-03 KNOWLEDGE, P-06 CLASSIFY)

3. EXECUTE AGENTS (sequential)
   a. Initiator agent:
      receives: system_prompt + user_message (topic only)
      produces: initiator_proposal record
   
   b. Respondent agent(s) — queue/branch:
      receives: system_prompt + topic + previous agent output
      produces: respondent_response record per agent
      supports: multiple respondents via agent queue
   
   c. Synthesis agent (optional):
      receives: system_prompt + all prior outputs
      produces: synthesis record
      if no synthesis agent active: node produces hash summary

4. RECORD
   Each agent output written to ledger immediately:
     record_type: cycle
     content: { role, agent_id, provider, model, output }
     audience: internal
     graph_type: episodic  ← upgraded to knowledge on promotion
     status: pending_review

5. BUILD BOARD REVIEW OBJECT
   board_review = {
     status: 'review_required',
     cycle_id: uuid,
     task: { id, topic, field },
     branch_states: [{ agent_id, role, record_id }],
     task_queue: [{ id, suggested_topic, source: 'synthesis' }],
     ledger_refs: [record_ids in this cycle],
     created_at: ISO8601
   }

6. NOTIFY
   If BOARD_NOTIFY=true: log pending count to console
   Future: push notification to board interface via SSE
```

---

## Record types produced by one cycle

| record_type | Count | Description |
|---|---|---|
| `initiator_proposal` | 1 | First agent's opening position |
| `respondent_response` | 1–N | Each respondent agent's reply |
| `synthesis` | 0–1 | Final synthesis if synthesis agent active |
| `board_action` | 0–1 | Written when board promotes or discards |

---

## Agent role selection

Agents are selected from the active pool by their `role` field in `agents.json`:

```json
{
  "agents": [
    { "id": "agent-alpha", "role": "initiator",  "active": true },
    { "id": "agent-beta",  "role": "respondent", "active": true },
    { "id": "agent-gamma", "role": "respondent", "active": true },
    { "id": "agent-delta", "role": "synthesis",  "active": false }
  ]
}
```

- Exactly one initiator is selected per cycle
- All active respondents run in queue order
- Synthesis agent is optional — cycle completes without one
- Pool is unlimited — add agents by editing `agents.json`

---

## Constitutional brief assembly

The brief is assembled at runtime from the decrypted constitution. The source text is never transmitted to any agent as quotable text:

```javascript
// constitution.js — assembles agent-injectable brief
function buildBrief(constitution, role) {
  return {
    system_base: constitution.prompts.system_base,
    // Rules transformed into behavioural dispositions
    // Source rule text never included in output
    dispositions: transformRules(constitution.non_variant_rules,
                                 constitution.variant_rules),
    role_prompt: selectPrompt(constitution.prompts[role]),
    brief_version: constitution.version
  };
}
```

---

## Cycle verification (prototype)

```javascript
// Run in Node.js REPL to verify:
const { simulateDeliberationCycle } = require('./src/agents/cycle');
const result = await simulateDeliberationCycle({
  prompt: 'Branch test',
  max_messages: 2,
  use_manifest: false
});
console.log({
  total: result.total_conversation_messages,
  types: result.conversation.map(m => m.record_type),
  boardStatus: result.board_review.status,
  taskCount: result.board_review.task_queue.length
});

// Expected output:
// { total: 4,
//   types: ['initiator_proposal','respondent_response',
//           'respondent_response','synthesis'],
//   boardStatus: 'review_required',
//   taskCount: 1 }
```

---

## Chat explorer vs cycle

| | Deliberation Cycle | Chat Explorer |
|---|---|---|
| Trigger | Scheduled / board | Human message |
| Agents | 2–N from pool | 1 agent |
| Ledger audience | internal | node-private |
| Board review | Required | Not required |
| Task queue | Feeds from promotions | Does not feed automatically |
| Knowledge context | Last 5 promoted (field-matched) | Last 5 promoted (topic-matched) |
| Status on write | pending_review | promoted |

---

## Future extensions

- **Multi-agent branching:** parallel respondents rather than sequential queue (Phase 2)
- **Mirror cycles:** VITA/IMAGO/ANIMA/RES run REFLECT protocol (P-12) using cycle output as input (Phase 2)
- **Cross-node cycles:** MISSION layer coordinates cycles across multiple nodes (Phase 3)
- **Cycle budget:** maximum steps per MISSION-triggered cycle (default 1,000 dispatches)

---

*Dencken Network — P-05 CYCLE — dencken.net — August 2026*
# P-06 — CLASSIFY
**Layer:** Constitutional
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/core/classifier.js` (create in Phase 1)
**Used by:** `src/agents/cycle.js`, `src/layers/pulse.js`, `src/core/records.js`

---

## Purpose

CLASSIFY is the knowledge routing layer. Every record produced by the network passes through the Classifier before being written to permanent storage. The Classifier decides: which sub-plane does this record belong to, which architecture field does it relate to, who is allowed to read it, and which graph type governs its retrieval pattern.

Classification happens at creation time — not at query time. Tags are written once, with the record, and never changed. This keeps retrieval fast and non-blocking.

---

## Classification is not a bottleneck

The Classifier uses three decision steps in sequence, each fast:

```
1. Primary route   — from record_type (enum lookup, < 1ms)
2. Field tag       — from role affinity + keyword signals (regex, < 5ms)
3. Audience tag    — from cycle config or board action (explicit, < 1ms)

Total classification time target: < 10ms
If classification fails: default to operational / internal / knowledge
Log the failure, never block the write.
```

---

## Step 1 — Primary route (sub-plane)

Determined entirely by `record_type`. No inference required.

| record_type | Sub-plane | Storage |
|---|---|---|
| `constitution_change` `rule_update` `authority_change` | Constitutional | Encrypted flat file |
| `cycle` `pulse` `board_action` `retraction` `system` | Ledger | Append-only SQLite |
| `cycle_output` `insight` `artifact` `agent_snapshot` `manifest_version` `promotion` | Knowledge | Content-addressed files |
| `working_memory` `cycle_context` `session_state` | Episodic | RAM only — never written |

---

## Step 2 — Field tag

Ten possible values corresponding to the ten architecture fields.

**Primary signal: agent role affinity**

| Agent role | Default field |
|---|---|
| CEO | operational |
| CFO | enterprise |
| CTO | operational |
| CMO | application |
| PRO | governance |
| CSO | self_reflection |
| PULSE layer | operational |
| VITA mirror | constitution |
| IMAGO mirror | self_reflection |
| ANIMA mirror | self_reflection |
| RES mirror | governance |
| ENTERPRISE mirror | enterprise |
| MESH mirror | mesh |
| Chat explorer | self_reflection |

**Secondary signal: keyword patterns (regex)**

```javascript
const FIELD_PATTERNS = {
  constitution:    /\b(rule|principle|founding|immutable|constitutional|mandate)\b/i,
  governance:      /\b(vote|decision|board|approve|reject|policy|authority)\b/i,
  enterprise:      /\b(revenue|cost|contract|client|invoice|commercial|profit)\b/i,
  learning:        /\b(insight|pattern|learned|knowledge|discovery|finding)\b/i,
  self_reflection: /\b(reflect|assess|character|identity|mirror|cycle|deliberat)\b/i,
  application:     /\b(feature|interface|user|product|publish|content|brand)\b/i,
  mesh:            /\b(external|api|integration|third.party|webhook|federation)\b/i,
  avatar:          /\b(presence|identity|public|persona|reputation|represent)\b/i,
  spells:          /\b(capability|emerge|unique|network.magic|cannot.replicate)\b/i,
};
// Default if no pattern matches: 'operational'
```

**Resolution:** role affinity takes precedence. Keyword pattern used only if role is unknown or generic.

---

## Step 3 — Audience tag

Four values. Default: `internal`.

| audience | Readable by | Set by |
|---|---|---|
| `node-private` | Local node processes only | Explicit in cycle config (e.g. chat explorer) |
| `internal` | Node + authorised internal roles | Default for all cycle outputs |
| `board-only` | Board interface only | PRO flag or sensitive cycle |
| `public` | No restriction | Board promotion with explicit public flag |

Audience is set at cycle configuration time or by board action. The Classifier enforces it at read time — a permission error (not "not found") is returned for mismatches.

---

## Graph type tag

Derived from primary route:

| Sub-plane | Graph type | Query pattern |
|---|---|---|
| Constitutional | `policy` | Full load at boot, memory cached |
| Ledger | `task_state` | Sequential, recent records only |
| Knowledge | `knowledge` | Field-scoped SQL query at inference |
| Episodic | `episodic` | RAM only, never queried after cycle |

---

## Classification base model v1

```javascript
// src/core/classifier.js

export function classify(record) {
  const subPlane = routeByRecordType(record.record_type);
  const field    = fieldByRoleAffinity(record.author_role)
                || fieldByKeyword(record.content_preview)
                || 'operational';
  const audience = record.explicit_audience || 'internal';
  const graph    = graphBySubPlane(subPlane);

  return { subPlane, field, audience, graph };
}
```

**Classification model evolution path:**

- v1 (now): role affinity + keyword regex. Fast, deterministic.
- v2 (Phase 2): ANIMA + RES run a classification review cycle on records that defaulted to `operational`. Reclassification proposals go to board.
- v3 (Phase 3): lightweight local discriminator model trained on the network's own classified records. Not a full LLM — a fine-tuned classifier only.
- v4 (future): Classifier becomes a constitutionally-briefed agent role, updatable through the variable constitution amendment process.

Each version is a constitutional evolution. Changes to the Classifier are logged in the ledger as `constitution_change` records.

---

## Field retrieval shortcuts

Pre-built SQL queries by consumer:

```javascript
// Context builder — for ANIMA reflection cycle
const animaContext = await knowledge.query({
  field: ['constitution', 'self_reflection', 'learning'],
  audience: ['internal', 'board-only', 'public'],
  status: 'promoted',
  limit: 10
});

// Context builder — for CFO financial brief
const cfoContext = await knowledge.query({
  field: ['enterprise', 'operational'],
  audience: ['internal', 'board-only'],
  status: 'promoted',
  limit: 5
});

// Context builder — for MESH mirror
const meshContext = await knowledge.query({
  field: ['mesh', 'avatar', 'spells'],
  audience: ['internal', 'public'],
  status: 'promoted',
  limit: 8
});
```

These shortcuts are pre-defined in `src/core/knowledge.js` as named query functions. The Classifier tags enable them. Without classification, every context query would require a full-text scan.

---

*Dencken Network — P-06 CLASSIFY — dencken.net — August 2026*
# P-07 — SYNC
**Layer:** Infrastructure
**Status:** Stub — Phase 3
**Will be implemented in:** `src/protocols/sync.js`

---

## Purpose

SYNC reconciles ledger state between two nodes after a period of disconnection. It uses a Merkle-tree diff to exchange only missing records, and checkpoint snapshots so new nodes do not need to replay full history.

---

## Planned mechanics

- **Merkle-tree diff:** each node computes a Merkle root over its ledger block hashes. Nodes exchange roots and drill down to find divergence points. Only missing leaf records are transmitted.
- **Checkpoint snapshots:** every 1,000 ledger blocks, a signed checkpoint is created (hash of all records to that point, signed by board). New nodes bootstrap from the most recent checkpoint + delta.
- **Constitutional record verification:** constitutional sub-plane records require author pubkey verification against the founding node's public key before acceptance.
- **CRL check:** all incoming records are checked against the local Certificate Revocation List before acceptance.
- **Operational records:** accepted on signature verification only (no CRL required).

---

## Prerequisite protocols

- P-01 SIGN (all records must be signed before sync)
- P-02 LEDGER (append-only target store)
- P-08 CAST (broadcast after sync completes)

---

*Stub — full specification in Phase 3*
*Dencken Network — P-07 SYNC — dencken.net — August 2026*
# P-08 — CAST
**Layer:** Infrastructure
**Status:** Stub — Phase 3
**Will be implemented in:** `src/protocols/cast.js`

---

## Purpose

CAST is the signed broadcast protocol. A node announces a signed record to all connected nodes simultaneously. No delivery confirmation is required — missed broadcasts are caught up via SYNC (P-07) on reconnection.

---

## Planned mechanics

- **TTL-aware:** every CAST record carries a TTL. Expired records move to dead-letter store, never transmitted. TTL by type: governance votes = voting window duration, PULSE heartbeats = 90s, cycle outputs = 30d.
- **Dead-letter store:** expired Pending records are never broadcast. Stored in Node-Local for audit, never transmitted.
- **Fan-out:** one record broadcast to all connected peers via WebRTC DataChannels.
- **No confirmation:** delivery is best-effort. SYNC handles catch-up.

---

## Use cases

- New constitutional proposal broadcast
- PULSE heartbeat (every 30s when multi-node)
- Promoted cycle output (audience: public or network-wide)
- Board action notification

---

*Stub — full specification in Phase 3*
*Dencken Network — P-08 CAST — dencken.net — August 2026*
# P-09 — FETCH
**Layer:** Infrastructure
**Status:** Stub — Phase 3
**Will be implemented in:** `src/protocols/fetch.js`

---

## Purpose

FETCH is the content-hash lookup protocol. A node requests a specific record from the network by its SHA-256 content hash. Any node holding the record can respond. Audience tags are enforced at the serving node.

---

## Planned mechanics

- **Hash-addressed:** every FETCH request specifies a content hash. The requesting node does not need to know which node holds it.
- **Audience enforcement:** serving node checks audience tag before responding. Permission error (not "not found") for mismatches — prevents timing-based content discovery.
- **Privacy:** a node cannot determine whether a record exists if it does not have read permission.
- **DHT-compatible:** designed to work with IPFS-compatible distributed hash tables in Phase 4.

---

## Use cases

- Cycle referencing a past knowledge record by hash
- External audit requesting a specific ledger record
- Agent context builder fetching promoted knowledge from peer node

---

*Stub — full specification in Phase 3*
*Dencken Network — P-09 FETCH — dencken.net — August 2026*
# P-10 — DISPATCH
**Layer:** Constitutional
**Status:** Stub — Phase 2
**Will be implemented in:** `src/core/dispatch.js`

---

## Purpose

DISPATCH is the Application Communication Layer (ACL) command flow. It is the gateway between the constitutional layer and all external systems — IoT devices, APIs, third-party services. Nothing from the external world writes to network memory directly. Nothing from the network executes externally without passing through DISPATCH.

---

## Planned mechanics

Five sequential steps — all must pass or command does not execute:

1. **Governance check:** verify ledger contains a valid pre-authorisation record for this command class at this governance level (Low / Medium / High / Critical)
2. **Source verification:** Ed25519 signature check on the command origin
3. **Rate limit:** token bucket check (1 token / 24h per command type, max 3 accumulated)
4. **Signed dispatch:** command signed by node private key before transmission
5. **Audit record:** ledger entry written for every dispatch regardless of outcome

---

## Governance levels

| Level | Human required | Pre-authorisation |
|---|---|---|
| Low | 0% | Not required |
| Medium | 40% | Required for sustained operation |
| High | 60% | Required + board sign-off |
| Critical | 75% | Required + recorded reasoning |

---

*Stub — full specification in Phase 2*
*Dencken Network — P-10 DISPATCH — dencken.net — August 2026*
# P-11 — VOTE
**Layer:** Constitutional
**Status:** Stub — Phase 3 (Governance Network)
**Will be implemented in:** `src/protocols/vote.js`

---

## Purpose

VOTE is the governance voting protocol. It manages constitutional proposals, vote collection, human floor enforcement, provider concentration checks, and ESCROW for votes cast when guardian quorum is unreachable.

---

## Planned mechanics

- **Human floor:** 0/40/60/75% by governance level. Enforced by Classifier before vote is counted.
- **Provider concentration:** no single AI provider may hold >25% of council votes. Classifier checks provider distribution before counting delegate votes.
- **ESCROW:** votes held when quorum of guardians unreachable. Operational work continues. Released when quorum restored.
- **CRL check:** revoked node public keys rejected before vote counted.
- **Ledger record:** every vote written to ledger with signature, timestamp, governance level, human/agent distinction.

---

## Company Node 0 note

VOTE is not active in Company Node 0 (single board, no council). The protocol is designed for the Governance Network. Board decisions in Node 0 are recorded as `board_action` ledger records via DISPATCH.

---

*Stub — full specification in Phase 3*
*Dencken Network — P-11 VOTE — dencken.net — August 2026*
# P-12 — REFLECT
**Layer:** Constitutional
**Status:** Stub — Phase 2
**Will be implemented in:** `src/protocols/reflect.js`

---

## Purpose

REFLECT is the evolution system protocol. It sequences the six reflection mirrors (VITA, IMAGO, ANIMA, RES, ENTERPRISE, MESH) through a structured reflection cycle using the Conductor/Classifier pattern. Mirrors examine the network's own behaviour across the ten architecture fields and produce findings that feed the deliberation process.

---

## Planned mechanics

- **Conductor sequences stages** using the saga pattern: each mirror runs in defined order, each stage's signed output enters the knowledge store before the next stage begins.
- **Input:** recent ledger history (configurable window), current PULSE state, task queue state.
- **Output per mirror:** a knowledge record tagged with the mirror's primary field, `status: pending_review` (except VITA which auto-promotes health records).
- **Failure handling:** if a mirror's AI provider fails, Conductor retries via failover pool. If all providers fail, partial reflection is recorded and board is notified.
- **Frequency:** triggered by ANIMA recommendation, board request, or after every N deliberation cycles (configurable).

---

## Six mirrors and their fields

| Mirror | Primary field | Output type |
|---|---|---|
| VITA | constitution, governance | Health status, integrity flags |
| IMAGO | self_reflection, avatar | Character assessment, OO + DD scores |
| ANIMA | self_reflection, learning | Meta-insights, evolution proposals |
| RES | governance, operational | Verification status, truth assessments |
| ENTERPRISE | enterprise, mesh | Commercial health, negotiation quality |
| MESH | mesh, avatar, spells | External interface health, dependency risks |

---

*Stub — full specification in Phase 2*
*Dencken Network — P-12-REFLECT — dencken.net — August 2026*
# Dencken Network — Architecture Overview
**Version:** v4.1 — August 2026

---

## Three-plane architecture

Every component belongs to exactly one plane. No cross-plane calls without defined interfaces. This separation is a design constraint, not a guideline.

```
┌─────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE PLANE                                    │
│  Standard solutions only. No custom where standard exists│
│  Transport · Storage · Sync · Crypto · Identity          │
│  Protocols: P-01 SIGN · P-02 LEDGER · P-03 KNOWLEDGE    │
│             P-04 PULSE · P-07 SYNC · P-08 CAST          │
│             P-09 FETCH                                   │
├─────────────────────────────────────────────────────────┤
│  CONSTITUTIONAL PLANE                                    │
│  Dencken-specific custom design                          │
│  Three sub-planes (see below)                            │
│  Protocols: P-05 CYCLE · P-06 CLASSIFY · P-10 DISPATCH  │
│             P-11 VOTE · P-12 REFLECT                     │
├─────────────────────────────────────────────────────────┤
│  OPERATIONAL PLANE                                       │
│  Standard patterns applied to Dencken concerns           │
│  PULSE health · Rate limiting · Audit trail              │
│  Workflow engine (saga) · SDK certification · CRL        │
└─────────────────────────────────────────────────────────┘
```

---

## Constitutional plane — three sub-planes

```
Constitutional sub-plane
  Storage: encrypted flat file (AES-256-GCM)
  Mutability: non-changeable rules immutable
              variable rules via governed process
  Governance: pre-change board action required
  Contents: founding sentence, non-changeable rules,
            variable rules, authority model
  ↓ writes to
Ledger sub-plane
  Storage: append-only SQLite, hash-chained
  Mutability: zero — append only
  Governance: none at write time (happened upstream)
  Contents: every governance action, cycle record,
            board decision, retraction notice
  ← receives retraction notices from
Knowledge sub-plane
  Storage: content-addressed files + SQLite metadata index
  Mutability: append + retract by signed notice
  Governance: post-creation board review
  Contents: cycle outputs, insights, agent snapshots,
            manifest versions, promoted deliberation
```

Data flow is one-directional. The ledger is the terminal record.

---

## Fractal pattern

The same three-plane separation repeats at every scale:

| Scale | Infrastructure | Constitutional | Operational |
|---|---|---|---|
| Record | content hash + IV + auth tag | signature + audience tag | TTL + status |
| Node | transport + storage + crypto | ledger + knowledge + manifest | PULSE + rate limit |
| Network | WebRTC + Merkle sync | governance ledger + replication | network PULSE map |
| Ecosystem | federation transport | lineage records + fork registry | ecosystem health |

If a component cannot be placed in exactly one plane at exactly one scale, it needs to be split.

---

## Node types

| Type | Description | Core modules |
|---|---|---|
| Server node | Always-on, full constitutional layer | All src/core/ + all layers |
| Device node | Client-side, board interface | src/core/ (portable) + browser UI |
| Embed node | Embedded in any application | src/core/ + src/agents/ only |

`src/core/` modules are interface-agnostic and portable to all node types unchanged.

---

## Encryption stack

| Purpose | Algorithm | Key source |
|---|---|---|
| At rest | AES-256-GCM | HKDF from MASTER_KEY |
| Signatures | Ed25519 | NODE_PRIVATE_KEY env |
| Content addressing | SHA-256 | deterministic |
| Key recovery | Shamir SSS 3-of-5 | distributed to witnesses |
| Device auth | mTLS | manufacturer certificate |

---

*Dencken Network — Architecture Overview — dencken.net — August 2026*
# Dencken Network — Knowledge Architecture
**Version:** v4.1 — August 2026
**See also:** P-03 KNOWLEDGE · P-06 CLASSIFY

---

## Four graph types

| Graph | Storage | Query | Update |
|---|---|---|---|
| Policy Graph | Constitutional sub-plane | Full load at boot, memory cached | Pre-change board action |
| Knowledge Graph | Knowledge sub-plane | Field-scoped SQL at inference | Append + retract |
| Task/State Graph | Ledger + operational state db | Recent records, sequential | Overwrite not append |
| Episodic Graph | RAM only during cycle | Never queried after cycle | Promote or discard |

---

## Classification tags

Every knowledge record carries three tags set at creation time:

```
field    — which of ten architecture fields this record relates to
audience — who can read it (node-private / internal / board-only / public)
graph    — which graph type governs retrieval
```

Tags are never changed after creation. Reclassification requires a new record.

---

## Ten architecture fields

**Category A — Constitutional**
- `constitution` — founding principles, immutable rules
- `governance` — structures, decisions, procedures

**Category B — Operational**
- `operational` — day-to-day execution
- `learning` — knowledge accumulation, pattern recognition
- `self_reflection` — deliberation cycles, meta-assessment
- `application` — user-facing functions, interfaces
- `enterprise` — commercial architecture, revenue, legal

**Category C — Real-World Expression**
- `mesh` — external interactions, federation
- `avatar` — network external identity
- `spells` — unique network capabilities

---

## Field → consumer affinity

| Field | Primary agent consumers |
|---|---|
| constitution, governance | ANIMA, RES, board |
| operational | CEO, CTO, PRO, ETHER |
| learning, self_reflection | ANIMA, IMAGO, CSO, VITA |
| application | CMO, CEO, CTO |
| enterprise | CFO, CEO, CSO, ENTERPRISE mirror |
| mesh, avatar, spells | All six mirrors simultaneously |

---

*Dencken Network — Knowledge Architecture — dencken.net — August 2026*
# Dencken Network — Node Types
**Version:** v4.1 — August 2026

---

## Three node types

### Server Node
Always-on. Full constitutional layer. Runs scheduled deliberation cycles, maintains the ledger and knowledge store, hosts the board interface.

**Current instance:** node.dencken.net — Plesk shared hosting, Node.js via Passenger
**Required modules:** all src/core/ + all src/layers/ + src/agents/ + src/board/
**Entry point:** app.js (Passenger default)

### Device Node
Client-side. Interface layer. Connects to a server node for ledger and knowledge.
**Portable from server node:** all src/core/ modules unchanged
**First device node interface:** determined by the network's own deliberation cycles once Phase 1 is stable.

### Embed Node
Embedded in any application. Minimal footprint.
**Portable from server node:** src/core/ + src/agents/ + src/protocols/

---

## node-meta.json structure

```json
{
  "node_id": "dencken-server-node-0",
  "node_type": "server",
  "network": "dencken-network",
  "public_key_file": "config/node-identity.pub",
  "brief_version": "1.0.0",
  "initialized_at": "2026-08-15T00:00:00Z",
  "peers": [],
  "capabilities": ["cycle", "pulse", "board", "chat"],
  "extensions_pending": ["roles", "mirrors", "protocols"]
}
```

Safe to commit. No secrets.

---

*Dencken Network — Node Types — dencken.net — August 2026*
# Dencken Network — Network Structure
**Version:** v4.1 — August 2026

---

## Three network tracks

| Track | Governance | Purpose |
|---|---|---|
| Company Node 0 | Single board. No DAO. | Foundation. Current operational entity. |
| Enterprise Network | Board + executive agents. No DAO. | Commercial operations. |
| Production Network | Board + technical agents. No DAO. | Development at network scale. |
| Governance Network | DAO + Guardian Council when proven. | Collective decision-making. |

---

## Growth sequence

```
Company Node 0 (now)
  ↓ when self-sustaining
Enterprise Network + Production Network
  ↓ when both demonstrate stable operation
Governance Network (DAO activates)
  ↓ when Governance Network matures
Federation outward (decided from within Governance Network)
```

---

## Constitutional DNA transfer

When a new node or network is founded:
1. Full constitutional manifest (encrypted, from dencken-internal)
2. Fresh ledger with genesis transaction referencing parent node
3. Knowledge store export (founding institutional memory)
4. Agent role briefs (constitutional context, not model-specific)

New node generates its own keypair. Identity is its own. Constitution inherited. Character develops from its own deliberation history.

---

## Human floor (future Governance Network)

| Level | Human votes required |
|---|---|
| Low | 0% |
| Medium | 40% |
| High | 60% |
| Critical | 75% |

No single AI provider may hold >25% of council votes.

---

## Democratic identity

Democratic governance is not a starting condition. It is an achievement earned through demonstrated operational maturity and community readiness. The federation decision is made from within the Governance Network — not imposed by the founding board.

---

*Dencken Network — Network Structure — dencken.net — August 2026*

Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.