# Prepaid Call Channels — v0 pseudo

Metered APIs and AI agents need to pay **per call** without a chain transaction per request, and without handing the agent a funded wallet.

This spec is a **prepaid unidirectional payment channel**: a human locks `price × calls` against a priced service; the payer (or a vault on their behalf) signs cumulative vouchers off-chain; the provider settles in one on-chain claim. Open channels snapshot price and version so a later service update cannot silently re-rate old IOUs.

Agents never hold spend keys. A human-managed vault dispenses the **next** voucher only, and only after the service returns `{ result, receipt }` for the previous call. The on-chain lock is the hard cap; vault policy is the fine-grained budget.

This directory is the chain-agnostic v0 program + off-chain protocol. Translate each file into the target language (e.g. Anchor/Rust). Do not invent extra instructions. How to slice the work for review: [ROADMAP.md](ROADMAP.md). Background: [RESEARCH.md](../../research/RESEARCH.md), [AGENTIC.md](../../research/AGENTIC.md).

Time unit (`now()`), token mint, and account addresses are target-defined. Protocol is not.

```
Organization
  └── Service          priced, versioned
        └── Channel    one per (payer, org, service); prepaid quota
```

Funds live in a single program-owned `VAULT`. Opens/updates transfer in; claims/refunds/closes transfer out.

---

## Files

| File | Translates to | Contents |
|---|---|---|
| [`01-types.md`](01-types.md) | `state` + `error` + events | records, config, errors, events |
| [`02-ids.md`](02-ids.md) | `ids` / crypto helpers | domain, id hashes, voucher + **receipt** digests, sig verify |
| [`03-organization.md`](03-organization.md) | org instructions | create / delete + members |
| [`04-service.md`](04-service.md) | service instructions | create / update / delete (`version++`) |
| [`05-channel.md`](05-channel.md) | channel instructions | `remaining()`, open, update |
| [`06-claim.md`](06-claim.md) | claim instruction | empty / payer refund / voucher settle |
| [`07-offchain.md`](07-offchain.md) | client / provider / vault (not the program) | sequential consume, **receipts**, watchers |

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
| `update_channel` | payer | 05 |
| `claim_channel_funds` | anyone | 06 |

Deletes are **rejected** while children exist (`services == 0` / `channels == 0`). Close channels via `claim_channel_funds` first.

---

## Invariants

1. One channel per `(payer, organization, service)`. Id is `hash_channel_id` (see 02).
2. Escrow for a channel equals `price × (calls − counter)` until claim, refund, or update.
3. On-chain `counter` only increases, and only with a payer signature (path C).
4. Claims use `channel.price`, never live `service.price`.
5. Voucher version must equal **`channel.version`** (snapshot), not live `service.version`.
6. Payer refund only after expiry **or** version mismatch.
7. Claim payout to `service.owner`, independent of tx sender.
8. `update_service` always increments `version`.
9. `delete_service` iff `channels == 0`; `delete_organization` iff `services == 0`.

v0 vs the original source: vouchers bind to `channel.version` (a bump does not burn IOUs); deletes refuse children; `update_channel` **persists** and **resets `counter` to 0**; `MAX_ORG_MEMBERS` is enforced; `trials` is stored, not billed.

---

## Out of v0 (target port)

PDA seeds, mint (SOL vs SPL), `Clock` slot vs unix time, event encoding. Pick them at translation time; do not change the instruction set or the voucher layout.

v1 ideas (members, cooperative close, `fund_channel`, channel seed, `trials` billing, challenge period) live in `research/RESEARCH.md` — do not invent them in this port.
