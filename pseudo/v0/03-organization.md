# Organization

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md).

Only members of an organization may create services under it. Owner is always a member (rank 0).

---

## `create_organization(name, members, metadata)`

Caller = `owner`.

```
require len(name) <= MAX_NAME_LENGTH            else NameTooLong
require len(metadata) <= MAX_METADATA_LENGTH    else MetadataTooLong

listed = members or []
listed = unique(listed excluding owner)
members_count = checked_add(1, len(listed))
require members_count <= MAX_ORG_MEMBERS        else TooManyMembers

reserve(owner, ORGANIZATION_DEPOSIT)            else InsufficientFunds

org_id = hash_org_id(owner, name)
require Organizations[(owner, org_id)] missing  else OrganizationExists

for each m in listed:
  Members[(org_id, m)] = 1

Members[(org_id, owner)] = 0

org = Organization {
  id: org_id, owner, name, metadata,
  members: members_count,
  services: 0,
  paused: false
}
Organizations[(owner, org_id)] = org

emit OrganizationCreated { id: org_id, owner, members: org.members }
```

---

## `delete_organization(name)`

Caller = `owner`. Requires no leftover services (hence no leftover channels).

```
org_id = hash_org_id(owner, name)
org = Organizations[(owner, org_id)]            else OrganizationNotFound
require caller == org.owner                     else OrganizationNotOwner
require org.services == 0                       else OrganizationHasServices

unreserve(owner, ORGANIZATION_DEPOSIT)
delete Organizations[(owner, org_id)]
delete all Members[(org_id, _)]

emit OrganizationDeleted { id: org_id, owner }
```

---

## Helper: `require_admitting(org, svc)`

```
require not org.paused                          else OrganizationPaused
require not svc.paused                          else ServicePaused
```

Used by `create_service` (org only), `open_channel`, and `AdoptTerms`. Close / fund / claim / finalize ignore this helper.

---

## `set_organization_paused(name, paused)`

Caller = `owner`. Stops **new** services and channels under the org, and blocks `AdoptTerms`. Existing channels keep their snapshots: claims, `fund_channel`, and Close still work. Does **not** bump any service `version`. Unpause by calling with `paused = false`.

```
org_id = hash_org_id(owner, name)
org = Organizations[(owner, org_id)]            else OrganizationNotFound
require caller == org.owner                     else OrganizationNotOwner

org.paused = paused
Organizations[(owner, org_id)] = org
emit OrganizationPausedSet { id: org_id, owner, paused }
```

To shut down: pause the org, wait for payers to close or exhaust channels, delete each empty service, then `delete_organization`. Pause is the admission gate; delete still requires `services == 0`.
