# Channel (open / update)

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md). Claim / close: [`06-claim.md`](06-claim.md).

---

## Helper

```
remaining(ch: Channel) -> Balance
  if ch.counter > ch.calls: return 0
  return ch.price × (ch.calls − ch.counter)
```

Always use `ch.price` (snapshot), never live `service.price`.

---

## `open_channel(org_id, service_id, calls)`

Caller = `payer`. Locks `service.price × calls` into `VAULT`. Snapshots terms.

```
svc = Services[(org_id, service_id)]            else ServiceNotFound
require calls >= svc.minimum_calls              else ChannelLowNumberOfCalls

channel_id = hash_channel_id(payer, org_id, service_id)
require Channels[(payer, channel_id)] missing   else ChannelExists

funds = svc.price × calls
transfer(payer → VAULT, funds)                  else InsufficientFunds

svc.channels += 1
Services[(org_id, service_id)] = svc

ch = Channel {
  id: channel_id, owner: payer,
  organization: org_id, service: service_id,
  version: svc.version,
  price: svc.price,
  calls,
  counter: 0,
  expiration: now() + svc.expiration_threshold
}
Channels[(payer, channel_id)] = ch

emit ChannelCreated {
  id: channel_id, owner: payer, organization: org_id, service: service_id,
  version: svc.version, calls, funds, expiration: ch.expiration
}
```

---

## `update_channel(payer, channel_id, calls?)`

Caller must be the channel owner. Replaces the escrow at **current** service terms. Same `channel_id`.

Use after a service version bump (to keep consuming) or to resize the quota.

```
require caller == payer                         else ChannelNotOwner
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound

// 1. refund unused quota at the OLD snapshot
refund = remaining(ch)
if refund > 0:
  transfer(VAULT → payer, refund)

// 2. new quota at live terms
N = calls ?? svc.minimum_calls
require N >= svc.minimum_calls                  else ChannelLowNumberOfCalls

funds = svc.price × N
transfer(payer → VAULT, funds)                  else InsufficientFunds

// 3. persist — counter resets: old vouchers are bound to the previous version
ch.version    = svc.version
ch.price      = svc.price
ch.calls      = N
ch.counter    = 0
ch.expiration = now() + svc.expiration_threshold
Channels[(payer, channel_id)] = ch

emit ChannelUpdated {
  id: channel_id, owner: payer,
  organization: ch.organization, service: ch.service,
  version: svc.version, calls: N, funds, expiration: ch.expiration
}
```

This is a **replace**, not a delta top-up: payer is refunded the old remainder, then locks a full `N` at the live price. `svc.channels` is unchanged (same channel). The refund races unclaimed vouchers — same as 06 path B.
