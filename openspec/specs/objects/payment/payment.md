# Object: Payment

**Table:** `payments` | **PK:** Long | **Module:** payment
**Indexes:** `order_id`, `idempotency_id` (unique)

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| order_id | Long | FK → orders.id, NOT NULL | |
| provider | PaymentProvider | NOT NULL | see [enums/payment.md](../../enums/payment.md) |
| status | PaymentStatus | NOT NULL | see [enums/payment.md](../../enums/payment.md) |
| idempotency_id | String | UNIQUE, NOT NULL | format: `pay_u{userId}_o{orderId}` |
| amount | BigDecimal | precision=10, scale=2, NOT NULL | |
| transaction_id | String | nullable | External gateway transaction ID |
| payload | String | TEXT, nullable | Raw gateway response (for audit) |
| error_message | String | nullable | |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Rules

- `idempotency_id` prevents duplicate payments for the same order
- `amount` is immutable once `status = SUCCESS`
- Refunds are explicit operations — not a silent field update
- `payload` stores the raw gateway response for audit and dispute resolution
