# Evena — REST API Spec

**Version:** 1.0
**Date:** 2026-04-05
**Status:** approved
**Base URL:** `http://localhost:8080`
**Auth scheme:** Bearer JWT in `Authorization` header; refresh token in `refreshToken` httpOnly cookie

---

## Conventions

- All responses are wrapped: `{ success: boolean, message: string, data: T }`
- Paginated responses: `{ content: T[], totalElements, totalPages, number }`
- UUIDs are returned as lowercase hyphenated strings
- Timestamps are ISO-8601 (UTC)
- Currency amounts are strings with 2 decimal places
- **Role annotations:** `[Public]` `[Auth]` `[ORGANIZER]` `[ADMIN]` `[ORGANIZER|ADMIN]`

---

## 1. Auth — `/api/auth`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/login` | [Public] | Email/password login |
| POST | `/register` | [Public] | Register as customer (USER role) |
| POST | `/registerOrganizer` | [Public] | Register as organizer (ORGANIZER role) |
| POST | `/forgot-password` | [Public] | Send password reset email |
| POST | `/reset-password` | [Public] | Consume reset token, set new password |
| GET | `/verify-email?token=` | [Public] | Verify email address |
| GET | `/me` | [Auth] | Get current user profile |
| POST | `/refresh` | [Public] | Refresh access token via cookie |
| POST | `/logout` | [Auth] | Invalidate refresh token cookie |

### POST `/api/auth/login`

**Request:**
```json
{
  "email": "user@example.com",
  "password": "••••••••"
}
```

**Response:**
```json
{
  "accessToken": "eyJ...",
  "refreshToken": "eyJ...",
  "tokenType": "Bearer",
  "expiresIn": 3600,
  "user": {
    "id": "uuid",
    "name": "Jane",
    "email": "user@example.com",
    "phone": null,
    "status": "ACTIVE",
    "emailVerified": true,
    "roles": ["USER"],
    "createdAt": "2026-01-01T00:00:00",
    "updatedAt": "2026-01-01T00:00:00"
  }
}
```

---

## 2. User — `/api/users`

| Method | Path | Auth | Description |
|---|---|---|---|
| PUT | `/avatar` (multipart) | [Auth] | Upload/replace avatar image |
| DELETE | `/avatar` | [Auth] | Delete avatar |

---

## 3. Organizations — `/api/organizations`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create organization |
| PUT | `/{organizationId}` | [ORGANIZER\|ADMIN] | Update organization |
| GET | `/{organizationId}` | [Public] | Get organization by ID |
| GET | `/my-organizations` | [Auth] | Get caller's organizations |
| GET | `/` | [Public] | List all (paginated) |
| GET | `/search?keyword=&page=&size=` | [Public] | Search organizations |
| PATCH | `/{organizationId}/verify` | [ADMIN] | Approve organization |
| DELETE | `/{organizationId}` | [ORGANIZER\|ADMIN] | Delete (PENDING only) |
| GET | `/{organizerId}/details` | [Auth] | Organization detail with members |
| GET | `/user/{userId}/organizations` | [Auth] | All organizations for a user |

### CreateOrganizationRequest
```json
{
  "name": "string (2-200)",
  "description": "string (10+)",
  "logoUrl": "string (optional)",
  "website": "https://... (optional)",
  "email": "org@example.com (optional)",
  "phone": "string (optional)"
}
```

---

## 4. Organization Members — `/api/organizations/{organizationId}/members`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/invite` | [ORGANIZER\|ADMIN] | Invite a member |
| POST | `/{memberId}/accept` | [Auth] | Accept membership |
| POST | `/{memberId}/reject` | [Auth] | Reject invitation |
| PATCH | `/{memberId}/role` | [ORGANIZER\|ADMIN] | Change member role |
| DELETE | `/{memberId}` | [ORGANIZER\|ADMIN] | Remove member |
| GET | `/` | [Auth] | List members |
| GET | `/paged?page=&size=` | [Auth] | List members (paginated) |

### Invitations — `/api/invitations`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/pending` | [Auth] | Get pending invitations for current user |
| POST | `/{invitationId}/accept` | [Auth] | Accept invitation |
| POST | `/{invitationId}/reject` | [Auth] | Reject invitation |

---

