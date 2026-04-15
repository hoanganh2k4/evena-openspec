# Service: FlexPassService

---

## Listing lifecycle

### `createListing(Long ticketId, BigDecimal submittedPrice)`
1. Guard: `ticket.user == currentUser`
2. Guard: `ticket.status == ACTIVE`
3. Guard: `ticket.transfer_count == 0`
4. Guard: `event.startAt > now`
5. Guard: `event.status == PUBLISHED`
6. Guard: `originalPrice × 0.50 ≤ submittedPrice ≤ originalPrice × 1.20`
7. `ticket.status ← TRANSFER_LOCKED`
8. Create `TicketTransfer(status=PENDING_APPROVAL, submittedPrice, expiresAt=now+14days)`
9. Log `LISTING_CREATED`

### `cancelListing(Long listingId)`
1. Guard: `transfer.seller == currentUser` or ADMIN
2. Guard: `status ∈ {PENDING_APPROVAL, APPROVED}` — **PRICE_LOCKED listings CANNOT be cancelled**
3. `ticket.status ← ACTIVE`; `transfer.status ← CANCELLED`

### `approveListing(Long listingId)` — ORGANIZER | ADMIN
`transfer.status ← APPROVED` → listing visible in marketplace (but not purchasable yet — purchase requires PRICE_LOCKED).

### `rejectListing(Long listingId)` — ORGANIZER | ADMIN
`transfer.status ← REJECTED`; `ticket.status ← ACTIVE`.

### `purchaseListing(Long listingId)`
1. Guard: `transfer.status == PRICE_LOCKED` (not APPROVED — purchase only allowed during an active sale window)
2. Guard: buyer ≠ seller
3. `transfer.buyer ← currentUser`; `transfer.status ← PAYMENT_PENDING`
4. Create escrow payment at `transfer.finalPrice` via `PaymentService`
5. Return payment URL / QR

### `completeTransfer(Long transferId, UUID buyerId)` — `@Transactional`

**All 6 steps in one transaction. Rollback on any failure.**

```
1. ticket.qrPayload    ← qrCodeService.generateQRPayload(ticketId, buyerId)
2. ticket.user_id      ← buyerId
3. ticket.transfer_count ← 1
4. ticket.status       ← ACTIVE
5. transfer.status     ← COMPLETED
6. transfer.completed_at ← now()
```

Post-commit: emit SSE `flexpass:sold` → seller + buyer; send emails; log with old/new QR.

### `failTransfer(Long transferId)`
- If sale window still `ACTIVE`: `transfer.status ← PRICE_LOCKED` (available for next buyer)
- If sale window `CLOSED` or null: `transfer.status ← EXPIRED`; `ticket.status ← ACTIVE`
- Emit SSE `flexpass:failed`; log.

### `expireListings()` — `@Scheduled`
Find `APPROVED` listings where `expires_at ≤ now`:
`transfer.status ← EXPIRED`; `ticket.status ← ACTIVE`; log.

---

## Price discovery

### `getPriceAnalysis(UUID eventId)` — ORGANIZER | ADMIN

For each ticket type that has `APPROVED` listings under this event, calculate:

| Metric | Formula |
|---|---|
| **Median** | Middle value when all `submittedPrice` values are sorted |
| **Mean** | Arithmetic average of all `submittedPrice` values |
| **Trimmed Mean** | See rules.md §9.9 trimming rules — varies by `listingCount` |

Return all 3 per ticket type plus:
- `listingCount` — number of APPROVED listings for this ticket type
- `recommended` — always `TRIMMED_MEAN`
- `recommendedPrice` — the value of the recommended metric

> See rules.md §9.9 for the full trimming table (edge cases when `listingCount < 3` or `< 10`).

### `createSaleWindow(UUID eventId, CreateSaleWindowRequest)` — ORGANIZER

1. Guard: event exists and belongs to currentUser's organization
2. Guard: no other sale window in `SCHEDULED` or `ACTIVE` state for this event
3. Guard: `saleStart > now` and `saleStart < saleEnd`
4. For each ticket type: compute `selectedPrice` using the chosen `priceMethod`
5. Create `FlexPassSaleWindow(status=SCHEDULED)` with per-ticket-type prices stored
6. Return `SaleWindowResponse` with all `selectedPrice` values so organizer can review before window opens

### `openSaleWindow(Long saleWindowId)` — `@Scheduled` (fires when `saleStart` is reached)

**All steps in one `@Transactional`:**
1. Load `FlexPassSaleWindow`; guard: `status == SCHEDULED`
2. `saleWindow.status ← ACTIVE`
3. For each `APPROVED` listing under this event:
   - Look up `selectedPrice` for listing's `ticketTypeId` from window data
   - `transfer.finalPrice ← selectedPrice`
   - `transfer.saleWindowId ← saleWindow.id`
   - `transfer.status ← PRICE_LOCKED`

Post-commit:
- Emit SSE `flexpass:window_open` → sellers (each on private channel) with their `finalPrice`
- Sellers are notified they are **forced** to sell at `finalPrice` — cancellation is no longer possible

### `closeSaleWindow(Long saleWindowId)` — `@Scheduled` (fires when `saleEnd` is reached)

**All steps in one `@Transactional`:**
1. `saleWindow.status ← CLOSED`
2. For each `PRICE_LOCKED` listing under this window:
   - `transfer.status ← EXPIRED`; `ticket.status ← ACTIVE`
3. For each `PAYMENT_PENDING` listing (buyer in progress): leave as-is — payment callback will resolve

Post-commit: emit SSE `flexpass:window_closed` to `organizer` channel; log.

### `cancelSaleWindow(Long saleWindowId)` — ORGANIZER

1. Guard: `saleWindow.status == SCHEDULED` (cannot cancel an ACTIVE window)
2. `saleWindow.status ← CANCELLED`
3. No listing state changes — listings remain APPROVED

---

## Price lock rule (CRITICAL)

When a listing is `PRICE_LOCKED`:
- `finalPrice` is immutable — cannot be changed by organizer or seller
- Seller CANNOT cancel the listing
- Buyer pays `finalPrice` (the aggregated price), not `submittedPrice`
- `submittedPrice` is retained for audit trail only
