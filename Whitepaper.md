# Veil — Decentralized Messaging Protocol — Draft v0.4

> **Status:** Working draft / ideation. Not a formal spec.
> **Author:** Andrea Fiocchi
> **Date:** April 2026
> **License:** CC0 1.0 (this document) · Apache 2.0 (reference implementation, when available)

---

## 1. Motivation

Email is the only universal, open messaging protocol. Everything else — Signal, WhatsApp, Telegram — is a closed platform controlled by a single entity. Email's openness is its greatest strength, but it was designed for trusted networks in 1982 and carries fundamental architectural flaws that cannot be fixed from within:

- Plaintext by default; encryption bolted on decades later (PGP, S/MIME)
- Servers must parse message content for routing and filtering — they are structurally in the trust path
- Identity tied to a mail server, not to the user or domain owner
- No native key management, key rotation, or key discovery standard
- Metadata (sender, recipient, subject, timing) always exposed to every relay
- Store-and-forward assumes trusted intermediaries

Meanwhile, decentralized messaging projects (XMTP, Session, Status) solve some of these problems but introduce new ones: wallet-based identity that businesses will never adopt, token economics that conflate protocol usage with speculation, and no path to organizational identity.

Matrix is federated and open, but homeservers are stateful — they own your identity, your room memberships, your history. Switching servers means starting over.

Veil proposes a clean-break alternative built on three principles:

1. **A permissionless platform with no operator** — the protocol exists on a blockchain and requires no foundation, company, or consortium to function
2. **Domain-based identity as an optional overlay** — businesses can anchor organizational address spaces to their DNS domains without sacrificing the permissionless nature of the platform
3. **Relay nodes as disposable, storage-agnostic proxies** — infrastructure that holds nothing the user cannot replace in minutes

---

## 2. Design Goals

- **Permissionless** — anyone can register an identity and send messages without asking anyone's permission
- **No platform owner** — protocol state lives on-chain; no entity controls the registry, the routing, or the key infrastructure
- **Encrypted at rest, always** — relay nodes receive only ciphertext; they structurally cannot read message content
- **Domain identity as an optional overlay** — businesses anchor organizational address spaces to DNS; individuals use the protocol without a domain
- **Relay nodes as disposable proxies** — infrastructure is interchangeable; identity, keys, and messages do not belong to the relay
- **Storage-agnostic** — relay nodes abstract the storage layer; local disk, S3, R2, or any backend works behind the same interface
- **Honest threat model** — explicit security tiers, no overclaiming
- **Email interoperability** — reach legacy email users on day one via a bridge layer
- **Open protocol** — no owner, free implementation, community governed

---

## 3. Core Concepts

The protocol is built on three independent pillars. Each can be understood on its own.

### 3.1 Blockchain as the Permissionless Control Plane (Layer 0)

The blockchain is the protocol itself. It stores:

- Per-user identity records (public keys, relay endpoints)
- Key rotation history (immutable, auditable)
- Relay node registry

No messages are ever stored on-chain. The blockchain is a global, permissionless registry — a database that nobody owns and anybody can read and write to (with proper authorization).

This layer alone constitutes a fully functional decentralized messaging system. Two users with on-chain identities can exchange encrypted messages through relay nodes without any DNS, any domain, or any centralized infrastructure whatsoever.

### 3.2 DNS Identity Overlay (Layer 1)

DNS is an optional layer that enables domain-based addressing. A domain owner publishes a DNS TXT record linking their domain to an on-chain namespace contract, claiming an organizational address space.

This is the bridge between decentralized infrastructure and business adoption. It lets a company own `@acme.com` as a messaging namespace, create and revoke employee addresses, and integrate with existing IT governance — all while using the same permissionless protocol underneath.

**The identity resolution rule is simple:**

- If an address contains a domain (`alice@acme.com`), the client MUST verify the domain's DNS anchor before trusting the on-chain record. Both sender and recipient enforce this.
- If an address is a bare alias or key-based identifier, no DNS verification occurs. The on-chain record is authoritative.

This creates two first-class identity paths on a single protocol.

### 3.3 Relay Nodes as Disposable, Storage-Agnostic Proxies

A relay node is a lightweight process that performs three functions:

