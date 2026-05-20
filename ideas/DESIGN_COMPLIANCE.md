# DESIGN.md compliance audit

Status of this implementation against each requirement in
[ideas/DESIGN.md](DESIGN.md). Last reviewed: 2026-05-20 (post-testnet end-to-end).

Legend: ✅ done · 🟡 partial · ❌ todo

## Setup (once)

| Requirement | Status | Notes |
|---|---|---|
| Agent account on Miden with hot key | ✅ | `examples/setup-testnet` deploys the agent via `MultisigGuardianBuilder` (`threshold=1`, single signer = agent hot key, `guardian_enabled=false`). The multisig auth component is what makes sign-without-prove work — it surfaces `TransactionSummary` via `Unauthorized` when no signature is staged in the advice map, whereas `BasicWallet + AuthSingleSig` actively signs during execution and fails without the secret key. (A `BasicWallet` helper still exists in `MidenIntegration::create_agent_account` but is no longer on the bench path.) |
| Agent account with cold key + Guardian co-sig for out-of-mandate ops | ❌ | Out of v1 scope. The cold-key recovery path is recovery-only; not on the payment hot path. |
| AP2 mandate stored on Guardian | 🟡 | `mandate.json` stored per agent. Signed-mandate verification not enforced; daily-totals not tracked. Per-tx amount cap + merchant allowlist + expiry are. |
| Guardian registers agent account | ✅ | `POST /agents` writes `agent.json`, `mandate.json`, `pending_state.json`, touches WAL files atomically. |

## Per-payment hot path

| Step | Status | Where |
|---|---|---|
| 0. Agent: GET /resource → 402 | ✅ | `reference-merchant` returns 402 with base64 `PAYMENT-REQUIRED` header. |
| 0. Agent: construct tx against pending state | ✅ | `AgenticClient::pay_with_metrics` reads its pending cache; real-Miden path calls `execute_for_summary`. |
| 0. Agent: sign with hot key | ✅ | Falcon over `TransactionSummary::to_commitment()`. |
| 0. Agent: send signed unproven tx + 402 context | ✅ | `POST /agents/{id}/payments` body. |
| 1. Check initial_state matches pending | ✅ | `built_on_state_commitment == pending.pending_state_commitment`. |
| 2. Verify hot-key signature | ✅ | Real Falcon verify against the inline-pubkey carried by `FalconSignature`, cross-checked with the stored commitment. |
| 3. Check tx output P2ID + amount + asset | ✅ | Five-clause check in `validate_p2id_output`: (1) exactly 1 output note, (2) `RawOutputNote::Full`, (3) script root equals `P2idNote::script_root()`, (4) recipient from `P2idNoteStorage::target()` equals `AccountId::from_hex(x402_context.merchant_account_id)`, (5) exactly 1 fungible asset with `faucet_id == x402_context.asset_faucet_id` and `amount == x402_context.amount`. Verified by `/tmp/x402-state/neg_test.py` — tampered amount and tampered merchant both rejected with `OUTPUT_CHECK_FAILED`. |
| 4. AP2 mandate: amount cap | ✅ | `amount <= mandate.per_tx_amount_cap`. |
| 4. AP2 mandate: merchant allowlist | ✅ | `mandate.merchant_allowlist.contains(merchant_id)` (empty list = wildcard for testing). |
| 4. AP2 mandate: time window | ✅ | `ctx.deadline_unix_secs <= mandate.expires_at_unix_secs`. |
| 4. AP2 mandate: daily totals | ❌ | Not tracked in v1 (explicit scope cut, 2026-05-20). |
| 5. Compute output nullifiers | ✅ | Real-Miden mode: `extract_nullifier_hexes` produces (a) input-note nullifiers via `ToInputNoteCommitments::nullifier()` and (b) output-note nullifiers via `Nullifier::new(script_root, storage_commit, asset_commit, serial_num)` for every `RawOutputNote::Full`. For a vault-spend P2ID payment this is exactly one real chain-visible nullifier — the future-spend-blocker of the merchant-bound P2ID note. Placeholder mode: trusts client-supplied. |
| 6. Reserved-nullifiers check | ✅ | WAL replayed into `HashSet<String>` at every verify. |
| 7. WAL reserve | ✅ | Append-only `nullifiers.wal` with `fsync_all` per entry. |
| 8. Advance pending state | ✅ | Atomic write of `pending_state.json` via tempfile + rename. |
| 9. Persist to queue | ✅ | One JSON file per accepted tx under `pending_queue/<seq>.json`. |
| 10. Ack (tx accepted, pending state = X) | ✅ | `AckResponse` carries `new_pending_state_commitment`, `reserved_nullifiers`, and a Falcon-signed `facilitator_ack_signature` over `(accepted_at, new_state, nullifiers)` — gives the client a non-repudiable receipt. |
| Merchant: x402 /verify | ✅ | `POST /verify` looks up the bound nullifier; returns `{valid, status}`. |
| Merchant: serve 200 OK + resource | ✅ | `reference-merchant` returns the resource with a `PAYMENT-RESPONSE` header. |

