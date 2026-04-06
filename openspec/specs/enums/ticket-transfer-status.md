# Enum: TicketTransferStatus

Status of a FlexPass resale listing / transfer record.

| Value | Meaning |
|---|---|
| `PENDING_APPROVAL` | Listing created; awaiting organizer review |
| `APPROVED` | Approved; visible in marketplace |
| `REJECTED` | Rejected by organizer; ticket unlocked to seller |
| `PAYMENT_PENDING` | Buyer initiated payment; escrow in progress |
| `COMPLETED` | Payment confirmed; QR rotated; ownership transferred |
| `FAILED` | Payment failed; ticket unlocked to seller |
| `CANCELLED` | Seller cancelled before buyer appeared |
| `EXPIRED` | Resale window elapsed; ticket unlocked to seller |

## Transitions

```
PENDING_APPROVAL → APPROVED        (organizer approves)
PENDING_APPROVAL → REJECTED        (organizer rejects → ticket.status = ACTIVE)
APPROVED → PAYMENT_PENDING         (buyer purchases)
APPROVED → CANCELLED               (seller cancels → ticket.status = ACTIVE)
APPROVED → EXPIRED                 (window ends → ticket.status = ACTIVE)
PAYMENT_PENDING → COMPLETED        (payment SUCCESS → QR rotation, owner change)
PAYMENT_PENDING → FAILED           (payment FAILED → ticket.status = ACTIVE)
```

See [objects/flexpass/ticket-transfer.md](../objects/flexpass/ticket-transfer.md) for atomic completion steps.
