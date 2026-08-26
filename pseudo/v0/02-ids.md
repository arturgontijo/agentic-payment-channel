# Identity, vouchers, signatures

`H` is a 256-bit cryptographic hash. `DOMAIN` is a fixed byte string per deployment (e.g. `apc/v0`). Concatenation is domain-separated so these bytes are not valid as some other protocol’s payload.

---

## Ids

```
hash_name(owner, name) -> Hash256
  H(DOMAIN || owner || name)

hash_channel_id(payer, org_id, service_id) -> Hash256
  H(DOMAIN || payer || org_id || service_id)
```

| Id | Formula | Unique among |
|---|---|---|
| `org_id` | `hash_name(owner, name)` | one owner |
| `service_id` | `hash_name(service_owner, name)` | one organization (storage key is `(org_id, service_id)`) |
| `channel_id` | `hash_channel_id(payer, org_id, service_id)` | global; implies one channel per payer per service |

Channel id does **not** include version or price. `update_channel` keeps the same id.

---

## Voucher digest

What the payer signs off-chain; what `claim_channel_funds` verifies on-chain.

```
voucher_digest(channel_id, version, counter) -> Hash256
  H(DOMAIN || channel_id || version || counter)
```

- `counter` is **cumulative** (after 17 billed calls, sign `17`, not 17 signatures of `1`).
- `version` is **`channel.version`** (snapshot at open / last `update_channel`). After `update_service`, outstanding vouchers remain claimable at `channel.price`.
- Encode `version` and `counter` as fixed-width `u32` (little-endian) so clients and the program hash the same bytes.

---

## Signature verify

Signer is always the **channel owner (payer)**.

```
verify_voucher(payer, channel_id, version, counter, signature) -> ok | ClaimInvalidSignature
  digest = voucher_digest(channel_id, version, counter)
  if verify(signature, digest, payer):
    return ok
  // wallet wrapping (optional; drop if the target standardizes on raw 32-byte Ed25519)
  wrapped = b"<Bytes>" || digest || b"</Bytes>"
  if verify(signature, wrapped, payer):
    return ok
  fail ClaimInvalidSignature
```

Anyone may submit a valid voucher. Payout is hardcoded to `service.owner` (see 06).

---

## Receipt digest (off-chain)

What the **provider** signs after serving call `n`. The vault requires it before dispensing `sig(n+1)`. Not verified on-chain; claims still use the payer voucher only.

Use a distinct domain so a voucher cannot be replayed as a receipt (or the reverse).

```
DOMAIN_RECEIPT = DOMAIN || "/rcpt"

receipt_digest(channel_id, version, counter, call_hash) -> Hash256
  H(DOMAIN_RECEIPT || channel_id || version || counter || call_hash)
```

- `version` is `channel.version` (same snapshot as the voucher).
- `counter` is the call just served (`n`, not `n+1`).
- `call_hash` is `H` of the billed request (or an idempotency key). Binds the receipt to that call for audit and retries.
- Encode `version` / `counter` as `u32` little-endian, same as vouchers.

Signer is the **receipt key** pinned on the capability (native: `service.owner`; gateway: the gateway’s key). Not the agent, not the payer.

```
verify_receipt(receipt_signer, channel_id, version, counter, call_hash, signature)
  -> ok | DenyReceipt
  digest = receipt_digest(channel_id, version, counter, call_hash)
  require verify(signature, digest, receipt_signer) else DenyReceipt
```

No `<Bytes>` wrapping: this is a service key, not a wallet popup.
