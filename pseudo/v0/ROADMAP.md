# v0 implementation roadmap

How to turn this spec into a working port on **any** chain, in small reviewable slices.

Source of truth: the files in this directory + [research/RESEARCH.md](../../research/RESEARCH.md) + [research/AGENTIC.md](../../research/AGENTIC.md). **Translate, do not redesign.** Do not add v1 features (members after create, `fund_channel`, channel seed, `trials` billing, challenge periods).

---

## Strategy (divide and conquer)

1. **One PR ≈ one spec file or one instruction.** Reviewers can diff against a few dozen lines of pseudo, not a 2k-line program dump.
2. **Isolate the chain.** Put `now()`, token transfers, account addressing, and signature verify behind an `adapter`. The protocol crate/module stays the same on Solana, EVM, etc.
3. **Pure helpers first.** Id hashes, voucher/receipt digests, `remaining()` have no I/O. They are cheap to review and become golden tests every port must pass.
4. **Happy path before branches.** Org → service → open → claim path C, then deletes, version bump, path A/B, `update_channel`.
5. **On-chain before off-chain.** The vault and HTTP gateway are useless until claim verifies a real voucher.
6. **Off-chain in three binaries, not one:** codec lib → provider (result+receipt) → vault (`next_voucher`) → thin agent client. Each is a separate review.

Suggested layout (names are illustrative):

```
adapter/          # clock, mint, accounts, sigs  ← only folder that changes per chain
protocol/         # 01–06 verbatim
offchain/         # 02 digests + 07 consume (shared, chain-agnostic)
  provider/
  vault/
  agent/
```

A second chain is a new `adapter/` + wiring. Do not fork `protocol/` or voucher bytes.

---

## Review rules (every PR)

- Spec section cited in the PR body (`04-service.md` `update_service`, etc.).
- No extra instructions, fields, or “while we’re here” refactors.
- Tests named after the spec: `claim_path_c_uses_channel_price`, `delete_service_refuses_open_channels`.
- Events emitted as in [`01-types.md`](01-types.md).
- Adapter-only PRs must not change digest bytes or instruction semantics.

---

## Milestone A — protocol crate (on-chain)

Ship this first. After A, any chain can settle vouchers. Off-chain can be mocked.

| # | Task | Spec | Tests (minimum) | Review focus |
|---|---|---|---|---|
| A0 | Scaffold + adapter traits (`now`, `transfer`, `reserve`, `verify_sig`, account ids) | [README](README.md) Out of v0 | Adapter fakes in unit tests | Traits are thin; no protocol logic here |
| A1 | Records, errors, events, config, storage keys | [01-types.md](01-types.md) | Encode/decode roundtrip; max name/metadata | Field order matches spec (esp. Channel snapshot fields) |
| A2 | `hash_name`, `hash_channel_id`, `voucher_digest`, `verify_voucher` | [02-ids.md](02-ids.md) | Golden hashes (fixed DOMAIN, keys, counters) | **Byte layout is the ABI.** LE `u32`. DOMAIN stable |
| A3 | `create_organization` / `delete_organization` | [03-organization.md](03-organization.md) | Create, dup name, deposit, members+owner, delete with `services>0` fails | Owner rank 0; `MAX_ORG_MEMBERS`; delete requires `services==0` |
| A4 | `create_service` / `delete_service` | [04-service.md](04-service.md) | Member-only create; `version==1`; delete with `channels>0` fails | Deposit; `org.services` count |
| A5 | `update_service` | [04-service.md](04-service.md) | Every field change bumps `version`; channels unchanged | Hard cut; no rewrite of open channels |
| A6 | `remaining()`, `open_channel` | [05-channel.md](05-channel.md) | Min calls; funds=`price×N`; snapshot version/price; one channel per payer/service | Escrow in; `counter=0`; expiration=`now+threshold` |
| A7 | `update_channel` | [05-channel.md](05-channel.md) | Refund at **old** price; re-lock at live price; **persist**; **`counter=0`** | Replace not top-up; same `channel_id` |
| A8 | Claim path C (voucher settle) | [06-claim.md](06-claim.md) | `n > counter`; verify **`channel.version`**; pay `service.owner`; cap vs remaining | Not live price; not live version; anyone can submit |
| A9 | Claim path A (empty) + path B (expiry / version mismatch refund) | [06-claim.md](06-claim.md) | A: no sig, delete; B: payer only, refund remaining, delete | Payer cannot refund while live+version match |
| A10 | End-to-end on-chain | 03–06 | Org → service → open → C → bump version → B or `update_channel` | Invariants 1–9 in [README](README.md) |

**Done when:** all eight instructions work on a local validator/sim; golden voucher from a script claims successfully.

A3–A5 can overlap after A1–A2. A6 needs A4. A8–A9 need A6. A7 can follow A6 in parallel with A8 if claim tests stub channels.

---

## Milestone B — off-chain codecs (still no HTTP)

Shared library. Same bytes on every chain.

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| B0 | `voucher_digest` + `receipt_digest` (`DOMAIN` vs `DOMAIN+"/rcpt"`) | [02-ids.md](02-ids.md) | Cross-check with A2 goldens; receipt ≠ voucher | Domain split; `call_hash` included |
| B1 | `verify_receipt` (raw sig, no `<Bytes>` wrap) | [02-ids.md](02-ids.md) | Wrong signer / wrong `n` / wrong version fail | `receipt_signer` is provider, not payer |

