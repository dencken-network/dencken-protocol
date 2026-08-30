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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.