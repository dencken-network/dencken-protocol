# Dencken Network — Node Types
**Version:** v4.1 — August 2026

---

## Three node types

### Server Node
Always-on. Full constitutional layer. Runs scheduled deliberation cycles, maintains the ledger and knowledge store, hosts the board interface.

**Current instance:** node.dencken.net — Plesk shared hosting, Node.js via Passenger
**Required modules:** all src/core/ + all src/layers/ + src/agents/ + src/board/
**Entry point:** app.js (Passenger default)

### Device Node
Client-side. Interface layer. Connects to a server node for ledger and knowledge.
**Portable from server node:** all src/core/ modules unchanged
**First device node interface:** determined by the network's own deliberation cycles once Phase 1 is stable.

### Embed Node
Embedded in any application. Minimal footprint.
**Portable from server node:** src/core/ + src/agents/ + src/protocols/

---

## node-meta.json structure

```json
{
  "node_id": "dencken-server-node-0",
  "node_type": "server",
  "network": "dencken-network",
  "public_key_file": "config/node-identity.pub",
  "brief_version": "1.0.0",
  "initialized_at": "2026-08-15T00:00:00Z",
  "peers": [],
  "capabilities": ["cycle", "pulse", "board", "chat"],
  "extensions_pending": ["roles", "mirrors", "protocols"]
}
```

Safe to commit. No secrets.

---

*Dencken Network — Node Types — dencken.net — August 2026*
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.