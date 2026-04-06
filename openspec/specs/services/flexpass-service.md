# Service: FlexPassService

### `createListing(Long ticketId, BigDecimal listingPrice)`
1. Guard: `ticket.user == currentUser`
2. Guard: `ticket.status == ACTIVE`
3. Guard: `ticket.transfer_count == 0`
4. Guard: `event.startAt > now`
5. Guard: `listingPrice ≤ originalPrice × 1.20`
6. `ticket.status ← TRANSFER_LOCKED`
7. Create `TicketTransfer(status=PENDING_APPROVAL, expiresAt=now+14days)`
8. Log `LISTING_CREATED`

### `cancelListing(Long listingId)`
1. Guard: `transfer.seller == currentUser` or ADMIN
2. Guard: `status ∈ {PENDING_APPROVAL, APPROVED}`
3. `ticket.status ← ACTIVE`; `transfer.status ← CANCELLED`

### `approveListing(Long listingId)` — ORGANIZER | ADMIN
`transfer.status ← APPROVED` → listing visible in marketplace.

### `rejectListing(Long listingId)` — ORGANIZER | ADMIN
`transfer.status ← REJECTED`; `ticket.status ← ACTIVE`.

### `purchaseListing(Long listingId)`
1. Guard: `transfer.status == APPROVED`; buyer ≠ seller
2. `transfer.buyer ← currentUser`; `transfer.status ← PAYMENT_PENDING`
3. Create escrow payment via `PaymentService`
4. Return payment URL / QR

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
`transfer.status ← FAILED`; `ticket.status ← ACTIVE`.
Emit SSE `flexpass:failed`; log.

### `expireListings()` — `@Scheduled`
Find `APPROVED` listings where `expires_at ≤ now`:
`transfer.status ← EXPIRED`; `ticket.status ← ACTIVE`; log.
