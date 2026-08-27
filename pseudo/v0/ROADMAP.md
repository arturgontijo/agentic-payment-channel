# v0 implementation roadmap

How to turn this spec into a working port on **any** chain, in small reviewable slices.

Source of truth: the files in this directory + [research/RESEARCH.md](../../research/RESEARCH.md) + [research/AGENTIC.md](../../research/AGENTIC.md). **Translate, do not redesign.** Nonce-isolated channels, version-bound vouchers, monotonic counters, `fund_channel`, challenged close/AdoptTerms, explicit assets, spend broker, and result-bound receipts are v0.

---

## Strategy (divide and conquer)

1. **One PR ≈ one spec file or one instruction.** Reviewers can diff against a few dozen lines of pseudo, not a 2k-line program dump.
2. **Isolate the chain.** Put `now()`, token transfers, account addressing, and signature verify behind an `adapter`. The protocol crate/module stays the same on Solana, EVM, etc.
3. **Pure helpers first.** Id hashes, voucher/receipt digests, `remaining()` have no I/O. They are cheap to review and become golden tests every port must pass.
4. **Happy path before branches.** Org → service → open → voucher claim, then fund, challenged close/AdoptTerms, and cleanup.
5. **On-chain before off-chain.** The spend broker and HTTP gateway are useless until claim verifies a real voucher.
6. **Off-chain in small boundaries:** codec lib → provider (idempotent result+receipt) → spend broker → thin agent client. No raw signature API.

Suggested layout (names are illustrative):

```
adapter/          # clock, asset, escrow/accounts, sigs ← only folder that changes per chain
protocol/         # 01–06 verbatim
offchain/         # 02 digests + 07 consume (shared, chain-agnostic)
  provider/
  broker/
  agent/
```

A second chain is a new `adapter/` + wiring. Do not fork `protocol/` or voucher bytes.

---

## Review rules (every PR)

- Spec section cited in the PR body (`04-service.md` `update_service`, etc.).
- No extra instructions, fields, or “while we’re here” refactors.
- Tests named after the spec: `claim_uses_snapshot_price`, `fund_keeps_counter`, `finalize_waits_for_challenge`, `delete_service_refuses_open_channels`.
- Events emitted as in [`01-types.md`](01-types.md).
- Adapter-only PRs must not change digest bytes or instruction semantics.

---

## Milestone A — protocol crate (on-chain)

Ship this first. After A, any chain can settle vouchers. Off-chain can be mocked.

| # | Task | Spec | Tests (minimum) | Review focus |
|---|---|---|---|---|
| A0 | Scaffold adapter (`now`, asset transfer, per-channel escrow, reserve, raw-digest signature, atomic rollback) | [README](README.md) | Adapter fakes; transfer failure rolls back | No protocol logic; asset explicit |
| A1 | Records/status/errors/events/config/storage/next-nonce | [01-types.md](01-types.md) | Encode/decode; status transitions; max fields | Monotonic nonce/counter, immutable asset, receipt signer |
| A2 | Deployment domain, tagged ids, nonce channel id, version voucher | [02-ids.md](02-ids.md) | Cross-language golden hashes | Canonical bytes are ABI; chain+program in domain |
| A3 | `create_organization` / `set_organization_paused` / `delete_organization` | [03-organization.md](03-organization.md) | Create, dup name, deposit, members+owner, pause blocks create_service, delete with `services>0` fails | Owner rank 0; `MAX_ORG_MEMBERS`; delete requires `services==0` |
| A4 | `create_service` / `set_service_paused` / `delete_service` | [04-service.md](04-service.md) | Member-only; immutable asset; receipt signer; pause blocks open/AdoptTerms; delete with channels fails; create fails if org paused | Checked counts; pause does not bump version |
| A5 | `update_service` | [04-service.md](04-service.md) | Every field change bumps `version`; channels unchanged | Hard cut; no rewrite of open channels |
| A6 | Checked `remaining()`, `open_channel(nonce)` | [05-channel.md](05-channel.md) | Per-channel asset escrow; snapshots; wrong/reused nonce rejects; paused org/service rejects | `Open`, increment next nonce, `counter=0` |
| A6b | `fund_channel` | [05-channel.md](05-channel.md) | Raises `calls`; keeps `counter`/`version`; stale version rejects | No challenge; escrow += price × added |
| A7 | Request/cancel transition | [05-channel.md](05-channel.md) | Closing freezes delivery; AdoptTerms pre-funds pending escrow; cancel refunds it; same-version AdoptTerms rejects; paused AdoptTerms rejects, Close still works | Owner authorizes funding at request |
| A8 | Voucher settle | [06-claim.md](06-claim.md) | `n>counter` pays delta; `n==counter` zero claim; verify version; pay owner; overflow rejects | Open or Closing; no cap-and-continue |
| A9 | Finalize Close/AdoptTerms + exhausted cleanup | [05-channel.md](05-channel.md), [06-claim.md](06-claim.md) | Before deadline fails; claims reduce refund; pending escrow promoted; `calls=counter+new_quota`; counter unchanged | Anyone can finalize without payer debit |
| A10 | End-to-end on-chain | 03–06 | Open two nonces; claim; fund; request AdoptTerms; claim during challenge; finalize | All README invariants |

