# Service: OrderService & OrderInternalService

---

## OrderService

### `createOrder(CreateOrderRequest)`
1. Load event; guard: `status ∈ {PUBLISHED, ONGOING}`
2. For each item: guard TicketType `ACTIVE` + sufficient capacity
3. Decrement `ticketType.sold` (reserve seats)
4. Create `Order(status=PENDING)` with `eventSnapshot`
5. Create `OrderItem` records with `ticketTypeSnapshot`
6. Persist; log `ORDER_CREATED`

### `processPayment(Long orderId, PaymentProvider provider)` — checkout
1. Guard: `order.status == PENDING`
2. Generate idempotency key
3. Call `PaymentService.initiatePayment()`
4. Set `order.status = PROCESSING`
5. Return `CheckoutResponse`

### `cancelOrder(Long orderId)`
1. Guard: caller owns order or is ADMIN
2. Guard: `status ∈ {PENDING, PROCESSING}`
3. Release reserved seats (decrement `ticketType.sold`)
4. Set `status=CANCELLED`
5. Emit SSE `order:cancel` → `user:{userId}`
6. Log `ORDER_CANCELLED`

---

## OrderInternalService

### `issueTickets(Long orderId)` — called from payment callback
1. Check: tickets already issued? (idempotency guard) → return existing
2. For each `OrderItem` × quantity:
   - **Save 1:** persist ticket with `PENDING_` placeholder payload (gets DB ID)
   - **Save 2:** generate HMAC-signed QR payload using ticket ID → update and persist
3. Send confirmation email + SSE `order:confirm`
