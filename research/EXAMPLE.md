# End-to-end example: parallel filing agents

A concrete v0 demo you can build and run. Spec: [RESEARCH.md](RESEARCH.md), [AGENTIC.md](AGENTIC.md), [pseudo/v0](../pseudo/v0/README.md).

---

## Use case

**Lumen** sells a metered HTTP API, `summarize-filing`: given a filing URL, it returns a structured summary (parties, amounts, risk flags). Price is **$0.01 USDC per call**.

**Northwind Capital** runs **10 overnight research agents**. Each agent owns a batch of filings and must call `summarize-filing` many times. Agents are untrusted (prompt injection, loops). Northwind will not put a funded wallet in the agent runtime.

They prepay with unidirectional call channels: one **nonce-isolated channel per agent**, one **spend-broker sidecar** per agent. The 10 agents run at the same time because they do not share a counter.

```
Lumen (provider)                         Northwind (payer)
  org + service on-chain                   funding key (human)
  summarize-filing API                     payer key / vault
  receipt signer                           10 channels (one nonce each)
  claim watcher                              │
                                             ├── agent-1 + sidecar broker-1
                                             ├── agent-2 + sidecar broker-2
                                             └── … agent-10 + sidecar broker-10
```

A first demo can use **1 agent / 1 channel** and still exercise every instruction. Ten channels only prove parallelism.

---

## Actors

| Actor | Who in the story | On-chain role |
|---|---|---|
| Lumen ops | Provider business | Organization owner |
| Lumen API owner | Same team or a member | Service owner, claim recipient |
| Northwind treasury | Human with 2FA / hardware | Channel owner (payer); never the agent |
| Research agent | Untrusted worker | `execute` only; no keys, no vouchers |

---

## Tools and services to build

Shared libraries first. Then one process per box below. Names are illustrative.

### Shared (both sides)

| Tool | What it is | Why |
|---|---|---|
| **Chain + program** | Port of `pseudo/v0` (e.g. Solana test validator + Anchor program, or an equivalent EVM deploy) | Escrow, open/fund/claim/close, pause |
| **Codec crate/lib** | Canonical `DOMAIN`, id hashes, `voucher_digest`, `receipt_digest`, raw-digest verify | Same bytes on chain, broker, and provider ([02-ids.md](../pseudo/v0/02-ids.md)) |
| **Test asset** | USDC-like mint or native test token matching `service.asset` | Explicit asset; no silent SOL/ETH mix-ups |
| **Clock** | Slot or unix, documented in the port README | `expiration`, `CLOSE_CHALLENGE_PERIOD`, subscription vest |

### Organization / service (Lumen)

| Tool | What it is | Why |
|---|---|---|
| **Provider admin CLI** (or thin UI) | `create_organization`, `create_service`, `update_service`, `set_*_paused`, `delete_*` | Humans publish terms; pause is the wind-down switch |
| **Receipt-signer key** | Keypair stored in Lumen’s KMS; pubkey is `Service.receipt_signer` | Brokers accept only this key’s receipts |
| **Native API (`summarize-filing`)** | HTTP service that implements [07-offchain.md](../pseudo/v0/07-offchain.md) | Verify voucher, serve **once**, persist `{ result, receipt }`, sequential `n` |
| **Idempotency store** | DB/KV keyed by `(channel_id, n)` (metered) | Retries return the same record; upstream runs once |
| **Claim watcher** | Indexes `ChannelCreated` / `ChannelTransitionRequested`; submits `claim_channel_funds` | Batched settlement; challenge deadline with a safety margin |
| **Discovery blob** | `Organization.metadata` / `Service.metadata` (endpoint, schema, model id) | Advertisement only; never authority |

Lumen does **not** need a spend broker. They need the program, the API, the receipt key, and a watcher.

For a later demo against a dumb upstream (OpenAI, Stripe, internal HTTP), add a **metering gateway** as the on-chain service ([AGENTIC.md](AGENTIC.md) § gateway). Skip it for this native example.

### Payer (Northwind treasury)

| Tool | What it is | Why |
|---|---|---|
| **Funding wallet** | Human wallet that holds idle USDC | Never given to agents or to the payer key’s daily host |
| **Payer account** | Dedicated spend account; channel owner | Least privilege; even a leak cannot drain the main treasury |
| **Payer admin CLI** | `open_channel`, `fund_channel`, `request/cancel/finalize` transition, `claim` (rare) | Agents must not call these |
| **Vault** | Durable store: capabilities, encrypted ladders or KMS policy, audit | Source of truth for `dispensed` / `acked` policy; shard by `channel_id` |
| **KMS or airgap signer** | Hot `sign_voucher` **or** offline pre-signed ladder + wrapping keys | Signs only canonical voucher digests; no arbitrary `transfer` |
| **Capability issuer** | Writes `Capability { agent_id, channel_id, pin, max_counter, window }` | Allowlist + windows; revoke without a chain tx |
| **Payer watcher** (optional) | Watches expiry, version mismatch, pause | Prompts a human to fund, AdoptTerms, or Close |

