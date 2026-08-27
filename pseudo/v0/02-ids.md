# Identity, vouchers, receipts, signatures

`H` is a 256-bit cryptographic hash. Every input uses one canonical encoding:

- fixed-width integers are unsigned little-endian;
- fixed-size values use their raw bytes;
- variable bytes are `u32_length || bytes`;
- tuple fields appear exactly in the order below.

```
DOMAIN = encode("apc", PROTOCOL_VERSION, CHAIN_ID, PROGRAM_ID)
```

Including chain and program/contract identity prevents cross-chain and cross-deployment replay. Tags (`"org"`, `"svc"`, `"ch"`, `"voucher"`, `"receipt"`) separate message types.

---

## Ids

```
hash_org_id(owner, name) -> Hash256
  H(DOMAIN || "org" || owner || len(name) || name)

hash_service_id(owner, name) -> Hash256
  H(DOMAIN || "svc" || owner || len(name) || name)

hash_channel_id(payer, org_id, service_id, nonce) -> Hash256
  H(DOMAIN || "ch" || payer || org_id || service_id || nonce)
```

| Id | Formula | Unique among |
|---|---|---|
| `org_id` | `H(DOMAIN, "org", owner, name)` | one owner |
| `service_id` | `H(DOMAIN, "svc", service_owner, name)` | one organization (storage key is `(org_id, service_id)`) |
| `channel_id` | `hash_channel_id(payer, org_id, service_id, nonce)` | global; permits isolated parallel agent channels |

Channel id does **not** include version, price, or asset. `fund_channel` and `AdoptTerms` keep the id. `counter` never resets, so old vouchers cannot replay as new spend. `nonce` is the protocol-assigned `NextChannelNonce[payer]` (`u64` little-endian), so it is never reused.

---

## Voucher digest

What the payer signs off-chain; what `claim_channel_funds` verifies on-chain.

```
voucher_digest(channel_id, version, counter) -> Hash256
  H(DOMAIN || "voucher" || channel_id || version || counter)
```

- `counter` is **cumulative** (after 17 billed calls, sign `17`, not 17 signatures of `1`). It only increases for the life of the channel.
- Require `0 < counter ≤ channel.calls`.
- `version` is the channel snapshot, not live `service.version`. Old-version vouchers remain claimable only during that version's transition challenge; finalized `AdoptTerms` snapshots the new version and rejects old-version digests. Same-term `fund_channel` does not change `version`; extra rungs are `old_calls+1 … new_calls`. After AdoptTerms, new rungs start at `counter+1`.
- `version` and `counter` are `u32` little-endian.

---

## Signature verify

Signer is always the **channel owner (payer)**.

```
verify_voucher(payer, channel_id, version, counter, signature)
  -> ok | ClaimInvalidSignature
  digest = voucher_digest(channel_id, version, counter)
  require verify_raw_digest(signature, digest, payer) else ClaimInvalidSignature
```

The deployment supports **one exact signing convention** over the raw 32-byte digest. Wallet adapters may prepare that operation, but the verifier must not accept alternate wrappers/encodings. Anyone may submit a valid voucher; payout is hardcoded to `service.owner` (see 06).

---

## Receipt digest (off-chain)

What the **provider** signs for a completed call `n`. The spend broker verifies it before acknowledging the call or returning the result. Not verified on-chain; claims still use the payer voucher only.

```
receipt_digest(channel_id, version, counter, request_hash, result_hash)
  -> Hash256
  H(DOMAIN || "receipt" || channel_id || version || counter
    || request_hash || result_hash)
```

- `version` is the same snapshot as the voucher.
- `counter` is the call just served (`n`, not `n+1`).
- `request_hash = H(canonical billed request including idempotency key)`.
- `result_hash = H(canonical complete result)`. For streaming, hash the completed stream and issue a final receipt/trailer.
- Binding both hashes lets the spend broker detect a receipt for the wrong request or result. It does **not** prove semantic correctness.

Signer is `channel.receipt_signer`, snapshotted from the owner-controlled service field at open / AdoptTerms. It may be a native provider or gateway key; metadata alone is never authority.

```
verify_receipt(receipt_signer, channel_id, version, counter,
               request_hash, result_hash, signature)
  -> ok | DenyReceipt
  digest = receipt_digest(channel_id, version, counter,
                          request_hash, result_hash)
  require verify_raw_digest(signature, digest, receipt_signer) else DenyReceipt
```
