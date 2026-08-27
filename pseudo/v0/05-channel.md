# Channel (open / fund / challenged close or adopt-terms)

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md), `require_admitting` from [`03-organization.md`](03-organization.md). Claim / close: [`06-claim.md`](06-claim.md).

---

## Helper

```
remaining(ch: Channel) -> Balance
  require ch.counter ≤ ch.calls
  return checked_mul(ch.price, ch.calls − ch.counter)
```

Always use channel snapshots. All transfers and state changes are atomic; any check/transfer failure rolls back the whole instruction.

---

## `open_channel(org_id, service_id, nonce, calls)`

Caller = `payer`. Client supplies the current protocol-assigned `NextChannelNonce[payer]` (normally maps this channel to one agent). Locks `service.price × calls` into isolated escrow and snapshots terms. Rejected while the organization or service is paused.

```
svc = Services[(org_id, service_id)]            else ServiceNotFound
org = Organizations[(svc.org_owner, org_id)]    else OrganizationNotFound
require_admitting(org, svc)
require calls >= svc.minimum_calls              else ChannelLowNumberOfCalls
require nonce == NextChannelNonce[payer]         else ChannelNonceMismatch

channel_id = hash_channel_id(payer, org_id, service_id, nonce)
require Channels[(payer, channel_id)] missing   else ChannelExists

funds = checked_mul(svc.price, calls)
create ESCROW(channel_id, svc.asset)
transfer(svc.asset, payer → ESCROW(channel_id), funds)
  else InsufficientFunds

svc.channels = checked_add(svc.channels, 1)
Services[(org_id, service_id)] = svc

ch = Channel {
  id: channel_id, owner: payer, nonce,
  organization: org_id, service: service_id,
  version: svc.version,
  receipt_signer: svc.receipt_signer,
  asset: svc.asset,
  price: svc.price,
  calls,
  counter: 0,
  expiration: checked_add(now(), svc.expiration_threshold),
  status: Open
}
Channels[(payer, channel_id)] = ch
NextChannelNonce[payer] = checked_add(nonce, 1)

emit ChannelCreated {
  id: channel_id, owner: payer, nonce, organization: org_id, service: service_id,
  version: svc.version, asset: svc.asset,
  calls, funds, expiration: ch.expiration
}
```

---

## `fund_channel(payer, channel_id, additional_calls)`

Caller = channel owner. Same-term top-up: raise the prepaid ceiling, lock more escrow, keep `counter` and `version`. No challenge window — unused credit stays locked and in-flight vouchers stay valid.

```
require caller == payer                         else ChannelNotOwner
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound
require ch.status == Open                       else ChannelNotOpen
require svc.version == ch.version               else ChannelTermsChanged
require svc.asset == ch.asset                   else AssetMismatch
require additional_calls > 0                    else ChannelLowNumberOfCalls

added = checked_mul(ch.price, additional_calls)
transfer(ch.asset, payer → ESCROW(channel_id), added)
  else InsufficientFunds

ch.calls = checked_add(ch.calls, additional_calls)
ch.expiration = max(ch.expiration, checked_add(now(), svc.expiration_threshold))
Channels[(payer, channel_id)] = ch

emit ChannelFunded {
  id: channel_id, owner: payer,
  additional_calls, calls: ch.calls, funds: added, expiration: ch.expiration
}
```

If live `service.version` differs, use `request_channel_transition(AdoptTerms)` instead. Do not re-price in place. Pause does not block `fund_channel`.

---

## `request_channel_transition(payer, channel_id, action)`

Caller = channel owner. `action` is `Close` or `AdoptTerms { calls }`.

This instruction moves the channel to `Closing`; off-chain delivery must stop. It does not refund or invalidate the current version. `AdoptTerms` is for a **terms change** (`service.version != channel.version`): it snapshots live terms and pre-funds a new remaining quota now, while the payer is authorizing the transaction. Providers may keep settling old-version vouchers until finalization. Same-term extra quota is `fund_channel`, not this instruction. `AdoptTerms` is rejected while the organization or service is paused; `Close` is not.