1. **Accepts inbound encrypted blobs** from senders
2. **Writes blobs to a storage backend** (local disk, S3, R2, MinIO, NAS — whatever is configured)
3. **Notifies the recipient's client** that a new message has arrived

The relay node does not own the user's identity (that's on-chain), does not hold the user's keys (that's the user's choice), and abstracts storage behind a pluggable interface. If a relay node disappears, the user updates their on-chain endpoint record, starts a new relay, and continues. Messages already written to external storage survive intact. Messages on local relay storage are lost only if the relay itself is unrecoverable — an explicit tradeoff of the local storage option.

A relay node is designed to run as a single container: `docker pull` and configure. Self-hosting should be a fifteen-minute operation, not a weekend project.

---

## 4. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 0: BLOCKCHAIN CONTROL PLANE                          │
│                                                             │
│  Per-user records:                                          │
│    • Public keys (encryption + signing)                     │
│    • Relay endpoint URL                                     │
│    • Identity type (bare alias / domain-verified)           │
│    • Key rotation history (immutable log)                   │
│                                                             │
│  Protocol registries:                                       │
│    • Relay node registry (listed / verified / unlisted)     │
│    • Domain namespace contracts (for DNS-verified domains)  │
│                                                             │
│  Properties: permissionless, ownerless, censorship-resistant│
│  Writes: registration, key rotation, endpoint changes only  │
│  Reads: every message send (key + endpoint lookup)          │
│  Messages on-chain: NEVER                                   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  LAYER 1: DNS IDENTITY OVERLAY (optional)                   │
│                                                             │
│  Domain owner publishes:                                    │
│    _veil.domain.com TXT → on-chain namespace contract       │
│                                                             │
│  Enables:                                                   │
│    • user@domain.com addressing                             │
│    • Organizational address space control                   │
│    • Domain-verified identity (enforced by both parties)    │
│                                                             │
│  Without this layer: protocol works with bare aliases       │
│  or key-based addresses. Fully functional, just no domain.  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  RELAY NODE                                                 │
│                                                             │
│  Inbound:   Accepts encrypted blobs via HTTPS endpoint      │
│  Storage:   Writes to configured backend (abstracted)       │
│             • Local disk (simplest, default for testing)    │
│             • S3-compatible (AWS S3, R2, MinIO, etc.)       │
│             • Any conforming backend                        │
│  Outbound:  Notifies client (WebSocket / push / long-poll)  │
│                                                             │
│  Properties: disposable, stateless by design, storage-      │
│  agnostic. A new relay can be spun up and pointed to the    │
│  same storage backend with zero data loss.                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  CLIENT                                                     │
│                                                             │
│  Local app, self-hosted interface, or third-party client    │
│  Decrypts locally using private key                         │
│  Signs outbound messages locally                            │
│  Verifies inbound signatures against on-chain public keys   │
└─────────────────────────────────────────────────────────────┘
```

**Key architectural property:** A user can replace their relay node, change storage backends, rotate keys, or switch clients without losing their identity. Identity lives on-chain (and optionally on DNS). Everything else is interchangeable.

---

## 5. Blockchain Control Plane (Layer 0)

### 5.1 On-Chain Record Structure

Each registered identity has an on-chain record. Records are keyed by a hash of the user's address — the plaintext address is never stored on-chain.

```
key:   hash(identity_string)

value: {
  encryption_key:     "<X25519 public key>",
  signing_key:        "<Ed25519 public key>",
  relay_endpoint:     "https://relay.example.com/inbox/<id>",
  identity_type:      "bare" | "domain-verified",
  domain_contract:    "<namespace contract address, if domain-verified>",
  protocol_version:   "1",
  updated_at:         <block_timestamp>,
  signature:          "<signed by registrant's signing key>"
}
```

Two keypairs are maintained per identity: an X25519 key for asymmetric encryption and an Ed25519 key for message signing. These MUST be separate keypairs — reusing a single key for both operations is a cryptographic anti-pattern.

The `relay_endpoint` is the relay node's inbound URL. This is the only address a sender needs to deliver a message.

### 5.2 Identity Types

**Bare identity (default)**

Any user can register an on-chain identity without a domain. The identity string is either a human-readable alias (subject to namespace rules to prevent squatting) or a key-derived identifier. No DNS verification is required or performed. This is the permissionless baseline — the protocol works at this layer alone.

**Domain-verified identity**

A user whose address includes a domain (`alice@acme.com`) has their on-chain record linked to a domain namespace contract. Clients interacting with domain-verified addresses MUST perform DNS verification (see Section 6). This is the business adoption path.

### 5.3 Sending a Message: One Lookup

A sender resolves a recipient's identity in a single on-chain query:

```
1.  Resolve identity_string to on-chain record
    → { encryption_key, signing_key, relay_endpoint, identity_type }

