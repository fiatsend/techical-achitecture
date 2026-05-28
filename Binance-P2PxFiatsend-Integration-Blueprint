# Binance P2P x Fiatsend Integration Blueprint

## Goal

Enable Ghana P2P trades to settle faster by automating local fiat confirmation and payout actions through Fiatsend.

This blueprint maps Binance P2P trade events to Fiatsend APIs so sellers do not need to stay online waiting for buyer proof-of-payment checks.

## Positioning

Fiatsend is the Ghana fiat settlement automation layer for P2P desks:

- Buyer pays in GHS using local rails.
- Fiatsend captures and verifies payment state.
- Binance-side release/settlement actions are triggered by trusted webhook state.
- Seller and buyer both get deterministic outcomes with audit trails.

## Integration Model

Use an orchestration service between Binance and Fiatsend.

- **Do not couple directly in UI clients.** Keep API keys and decision logic server-side.
- **Treat Binance and Fiatsend as two event sources.** Resolve state in one internal trade state machine.
- **Drive all terminal actions from verified webhooks + idempotent command handlers.**

## Core Flows

### Flow A: Buyer pays seller in GHS (seller receives payment confirmation without waiting online)

1. Binance P2P order is created (`tradeId`).
2. Orchestrator creates Fiatsend payment intent:
   - `POST /v1/payment-intents`
   - Set `merchant_reference = tradeId`.
   - Include `consumer_phone` and optional `terminal_id`.
3. Buyer approves in Fiatsend consumer flow (or partner checkout flow).
4. Fiatsend emits signed webhook:
   - `payment_intent.approved` then `payment_intent.completed` (or failure states).
5. Orchestrator verifies `X-Fiatsend-Signature`, deduplicates by `event.id`, and updates internal trade status.
6. On `payment_intent.completed`, orchestrator calls Binance release endpoint for escrowed USDT.

Result: seller does not need to be online to manually confirm payment.

### Flow B: Seller sells USDT, buyer gets GHS payout

1. Binance marks trade ready for payout.
2. Orchestrator creates Fiatsend withdrawal:
   - `POST /v1/withdrawals`
   - Set `reference_id = tradeId + ":payout"`.
   - Provide `recipient_phone`, `mobile_network`, `amount`, `currency`.
3. Fiatsend emits status webhooks:
   - `withdrawal.pending`
   - `withdrawal.processing`
   - `withdrawal.completed` or `withdrawal.failed`
4. On `withdrawal.completed`, orchestrator finalizes Binance-side trade state and notifies parties.
5. On `withdrawal.failed`, orchestrator triggers compensating path (retry/manual queue/dispute state).

## Endpoint Mapping

### Fiatsend

- `POST /v1/payment-intents` - create buyer payment request (`merchant_reference` as idempotency key)
- `GET /v1/payment-intents/:id` - fetch payment intent status
- `POST /v1/withdrawals` - create seller-to-buyer GHS payout
- `GET /v1/withdrawals/:id` - fetch payout status
- `POST /v1/webhooks` - subscribe integration endpoint
- `GET /v1/webhook-deliveries` - investigate delivery failures

### Binance (or broker layer)

- Trade created / escrow opened event
- Trade release endpoint
- Trade cancel/dispute endpoint
- Balance movement endpoint

Keep Binance API calls inside dedicated command handlers:

- `releaseEscrow(tradeId)`
- `markTradePaid(tradeId)`
- `markTradeFailed(tradeId, reason)`

## Data Contract and Idempotency

Use deterministic keys across systems:

- `trade_id`: Binance trade ID (primary business key)
- `merchant_reference`: exactly `trade_id` for payment intents
- `reference_id`: `trade_id + ":withdrawal"` for withdrawals

Store cross-reference table:

- `trade_id`
- `fiatsend_payment_intent_id`
- `fiatsend_withdrawal_id`
- `latest_fiatsend_event_id`
- `binance_action_status`

Rules:

- Retry-safe create calls rely on Fiatsend idempotency (`merchant_reference`, `reference_id`).
- Webhook dedupe key is `event.id`.
- Binance command dedupe key is `trade_id + action_type`.

## Webhook Security and Reliability

Mandatory controls:

- Verify `X-Fiatsend-Signature` (HMAC-SHA256 on raw request body).
- Reject invalid signatures with HTTP 401.
- Respond `200` quickly and enqueue async processing.
- Deduplicate by `event.id`.
- Keep replay tooling via `GET /v1/webhook-deliveries`.

Operational targets:

- Webhook processing p95 < 1s (ack path only).
- End-to-end trade update p95 < 10s after event receipt.
- 0 duplicate Binance release actions.

## State Machine (Recommended)

Use explicit internal states to avoid race conditions:

- `TRADE_CREATED`
- `PAYMENT_INTENT_CREATED`
- `PAYMENT_PENDING_APPROVAL`
- `PAYMENT_APPROVED`
- `PAYMENT_COMPLETED`
- `BINANCE_RELEASE_REQUESTED`
- `BINANCE_RELEASED`
- `WITHDRAWAL_CREATED`
- `WITHDRAWAL_PROCESSING`
- `WITHDRAWAL_COMPLETED`
- `FAILED`
- `CANCELLED`
- `DISPUTE`

Only allow forward transitions unless a compensating action is explicitly defined.

## Gaps To Close Before "Easy Binance Integration" Claim

1. **Define Binance connector module**
   - Abstract all Binance API interactions behind one internal interface.
2. **Promote payment intent APIs in external docs**
   - Current public quick starts are withdrawal-heavy; add payment-intent examples in the same style.
3. **Add canonical trade orchestration examples**
   - Provide Node and Python examples that join Binance event handling with Fiatsend webhook verification.
4. **Persist event processing ledger**
   - Store processed webhook IDs and Binance command IDs in durable storage.
5. **Add runbooks**
   - Failed withdrawal, webhook retry exhaustion, Binance API downtime, and dispute handling.

## MVP Delivery Plan (2 Weeks)

### Week 1

- Implement orchestrator service with:
  - payment-intent create/get
  - withdrawal create/get
  - webhook verification + queue
  - state store + idempotency ledger
- Build Binance connector skeleton with mocked endpoints.

### Week 2

- Integrate real Binance endpoints.
- Run sandbox end-to-end tests for both flows.
- Add monitoring:
  - webhook failures
  - stuck state alerts (no transition > N minutes)
  - release/payout success ratios
- Produce go-live checklist and rollback playbook.

## Messaging For Partners

"Fiatsend helps Binance P2P merchants in Ghana automate payment confirmation and payout settlement so trades complete faster even when sellers are offline. We provide API-first payment intents, automated withdrawals, signed webhooks, and idempotent transaction handling to reduce manual intervention and failed settlements."
