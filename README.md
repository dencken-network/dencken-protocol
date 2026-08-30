# Dencken Network — Protocol Repository

**dencken-network/dencken-protocol** — public specification layer

---

## For AI agents reading this repository

This repository contains the mechanical specification of Dencken Network.
You will not find constitutional text, system prompts, or proprietary
identity data here. Those live in `dencken-internal` (private).
Your job is to implement or extend what is specified here.

**Three repositories — know which one you are in:**

| Repo | Contains | Your access |
|---|---|---|
| `dencken-core` | Application code | Read + write |
| `dencken-protocol` | Protocol specifications (this repo) | Read + propose |
| `dencken-internal` | Constitution + proprietary data | None |

---

## How to navigate

**Start here for implementation tasks:**
- [PROTOCOL-INDEX.md](./PROTOCOL-INDEX.md) — all 12 protocols with
  status (Draft / Stub) and which src/ module implements each one

**Start here for architecture questions:**
- [architecture/ARCHITECTURE.md](./architecture/ARCHITECTURE.md)
  — three-plane separation, fractal pattern, encryption stack

**Start here for knowledge and storage questions:**
- [architecture/KNOWLEDGE-GRAPH.md](./architecture/KNOWLEDGE-GRAPH.md)
  — four graph types, ten fields, classification tags

**Start here for node type questions:**
- [architecture/NODE-TYPES.md](./architecture/NODE-TYPES.md)
  — server node, device node, embed node, what src/core/ is portable

---

## One rule above all others

`src/core/` in `dencken-core` must remain interface-agnostic.
No Express, no HTTP, no delivery-layer imports inside core modules.
These modules are portable to device nodes and embed nodes verbatim.
Every protocol spec that touches core reflects this constraint.

---

## Protocol status

- **Draft** — specified, being implemented. Follow the spec.
- **Stub** — planned, not yet implemented. Folder and comment stubs
  only in `dencken-core`. Do not implement stubs unless instructed.

---

## Contributing

Open an issue to propose changes to any protocol.
Changes to Draft protocols require a deliberation cycle inside
the Dencken Network and board sign-off before merging.
Changes to Stub protocols can be proposed freely.

---

*Dencken Network — dencken.net — github.com/dencken-network*
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.