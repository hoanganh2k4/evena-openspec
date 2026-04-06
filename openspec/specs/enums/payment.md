# Enums: Payment

---

## PaymentStatus

| Value | Meaning |
|---|---|
| `PENDING` | Payment initiated; awaiting gateway |
| `SUCCESS` | Gateway confirmed |
| `FAILED` | Gateway rejected |
| `REFUNDED` | Refund processed |

---

## PaymentProvider

| Value | Description |
|---|---|
| `CASH` | Cash — recorded manually |
| `CARD` | Card (Stripe or similar) |
| `MOMO` | MoMo e-wallet (Vietnam) |

---

## CurrencyMinimum

Minimum price for **paid** ticket types. Free tickets (`price = 0`) bypass this check.

| Currency | Minimum |
|---|---|
| VND | 1 000 |
| USD | 1 |
| EUR | 1 |
| GBP | 1 |
| SGD | 1 |
