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
require 1 + len(listed) <= MAX_ORG_MEMBERS      else TooManyMembers
  // owner counts as 1; listed should not include owner (if it does, rank is overwritten to 0)

reserve(owner, ORGANIZATION_DEPOSIT)            else InsufficientFunds

org_id = hash_name(owner, name)
require Organizations[(owner, org_id)] missing  else OrganizationExists

for each m in listed:
  Members[(org_id, m)] = 1

Members[(org_id, owner)] = 0

org = Organization {
  id: org_id, owner, name, metadata,
  members: 1 + len(listed_without_owner),
  services: 0
}
Organizations[(owner, org_id)] = org

emit OrganizationCreated { id: org_id, owner, members: org.members }
```

---

## `delete_organization(name)`

Caller = `owner`. Requires no leftover services (hence no leftover channels).

```
org_id = hash_name(owner, name)
org = Organizations[(owner, org_id)]            else OrganizationNotFound
require caller == org.owner                     else OrganizationNotOwner
require org.services == 0                       else OrganizationHasServices

unreserve(owner, ORGANIZATION_DEPOSIT)
delete Organizations[(owner, org_id)]
delete all Members[(org_id, _)]

emit OrganizationDeleted { id: org_id, owner }
```
