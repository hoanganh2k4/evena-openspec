# Enum: TicketStatus

| Value | Meaning |
|---|---|
| `ACTIVE` | Valid; not yet used |
| `USED` | Checked in (confirmed) |
| `CANCELLED` | Cancelled via order cancellation |
| `EXPIRED` | Event ended; no longer valid |
| `TRANSFER_LOCKED` | Listed for FlexPass transfer; **cannot be scanned** |

## Transitions

```
ACTIVE → USED             (check-in: /tickets/scan or /tickets/{id}/check-in)
ACTIVE → CANCELLED        (order cancelled)
ACTIVE → EXPIRED          (event completed/cancelled)
ACTIVE → TRANSFER_LOCKED  (FlexPass: seller creates listing)
TRANSFER_LOCKED → ACTIVE  (FlexPass: listing cancelled / failed / expired → back to seller)
TRANSFER_LOCKED → ACTIVE  (FlexPass: transfer COMPLETED → new owner, new QR)
```

## Rules

- `TRANSFER_LOCKED` tickets MUST NOT be checked in → `ScanLogResult.TRANSFER_LOCKED`
- After transfer: `user_id` changes to buyer, `qr_payload` rotated to new HMAC payload