## Batch settlement (async, off the critical path)

| Step | Status | Where |
|---|---|---|
| Trigger: every N seconds **or** M pending txs | ✅ | `BatchConfig { interval_ms, max_size }` — hybrid; per-tick scan with `take(max_size)`. |
| 1. Snapshot pending across all agents | ✅ | `jobs::run_once` walks `list_agents()` × `list_queued(agent_id)`. |
| 2. Order per agent | ✅ | `list_queued` sorts by `seq`. |
| 3. Generate STARK proofs in parallel | ✅ | `submitter::Command::RebuildAndSubmit` deserializes the client's serialized `TransactionRequest`, injects the hot-key signature into the advice map at the executor-expected key, and calls `client.submit_new_transaction()` — which proves locally via `LocalTransactionProver`. Per-tx serially today (one proof at a time); parallelization is one tokio task per agent away. |
| 4. Assemble batch tx submission | 🟡 | Per-tx submission to the Miden RPC works end-to-end. "Batched-by-block" (multiple txs proven and submitted in one network round trip) is not yet — we submit each proven tx individually. |
| 5. Submit proven txs to chain | ✅ | Verified on testnet 2026-05-20. Tx ids `0xcd60…92c2`, `0xf72f…5c4c`, `0x771c…4149` (and other smoke runs) returned from `client.submit_new_transaction()` and logged by the batch worker. |
| 6. Block inclusion confirmation | ❌ | No inclusion watcher yet. `t_committed_unix_micros` stays `null` after submission; flipping `reserved → spent` waits on this. |
| 7. Move nullifiers reserved → spent | 🟡 | `mark_spent` storage method exists and is wired; not called yet because we don't yet observe block inclusion. |
| 8. Mark pending states as committed | 🟡 | `move_to_committed` exists and is wired; not called yet for the same reason. |
| 9. Notify Agentic Clients of commitment | 🟡 | Pull-side works (`GET /agents/{id}/payments/{nullifier}` returns full lifecycle timestamps). No push (SSE/websocket) yet. |

## Failure recovery

| Mode | Status | Where |
|---|---|---|
| Batch failure: identify failing tx, isolate | ✅ | On any error from `rebuild_and_submit`, `jobs::run_once` sets status = `Failed`, writes the error string into the queued record, and calls `store.move_to_failed(agent_id, seq)`. |
| Batch failure: resubmit remainder | 🟡 | The batch loop processes each queued tx independently — a failure on tx N doesn't block tx N+1. There's no explicit "retry the failed tx" path; that's a follow-up. |
| Batch failure: release nullifiers reserved by failing tx | ❌ | When a tx is marked failed, its WAL-reserved nullifier (or summary-commitment dedup key) stays in the reserved set. A `release_reserved(agent_id, &[nullifiers])` storage method is the missing piece. |
| Batch failure: notify Agentic Client | 🟡 | Reflected in `GET /payments/{nullifier}` (status = `failed`, with error string). Pull-only. |
| Guardian crash: recover reserved_nullifiers from WAL | ✅ | Verified by integration test (kill + restart + double-spend rejected). |
| Guardian crash: recover pending state from persistent store | ✅ | Same test. |
| Guardian crash: replay in-flight txs not yet committed | ✅ | `pending_queue/` survives restart; batch worker picks up where it left off. |

