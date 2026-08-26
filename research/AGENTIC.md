# Agentic spend: humans, vaults, and call channels

How an AI agent pays for **any** online service using prepaid call channels ([RESEARCH.md](RESEARCH.md)) without holding the money or the keys.

The agent is an untrusted worker. The human is the budget owner. A KMS/vault is the only component that can move value. On-chain channels are the hard ceiling; vault policy is the fine-grained limiter and the audit log.

---

## Problem

Agents will call tools, models, SaaS APIs, and other agents continuously. Asking a human to sign every call is unusable. Giving the agent a funded wallet is unsafe:

- Prompt injection / a compromised tool can drain the wallet.
- A loop can spend faster than a human can notice.
- The same key that signs a $0.001 voucher can usually also `transfer` the rest of the account.
- “Track spend” after the fact is not a limit.

Needed:

1. **Hard caps** the agent cannot raise.
2. **Per-service and per-agent budgets** that can be stricter than the on-chain lock.
3. **Least-privilege signing** (vouchers only, not arbitrary txs).
4. **Instant revoke** without waiting for channel expiry.
5. **A path to arbitrary HTTP APIs**, not only services that speak the channel protocol.

---

## Threat model

| Attacker | Assumption | What must still hold |
|---|---|---|
| Runaway or injected agent | Can call `next_voucher` | Cannot skip receipts; cannot get `sig(n+1)` without an honest provider receipt for `n`; cannot open channels; cannot export keys |
| Malicious / buggy service | Asks for `counter = calls` up front, or a huge jump | Vault refuses; on-chain cap still binds if a signature leaks |
| Compromised agent identity (stolen API key to the vault) | Can request signatures as that agent | Damage ≤ remaining capability **and** remaining channel lock; human can revoke |
| Compromised vault/KMS | Can sign as the payer **or** dump stored signatures | Damage ≤ sum of open channels, and ≤ **highest `sig(n)` sitting in that vault**; keep hot tranches small |
| Malicious provider on-chain | Claims a valid voucher | Only gets `channel.price × Δcounter`; cannot hit live price or other channels |

Out of scope for v0: a fully malicious human owner, and a KMS that also holds the human’s root key with no second factor.

---

## Principle

**The agent never sees a spending private key.** It only receives the next voucher after the vault has a **provider receipt** for the previous call (none for `n = 1`).

Damage is the **min** of:

```
on-chain remaining     = channel.price × (channel.calls − channel.counter)
authorized remaining   = policy max − already signed for that agent/service/window
```

Even a fully compromised agent plus a tricked vault sign endpoint cannot spend more than what the human already locked in channels. The vault exists so they usually spend **less**, and so revoke does not wait for expiry.

```
Human owner
  │  policies, allowlists, top-ups, revoke
  │  optional: airgap-sign a voucher ladder, load vault in tranches
  ▼
Vault (KMS and/or payload store)
  │  next_voucher + submit_receipt     ← agent never talks to the key
  │  open/update/close channel         ← human / high-privilege role
  ▼
Agent (no key material)
  │  HTTP + voucher  /  receipt back to vault
  ▼
Service or metering gateway → chain (claim)
```

---

## Four numbers (track all four)

Do not treat “what the agent spent” as a single figure.

| Number | Where | Meaning |
|---|---|---|
| **Locked** | Chain: `price × calls` | Hard budget parked in escrow for that service |
| **Authorized** | Vault: highest **dispensed** `counter` | Voucher the agent has been given |
| **Acked** | Vault: highest **receipt** `counter` | Calls the provider attested |
| **Settled** | Chain: `channel.counter` | What the provider has cashed |

```
settled ≤ dispensed ≤ calls
acked ≤ dispensed
acked == dispensed  or  dispensed == acked + 1    // at most one outstanding voucher
spend_authorized = price × dispensed
spend_acked      = price × acked
unsettled IOU    = price × (dispensed − settled)
unused_lock      = price × (calls − dispensed)
```

The vault’s ledger is source of truth for **limits**. The chain is source of truth for **settlement**. Reconcile them on a timer (provider may lag on claims).

---

## Layered limits

### 1. On-chain envelope (hard, coarse)

Opening a channel **is** the human committing money to one service. Keep `calls` small (hours of expected use, not months). The agent cannot raise `calls`; that is `open_channel` / `update_channel`, vault-gated to the human.

