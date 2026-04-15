# Service: PaymentService

## Overview

PaymentService orchestrates MoMo payment initiation, webhook callbacks, and refunds.
It is the single authority for payment-state transitions and SSE publication for payment/refund events.

---

## `initiatePayment(PaymentRequest)` — CUSTOMER

### Happy path
1. Resolve current user from `SecurityContext`.
2. Derive idempotency key: `pay_u{userId}_o{orderId}`.
3. Check `paymentRepository.findByIdempotencyId(key)` → if found with a non-null `payload`, return existing `PaymentResponse` (idempotent retry).
4. Fallback check: `paymentRepository.findByOrderId(orderId)` filtered to records with non-null `payload` → return if found.
5. Lock order row (`findByIdForUpdate`) — prevents concurrent duplicate initiation.
6. Guard: `order.status ∈ {PENDING, PROCESSING}` — else throw `PaymentException`.
7. Persist `Payment(status=PENDING, idempotencyId, amount, provider)`.
8. Call `PaymentGatewayService.createPayment(payment)` → returns `payUrl`.
9. Set `payment.payload = payUrl`; save; return `PaymentResponse`.

### Error: MoMo "duplicated orderId"
MoMo already registered a payment for this order but the prior DB record was lost (TX rollback).
- Detect `e.getMessage().contains("duplicated orderId")`.
- Look up any other `Payment` for the same order that has a non-null `payload` and a different ID.
- **If found: this is a recovery success path** — delete the newly-created failed record; return the recovered payment as if the original call had succeeded. Caller receives a normal `PaymentResponse`.
- **If not found: this is a fatal path** — mark payment `FAILED`, persist, rethrow `PaymentException`.

### Error: any other gateway failure
- Set `payment.status = FAILED`, `payment.errorMessage = e.getMessage()`.
- Persist.
- Throw `PaymentException`.

---

## `handlePaymentCallback(PaymentCallbackDTO)` — MoMo IPN webhook

### Routing
- MoMo `orderId` param uses prefix routing:
  - `PAY_{paymentId}` (current format) → look up by payment ID using `findByIdWithOrder`.
  - `ORDER_{orderId}` (legacy format) → look up by order ID using `findByOrderId`.

### Signature verification
- Always verify HMAC-SHA256 before processing. Reject with `PaymentException` if invalid.

### SUCCESS (`resultCode == 0`)
1. `payment.status ← SUCCESS`, `payment.transactionId ← callback.transId`.
2. `order.status ← CONFIRMED`; save order.
3. Publish `PaymentSuccessEvent(orderId)` — `OrderService.handlePaymentSuccess` picks this up via `@TransactionalEventListener(AFTER_COMMIT)` + `REQUIRES_NEW` and calls `OrderInternalService.fulfillOrder()` (issues tickets, emits SSE `order:confirm`).
4. Call `notificationService.sendPaymentConfirmation(payment)` (email — non-critical, exceptions suppressed).

### FAILED (`resultCode != 0`)
1. `payment.status ← FAILED`, `payment.errorMessage ← callback.message`.
2. If `order.status == PROCESSING` → revert to `PENDING` (allows user to retry payment).
3. Log `PAYMENT_FAILED` via `ActivityLogService` (async, fire-and-forget).
4. Save payment.

> **No sold-count release on FAILED callback**: sold counts were reserved at order creation (saga step 1). They are only released when the order is explicitly cancelled or expired.

---

## `refundPayment(RefundRequest)` — ORGANIZER | ADMIN

### Guards (evaluated in order)
1. Load payment with full details using `findByIdWithFullDetails` (JOIN FETCH order + user + items + ticketType — eliminates N+1).
2. `payment.status == REFUNDED` → throw `"Payment has already been refunded"`.
3. `payment.status != SUCCESS` → throw `"Only successful payments can be refunded"`.
4. `order.status != CONFIRMED` → throw `"Refund is only allowed for CONFIRMED orders"`.

### Happy path
1. Call `PaymentGatewayService.refundPayment(payment, reason)`.
2. `payment.status ← REFUNDED`; save.
3. `order.status ← REFUNDED`; save.
4. Cancel all tickets: `ticketRepository.findByOrderId(orderId)` → set each `ticket.status = CANCELLED` → `saveAll`.
5. Release sold counts: for each `OrderItem` in `order.getItems()` → `ticketTypeRepository.decrementSoldCount(ticketType.id, quantity)`. This atomically decrements `sold` and restores `ACTIVE` status if the type was previously `SOLD_OUT`.
6. Log `PAYMENT_REFUNDED` via `ActivityLogService` (async).
7. Call `notificationService.sendRefundConfirmation(payment)` (email).
8. Publish `RefundCompletedEvent(orderId, userId, eventId, eventTitle, amount)` — `RefundEventListener` emits SSE `order:refund` after commit (SSE-008 compliant).

### Gateway failure
- Wrap in try/catch; throw `PaymentException("Refund processing failed: ...")`.
- If `refunded == false` (gateway returned non-zero without exception): return `RefundResponse(false, "Refund failed")` without mutating DB state.

---

## `getPaymentById(Long)` — internal

Returns `Payment` by ID or throws `ResourceNotFoundException`.

## `getPaymentByOrderId(Long)` — internal

Returns `Payment` by order ID or throws `ResourceNotFoundException`.

## `getPaymentsByUserId(Long)` — internal

Returns all payments for a user using `findByUserIdWithOrder` (JOIN FETCH order — eliminates N+1 in `buildResponse`).

---

## N+1 Query Strategy

| Call site | Repository method | Fetches |
|---|---|---|
| `handlePaymentCallback` (PAY_ format) | `findByIdWithOrder` | payment + order |
| `refundPayment` | `findByIdWithFullDetails` | payment + order + user + items + ticketType |
| `getPaymentsByUserId` | `findByUserIdWithOrder` | payment + order (per user) |

---

## State machine

```
PENDING  ──(gateway call success)──► payload set, status still PENDING
         ──(IPN SUCCESS)───────────► SUCCESS  ──(refund)──► REFUNDED
         ──(IPN FAILED)────────────► FAILED
         ──(gateway call fails)────► FAILED
```

Order state driven by payment:

```
PENDING/PROCESSING ──(IPN SUCCESS)──► CONFIRMED ──(refund)──► REFUNDED
PROCESSING         ──(IPN FAILED)───► PENDING   (allows retry)
```

Ticket state driven by refund:
```
ACTIVE ──(refund)──► CANCELLED
```

---

## SSE compliance (SSE-008)

All SSE emissions from payment/refund events are deferred to `AFTER_COMMIT` via event listeners:

| Event | Listener | SSE channel |
|---|---|---|
| `PaymentSuccessEvent` | `OrderService.handlePaymentSuccess` → `OrderInternalService.fulfillOrder` | `user:{userId}` — `order:confirm` |
| `RefundCompletedEvent` | `RefundEventListener.onRefundCompleted` | `user:{userId}` — `order:refund` |

Neither `PaymentService` nor `handlePaymentCallback` emit SSE directly.

---

## MoMo redirect URL design

`returnUrl` embeds the internal order ID as `?ref={orderId}` **before** sending to MoMo.
MoMo appends its own `orderId=PAY_xx` param on redirect; using a distinct param name (`ref`) avoids collision.
The return page reads `searchParams.get('ref')` to navigate to the correct order.
