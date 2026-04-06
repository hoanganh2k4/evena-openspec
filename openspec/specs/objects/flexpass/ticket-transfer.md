# Object: TicketTransfer

**Table:** `ticket_transfers` | **PK:** Long | **Module:** flexpass
**Indexes:** `ticket_id`, `seller_id`, `status`

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| ticket_id | Long | FK → tickets.id, NOT NULL | |
| seller_id | UUID | FK → users.id, NOT NULL | Original owner |
| buyer_id | UUID | FK → users.id, nullable | Set when buyer purchases |
| listing_price | BigDecimal | precision=10, scale=2, NOT NULL | ≤ `original_price × 1.20` |
| original_price | BigDecimal | precision=10, scale=2, NOT NULL | Snapshot from `order_items.unit_price` |
| status | TicketTransferStatus | NOT NULL, default=`PENDING_APPROVAL` | see [enums/ticket-transfer-status.md](../../enums/ticket-transfer-status.md) |
| expires_at | LocalDateTime | nullable | Resale window end |
| completed_at | LocalDateTime | nullable | Set on COMPLETED |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Relationships

- ManyToOne → `Ticket`
- ManyToOne → `User` (seller)
- ManyToOne → `User` (buyer, nullable)

## Constraints

- Only **one** active transfer per ticket at a time
  (`status ∈ {PENDING_APPROVAL, APPROVED, PAYMENT_PENDING}`)
- `listing_price ≤ original_price × 1.20` (enforced by backend)
- Ticket must have `transfer_count == 0` to be listed
- Ticket must not have started (`event.startAt > now`)

## Atomic completion (rules.md §11.4)

When transfer reaches `COMPLETED`, these 6 steps execute in **one `@Transactional`**:

```
1. ticket.qr_payload   ← new HMAC-signed payload   (old QR becomes INVALID_QR)
2. ticket.user_id      ← buyer.id
3. ticket.transfer_count ← 1
4. ticket.status       ← ACTIVE
5. transfer.status     ← COMPLETED
6. transfer.completed_at ← now()
```

Rollback on any failure — seller's QR remains valid until commit.

## State machine

See [enums/ticket-transfer-status.md](../../enums/ticket-transfer-status.md).
