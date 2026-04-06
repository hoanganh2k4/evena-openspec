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

## QR payload format

```
{ticketId}:{userId}:{nonce}:{hmac}
```

| Part | Description |
|---|---|
| `ticketId` | Long — DB primary key |
| `userId` | UUID — current ticket owner at time of generation |
| `nonce` | 32-char hex (UUID without hyphens) — ensures uniqueness per generation |
| `hmac` | 64-char hex — `HMAC-SHA256("ticketId:userId:nonce", QR_SECRET)` |

**Max length:** ~145 chars (fits in VARCHAR(255))

### Scan verification steps

1. `QRCodeService.verifyAndExtractTicketId(payload)` — verify HMAC (constant-time). If invalid → `INVALID_QR`.
2. `ticketRepository.findById(ticketId).filter(t -> t.qrPayload.equals(payload))` — PK lookup + exact match. If not found → `INVALID_QR` (QR may have been rotated by FlexPass transfer).

### Why PK lookup instead of text-column lookup

- Indexed PK lookup is faster than full text scan on `qr_payload`
- The exact-payload filter is the **FlexPass rotation guard**: after a transfer the field is overwritten with a new HMAC payload, so the old QR fails the filter even though its HMAC is structurally valid

## FlexPass notes

- `user_id` and `qr_payload` are the only fields mutable after issuance (transfer only)
- `transfer_count` increments to 1 on transfer completion and is then immutable
- See [flexpass/ticket-transfer.md](../flexpass/ticket-transfer.md)
