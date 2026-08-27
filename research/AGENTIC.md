# Agentic spend: humans, vaults, and call channels

How an AI agent pays for **any** online service using prepaid call channels ([RESEARCH.md](RESEARCH.md)) without holding the money or the keys.

The agent is an untrusted worker. The human is the budget owner. A **spend broker** combines vault policy, KMS/pre-signed vouchers, provider transport, and receipt verification. The agent never handles a key or bearer voucher. On-chain channels are the hard ceiling; broker policy is the fine-grained limiter and audit log.

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
| Runaway or injected agent | Can call broker `execute` | Cannot choose counters, see vouchers, skip receipts, open/fund channels, or export keys |
| Malicious / buggy service | Tries to jump counter or falsify a result | Broker rejects jumps/bad hashes; isolated escrow remains the hard cap |
| Compromised agent identity | Can request broker executions as that agent | Damage ≤ capability windows and channel escrow; receipts still gate honest-provider calls; human can revoke |
| Compromised broker/KMS | Can sign as payer or dump its hot tranche | Damage ≤ affected channel escrows / highest hot rungs; keep both small |
| Malicious provider | Holds a voucher before serving; may claim or sign a false receipt | Damage stays within one small channel/tranche; base protocol does **not** prove semantic delivery |

Delivery trust is explicit: v0 assumes an honest native provider or trusted metering gateway. Stronger assurance requires a provider bond/dispute process, TEE, or service-specific verifiable computation.

---

## Principle

**The agent never sees a spending private key or voucher.** It sends an intended request to the broker. The broker chooses the next counter, sends the voucher, verifies the provider receipt against request/result hashes, persists it, then returns the result.

Damage is the **min** of:

```
on-chain remaining     = channel.price × (channel.calls − channel.counter)
authorized remaining   = policy max − already signed for that agent/service/window
```

Even a fully compromised agent plus an abused broker API cannot spend more than capability limits and channel escrow. Revoke blocks new calls immediately; challenged close recovers unused escrow.

```
Human owner
  │  policies, allowlists, fund/close/AdoptTerms, revoke
  │  optional: airgap-sign a voucher ladder, load vault in tranches
  ▼
Spend broker (vault/KMS + durable state)
  ▲       │
  │       └── voucher + request ──► Service/gateway ──► chain claim
  │                                  │
  │                                  └── result + receipt ──► broker
  │
Agent (execute request / verified result; no key or voucher)
```

---

## Four numbers (track all four)

Do not treat “what the agent spent” as a single figure.

| Number | Where | Meaning |
|---|---|---|
| **Locked** | Chain: `price × calls` | Hard budget parked in escrow for that service |
| **Authorized** | Broker: highest **dispensed** `counter` | Voucher sent to the provider |
| **Acked** | Broker: highest verified receipt `counter` | Calls the provider attested |
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

The broker ledger is source of truth for limits/receipts. The chain is source of truth for settlement. These counters share one number line for the life of the channel: `fund_channel` raises `calls`; AdoptTerms continues from the same `dispensed` / `counter`.

---

## Layered limits

### 1. On-chain envelope (hard, coarse)

Opening a nonce-isolated channel **is** the human committing one asset to one service/agent. Keep `calls` small (hours of expected use, not months). Top up with `fund_channel` rather than a huge initial lock. The agent cannot open, fund, close, or adopt terms.

This is the last backstop if policy fails. Isolated escrows bound blast radius per agent/task.

### 2. Spend-broker policy (hard for the agent, fine-grained)

Evaluated inside the broker on every `execute`. The agent cannot bypass it or obtain signatures directly.

Suggested fields on a **capability** (issued by the human, stored in the vault, not on-chain):

```
Capability {
  agent_id:           Bytes
  payer:              AccountId
  allowlist:          [ {
    org_id, service_id, channel_id, nonce,
    version, asset, receipt_signer
  } ]
  max_counter:        u32                // ≤ channel.calls; per channel
  max_amount:         Balance            // optional cap across services
  window:             { period, max_amount, max_calls }
  valid_until:        Time
  ops:                { execute }
}
```

