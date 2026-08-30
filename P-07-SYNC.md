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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.