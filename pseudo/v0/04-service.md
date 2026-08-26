# Service

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md).

`update_service` is a **hard cut for new terms**: every edit increments `version`. Open channel epochs keep their `price`, `version`, `asset`, and `receipt_signer` snapshots. Outstanding vouchers settle during the transition challenge (see 05–06). `asset` is immutable for a service; publish a new service to change payment asset.

---

## `create_service(org_owner, org_id, name, asset, receipt_signer, price, minimum_calls, expiration_threshold, trials, metadata)`

Caller must be a member of the organization. Caller becomes `service.owner` (claim recipient).

```
require len(name) <= MAX_NAME_LENGTH            else NameTooLong
require len(metadata) <= MAX_METADATA_LENGTH    else MetadataTooLong

org = Organizations[(org_owner, org_id)]        else OrganizationNotFound
require Members[(org_id, caller)] exists        else ServiceNotOrgMember
require expiration_threshold > 0                else ChannelInvalidExpiration

reserve(caller, SERVICE_DEPOSIT)                else InsufficientFunds

service_id = hash_service_id(caller, name)
require Services[(org_id, service_id)] missing  else ServiceExists

org.services = checked_add(org.services, 1)
Organizations[(org_owner, org_id)] = org

svc = Service {
  id: service_id, owner: caller, receipt_signer, organization: org_id,
  name, metadata,
  version: 1,
  asset, price, minimum_calls, expiration_threshold, trials,
  channels: 0
}
Services[(org_id, service_id)] = svc

emit ServiceCreated {
  id: service_id, owner: caller, organization: org_id,
  asset, price, receipt_signer
}
```

`trials` is stored and ignored by billing in v0.

---

## `update_service(org_owner, org_id, service_id, name?, receipt_signer?, price?, minimum_calls?, expiration_threshold?, trials?, metadata?)`

Caller = service owner. Omitted fields keep their current value. `version` always increments.

```
svc = Services[(org_id, service_id)]            else ServiceNotFound
require caller == svc.owner                     else ServiceNotOwner

if name:     require len(name) <= MAX_NAME_LENGTH;         svc.name = name
if metadata: require len(metadata) <= MAX_METADATA_LENGTH; svc.metadata = metadata
if expiration_threshold:
  require expiration_threshold > 0 else ChannelInvalidExpiration
svc.receipt_signer         = receipt_signer         ?? svc.receipt_signer
svc.price                  = price                  ?? svc.price
svc.minimum_calls          = minimum_calls          ?? svc.minimum_calls
svc.expiration_threshold   = expiration_threshold   ?? svc.expiration_threshold
svc.trials                 = trials                 ?? svc.trials

svc.version = checked_add(svc.version, 1)
Services[(org_id, service_id)] = svc

emit ServiceUpdated { id: service_id, owner: caller, organization: org_id, version: svc.version }
```

Does **not** rewrite open epochs or refund them. The spend broker freezes new calls when versions differ. The payer requests a challenged rollover/close; the provider settles old vouchers before finalization.

---

## `delete_service(org_owner, org_id, service_id)`

Caller = service owner. Requires no open channels.

```
org = Organizations[(org_owner, org_id)]        else OrganizationNotFound
svc = Services[(org_id, service_id)]            else ServiceNotFound
require caller == svc.owner                     else ServiceNotOwner
require svc.channels == 0                       else ServiceHasOpenChannels

org.services = checked_sub(org.services, 1)
Organizations[(org_owner, org_id)] = org

unreserve(caller, SERVICE_DEPOSIT)
delete Services[(org_id, service_id)]

emit ServiceDeleted { id: service_id, owner: caller, organization: org_id }
```
