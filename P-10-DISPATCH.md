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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.