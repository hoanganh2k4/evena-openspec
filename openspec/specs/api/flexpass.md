# API: FlexPass

**Base:** `/api/flexpass`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/listings` | [Auth] | Create listing — locks ticket (TRANSFER_LOCKED) |
| DELETE | `/listings/{id}` | [Auth] | Cancel listing — unlocks ticket (ACTIVE) |
| GET | `/listings?eventId=&page=&size=` | [Public] | Browse marketplace (APPROVED only) |
| GET | `/listings/my` | [Auth] | Seller's own listings |
| GET | `/listings/{id}` | [Public] | Listing detail |
| PATCH | `/listings/{id}/approve` | [ORGANIZER\|ADMIN] | Approve listing |
| PATCH | `/listings/{id}/reject` | [ORGANIZER\|ADMIN] | Reject — unlocks ticket |
| POST | `/listings/{id}/purchase` | [Auth] | Buyer purchases — creates escrow |

Payment callback is handled by `/api/payment/callback` (transfer orders identified via `order.type = FLEXPASS`).

### CreateListingRequest
```json
{ "ticketId": 1, "listingPrice": 120000.00 }
```

**Backend validations** (rules.md §11.1):
- `ticket.status == ACTIVE`
- `ticket.transfer_count == 0`
- `event.startAt > now`
- `listingPrice <= originalPrice × 1.20`

### TicketTransferResponse
```json
{
  "id": 1, "ticketId": 1,
  "eventTitle": "...", "ticketTypeName": "...",
  "seller": { "id": "uuid", "name": "..." },
  "buyer": null,
  "listingPrice": 120000.00, "originalPrice": 100000.00,
  "priceIncreasePercent": 20.0,
  "status": "APPROVED",
  "expiresAt": "...", "completedAt": null, "createdAt": "..."
}
```
