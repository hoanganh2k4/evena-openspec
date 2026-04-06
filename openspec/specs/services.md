# Evena — Service Layer Spec

**Version:** 1.0
**Date:** 2026-04-05
**Status:** approved
**Applies to:** backend

This document describes the business logic contained in each service class.
For API signatures see `api.md`. For data model see `data-model.md`.

---

## 1. AuthService

**Responsibilities:** Registration, login, token management, email verification,
password reset.

### 1.1 `register(RegisterRequest)` / `registerOrganizer(RegisterRequest)`

1. Validate email uniqueness across `users` and `pending_registrations`
2. Hash password (bcrypt)
3. Create `PendingRegistration` with generated verification token (UUID)
4. Send verification email with token link
5. Return `UserResponse` (not yet in `users` table)

### 1.2 `verifyEmail(String token)`

1. Find `PendingRegistration` by token; throw if not found or expired
2. Create `User` in `users` table with `emailVerified=true`, `status=ACTIVE`
3. Assign role (`USER` or `ORGANIZER`)
4. Delete `PendingRegistration` row
5. Return `UserResponse`

### 1.3 `login(LoginRequest)`

1. Find user by email; throw if not found
2. Verify bcrypt hash
3. Check `emailVerified=true`; throw if false
4. Check status is `ACTIVE`
5. Generate access JWT + refresh JWT
6. Set refresh token in `HttpOnly` cookie
7. Return `LoginResponse`

### 1.4 `refreshAccessToken(String refreshToken)`

1. Validate refresh token signature and expiry
2. Load user from token subject
3. Generate new access JWT
4. Return `RefreshTokenResponse`

### 1.5 `requestPasswordReset(PasswordResetRequest)`

1. Find user by email
2. Generate reset token (UUID) and expiry (+1 hour)
3. Persist `passwordResetToken` and `passwordResetExpiresAt` on user
4. Send email with reset link

### 1.6 `resetPassword(PasswordChangeRequest)`

1. Find user by `passwordResetToken`
2. Check token not expired
3. Hash new password; update user
4. Clear `passwordResetToken` and `passwordResetExpiresAt`

---

## 2. UserService

### 2.1 `uploadAvatar(UUID userId, MultipartFile file)`

1. Upload to S3/MinIO; get URL
2. Delete previous avatar file if it exists
3. Update `user.avatarUrl`
4. Return `UploadResponse`

### 2.2 `deleteAvatar(UUID userId)`

1. Delete file from S3/MinIO
2. Set `user.avatarUrl = null`

---

## 3. OrganizationService

### 3.1 `createOrganization(CreateOrganizationRequest)`

1. Validate caller is ORGANIZER or ADMIN
2. Set `owner = currentUser`, `verified = false`
3. Persist and return `OrganizationResponse`

### 3.2 `updateOrganization(Long id, UpdateOrganizationRequest)`

1. Verify caller owns the organization or is ADMIN
2. If `verified=true` (APPROVED): block changes to identity fields
   (`name`, registration number, tax ID, bank account)
3. Apply updates to non-identity fields
4. Persist and return

### 3.3 `verifyOrganization(Long id)`

ADMIN only. Sets `verified=true`. This enables event creation.

### 3.4 `deleteOrganization(Long id)`

1. Only `PENDING` (unverified) organizations with no events may be deleted
2. `APPROVED` organizations MUST NOT be deleted — admin may archive instead
3. Soft-delete or hard-delete based on state

---

## 4. EventService

### 4.1 `createEvent(CreateEventRequest)`

1. Verify caller has access to the organization
2. Verify `organization.verified == true` — throw if PENDING/ARCHIVED
3. Set `status=DRAFT`, `eventVersion=1`
4. Persist; log `EVENT_CREATED`

### 4.2 `updateEvent(UUID eventId, UpdateEventRequest)`

1. Load event; verify ownership
2. If status ∈ {PUBLISHED, ONGOING, COMPLETED, CANCELLED}: block contractual field changes
   (title, startAt, endAt, venueId, organizationId, categoryId)
3. Increment `eventVersion` if contractual fields were changed
4. Persist; log `EVENT_UPDATED`

### 4.3 `publishEvent(UUID eventId)`

1. Verify `organization.verified == true`
2. Verify event is `DRAFT`
3. Set `status=PUBLISHED`
4. Persist; emit SSE `event:publish` to `public` + `organizer` channels
5. Log `EVENT_APPROVED` (or similar publish action)

