# Voucher claim / exhausted cleanup

Requires: [`01-types.md`](01-types.md), [`02-ids.md`](02-ids.md), `remaining()` from [`05-channel.md`](05-channel.md).

Anyone may call. Voucher payout always goes to `service.owner`, never the submitter.

Claims remain valid while a channel is `Open` **or** `Closing`, including after service version changes and expiry. A payer never receives an immediate refund: close/AdoptTerms uses the challenge state machine in [`05-channel.md`](05-channel.md). Same-term top-up is `fund_channel` and does not freeze claims.

---

## `claim_channel_funds(payer, channel_id, counter?, signature?)`

`counter` defaults to `0`. `signature` is required only for voucher settlement.

```
ch  = Channels[(payer, channel_id)]             else ChannelNotFound
svc = Services[(ch.organization, ch.service)]   else ServiceNotFound

left = remaining(ch)
n    = counter ?? 0

// ---- Path A: exhausted cleanup ----------------------------------------------
if left == 0:
  // Preserve a requested AdoptTerms: finalization will fund the new remaining quota.
  if ch.status is Closing { action: AdoptTerms }:
    return

  svc.channels = checked_sub(svc.channels, 1)
  Services[(ch.organization, ch.service)] = svc
  close ESCROW(channel_id)
  delete Channels[(payer, channel_id)]
  emit ChannelDeleted { id: channel_id, by: caller, funds: 0 }
  return

// ---- Path B: voucher settlement ---------------------------------------------
require n > 0                                   else ClaimLowCounter
require n <= ch.calls                           else ClaimCounterTooHigh
require n >= ch.counter                         else ClaimLowCounter
sig = signature                                 else ClaimInvalidSignature
verify_voucher(ch.owner, channel_id, ch.version, n, sig)
  // channel version snapshot, never live svc.version

if n == ch.counter:
  return                                        // idempotent: already settled, zero claim

delta = checked_sub(n, ch.counter)
claim = checked_mul(ch.price, delta)            // snapshotted price
require claim <= left                           else ClaimNotEnoughFunds

transfer(ch.asset, ESCROW(channel_id) → svc.owner, claim)

ch.counter = n
Channels[(payer, channel_id)] = ch

emit ChannelClaimed {
  id: channel_id, by: caller, counter: n, funds: claim
}
```

Channel stays present after voucher settlement, even if `counter == calls`; a later call with no voucher performs exhausted cleanup. For `Closing/AdoptTerms`, exhausted state remains until transition finalization.

Re-submitting the exact already-settled voucher is a zero-payout no-op after signature verification. There is no cap-and-continue behavior: `n > calls`, overflow, asset mismatch, or insufficient escrow aborts. State and balances update atomically. `counter` never decreases.
