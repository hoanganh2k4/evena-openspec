# Enum: OrderStatus

| Value | Meaning |
|---|---|
| `PENDING` | Created; payment not yet initiated |
| `PROCESSING` | Payment in progress at gateway |
| `CONFIRMED` | Payment succeeded; tickets issued |
| `CANCELLED` | Cancelled by user or cascade |
| `EXPIRED` | Payment window elapsed; sold count released |
| `REFUNDED` | Refund completed |

## Transitions

```
PENDING    → PROCESSING   (payment initiated)
PROCESSING → CONFIRMED    (payment callback: SUCCESS)
PROCESSING → PENDING      (payment callback: FAILED — reverted to allow retry)
PENDING    → CANCELLED    (user cancel OR event cancel cascade)
PENDING    → EXPIRED      (TTL elapsed)
CONFIRMED  → REFUNDED     (admin/organizer explicit refund)
```

## Rules

- Once `CONFIRMED`: `total_amount` is immutable (rules.md §6)
- Event cancel cascade affects only `PENDING` orders
- `CONFIRMED` orders require explicit refund — not auto-cancelled
