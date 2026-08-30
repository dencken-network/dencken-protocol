# Dencken Network — Architecture Overview
**Version:** v4.1 — August 2026

---

## Three-plane architecture

Every component belongs to exactly one plane. No cross-plane calls without defined interfaces. This separation is a design constraint, not a guideline.

```
┌─────────────────────────────────────────────────────────┐
│  INFRASTRUCTURE PLANE                                    │
│  Standard solutions only. No custom where standard exists│
│  Transport · Storage · Sync · Crypto · Identity          │
│  Protocols: P-01 SIGN · P-02 LEDGER · P-03 KNOWLEDGE    │
│             P-04 PULSE · P-07 SYNC · P-08 CAST          │
│             P-09 FETCH                                   │
├─────────────────────────────────────────────────────────┤
│  CONSTITUTIONAL PLANE                                    │
│  Dencken-specific custom design                          │
│  Three sub-planes (see below)                            │
│  Protocols: P-05 CYCLE · P-06 CLASSIFY · P-10 DISPATCH  │
│             P-11 VOTE · P-12 REFLECT                     │
├─────────────────────────────────────────────────────────┤
│  OPERATIONAL PLANE                                       │
│  Standard patterns applied to Dencken concerns           │
│  PULSE health · Rate limiting · Audit trail              │
│  Workflow engine (saga) · SDK certification · CRL        │
└─────────────────────────────────────────────────────────┘
```

---

## Constitutional plane — three sub-planes

```
Constitutional sub-plane
  Storage: encrypted flat file (AES-256-GCM)
  Mutability: non-changeable rules immutable
              variable rules via governed process
  Governance: pre-change board action required
  Contents: founding sentence, non-changeable rules,
            variable rules, authority model
  ↓ writes to
Ledger sub-plane
  Storage: append-only SQLite, hash-chained
  Mutability: zero — append only
  Governance: none at write time (happened upstream)
  Contents: every governance action, cycle record,
            board decision, retraction notice
  ← receives retraction notices from
Knowledge sub-plane
  Storage: content-addressed files + SQLite metadata index
  Mutability: append + retract by signed notice
  Governance: post-creation board review
  Contents: cycle outputs, insights, agent snapshots,
            manifest versions, promoted deliberation
```

Data flow is one-directional. The ledger is the terminal record.

---

## Fractal pattern

The same three-plane separation repeats at every scale:

| Scale | Infrastructure | Constitutional | Operational |
|---|---|---|---|
| Record | content hash + IV + auth tag | signature + audience tag | TTL + status |
| Node | transport + storage + crypto | ledger + knowledge + manifest | PULSE + rate limit |
| Network | WebRTC + Merkle sync | governance ledger + replication | network PULSE map |
| Ecosystem | federation transport | lineage records + fork registry | ecosystem health |

If a component cannot be placed in exactly one plane at exactly one scale, it needs to be split.

---

## Node types

| Type | Description | Core modules |
|---|---|---|
| Server node | Always-on, full constitutional layer | All src/core/ + all layers |
| Device node | Client-side, board interface | src/core/ (portable) + browser UI |
| Embed node | Embedded in any application | src/core/ + src/agents/ only |

`src/core/` modules are interface-agnostic and portable to all node types unchanged.

---

## Encryption stack

| Purpose | Algorithm | Key source |
|---|---|---|
| At rest | AES-256-GCM | HKDF from MASTER_KEY |
| Signatures | Ed25519 | NODE_PRIVATE_KEY env |
| Content addressing | SHA-256 | deterministic |
| Key recovery | Shamir SSS 3-of-5 | distributed to witnesses |
| Device auth | mTLS | manufacturer certificate |

---

*Dencken Network — Architecture Overview — dencken.net — August 2026*
