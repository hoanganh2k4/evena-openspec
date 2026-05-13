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
| `FAILED` | Payment failed (terminal) or post-payment transfer failure; ticket unlocked to seller |
| `CANCELLED` | Seller cancelled (only allowed before PRICE_LOCKED) |
| `EXPIRED` | Window closed without purchase, or 14-day TTL reached; ticket unlocked |

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
PAYMENT_PENDING  → FAILED          (payment FAILED terminal, or post-payment transfer failure → ticket.status = ACTIVE)
PAYMENT_PENDING  → PRICE_LOCKED    (payment FAILED and sale window still OPENED → available for next buyer)
```

## Key rules

- `PRICE_LOCKED` and `PAYMENT_PENDING` listings are **not cancellable** by seller
- Buyers pay `finalPrice` (aggregated), not `submittedPrice` (seller's input)
- Only `PRICE_LOCKED` listings accept purchase — `APPROVED` listings are visible but not buyable

See [objects/flexpass/ticket-transfer.md](../objects/flexpass/ticket-transfer.md) for atomic completion steps.
See [objects/flexpass/flexpass-sale-window.md](../objects/flexpass/flexpass-sale-window.md) for sale window entity.
