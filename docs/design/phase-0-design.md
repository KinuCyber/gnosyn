# Gnosyn — Phase 0 Design Document

## Overview

Phase 0 validates the core hypothesis: can two onion-hosted nodes reliably exchange signed JSON objects in a federated manner? This document captures the object structure, signing mechanics, transport model, and exchange flow agreed upon for Phase 0.

---

## CNCF Conventions

Gnosyn follows Linux Foundation / CNCF conventions where they align with its constraints. The goal is to avoid reinventing conventions, ease adoption, and enable future interoperability without introducing centralization or external trust dependencies.

### Adopted

| Convention | Usage |
|------------|-------|
| **CloudEvents 1.0** | Envelope format for all signed objects |
| **JCS — RFC 8785** | Canonical serialization for signing surface |
| **OpenTelemetry** | Instrumentation for hypothesis measurement (see instrumentation doc) |

### Evaluated and Excluded

| Convention | Reason Excluded |
|------------|-----------------|
| **SPIFFE / SPIRE** | Requires a central control plane — incompatible with Gnosyn's no-centralization constraint |
| **Sigstore** | Rekor transparency log is a centralized service — same incompatibility |
| **OpenMetrics** | OpenTelemetry has sufficent metrics features for this experiment |
| **CloudEvents Subscriptions API** | Relevant for follow/delivery model — deferred to Phase 1 |
| **in-toto** | Relevant for object provenance and relay attestation — deferred to Phase 1 |

### CloudEvents design decisions

- Extension attributes prefixed `gn` — short, unambiguous, collision-safe
- `datacontenttype` omitted — Gnosyn is JSON-only; Adds no value in Phase 0
- `time` uses RFC 3339 — aligns with CloudEvents ecosystem; canonicalizes unambiguously under JCS

---

## Identity Primitive

Tor v3 onion addresses are not arbitrary — they encode the node's Ed25519 public key directly in the hostname (32 bytes of pubkey + checksum + version, base32-encoded). This means:

- A verifier can extract the public key from the actor's onion address alone
- No key lookup, no registry, no external trust anchor
- Identity, routing, and cryptographic verification collapse into a single primitive

---

## Signed Object Envelope

Gnosyn envelopes are **CloudEvents 1.0 compliant** (CNCF spec). Every signed object shares a common CloudEvents envelope, with Gnosyn-specific fields carried as CloudEvents extension attributes (prefixed `gn` for brevity and collision-safety).

```json
{
  "specversion": "1.0",
  "id": "<uuid-per-event>",
  "type": "io.gnosyn.<object-type>",
  "source": "http://abc123.onion",
  "time": "2024-01-01T00:00:00Z",
  "gnkid": "onion-primary",
  "gnsig": "<base64url Ed25519 signature>",
  "data": { ... }
}
```

| Field | CloudEvents field | Description |
|-------|-------------------|-------------|
| `specversion` | Required | Always `"1.0"` |
| `id` | Required | UUID unique per event — used for deduplication at `/inbox` and cross-referencing between objects |
| `type` | Required | Reverse-DNS namespaced: `io.gnosyn.post`, `io.gnosyn.profile`, etc. |
| `source` | Required | Signer's onion address as URI — also the identity and key source |
| `time` | Optional (used) | RFC 3339 timestamp — included in signing surface; receiver-side replay mitigation policy deferred to Phase 2 |
| `gnkid` | Extension attribute | Key ID — always `"onion-primary"` in Phase 0; reserved for key rotation in Phase 1 |
| `gnsig` | Extension attribute | Ed25519 signature over the canonical form of the signing surface |
| `data` | Required | Object-specific payload |

Note: In source, we use `http://` instead of `https://` because Tor handles encryption at transport layer. HTTP is correct here

---

## Signing and Verification

### Canonicalization

JSON has a serialization ambiguity problem — key ordering is not guaranteed. All signing and verification uses **JCS (JSON Canonicalization Scheme, RFC 8785)**: keys sorted lexicographically, no extra whitespace, UTF-8 encoding.

### Signing surface

The signature is computed over the CloudEvents required and Gnosyn extension fields that carry semantic meaning — `data` is included, `gnsig` itself is excluded:

```
sig = Ed25519Sign(
  privateKey,
  JCS({ specversion, id, type, source, time, gnkid, data })
)
```

### Signing

```python
import jcs
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey

signing_input = jcs.canonicalize({
    "specversion": event["specversion"],
    "id":          event["id"],
    "type":        event["type"],
    "source":      event["source"],
    "time":        event["time"],
    "gnkid":       event["gnkid"],
    "data":        event["data"],
})

event["gnsig"] = base64url_encode(private_key.sign(signing_input))
```

