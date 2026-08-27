# Types, config, errors, events

Primitives: `AccountId`, `AssetId`, `Hash256`, `Balance`, `Time` (block / slot / unix — target-defined), `Bytes`, `Rank = u32`, `ChannelNonce = u64`.

Every channel has an isolated program-owned `ESCROW(channel_id, asset)`. A pooled contract balance is permitted only if the adapter enforces equivalent per-channel liabilities and aggregate conservation.

---

## Config

| Knob | Role |
|---|---|
| `PROTOCOL_VERSION` | Fixed encoding version (`0` for this spec). |
| `CHAIN_ID` / `PROGRAM_ID` | Deployment identity included in every signed/hash domain. |
| `CLOSE_CHALLENGE_PERIOD` | Time after a Close / AdoptTerms request during which old-version vouchers may settle. Must be non-zero. |
| `ORGANIZATION_DEPOSIT` | Reserved from owner on org create; unreserved on delete. |
| `SERVICE_DEPOSIT` | Reserved from service owner on create; unreserved on delete. |
| `MAX_ORG_MEMBERS` | Cap on member list at org create (owner counts as 1). |
| `MAX_NAME_LENGTH` | Bound on `name`. |
| `MAX_METADATA_LENGTH` | Bound on `metadata`. |

`minimum_calls` and `expiration_threshold` are per-service, not global.

---

## Records

```
Organization {
  id:        Hash256          // hash_org_id(owner, name)
  owner:     AccountId
  name:      Bytes
  metadata:  Bytes
  members:   u32              // including owner
  services:  u32
  paused:    bool             // admission hold; default false
}

// storage: Members[(org_id, account)] -> Rank
// owner rank = 0; other members start at 1. Only members may create services.

Service {
  id:                    Hash256   // hash_service_id(service_owner, name)
  owner:                 AccountId // receives claims
  org_owner:             AccountId // parent org owner (lookup key)
  receipt_signer:        AccountId // signs off-chain delivery receipts
  organization:          Hash256
  name:                  Bytes
  metadata:              Bytes
  version:               u32       // starts at 1; ++ on every update
  asset:                 AssetId   // immutable payment asset
  price:                 Balance   // per call
  minimum_calls:         u32
  expiration_threshold:  Time
  trials:                u32       // stored; not enforced in v0
  channels:              u32       // open channels against this service
  paused:                bool      // admission hold; default false
}

ChannelStatus =
  Open
  | Closing {
      requested_at: Time,
      close_after:  Time,
      action:
        Close
        | AdoptTerms {
            calls: u32,                 // new remaining quota at new terms
            version: u32,
            receipt_signer: AccountId,
            asset: AssetId,
            price: Balance,
            expiration_threshold: Time,
            funds: Balance
          }
    }

Channel {
  id:            Hash256    // hash_channel_id(payer, org_id, service_id, nonce)
  owner:         AccountId  // payer
  nonce:         ChannelNonce // protocol-assigned monotonic payer nonce
  organization:  Hash256
  service:       Hash256
  version:       u32        // service version snapshotted at open / AdoptTerms
  receipt_signer: AccountId // service receipt key snapshotted at open / AdoptTerms
  asset:         AssetId    // service asset snapshotted at open / AdoptTerms
  price:         Balance    // snapshotted at open / AdoptTerms
  calls:         u32        // prepaid ceiling; grows on fund / AdoptTerms
  counter:       u32        // highest settled count; starts 0; never resets
  expiration:    Time       // refreshed on fund / AdoptTerms
  status:        ChannelStatus
}
```

The nonce permits parallel channels for the same payer/service (normally one per agent). `NextChannelNonce[payer]` only increases, so a closed channel id cannot reopen and old vouchers cannot replay. Live service edits do **not** rewrite open channels. `counter` never resets: same-term top-ups raise `calls`; terms changes keep `counter` and bind new vouchers to the new `version`.

If `calls` would exceed `u32` max, open a new channel with the next payer nonce.

```
require channel.counter ≤ channel.calls
remaining = checked_mul(channel.price, channel.calls − channel.counter)
active ESCROW balance == remaining
if status is Closing/AdoptTerms:
  PENDING_ESCROW balance == status.action.funds
```

Admission: `org.paused` or `svc.paused` blocks `create_service`, `open_channel`, and `AdoptTerms`. It does **not** bump `version`, freeze existing Open channels, or block `fund_channel`, claims, Close, cancel, or finalize. Pause is not a substitute for challenged close.

All arithmetic is checked. Overflow/underflow aborts the whole transaction; never saturate or partially update state.

---

## Storage

```
Organizations[(owner, org_id)]       -> Organization
Members[(org_id, account)]           -> Rank
Services[(org_id, service_id)]       -> Service
Channels[(payer, channel_id)]        -> Channel
NextChannelNonce[payer]              -> ChannelNonce
ESCROW(channel_id, asset)            -> isolated program-owned escrow
PENDING_ESCROW(channel_id)           -> pre-funded AdoptTerms escrow (Closing only)
```

---

## Errors

```
OrganizationExists, OrganizationNotFound, OrganizationNotOwner, OrganizationHasServices
OrganizationPaused
ServiceExists, ServiceNotFound, ServiceNotOwner, ServiceNotOrgMember, ServiceHasOpenChannels
ServicePaused
ChannelExists, ChannelNonceMismatch, ChannelNotFound, ChannelNotOwner, ChannelLowNumberOfCalls
ChannelInvalidExpiration
ChannelNotOpen, ChannelNotClosing, ChallengeNotElapsed, InvalidChallengePeriod
ChannelTermsChanged, ChannelTermsUnchanged
ClaimLowCounter, ClaimCounterTooHigh, ClaimNotEnoughFunds, ClaimInvalidSignature
InsufficientFunds, ArithmeticOverflow, AssetMismatch
TooManyMembers, NameTooLong, MetadataTooLong
```

---

## Events

```
OrganizationCreated { id, owner, members }
OrganizationPausedSet { id, owner, paused }
OrganizationDeleted { id, owner }

ServiceCreated { id, owner, organization, asset, price, receipt_signer }
ServiceUpdated { id, owner, organization, version }
ServicePausedSet { id, owner, organization, paused }
ServiceDeleted { id, owner, organization }

ChannelCreated { id, owner, nonce, organization, service, version, asset, calls, funds, expiration }
ChannelFunded { id, owner, additional_calls, calls, funds, expiration }
ChannelTransitionRequested { id, owner, action, close_after }
ChannelTransitionCancelled { id, owner }
ChannelTermsAdopted { id, owner, version, calls, funds, expiration }
ChannelClaimed { id, by, counter, funds }          // voucher settle (funds > 0)
ChannelClosed { id, by, funds }                    // finalized refund
ChannelDeleted { id, by, funds }                   // exhausted cleanup
```
