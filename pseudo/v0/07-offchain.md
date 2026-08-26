# Off-chain protocol

The program never sees individual calls, payloads, API keys, or receipts. It is escrow + voucher verify + versioned pricing.

Voucher and receipt layouts: [`02-ids.md`](02-ids.md). Receipts are **vault / delivery** only. On-chain claim still uses the highest payer voucher.

---

## Consume (per billed call)

`n = 1` needs no receipt. Every later voucher requires a provider receipt for `n − 1`.

```
agent                         vault                         provider
  |  next_voucher()              |                              |
  |  (receipt if n > 1)          |                              |
  |----------------------------->|                              |
  |                              |  if dispensed > acked:       |
  |                              |    verify_receipt(…)         |
  |                              |    acked = dispensed         |
  |                              |  n = dispensed + 1           |
  |                              |  return (n, sig(n))          |
  |  (n, sig(n))                 |                              |
  |<-----------------------------|                              |
  |  HTTP + voucher (n, sig(n))  |                              |
  |------------------------------------------------------------>|
  |                              |                              |  require n == last_accepted + 1
  |                              |                              |  verify_voucher(payer, …)
  |                              |                              |  serve; if fail, error (no receipt)
  |                              |                              |  else receipt = sign_receipt(n, call_hash)
  |                              |                              |  keep highest sig for chain claim
  |  { result, receipt }         |                              |  both or neither
  |<------------------------------------------------------------|
  |  next_voucher(receipt)       |                              |
  |----------------------------->|  …                           |
```

### Vault

```
acked[(agent, channel)]      = 0    # last counter with a valid receipt
dispensed[(agent, channel)]  = 0    # last voucher handed out
# invariant: dispensed == acked  or  dispensed == acked + 1

next_voucher(capability_id, channel_id, receipt?) -> (n, signature)
  require capability live, channel allowlisted
  require channel.version == capability pin else DenyVersion

  if dispensed > acked:
    r = receipt else DenyNeedReceipt
    require r.counter == dispensed
            r.channel_id == channel_id
            r.version == channel.version
            else DenyReceipt
    verify_receipt(capability.receipt_signer, …) else DenyReceipt
    acked = dispensed                          # idempotent if retried with same r

  n = dispensed + 1
  require n ≤ max_counter and n ≤ channel.calls else DenyCap
  require window has room else DenyWindow
  require rung n in current tranche else DenyEmptyTill

  dispensed = n
  return (n, sig(n))                           # exactly one rung; agent does not pick n
```

`submit_receipt(receipt)` is the same verify/ack **without** dispensing, for end-of-job.

Re-submit of a receipt for the current outstanding `dispensed` is idempotent (lost HTTP). A receipt for an already-acked counter is ignored (no extra dispense). A receipt for any other counter is `DenyReceipt`.

### Provider (delivery)

A **billed success** is one message: result **and** receipt. No receipt → not a successful call (even if a body was produced). Do not serve now and attach a receipt later except **re-issue** of the same receipt after a dropped response.

```
expected = last_accepted + 1                   # starts at 1
on request (n, voucher_sig, body):
  require n == expected else reject jump       # no receipt, not billed
  verify_voucher(payer, channel_id, ch.version, n, voucher_sig)
    else reject                                # no receipt, not billed

  result = serve(body)                         # if serve fails: error, no receipt, n unused
  if serve failed:
    return error                               # agent may retry same n / same voucher

  last_accepted = n
  call_hash = H(billed request)
  receipt_sig = sign(receipt_digest(channel_id, ch.version, n, call_hash))
  keep highest voucher for on-chain claim
  persist (channel_id, n) → call_hash          # for re-issue

  return {                                     # atomic: both or neither
    result,
    receipt: { channel_id, version, n, call_hash, signature: receipt_sig }
  }
```

**Agent:** treat a 2xx/result **without** a verifiable `receipt` as failure. Persist `result` and `receipt` together; submit the receipt to the vault before the next `next_voucher`. Do not use `result` as “call complete” until the receipt is in hand.

**Re-issue:** `GET receipt(channel_id, n)` returns the **same** `{ n, call_hash, signature }` already issued. Never a new `call_hash` for that `n`.

On-chain the provider still submits the **highest** voucher (`sig(5)` cashes calls 1..5). Receipts never go on-chain.

---

## Rules

- Sequential **fetch**: vault `n = dispensed + 1`, and `n > 1` only after receipt for `n − 1`.
- Sequential **delivery**: provider `n == last_accepted + 1`. Reject jumps.
- **Receipt-in-response**: billed success **must** include `result` and `receipt` in the same message. Missing receipt ⇒ incomplete call; agent retries or re-fetches the receipt, and the vault will not dispense `n+1`.
- `lookahead` is **1**. Receipt-gating and prefetch conflict; do not dump `sig(1)…sig(5)`.
- Window caps (e.g. 100/day) count `next_voucher` successes, not a single `sig(100)`.
- After `update_service`, freeze new dispenses (version pin). Outstanding receipts/vouchers for the old snapshot still settle as in RESEARCH.md.
- Quota exhausted (`n > channel.calls`): `update_channel` or a new service.

---

## What receipts do not prevent

A **colluding** provider can sign receipts without serving. The vault will then walk the ladder. Receipts stop a **runaway agent against an honest provider** (cannot obtain `sig(n+1)` without call `n`). They do not replace the on-chain lock or small tranches.

---

## Provider watcher

Claim before `ch.expiration`, and **before** `update_service` (after a bump, path B races the remainder):

```
on timer or on each N new vouchers:
  submit claim_channel_funds(payer, channel_id, highest.counter, highest.signature)
```

Do not wait until close. After expiry the payer can take `remaining` even if a voucher exists.

---

## Payer watcher

```
if now() >= ch.expiration or ch.version != live svc.version:
  submit claim_channel_funds(payer, channel_id, none, none)
  // path B: refund remaining to payer
```

---

## Discovery

`Organization.metadata` / `Service.metadata` are the registry (endpoint, model id, schema, rate limits, **receipt key** if not the service owner). Index `ServiceCreated` / `ServiceUpdated` events. Not on-chain logic.
