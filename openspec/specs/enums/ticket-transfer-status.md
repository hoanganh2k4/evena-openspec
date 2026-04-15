# Enum: TicketTransferStatus

Status of a FlexPass resale listing / transfer record.

| Value | Meaning |
|---|---|
| `PENDING_APPROVAL` | Listing created; awaiting organizer review |
| `APPROVED` | Approved; visible in marketplace but NOT yet purchasable |
| `REJECTED` | Rejected by organizer; ticket unlocked to seller |
| `PRICE_LOCKED` | Sale window opened; `finalPrice` set; purchasable; seller cannot cancel |
| `PAYMENT_PENDING` | Buyer initiated payment; escrow in progress |
| `COMPLETED` | Payment confirmed; QR rotated; ownership transferred |
| `CANCELLED` | Seller cancelled (only allowed before PRICE_LOCKED) |
| `EXPIRED` | Window elapsed without purchase, or 14-day TTL reached; ticket unlocked |

## Transitions

```
PENDING_APPROVAL → APPROVED        (organizer approves)
PENDING_APPROVAL → REJECTED        (organizer rejects → ticket.status = ACTIVE)
APPROVED         → PRICE_LOCKED    (sale window opens → finalPrice set by aggregation)
APPROVED         → CANCELLED       (seller cancels before window opens → ticket.status = ACTIVE)
APPROVED         → EXPIRED         (14-day TTL elapsed → ticket.status = ACTIVE)
PRICE_LOCKED     → PAYMENT_PENDING (buyer initiates purchase at finalPrice)
PRICE_LOCKED     → EXPIRED         (sale window closes without purchase → ticket.status = ACTIVE)
PAYMENT_PENDING  → COMPLETED       (payment SUCCESS → QR rotation, owner change)
PAYMENT_PENDING  → PRICE_LOCKED    (payment FAILED and sale window still ACTIVE → available for next buyer)
PAYMENT_PENDING  → EXPIRED         (payment FAILED and sale window CLOSED → ticket.status = ACTIVE)
```

## Key rules

- `PRICE_LOCKED` listings are **not cancellable** by seller
- Buyers pay `finalPrice` (aggregated), not `submittedPrice` (seller's input)
- Only `PRICE_LOCKED` listings accept purchase — `APPROVED` listings are visible but not buyable

See [objects/flexpass/ticket-transfer.md](../objects/flexpass/ticket-transfer.md) for atomic completion steps.
See [objects/flexpass/flexpass-sale-window.md](../objects/flexpass/flexpass-sale-window.md) for sale window entity.
