# Gnosyn
## Overview
An experiment in identity-bound federation over Tor onion service.
Individuals can sign up through running "Tor Hidden Dir", using the generated hostname
to serve their PDS anonymously

## Problem Statement
Mainstream knowledge and social platforms rely on centralized, DNS identity and IP infrastructure.
Tor provides anonymity but lacks federated application

Gnosyn explores whether tor hidden service can serve as foundation for federated data exchange

## Thesis
Onion hostnames are self-certifying public keys
This enables identity-bound federation without DNS, DID or central registries

Gnosyn tests this hypothesis

## Scope of Phase 1
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
- Interoperability with other onion-native apps

## Gnosyn meaning
Derived from two greek words, **Gno**se (meaning "knowledge") and **syn** (meaning "joint").
In essence, Gnosyn is a knowledge joint. 

## Architecture Overview (Bound to change until Phase 3 of roadmap)
Architecture is in two direction
- Gnosyn app itself
  - Frontend for interaction (terminal | Web | Android)
    - Messenger
      - lets DM and community messages
      - Imports profile from user's hostname
      - Lets create communities (needs hoster)
    - Wall
      - Lets DM and public posts
      - Lets private posting
      - Improts profile from user's hostname

  - APIs for communication and adoption
  
  - Backend
    - Messenger (by gnosyn, to users)
      - Community messages will be encrypted by public key of community (derived from hostname)
      - DMs will be encrypted by the public key of recipient
      - User will need to decrypt DM from their private key (if DMed) or community private key (if community message)
    - Wall (by gnosyn, to users)
      - Users can post stuff
      - Not encrypted (to be used by everyone)

- Gnosyn Server (for wall)
  - Frontend
    - Hosted by me and others, with individual private key
  - Backend
    - Private key pool (not sure about this)
      - Lets multiple people host the same server and database
    - Database (for gnosyn)
      - I've only heard that database is used to aggregate user data
      - So will most likely need to implement this
      - Will only save encrypted messages

- Any kickstarters
  - Hoster (not part of gnosyn, just to kickstart)
    - Uses tor hidden service library
    - Lets user create their identity
    - Stores the private key and informs user
    - User can use the same private key to host itself again
  - 


## Identity model
- Randomly assigned guest identity (with reduced capabilities to reduce abuse surface)
- PDS through tor hidden service, where assigned hostname serves as Decentralized IDentity

## Federation concept
Users can host PDS over tor through Hidden Service, allowing multiple platforms (including gnosyn) to aggregate and work through the data

## Roadmap
0. Validate Core Hypothesis: Can two onion-hosted identities exchange signed objects reliably in federated manner?
  0.1 To test
    - Onion A
    - Onion B
    - Signed JSON
    - Fetch + Verify
  0.2 To determine
    - Is this painfully slow, unreliable or impractical? Entire vision must adapt
    - is this viable, reliable and practical? Now we have a backbone
1. Architectural Layout
  1.1 Breaking down AT Protocol architecture
  1.2 Grasping the nature of Federation (including SMTP architecture)
  1.3 Breaking down Tor Hidden Service architecture
  1.4 Mapping Tor Hidden Service architecture over AT Protocol
  1.5 Brainstorming through any gaps
  1.6 Bridging any gaps in architecture theoretically
2. Technology Layout
  2.1 Figuring out techstack required/preferred
  2.2 Laying the architecture over selected techstack
  2.3 Forming and separating layers and architectures
  2.4 Finalizing a rough implementation
3. Developing Proof of Concept
  3.1 Preparing mininal front-ends (for web and terminal) and APIs
  3.2 Preparing testing units guideline
  3.3 Preparing layers from bottom-up
  3.4 Preparing backend
  3.5 Testing all layers separately
  3.6 Integrating layers
  3.7 Preparing deployment layer
  3.8 Alpha Testing
  3.9 Fixing bugs and Documentation
  p.s. Saying layers due to unknown number of moving parts
4. Developing v1
  4.1 Preparing for continious Beta testing
  4.2 Polishing UI/UX
  4.3 Preparing rudimentary Android App
  4.4 Laying down the whole concept in standardized documentation
  4.5 Inviting public to explore the concept
  4.6 Preparing v1 for continued iteration
