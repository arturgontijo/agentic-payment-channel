# Off-chain protocol

The program never sees individual calls, payloads, API keys, or receipts. It is escrow + voucher verification + fund + challenged transitions.

Voucher and receipt layouts: [`02-ids.md`](02-ids.md). Receipts are **spend-broker / delivery** only. On-chain claim still uses the highest payer voucher.

---

## Consume (per billed call)

The agent calls a **spend broker**, not the provider and not a signature endpoint. The broker holds the pre-signed ladder/KMS policy, attaches the next voucher, verifies `{ result, receipt }`, persists both, then returns only the result. The agent never receives a bearer voucher.

```
agent                    spend broker                    provider/gateway
  | execute(cap, channel,       |                               |
  | idempotency_key, request)   |                               |
  |---------------------------->|                               |
  |                             | require channel Open          |
  |                             | n = dispensed + 1             |
  |                             | attach sig(version, n)        |
  |                             |------------------------------>|
  |                             |                               | require n == expected
  |                             |                               | verify voucher
  |                             |                               | serve once
  |                             |                               | persist result+receipt
  |                             | { result, receipt }           |
  |                             |<------------------------------|
  |                             | verify request/result hashes  |
  |                             | atomically persist ack+result |
  | result                      |                               |
  |<----------------------------|                               |
```

### Spend broker

```
acked[(agent, channel)]      = 0    # last counter with a valid receipt
dispensed[(agent, channel)]  = 0    # last voucher sent to provider
# invariant: dispensed == acked  or  dispensed == acked + 1

execute(capability_id, channel_id, idempotency_key, request) -> result
  require capability live, channel allowlisted
  require channel.status == Open else DenyChannelClosing
  require (channel.version, channel.receipt_signer,
           channel.asset) == capability pin else DenySnapshot
  require dispensed == acked else retry outstanding request

  n = dispensed + 1
  require n ≤ max_counter and n ≤ channel.calls else DenyCap
  require window has room else DenyWindow
  require rung n in current tranche else DenyEmptyTill

  request_hash = H(canonical(request, idempotency_key))
  atomically persist { dispensed: n, idempotency_key, request_hash }
  response = provider.call(request, voucher_sig(n))

  require response contains result + receipt else IncompleteResponse
  result_hash = H(canonical(response.result))
  require receipt fields == {
    channel_id, channel.version, n,
    request_hash, result_hash
  } else DenyReceipt
  verify_receipt(channel.receipt_signer, …) else DenyReceipt

  atomically persist { acked: n, result, receipt, audit }
  return result
```

If the network fails while `dispensed == acked + 1`, retry the **same** `(version, n, idempotency_key, request_hash, voucher)`; never allocate `n+1`. Use a database transaction / row lock / compare-and-swap so concurrent agent calls cannot duplicate or skip counters.

After `fund_channel`, `calls` is higher and `version` is unchanged: load additional rungs `old_calls+1 … new_calls` and raise `max_counter`. After finalized `AdoptTerms`, pin the new version and load a new ladder from `counter+1` to the new `calls`.

### Provider (delivery)

A **billed success** is one logical record: result **and** receipt. Network delivery cannot be physically atomic, so provider and broker use idempotent retry to recover the same stored record.

```
expected[(channel_id)] = last_accepted + 1
on request (channel_id, version, n, idempotency_key, voucher_sig, body):
  load channel
  require channel.status == Open               # no new delivery while Closing
  require version == channel.version
  require n == expected else reject jump
  require n <= channel.calls
  verify_voucher(payer, channel_id, ch.version, n, voucher_sig)
    else reject                                # no receipt, not billed

  request_hash = H(canonical(body, idempotency_key))
  if stored[(channel_id, n)] exists:
    require stored.request_hash == request_hash
    return stored.result + stored.receipt       # exact re-issue

  result = serve_once(body, idempotency_key)
  if serve failed:
    return error                               # n remains retryable

  result_hash = H(canonical(result))
  receipt_sig = sign(receipt_digest(
    channel_id, ch.version, n, request_hash, result_hash
  ))

  atomically persist {
    (channel_id, n) → {
      idempotency_key, request_hash, result, result_hash, receipt_sig
    },
    last_accepted: n,
    highest_voucher: voucher_sig
  }

  return {
    result,
    receipt: {
      channel_id, version: ch.version, n,
      request_hash, result_hash, signature: receipt_sig
    }
  }
```

**Spend broker:** treat a result without a verifiable receipt as incomplete; retry the same request. Return the result to the agent only after receipt verification and durable acknowledgement.

**Re-issue:** retry/lookup returns the same result and receipt for `(channel_id, n)`. Never execute the upstream call twice or change either hash. `n` is unique for the life of the channel.

On-chain the provider still submits the **highest** voucher (`sig(5)` cashes calls 1..5). Receipts never go on-chain. Re-submitting an already-settled highest voucher is a zero-payout no-op.

---

## Rules

- Sequential **issue**: broker `n = dispensed + 1`, with at most one unacknowledged request.
- Sequential **delivery**: provider `n == last_accepted + 1`. Reject jumps.
- **Result+receipt record**: billed success includes both, bound by request/result hashes. Missing/invalid receipt ⇒ broker withholds result and retries the same `n`.
- `lookahead` is **1**. Do not expose signatures or dump `sig(1)…sig(5)` to the agent.
- Window caps count successful broker executions, not a single `sig(100)`.
- When channel status is `Closing`, stop new delivery. Old-version claims remain valid until transition finalization.
- After service update, freeze until a challenged `AdoptTerms` snapshots new terms. `counter` continues; new vouchers use the new version from `counter + 1`.
- Same-term extra quota: `fund_channel` (no freeze). Quota exhausted at new terms: `AdoptTerms` or open another channel with the next payer nonce.

---

## What receipts do not prevent

A receipt proves that the pinned provider key attested to request/result hashes. It does **not** prove semantic correctness or fair delivery. A malicious/colluding provider already holds the voucher before serving and can claim or sign a false receipt.

v0 therefore assumes an honest provider or trusted metering gateway for delivery. Stronger trust requires a provider bond/dispute process, a TEE, or service-specific verifiable computation. These are adapter/product layers, not claims made by the base channel.

---

## Provider watcher

Monitor `ChannelTransitionRequested`. Submit the highest old-version voucher before `close_after`:

```
on timer, each N vouchers, or transition event:
  submit claim_channel_funds(payer, channel_id, highest.counter, highest.signature)
```

Claims may also run periodically while Open. The challenge deadline is the final deterministic settlement window; keep a safety margin. Same-term `fund_channel` does not require a challenge claim.

---

## Payer watcher

```
if user wants more quota and ch.version == live svc.version:
  fund_channel(payer, channel_id, additional_calls)
if user wants to stop, ch.expiration reached, or ch.version != live svc.version:
  request_channel_transition(payer, channel_id, Close or AdoptTerms)
after close_after:
  finalize_channel_transition(payer, channel_id)
```

---

## Discovery

`Organization.metadata` / `Service.metadata` are discovery (endpoint, model id, schema, rate limits), never authority. The owner-controlled on-chain `Service.receipt_signer` may be a provider or gateway key. Index service/channel/transition events.
