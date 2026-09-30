# Split the bill

**Status:** Ready for handoff

**Owner:** Ngô Gia An (23120205)

**Feature of:** smart-restaurant

## Goal

Let customers at one table divide its single open order into equal shares and pay independently from their phones. The order remains `open` until every share is paid, then becomes `closed`.

## Out of scope

- Splitting by item, percentage, or customer-entered amount.
- Tips, refunds, share transfers, split cancellation, combining tables, and currencies other than VND.
- Kitchen-ticket changes. A split freezes the order's items, discounts, VAT-inclusive total, and other price inputs until payment completes.

## Flow

1. An authenticated customer currently joined to the table requests `ways` equal shares for its non-empty open order.
2. The server freezes the total, creates every share atomically, assigns rounding remainder to the lowest positions, and broadcasts the split.
3. Customers pay unpaid shares through the existing provider. The first confirmed share changes the split to `partially_paid`; each result updates and broadcasts only that share.
4. The final confirmed share changes the split to `paid`, the order to `closed`, and emits `bill.paid` exactly once.

## Contract

Amounts are integer VND after discounts and VAT. Every endpoint requires a valid JWT and current table membership.

- `POST /api/tables/:tableId/bill/split` with `{ "mode":"equal", "ways":3, "idempotencyKey":"uuid" }` returns `201 BillSplit`.
- `GET /api/tables/:tableId/bill/split` returns `200 BillSplit`, or `404 no_active_split`.
- `POST /api/bill-shares/:shareId/pay` with `{ "paymentMethodToken":"provider-token", "idempotencyKey":"uuid" }` returns `201 SharePayment` when confirmed or `202 SharePayment` when pending.

`BillSplit` example for `400000 / 3`:

```json
{"splitId":42,"orderId":91,"totalAmount":400000,"currency":"VND","status":"unpaid","shares":[{"id":1,"position":1,"amount":133334,"status":"unpaid","paymentId":null},{"id":2,"position":2,"amount":133333,"status":"unpaid","paymentId":null},{"id":3,"position":3,"amount":133333,"status":"unpaid","paymentId":null}]}
```

`SharePayment` is `{ "shareId":1, "status":"paid|pending", "paymentId":"pay_123" }`. Repeating a POST with the same actor and idempotency key replays its original status and body without another split or provider charge. A different key may retry a failed attempt; it returns `409 payment_exists` while that share has a pending or paid attempt.

Socket.IO table-room events have fixed payloads: `bill.split.created` carries `BillSplit`; `bill.share.updated` carries `{ "splitId":42,"share":{"id":1,"status":"paid","paymentId":"pay_123"} }`; `bill.paid` carries `{ "splitId":42,"orderId":91,"splitStatus":"paid","orderStatus":"closed" }`.

All errors use `ApiError`:

```json
{"error":{"code":"split_exists","message":"The order already has a split.","details":{"splitId":42}}}
```

Codes: `401 unauthenticated`; `403 not_at_table`; `404 table_not_found|share_not_found|no_open_order|no_active_split`; `409 split_exists|bill_locked|payment_exists`; `422 order_empty|invalid_ways`; `429 rate_limited` with `retryAfterSeconds`; `502 payment_failed` with `paymentId` when available.

## Data

- `bill_splits(id, order_id UNIQUE, created_by, idempotency_key, ways, total_amount, status, UNIQUE(created_by,idempotency_key))`; status: `unpaid|partially_paid|paid`.
- `bill_shares(id, split_id, position, amount, status, payment_id NULL, UNIQUE(split_id,position))`; status: `unpaid|pending|paid`.
- `payment_attempts(id, share_id, actor_id, idempotency_key, provider_payment_id NULL, status, UNIQUE(actor_id,idempotency_key), UNIQUE(provider_payment_id))`; status: `pending|paid|failed`.
- Existing `orders.status` stays `open` until completion, then becomes `closed`. An active split makes price-changing order operations return `409 bill_locked`.
- Invariants: `2 <= ways <= min(20,totalAmount)`; exactly `ways` positive shares; their sum equals the frozen total; paid shares are immutable; one active split per order.

## Errors

- **Empty state:** no open order returns `404 no_open_order`; an order with no items or total `<= 0` returns `422 order_empty` and writes nothing.
- **Partial failure:** a provider decline marks the attempt `failed`, leaves the share `unpaid`, and returns `502`; other shares remain unchanged and are never auto-refunded. An accepted but unconfirmed charge returns `202 pending`; the existing provider callback resolves it to `paid` or `unpaid` and emits one `bill.share.updated`.
- **Permissions:** non-members receive `403` with no table, split, payment, or event mutation.
- **Concurrency and duplicates:** split creation locks the order. Competing keys yield one `201` and one `409 split_exists`. Payment locks the share; competing keys yield one provider charge and one `409 payment_exists` containing its payment ID and status. Same-key requests replay.
- **Limits:** invalid `ways` returns `422`. More than 10 split/payment requests per user per table in 60 seconds returns `429`; no operation is attempted before `retryAfterSeconds` expires.

## Acceptance

- **AC1:** `405000 / 3` produces `135000,135000,135000`; `400000 / 3` produces `133334,133333,133333`; shares always sum to the frozen total.
- **AC2:** an empty order returns `422 order_empty`, creates no split/share rows, and emits no event.
- **AC3:** after two shares are paid, a failed third payment leaves two `paid`, one `unpaid`, the split `partially_paid`, and the order `open`; no refund exists.
- **AC4:** two simultaneous payments for one unpaid share produce one provider charge; the loser gets `409 payment_exists` with the winner's payment ID and status.
- **AC5:** a same-key split retry replays the original `201 BillSplit`; a different key gets `409 split_exists` with its ID.
- **AC6:** an unconfirmed accepted charge returns `202 pending`; same-key retry creates no charge; one provider callback changes it to `paid` or `unpaid` and emits one update.
- **AC7:** a non-member split or payment request returns `403`, changes no row, and emits no event.
- **AC8:** the eleventh split/payment POST by one user for one table within 60 seconds returns `429` with a positive `retryAfterSeconds` and performs no operation.
- **AC9:** the final payment changes split `paid` and order `closed` exactly once and emits one `bill.paid` payload matching the contract.
- **AC10:** after split creation, adding an item or changing discount/VAT returns `409 bill_locked`; the order total and all shares remain unchanged.

## Constraints

Use the existing React, Express, JWT, Socket.IO, and payment-provider stack; add no runtime dependency. Never store card data or retain a payment token after the provider call. Split creation and every local state transition are transactional.
