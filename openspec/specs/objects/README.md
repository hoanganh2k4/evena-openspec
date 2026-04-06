# Objects — Entity Relationship Overview

**Applies to:** backend · frontend · mobile

## Global conventions

- `User`, `Event` PK: **UUID** (UUIDv7)
- All other entities PK: **Long** (auto-increment)
- All entities extend `AuditableEntity` → `created_at`, `updated_at` (auto)
- Soft-delete / status flags used instead of hard-delete for any entity with financial data

## Entity map

```
User ──< OrganizationMember >── Organization ──< Event ──< TicketType ──< OrderItem
 │                                               │                            │
 │                                               ├──< EventImage           ──< Ticket ──< ScanLog
 │                                               └──< EventFile               │
 │                                                                             └──< TicketTransfer
 └──< Order (event_snapshot) ──< OrderItem ──< Ticket
       └──< Payment
```

## Snapshot pattern

Booking integrity requires snapshots at purchase time.
These JSON blobs are stored and **MUST NOT be replaced** after creation:

| Snapshot field | Stored in | Contains |
|---|---|---|
| `event_snapshot` | `orders` | id, title, startAt, endAt, venueName, currency |
| `ticket_type_snapshot` | `order_items` | id, name, price, currency, benefits |

See `rules.md` §5.

## Modules

| Folder | Entities |
|---|---|
| [auth/](auth/) | User, Role, PendingRegistration |
| [organization/](organization/) | Organization, OrganizationMember |
| [event/](event/) | Event, Category, Venue, EventImage, EventFile |
| [ticket-type/](ticket-type/) | TicketType |
| [order/](order/) | Order, OrderItem, Ticket, ScanLog |
| [payment/](payment/) | Payment |
| [flexpass/](flexpass/) | TicketTransfer |
| [monitor/](monitor/) | ActivityLog |