**Done when:** a tiny CLI prints hex digests that A8’s tests already use.

---

## Milestone C — provider (billed success = `{ result, receipt }`)

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| C0 | Accept only `n == last_accepted + 1`; verify payer voucher | [07-offchain.md](07-offchain.md) | Jump rejected; bad voucher rejected; **no receipt** on error | Not billed ⇒ no receipt |
| C1 | On success, **same response**: `{ result, receipt }` | [07-offchain.md](07-offchain.md) | Missing receipt is invalid; agent fixture treats body-only as failure | Atomic both-or-neither |
| C2 | Persist `(channel, n) → call_hash`; re-issue **same** receipt | [07-offchain.md](07-offchain.md) | Re-issue matches first; never new `call_hash` for that `n` | Retry path |
| C3 | Keep highest voucher; claim watcher (timer / every N) | [07-offchain.md](07-offchain.md) + [06-claim.md](06-claim.md) | Watcher submits path C before expiry and before `update_service` | Race with path B |

**Done when:** curl a local provider with `sig(1)` gets result+receipt; `sig(3)` first is 4xx.

---

## Milestone D — vault (human till, agent dispenser)

Do not give the agent a key. Pre-signed ladder first (faster, key stays cold). Hot KMS sign is an optional later swap behind the same API.

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| D0 | Capability store: allowlist, `receipt_signer`, windows, pin `channel.version` | [AGENTIC.md](../../research/AGENTIC.md) | Unknown service / expired / revoked | Agent cannot set `receipt_signer` |
| D1 | `load_tranche` — only the next rungs, never `sig(calls)` in the hot till | AGENTIC ladder | Dump of vault ≠ full channel | Blast radius = highest stored `n` |
| D2 | `next_voucher` — vault picks `n = dispensed + 1`; first call no receipt | [07-offchain.md](07-offchain.md) | Agent cannot pass `n`; `lookahead` is 1 | Sequential fetch |
| D3 | Receipt gate: `dispensed > acked` ⇒ verify receipt for `dispensed` | [07-offchain.md](07-offchain.md) | No receipt → `DenyNeedReceipt`; bad receipt → `DenyReceipt`; idempotent retry | Unlock `n+1` only after honest serve |
| D4 | `submit_receipt` without dispense (end of job) | [07-offchain.md](07-offchain.md) | acked catches up; no extra voucher | |
| D5 | Human: `open_channel` / `update_channel` / refund via adapter; revoke | AGENTIC who-signs | Agent role cannot open/resize | Funding is human |

**Done when:** scripted agent cannot obtain `sig(2)` without a C1 receipt for `n=1`.

C and D can start after B0; they integrate with A on a devnet in E.

---

## Milestone E — agent client + thin demo

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| E0 | Client: `next_voucher` → HTTP → persist `{ result, receipt }` atomically → `submit_receipt` | [07-offchain.md](07-offchain.md) | Body without receipt = failure | No keys in the agent |
| E1 | Happy demo: human opens small channel, loads tranche, agent makes N calls, provider claims | All | Balances: locked / dispensed / acked / settled | Four numbers in AGENTIC |
| E2 | Version bump: vault freezes; human `update_channel` + new ladder | RESEARCH versioning | Old voucher still claimable at snapshot; new calls need new rungs | |

**Done when:** a stranger can follow E1 on the target chain in one README.

---

## Milestone F — “any HTTP API” (optional, after E)

| # | Task | Spec | Notes |
|---|---|---|---|
| F0 | Metering gateway as the on-chain **service**; one gateway-service per upstream | AGENTIC § gateway | Same C0–C3 interface |
| F1 | Upstream adapter (bill one upstream call per accepted `n`) | AGENTIC | Gateway is `receipt_signer` |

Skip F if the first ship is a native service only.

---

## Suggested PR order (fast path)

```
A0 → A1 → A2 → A3 → A4 → A5 → A6 → A8 → A9 → A7 → A10
                ↘ B0 → B1 → C0 → C1 → C2
                              ↘ D0 → D1 → D2 → D3 → D4 → D5
                                         ↘ E0 → E1 → E2
```

Critical path to “money moves”: **A0–A2, A3–A4, A6, A8**. Everything else can trail. That is the fastest useful ship on a new chain: open a channel and claim a voucher. Then bolt on receipts/vault so agents can use it safely.

---

## Per-chain adapter checklist (do not leak into protocol)

Pick once in A0; document in the port README:

| Concern | Spec | Typical choice |
|---|---|---|
| Time | `now()` | slot or unix |
| Token | `price`, escrow | native or one SPL/ERC-20 mint |
| Escrow | VAULT | **per-channel** PDA/account preferred |
| Ids | `hash_name` / `hash_channel_id` | same `H` and DOMAIN as 02 |
| Payer sig | `verify_voucher` | Ed25519 / secp256k1 — **raw 32-byte digest** |
| Receipt sig | `verify_receipt` | same scheme, provider key |
| Events | 01-types | program logs / ABI events |

---

## Out of scope (reject in review)

Listed in RESEARCH “Open for v1” and README. Also: putting payer keys in the agent, `lookahead > 1`, releasing `sig(100)` as a day’s batch, settling at live `service.price`, verifying vouchers against live `service.version`.
