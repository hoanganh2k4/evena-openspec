# Object: OrderItem

**Table:** `order_items` | **PK:** Long | **Module:** order

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| order_id | Long | FK → orders.id, NOT NULL | |
| ticket_type_id | Long | FK → ticket_types.id, NOT NULL | |
| quantity | Integer | NOT NULL | |
| unit_price | BigDecimal | precision=10, scale=2, NOT NULL | Price snapshot at purchase time |
| ticket_type_snapshot | TEXT | NOT NULL | JSON blob — full TicketType snapshot |

## Relationships

- ManyToOne → `Order` (LAZY)
- ManyToOne → `TicketType` (LAZY)
- OneToMany → `Ticket` (cascade ALL, orphanRemoval)

## Helper

- `getSubtotal()` → `unit_price × quantity`

## Rules

- `unit_price` is the price at time of purchase — never updated
- `ticket_type_snapshot` is stored at creation and never modified
- One `Ticket` is issued per unit of quantity after payment confirmation
