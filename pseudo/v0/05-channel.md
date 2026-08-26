# Channel (open / challenged close or rollover)

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md). Claim / close: [`06-claim.md`](06-claim.md).

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

Caller = `payer`. Client supplies the current protocol-assigned `NextChannelNonce[payer]` (normally maps this channel to one agent). Locks `service.price × calls` into isolated escrow and snapshots terms.

```
svc = Services[(org_id, service_id)]            else ServiceNotFound
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
  epoch: 1,
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
  epoch: 1, version: svc.version, asset: svc.asset,
  calls, funds, expiration: ch.expiration
}
```

---

## `request_channel_transition(payer, channel_id, action)`

Caller = channel owner. `action` is `Close` or `Rollover { calls }`.

This instruction moves the channel to `Closing`; off-chain delivery must stop. It does not refund/invalidate the current epoch. A rollover pre-funds and snapshots the next epoch now, while the payer is authorizing the transaction. Providers may keep settling the old epoch until finalization.

```
require caller == payer                         else ChannelNotOwner
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound
require ch.status == Open                       else ChannelNotOpen

if requested action is Rollover { calls }:
  require calls >= svc.minimum_calls            else ChannelLowNumberOfCalls
  require svc.asset == ch.asset                 else AssetMismatch
  next_epoch = checked_add(ch.epoch, 1)
  pending_funds = checked_mul(svc.price, calls)
  create PENDING_ESCROW(channel_id, next_epoch, svc.asset)
  transfer(svc.asset, payer → PENDING_ESCROW(channel_id, next_epoch), pending_funds)
    else InsufficientFunds
  action = Rollover {
    calls,
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

Caller = channel owner. Restores `Open`. If rollover was pending, refunds its pre-funded next epoch.

```
require caller == payer                         else ChannelNotOwner
ch = Channels[(payer, channel_id)]              else ChannelNotFound
closing = ch.status as Closing                  else ChannelNotClosing

if closing.action is Rollover:
  pending = closing.action
  next_epoch = checked_add(ch.epoch, 1)
  transfer(pending.asset,
           PENDING_ESCROW(channel_id, next_epoch) → ch.owner,
           pending.funds)
  close PENDING_ESCROW(channel_id, next_epoch)

ch.status = Open
Channels[(payer, channel_id)] = ch
emit ChannelTransitionCancelled { id: channel_id, owner: payer }
```

If the channel is already expired or version-stale, off-chain policy remains frozen even after cancellation.

---

## `finalize_channel_transition(payer, channel_id)`

Anyone may submit after the deadline. Reads the latest settled counter, so claims landed before finalization reduce the refund. Rollover never debits the payer here: next epoch was pre-funded in the owner-signed request.

```
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound
closing = ch.status as Closing                  else ChannelNotClosing
require now() >= closing.close_after            else ChallengeNotElapsed

refund = remaining(ch)                          // old epoch / old price

if closing.action == Close:
  if refund > 0:
    transfer(ch.asset, ESCROW(channel_id) → ch.owner, refund)
  close ESCROW(channel_id)
  svc.channels = checked_sub(svc.channels, 1)
  Services[(ch.organization, ch.service)] = svc
  delete Channels[(payer, channel_id)]
  emit ChannelClosed { id: channel_id, by: caller, funds: refund }
  return

pending = closing.action as Rollover
next_epoch = checked_add(ch.epoch, 1)
if refund > 0:
  transfer(ch.asset, ESCROW(channel_id) → ch.owner, refund)
transfer(pending.asset,
         PENDING_ESCROW(channel_id, next_epoch) → ESCROW(channel_id),
         pending.funds)
close PENDING_ESCROW(channel_id, next_epoch)

ch.epoch          = next_epoch
ch.version        = pending.version
ch.receipt_signer = pending.receipt_signer
ch.asset          = pending.asset
ch.price          = pending.price
ch.calls          = pending.calls
ch.counter        = 0
ch.expiration     = checked_add(now(), pending.expiration_threshold)
ch.status         = Open
Channels[(payer, channel_id)] = ch

emit ChannelRolledOver {
  id: channel_id, owner: ch.owner, epoch: ch.epoch, version: ch.version,
  calls: pending.calls, funds: pending.funds, expiration: ch.expiration
}
```

A finalized rollover is the only counter reset. Because epoch increments, old vouchers cannot replay. If service terms change again during the challenge, the finalized channel still uses the requested snapshot and the broker freezes until it matches live terms or another rollover occurs.
