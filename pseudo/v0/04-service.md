# Service

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md).

`update_service` is a **hard cut for new terms**: every edit increments `version`. Open channels keep their snapshot. Outstanding vouchers still settle at `channel.price` / `channel.version` (see 06). The payer may refund unused remaining immediately, or `update_channel` to re-lock (see 05).

---

## `create_service(org_owner, org_id, name, price, minimum_calls, expiration_threshold, trials, metadata)`

Caller must be a member of the organization. Caller becomes `service.owner` (claim recipient).

```
require len(name) <= MAX_NAME_LENGTH            else NameTooLong
require len(metadata) <= MAX_METADATA_LENGTH    else MetadataTooLong

org = Organizations[(org_owner, org_id)]        else OrganizationNotFound
require Members[(org_id, caller)] exists        else ServiceNotOrgMember

reserve(caller, SERVICE_DEPOSIT)                else InsufficientFunds

service_id = hash_name(caller, name)
require Services[(org_id, service_id)] missing  else ServiceExists

org.services += 1
Organizations[(org_owner, org_id)] = org

svc = Service {
  id: service_id, owner: caller, organization: org_id,
  name, metadata,
  version: 1,
  price, minimum_calls, expiration_threshold, trials,
  channels: 0
}
Services[(org_id, service_id)] = svc

emit ServiceCreated { id: service_id, owner: caller, organization: org_id, price }
```

`trials` is stored and ignored by billing in v0.

---

## `update_service(org_owner, org_id, service_id, name?, price?, minimum_calls?, expiration_threshold?, trials?, metadata?)`

Caller = service owner. Omitted fields keep their current value. `version` always increments.

```
svc = Services[(org_id, service_id)]            else ServiceNotFound
require caller == svc.owner                     else ServiceNotOwner

if name:     require len(name) <= MAX_NAME_LENGTH;         svc.name = name
if metadata: require len(metadata) <= MAX_METADATA_LENGTH; svc.metadata = metadata
svc.price                  = price                  ?? svc.price
svc.minimum_calls          = minimum_calls          ?? svc.minimum_calls
svc.expiration_threshold   = expiration_threshold   ?? svc.expiration_threshold
svc.trials                 = trials                 ?? svc.trials

svc.version += 1
Services[(org_id, service_id)] = svc

emit ServiceUpdated { id: service_id, owner: caller, organization: org_id, version: svc.version }
```

Does **not** rewrite open channels and does **not** refund them. Stop accepting new calls off-chain until payers `update_channel`. Claim outstanding vouchers **before** bumping (path B races the remainder).

---

## `delete_service(org_owner, org_id, service_id)`

Caller = service owner. Requires no open channels.

```
org = Organizations[(org_owner, org_id)]        else OrganizationNotFound
svc = Services[(org_id, service_id)]            else ServiceNotFound
require caller == svc.owner                     else ServiceNotOwner
require svc.channels == 0                       else ServiceHasOpenChannels

org.services -= 1
Organizations[(org_owner, org_id)] = org

unreserve(caller, SERVICE_DEPOSIT)
delete Services[(org_id, service_id)]

emit ServiceDeleted { id: service_id, owner: caller, organization: org_id }
```