2.  If identity_type = "domain-verified":
    → Perform DNS verification (Section 6.2)
    → Abort if verification fails

3.  Construct envelope:
    → Sign hash(plaintext + sender + recipient + timestamp)
      with sender's Ed25519 private key

4.  Encrypt envelope with recipient's X25519 public key → blob

5.  POST encrypted blob to relay_endpoint

Done.
```

### 5.4 Key Rotation

Key rotation is a first-class on-chain operation. Publishing a new key creates a new record; the previous record remains in the chain's history as an immutable audit trail. Clients MUST detect unexpected key changes and alert the user prominently — an unexpected key change is a potential compromise signal.

### 5.5 Blockchain Write Costs

On-chain writes occur only for infrequent lifecycle events:

```
Per user, one-time:    Register identity record     →  1 write tx
Per user, occasional:  Key rotation                 →  1 write tx
                       Relay endpoint change        →  1 write tx

Per message:           Nothing. Zero on-chain writes.
```

At realistic L2 pricing ($0.01–$0.10/tx), a typical user incurs approximately 2–3 transactions per year. Reads (key lookups for message sending) are free on all major chains.

### 5.6 On-Chain Read Trust Model

Clients need to read on-chain records to resolve recipients. There are three approaches, each with a different trust profile:

**Via the user's own relay node (recommended default)**
The relay node runs a chain light client or full node. The user trusts their own relay for read integrity. Since the relay is self-hosted or explicitly chosen, this does not expand the trust surface beyond what the user has already accepted.

**Via a third-party RPC provider (Alchemy, Infura, etc.)**
Convenient but introduces a trust dependency — the RPC provider could return false records. Appropriate for convenience-tier deployments.

**Via a local light client in the app**
The client verifies on-chain reads cryptographically without trusting any intermediary. This is the highest-security option but depends on L2 light client maturity, which is still developing.

The whitepaper recommends relay-hosted reads as the practical default, with RPC and light clients as alternatives. The trust model for on-chain reads SHOULD be visible to the user in the client's security tier display.

### 5.7 Relay Node Registry

An on-chain registry lists relay nodes that meet published baseline criteria:

```
✓   Listed     — meets baseline criteria:
                  open source, reproducible builds,
                  published privacy policy, security contact
✓✓  Verified   — independently security audited
⚠   Unlisted   — use at own risk, explicit user warning required
```

Self-hosted relay nodes are inherently trusted by their operator and do not require registry listing.

---

## 6. DNS Identity Overlay (Layer 1)

### 6.1 Domain Namespace Registration

A domain owner anchors their address space to the blockchain by publishing a DNS TXT record:

```
_veil.domain.com  TXT  "v=1; chain=<chain>; contract=<namespace_contract>"
```

The namespace contract is an on-chain record that maps local-parts (alice, bob, hr) to individual identity records within that domain. The domain owner controls this contract — they can create, revoke, and reassign addresses.

This reuses an already-accepted trust assumption: every business already trusts DNS for domain ownership. Spoofing `alice@acme.com` requires controlling the DNS zone for `acme.com` — the same security model as HTTPS, DMARC, and enterprise SSO.

### 6.2 DNS Verification Rule

**When either party in a message exchange uses a domain-verified address, both clients MUST perform DNS verification:**

```
1.  Extract domain from address (e.g., acme.com from alice@acme.com)
2.  Query _veil.acme.com TXT record
3.  Verify it points to the same namespace contract as the on-chain record
4.  If mismatch → ABORT with prominent warning to user
5.  If DNS record absent → ABORT — domain has not opted into the protocol
```

This is enforced bidirectionally. If Alice (`alice@acme.com`) messages Bob (`bob@corp.com`), both clients verify both domains. If Alice messages Charlie (a bare alias with no domain), only Alice's domain is verified; Charlie's on-chain record is authoritative without DNS.

### 6.3 Per-User Address Resolution

Individual addresses within a domain are resolved via hashed DNS labels, following the approach established by RFC 7929 (OPENPGPKEY):

```
SHA-256(lowercase(localpart))[:28 bytes] + "._veil." + domain