### 4.4 `cancelEvent(UUID eventId)`

1. Verify caller has access to event
2. Transition to `CANCELLED`
3. Cascade: deactivate all TicketTypes (ACTIVE, SOLD_OUT, DRAFT → DEACTIVATED)
4. Cascade: cancel all PENDING orders → release sold counts → emit `order:cancel` SSE per user
5. Emit `event:cancel` SSE to `public` + `organizer` channels
6. Persist; log `EVENT_REJECTED` (or cancel action)

> CONFIRMED orders are NOT cancelled here — they require explicit refund flow.

### 4.5 `searchEvents(EventSearchRequest)`

Customer-visible filter: `status IN (PUBLISHED, ONGOING)`
Supports filters: keyword, categoryId, city, dateRange, explicit status (organizer/admin).

---

## 5. TicketTypeService

### 5.1 `createTicketType(UUID eventId, CreateTicketTypeRequest)`

1. Verify event exists and caller has access
2. Set `status=DRAFT`
3. Persist; log `TICKET_TYPE_ADDED`

### 5.2 `activateTicketType(UUID eventId, Long ticketTypeId)`

Validates:
- `price > 0` (or free if explicitly 0 — CurrencyMinimum applies)
- `total > 0`
- `salesStart` and `salesEnd` are set and `salesStart < salesEnd`
- Event is not CANCELLED

Transitions `DRAFT → ACTIVE`; logs `TICKET_TYPE_UPDATED`.

### 5.3 `updateTicketType(UUID eventId, Long ticketTypeId, UpdateTicketTypeRequest)`

1. If `status == ACTIVE`: reject changes to price, currency, capacity decrease,
   benefits, refundPolicy
2. Capacity increase after ACTIVE is allowed but must be logged
3. name, description, visible: always editable
4. Persist; log `TICKET_TYPE_UPDATED`

### 5.4 `deactivateTicketType(UUID eventId, Long ticketTypeId)`

1. Set `status=DEACTIVATED`
2. MUST NOT delete if it has ever been ACTIVE or has bookings
3. Log `TICKET_TYPE_DEACTIVATED`

### 5.5 `deleteTicketType(UUID eventId, Long ticketTypeId)`

Allowed only if:
- `status == DRAFT`
- No bookings exist (zero OrderItems linked)

---

## 6. OrderService

### 6.1 `createOrder(CreateOrderRequest)`

1. Load event; verify status ∈ {PUBLISHED, ONGOING} — throw if DRAFT, COMPLETED, CANCELLED
2. For each item: verify TicketType is `ACTIVE` and has sufficient capacity
3. Decrement `ticketType.sold` (reserve seats)
4. Create `Order` (status=PENDING) with `eventSnapshot`
5. Create `OrderItem` records with `ticketTypeSnapshot`
6. Persist; log `ORDER_CREATED`
7. Return `OrderResponse`

### 6.2 `processPayment(Long orderId, PaymentProvider provider)` / checkout

1. Verify order is `PENDING`
2. Generate idempotency key
3. Call `PaymentService.initiatePayment()`
4. Set order `status=PROCESSING`
5. Return `CheckoutResponse` (payment URL or QR)

### 6.3 `cancelOrder(Long orderId)`

1. Caller must own order or be ADMIN
2. Order must be `PENDING` or `PROCESSING`
3. Release reserved seats: decrement `ticketType.sold` for each item
4. Set `status=CANCELLED`
5. Emit SSE `order:cancel` to `user:{userId}` channel
6. Log `ORDER_CANCELLED`

---

## 7. TicketService

### 7.1 `scanTicket(ScanRequest)` — confirms check-in

1. Look up ticket by `qrPayload`
2. Verify `ticket.orderItem.event.id == request.eventId`
3. Verify ticket `status == ACTIVE`
4. Set `ticket.status = USED`, `ticket.usedAt = now`
5. Write `ScanLog(result=SUCCESS)`
6. Log `TICKET_USED`
7. Return `ScanTicketResponse(success=true, result=SUCCESS)`

For invalid cases: write `ScanLog` with appropriate result; return `success=false`.

### 7.1a QR verification — shared logic for scan and validate

Both `scanTicket` and `validateTicket` run this two-step lookup before any business logic:

```
1. qrCodeService.verifyAndExtractTicketId(qrPayload)
   → if empty → INVALID_QR (HMAC tampered or malformed)

2. ticketRepository.findById(ticketId)
     .filter(t -> t.getQrPayload().equals(qrPayload))
   → if empty → INVALID_QR (QR was rotated by FlexPass transfer)
```