The capability pins all channel snapshots, especially `receipt_signer`; never take authority from agent input or mutable metadata.

Checks on `execute(capability, channel, idempotency_key, request)` ([pseudo/v0/07-offchain.md](../pseudo/v0/07-offchain.md)):

1. Capability not expired / not revoked.
2. `channel_id` in allowlist.
3. Channel is `Open`; version/asset/receipt signer match the pin.
4. `dispensed == acked` (otherwise retry the same outstanding request).
5. Broker chooses `n = dispensed + 1`; `n ≤ max_counter` and `n ≤ channel.calls`.
6. Window still has room.
7. Response receipt binds `channel_id`, version, `n`, request hash, and result hash.

On success: broker durably stores the result/receipt and advances `acked` before returning only the result. Receipts do not go on-chain.

After `fund_channel`, raise `max_counter` and load extra rungs. After AdoptTerms, re-pin version and load a new ladder from `counter + 1`.

### 3. Human gates (rare, high impact)

Vault roles:

| Role | Auth | Allowed |
|---|---|---|
| **Root / owner** | Hardware key, 2FA | Create payer keys, set policies, open/fund/close/AdoptTerms, revoke, export never |
| **Approver** | Same or a second human | Optional: signatures with `Δamount` above a threshold |
| **Agent** | Workload identity (short-lived JWT / SPIFFE / vault AppRole) | `execute` only |

Do not expose `next_voucher`, raw signing, `open_channel`, `fund_channel`, or transition instructions to the agent role.

---

## Least-privilege key

The payer key in the KMS must be **unable** to do anything except channel vouchers (and, under the human role, the channel instructions).

If that key can `transfer` or sign arbitrary bytes, policy is theater.

Practical shapes:

- **Message-policy KMS**: allow only canonical voucher digests `(deployment, channel_id, version, counter)`.
- **Dedicated spend account**: zero extra balance; the only value it controls is already inside channels. Even a leaked key cannot steal funds that were never locked.
- **Split roles**: `funding_key` (human, holds idle capital) vs `payer_key` (vault, only used as channel owner). Human transfers into the payer just enough to open or fund the next small channel.

Prefer a dedicated spend account **and** message policy. Then a leak of `payer_key` still cannot empty the human’s main wallet, and cannot sign a voucher the policy would have rejected *if the leak is of the agent API rather than the KMS itself*. A raw leak of `payer_key` bypasses policy — hence keep that material in HSM/KMS, never in the agent runtime, never in logs.

---

## Pre-signed ladder (store signatures, not keys)

Hot signing keeps the payer key **online** in a KMS. The alternative: the human signs offline, stores **only the signatures** in the vault, and never gives the vault (or the agent) the key.

Example: open a $100 channel at $0.01/call → `calls = 10_000`. Human signs the ladder (`counter = 1 … 10_000`) offline. The full ladder remains encrypted under offline/HSM-controlled wrapping keys; only a small current tranche is loaded into the broker. The agent never fetches a signature.

Same-term top-up is append-only: if `calls` becomes `12_000`, sign `10_001 … 12_000` at the same version. After AdoptTerms, sign a new ladder from the latest `counter + 1` at the new version.

That works. Two properties of this protocol change how you should store and release them.

### Cumulative IOUs, not tickets

Vouchers replace each other. `sig(100)` pays `price × (100 − already_settled)` by itself. `sig(1) … sig(99)` are worthless once `sig(100)` is settled. Re-submitting `sig(100)` after settlement is a zero claim.

Loading the first 100 signatures is economically one permission: raise authorized counter to 100 ($1). A broker/storage compromise needs only the last row. Treat the highest online counter as the hot exposure.

You do not need 10_000 hot rows for a 100/day budget. That budget is a broker window on successful `execute` calls, not a reason to load `sig(100)` online.

| Goal | What to pre-sign | What to dispense |
|---|---|---|
| Sequential calls (default) | `1, 2, 3, …, 10_000` (then `10_001…` on fund) | Broker internally uses only `sig(dispensed + 1)` |
| Coarse *budget* (not sequential) | `100, 200, …` | One rung = one day’s max claim — **do not use for agents** that must call +1 |

