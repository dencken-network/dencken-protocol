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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.