# API: Orders & Tickets

---

## `/api/orders`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [Auth] | Create order (status=PENDING) |
| POST | `/checkout` | [Auth] | Process payment for order |
| GET | `/{orderId}` | [Auth] | Get order (owner or admin) |
| GET | `/my-orders?page=&size=&status=` | [Auth] | Caller's orders (status optional) |
| GET | `/organizer?page=&size=&status=` | [ORGANIZER\|ADMIN] | Orders for organizer's events |
| PATCH | `/{orderId}/cancel` | [Auth] | Cancel PENDING order |

### my-orders — status filter

`status` is optional. When omitted, all orders are returned.  
Valid values: `PENDING`, `PROCESSING`, `CONFIRMED`, `CANCELLED`, `EXPIRED`, `REFUNDED`.

Backend uses `(:status IS NULL OR o.status = :status)` JPQL pattern — no separate query needed.  
Frontend sends a separate `size=1` request for the "All" chip total count to avoid fetching all rows.

### CreateOrderRequest
```json
{
  "eventId": "uuid",
  "items": [
    { "ticketTypeId": 1, "quantity": 2 }
  ]
}
```

### OrderResponse
```json
{
  "id": 1, "userId": "uuid", "userEmail": "...",
  "eventId": "uuid", "eventVersion": 1,
  "eventSnapshot": { "title": "...", "startAt": "...", "venueName": "..." },
  "status": "PENDING", "totalAmount": "200000.00", "currency": "VND",
  "items": [
    {
      "id": 1, "ticketTypeId": 1, "quantity": 2,
      "unitPrice": "100000.00",
      "ticketTypeSnapshot": { "name": "...", "price": "100000.00" },
      "tickets": []
    }
  ],
  "payments": [], "createdAt": "...", "updatedAt": "..."
}
```

---

## `/api/orders/tickets`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/my-tickets` | [Auth] | All tickets for caller |
| GET | `/admin/tickets?page=&size=` | [ADMIN] | All tickets in system (paginated) |
| GET | `/{ticketId}` | [Auth] | Get ticket by ID |
| GET | `/qr/{qrPayload}` | [ORGANIZER\|ADMIN] | Get ticket by QR payload |
| POST | `/{ticketId}/check-in` | [ORGANIZER\|ADMIN] | Confirm check-in by ID |
| POST | `/validate` | [ORGANIZER\|ADMIN] | Preview scan (VALID — no state change) |
| POST | `/scan` | [ORGANIZER\|ADMIN] | Confirm scan (SUCCESS — marks USED) |

### ScanRequest
```json
{ "qrPayload": "123:uuid:nonce:hmac", "eventId": "uuid" }
```

### ScanTicketResponse
```json
{
  "success": true,
  "result": "SUCCESS",
  "ticket": { "id": 1, "status": "USED", "qrPayload": "...", "... " },
  "message": "Check-in confirmed"
}
```

> Both `/validate` and `/scan` write a `ScanLog` row for every call.
> `/validate` does **not** change ticket state.

---

## `/api/orders/events/{eventId}/scan-logs`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/` | [ORGANIZER\|ADMIN] | All scan logs for event |

### ScanLogResponse
```json
{
  "id": 1, "ticketId": 1, "eventId": "uuid",
  "scannedBy": "uuid", "scannedByName": "...",
  "scannedAt": "...", "result": "SUCCESS",
  "eventTitle": "...", "ticketTypeName": "...",
  "customerName": "...", "qrPayload": "..."
}
```
