# Agentic payment channels

Prepaid, unidirectional **call channels** so metered APIs and AI agents can pay per request without a blockchain transaction each time — and without giving the agent a funded wallet.

A human locks `price × calls` of an explicit asset in a nonce-isolated channel. Off-chain, a spend broker uses cumulative, version-bound vouchers. The provider settles many calls in one claim. Same-term top-ups raise the ceiling without resetting `counter`. Challenged close / AdoptTerms gives the provider time to settle before any refund or re-price.

Agents never hold spend keys or bearer vouchers. A human-managed spend broker attaches the next voucher, verifies a provider receipt bound to request/result hashes, persists it, then returns the result. The on-chain escrow is the hard cap; broker policy is the budget.

The protocol is **chain-agnostic**. Port it by translating `pseudo/v0` (Solana/Anchor, or any other runtime). Do not add v1 features in the first port.

```
Organization
  └── Service          priced, versioned
        └── Channel    nonce-isolated prepaid quota, monotonic counter
```

## Start here

| Doc | What it is |
|---|---|
| [research/RESEARCH.md](research/RESEARCH.md) | On-chain model, claims, versioning, invariants |
| [research/AGENTIC.md](research/AGENTIC.md) | Agents, KMS/vault, pre-signed ladder, receipts |
| [pseudo/v0/README.md](pseudo/v0/README.md) | v0 spec to implement (types, instructions, off-chain) |
| [pseudo/v0/ROADMAP.md](pseudo/v0/ROADMAP.md) | Small PRs, adapter-per-chain, review-friendly order |
