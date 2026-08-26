# Prepaid Call Channels for Metered Services

A chain-agnostic design for prepaid, unidirectional payment channels used to meter API / agent calls. Consumers lock funds once, consume off-chain with signed vouchers, and settle on-chain in batches. The on-chain program never sees individual calls.

This is a fit for agent-to-service payments: high call volume, small unit price, and a need to avoid a transaction per request.

---

## Problem

Metered services (LLM inference, tool APIs, agent runtimes) need:

1. **Prepayment** so the provider is not extending credit.
2. **Off-chain usage** so each call does not hit the chain.
3. **Trust-minimized settlement** so the provider can prove how many calls were consumed, and the consumer can recover unused funds.
4. **Price / terms changes** without silently applying new rates to already-funded channels.

A unidirectional prepaid channel with a monotonically increasing counter covers that set.

---

## Roles

| Role | Who | What they do |
|---|---|---|
| **Organization owner** | Provider business | Registers an organization, manages members, holds the org deposit. |
| **Service owner** | Org member | Publishes a priced service, receives claimed funds. |
| **Channel owner (payer)** | Consumer / agent wallet | Opens a channel, signs vouchers as calls are consumed, can refund unused funds after expiry or a service version bump. |
| **Claimer** | Typically the service owner | Submits the latest signed voucher and withdraws earned funds. Anyone with a valid voucher can trigger a claim; payout always goes to the service owner. |

---

## On-chain objects

Three nested records, plus a program-owned escrow vault.

```
Organization
  └── Service          (priced offering, versioned)
        └── Channel    (payer prepaid quota against that service)
```

### Organization

A named provider entity.

| Field | Meaning |
|---|---|
| `id` | Deterministic hash of `(domain, owner, name)` |
| `owner` | Account that created it |
| `name` | Unique per owner |
| `members` | Count of member accounts |
| `services` | Count of published services |
| `metadata` | Arbitrary blob (endpoint, docs, branding) |

Creating an organization **reserves a deposit** from the owner (anti-spam). The deposit is returned on delete. Members are stored as `(organization_id, account) → rank`. Rank `0` is the owner; other members start at rank `1`. Only members may create services under the org.

IDs are content-addressed. Two owners can reuse the same display name; collisions under one owner are rejected.

### Service

A priced, versioned offering.

| Field | Meaning |
|---|---|
| `id` | Deterministic hash of `(domain, owner, name)` |
| `owner` | Account that receives claims |
| `organization` | Parent org id |
| `name` | Unique per organization |
| `version` | Starts at `1`; increments on every update |
| `price` | Unit price **per call** |
| `minimum_calls` | Floor on prepaid quota when opening / updating a channel |
| `expiration_threshold` | Channel lifetime, in chain time units (blocks / slots) |
| `trials` | Reserved for a free-trial quota (not enforced in the current logic) |
| `channels` | Count of open channels against this service |
| `metadata` | Arbitrary blob (schema, model id, rate-limit hints) |