alice@acme.com  →  a3f9b2c1...._veil.acme.com  →  on-chain record
```

The hash prevents DNS zone enumeration — an observer sees only opaque hashes, not a directory of valid addresses. Any sender who knows an address can compute the lookup directly.

**Brute-force caveat:** Common names and role addresses (alice, hr, info) are low-entropy and hashable from a wordlist. This is a known limitation shared with all DNS-based address systems including email itself. For high-security deployments, authenticated directory mode (Section 6.5) mitigates this.

### 6.4 Address Space Ownership

DNS hierarchy enables native organizational address spaces:

```
acme.com             →  company root (admin controls namespace contract)
alice@acme.com       →  individual address
hr@acme.com          →  department or role address
dept.acme.com        →  delegatable subdomain namespace
```

The domain admin controls the entire address space via the namespace contract. This is an intentional design decision: it maps directly to how businesses operate. IT controls `acme.com`; employees do not have personal sovereignty over company addresses.

**Personal sovereignty** requires owning a personal domain. The user who owns `alice.dev` is the sole admin of their namespace. Alternatively, users who do not want domain identity use a bare on-chain alias — no domain, no organizational control, full personal sovereignty.

### 6.5 Directory Privacy: Open vs. Authenticated

The protocol supports two directory modes, declared in the namespace contract:

**Open directory (default)**
Any sender resolves any address by computing the DNS hash and performing an on-chain lookup. Appropriate for businesses and individuals who want to be publicly reachable.

**Authenticated directory**
The namespace contract declares `directory: authenticated`. Senders must present a verified protocol identity (on-chain, optionally DNS-verified) to the recipient's relay node before receiving the public key and relay endpoint. The relay acts as a directory gatekeeper.

This creates a trust tradeoff: the relay gains visibility into who is attempting contact. This is stated explicitly because it is intentional — enterprises, law firms, and healthcare organizations often require access control over their directory.

---

## 7. Relay Nodes

### 7.1 What a Relay Node Does

A relay node is a lightweight process with three responsibilities:

```
1. ACCEPT     Receive encrypted blobs via HTTPS endpoint
              Validate sender identity (on-chain lookup)
              Validate blob format (encrypted, size limits)
              Apply rate-limiting / anti-spam rules

2. STORE      Write blob to configured storage backend
              Storage is abstracted — the relay node does not
              care where blobs go, only that the write succeeds

3. NOTIFY     Push notification to recipient's client
              (WebSocket, long-poll, push notification)
```

That is the complete scope. The relay node does not parse, decrypt, route based on content, or own any user state.

### 7.2 Storage Abstraction

The relay node exposes a pluggable storage interface. The protocol defines the interface contract; the implementation is the operator's choice:

```
storage.write(blob_id, encrypted_blob)  →  success / failure
storage.read(blob_id)                   →  encrypted_blob
storage.list(after_timestamp)           →  [blob_ids]
storage.delete(blob_id)                 →  success / failure
```

Conforming backends include:

- **Local disk** — simplest option, good for testing and small deployments. Data lives on the relay itself.
- **S3-compatible** — AWS S3, Cloudflare R2, MinIO, Backblaze B2. Data survives relay replacement.
- **Self-hosted NAS / network storage** — data on the user's own hardware, relay is a pure proxy.
- **Any future backend** — the interface is intentionally minimal to allow backends not yet imagined.

**Disposability depends on storage choice.** When storage is external (S3, NAS), the relay is fully disposable — spin up a new container, point it at the same storage, update the on-chain endpoint, done. When storage is local, the relay is replaceable but messages on local disk are lost if the relay is unrecoverable. This is an explicit, visible tradeoff — not a hidden risk.

### 7.3 What the Relay Node Sees

```
Relay DOES see:
  - Encrypted blob (ciphertext only — cannot read content)
  - Sender's identity and relay endpoint (from on-chain lookup)
  - Blob size and timing metadata
  - Client connection metadata (IP, device, activity patterns)