This is the last backstop if policy fails. Sum of all open channels = maximum blast radius of a vault bug.

### 2. Vault policy (hard for the agent, fine-grained)

Evaluated **inside** the vault on every `next_voucher`. The agent cannot skip this; it has no other source of signatures.

Suggested fields on a **capability** (issued by the human, stored in the vault, not on-chain):

```
Capability {
  agent_id:           Bytes
  payer:              AccountId
  allowlist:          [ { org_id, service_id, channel_id, receipt_signer } ]
  max_counter:        u32                // ≤ channel.calls; per channel
  max_amount:         Balance            // optional cap across services
  window:             { period, max_amount, max_calls }
  valid_until:        Time
  ops:                { next_voucher, submit_receipt }
}
```

`receipt_signer` is the provider key that must sign receipts (native: `service.owner`; gateway: gateway key). Pin it; do not take it from the agent.

Checks on `next_voucher` — vault picks `n = dispensed + 1` ([pseudo/v0/07-offchain.md](../pseudo/v0/07-offchain.md)):

1. Capability not expired / not revoked.
2. `channel_id` in allowlist.
3. If `dispensed > acked`: valid **provider receipt** for `counter == dispensed` (skip when `dispensed == 0`).
4. `n ≤ max_counter` and `n ≤ channel.calls`.
5. Window still has room.
6. Live `channel.version` matches the capability pin (bump → freeze).

On success: return one pre-signed `sig(n)` (or hot-sign it), set `dispensed = n`. Receipts do not go on-chain.

### 3. Human gates (rare, high impact)

Vault roles:

| Role | Auth | Allowed |
|---|---|---|
| **Root / owner** | Hardware key, 2FA | Create payer keys, set policies, open/update/close channels, revoke, export never |
| **Approver** | Same or a second human | Optional: signatures with `Δamount` above a threshold |
| **Agent** | Workload identity (short-lived JWT / SPIFFE / vault AppRole) | `next_voucher`, `submit_receipt` |

Do **not** put `open_channel` on the agent role. Funding is a conscious human act.

---

## Least-privilege key

The payer key in the KMS must be **unable** to do anything except channel vouchers (and, under the human role, the eight channel instructions).

If that key can `transfer` or sign arbitrary bytes, policy is theater.

Practical shapes:

- **Message-policy KMS** (Turnkey, HashiCorp Vault Transit + wrap, cloud KMS with a custom signer): allowlist digest prefix `DOMAIN`, payload layout `(channel_id, version, counter)` only.
- **Dedicated spend account**: zero extra balance; the only value it controls is already inside channels. Even a leaked key cannot steal funds that were never locked.
- **Split roles**: `funding_key` (human, holds idle capital) vs `payer_key` (vault, only used as channel owner). Human transfers into the payer just enough to open the next small channel.

Prefer a dedicated spend account **and** message policy. Then a leak of `payer_key` still cannot empty the human’s main wallet, and cannot sign a voucher the policy would have rejected *if the leak is of the agent API rather than the KMS itself*. A raw leak of `payer_key` bypasses policy — hence keep that material in HSM/KMS, never in the agent runtime, never in logs.

---

## Pre-signed ladder (store signatures, not keys)

Hot signing keeps the payer key **online** in a KMS. The alternative: the human signs offline, stores **only the signatures** in the vault, and never gives the vault (or the agent) the key.

Example: open a $100 channel at $0.01/call → `calls = 10_000`. Human signs 10_000 vouchers (`counter = 1 … 10_000`) on an air-gapped machine, then loads them into the vault. Policy might release the first 100 on day 1, the next 100 on day 2, and so on. The agent only **fetches**; it never triggers a signing operation.

That works. Two properties of this protocol change how you should store and release them.

### Cumulative IOUs, not tickets

Vouchers replace each other. `sig(100)` pays `price × 100` by itself. `sig(1) … sig(99)` are worthless once `sig(100)` exists.

So “release the first 100 signatures on day 1” is economically **one** permission: raise authorized counter to 100 ($1). If the agent (or a thief) can read all 100 rows, they only need the last one. The vault should treat the **highest released counter** as the spendable amount, and can even store/release **only** that rung for the window.

