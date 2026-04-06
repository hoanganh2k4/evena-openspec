# API: Payment

**Base:** `/api/payment`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/initiate` | [Auth] | Initiate payment for an order |
| POST | `/callback` | [Public] | Gateway webhook (signature validated) |
| POST | `/refund` | [ORGANIZER\|ADMIN] | Initiate refund |
| GET | `/order/{orderId}` | [Auth] | Payment for an order |
| GET | `/{paymentId}` | [Auth] | Payment by ID |
| GET | `/user/{userId}` | [ORGANIZER\|ADMIN or own] | Payments by user |

### PaymentRequest
```json
{
  "orderId": 1,
  "provider": "MOMO",
  "amount": "200000.00",
  "returnUrl": "https://...",
  "cancelUrl": "https://..."
}
```

### PaymentResponse
```json
{
  "paymentId": 1, "orderId": 1,
  "amount": "200000.00", "status": "PENDING",
  "provider": "MOMO", "transactionId": null,
  "paymentUrl": "https://payment-gateway/...",
  "createdAt": "...", "updatedAt": "..."
}
```

## Payment flow

```
1. POST /api/orders/checkout  →  order.status = PROCESSING
2. POST /api/payment/initiate →  payment row (PENDING), gateway contacted
3. Gateway callback            →  order = CONFIRMED, tickets issued
                               OR order = CANCELLED, seats released
```

**Idempotency key:** `pay_u{userId}_o{orderId}` — prevents duplicate payment records.
