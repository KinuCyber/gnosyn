# Gnosyn — Phase 0 Design Document

## Overview

Phase 0 validates the core hypothesis: can two onion-hosted nodes reliably exchange signed JSON objects in a federated manner? This document captures the object structure, signing mechanics, transport model, and exchange flow sketched up for Phase 0.

---

## Identity Primitive

Tor v3 onion addresses are not arbitrary — they encode the node's Ed25519 public key directly in the hostname (32 bytes of pubkey + checksum + version, base32-encoded). This means:

- A verifier can extract the public key from the actor's onion address alone
- No key lookup, no registry, no external trust anchor
- Identity, routing, and cryptographic verification collapse into a single primitive

---

## Signed Object Envelope

Every signed object, regardless of type, shares a common envelope:

```json
{
  "type": "<object type>",
  "actor": "<signer's onion address>",
  "ts": 1700000000,
  "kid": "onion-primary",
  "payload": { ... },
  "sig": "<base64url Ed25519 signature>"
}
```

| Field | Description |
|-------|-------------|
| `type` | Object kind discriminator |
| `actor` | Signer's `.onion` address — also the identity and key source |
| `ts` | Unix timestamp — basis for replay attack mitigation |
| `kid` | Key ID — always `"onion-primary"` in Phase 0; reserved for key rotation in Phase 1 |
| `payload` | Object-specific data |
| `sig` | Ed25519 signature over the canonical form of `{type, actor, ts, payload}` |

---

## Signing and Verification

### Canonicalization

JSON has a serialization ambiguity problem — key ordering is not guaranteed. All signing and verification uses **JCS (JSON Canonicalization Scheme, RFC 8785)**: keys sorted lexicographically, no extra whitespace, UTF-8 encoding.

### Signing

```
sig = Ed25519Sign(
  privateKey,
  JCS({ type, actor, ts, kid, payload })
)
```

### Verification

```
pubkey  = extract_ed25519_pubkey_from_onion(actor)
valid   = Ed25519Verify(pubkey, JCS({ type, actor, ts, kid, payload }), sig)
```

No external resolution step. The onion address is the only trust anchor needed.

---

## Phase 0 Object Types

### Profile

```json
{
  "type": "profile",
  "actor": "abc123.onion",
  "ts": 1700000000,
  "kid": "onion-primary",
  "payload": {
    "display_name": "...",
    "bio": "...",
    "endpoint": "http://abc123.onion"
  },
  "sig": "..."
}
```

### Post

```json
{
  "type": "post",
  "actor": "abc123.onion",
  "ts": 1700000000,
  "kid": "onion-primary",
  "payload": {
    "id": "<unique-post-id>",
    "content": "...",
    "reply_to": null
  },
  "sig": "..."
}
```

### Follow

```json
{
  "type": "follow",
  "actor": "abc123.onion",
  "ts": 1700000000,
  "kid": "onion-primary",
  "payload": {
    "target": "xyz456.onion"
  },
  "sig": "..."
}
```

### Message Envelope

```json
{
  "type": "message",
  "actor": "abc123.onion",
  "ts": 1700000000,
  "kid": "onion-primary",
  "payload": {
    "to": "xyz456.onion",
    "body": "..."
  },
  "sig": "..."
}
```

---

## Transport Model

Each node runs a Tor process doing two things simultaneously:

1. **Inbound** — exposes a hidden service; other nodes reach it at `yournode.onion`
2. **Outbound** — provides a SOCKS5 proxy on `127.0.0.1:9050`; all outbound requests are routed through it

Transport is plain HTTP. Tor handles anonymity and routing transparently.

```
Node B                        Tor Network                    Node A
  │                               │                             │
  ├─ HTTP GET via SOCKS5:9050 ───►│                             │
  │                               ├──── onion circuit ─────────►│
  │                               │                             ├─ serves
  │                               │                             │  signed
  │                               │                             │  JSON
  │◄──────────────────────────────┼─────────────────────────────┤
```

