# Agentic payment channels

Prepaid, unidirectional **call channels** so metered APIs and AI agents can pay per request without a blockchain transaction each time — and without giving the agent a funded wallet.

A human (or a vault they control) locks `price × calls` against a priced service. Off-chain, the payer signs cumulative vouchers as calls happen. The provider settles many calls in one on-chain claim. Channels snapshot price and version so a later service update cannot silently re-rate old IOUs.

Agents never hold spend keys. A human-managed vault only releases the **next** voucher, and only after the service returns `{ result, receipt }` in the same response. The on-chain lock is the hard cap; vault policy is the budget.

The protocol is **chain-agnostic**. Port it by translating `pseudo/v0` (Solana/Anchor, or any other runtime). Do not add v1 features in the first port.

```
Organization
  └── Service          priced, versioned
        └── Channel    one per (payer, org, service); prepaid quota
```

## Start here

| Doc | What it is |
|---|---|
| [research/RESEARCH.md](research/RESEARCH.md) | On-chain model, claims, versioning, invariants |
| [research/AGENTIC.md](research/AGENTIC.md) | Agents, KMS/vault, pre-signed ladder, receipts |
| [pseudo/v0/README.md](pseudo/v0/README.md) | v0 spec to implement (types, instructions, off-chain) |
| [pseudo/v0/ROADMAP.md](pseudo/v0/ROADMAP.md) | Small PRs, adapter-per-chain, review-friendly order |
