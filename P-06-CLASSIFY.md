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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.