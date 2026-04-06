# Service: PaymentService

### `initiatePayment(PaymentRequest)`
1. Load order; guard: `status ∈ {PENDING, PROCESSING}`
2. Check idempotency: if `Payment` with key exists → return existing
3. Call gateway (MOMO / CARD / CASH) via provider strategy
4. Create `Payment(status=PENDING)`
5. Return `PaymentResponse` with `paymentUrl` or QR

### `handlePaymentCallback(PaymentCallbackDTO)` — gateway webhook
1. Validate signature
2. Find `Payment` by `transactionId`
3. **If SUCCESS:**
   - `payment.status ← SUCCESS`
   - `order.status ← CONFIRMED`
   - Issue tickets via `OrderInternalService.issueTickets()`
   - Emit SSE `order:confirm` → `user:{userId}`
   - Log `PAYMENT_SUCCESS`, `ORDER_SUCCESSFULL`, `TICKET_ISSUED`
4. **If FAILED:**
   - `payment.status ← FAILED`
   - Release reserved seats
   - `order.status ← CANCELLED`
   - Emit SSE `order:cancel` → `user:{userId}`
   - Log `PAYMENT_FAILED`, `ORDER_CANCELLED`

### `refundPayment(RefundRequest)` — ORGANIZER | ADMIN
1. Guard: `order.status == CONFIRMED`
2. Call gateway refund API
3. `payment.status ← REFUNDED`, `order.status ← REFUNDED`
4. Cancel associated tickets; release sold counts
5. Emit SSE `order:refund` → `user:{userId}`
6. Log `PAYMENT_REFUNDED`
