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
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.