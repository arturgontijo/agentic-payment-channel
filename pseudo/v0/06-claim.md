# Claim / close

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md), `remaining()` from [`05-channel.md`](05-channel.md).

Anyone may call. Voucher payout always goes to `service.owner`, not the tx sender.

Until expiry **and** while `channel.version == service.version`, the payer cannot unilaterally withdraw. That is the provider’s claim window.

After expiry **or** a version bump, path C still works (vouchers bind to `channel.version`). Path B also unlocks: the payer may take `remaining` (on-chain `counter`, not an unclaimed voucher). First successful tx wins. Claim before expiry and before `update_service`.

---

## `claim_channel_funds(payer, channel_id, counter?, signature?)`

`counter` defaults to `0`. `signature` is required only on path C.

```
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound

left           = remaining(ch)
expired        = ch.expiration <= now()
version_changed = ch.version != svc.version
n              = counter ?? 0

// ---- Path A: nothing left to claim ------------------------------------------
if left == 0:
  svc.channels -= 1
  Services[(ch.organization, ch.service)] = svc
  delete Channels[(payer, channel_id)]
  emit ChannelDeleted { id: channel_id, by: caller, funds: 0 }
  return

// ---- Path B: payer refund (expiry or version mismatch) ----------------------
if caller == ch.owner:
  if expired or version_changed:
    transfer(VAULT → ch.owner, left)
    svc.channels -= 1
    Services[(ch.organization, ch.service)] = svc
    delete Channels[(payer, channel_id)]
    emit ChannelExpiredClaimed { id: channel_id, by: caller, funds: left }
    return
  else if n == 0:
    fail ClaimNotExpired
  // else fall through to path C (payer may settle a voucher on the provider's behalf)

// ---- Path C: voucher settlement ---------------------------------------------
require n > ch.counter                          else ClaimLowCounter
sig = signature                                 else ClaimInvalidSignature
verify_voucher(ch.owner, channel_id, ch.version, n, sig)
  // channel snapshot, not live svc.version

claim = ch.price × (n − ch.counter)             // snapshotted price
max   = ch.price × ch.calls
if claim > max: claim = max
require claim <= left                           else ClaimNotEnoughFunds

transfer(VAULT → svc.owner, claim)

ch.counter = n
Channels[(payer, channel_id)] = ch
Services[(ch.organization, ch.service)] = svc   // unchanged counts; persist if target requires it

emit ChannelClaimed { id: channel_id, by: caller, counter: n, funds: claim }
```

Channel stays open after path C if `remaining(ch) > 0`. Provider can claim incrementally (keep the highest voucher, cash out any time before expiry).

A signed `n > calls` cannot overdraw: `claim` is capped and then checked against `left`.
