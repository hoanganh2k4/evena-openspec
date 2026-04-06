# Object: Event

**Table:** `events` | **PK:** UUID (UUIDv7) | **Module:** event
**Indexes:** `organization_id`, `status`, `start_at`

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | UUIDv7 |
| title | String | NOT NULL | Contractual — locked after PUBLISHED |
| description | String | TEXT, nullable | Metadata — editable after PUBLISHED |
| start_at | LocalDateTime | NOT NULL | Contractual |
| end_at | LocalDateTime | NOT NULL | Contractual |
| status | EventStatus | NOT NULL, default=DRAFT | see [enums/event-status.md](../../enums/event-status.md) |
| cover_url | String | nullable | Metadata |
| event_version | Integer | NOT NULL, default=1 | incremented on contractual field change |
| organization_id | Long | FK → organizations.id, NOT NULL | Contractual |
| category_id | Long | FK → categories.id, NOT NULL | Contractual |
| venue_id | Long | FK → venues.id, NOT NULL | Contractual |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Relationships

- ManyToOne → `Organization` (LAZY)
- ManyToOne → `Category` (EAGER)
- ManyToOne → `Venue` (EAGER)
- OneToMany → `EventImage` (cascade ALL, orphanRemoval)
- OneToMany → `TicketType` (cascade ALL, orphanRemoval)

## Field classification (rules.md §3.3)

| Field | Classification | Editable after PUBLISHED? |
|---|---|---|
| title, start_at, end_at, venue_id, organization_id, category_id | Contractual | **No** |
| description, cover_url, images | Metadata | Yes |
| contact info, FAQ, notes | Metadata | Yes — with restrictions (rules.md §3.3) |

## Helper methods

- `isPublished()` → `status ∈ {PUBLISHED, ONGOING, COMPLETED}`
- `allowsTicketTypeModification()` → `status == DRAFT`
- `incrementVersion()` → call on any contractual field change

## State machine

See [enums/event-status.md](../../enums/event-status.md).
Auto-transitions driven by `EventScheduler` (every 5 min).

## Rules

- Organization must be `APPROVED` before creating or publishing
- DRAFT events are not visible to customers
- Published events MUST NOT be deleted — cancel instead
- Cancellation cascades: all TicketTypes → DEACTIVATED, PENDING Orders → CANCELLED
