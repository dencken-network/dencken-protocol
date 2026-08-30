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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.