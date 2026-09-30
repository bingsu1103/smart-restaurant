# Split the bill

**Status:** Draft for implementation

**Owner:** Ngô Gia An (23120205)

**Feature of:** smart-restaurant

## Goal

Allow customers at one table to divide its single open order into equal shares and pay independently from their own phones, while keeping the order open until every share is paid.

## Out of scope

- Splitting by item, percentage, or a customer-entered amount.
- Tips, refunds, transfers between shares, split cancellation, combining tables, and multiple currencies.
- Changes to kitchen tickets. Creating a split freezes the order's items, discounts, and VAT-inclusive total until payment completes.

## Flow

1. Any authenticated customer currently joined to the table requests an equal split of its non-empty open order into `ways` shares.
2. The server freezes the order total, creates the split atomically, assigns any rounding remainder to the lowest-numbered shares, and broadcasts `bill.split.created` to the table room.
3. Each customer pays one unpaid share using the existing payment flow. A successful payment marks only that share paid and broadcasts `bill.share.paid`.
4. A failed or pending payment does not change other shares. When every share is paid, the server marks the split paid, closes the order, and broadcasts `bill.paid`.

## Contract

All amounts are integer VND and already include discounts and VAT. All endpoints require a valid JWT and table membership.

`POST /api/tables/:tableId/bill/split`

```json
{ "mode": "equal", "ways": 3, "idempotencyKey": "uuid" }
```

`201` returns `BillSplit`:

```json
{ "splitId": 42, "orderId": 91, "totalAmount": 400000, "currency": "VND", "status": "unpaid", "version": 1, "shares": [{ "id": 1, "position": 1, "amount": 133334, "status": "unpaid", "paymentId": null }] }
```

`POST /api/bill-shares/:shareId/pay`

```json
{ "paymentMethodToken": "provider-token", "idempotencyKey": "uuid" }
```

`201` returns `{ "shareId": 1, "status": "paid", "paymentId": "pay_123" }`; `202` returns the same shape with `status: "pending"`. Repeating either POST with the same authenticated user and idempotency key returns the original status and body without creating another split or charge.

All errors use `ApiError`:

```json
{ "error": { "code": "split_exists", "message": "The order already has a split.", "details": { "splitId": 42 } } }
```

Relevant codes: `401 unauthenticated`; `403 not_at_table`; `404 table_or_share_not_found`; `409 split_exists`, `order_changed`, or `share_already_paid` (includes the existing ID in `details`); `422 order_empty` or `invalid_ways`; `429 rate_limited` (includes `retryAfterSeconds`); `502 payment_failed` (includes `paymentId` when available).

## Data

- `bill_splits(id, order_id UNIQUE, ways, total_amount, status, idempotency_key UNIQUE)`; status is `unpaid`, `partially_paid`, or `paid`.
- `bill_shares(id, split_id, position, amount, status, payment_id NULL, UNIQUE(split_id, position))`; status is `unpaid`, `pending`, or `paid`.
- Invariants: `2 <= ways <= 20`; exactly `ways` shares exist; each amount is positive; share amounts sum exactly to the frozen order total; paid shares are immutable; at most one active split exists per order.

## Errors and thin places

- **Empty state:** an absent open order returns `404`; an order with no items or total `<= 0` returns `422 order_empty` and creates nothing.
- **Partial failure:** a declined provider charge returns `502` and leaves that share unpaid. If the provider accepted but confirmation is incomplete, return `202 pending`; retries reuse the payment ID and never charge again. Other paid shares remain paid and no automatic refund occurs.
- **Permissions:** only authenticated current table members may create a split or pay its shares; other users receive `403` with no data mutation.
- **Concurrency and duplicates:** creation runs under an order lock. Competing keys produce one split and one `409 split_exists`; the same key replays the original response. Competing payments produce one charge; the loser receives `409 share_already_paid` with the existing payment ID.
- **Limits:** `ways` outside `2..20` returns `422`. More than 10 split/payment requests per user per table per minute returns `429`; no operation is attempted until the reported retry interval expires.

## Acceptance

- **AC1 — arithmetic:** splitting `405000` three ways produces `135000, 135000, 135000`; splitting `400000` three ways produces `133334, 133333, 133333`. In every case the shares sum to the frozen total.
- **AC2 — failure isolation:** after two of three shares are paid, a failed third payment leaves the first two paid, the third unpaid, and the order open; no refund is created.
- **AC3 — concurrency:** two simultaneous payment requests for one unpaid share result in exactly one provider charge; the other response is `409` and contains the same payment ID.
- **AC4 — duplicate creation:** retrying split creation with the same key returns the original `201` body; a different key returns `409` with the existing split ID.
- **AC5 — completion:** paying the final unpaid share changes the split and order to paid/closed exactly once and emits one `bill.paid` event.
- **AC6 — frozen order:** adding an item, discount, or VAT change after split creation returns `409 order_changed`; no order or share amount changes.

## Constraints

Use the existing React, Express, JWT, Socket.IO, and payment-provider stack. Add no runtime dependency. Store no card data or payment token after the provider call. Database writes for split creation and state transitions must be transactional, and monetary arithmetic must use integer VND only.
