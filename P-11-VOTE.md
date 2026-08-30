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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.