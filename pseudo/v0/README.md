# Prepaid Call Channels — v0 pseudo

Metered APIs and AI agents need to pay **per call** without a chain transaction per request, and without handing the agent a funded wallet.

This spec is a **prepaid unidirectional payment channel**: a human locks `price × calls` against a priced service; the payer (or a vault on their behalf) signs cumulative vouchers off-chain; the provider settles in one on-chain claim. Channel epochs snapshot asset, price, version, and receipt key so later service changes cannot silently re-rate old IOUs.

Agents never hold spend keys **or bearer vouchers**. A human-managed spend broker attaches the next voucher, accepts only `{ result, receipt }`, and returns the result after receipt verification. The on-chain lock is the hard cap; broker/vault policy is the fine-grained budget.

This directory is the chain-agnostic v0 program + off-chain protocol. Translate each file into the target language (e.g. Anchor/Rust). Do not invent extra instructions. How to slice the work for review: [ROADMAP.md](ROADMAP.md). Background: [RESEARCH.md](../../research/RESEARCH.md), [AGENTIC.md](../../research/AGENTIC.md).

Time unit (`now()`), token mint, and account addresses are target-defined. Protocol is not.

```
Organization
  └── Service          priced, versioned
        └── Channel    nonce-isolated (usually one per agent); prepaid quota
```

Funds live in an isolated program-owned escrow per channel/asset. Claims transfer out; close/rollover refunds only after an on-chain challenge window.

---

## Files

| File | Translates to | Contents |
|---|---|---|
| [`01-types.md`](01-types.md) | `state` + `error` + events | records, config, errors, events |
| [`02-ids.md`](02-ids.md) | `ids` / crypto helpers | domain, id hashes, voucher + **receipt** digests, sig verify |
| [`03-organization.md`](03-organization.md) | org instructions | create / delete + members |
| [`04-service.md`](04-service.md) | service instructions | create / update / delete (`version++`) |
| [`05-channel.md`](05-channel.md) | channel instructions | open, request/cancel/finalize transition |
| [`06-claim.md`](06-claim.md) | claim instruction | exhausted cleanup / voucher settle |
| [`07-offchain.md`](07-offchain.md) | spend broker / provider (not the program) | sequential consume, receipts, retries, watchers |

Helpers are defined once (`02`, `remaining` in `05`) and referenced, never copied.

---

## Instructions

| Instruction | Caller | File |
|---|---|---|
| `create_organization` | future owner | 03 |
| `delete_organization` | owner | 03 |
| `create_service` | org member | 04 |
| `update_service` | service owner | 04 |
| `delete_service` | service owner | 04 |
| `open_channel` | payer | 05 |
| `request_channel_transition` | payer | 05 |
| `cancel_channel_transition` | payer | 05 |
| `finalize_channel_transition` | anyone | 05 |
| `claim_channel_funds` | anyone | 06 |

Deletes are rejected while children exist. Close channels via a challenged transition first.

---

## Invariants

1. Channel id is `(domain, payer, organization, service, nonce)`; `NextChannelNonce[payer]` only increases.
2. Each channel/asset escrow equals `price × (calls − counter)` for the active epoch.
3. `0 ≤ counter ≤ calls`; counter only increases with a valid payer voucher.
4. Claims use `channel.price`, never live `service.price`.
5. Vouchers bind to channel `epoch` + `version` and the deployment domain.
6. Refund/rollover only after `CLOSE_CHALLENGE_PERIOD`; rollover is pre-funded at request and old-epoch claims stay valid until finalization.
7. Claim payout to `service.owner`, independent of tx sender.
8. `update_service` always increments `version`.
9. `delete_service` iff `channels == 0`; `delete_organization` iff `services == 0`.
10. Asset is explicit/immutable per service; all arithmetic is checked and all state/transfers are atomic.
11. Rollover is the only counter reset; it increments epoch.

v0 vs the original source: deployment-scoped canonical digests; nonce-isolated channels; epoch-bound vouchers; per-channel escrow; challenged close/rollover; checked arithmetic and strict counter bounds; result-bound receipts; spend broker; deletes refuse children; member cap enforced; `trials` stored but not billed.

---

## Out of v0 (target port)

Account/PDA mechanics, concrete signature scheme, time source, and event encoding are adapter choices. They must not change instruction semantics or canonical digest bytes.

v1 ideas (member lifecycle, `fund_channel`, `trials` billing, optional disputes/TEEs/verifiable services) live in `research/RESEARCH.md` — do not invent them in this port.