You do not need 10_000 hot rows for a 100/day **budget**. That budget is a vault window on how many times `next_voucher` may succeed, not a reason to hand the agent `sig(100)`.

| Goal | What to pre-sign | What to dispense |
|---|---|---|
| Sequential calls (default) | `1, 2, 3, …, 10_000` | Only `sig(dispensed + 1)`, after receipt for `n − 1` |
| Coarse *budget* (not sequential) | `100, 200, …` | One rung = one day’s max claim — **do not use for agents** that must call +1 |

Agent mode is the first row. See [Forced sequential fetch](#forced-sequential-fetch).

### The vault still holds cash

A stored `sig(n)` is a **bearer instrument**. Anyone who has it can hand it to the provider; claim is anyone-can-submit. If the hot vault contains `sig(10_000)`, a vault dump **is** the $100 channel — same blast radius as leaking the key for channel-spend purposes.

Keys-not-in-vault only reduces risk if the vault **never holds the top of the ladder**:

```
cold / airgap (human)     sig(501) … sig(10_000)     key lives here only while signing
hot vault                 sig(up to today's cap)     e.g. next 100 rungs
agent                     one (or a few) fetched sigs
```

Load the next tranche like loading a till. Revoke = stop dispensing and, if the remaining hot rungs matter, **refund/close the channel** (those signatures stay valid until the escrow is gone or the counter is already past them).

### Why do this vs hot KMS signing

| | Hot `sign_voucher` | Pre-signed ladder |
|---|---|---|
| Payer key | Online in KMS | Offline after the batch; vault has no key |
| Agent | Requests a fresh signature | Fetches the next stored payload |
| Version bump | Freeze; human re-approves | Old ladder still claimable at old snapshot; freeze dispense; human re-signs a new ladder after `update_channel` |
| Revoke unused | Easy (never signed) | Only unreleased rungs; released/stolen `sig(n)` still works |
| `max_per_request` | Check Δcounter at sign time | Dispense one rung (or one window cap) at a time |
| Vault compromise | Can sign up to channel lock | Can spend up to **highest signature stored in that vault** |

Best default for “don’t give the agent money”: **+1 ladder + tranche-loaded vault + receipt-gated fetch**. Use hot KMS signing when the human cannot batch (unknown `calls`, frequent version bumps, many short-lived channels).

---

## Forced sequential fetch (receipt-gated)

The agent only ever obtains **the next** signature, and only after the provider attests the previous call. It does not choose `n`. Spec: [pseudo/v0/07-offchain.md](../pseudo/v0/07-offchain.md), digests: [pseudo/v0/02-ids.md](../pseudo/v0/02-ids.md).

```
acked = 0, dispensed = 0

next_voucher(receipt?):
  if dispensed > acked:           # outstanding sig(n) was used (or must have been)
    verify provider receipt for counter == dispensed
    acked = dispensed
  n = dispensed + 1               # first call: no receipt
  dispensed = n
  return (n, sig(n))              # exactly one rung
```

Loop: fetch `sig(1)` → call service → receipt for `1` → fetch `sig(2)` → …

### Sequential delivery

On-chain claims may jump (`0 → 5` in one tx). That is settlement, not delivery.

The service accepts only `n == last_accepted + 1`. A billed success is **one** response: `{ result, receipt }`. No receipt ⇒ the call is not complete (agent must not treat `result` as done; vault will not release `sig(n+1)`).

```
sign_provider( H(DOMAIN||"/rcpt" || channel_id || version || n || call_hash) )
```

`call_hash = H(request)`. Vault pins `receipt_signer` on the capability. Provider persists `(channel, n) → call_hash` so it can re-issue **the same** receipt if the HTTP body was dropped. Never a delayed “receipt-only” success path except that re-issue.

### What receipts force — and what they do not

| Control | Forces |
|---|---|
| Vault `n = dispensed + 1` | Sequential **fetch** |
| Receipt for `n` before `sig(n+1)` | Agent cannot stockpile vouchers against an **honest** provider |
| Service `n == last_accepted + 1` | Sequential **delivery** vs an honest service |
| On-chain lock + small hot tranche | Blast radius if the vault or provider is hostile |

A **colluding** provider can sign receipts without serving and walk the ladder. Receipts are not a substitute for small channels. They stop a runaway agent when the service is honest.

Do not set lookahead > 1. Prefetch and receipts conflict: holding `sig(5)` is holding five calls of bearer value.

---

## Who signs what

Maps onto [RESEARCH.md](RESEARCH.md) instructions.

| Action | Signer | Why |
|---|---|---|
| `create/delete` org/service | Provider, not this flow | — |
| `open_channel` | Human via vault | Locks money |
| `update_channel` | Human via vault | Re-lock / resize; resets counter |
| `next_voucher` | Vault (dispense pre-signed rung or hot-sign) | Agent does not pick `n`; receipt required if one voucher is outstanding |
| `submit_receipt` | Vault verifies **provider** signature | Acks call `n`; unlocks `sig(n+1)` |
| Provider receipt | Service / gateway key (`receipt_signer`) | Off-chain only; not an instruction |
| `claim_channel_funds` (path C) | Provider | Agent is not the claimer |
| `claim_channel_funds` (path B refund) | Human via vault | Recovers unused lock |
| Revoke capability | Human via vault | Instant; no chain tx |
| Rotate agent identity | Human via vault | Stolen workload key |

Revoke ≠ close channel. After revoke, the agent cannot get new signatures. Unsettled vouchers already issued remain claimable (that money is already promised). Unused lock stays in escrow until the human refunds or the channel expires.

---

## Tracking

Every successful sign writes an append-only audit event:

```
Audit {
  time, agent_id, channel_id, service_id,
  counter, call_hash,                // from receipt when acked
  dispensed, acked, settled, locked,
  capability_id, request_id
}
```

Dashboards for the human:

- Per agent: dispensed vs acked vs cap vs window (`dispensed == acked + 1` means a call in flight).
- Per service: locked vs dispensed vs settled.
- Alerts: window 80%, near `max_counter`, version freeze, `DenyNeedReceipt` / `DenyReceipt` spikes, dispensed stuck ahead of acked.

Reconciliation: `acked` (vault receipts) vs `channel.counter` (chain). If `settled > dispensed`, a voucher leaked outside the vault — incident. If `dispensed − acked > 1`, the dispenser is buggy.

---

## Arbitrary online services

Most APIs will not verify vouchers. Two consumption modes:

### A. Native service

The API implements the provider side of [pseudo/v0/07-offchain.md](../pseudo/v0/07-offchain.md): accept **only** `counter + 1`; on billed success return **`{ result, receipt }` in the same response**; keep the highest voucher for claim.

### B. Metering gateway (default for “any HTTP API”)

A gateway is the **on-chain service**. It holds upstream API keys / billed accounts, meters requests, and is the only voucher verifier the agent needs.

```
Agent + vault  --voucher-->  Gateway (on-chain Service, receipt_signer)
                                │  receipt
                                |<--
                                │  allowlisted upstream
                                ▼
                             Stripe / OpenAI / SaaS / another agent
```

The human opens channels **to the gateway**, not to each SaaS. Gateway metadata maps `(org, service)` → upstream. Limits then stack:

| Layer | Cap |
|---|---|
| Chain | Channel to the gateway (`calls`) |
| Vault | Per-agent, per-upstream (allowlist entries can be gateway service ids, one per upstream) |
| Gateway | Optional extra quota, auth to upstream |

The gateway is trusted for **upstream correctness** (it could over-bill in calls). It cannot take more than vouchers the vault signed, and cannot take more than the channel lock. That is the same trust model as any native provider.

One gateway organization with **one on-chain service per upstream** keeps per-service vault allowlists meaningful. A single “misc HTTP” service collapses tracking into one bucket — avoid that.

HTTP transport can be boring: `402` + voucher headers, or a required `Authorization: Voucher …` on every call. The program does not care.

---

## Lifecycle (owner + agent)

```
human: create payer_key (airgap or KMS; no export to agent)
human: allowlist services / gateway routes + pin receipt_signer
human: open_channel(calls = small)          // locked
human: issue Capability(...)
human: optional: pre-sign ladder, load today's tranche into vault

agent loop:
  vault.next_voucher(channel, receipt?)     // n=1: no receipt; else receipt for n-1
  HTTP call + (n, signature)
  gateway/service serves + receipt(n, call_hash)
  vault.submit_receipt(receipt)             // or pass it on the next next_voucher

human anytime:
  revoke capability / stop dispensing
  and/or update_channel / refund
```

If the service bumps `version`: vault **freezes** signing for that channel (pinned version mismatch). Human decides: refund remaining, or `update_channel` and issue a new capability. The agent must not silently re-price.

---

## What not to do

- Put a seed phrase or payer key in the agent prompt, env, or tool-calling host.
- Reuse the human’s main wallet as `payer`.
- Let the agent role call `open_channel` / `update_channel` “when budget is low.” That is self-service top-up.
- Sign `counter = calls` in one step, or dump `sig(1)…sig(k)` to the agent. Receipts exist so that cannot happen.
- Skip provider receipts, accept a result with no receipt, or let the agent pick `receipt_signer`.
- Treat provider invoices as the spend log. The vault’s `acked` counter is the owner-side truth.
- One shared capability for every agent on the same payer. Split `agent_id` even if v0 forces one on-chain channel per payer/service: the vault partitions `max_counter` across agents (sum ≤ `calls`).

v0 channel id has no `seed`, so two agents under one payer **share one chain envelope** for a given service. Isolation is a vault concern until a v1 seed exists ([RESEARCH.md](RESEARCH.md) open list).

---

## Failure and revoke

| Event | Response |
|---|---|
| Agent misbehaving | Revoke capability (seconds). Optionally freeze all capabilities on that `payer`. |
| Stolen agent JWT | Rotate workload identity; revoke old capability id. |
| Lost receipt after a real call | Provider re-issues the same `(n, call_hash)` receipt. Do not skip-ack in the vault. |
| Service compromised | Remove from allowlist; stop dispensing; human claims/refunds as in RESEARCH.md. |
| Vault sign endpoint abused | Still bounded by capabilities + channel locks; rotate agent identities; inspect audit. |
| KMS key leak **or** full ladder dump | Worst case: all open channels for that payer, or `price × max stored counter`. Close/refund; treat unclaimed high vouchers as spent. Keep **hot** rungs small. |

---

## Implementation sketch (vault API)

Minimum surface. Not on-chain.

```
# human
create_payer_key()
open_channel(service, calls)
update_channel(channel, calls?)
refund_channel(channel)                  # path B when allowed
put_capability(Capability)
revoke_capability(id)
load_tranche(channel_id, [(counter, signature), ...])   # pre-signed rungs; highest n = till

# agent  — cannot choose n
next_voucher(capability_id, channel_id, receipt?) -> (counter, signature)
submit_receipt(capability_id, channel_id, receipt) -> acked
# denies: DenyAllowlist | DenyCap | DenyWindow | DenyEmptyTill | DenyVersion
#         DenyRevoked | DenyNeedReceipt | DenyReceipt
```

State the vault must keep: capabilities (incl. `receipt_signer`), `dispensed` / `acked` per `(agent, channel)`, window buckets, audit log, cached `{channel.version, price, calls, expiration}`, current-tranche rungs, last `call_hash` per acked counter.

---

## Fit with v0 / v1

v0 is enough for the envelope: one channel per payer/service, voucher = `(channel_id, version, counter)`, human opens, agent only signs through the vault.

Nice-to-have later (do not block the vault):

- Channel `seed` so each agent can have its own on-chain envelope (true isolation + per-agent refund).
- `fund_channel` without resetting `counter` so the human tops up without invalidating in-flight vouchers.
- Cooperative close so unused lock is not stuck until TTL.

The vault does not replace those; it makes v0 usable for agents **now**.

---

## Recommended default

For a human running agents against mixed APIs:

1. Dedicated `payer` account, funded little and often.
2. **Pre-signed ladder**, key stays air-gapped; vault holds only the next tranche (e.g. one day). Hot KMS signing only if you cannot batch.
3. Metering gateway as the on-chain service, one gateway-service per upstream.
4. Small channels (short TTL, modest `calls`).
5. Per-agent capabilities with allowlist and pinned `receipt_signer`; `next_voucher` + receipts; `lookahead = 1`. Service billed success = `{ result, receipt }` in one response; accept only `counter + 1`.
6. Version pin + freeze on bump; re-sign a new ladder after `update_channel`.
7. Revoke by default when an agent job ends; unused rungs stay in cold storage, not in the agent.

The chain limits **how much can ever leave escrow**. The vault limits **who can promise how much, to whom, how fast**, and is the only place the agent is allowed to talk to money.
