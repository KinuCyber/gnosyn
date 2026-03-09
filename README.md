# Gnosyn
## Overview
An experiment in identity-bound federation over Tor onion service. Individuals can sign up through running "Tor Hidden Dir", using the generated hostname to serve their PDS anonymously

## Problem Statement
Mainstream knowledge and social platforms rely on centralized, DNS identity and IP infrastructure.
Tor provides self-certifying anonymous identities but lacks federated data layer built on top of them

Gnosyn explores whether tor hidden service can serve as foundation for federated data exchange

## Thesis
Onion addresses bind identity to routing at cryptographic level.
This enables identity-bound federation without external identity infrastructure such as DNS or centralized registries

Gnosyn tests this hypothesis

## Scope of Phase 0 of Roadmap
- Identity = Onion Address
- Node exposes signed JSON objects
- Peers fetch and verify via public key derived from hostname
- Minimal object types
  - Profile
  - Post
  - Follow
  - Message envelope

## Non-Goals
- Competing with Bluesky
- Replacing Matrix
- Large-scale indexing
- Global search
- Monetization
This is purely experimentation

## Minimal PoC (bound to change until 1st roadmap is achieved)
- Identity hosting via onion
- Graph endpoints
- Signed object exchange
- Basic flow + fetch

## Long-Term Direction
- Graph-based data layer, file-based data structure
- Anonymous communities
- Optional indexing nodes
  - Indexing introduces trust asymmetry
- Interoperability with other onion-native apps

## Gnosyn meaning
Derived from two greek words, **Gno**se (meaning "knowledge") and **syn** (meaning "joint").
In essence, Gnosyn is a knowledge joint. 

## Identity model
- PDS through tor hidden service, where assigned hostname serves as Decentralized IDentity

## Federation concept
Users can host PDS over tor through Hidden Service, allowing multiple platforms (including gnosyn) to aggregate and work through the data

## Roadmap
0. Validate Core Hypothesis: Can two onion-hosted identities exchange signed objects reliably in federated manner?
  - 0.1. To test
    - Onion A
    - Onion B
    - Signed JSON
    - Fetch + Verify
  - 0.2. To determine
    - Is this painfully slow, unreliable or impractical? Entire vision must adapt
    - is this viable, reliable and practical? Now we have a backbone
1. Primitives
  - 1.1. Identity Primitives: `Onion hostname = DID + key root`
    - Can onion hostname itself act as identity?
    - Embed public key inside payload?
    - Rely on tor key only?
    - How to rotate keys?
    - How to revoke?
  - 1.2. Federation Primitives
    - `Follow (actor)`
    - `Publish (object)`
    - `Fetch (actor)`
    - `Verify (signature)`
    - `Resolve (identity -> endpoint)`
    - Questions to ask
      - Can Federation primitives be defined consistently?
      - Can Onions reliably maintain flow?
      - Are federated graphs viable?
2. Abuse and Scale Stress Test
  - 2.1. Goal: Figure out how much stress it can endure through following
    - Spam simulaiton
    - Sybil swarm simulation
    - Malicious nodes
    - Onion downtime
    - Identity spoof attempts
    - Key compromise scenario
  - 2.2. Determine
    - If it collapses early: Need to revisit the vision
    - If it collapses late: Needs thorough improvement
    - If stable: Document limits and constraints
3. Add a single UX layer
  - 3.1. Minimal Web UI + CLI Client
    - Something incredibly simple to prove "This feels real"