### Critical: use `socks5h` not `socks5`

The `h` variant delegates DNS resolution to the proxy. This is required for `.onion` addresses, which don't resolve on the public internet.

```python
session.proxies = {
    'http': 'socks5h://127.0.0.1:9050',
}
```

---

## Endpoints

Each node exposes a minimal HTTP server bound to `127.0.0.1` (Tor handles onion exposure):

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/profile` | Returns signed profile object |
| `GET` | `/posts` | Returns array of signed post objects |
| `GET` | `/follows` | Returns array of signed follow objects |
| `POST` | `/inbox` | Receives a signed object from another node |

---

## Exchange Flow

### Pull (Node B fetches from Node A)

```
1. Node B knows Node A address: abc123.onion  (out of band for Phase 0)
2. Node B → GET http://abc123.onion/profile  via SOCKS5
3. Node A responds with signed JSON object
4. Node B reads `actor` field → "abc123.onion"
5. Node B asserts actor === onion address
6. Node B extracts Ed25519 pubkey from onion address
7. Node B runs JCS canonicalization on {type, actor, ts, kid, payload}
8. Node B verifies sig against extracted pubkey
9. Accept or reject
```

Pull is the **reliability baseline** for Phase 0.

### Push (Node A delivers to Node B)

```
1. Node A → POST http://xyz456.onion/inbox  (body = signed object)
2. Node B receives, runs same verification steps
3. Node B reads `actor` field → "abc123.onion"
4. Node B extracts Ed25519 pubkey from onion address
5. Node B asserts actor === onion address
6. Node B runs JCS canonicalization on {type, actor, ts, kid, payload}
7. Node B verifies sig against extracted pubkey
8. Accept or reject
```

Push is **opportunistic**. A failed push is not an error state — Node B can always pull.

### Verificaion
Note that node B performs verification in the last 5 steps of both pull and push.

```
1. Node B reads `actor` field → "abc123.onion"
2. Node B asserts actor === onion address
3. Node B extracts Ed25519 pubkey from onion address
4. Node B runs JCS canonicalization on {type, actor, ts, kid, payload}
5. Node B verifies sig against extracted pubkey
6. Accept or reject
```

---

## Tor Reliability Considerations

Onion circuits are not like clearnet TCP connections. Key failure modes to handle:

| Issue | Behaviour | Mitigation |
|-------|-----------|------------|
| Circuit build time | First connection: 10–30s | Set connect timeout ≥ 30s |
| Circuit drops | Mid-request disconnect | Retry with exponential backoff |
| Node offline | No circuit establishes | Expected; not a loud error in Phase 0 |

### Recommended timeout and retry defaults

```
connect_timeout : 30s
read_timeout    : 60s
max_retries     : 3
backoff         : 2s → 4s → 8s
```

---

## Recommended Stack

| Concern | Library |
|---------|---------|
| Tor control | `stem` |
| Outbound HTTP (SOCKS5) | `requests` + `PySocks` |
| Inbound HTTP server | `flask` or `fastapi` |
| Ed25519 signing/verification | `cryptography` |
| JCS canonicalization | `jcs` (PyPI) |

---

## Phase 1 Debt Flags

| Decision | Debt |
|----------|------|
| Pubkey always derived from onion hostname | Phase 1 introduces key rotation — new key, same onion. Without `kid`, this is a breaking schema change. **Mitigated: `kid` field reserved now, defaulting to `"onion-primary"`** |
| `/inbox` accepts any verified inbound object | Phase 2 is where spam/Sybil enters. Keep acceptance logic as a clean, isolated function now: `if verify(obj): store(obj)` |

---

## What Phase 0 Must Determine

- Is onion-to-onion HTTP exchange **reliably fast enough** to be practical?
- Does signature verification from the onion address work end-to-end?
- Is the circuit stability acceptable for federated pulls?

If yes → we have a backbone. If no → the vision must adapt before Phase 1.
