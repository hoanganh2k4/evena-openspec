# Object: Ticket

**Table:** `tickets` | **PK:** Long | **Module:** order
**Indexes:** `user_id`, `order_item_id`

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| order_item_id | Long | FK → order_items.id, NOT NULL | |
| user_id | UUID | FK → users.id, NOT NULL | **Current owner** — changes on FlexPass transfer |
| qr_payload | String | UNIQUE, NOT NULL, length=255 | HMAC-signed (see QR format below) |
| status | TicketStatus | NOT NULL, default=ACTIVE | see [enums/ticket-status.md](../../enums/ticket-status.md) |
| transfer_count | Integer | NOT NULL, default=0 | Max 1 — enforced at FlexPass listing creation |
| issued_at | LocalDateTime | NOT NULL | |
| used_at | LocalDateTime | nullable | Set on successful check-in |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## QR payload formats

### Static QR (stored in DB — used for FlexPass rotation guard)

```
{ticketId}:{userId}:{nonce}:{hmac}
```

| Part | Description |
|---|---|
| `ticketId` | Long — DB primary key |
| `userId` | UUID — current ticket owner at time of generation |
| `nonce` | 32-char hex (UUID without hyphens) |
| `hmac` | 64-char hex — `HMAC-SHA256("ticketId:userId:nonce", QR_SECRET)` |

> Stored permanently in `ticket.qr_payload`. **Never returned directly to clients** — used only for FlexPass transfer rotation tracking.

### Time-window token (returned in API responses — rotates every 5 min)

```
TW:{ticketId}:{userId}:{timeSlot}:{hmac}
```

| Part | Description |
|---|---|
| `TW` | Literal prefix — distinguishes from static QR |
| `ticketId` | Long — DB primary key |
| `userId` | UUID — ticket owner |
| `timeSlot` | `floor(epochSeconds / 300)` — increments every 5 minutes |
| `hmac` | 64-char hex — `HMAC-SHA256("TW:ticketId:userId:timeSlot", QR_SECRET)` |

**Properties:**
- Computed on-the-fly in `OrderMapper.toTicketResponse()` — no DB write
- Screenshot valid for at most **5 minutes** (current slot only, no grace period)
- Cannot forge without `QR_SECRET`

### Scan verification priority

```
1. TW: prefix → verifyTimeWindowToken()   (current slot only)
2. 4-part     → verifyAndExtractTicketId() (static QR fallback — backward compat)
3. Other      → INVALID_QR
```

## FlexPass notes

- `user_id` and `qr_payload` are the only fields mutable after issuance (transfer only)
- `transfer_count` increments to 1 on transfer completion and is then immutable
- After transfer, new owner receives a new static `qr_payload` — old time-window tokens bound to previous `userId` become invalid automatically
- See [flexpass/ticket-transfer.md](../flexpass/ticket-transfer.md)