Relay does NOT see:
  - Message content (encrypted with recipient's public key)
  - Internal message metadata (subject, thread, attachments)
```

The residual metadata exposure (timing, size, activity) is a known and bounded tradeoff. Full metadata privacy (sealed sender, padding, dummy traffic) is a possible future extension but is out of scope for v1.

### 7.4 Relay Portability

Because all user-owned state lives outside the relay:

```
Identity:          On-chain record         →  user controls
Public keys:       On-chain record         →  user controls rotation
Relay endpoint:    On-chain record         →  points to current relay, updatable
Messages:          Storage backend         →  survives relay change (if external)
Private key:       User's choice           →  see Section 9
```

Switching relays requires:

1. Start a new relay node pointed at the same (or new) storage backend
2. Publish a new on-chain record with the updated relay endpoint
3. New messages route to the new relay immediately after propagation

If storage is external, this is a zero-downtime, zero-data-loss operation. The relay is as disposable as the protocol promises.

### 7.5 Deployment Models

```
Self-hosted:     User runs their own relay node (single container).
                 Full control over all layers. Maximum sovereignty.
                 Designed to be a 15-minute setup.

Third-party:     User delegates to a relay service.
                 The relay operator handles uptime, networking, and 
                 optionally provides storage.
                 May charge a fee. Trust bounded by Section 10.

Client-only:     Not possible for receiving messages — a reachable
                 relay endpoint is required for inbound delivery.
                 Equivalent to email's requirement for a reachable 
                 MX server. This is a protocol property, not a limitation.
```

---

## 8. Message Flow

### 8.1 Sending a Message

```
Alice (sender) → Bob (recipient)

Alice's client:
  1.  Resolve Bob's identity string to on-chain record
      → { encryption_key, signing_key, relay_endpoint, identity_type }

  2.  If Bob's identity_type = "domain-verified":
      → DNS verification (Section 6.2). Abort on failure.
      If Alice's identity is also domain-verified:
      → Bob's client will verify Alice's domain on receipt.

  3.  Construct plaintext envelope:
      {
        plaintext:   "<message content>",
        sender:      "alice@acme.com",
        recipient:   "bob@corp.com",
        timestamp:   <unix timestamp>,
        signature:   Ed25519_sign(
                       alice_private_signing_key,
                       hash(plaintext + sender + recipient + timestamp)
                     )
      }

  4.  Encrypt envelope with Bob's X25519 public key → blob

  5.  POST blob to Bob's relay_endpoint

  6.  Relay returns delivery acknowledgment
```

### 8.2 Receiving a Message

```
Bob's relay node:
  1.  Receives encrypted blob
  2.  Validates sender identity via on-chain lookup
  3.  Validates blob format and size
  4.  Writes blob to configured storage backend
  5.  Notifies Bob's client of new message

Bob's client:
  1.  Retrieves blob from storage (via relay or direct storage access)
  2.  Decrypts with Bob's X25519 private key → envelope
  3.  Extracts sender address from envelope
  4.  Resolves sender's on-chain record → sender's Ed25519 signing key
  5.  If sender is domain-verified: performs DNS verification
  6.  Verifies signature against plaintext + sender + recipient + timestamp
  7.  Confirms recipient field matches Bob's own address
  8.  ✅ Valid: message is authentic, untampered, and addressed to Bob
      ❌ Invalid: client MUST warn user prominently — never silently discard
```

### 8.3 Why Sign-Then-Encrypt, and Why the Recipient is in the Signature

Sign-then-encrypt is the required order. Signing after encryption would produce a signature over ciphertext rather than content, which does not prove authorship of the plaintext the recipient reads.

The recipient's address is included in the signed payload to prevent a known signer-ambiguity attack: without it, a recipient could decrypt the message and re-encrypt it to a third party, who would see a valid signature from the original sender apparently addressed to them. Including the recipient binds the signature to the intended conversation.

**What this provides:**

- **Authenticity** — message provably came from the stated sender
- **Integrity** — message was not tampered with in transit or at the relay
- **Non-repudiation** — sender cannot credibly deny having sent the message; signature is anchored to their on-chain key (and optionally their DNS-verified domain)
- **Recipient binding** — signature is invalid if the message is forwarded to an unintended recipient

---

## 9. Key Management

### 9.1 Key Storage Options

Private key custody is the user's most consequential security decision. The protocol supports multiple storage options, each an explicit point on the security/convenience spectrum:

**Option A — Relay-local (encrypted)**
The private key is stored on the relay node, encrypted with the user's passphrase. The relay never sees the plaintext key if client-side encryption is implemented correctly. Changing relays requires exporting and re-uploading the encrypted key blob. Simplest option; appropriate for convenience-tier deployments.

**Option B — User-owned external storage (encrypted)**
The private key is stored in a user-controlled S3 bucket, NAS, or other external storage, encrypted with the user's passphrase. The relay is given read access to fetch the key blob at login. Changing relays requires only pointing the new relay at the same storage — no key re-upload. This fully decouples key custody from relay dependency and is the **recommended default**.

**Option C — Downloaded locally**
The private key is downloaded and stored by the user (optionally encrypted with a passphrase). Maximum sovereignty. Loss of the key file means permanent loss of access to encrypted messages. Users must explicitly acknowledge this at setup.

### 9.2 Encryption Standard

All stored key blobs MUST use:

```
Argon2id(passphrase, salt, params) → derived_key → AES-256-GCM(private_key)
```

Recommended Argon2id parameters (per OWASP guidelines):

```
Memory:       64 MiB minimum (higher is better)
Iterations:   3 minimum
Parallelism:  1 (for portability) or 4 (for server-side)
Salt:         16 bytes, cryptographically random
```

Passphrase strength enforcement (zxcvbn score ≥ 3 or equivalent) is REQUIRED at key generation. A weak passphrase on an exposed encrypted blob is a total compromise. This is non-negotiable.

### 9.3 Forward Secrecy

Static X25519 keys do not provide forward secrecy. If a private key is ever compromised — even years later — all messages ever encrypted to that key are retroactively decryptable by an adversary who has access to the stored ciphertext.

**This is a deliberate v1 simplification.** The practical risk is bounded by the fact that stored ciphertext lives on the user's own storage backend (not on a public network), and an attacker needs both the key and storage access. However, a state-level adversary practicing "collect now, decrypt later" is a real threat for high-security users.

Forward secrecy via ephemeral key exchange (e.g., a Signal-style double ratchet or MLS) is planned as a v2 extension for protocol-to-protocol sessions. This adds statefulness and complexity that conflicts with v1's goal of simplicity.

The whitepaper is explicit about this tradeoff rather than obscuring it.

---

## 10. Threat Model

### 10.1 Layer Isolation

| Attack Vector          | Layer 0 (Chain)           | Layer 1 (DNS)                   | Relay Node        | Storage           | Client                    |
| ---------------------- | ------------------------- | ------------------------------- | ----------------- | ----------------- | ------------------------- |
| Blockchain 51% attack  | ⚠ Key records at risk     | ✅ DNS intact                    | ✅ Data unreadable | ✅ Unaffected      | ✅ Unaffected              |
| Domain hijacking       | ✅ On-chain records intact | ⚠ DNS-verified identity at risk | ✅ Data unreadable | ✅ Unaffected      | ✅ Unaffected              |
| Relay node breach      | ✅ Intact                  | ✅ Intact                        | ✅ Ciphertext only | ✅ Ciphertext only | ✅ Unaffected              |
| Storage backend breach | ✅ Intact                  | ✅ Intact                        | ✅ Intact          | ✅ Ciphertext only | ✅ Unaffected              |
| Rogue relay node       | ✅ Intact                  | ✅ Intact                        | ⚠ See 10.2        | ✅ Intact          | ⚠ See 10.2                |
| Key blob theft         | ✅ Intact                  | ✅ Intact                        | ✅ Intact          | ✅ Intact          | ⚠ Passphrase is last line |
| RPC provider attack    | ⚠ False key returned      | ✅ DNS unaffected                | ✅ Unaffected      | ✅ Unaffected      | ⚠ Misdirected encryption  |

### 10.2 Rogue Relay Node

A rogue relay node can serve malicious client code (if serving a web client) that exfiltrates the user's passphrase before key decryption. Combined with the encrypted key blob (Option A), this gives the relay full access to all messages.

**This is not fully solvable at the protocol level.** It is a fundamental property of any system where infrastructure serves client code — Signal, ProtonMail, and all web-based E2E systems share this weakness. The protocol is explicit about it.

**Structural mitigations:**

1. **Relay node registry (on-chain)** — clients only connect to listed relays by default; unlisted relays require explicit user acknowledgment
2. **Open source clients** — client code is publicly auditable; malicious changes are detectable
3. **Reproducible builds** — published binaries can be verified against source
4. **Option B/C key storage** — relay never holds the key blob; compromise requires accessing the user's external storage independently
5. **Native clients over web** — native clients eliminate the code injection vector entirely
6. **Self-hosting** — a self-hosted relay is trusted by definition; the operator is the user

### 10.3 Security Tiers

```
🟢  Maximum security:
    Self-hosted relay node + external storage
    + Native client (verified reproducible build)
    + Option B or C key storage
    + Direct on-chain reads (relay runs own node or client uses light client)
    Relay is a pure proxy. Cannot read messages, steal keys,
    or falsify identity lookups.

🟡  Standard security:
    Listed third-party relay + native/open-source client
    + Option B key storage
    Attack requires relay to push a malicious client update.
    Detectable by the open source community.

🟠  Convenience tier:
    Listed third-party relay + web client + Option A key storage
    User explicitly trusts the relay operator.
    Mitigated by registry accountability and open source requirement.

🔴  Unverified:
    Unlisted relay + any configuration
    User has explicitly acknowledged the risk.
    No protocol guarantees beyond blob encryption.
```

The client UX MUST display the current security tier visibly and persistently. Users must never be in a lower tier without knowing it.

---

## 11. Email Bridge

### 11.1 Purpose

To address the cold-start problem, the protocol includes an optional email bridge layer. This allows protocol users to reach any standard email address on day one.

### 11.2 How It Works

```
Sender (protocol user)
  →  Encrypts message with a one-time session key
  →  Stores encrypted blob at bridge relay
  →  Sends notification email via SMTP:

From: bridge-noreply@bridgeprovider.com
Subject: Secure message from alice@acme.com

alice@acme.com sent you a secure message.
This message is encrypted and cannot be read by any mail server.

[Read Message]    →  one-time link to view content
[Reply Securely]  →  register to reply with full protocol encryption

---
Sent via Veil. This email contains no message content.
```

The SMTP envelope sender is the bridge provider's own domain — not the sender's domain — to avoid SPF/DMARC failures. The sender's identity is displayed in the message body, not the email headers.

### 11.3 Security Properties

- SMTP carries **zero message content** — only a notification and an access pointer
- Content is stored encrypted; the link provides access to one-time decryption
- **This is explicitly a low-security tier.** The access link is susceptible to interception via browser history, corporate proxies, email provider link scanning, and URL logging. It is appropriate for general correspondence, not sensitive content.
- The UI must clearly label bridge messages as bridge-tier, distinct from protocol-to-protocol messages

### 11.4 Adoption Mechanics

Every notification email is an implicit invitation. The friction of link-based replies drives registration interest. No active marketing required — the security UX does the selling.

### 11.5 Architectural Separation

The email bridge is a **separate optional module**, not part of the core protocol. It can be operated by any relay as an add-on service, run by independent third parties, or deprecated without affecting the core. The core protocol has zero SMTP dependency.

---

## 12. Prior Art and Differentiation

| System          | Permissionless Platform     | Domain Identity                          | Encrypted by Default      | Disposable Infrastructure   | Email Bridge |
| --------------- | --------------------------- | ---------------------------------------- | ------------------------- | --------------------------- | ------------ |
| Email (SMTP)    | ❌ (requires server)         | ✅ (MX records)                           | ❌ (plaintext default)     | ❌ (server is stateful)      | —            |
| DANE / RFC 7929 | ❌ (bolt-on to SMTP)         | ✅                                        | ❌                         | ❌                           | —            |
| Signal          | ❌ (centralized)             | ❌ (phone number)                         | ✅                         | ❌ (Signal operates servers) | ❌            |
| Matrix          | ❌ (federated, server-bound) | Partial (domain in ID, but server-bound) | Partial (opt-in per room) | ❌ (homeserver is stateful)  | ❌            |
| XMTP            | ✅                           | ❌ (wallet-based)                         | ✅                         | Partial                     | ❌            |
| Session         | ✅                           | ❌ (key-based)                            | ✅                         | Partial                     | ❌            |
| **Veil**        | ✅                           | ✅ (optional overlay)                     | ✅                         | ✅                           | ✅            |

**Matrix** is the closest prior art in spirit. It is federated, supports domain-based identity (`@user:domain.com`), and can be self-hosted. The key differences: Matrix homeservers are stateful and own user identity, room state, and history. Switching homeservers means abandoning your identity. Matrix federation replicates messages across all participating servers, expanding the attack surface. Veil keeps messages on user-controlled storage and makes the relay node a stateless proxy.

**XMTP** is the closest prior art in architecture. It is permissionless, blockchain-native, and end-to-end encrypted. The key difference: XMTP's identity model is wallet-first, with no path to domain-based organizational identity. Veil's DNS overlay is specifically designed to bridge the gap between crypto-native infrastructure and business adoption.

The specific combination — permissionless blockchain control plane, optional DNS identity overlay for businesses, and relay nodes as disposable storage-agnostic proxies — does not exist in any prior system.

---

## 13. Open Questions

Known unknowns requiring further design work:

- **Blockchain selection** — Criteria: low write cost, high availability, no single controlling entity, mature tooling, production-ready light clients, long-term viability. Ethereum L2s (Arbitrum, Base, Optimism), Solana, and purpose-built chains are candidates. Requires dedicated analysis.
- **On-chain read verification** — Light client maturity on L2s varies. The practical timeline for trustless client-side verification needs assessment. Until then, relay-hosted reads are the recommended default.
- **Bare alias namespace governance** — How are human-readable aliases (without domains) allocated? First-come-first-served invites squatting. A registration fee, a minimum length, or an auction model are options. Needs formal design.
- **Group messaging** — Not addressed in this draft. Symmetric key distribution for groups is non-trivial. MLS (RFC 9420) is the most mature prior art and a natural fit for a v2 extension.
- **Forward secrecy** — Static X25519 keys are a deliberate v1 simplification. A Signal-style double ratchet or MLS-based session could be layered on for protocol-to-protocol sessions in v2.
- **Metadata obfuscation** — Sealed sender, message padding, and dummy traffic are possible extensions. Out of scope for v1.
- **Spam at scale** — Domain ownership raises the cost of address spoofing but not of bulk sending from legitimate domains. Encrypted content makes content-based filtering impossible. A reputation, stake-based, or rate-limiting model at the relay layer needs design.
- **Message size limits and chunking** — The protocol must define maximum blob sizes for relay interoperability and a chunking mechanism for large payloads (attachments, media).
- **Message export and relay migration** — When storage is local (not external), a standardized export format for message portability needs design.
- **Governance** — Who maintains the relay node registry and sets listing criteria? A DAO, a foundation, or a multi-sig committee? Needs formal design.

---

## 14. Non-Goals

- **Replacing SMTP** — The email bridge is a transition tool, not a permanent compatibility layer.
- **Full anonymity** — Veil provides message privacy and key sovereignty but not sender/recipient anonymity. On-chain identity is pseudonymous; DNS identity is explicitly non-anonymous. Anonymity layers (Tor, mixnets) can be used by participants but are outside protocol scope.
- **Blockchain as message storage** — Messages are never stored on-chain. The blockchain is a control plane only.
- **Metadata privacy (v1)** — Timing, blob size, and activity patterns are visible to the relay node. Full metadata privacy is a future extension.
- **Forward secrecy (v1)** — Static keys are a deliberate simplification. See Section 9.3.
- **Mandating a specific storage backend** — The protocol defines a storage interface, not a storage implementation. Local disk is as valid as S3.
- **Patenting** — Veil is intended to be open and unencumbered. This document constitutes prior art against future patent claims on the described architecture.

---

*This document is a starting point. All design decisions are open for revision. Feedback welcome.*
