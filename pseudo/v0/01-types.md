# Types, config, errors, events

Primitives: `AccountId`, `Hash256`, `Balance`, `Time` (block / slot / unix — target-defined), `Bytes`, `Rank = u32`.

`VAULT` is a single program-owned account. All channel funds go here.

---

## Config

| Knob | Role |
|---|---|
| `DOMAIN` | Domain prefix for hashes and vouchers. Stable per deployment. |
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
  id:        Hash256          // hash_name(owner, name)
  owner:     AccountId
  name:      Bytes
  metadata:  Bytes
  members:   u32              // including owner
  services:  u32
}

// storage: Members[(org_id, account)] -> Rank
// owner rank = 0; other members start at 1. Only members may create services.

Service {
  id:                    Hash256   // hash_name(service_owner, name)
  owner:                 AccountId // receives claims
  organization:          Hash256
  name:                  Bytes
  metadata:              Bytes
  version:               u32       // starts at 1; ++ on every update
  price:                 Balance   // per call
  minimum_calls:         u32
  expiration_threshold:  Time
  trials:                u32       // stored; not enforced in v0
  channels:              u32       // open channels against this service
}

Channel {
  id:            Hash256    // hash_channel_id(payer, org_id, service_id)
  owner:         AccountId  // payer
  organization:  Hash256
  service:       Hash256
  version:       u32        // snapshotted at open / update
  price:         Balance    // snapshotted at open / update
  calls:         u32        // prepaid quota
  counter:       u32        // highest settled count; starts 0
  expiration:    Time       // now() + service.expiration_threshold
}
```

At most one channel per `(payer, org, service)`. Live service edits do **not** rewrite open channels.

```
funds     = channel.price × channel.calls
remaining = channel.price × (channel.calls − channel.counter)   // 0 if counter > calls
```

---

## Storage

```
Organizations[(owner, org_id)]       -> Organization
Members[(org_id, account)]           -> Rank
Services[(org_id, service_id)]       -> Service
Channels[(payer, channel_id)]        -> Channel
VAULT                                -> program-owned escrow
```

---

## Errors

```
OrganizationExists, OrganizationNotFound, OrganizationNotOwner, OrganizationHasServices
ServiceExists, ServiceNotFound, ServiceNotOwner, ServiceNotOrgMember, ServiceHasOpenChannels
ChannelExists, ChannelNotFound, ChannelNotOwner, ChannelLowNumberOfCalls
ClaimNotExpired, ClaimLowCounter, ClaimNotEnoughFunds, ClaimInvalidSignature
InsufficientFunds
TooManyMembers, NameTooLong, MetadataTooLong
```

Reserved (unused in v0 instructions, keep for ports): `ChannelInvalidExpiration`, `ClaimNotAllowed`, `ClaimInvalidSigner`, `InvalidBlockNumber`.

---

## Events

```
OrganizationCreated { id, owner, members }
OrganizationDeleted { id, owner }

ServiceCreated { id, owner, organization, price }
ServiceUpdated { id, owner, organization, version }
ServiceDeleted { id, owner, organization }

ChannelCreated { id, owner, organization, service, version, calls, funds, expiration }
ChannelUpdated { id, owner, organization, service, version, calls, funds, expiration }
ChannelClaimed { id, by, counter, funds }             // voucher settle
ChannelExpiredClaimed { id, by, funds }               // payer refund
ChannelDeleted { id, by, funds }                      // remaining == 0
```