## Component 1: `miden-agentic-client`

| Requirement | Status |
|---|---|
| Sign-only mode (sign without prove, serialize for transmission) | ✅ |
| Track pending state per agent account | ✅ |
| Handle multiple in-flight txs | ✅ (sequential per agent is the design's stated UX; the cache + stale-base retry handle the corner case) |
| Reconcile with on-chain commitment | 🟡 (pull-side polling exists; no auto-rollback on facilitator-reported failure) |
| Light footprint (no proving keys on the agent) | ✅ |

## Component 2: `agentic-guardian` (here: `x402-facilitator-server`)

| Requirement | Status |
|---|---|
| Accept signed unproven txs | ✅ |
| Per-agent pending state tracking | ✅ |
| Per-agent nullifier reservation DB with WAL + fsync | ✅ |
| AP2 mandate enforcement | 🟡 (per-tx cap, merchant allowlist, expiry; no daily totals, no signed-mandate verify in v1) |
| Concurrent multi-agent throughput | ✅ (per-agent `DashMap<_, Mutex<()>>` so cross-agent throughput is unbounded) |
| Batch proving | ✅ (real `LocalTransactionProver` via `client.submit_new_transaction`; submitter actor synced to testnet; per-tx serial proving today) |
| Batch submission and reconciliation | 🟡 (submission to chain works; reconciliation — observing block inclusion to flip `submitted → committed` — is the inclusion-watcher follow-up) |
| Crash recovery | ✅ |

## Summary

**Hot path (per-payment) is fully real end to end on Miden testnet**, including Falcon sig verify, full P2ID output-note check (recipient/asset/amount cross-checked against `x402_context`), and real chain-visible output-note nullifier extraction.

**Batch settlement is also real**: the facilitator's submitter actor rebuilds the `TransactionRequest` from the agent's serialized bytes, injects the hot-key signature into the advice map, calls `submit_new_transaction` (which proves locally via `LocalTransactionProver` and submits to the configured Miden RPC), and writes back the resulting tx id + `t_submitted` timestamp.

**Verified on Miden testnet on 2026-05-20.** Setup deployed a faucet, a merchant, and one agent (multisig+guardian, threshold=1, guardian disabled), minted 1M XTEST to the agent, consumed the mint note. The agentic client then signed three sequential P2ID payments without proving; the facilitator picked up each one, proved it (~4 s each), and submitted to `https://rpc.testnet.miden.io`. Submitted tx ids:
- `0xcd608936e697fb7a923fd01357c04b2da10ee17f3457920a2e20e0fb78ed92c2`
- `0xf72f80c551a88c94ab309a147032ecbd8540d026aa4f6eadf5f143a033d65c4c`
- `0x771ca8c55e305b5288ffc7c785f86c2ab911130ea9da9cad939a07b39dbf4149`

What's still open after this:
- **Inclusion watcher** — `submit_new_transaction` returns once the tx is accepted by the node; we don't yet poll the chain for block inclusion to flip `reserved → spent` and set `t_committed_unix_micros`. Will use `miden-client::sync_state()` deltas or a `get_account_commitment` poll to detect commitment. Estimated effort: ~1h.
- **Daily-total AP2 enforcement** — explicit scope cut at 2026-05-20; per-tx cap, allowlist, and expiry already enforced.
- **Push notifications to clients on commitment** — pull works (`GET /agents/{id}/payments/{nullifier}` returns the full lifecycle). Push (SSE/websocket) is a polish item.
- **Batch failure handling** — if a tx fails on chain, the submitter reports `Failed`. Releasing nullifier reservations on chain-level failure is a follow-up.
