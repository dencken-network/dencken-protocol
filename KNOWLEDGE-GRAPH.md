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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.