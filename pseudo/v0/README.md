# Prepaid Call Channels — v0 pseudo

Metered APIs and AI agents need to pay **per call** without a chain transaction per request, and without handing the agent a funded wallet.

This spec is a **prepaid unidirectional payment channel**: a human locks `price × calls` against a priced service; the payer (or a vault on their behalf) signs cumulative vouchers off-chain; the provider settles in one on-chain claim. Channel `version` snapshots asset, price, and receipt key so later service changes cannot silently re-rate old IOUs. `counter` only increases: same-term top-ups raise `calls`; terms changes keep the counter and bind new vouchers to the new version.

Agents never hold spend keys **or bearer vouchers**. A human-managed spend broker attaches the next voucher, accepts only `{ result, receipt }`, and returns the result after receipt verification. The on-chain lock is the hard cap; broker/vault policy is the fine-grained budget.

This directory is the chain-agnostic v0 program + off-chain protocol. Translate each file into the target language (e.g. Anchor/Rust). Do not invent extra instructions. How to slice the work for review: [ROADMAP.md](ROADMAP.md). Background: [RESEARCH.md](../../research/RESEARCH.md), [AGENTIC.md](../../research/AGENTIC.md).

Time unit (`now()`), token mint, and account addresses are target-defined. Protocol is not.

```
Organization
  └── Service          priced, versioned
        └── Channel    nonce-isolated (usually one per agent); prepaid quota
```

Funds live in an isolated program-owned escrow per channel/asset. Claims transfer out. Same-term `fund_channel` adds escrow immediately. Close / AdoptTerms refunds only after an on-chain challenge window.

---

## Files

| File | Translates to | Contents |
|---|---|---|
| [`01-types.md`](01-types.md) | `state` + `error` + events | records, config, errors, events |
| [`02-ids.md`](02-ids.md) | `ids` / crypto helpers | domain, id hashes, voucher + **receipt** digests, sig verify |
| [`03-organization.md`](03-organization.md) | org instructions | create / pause / delete + members |
| [`04-service.md`](04-service.md) | service instructions | create / update / pause / delete (`version++` on update only) |
| [`05-channel.md`](05-channel.md) | channel instructions | open, fund, request/cancel/finalize transition |
| [`06-claim.md`](06-claim.md) | claim instruction | exhausted cleanup / voucher settle |
| [`07-offchain.md`](07-offchain.md) | spend broker / provider (not the program) | sequential consume, receipts, retries, watchers |

Helpers are defined once (`02`, `remaining` in `05`) and referenced, never copied.

---

## Instructions

| Instruction | Caller | File |
|---|---|---|
| `create_organization` | future owner | 03 |
| `set_organization_paused` | owner | 03 |
| `delete_organization` | owner | 03 |
| `create_service` | org member | 04 |
| `update_service` | service owner | 04 |
| `set_service_paused` | service owner | 04 |
| `delete_service` | service owner | 04 |
| `open_channel` | payer | 05 |
| `fund_channel` | payer | 05 |
| `request_channel_transition` | payer | 05 |
| `cancel_channel_transition` | payer | 05 |
| `finalize_channel_transition` | anyone | 05 |
| `claim_channel_funds` | anyone | 06 |

Deletes are rejected while children exist. Close channels via a challenged transition first. Pause the org or service to stop new admission while children drain.

---

## Invariants

1. Channel id is `(domain, payer, organization, service, nonce)`; `NextChannelNonce[payer]` only increases.
2. Each channel/asset escrow equals `price × (calls − counter)` for the active snapshot.
3. `0 ≤ counter ≤ calls`; counter only increases (or stays) with a valid payer voucher; it never resets.
4. Claims use `channel.price`, never live `service.price`.
5. Vouchers bind to channel `version` and the deployment domain.
6. Refund / AdoptTerms only after `CLOSE_CHALLENGE_PERIOD`; AdoptTerms is pre-funded at request and old-version claims stay valid until finalization. Same-term `fund_channel` has no challenge.
7. Claim payout to `service.owner`, independent of tx sender.
8. `update_service` always increments `version`.
9. `delete_service` iff `channels == 0`; `delete_organization` iff `services == 0`.
10. Asset is explicit/immutable per service; all arithmetic is checked and all state/transfers are atomic.
11. `fund_channel` requires matching live `service.version` and raises `calls`. Finalized `AdoptTerms` requires a version change, sets `calls = counter + new_quota`, and does not reset `counter`.
12. `org.paused` or `svc.paused` blocks `create_service`, `open_channel`, and `AdoptTerms` only. Pause does not bump `version` or seize escrow.

v0 vs the original source: deployment-scoped canonical digests; nonce-isolated channels; version-bound vouchers; monotonic counter; per-channel escrow; unchallenged same-term fund; challenged close/AdoptTerms; checked arithmetic and strict counter bounds; result-bound receipts; spend broker; deletes refuse children; member cap enforced; `trials` stored but not billed.

---

## Out of v0 (target port)

Account/PDA mechanics, concrete signature scheme, time source, and event encoding are adapter choices. They must not change instruction semantics or canonical digest bytes.

v1 ideas (member lifecycle, `trials` billing, optional disputes/TEEs/verifiable services) live in `research/RESEARCH.md` — do not invent them in this port.
