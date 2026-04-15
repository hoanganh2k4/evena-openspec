# Object: TicketType

**Table:** `ticket_types` | **PK:** Long | **Module:** event
**Indexes:** `event_id`, `status`

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| event_id | UUID | FK → events.id, NOT NULL | |
| name | String | NOT NULL | Editable at any time |
| description | String | TEXT, nullable | Editable at any time |
| price | BigDecimal | precision=10, scale=2, NOT NULL | Contractual — **locked once ACTIVE** |
| currency | String | default=`VND` | Contractual — locked once ACTIVE |
| total | Integer | NOT NULL | Capacity |
| sold | Integer | NOT NULL, default=0 | System-managed — never set manually |
| per_user_limit | Integer | nullable | Max tickets per customer |
| sales_start | LocalDateTime | nullable | Required for activation |
| sales_end | LocalDateTime | nullable | Required for activation |
| status | TicketTypeStatus | NOT NULL, default=DRAFT | see [enums/ticket-type-status.md](../../enums/ticket-type-status.md) |
| early_bird | Boolean | default=false | |
| early_bird_discount | BigDecimal | nullable | |
| visible | Boolean | default=true | Controls public display |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Relationships

- ManyToOne → `Event` (LAZY)
- OneToMany → `OrderItem` (cascade ALL, orphanRemoval)

## Helper methods

- `getAvailable()` → `total - sold`
- `getSoldPercentage()` → `(sold / total) × 100`
- `isAvailable()` → checks availability + sales window
- `isSalesOpen()` → `now` between `salesStart` and `salesEnd`
- `canUserPurchase(quantity)` → validates `perUserLimit`

## Field lock rules (rules.md §4.3)

| Field | Rule |
|---|---|
| price, currency | Locked once ACTIVE (DB columns — see schema above) |
| benefits, refundPolicy | Locked once ACTIVE (**not DB columns** — stored in description/metadata; referenced by rules as logical concepts) |
| total (decrease below `sold`) | Forbidden **at all times** |
| total (increase) | Permitted after ACTIVE — must be logged |
| name, description, visible | Never locked |

## Activation requirements (DRAFT → ACTIVE)

- `price ≥ 0` (0 = free ticket; paid tickets must meet CurrencyMinimum)
- `total > 0`
- `salesStart` and `salesEnd` set, `salesStart < salesEnd`
- Parent event not CANCELLED

## State machine

See [enums/ticket-type-status.md](../../enums/ticket-type-status.md).