## 5. Events — `/api/events`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create event (status=DRAFT) |
| PUT | `/{eventId}` | [ORGANIZER\|ADMIN] | Update event |
| GET | `/{eventId}` | [Public] | Get event by ID |
| GET | `/` | [Public] | All upcoming events (PUBLISHED + ONGOING) |
| GET | `/search` | [Public] | Search with filters |
| GET | `/organizer/{organizerId}` | [Public] | Events by organizer |
| GET | `/my-events` | [ORGANIZER\|ADMIN] | Events the caller manages |
| PATCH | `/{eventId}/publish` | [ORGANIZER\|ADMIN] | Publish event |
| PATCH | `/{eventId}/cancel` | [ORGANIZER\|ADMIN] | Cancel event |
| DELETE | `/{eventId}` | [ORGANIZER\|ADMIN] | Soft-delete DRAFT event |

### EventSearchRequest (query params)
```
keyword    — full-text search on title + description
categoryId — filter by category
city       — filter by venue city
startDate  — range filter (ISO date)
endDate    — range filter (ISO date)
status     — explicit status (admin/organizer only; customer sees PUBLISHED+ONGOING)
page       — default 0
size       — default 10
```

### CreateEventRequest
```json
{
  "title": "string (3-200)",
  "description": "string (10+)",
  "startAt": "2026-06-01T09:00:00",
  "endAt": "2026-06-01T18:00:00",
  "organizationId": 1,
  "categoryId": 1,
  "venueId": 1,
  "coverUrl": "https://... (optional)",
  "imageUrls": ["https://..."]
}
```

### EventResponse
```json
{
  "id": "uuid",
  "title": "...",
  "description": "...",
  "startAt": "...",
  "endAt": "...",
  "status": "PUBLISHED",
  "coverUrl": "...",
  "eventVersion": 1,
  "organization": { "id": 1, "name": "..." },
  "category": { "id": 1, "name": "..." },
  "venue": { "id": 1, "name": "...", "city": "..." },
  "images": [],
  "ticketTypes": [],
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

## 6. Categories — `/api/categories`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ADMIN] | Create category |
| PUT | `/{categoryId}` | [ADMIN] | Update category |
| GET | `/{categoryId}` | [Public] | Get category |
| GET | `/` | [Public] | List all categories |
| DELETE | `/{categoryId}` | [ADMIN] | Delete category |

---

## 7. Venues — `/api/venues`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create venue |
| PUT | `/{venueId}` | [ORGANIZER\|ADMIN] | Update venue |
| GET | `/{venueId}` | [Public] | Get venue |
| GET | `/` | [Public] | List venues (paginated) |
| GET | `/search?keyword=&page=&size=` | [Public] | Search venues |
| GET | `/city/{city}` | [Public] | Venues in a city |
| GET | `/cities` | [Public] | All unique city names |
| DELETE | `/{venueId}` | [ADMIN] | Delete venue |

---

## 8. Ticket Types — `/api/events/{eventId}/ticket-types`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create ticket type (status=DRAFT) |
| GET | `/{ticketTypeId}` | [Public] | Get ticket type |
| GET | `/` | [Public] | All ticket types for event |
| GET | `/public` | [Public] | Visible (public) ticket types only |
| GET | `/paged?page=&size=` | [ORGANIZER\|ADMIN] | Paginated list |
| GET | `/available` | [Public] | Available ticket types (stock > 0) |
| GET | `/early-bird` | [Public] | Early bird types only |
| PUT | `/{ticketTypeId}` | [ORGANIZER\|ADMIN] | Update ticket type |
| PATCH | `/{ticketTypeId}/activate` | [ORGANIZER\|ADMIN] | Activate (DRAFT → ACTIVE) |
| PATCH | `/{ticketTypeId}/deactivate` | [ORGANIZER\|ADMIN] | Deactivate |
| DELETE | `/{ticketTypeId}` | [ORGANIZER\|ADMIN] | Delete DRAFT (no bookings) |

### CreateTicketTypeRequest
```json
{
  "name": "string (2-100)",
  "description": "string (0-500, optional)",
  "price": 0,
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

---

## 9. Event Images — `/api/events/{eventId}/images`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/cover` (multipart) | [ORGANIZER\|ADMIN] | Upload/replace cover image |
| POST | `/gallery` (multipart) | [ORGANIZER\|ADMIN] | Add gallery image |
| DELETE | `/gallery?url=` | [ORGANIZER\|ADMIN] | Remove gallery image by URL |
| DELETE | `/` | [ADMIN] | Delete all images |

---

## 10. Event Files — `/api/events/{eventId}/files`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` (multipart) | [ORGANIZER\|ADMIN] | Upload file attachment |
| GET | `/` | [Public] | List files for event |
| DELETE | `/{fileId}` | [ORGANIZER\|ADMIN] | Delete file |

---

## 11. Orders — `/api/orders`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [Auth] | Create order (status=PENDING) |
| POST | `/checkout` | [Auth] | Process payment for order |
| GET | `/{orderId}` | [Auth] | Get order (owner or admin) |
| GET | `/my-orders?page=&size=` | [Auth] | Caller's orders |
| GET | `/organizer?page=&size=&status=` | [ORGANIZER\|ADMIN] | Orders for organizer's events |
| PATCH | `/{orderId}/cancel` | [Auth] | Cancel PENDING order |

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
  "id": 1,
  "userId": "uuid",
  "userEmail": "...",
  "eventId": "uuid",
  "eventVersion": 1,
  "eventSnapshot": { ... },
  "status": "PENDING",
  "totalAmount": "200000.00",
  "currency": "VND",
  "items": [
    {
      "id": 1,
      "ticketTypeId": 1,
      "quantity": 2,
      "unitPrice": "100000.00",
      "ticketTypeSnapshot": { ... },
      "tickets": []
    }
  ],
  "payments": [],
  "createdAt": "...",
  "updatedAt": "..."
}
```

---

## 12. Tickets — `/api/orders/tickets`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/my-tickets` | [Auth] | All tickets belonging to caller |
| GET | `/{ticketId}` | [Auth] | Get ticket by ID |
| GET | `/qr/{qrPayload}` | [ORGANIZER\|ADMIN] | Get ticket by QR payload |
| POST | `/{ticketId}/check-in` | [ORGANIZER\|ADMIN] | Confirm check-in by ticket ID |
| POST | `/validate` | [ORGANIZER\|ADMIN] | Preview scan (VALID result, no state change) |
| POST | `/scan` | [ORGANIZER\|ADMIN] | Confirm scan (SUCCESS, marks ticket USED) |

