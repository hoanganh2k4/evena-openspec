# API — Conventions

**Base URL:** `http://localhost:8080`
**Auth scheme:** Bearer JWT (`Authorization: Bearer <token>`); refresh token in `refreshToken` httpOnly cookie

## Response envelope

```json
{ "success": true, "message": "...", "data": <T> }
```

Paginated responses wrap `data` as:
```json
{ "content": [], "totalElements": 0, "totalPages": 0, "number": 0 }
```

## Auth annotations

| Tag | Meaning |
|---|---|
| `[Public]` | No auth required |
| `[Auth]` | Any authenticated user |
| `[ORGANIZER\|ADMIN]` | Must have ORGANIZER or ADMIN role |
| `[ADMIN]` | Admin only |

## Error response

```json
{ "success": false, "message": "TicketType is ACTIVE — price cannot be edited", "data": null }
```

HTTP codes: `400` validation · `401` unauthenticated · `403` forbidden · `404` not found · `409` conflict · `500` server error

## Endpoints by domain

| File | Base path |
|---|---|
| [auth.md](auth.md) | `/api/auth`, `/api/users` |
| [organization.md](organization.md) | `/api/organizations`, `/api/invitations` |
| [event.md](event.md) | `/api/events`, `/api/categories`, `/api/venues` |
| [ticket-type.md](ticket-type.md) | `/api/events/{eventId}/ticket-types` |
| [order.md](order.md) | `/api/orders` |
| [payment.md](payment.md) | `/api/payment` |
| [flexpass.md](flexpass.md) | `/api/flexpass` |
| [monitor.md](monitor.md) | `/api/activity-log` |
