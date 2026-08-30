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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.