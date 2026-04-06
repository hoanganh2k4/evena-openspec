# Object: Order

**Table:** `orders` | **PK:** Long | **Module:** order
**Indexes:** `user_id`, `event_id`, `status`, `created_at`

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| user_id | UUID | FK → users.id, NOT NULL | |
| event_id | UUID | NOT NULL | Denormalized for quick lookups |
| event_version | Integer | NOT NULL | Version at purchase time |
| event_snapshot | TEXT | NOT NULL | JSON blob — full event snapshot |
| status | OrderStatus | NOT NULL, default=PENDING | see [enums/order-status.md](../../enums/order-status.md) |
| total_amount | BigDecimal | precision=10, scale=2, NOT NULL | |
| currency | String | default=`VND` | |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Relationships

- ManyToOne → `User` (LAZY)
- OneToMany → `OrderItem` (cascade ALL, orphanRemoval)
- OneToMany → `Payment` (cascade ALL, orphanRemoval)

## Rules

- `event_snapshot` MUST be stored at creation time and **never modified**
- Order display must use snapshot data — never live event data
- Once `CONFIRMED`, `total_amount` is immutable
- Only `PENDING` orders can be cancelled by the user
- `CONFIRMED` orders require explicit refund flow to become `REFUNDED`