Updating a service **always bumps `version`**. Open channels keep their snapshot; new terms apply only after the payer `update_channel`s (see [Service versioning](#service-versioning)).

### Channel

A prepaid escrow from one payer to one service. At most **one channel per payer per service**.

| Field | Meaning |
|---|---|
| `id` | Deterministic hash of `(domain, payer, organization_id, service_id)` |
| `owner` | Payer |
| `organization` / `service` | Target |
| `version` | Service version **snapshotted at open / update** |
| `price` | Unit price **snapshotted at open / update** |
| `calls` | Prepaid quota (number of calls funded) |
| `counter` | Highest settled call count (starts at `0`) |
| `expiration` | `open_time + service.expiration_threshold` |

Locked amount at open:

```
funds = channel.price × channel.calls
```

Unsettled remainder at any time:

```
remaining = channel.price × (channel.calls − channel.counter)
```

The channel stores its own `price` and `version`. Later service updates do not rewrite those fields in-place; they only take effect if the payer **updates** the channel, or if the payer **closes** it after a version mismatch.

### Escrow vault

Channel funds sit in a program-owned escrow (not in the payer's or provider's wallet). Opens and updates transfer **in**; claims, refunds, and closes transfer **out**. A single shared vault is enough for the model; a per-channel vault is safer (see [Solana mapping](#mapping-notes-for-solana)).

---

## Identity and domain separation

All hashes and voucher messages share a fixed domain prefix so they cannot be replayed as some other protocol's payload.

```
org_id     = H( domain || owner || name )
service_id = H( domain || owner || name )
channel_id = H( domain || payer || org_id || service_id )

voucher    = H( domain || channel_id || channel.version || counter )
```

`H` is a 256-bit cryptographic hash. The voucher is what the payer signs off-chain.

Signature verification should accept both the raw message bytes and the common wallet wrapping `<Bytes>…</Bytes>`, so browser / extension wallets remain usable.

---

## Lifecycle

```
create organization
        │
        ▼
create service  ──────────────────────────────► update service (version++)
        │                                              │
        ▼                                              │ does not rewrite channels;
open channel (lock price × calls)                      │ old vouchers still settle
        │                                              │ at snapshotted price
        │     off-chain: payer signs vouchers          │
        │     (channel_id, channel.version, counter)   │
        │                                              │
        ▼                                              ▼
claim (provider, signed counter)              payer may refund remaining
        │                                     if expired OR version changed
        ├── remaining > 0 → counter advances           (races unclaimed vouchers)
        └── remaining = 0 → channel deleted
        │
        ▼
update channel (payer): refund remainder,
re-lock at current service price / version,
reset counter to 0, reset expiration
```

### 1. Open

Payer specifies a service and a call quota `N`.

Checks:

- Service exists.
- `N ≥ service.minimum_calls`.
- No channel already exists for `(payer, channel_id)`.

Effects:

- Transfer `service.price × N` from payer to the escrow vault.
- Snapshot `version`, `price`, `calls = N`, `counter = 0`.
- Set `expiration = now + service.expiration_threshold`.
- Increment `service.channels`.

### 2. Off-chain consumption (the actual “channel”)

Each successful call, the payer issues a **replaceable IOU**: a signature over

```
(domain, channel_id, channel.version, counter)
```

`counter` is **cumulative**, not per-call. After 17 billed calls the voucher is `counter = 17`, not 17 separate signatures of `1`. The provider keeps only the highest-counter signature. Older vouchers are worthless.

This is a classic unidirectional payment channel:

- The payer cannot decrease the counter (on-chain claim requires `counter > channel.counter`).
- The provider cannot invent a higher counter without the payer's signature.
- Settlement cost is one on-chain claim for any number of calls.

The signed version is the **channel snapshot**, not the live service version. After `update_service`, outstanding vouchers remain claimable at `channel.price`. New calls should stop until the payer `update_channel`s (new snapshot, `counter = 0`).

For agents, delivery is stricter than settlement: the service accepts only `counter + 1`, and the vault releases the next payer voucher only after a **provider receipt**. That protocol is off-chain ([AGENTIC.md](AGENTIC.md), [pseudo/v0/07-offchain.md](../pseudo/v0/07-offchain.md)).

### 3. Claim

Anyone may submit `(channel, counter, signature)`.

Payout always goes to **`service.owner`**, not the transaction sender.

Path A — **no remaining funds** (`counter` already consumed the quota):

- Delete the channel, decrement `service.channels`. No signature needed.

Path B — **payer is closing**:

- Allowed only if the channel is **expired** or the **service version no longer matches** the channel snapshot.
- Refund `remaining` to the payer, delete the channel.

Path C — **voucher settlement** (the normal provider path):

- `counter > channel.counter` (strictly increasing).
- Signature verifies as the **payer** over `(domain, channel_id, channel.version, counter)`.
- `claim = channel.price × (counter − channel.counter)`, capped by remaining funds.
- Transfer `claim` from escrow to `service.owner`.
- Persist the new `counter`. Channel stays open if funds remain.

Path C still works after a service version bump: the voucher and the payout use the snapshot. A bump does not burn unclaimed IOUs.

Claims are **incremental**. The provider can cash out every few calls, or wait and submit the latest voucher once.

### 4. Update channel (payer)

Used when the payer wants to keep using a service after a price/version change, or to resize the quota. This is a **replace**, not a delta top-up.

1. Refund `remaining` (at the **old** snapshotted price) to the payer.
2. Require new `N ≥ service.minimum_calls` (defaults to the minimum if omitted).
3. Snapshot the **current** service `version` and `price`.
4. Reset `counter = 0` (old vouchers are bound to the previous snapshot).
5. Reset `expiration = now + current expiration_threshold`.
6. Lock `current_price × N` into escrow and **persist** the channel.

The channel id does not change (it is derived from payer + org + service, not from version). Refunding `remaining` races unclaimed vouchers the same way path B does — providers should claim before the payer updates.

### 5. Delete organization / service

Owner-only. Unreserves the creation deposit. Organization delete also wipes members.

Deletes are **rejected** while children exist, so escrow cannot be frozen against a missing service:

- `delete_service` requires `service.channels == 0`.
- `delete_organization` requires `organization.services == 0` (hence all channels already gone).

Close every channel via claim/refund first, then delete service, then delete org.

---

## Service versioning

`update_service` is a **hard cut** for *new* terms, not a wipe of in-flight IOUs. Any change — name, price, minimum calls, expiry window, trials, metadata — increments `version`. Open channels keep their snapshot.

Consequences:

| Actor | After a version bump |
|---|---|
| Provider | Outstanding vouchers still settle at `channel.price` / `channel.version`. Cannot apply the new price to old IOUs. Should claim before bumping (same race as expiry). Stop accepting new calls until the payer re-locks. |
| Payer | May refund **unused** remaining immediately (`version_changed`), without waiting for expiry. That refund races any unclaimed voucher. |
| Payer who wants to continue | Must `update_channel`, which re-prices, re-versions, and resets `counter` to 0. |

This keeps prepaid rates honest: a provider cannot raise `price` and cash old counters at the new rate. It also does not let a bump destroy already-signed work.

---

## Time and refunds

Channels are not infinite. `expiration = open_or_update_time + service.expiration_threshold`.

Until expiry **and** while `channel.version == service.version`, the payer **cannot** unilaterally withdraw. That is what gives the provider a window to claim vouchers. After expiry **or** a version bump, the payer can reclaim whatever is still in escrow (`remaining` uses the on-chain `counter`, not an unclaimed voucher).

If the provider holds a valid voucher, they should claim before expiry and before bumping the service. Nothing stops a payer from refunding `remaining` once the window opens, even if a voucher exists. Race: first successful on-chain transaction wins. Operationally, providers should claim on a timer — and claim (or avoid bumping) before `update_service`.

---

## Economic parameters (suggested)

These were used in the reference implementation and are knobs, not protocol invariants.

| Parameter | Role |
|---|---|
| Organization deposit | Anti-spam for org creation; returned on delete. |
| Service deposit | Anti-spam for service creation; returned on delete. |
| Max name / metadata length | Bound storage. |
| Max organization members | Enforced at org create (owner counts as 1). |
| `minimum_calls` | Prevents dust channels that cost more to settle than they hold. |
| `expiration_threshold` | Provider claim window vs. payer capital lockup. |

`trials` is stored on the service but is not billed in v0 (intended as a free-call allowance before the counter starts billing).

---

## What stays off-chain

The chain does **not** record individual calls, request payloads, or API keys. Off-chain components that a Solana (or any other) implementation still needs:

1. **Voucher transport** — how the payer sends `(counter, signature)` to the provider (HTTP header, streaming session, wallet popup).
2. **Provider receipts** — a billed success is `{ result, receipt }` in the **same** response. Receipt is `sign(H(DOMAIN||"/rcpt" || channel_id || version || n || call_hash))`. The vault requires it before releasing `sig(n+1)`. Not on-chain. A body without a receipt is not a successful call.
3. **Provider accounting** — persist the highest voucher; accept only `counter+1` for delivery; persist `call_hash` so receipts can be re-issued.
4. **Watchers** — provider bot that claims before expiry **and before `update_service`**; payer bot that refunds after expiry or version change.
5. **Discovery** — org/service metadata as a registry of endpoints, prices, and receipt keys.

The on-chain program is only escrow + voucher verification + versioned pricing.

---

## Invariants (for a port)

A port to Solana (or any other chain) should preserve these, regardless of account layout:

1. **One channel per `(payer, organization, service)`.** Id is a pure hash of those three plus a domain prefix.
2. **Escrow conservation.** Funds attributable to a channel equal `price × (calls − counter)` until a claim, refund, or update.
3. **Monotonic counter.** On-chain `counter` only increases, and only with a payer signature.
4. **Snapshotted price.** Claims use `channel.price`, never the live `service.price`.
5. **Version-bound vouchers.** The signed version must equal **`channel.version`** (the snapshot), not the live service version.
6. **Payer refund only after expiry or version mismatch.**
7. **Claim payout to `service.owner`**, independent of who submits the tx.
8. **Service update always increments version** (no silent field edits).
9. **No delete while children exist.** `delete_service` iff `channels == 0`; `delete_organization` iff `services == 0`.

---

## Mapping notes for Solana

Not required to understand the model; useful when implementing.

| Concept here | Typical Solana shape |
|---|---|
| Program-owned escrow vault | Prefer a PDA / token account **per channel** so conservation is local; a single shared vault is logically equivalent but one overdraw hits every payer |
| Organization / service / channel records | PDAs: `["org", owner, name]`, `["svc", org, name]`, `["ch", payer, org, svc]` |
| Reserved deposits | Token accounts or lamport holds on those PDAs |
| `block_number + threshold` | `Clock::get().slot` (or unix timestamp) + configured TTL |
| Payer signature over voucher | Ed25519 via `sysvar::instructions` (verify in-program) or a signed off-chain message checked with `ed25519_program` |
| Native token | SOL or an SPL mint (USDC) — the model is mint-agnostic as long as `price` and escrow use the same mint |
| Events | Anchor events / program logs for indexers |

The voucher format should stay chain-independent so the same agent wallet can sign for multiple deployments:

```
sign( H( domain || channel_id || channel.version || counter ) )
```

Pick a domain string per deployment (`apc/v0/...`) and keep it stable. `channel.version` is the snapshot at open / last `update_channel`.

---

## Security notes for implementers

- **Do not settle at live price.** Always use the snapshotted `channel.price`.
- **Bind vouchers to `channel.version`.** Live `service.version` would let a bump burn unclaimed IOUs (or, if combined with live price, cash old counters at a new rate).
- **Cap the claim** by remaining escrow. A signed `counter > calls` must not overdraw.
- **Refuse delete while children exist.** Otherwise claim/refund cannot load `service.owner` and funds freeze.
- **`update_channel` must persist** and **reset `counter` to 0**.
- **Anyone-can-submit claims** is intentional (relayer-friendly) only if payout is hardcoded to `service.owner`.
- **Signature wrapping.** If wallets wrap messages, verify both raw and wrapped forms, or standardize on raw 32-byte Ed25519 in the Solana port.

---

## Minimal instruction set

A complete port needs roughly these instructions:

| Instruction | Caller | Purpose |
|---|---|---|
| `create_organization` | future owner | Register org, lock deposit, add members |
| `delete_organization` | owner | Remove org, return deposit (requires `services == 0`) |
| `create_service` | org member | Publish priced service, lock deposit |
| `update_service` | service owner | Edit terms, bump version (does not rewrite channels) |
| `delete_service` | service owner | Remove service, return deposit (requires `channels == 0`) |
| `open_channel` | payer | Lock `price × calls`, snapshot terms |
| `update_channel` | payer | Refund remainder, re-lock at current terms |
| `claim_channel_funds` | anyone | Settle voucher, or payer refund on expiry / version change |

That is the whole on-chain surface. Everything else is off-chain protocol around the voucher.

---

## Open for v1

Not in v0. Do not invent these during the first port:

- Add/remove members after org create; transfer org or service ownership (rank is unused beyond owner vs member).
- Cooperative close while the channel is live (payer is locked until TTL or a version bump).
- `fund_channel`: add calls at the current snapshot without resetting `counter`.
- Optional `seed` in `channel_id` so one wallet can hold parallel quotas (multiple agents).
- Bill `trials` as a free-call allowance before `counter` starts.
- On-chain challenge period so a version bump / expiry cannot race an unclaimed voucher (today: first tx wins).