This replaces the old `findByQrPayload(String)` text-column lookup.
The PK lookup is faster and the exact-payload filter is the FlexPass rotation guard.

### 7.2 `validateTicket(ScanRequest)` — preview only

Same lookup logic as `scanTicket` but:
- Does NOT modify ticket state
- Writes `ScanLog`:
  - Valid ticket: `result=VALID`
  - Already used: `result=ALREADY_USED`
  - Cancelled: `result=CANCELLED`
  - Expired: `result=EXPIRED`
  - QR not found: `result=INVALID_QR`
  - Wrong event: `result=WRONG_EVENT`
- Transaction: `@Transactional` (not readOnly)

### 7.3 `checkInTicket(Long ticketId)`

Alternative check-in by ticket ID (no QR scan). Same state transition as `scanTicket`.
Writes `ScanLog(result=SUCCESS)`.

### 7.4 `getScanLogs(UUID eventId)`

Returns all `ScanLog` rows for the event, ordered by `scannedAt` descending.

---

## 8. PaymentService

### 8.1 `initiatePayment(PaymentRequest)`

1. Load order; verify status ∈ {PENDING, PROCESSING}
2. Check idempotency: if `Payment` with this key already exists → return existing
3. Call payment gateway (MOMO, CARD, or CASH) via provider strategy
4. Create `Payment` row (status=PENDING)
5. Return `PaymentResponse` with paymentUrl or QR code

### 8.2 `handlePaymentCallback(PaymentCallbackDTO)`

Called by gateway webhook:
1. Validate signature
2. Find `Payment` by `transactionId`
3. If SUCCESS:
   - Set `payment.status = SUCCESS`
   - Set `order.status = CONFIRMED`
   - Issue `Ticket` for each `OrderItem` (generate QR payloads)
   - Emit SSE `order:confirm` to `user:{userId}` channel
   - Log `PAYMENT_SUCCESS`, `ORDER_SUCCESSFULL`, `TICKET_ISSUED`
4. If FAILED:
   - Set `payment.status = FAILED`
   - Release reserved seats
   - Set `order.status = CANCELLED`
   - Emit SSE `order:cancel` to `user:{userId}` channel
   - Log `PAYMENT_FAILED`, `ORDER_CANCELLED`

### 8.3 `refundPayment(RefundRequest)`

ORGANIZER or ADMIN only:
1. Verify `order.status == CONFIRMED`
2. Create refund via gateway
3. Set `payment.status = REFUNDED`, `order.status = REFUNDED`
4. Cancel associated tickets
5. Release sold counts
6. Emit SSE `order:refund` to `user:{userId}` channel
7. Log `PAYMENT_REFUNDED`

---

## 9. SSENotificationService

Manages SSE emitters by channel name.

### Channel naming convention

| Channel | Audience |
|---|---|
| `public` | All users (including unauthenticated) |
| `organizer` | All organizers |
| `admin` | Admin users |
| `user:{userId}` | Specific user (private) |

### Key events emitted

| Event name | Channel(s) | Trigger |
|---|---|---|
| `event:publish` | `public`, `organizer` | Event published |
| `event:cancel` | `public`, `organizer` | Event cancelled |
| `event:update` | `public`, `organizer` | Event metadata updated |
| `order:confirm` | `user:{userId}` | Payment confirmed |
| `order:cancel` | `user:{userId}` | Order cancelled |
| `order:refund` | `user:{userId}` | Refund processed |
| `tickettype:update` | `public` | TicketType capacity changed |
| `venue:update` | `public`, `admin` | Venue updated |

For full SSE rules see `sse-flow-spec.md`.

---

## 10. ActivityLogService

All logging is `@Async` (non-blocking).

### `log(...)` parameters

| Parameter | Type | Description |
|---|---|---|
| action | LogAction | What happened |
| entityType | EntityType | Which entity type |
| entityId | String | Entity primary key as string |
| actorId | UUID | User who triggered the action |
| actorRole | ActorRole | ADMIN / ORGANIZER / CUSTOMER |
| ownerUserId | UUID | Owner of the entity |
| eventId | UUID | Associated event (if relevant) |
| oldValue | Object | Pre-change state (serialized to JSON) |
| newValue | Object | Post-change state (serialized to JSON) |
| description | String | Human-readable description |

