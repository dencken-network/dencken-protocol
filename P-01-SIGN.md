# P-01 — SIGN

**Layer:** Infrastructure
**Status:** Draft v0.1 — August 2026
**Implemented in:** `src/core/crypto.js`, `src/core/records.js`

\---

## Purpose

Every record produced by a Dencken node must be signed before it leaves the Node-Local bucket. SIGN is the foundational integrity guarantee: it makes every record attributable to a specific node and tamper-evident.

Without SIGN, the ledger is a log. With SIGN, the ledger is a constitutional record.

\---

## Algorithm

**Ed25519** — Edwards-curve Digital Signature Algorithm over Curve25519.

Chosen because:

* Fast verification (critical at retrieval time)
* Small signature size (64 bytes)
* No parameter choices that can be misconfigured
* Native in Node.js `crypto` module (no external dependency)

\---

## Record structure

Every signed record contains exactly these fields:

```json
{
  "id": "uuid-v4",
  "created\\\_at": "ISO8601 UTC",
  "record\\\_type": "cycle | pulse | board\\\_action | system | knowledge | retraction",
  "brief\\\_version": "1.0.0",
  "field": "constitution | governance | operational | learning | self\\\_reflection | application | enterprise | mesh | avatar | spells",
  "audience": "node-private | internal | board-only | public",
  "graph\\\_type": "policy | knowledge | task\\\_state | episodic",
  "content\\\_hash": "sha256-hex",
  "content\\\_encrypted": "base64-aes-256-gcm",
  "author\\\_pubkey": "hex",
  "signature": "base64-ed25519",
  "prev\\\_hash": "sha256-hex",
  "status": "pending\\\_review | promoted | discarded",
  "board\\\_note": "string | null"
}
```

\---

## Signing procedure

```
1. Assemble plaintext content (JSON string, deterministic key order)
2. Compute content\\\_hash = SHA-256(plaintext)
3. Encrypt plaintext → content\\\_encrypted (AES-256-GCM, see P-02)
4. Compute prev\\\_hash = SHA-256(previous ledger entry's content\\\_hash)
   — Genesis record: prev\\\_hash = SHA-256("GENESIS")
5. Assemble signing payload:
   payload = content\\\_hash + "|" + prev\\\_hash + "|" + created\\\_at + "|" + record\\\_type
6. signature = Ed25519.sign(payload, node\\\_private\\\_key)
7. Record is complete. Move from Pending bucket to Ledger.
```

\---

## Verification procedure

```
1. Recompute payload from record fields
2. Ed25519.verify(payload, signature, author\\\_pubkey)
3. Verify content\\\_hash matches SHA-256 of decrypted content
4. Verify prev\\\_hash matches previous ledger entry
5. All three must pass. Any failure = record invalid.
```

\---

## Key management

* **Private key:** Ed25519, generated on first boot, stored in `.env` as `NODE\\\_PRIVATE\\\_KEY` (PEM). Never written to disk in plaintext. Never logged.
* **Public key:** stored in `config/node-identity.pub`. Safe to commit to `dencken-core`. Used by peers and external verifiers.
* **Key recovery:** Shamir Secret Sharing, 3-of-5 shares. Shares distributed to trusted witnesses. See P-07 (SYNC) for multi-node key trust.

\---

## What SIGN does NOT cover

* Encryption of content (see P-02 LEDGER for AES-256-GCM)
* Routing decisions (see P-06 CLASSIFY)
* Audience enforcement (see P-09 FETCH)

\---

## Implementation notes

```javascript
// src/core/crypto.js — sign a payload
import { createSign } from 'crypto';

export function signPayload(payload, privateKeyPem) {
  const sign = createSign('Ed25519');
  sign.update(payload);
  return sign.sign(privateKeyPem, 'base64');
}

export function verifySignature(payload, signatureB64, publicKeyPem) {
  const verify = createVerify('Ed25519');
  verify.update(payload);
  return verify.verify(publicKeyPem, signatureB64, 'base64');
}
```

\---

*Dencken Network — P-01 SIGN — dencken.net — August 2026*
Copyright (c) 2026 Dencken - Oddsized - CP Müller. All Rights Reserved.