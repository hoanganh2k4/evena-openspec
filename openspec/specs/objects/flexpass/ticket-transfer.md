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
| submitted_price | BigDecimal | precision=10, scale=2, NOT NULL | Seller's desired price; `originalPrice × 0.50 ≤ value ≤ originalPrice × 1.20` |
| final_price | BigDecimal | precision=10, scale=2, nullable | Aggregated price set when sale window opens; buyers pay this |
| original_price | BigDecimal | precision=10, scale=2, NOT NULL | Snapshot from `order_items.unit_price` |
| sale_window_id | Long | FK → flexpass_sale_windows.id, nullable | Set when PRICE_LOCKED |
| status | TicketTransferStatus | NOT NULL, default=`PENDING_APPROVAL` | see [enums/ticket-transfer-status.md](../../enums/ticket-transfer-status.md) |
| expires_at | LocalDateTime | nullable | 14-day TTL from listing creation |
| completed_at | LocalDateTime | nullable | Set on COMPLETED |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Relationships

- ManyToOne → `Ticket`
- ManyToOne → `User` (seller)
- ManyToOne → `User` (buyer, nullable)

## Constraints

- Only **one** active transfer per ticket at a time
  (`status ∈ {PENDING_APPROVAL, APPROVED, PRICE_LOCKED, PAYMENT_PENDING}`)
- `original_price × 0.50 ≤ submitted_price ≤ original_price × 1.20` (enforced by backend)
- `final_price` is immutable once set — MUST NOT be changed after window opens
- Ticket must have `transfer_count == 0` to be listed
- `event.startAt > now` and `event.status == PUBLISHED` at listing creation time

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