Callers MUST provide `actorId`, `action`, `entityType`, `entityId`.
The service persists a row in `activity_logs`.

---

## 11. EmailService

Transactional emails sent for:

| Trigger | Template |
|---|---|
| Registration | Email verification link |
| Forgot password | Password reset link |
| Order confirmed | Order receipt + ticket QR codes |
| Order cancelled | Cancellation notice |
| Event cancelled | Cancellation notice with refund info |
| Organization member invite | Invitation link |

Email is sent asynchronously. Failures are logged but do not roll back the
triggering business transaction.

---

## 12. EventScheduler

Runs as Spring `@Scheduled` tasks.

| Task | Query | Interval |
|---|---|---|
| `updateOngoingEvents()` | `PUBLISHED` events where `startAt < now AND endAt > now` → ONGOING | 5 min |
| `updateCompletedEvents()` | `ONGOING` events where `endAt <= now` → COMPLETED | 5 min |

Both tasks loop through results and call `eventRepository.save()` per event.
Errors are caught and logged but do not stop the scheduler.

> Sold-out auto-transition (`COMPLETED` when all TicketTypes are SOLD_OUT) is driven
> by `OrderService` / `TicketTypeService` at booking time, not by the scheduler.

---

## 13. FlexPassService

### 13.1 `createListing(Long ticketId, BigDecimal listingPrice)`

1. Load ticket; verify `ticket.user == currentUser`
2. Guard: `ticket.status == ACTIVE` — throw if not
3. Guard: `ticket.transfer_count == 0` — throw if already transferred
4. Guard: `event.startAt > now` — throw if event already started
5. Guard: `listingPrice <= originalPrice × 1.20` — throw if price cap exceeded
6. `ticket.status ← TRANSFER_LOCKED`; persist ticket
7. Create `TicketTransfer(status=PENDING_APPROVAL, expiresAt=now+14days)`
8. Log `LISTING_CREATED`

### 13.2 `cancelListing(Long listingId)`

1. Verify `transfer.seller == currentUser` or caller is ADMIN
2. Verify `transfer.status ∈ {PENDING_APPROVAL, APPROVED}` — throw if PAYMENT_PENDING or terminal
3. `ticket.status ← ACTIVE`; persist
4. `transfer.status ← CANCELLED`; persist
5. Log `LISTING_CANCELLED`

### 13.3 `approveListing(Long listingId)` / `rejectListing(Long listingId)`

ORGANIZER or ADMIN only.

- **approve:** `transfer.status ← APPROVED`; listing becomes public
- **reject:** `transfer.status ← REJECTED`; `ticket.status ← ACTIVE` (unlocked)
- Log accordingly

### 13.4 `purchaseListing(Long listingId)`

1. Verify `transfer.status == APPROVED`
2. Verify buyer ≠ seller
3. Set `transfer.buyer = currentUser`; `transfer.status ← PAYMENT_PENDING`
4. Create escrow payment via `PaymentService`
5. Return payment URL / QR for buyer

### 13.5 `completeTransfer(Long transferId, UUID buyerId)` — called from payment callback

**Single `@Transactional` method. All 6 steps must succeed or the entire transaction rolls back.**

```
1. ticket.qrPayload   ← UUID.randomUUID()   // QR rotation — old QR becomes INVALID_QR
2. ticket.user_id     ← buyerId             // ownership change
3. ticket.transfer_count ← 1               // block re-listing
4. ticket.status      ← ACTIVE             // unlock for buyer
5. transfer.status    ← COMPLETED
6. transfer.completed_at ← now()
```

Post-commit (non-transactional):
- Emit SSE `flexpass:sold` → `user:{sellerId}` + `user:{buyerId}`
- Send confirmation emails to both parties
- Log `TRANSFER_COMPLETED` with old_qr, new_qr, seller, buyer

### 13.6 `failTransfer(Long transferId)` — called from payment callback on failure

1. `transfer.status ← FAILED`
2. `ticket.status ← ACTIVE` (unlock back to seller)
3. Emit SSE `flexpass:failed` → `user:{sellerId}` + `user:{buyerId}`
4. Log `TRANSFER_FAILED`

### 13.7 `expireListings()` — scheduled task

Runs periodically. Finds `APPROVED` listings where `expires_at <= now`:
1. `transfer.status ← EXPIRED`
2. `ticket.status ← ACTIVE` (unlock back to seller)
3. Log `LISTING_EXPIRED`