**Done when:** every listed instruction works on a local validator/sim; golden voucher claims; no immediate-refund race remains.

A3–A5 can overlap after A1–A2. A6 needs A4. A6b/A7/A8 follow A6 in parallel; A9 needs A7 and A8.

---

## Milestone B — off-chain codecs (still no HTTP)

Shared library. Same bytes on every chain.

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| B0 | Canonical deployment domain + voucher/receipt digests | [02-ids.md](02-ids.md) | Cross-check A2; every tag differs | Version; request+result hashes |
| B1 | `verify_receipt` raw digest | [02-ids.md](02-ids.md) | Wrong signer/version/n/request/result fail | Signer is channel snapshot |

**Done when:** a tiny CLI prints hex digests that A8’s tests already use.

---

## Milestone C — provider (billed success = `{ result, receipt }`)

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| C0 | Accept only Open/current version and `n == last_accepted + 1`; verify voucher | [07-offchain.md](07-offchain.md) | Closing/jump/bad voucher reject | Strict `n≤calls` |
| C1 | Persist `{ result, receipt(request_hash,result_hash) }` | [07-offchain.md](07-offchain.md) | Missing/wrong hash invalid | Durable record before response |
| C2 | Idempotent retry by `(channel,n,idempotency_key)` | [07-offchain.md](07-offchain.md) | Same result/receipt; upstream runs once | Exactly-once effect |
| C3 | Highest voucher + transition-aware claim watcher | 07 + 06 | Claims before `close_after` with margin; already-settled n is zero claim | Challenge event handling |

**Done when:** curl a local provider with `sig(1)` gets result+receipt; `sig(3)` first is 4xx.

---

## Milestone D — spend broker (human till, no agent vouchers)

Do not give the agent a key. Pre-signed ladder first (faster, key stays cold). Hot KMS sign is an optional later swap behind the same API.

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| D0 | Capability pins channel id/nonce/version/asset/receipt signer + windows | AGENTIC | Unknown/stale/revoked/Closing deny | Agent controls none of snapshots |
| D1 | Encrypted cold ladder + small hot tranche | AGENTIC ladder | Hot dump bounded; ciphertext useless without wrapping key; fund appends rungs | Never load top rung |
| D2 | Internal voucher issue under durable `execute` (no signature API) | 07 | Agent cannot obtain/pass `n`; concurrent calls serialize | `lookahead=1` |
| D3 | Provider call + receipt verification + atomic ack/result | 07 | Missing/wrong receipt withheld; same outstanding retry | Request/result hashes |
| D4 | Crash/retry/CAS state machine | 07 | Fail at every persistence/network boundary | No duplicate/skip |
| D5 | Human open/fund/transition/finalize/revoke | AGENTIC | Agent role cannot fund/transition | Funding is human |

**Done when:** scripted agent never receives a signature; broker cannot issue `n+1` before durable receipt for `n`.

