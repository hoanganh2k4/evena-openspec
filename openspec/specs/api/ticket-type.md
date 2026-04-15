# API: Ticket Types

**Base:** `/api/events/{eventId}/ticket-types`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create (status=DRAFT) |
| GET | `/{ticketTypeId}` | [Public] | Get by ID |
| GET | `/` | [Public] | All ticket types for event — **ACTIVE and SOLD_OUT only** for public; all states for ORGANIZER/ADMIN |
| GET | `/public` | [Public] | Visible types only (visible=true) |
| GET | `/paged?page=&size=` | [ORGANIZER\|ADMIN] | Paginated |
| GET | `/available` | [Public] | Types with stock > 0 |
| GET | `/early-bird` | [Public] | Early bird types |
| PUT | `/{ticketTypeId}` | [ORGANIZER\|ADMIN] | Update |
| PATCH | `/{ticketTypeId}/activate` | [ORGANIZER\|ADMIN] | DRAFT → ACTIVE |
| PATCH | `/{ticketTypeId}/deactivate` | [ORGANIZER\|ADMIN] | → DEACTIVATED |
| DELETE | `/{ticketTypeId}` | [ORGANIZER\|ADMIN] | Delete DRAFT with no bookings |

### CreateTicketTypeRequest
```json
{
  "name": "string (2-100)",
  "description": "string (0-500, optional)",
  "price": 100000,
  "currency": "VND",
  "total": 100,
  "perUserLimit": null,
  "salesStart": "2026-05-01T00:00:00",
  "salesEnd": "2026-06-01T00:00:00",
  "earlyBird": false,
  "earlyBirdDiscount": null,
  "visible": true
}
```

### TicketTypeResponse
```json
{
  "id": 1, "eventId": "uuid", "name": "...",
  "price": "100000.00", "currency": "VND",
  "total": 100, "sold": 0, "available": 100,
  "soldPercentage": 0.0, "status": "ACTIVE",
  "salesStart": "...", "salesEnd": "...",
  "earlyBird": false, "visible": true,
  "createdAt": "...", "updatedAt": "..."
}
```
