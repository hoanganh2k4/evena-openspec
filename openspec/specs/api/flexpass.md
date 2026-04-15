# API: FlexPass

**Base:** `/api/flexpass`

---

## Listing endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/listings` | [Auth] | Create listing — locks ticket (TRANSFER_LOCKED) |
| DELETE | `/listings/{id}` | [Auth] | Cancel listing — only allowed while status ∈ {PENDING_APPROVAL, APPROVED} |
| GET | `/listings?eventId=&page=&size=` | [Public] | Browse marketplace (APPROVED and PRICE_LOCKED only) |
| GET | `/listings/my` | [Auth] | Seller's own listings |
| GET | `/listings/{id}` | [Public] | Listing detail |
| PATCH | `/listings/{id}/approve` | [ORGANIZER\|ADMIN] | Approve listing |
| PATCH | `/listings/{id}/reject` | [ORGANIZER\|ADMIN] | Reject — unlocks ticket |
| POST | `/listings/{id}/purchase` | [Auth] | Buyer purchases — only allowed while status = PRICE_LOCKED |

Payment callback is handled by `/api/payment/callback` (transfer orders identified via `order.type = FLEXPASS`).

### CreateListingRequest
```json
{ "ticketId": 1, "submittedPrice": 90000.00 }
```

**Backend validations** (rules.md §9.1):
- `ticket.status == ACTIVE`
- `ticket.transfer_count == 0`
- `event.startAt > now`
- `event.status == PUBLISHED`
- `originalPrice × 0.50 ≤ submittedPrice ≤ originalPrice × 1.20`

### TicketTransferResponse
```json
{
  "id": 1, "ticketId": 1,
  "eventTitle": "...", "ticketTypeName": "...",
  "seller": { "id": "uuid", "name": "..." },
  "buyer": null,
  "submittedPrice": 90000.00,
  "finalPrice": null,
  "originalPrice": 100000.00,
  "status": "APPROVED",
  "saleWindowId": null,
  "expiresAt": "...", "completedAt": null, "createdAt": "..."
}
```

> `finalPrice` is `null` until a sale window opens and locks the price.
> Buyers always pay `finalPrice`, not `submittedPrice`.

---

## Price discovery endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/events/{eventId}/price-analysis` | [ORGANIZER\|ADMIN] | Calculate all 3 price metrics per ticket type |
| POST | `/events/{eventId}/sale-window` | [ORGANIZER] | Schedule a sale window |
| GET | `/events/{eventId}/sale-window` | [ORGANIZER\|ADMIN] | Get current sale window |
| DELETE | `/events/{eventId}/sale-window/{id}` | [ORGANIZER] | Cancel a SCHEDULED window |

### PriceAnalysisResponse
```json
{
  "eventId": "uuid",
  "ticketTypes": [
    {
      "ticketTypeId": 1,
      "ticketTypeName": "VIP",
      "listingCount": 12,
      "median": 105000.00,
      "mean": 108500.00,
      "trimmedMean": 106000.00,
      "recommended": "TRIMMED_MEAN",
      "recommendedPrice": 106000.00
    }
  ]
}
```

> `recommended` is always `TRIMMED_MEAN` (20% trim — removes top/bottom 10% of listings by price).
> Organizer may override by selecting a different `priceMethod` in CreateSaleWindowRequest.

### CreateSaleWindowRequest
```json
{
  "saleStart": "2026-05-10T10:00:00",
  "saleEnd":   "2026-05-10T22:00:00",
  "priceMethod": "TRIMMED_MEAN"
}
```

> `priceMethod` ∈ {`MEDIAN`, `MEAN`, `TRIMMED_MEAN`}. Defaults to `TRIMMED_MEAN`.

### SaleWindowResponse
```json
{
  "id": 1,
  "eventId": "uuid",
  "status": "SCHEDULED",
  "saleStart": "2026-05-10T10:00:00",
  "saleEnd": "2026-05-10T22:00:00",
  "priceMethod": "TRIMMED_MEAN",
  "ticketTypePrices": [
    {
      "ticketTypeId": 1,
      "ticketTypeName": "VIP",
      "listingCount": 12,
      "selectedPrice": 106000.00
    }
  ]
}
```