One vault/KMS cluster is enough for ten agents if state is keyed by channel. It must not take a global lock across channels.

### Agents (Northwind research workers)

| Tool | What it is | Why |
|---|---|---|
| **Spend-broker sidecar** | Local process next to the agent; `execute` API only | Attaches the next voucher, calls Lumen, verifies receipt, never returns the signature |
| **Workload identity** | Short-lived JWT / SPIFFE / vault AppRole on `execute` | Agent role cannot open channels or export keys |
| **Thin agent client** | Few dozen lines: `broker.execute(capability_id, channel_id, idempotency_key, request)` | No voucher/signature in process or logs |
| **Agent runtime** | Script, LangGraph, Cursor agent, etc. | Untrusted worker; only sees summaries |

Ten sidecars + ten channels ⇒ ten concurrent calls into Lumen. Two sidecars on **one** channel would serialize (`lookahead = 1`).

---

## Topology (ten agents)

```
                    ┌──────────── vault / KMS (shared, sharded by channel) ────────────┐
                    │  ladders or sign_voucher(channel_id, version, n)                   │
                    └──────────┬──────────┬──────────┬───────────────────────────────────┘
                               │          │          │
                         sidecar-1   sidecar-2  … sidecar-10
                               │          │          │
                            agent-1    agent-2     agent-10
                               │          │          │
                               └──────────┴──────────┴── execute only
                                              │
                                              ▼
                                   Lumen summarize-filing
                                              │
                                    persist result+receipt
                                              │
                                              ▼
                                      claim watcher ──► program.claim_channel_funds
```

---

## Happy-path walkthrough (one night)

Numbers are for the story, not protocol constants.

1. **Lumen** deploys the program, creates org `lumen`, creates service `summarize-filing`:
   - `billing = Metered`, `asset = USDC`, `price = 0.01`, `minimum_calls = 100`, `expiration_threshold` = a few days.
   - Sets `receipt_signer` to the API’s signing key.
   - Publishes the HTTPS endpoint in metadata.

2. **Northwind treasury** creates a dedicated payer account, funds it with USDC, and calls `open_channel` **ten times** (protocol-assigned nonces `0…9`). Each channel: `calls = 2_000` → **$20** locked per agent, **$200** total.

3. **Treasury** airgap-signs (or KMS-signs) ladders `counter = 1 … 2_000` per channel. Encrypted cold store keeps the tail. Each sidecar gets a small hot tranche (for example 50 rungs).

4. **Treasury** issues ten capabilities: agent *k* may `execute` only on channel *k*, pin `(version, asset, billing, receipt_signer)`, `max_counter = 2_000`, a per-hour window.

5. **Agents run.** Each sidecar:
   - accepts `execute`;
   - requires `Open` and pin match;
   - sets `n = dispensed + 1` only if `dispensed == acked`;
   - POSTs to Lumen with the request + `sig(n)` (agent never sees `sig`);
   - verifies the receipt (`request_hash`, `result_hash`, `receipt_signer`);
   - persists ack and returns only the summary.

6. **Lumen** accepts only `n == last_accepted + 1` for that channel, serves once, stores the record. The claim watcher periodically submits the highest voucher per channel (for example every 100 calls, or on `Closing`).

7. **Morning.** Treasury reads four numbers per agent: locked, dispensed, acked, settled. Unused escrow stays until a challenged Close (or the channel exhausts and cleans up). Revoke kills a bad agent in seconds without waiting for expiry.

Parallelism: agent-3 on `n = 40` does not block agent-7 on `n = 12`. Those are different `channel_id`s.

---

## Minimum slice (first runnable demo)

Build this before pause, AdoptTerms, or subscriptions:

| Order | Piece |
|---|---|
| 1 | Program + codec + local validator + test mint |
| 2 | Provider admin CLI: org + metered service |
| 3 | Native API: sequential voucher + receipt + idempotency store |
| 4 | Payer CLI: `open_channel` (one nonce) |
| 5 | Vault: one capability + small ladder or hot KMS |
| 6 | One sidecar + one scripted agent making **N** `execute` calls |
| 7 | Claim watcher (or a manual `claim_channel_funds`) |
| 8 | Print locked / dispensed / acked / settled |

Then add: second nonce (parallelism), `fund_channel`, challenged close, pause, version bump + AdoptTerms, subscription pass if you want a monthly SKU.

---

## What this example is not

- Agents calling Lumen with a wallet or a `get_vouchers` API.
- One shared channel for ten agents (that serializes and mixes budgets).
- A single global spend broker for every payer on the chain. Northwind’s vault is **their** till; Lumen does not operate it.
- Proof that Lumen’s summary is correct. Receipts attest bytes; v0 assumes an honest native provider.

When this demo runs on a testnet, the deployment README should still state that trust mode: honest native provider.