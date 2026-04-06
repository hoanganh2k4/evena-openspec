# Enum: TicketTypeStatus

| Value | Meaning |
|---|---|
| `DRAFT` | Not visible to customers; fully editable |
| `INACTIVE` | Reserved (not currently used in business logic) |
| `ACTIVE` | Public; purchasable; contractual fields locked |
| `SOLD_OUT` | Auto-set when `sold >= total`; auto-reverts when freed |
| `DEACTIVATED` | Permanently removed from sale; historical data retained |

## Transitions

```
DRAFT    → ACTIVE       (organizer/admin explicit activation)
ACTIVE   → SOLD_OUT     (auto: when sold >= total)
SOLD_OUT → ACTIVE       (auto: when booking cancelled, sold < total)
ACTIVE   → DEACTIVATED  (organizer/admin)
DRAFT    → DEACTIVATED  (cascade on event cancel)
ACTIVE   → DEACTIVATED  (cascade on event cancel)
SOLD_OUT → DEACTIVATED  (cascade on event cancel)
```

## Activation requirements (DRAFT → ACTIVE)

- `price ≥ 0` and meets CurrencyMinimum if paid
- `total > 0`
- `salesStart` and `salesEnd` are set, `salesStart < salesEnd`
- Parent event not CANCELLED

## Contractual fields locked once ACTIVE

`price`, `currency`, `total` (decrease forbidden), benefits, refundPolicy
