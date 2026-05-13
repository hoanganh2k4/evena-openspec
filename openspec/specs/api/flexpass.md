# API: FlexPass

**Base:** `/api/flexpass`

---

## Seller / buyer endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/listings` | [Auth] | Create listing — locks ticket (TRANSFER_LOCKED) |
| PATCH | `/listings/{listingId}/cancel` | [Auth] | Cancel listing — only allowed while status ∈ {PENDING_APPROVAL, APPROVED} |
| GET | `/listings/my` | [Auth] | Seller's own listings |
| GET | `/listings/{listingId}` | [Auth] | Listing detail |

## Marketplace endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/marketplace/listings?eventId=` | [Public] | Browse marketplace (APPROVED and PRICE_LOCKED only) |
| POST | `/listings/{listingId}/purchase` | [Auth] | Buyer purchases — only allowed while status = PRICE_LOCKED (handled by FlexPassCheckoutService) |

Payment callback is handled by `/api/payment/callback` (transfer orders identified via `order.type = FLEXPASS`).

## Organizer endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/organizer/listings` | [ORGANIZER\|ADMIN] | All listings for organizer's events (ADMIN gets all system listings) |
| PATCH | `/organizer/listings/{listingId}/approve` | [ORGANIZER\|ADMIN] | Approve listing |
| PATCH | `/organizer/listings/{listingId}/reject` | [ORGANIZER\|ADMIN] | Reject — unlocks ticket |
| GET | `/organizer/events/{eventId}/sale-window` | [ORGANIZER\|ADMIN] | All non-cancelled sale windows for event |
| GET | `/organizer/events/{eventId}/price-analysis` | [ORGANIZER\|ADMIN] | Calculate all 3 price metrics per ticket type |
| POST | `/organizer/sale-windows` | [ORGANIZER\|ADMIN] | Schedule a sale window |
| PATCH | `/organizer/sale-windows/{saleWindowId}/cancel` | [ORGANIZER\|ADMIN] | Cancel a SCHEDULED window |

## Admin endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/admin/sale-windows` | [ADMIN] | All sale windows across all events |

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

## Price discovery (request/response schemas)

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
  "eventId": "uuid",
  "startAt": "2026-05-10T10:00:00",
  "endAt":   "2026-05-10T22:00:00",
  "pricingMethodByTicketTypeId": {
    "1": "TRIMMED_MEAN",
    "2": "MEDIAN"
  }
}
```

> `pricingMethodByTicketTypeId` is optional per ticket type key. Any ticket type not present defaults to `TRIMMED_MEAN`.
> Values ∈ {`MEDIAN`, `MEAN`, `TRIMMED_MEAN`}.

### SaleWindowResponse
```json
{
  "id": 1,
  "eventId": "uuid",
  "eventTitle": "...",
  "status": "SCHEDULED",
  "pricingMethod": "TRIMMED_MEAN",
  "startAt": "2026-05-10T10:00:00",
  "endAt": "2026-05-10T22:00:00",
  "openedAt": null,
  "closedAt": null,
  "cancelledAt": null,
  "createdAt": "...",
  "updatedAt": "...",
  "prices": [
    {
      "ticketTypeId": 1,
      "ticketTypeName": "VIP",
      "listingCount": 12,
      "medianPrice": 105000.00,
      "meanPrice": 108500.00,
      "trimmedMeanPrice": 106000.00,
      "selectedPrice": 106000.00,
      "pricingMethod": "TRIMMED_MEAN"
    }
  ]
}
```