```
require caller == payer                         else ChannelNotOwner
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound
require ch.status == Open                       else ChannelNotOpen

if requested action is AdoptTerms { calls }:
  org = Organizations[(svc.org_owner, ch.organization)] else OrganizationNotFound
  require_admitting(org, svc)
  require svc.version != ch.version             else ChannelTermsUnchanged
  require calls >= svc.minimum_calls            else ChannelLowNumberOfCalls
  require svc.asset == ch.asset                 else AssetMismatch
  pending_funds = checked_mul(svc.price, calls)
  create PENDING_ESCROW(channel_id, svc.asset)
  transfer(svc.asset, payer → PENDING_ESCROW(channel_id), pending_funds)
    else InsufficientFunds
  action = AdoptTerms {
    calls,                                      // new remaining quota after finalize
    version: svc.version,
    receipt_signer: svc.receipt_signer,
    asset: svc.asset,
    price: svc.price,
    expiration_threshold: svc.expiration_threshold,
    funds: pending_funds
  }

requested_at = now()
close_after = checked_add(requested_at, CLOSE_CHALLENGE_PERIOD)
ch.status = Closing { requested_at, close_after, action }
Channels[(payer, channel_id)] = ch

emit ChannelTransitionRequested { id: channel_id, owner: payer, action, close_after }
```

The payer may request this at any time (expiry/version mismatch are common reasons). The non-zero challenge window, not transaction ordering, protects unclaimed vouchers.

---

## `cancel_channel_transition(payer, channel_id)`

Caller = channel owner. Restores `Open`. If AdoptTerms was pending, refunds its pre-funded next quota.

```
require caller == payer                         else ChannelNotOwner
ch = Channels[(payer, channel_id)]              else ChannelNotFound
closing = ch.status as Closing                  else ChannelNotClosing

if closing.action is AdoptTerms:
  pending = closing.action
  transfer(pending.asset,
           PENDING_ESCROW(channel_id) → ch.owner,
           pending.funds)
  close PENDING_ESCROW(channel_id)

ch.status = Open
Channels[(payer, channel_id)] = ch
emit ChannelTransitionCancelled { id: channel_id, owner: payer }
```

If the channel is already expired or version-stale, off-chain policy remains frozen even after cancellation.

---

## `finalize_channel_transition(payer, channel_id)`

Anyone may submit after the deadline. Reads the latest settled counter, so claims landed before finalization reduce the refund. AdoptTerms never debits the payer here: the new remaining quota was pre-funded in the owner-signed request. `counter` is not reset.

```
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound
closing = ch.status as Closing                  else ChannelNotClosing
require now() >= closing.close_after            else ChallengeNotElapsed

refund = remaining(ch)                          // old version / old price

if closing.action == Close:
  if refund > 0:
    transfer(ch.asset, ESCROW(channel_id) → ch.owner, refund)
  close ESCROW(channel_id)
  svc.channels = checked_sub(svc.channels, 1)
  Services[(ch.organization, ch.service)] = svc
  delete Channels[(payer, channel_id)]
  emit ChannelClosed { id: channel_id, by: caller, funds: refund }
  return

pending = closing.action as AdoptTerms
if refund > 0:
  transfer(ch.asset, ESCROW(channel_id) → ch.owner, refund)
transfer(pending.asset,
         PENDING_ESCROW(channel_id) → ESCROW(channel_id),
         pending.funds)
close PENDING_ESCROW(channel_id)

ch.version        = pending.version
ch.receipt_signer = pending.receipt_signer
ch.asset          = pending.asset
ch.price          = pending.price
ch.calls          = checked_add(ch.counter, pending.calls)
ch.expiration     = checked_add(now(), pending.expiration_threshold)
ch.status         = Open
Channels[(payer, channel_id)] = ch

emit ChannelTermsAdopted {
  id: channel_id, owner: ch.owner, version: ch.version,
  calls: ch.calls, funds: pending.funds, expiration: ch.expiration
}
```

`counter` keeps its latest settled value. New vouchers must use the new `version` and continue from `counter + 1`. Old-version vouchers no longer verify. If service terms change again during the challenge, the finalized channel still uses the requested snapshot and the broker freezes until it matches live terms or another AdoptTerms occurs.