C and D can start after B0; they integrate with A on a devnet in E.

---

## Milestone E — agent client + thin demo

| # | Task | Spec | Tests | Review focus |
|---|---|---|---|---|
| E0 | Thin client: agent calls broker `execute`; broker returns verified result | [07-offchain.md](07-offchain.md) | No voucher/signature in agent process/logs | No keys/bearer IOUs |
| E1 | Happy demo: human opens small channel, loads tranche, agent makes N calls, provider claims | All | Balances: locked / dispensed / acked / settled | Four numbers in AGENTIC |
| E2 | Version bump: broker freezes; challenged AdoptTerms; new ladder from `counter+1` | RESEARCH versioning | Old claim lands before deadline; new version rejects old voucher; counter continues | |
| E3 | Same-term `fund_channel`: extra rungs, no freeze | 05 + AGENTIC | `counter` unchanged; next `n` continues | |

**Done when:** a stranger can follow E1 on the target chain in one README.

---

## Milestone F — “any HTTP API” (optional, after E)

| # | Task | Spec | Notes |
|---|---|---|---|
| F0 | Metering gateway as the on-chain **service**; one gateway-service per upstream | AGENTIC § gateway | Same C0–C3 interface |
| F1 | Upstream adapter (bill one upstream call per accepted `n`) | AGENTIC | Gateway is `receipt_signer` |

Skip F if the first ship is a native service only.

---

## Milestone G — production gate

Prototype completion is not production readiness.

| # | Task | Exit condition |
|---|---|---|
| G0 | Model/property tests for all README invariants | Random transition sequences conserve every channel/asset escrow |
| G1 | Differential golden vectors | On-chain, broker, provider, and CLI emit identical ids/vouchers/receipts |
| G2 | Fuzz malformed encodings/signatures/counters/arithmetic | No panic, saturation, partial state, or alternate accepted encoding |
| G3 | Concurrency/chaos tests | Crash/network loss at every broker/provider persistence boundary causes no duplicate upstream effect or skipped counter |
| G4 | Testnet soak with real agents/watchers | Transition deadlines, claims, retries, revocation, and reconciliation run unattended |
| G5 | Independent security review | Claim math, challenge state machine, escrow adapter, digest ABI, KMS/tranche controls reviewed |
| G6 | Mainnet canary | One asset/service, tiny per-channel/tranche limits, monitored rollback plan |

The deployment README must state its provider trust mode: honest native provider, trusted gateway, bond/dispute, TEE, or verifiable service.

---

## Suggested PR order (fast path)

```
A0 → A1 → A2 → A3 → A4 → A5 → A6 → A8 → A6b → A7 → A9 → A10
                ↘ B0 → B1 → C0 → C1 → C2
                              ↘ D0 → D1 → D2 → D3 → D4 → D5
                                         ↘ E0 → E1 → E2 → E3 → G0…G6
```

Critical path to “money moves”: **A0–A2, A3–A4, A6, A8**. Production safety also requires A6b (top-up), A7/A9 (no refund race) and D (agent never handles vouchers).

---

## Per-chain adapter checklist (do not leak into protocol)

Pick once in A0; document in the port README:

| Concern | Spec | Typical choice |
|---|---|---|
| Time | `now()` | slot or unix |
| Asset | `service.asset`, channel escrow | explicit native id or token mint |
| Escrow | `ESCROW(channel,asset)` | per-channel PDA/account; pooled only with equivalent liabilities |
| Ids | tagged org/service/channel hashes | DOMAIN includes protocol+chain+program; nonce |
| Payer sig | `verify_voucher` | Ed25519 / secp256k1 — **raw 32-byte digest** |
| Receipt sig | `verify_receipt` | same scheme, provider key |
| Events | 01-types | program logs / ABI events |

---

## Out of scope (reject in review)

Listed in RESEARCH “Open for v1” and README. Also reject: agent key/voucher access, raw-signature APIs, lookahead >1, plaintext full ladders online, immediate refunds, nonce reuse, counter reset, saturating arithmetic, live-price settlement, alternate signature encodings.