Agent mode is the first row. See [Forced sequential execution](#forced-sequential-execution-broker-gated).

### The vault still holds cash

A stored `sig(n)` is a **bearer instrument**. Anyone who has it can hand it to the provider; claim is anyone-can-submit. If the hot vault contains `sig(10_000)`, a vault dump **is** the $100 channel — same blast radius as leaking the key for channel-spend purposes.

Keys-not-in-vault only reduces risk if no online store holds the top of the ladder and the cold ladder is encrypted:

```
offline/HSM wrapping key  decrypts future tranches; never online
encrypted cold store      sig(101) … sig(10_000) ciphertext
spend broker              only current small tranche; vouchers never returned to agent
```

Load the next tranche like a till. Revoke stops broker execution; request a challenged close to remove remaining escrow. Already-sent vouchers remain valid through the challenge.

### Why do this vs hot KMS signing

| | Hot `sign_voucher` | Pre-signed ladder |
|---|---|---|
| Payer key | Online in KMS | Offline after the batch; vault has no key |
| Broker | Requests a fresh signature internally | Consumes the next stored payload internally |
| Version bump | Freeze; human re-approves | Old ladder remains claimable during challenge; new ladder starts at `counter+1` after AdoptTerms |
| Same-term fund | Sign / load extra rungs | Append rungs at the same version |
| Revoke unused | Easy (never signed) | Only unreleased rungs; released/stolen `sig(n)` still works |
| Per-call sequencing | Broker signs one next counter | Broker consumes one next rung |
| Broker/KMS compromise | Can sign up to channel escrow | Can spend up to highest hot signature |

Best default: **+1 ladder + encrypted cold storage + small broker tranche + receipt verification**. Use hot KMS signing when batching is impractical.

---

## Forced sequential execution (broker-gated)

The broker only sends the next signature; the agent obtains none and cannot choose `n`. Spec: [pseudo/v0/07-offchain.md](../pseudo/v0/07-offchain.md), digests: [pseudo/v0/02-ids.md](../pseudo/v0/02-ids.md).

```
execute(idempotency_key, request):
  require dispensed == acked
  n = dispensed + 1
  atomically persist dispensed = n and request_hash
  send request + sig(version, n)
  verify receipt(request_hash, result_hash)
  atomically persist acked = n and result
  return result                    # signature is never returned
```

On loss, retry the same `n` and idempotency key. Never allocate `n+1` while one request is unacknowledged.

### Sequential delivery

On-chain claims may jump (`0 → 5` in one tx). That is settlement, not delivery.

The service accepts only `n == last_accepted + 1` for the current Open version. A billed success is a durable `{ result, receipt }` record:

```
sign_provider(H(
  DOMAIN || "receipt" || channel_id || version || n
  || request_hash || result_hash
))
```

The broker pins `channel.receipt_signer`. Provider persists the request hash, complete result, result hash, receipt, and highest voucher under `(channel, n)`. A retry returns exactly that stored record and never repeats the upstream call.

### What receipts force — and what they do not

| Control | Forces |
|---|---|
| Broker `n = dispensed + 1` | Sequential voucher **issue** |
| Signatures never leave broker | Agent cannot steal/stockpile bearer vouchers |
| Valid receipt before returning result | Broker records request/result attestation |
| Service `n == last_accepted + 1` | Sequential **delivery** vs an honest service |
| On-chain lock + small hot tranche | Blast radius if the vault or provider is hostile |

A malicious provider can claim after receiving a voucher or sign a false receipt. Receipts prevent runaway-agent sequencing errors against an honest provider; they do not prove service correctness.

Do not set lookahead > 1. Prefetch conflicts with the bounded one-request state machine.

---

## Who signs what

Maps onto [RESEARCH.md](RESEARCH.md) instructions.

| Action | Signer | Why |
|---|---|---|
| `create/delete` org/service | Provider, not this flow | — |
| `set_organization_paused` / `set_service_paused` | Provider org/service owner | Stop new admission; not a payer/agent action |
| `open_channel` | Human via vault | Locks money |
| `fund_channel` | Human via vault | Same-term top-up |
| `request/cancel` channel transition | Human via vault | Starts/stops challenged close / AdoptTerms |
| `finalize_channel_transition` | Anyone | Refunds / adopts terms after challenge; funds always follow protocol |
| Voucher | Spend broker (pre-signed rung or hot KMS) | Internal only; agent never sees it |
| Receipt | Service/gateway `channel.receipt_signer` | Binds request/result hashes; off-chain only |
| `claim_channel_funds` (voucher settlement) | Provider | Agent is not the claimer |
| Revoke capability | Human via vault | Instant; no chain tx |
| Rotate agent identity | Human via vault | Stolen workload key |

Revoke ≠ close channel. Revoke blocks new broker calls; request a challenged close to recover unused escrow. Already-issued vouchers remain claimable until finalization.

---

## Tracking

Every successful sign writes an append-only audit event:

```
Audit {
  time, agent_id, channel_id, service_id,
  version, counter, request_hash, result_hash,
  dispensed, acked, settled, locked,
  capability_id, idempotency_key
}
```

Dashboards for the human:

- Per agent: dispensed vs acked vs cap vs window (`dispensed == acked + 1` means a call in flight).
- Per service: locked vs dispensed vs settled.
- Alerts: window 80%, near `max_counter`, snapshot/status freeze, invalid receipt spikes, outstanding request stuck.

Reconciliation is per channel: `acked` (broker receipts) vs `channel.counter` (chain). If `settled > dispensed`, a voucher escaped the broker or state was corrupted. If `dispensed − acked > 1`, the broker concurrency guard failed.

---

## Arbitrary online services

Most APIs will not verify vouchers. Two consumption modes:

### A. Native service

The spend broker calls the API. The API implements [pseudo/v0/07-offchain.md](../pseudo/v0/07-offchain.md): accept `counter + 1` for the current Open version, execute idempotently, return `{ result, receipt(request_hash, result_hash) }`, and keep the highest voucher for claim.

### B. Metering gateway (default for “any HTTP API”)

A gateway is the **on-chain service**. It holds upstream API keys / billed accounts, meters requests, and is the only voucher verifier the agent needs.

```
Agent --request--> Spend broker --voucher--> Gateway (receipt_signer)
          ^             |                        │
          |             | verified result       │ allowlisted upstream
          +-------------+                        ▼
                                      Stripe / OpenAI / SaaS / another agent
```

The human opens channels **to the gateway**, not to each SaaS. Gateway metadata maps `(org, service)` → upstream. Limits then stack:

| Layer | Cap |
|---|---|
| Chain | Channel to the gateway (`calls`) |
| Spend broker | Per-agent, per-upstream capabilities |
| Gateway | Optional extra quota, auth to upstream |

The gateway is trusted for upstream correctness: request/result hashes attest bytes, not usefulness. It cannot take more than broker-issued vouchers or channel escrow. Optional stronger layers are bond/dispute, TEE, or verifiable computation.

One gateway organization with one on-chain service per upstream keeps capabilities meaningful. Avoid a single “misc HTTP” bucket.

HTTP transport can be boring: `402` + voucher headers, or a required `Authorization: Voucher …` on every call. The program does not care.

---

## Lifecycle (owner + agent)

```
human: create payer_key (airgap or KMS; no export to agent)
human: allowlist service/gateway + pin channel snapshots
human: open_channel(nonce = agent_id/random, calls = small)
human: issue Capability(...)
human: pre-sign/encrypt ladder; load small current tranche into broker

agent loop:
  broker.execute(capability, channel, idempotency_key, request)
  broker sends voucher, verifies result+receipt, persists, returns result

human anytime:
  revoke capability
  fund_channel(additional_calls)                 # same terms
  request_channel_transition(Close or AdoptTerms)
  after challenge: finalize_channel_transition
```

If service terms change, broker freezes. Human requests AdoptTerms; provider settles old-version vouchers during the challenge; finalization snapshots new terms and keeps `counter`. Human loads a new ladder/capability from `counter + 1`. The agent cannot silently re-price.

---

## What not to do

- Put a seed phrase or payer key in the agent prompt, env, or tool-calling host.
- Reuse the human’s main wallet as `payer`.
- Let the agent call raw signing, `open_channel`, `fund_channel`, or transition instructions.
- Return vouchers from the broker API or dump `sig(1)…sig(k)` to the agent.
- Skip result-hash receipts or let agent/metadata choose `receipt_signer`.
- Treat provider invoices as the spend log. The broker’s `acked` counter is the owner-side truth.
- Reuse a channel nonce. Give each agent/task its own channel envelope.
- Store the full plaintext ladder online; ciphertext is only safe while wrapping keys remain offline/HSM-controlled.

---

## Failure and revoke

| Event | Response |
|---|---|
| Agent misbehaving | Revoke capability (seconds). Optionally freeze all capabilities on that `payer`. |
| Stolen agent JWT | Rotate workload identity; revoke old capability id. |
| Lost response after execution | Broker retries the same `(version, n, idempotency_key, request_hash)`; provider returns stored result+receipt. |
| Service compromised | Remove from allowlist; freeze broker; request challenged close; provider may claim already-issued vouchers. |
| Broker API abused | Still bounded by capabilities, receipt gate, tranche, and escrow; rotate agent identities; inspect audit. |
| KMS key leak **or** full ladder dump | Worst case: all open channels for that payer, or `price × max stored counter`. Close/refund; treat unclaimed high vouchers as spent. Keep **hot** rungs small. |
| Broker concurrency/storage failure | Freeze channel; compare `dispensed/acked/settled`; never skip a counter manually. |

---

## Implementation sketch (broker API)

Minimum surface. Not on-chain.

```
# human
create_payer_key()
open_channel(service, nonce, calls)
fund_channel(channel, additional_calls)
request_channel_transition(channel, Close | AdoptTerms { calls })
cancel_channel_transition(channel)
finalize_channel_transition(channel)
put_capability(Capability)
revoke_capability(id)
load_encrypted_tranche(channel_id, version, rungs)

# agent — no signature API
execute(capability_id, channel_id, idempotency_key, request) -> result
# denies: DenyAllowlist | DenyCap | DenyWindow | DenyEmptyTill | DenySnapshot
#         DenyRevoked | DenyChannelClosing | IncompleteResponse | DenyReceipt
```

Broker state: capabilities; `dispensed/acked` per `(agent, channel)`; windows; audit; cached snapshots/status; current encrypted tranche; outstanding idempotency/request hash; verified result/receipt. Updates use durable transactions/CAS.

---

## Fit with v0 / v1

v0 includes nonce-isolated channels, version-bound vouchers, monotonic counters, `fund_channel`, challenged close/AdoptTerms, explicit assets, and deployment-scoped vouchers. One agent/task should have one nonce/channel.

Nice-to-have later (do not block v0):

- Provider bonds/disputes, TEEs, or verifiable computation for stronger delivery assurance.

The broker does not prove semantic delivery; it makes v0 safely operable under the stated provider/gateway trust model.

---

## Recommended default

For a human running agents against mixed APIs:

1. Dedicated payer account; one protocol-assigned next-nonce channel per agent/task.
2. Pre-signed ladder encrypted under offline/HSM wrapping keys; broker holds only a small tranche. Append rungs on `fund_channel`. Hot KMS signing only if batching is impractical.
3. Metering gateway as the on-chain service, one gateway-service per upstream.
4. Small per-channel escrow, short TTL, modest calls, non-zero challenge period.
5. Agent calls broker `execute`; no raw voucher endpoint. Provider accepts only `counter+1` and returns request/result-bound receipt.
6. Freeze on status/snapshot change; challenged AdoptTerms keeps `counter` and re-pins version; then load a new ladder/capability from `counter+1`.
7. Revoke at job end and request challenged close; future rungs remain encrypted offline.

The chain limits how much can leave escrow. The broker limits who may spend, on which channel, and how fast; it is the only component through which an agent may request paid work.
