# Object: ActivityLog

**Table:** `activity_logs` | **PK:** Long | **Module:** monitor

## Schema

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| actor_id | UUID | nullable |
| actor_role | ActorRole | nullable |
| action | LogAction | nullable |
| entity_type | EntityType | nullable |
| entity_id | String | nullable |
| owner_user_id | UUID | nullable |
| event_id | UUID | nullable |
| old_value | JSONB | nullable |
| new_value | JSONB | nullable |
| description | String | nullable |
| ip_address | String | nullable |
| created_at | LocalDateTime | auto |

## Rules

- Rows are **append-only** — never updated or deleted
- Logging is `@Async` — failures must not roll back the triggering transaction
- `old_value` and `new_value` store full JSON snapshots of the entity before/after change
- All schema columns are nullable at DB level (to tolerate async failures), but **semantically required for a meaningful audit entry:** `actor_id`, `action`, `entity_type`, `entity_id` — these MUST be populated by every caller

## Audit-required actions (rules.md §7.3)

- Organization verification / rejection / archiving
- Event publish / cancel / post-publish metadata edit
- TicketType activation / deactivation / capacity change
- Booking creation
- Order confirmation
- Refund initiation
- FlexPass listing, approval, transfer completion

See [enums/audit.md](../../enums/audit.md) for full `LogAction` list.