### Verification

```python
pubkey = extract_ed25519_pubkey_from_onion(event["source"])

signing_input = jcs.canonicalize({
    "specversion": event["specversion"],
    "id":          event["id"],
    "type":        event["type"],
    "source":      event["source"],
    "time":        event["time"],
    "gnkid":       event["gnkid"],
    "data":        event["data"],
})

pubkey.verify(base64url_decode(event["gnsig"]), signing_input)
# raises InvalidSignature on failure
```

No external resolution step. The `source` onion address is the only trust anchor needed.

---

## Phase 0 Object Types

### Profile

```json
{
  "specversion": "1.0",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "io.gnosyn.profile",
  "source": "http://abc123.onion",
  "time": "2024-01-01T00:00:00Z",
  "gnkid": "onion-primary",
  "gnsig": "...",
  "data": {
    "display_name": "...",
    "bio": "..."
  }
}
```

### Post

```json
{
  "specversion": "1.0",
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "type": "io.gnosyn.post",
  "source": "http://abc123.onion",
  "time": "2024-01-01T00:00:00Z",
  "gnkid": "onion-primary",
  "gnsig": "...",
  "data": {
    "content": "...",
    "reply_to": null
  }
}
```

Note: `id` at the envelope level serves as the post's unique identifier — no need for a separate `id` in `data`.

### Follow

```json
{
  "specversion": "1.0",
  "id": "550e8400-e29b-41d4-a716-446655440002",
  "type": "io.gnosyn.follow",
  "source": "http://abc123.onion",
  "time": "2024-01-01T00:00:00Z",
  "gnkid": "onion-primary",
  "gnsig": "...",
  "data": {
    "target": "http://xyz456.onion"
  }
}
```

### Message

```json
{
  "specversion": "1.0",
  "id": "550e8400-e29b-41d4-a716-446655440003",
  "type": "io.gnosyn.message",
  "source": "http://abc123.onion",
  "time": "2024-01-01T00:00:00Z",
  "gnkid": "onion-primary",
  "gnsig": "...",
  "data": {
    "to": "http://xyz456.onion",
    "body": "..."
  }
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
3. Node A responds with CloudEvents-shaped signed JSON object
4. Node B verifies `source` field → "http://abc123.onion"
5. Node B extracts Ed25519 pubkey from onion address in `source`
6. Node B runs JCS canonicalization on {specversion, id, type, source, time, gnkid, data}
7. Node B verifies gnsig against extracted pubkey
8. Accept or reject
```

Pull is the **reliability baseline** for Phase 0.

### Push (Node A delivers to Node B)

```
1. Node A → POST http://xyz456.onion/inbox  (body = CloudEvents signed object)
2. Node B receives the object
3. Node B verifies `source` field → "http://abc123.onion"
4. Node B extracts Ed25519 pubkey from onion address in `source`
5. Node B runs JCS canonicalization on {specversion, id, type, source, time, gnkid, data}
6. Node B verifies gnsig against extracted pubkey
7. Accept or reject
```

Push is **opportunistic**. A failed push is not an error state — Node B can always pull.

### Verification

Both flows have the same verification method
```
1. Node B verifies `source` field → "http://abc123.onion"
2. Node B extracts Ed25519 pubkey from onion address in `source`
3. Node B runs JCS canonicalization on {specversion, id, type, source, time, gnkid, data}
4. Node B verifies gnsig against extracted pubkey
5. Accept or reject
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
| Envelope format | CloudEvents 1.0 (CNCF) — no additional library required, plain JSON |

---

## Phase 1 Debt Flags

| Decision | Debt |
|----------|------|
| CloudEvents Subscriptions API not yet adopted | Phase 1 follow/delivery model should evaluate this for standardizing subscription semantics |
| in-toto not yet adopted | Phase 1+ relay semantics should evaluate this for object provenance and intermediary attestation |
| Pubkey always derived from onion hostname | Phase 1 introduces key rotation — new key, same onion. Without `gnkid`, this is a breaking schema change. **Mitigated: `gnkid` field reserved now, defaulting to `"onion-primary"`** |
| `/inbox` accepts any verified inbound object | Phase 2 is where spam/Sybil enters. Keep acceptance logic as a clean, isolated function now: `if verify(obj): store(obj)` |

---

## What Phase 0 Must Determine

- Is onion-to-onion HTTP exchange **reliably fast enough** to be practical?
- Does signature verification from the onion address work end-to-end?
- Is the circuit stability acceptable for federated pulls?

If yes -> we have a backbone. If no -> the vision must adapt before Phase 1.