> `POST /validate` and `POST /scan` both accept `ScanRequest` and both write a `ScanLog` row.
> The difference: `/validate` does NOT mark the ticket as `USED`.

### ScanRequest
```json
{
  "qrPayload": "string",
  "eventId": "uuid"
}
```

### ScanTicketResponse
```json
{
  "success": true,
  "result": "SUCCESS",
  "ticket": { ... },
  "message": "Check-in confirmed"
}
```

---

## 13. Scan Logs — `/api/orders/events/{eventId}/scan-logs`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/orders/events/{eventId}/scan-logs` | [ORGANIZER\|ADMIN] | All scan logs for event |

### ScanLogResponse
```json
{
  "id": 1,
  "ticketId": 1,
  "eventId": "uuid",
  "scannedBy": "uuid",
  "scannedByName": "...",
  "scannedAt": "...",
  "result": "SUCCESS",
  "qrPayload": "..."
}
```

---

## 14. Payment — `/api/payment`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/initiate` | [Auth] | Initiate payment for an order |
| POST | `/callback` | [Public] | Payment gateway webhook |
| POST | `/refund` | [ORGANIZER\|ADMIN] | Initiate refund |
| GET | `/order/{orderId}` | [Auth] | Payment for an order |
| GET | `/{paymentId}` | [Auth] | Payment by ID |
| GET | `/user/{userId}` | [ORGANIZER\|ADMIN or own] | Payments by user |

### PaymentRequest
```json
{
  "orderId": 1,
  "provider": "MOMO",
  "amount": "200000.00",
  "returnUrl": "https://...",
  "cancelUrl": "https://..."
}
```

### PaymentResponse
```json
{
  "paymentId": 1,
  "orderId": 1,
  "amount": "200000.00",
  "status": "PENDING",
  "provider": "MOMO",
  "transactionId": null,
  "paymentUrl": "https://payment-gateway/...",
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Idempotency key format:** `pay_u{userId}_o{orderId}`

---

## 15. Activity Log — `/api/activity-log`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/stream` (SSE) | [Auth] | Open SSE connection for activity updates |
| GET | `/?action=&entityType=&from=&to=&page=&size=` | [Auth] | Query audit log |

---

## Error Response Shape

```json
{
  "success": false,
  "message": "TicketType is ACTIVE — price cannot be edited",
  "data": null
}
```

HTTP status codes:
- `400` — validation error, forbidden state transition
- `401` — unauthenticated
- `403` — insufficient role
- `404` — entity not found
- `409` — conflict (e.g., duplicate idempotency key)
- `500` — unhandled server error
