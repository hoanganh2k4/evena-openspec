# Object: FlexPassSaleWindow

**Table:** `flexpass_sale_windows` | **PK:** Long | **Module:** flexpass
**Indexes:** `event_id`, `status`

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| event_id | UUID | FK → events.id, NOT NULL | |
| status | SaleWindowStatus | NOT NULL, default=`SCHEDULED` | SCHEDULED → ACTIVE → CLOSED |
| price_method | PriceMethod | NOT NULL | MEDIAN, MEAN, or TRIMMED_MEAN |
| sale_start | LocalDateTime | NOT NULL | When window opens (future) |
| sale_end | LocalDateTime | NOT NULL | When window closes |
| created_by | UUID | FK → users.id, NOT NULL | Organizer who created the window |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Relationships

- ManyToOne → `Event`
- OneToMany → `FlexPassSaleWindowPrice` (per-ticket-type price breakdown)
- OneToMany → `TicketTransfer` (listings locked to this window)

---

## Object: FlexPassSaleWindowPrice

**Table:** `flexpass_sale_window_prices` | **PK:** Long

Stores the calculated price breakdown per ticket type for a sale window.
Created when the organizer calls `createSaleWindow`.

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| sale_window_id | Long | FK → flexpass_sale_windows.id, NOT NULL | |
| ticket_type_id | Long | FK → ticket_types.id, NOT NULL | |
| listing_count | Integer | NOT NULL | Number of APPROVED listings at calculation time |
| median_price | BigDecimal | precision=10, scale=2, NOT NULL | |
| mean_price | BigDecimal | precision=10, scale=2, NOT NULL | |
| trimmed_mean_price | BigDecimal | precision=10, scale=2, NOT NULL | 20% trim (10% each end) |
| selected_price | BigDecimal | precision=10, scale=2, NOT NULL | Value of the chosen `price_method` |

## Enums

### SaleWindowStatus

| Value | Meaning |
|---|---|
| `SCHEDULED` | Window created; not yet open; can be cancelled |
| `ACTIVE` | Window is open; listings are PRICE_LOCKED; purchases allowed |
| `CLOSED` | Window ended; unsold PRICE_LOCKED listings → EXPIRED |
| `CANCELLED` | Organizer cancelled before opening; listings remain APPROVED |

### PriceMethod

| Value | Formula |
|---|---|
| `MEDIAN` | Middle value of sorted `submitted_price` values |
| `MEAN` | Arithmetic average of all `submitted_price` values |
| `TRIMMED_MEAN` | Trim each end by `floor(listingCount × 0.10)`, min 1 when listingCount ∈ [3,9], no trim when listingCount < 3 — see rules.md §9.9 |

## Constraints

- Only **one** `SCHEDULED` or `ACTIVE` window per event at a time
- `sale_start > now` at creation time
- `sale_start < sale_end`
- Cannot cancel an `ACTIVE` window — only `SCHEDULED` windows can be cancelled
- `selected_price` and per-metric prices are computed at creation time and **never recalculated**
  (even if new listings are added after window is created)

## State machine

```
SCHEDULED → ACTIVE     (auto: EventScheduler when sale_start is reached)
ACTIVE    → CLOSED     (auto: EventScheduler when sale_end is reached)
SCHEDULED → CANCELLED  (organizer cancels before sale_start)
```
