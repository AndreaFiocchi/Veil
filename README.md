# Veil

**A decentralized, end-to-end encrypted messaging protocol with domain-based identity.**

> ⚠️ **Status: Ideation / working draft.** This is a whitepaper, not a working implementation. There is no active development timeline.

---

## What is Veil?

Veil is a protocol design for permissionless messaging that doesn't require a platform owner, a foundation, or a consortium to function. It combines three ideas:

1. **Blockchain as a control plane** — identity records, public keys, and relay endpoints live on-chain. Messages never do.
2. **DNS as an optional identity overlay** — businesses can anchor `user@domain.com` addressing to the protocol without sacrificing its permissionless nature.
3. **Relay nodes as disposable proxies** — infrastructure that holds nothing the user can't replace in minutes. Storage-agnostic, stateless by design.

The result is a system where identity belongs to the user, encryption is structural (not opt-in), and no single entity controls the registry, the routing, or the keys. And never will.
No law can mandate censorship or force the disclosure of comunications.

## Read the Whitepaper

📄 **[Whitepaper.md](Whitepaper.md)**

## Key Design Properties

- **Permissionless** — anyone can register an identity and send messages without asking permission
- **Encrypted at rest, always** — relay nodes only ever see ciphertext
- **Domain identity for businesses** — optional DNS overlay for organizational address spaces
- **Disposable infrastructure** — swap relay nodes, rotate keys, change storage backends — identity survives
- **Email interop on day one** — an optional bridge layer reaches legacy SMTP users without exposing message content
- **Honest threat model** — explicit security tiers, no overclaiming

## Project Structure

```
.
├── README.md
└── Whitepaper.md    # Protocol spec (working draft)
```

## Contributing

This is an early-stage design. If you find flaws, have questions, or want to push back on any of the architectural decisions, [open an issue](../../issues). The whitepaper's Section 13 lists known open questions — contributions to any of those are especially welcome.

## License

### Veil is intended to be open and unencumbered. This document constitutes prior art against future patent claims on the described architecture.

- **Whitepaper:** [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) — public domain, no restrictions
- **Reference implementation (future):** [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0)
