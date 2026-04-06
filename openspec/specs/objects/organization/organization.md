# Object: Organization

**Table:** `organizations` | **PK:** Long | **Module:** event

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| owner_user_id | UUID | FK → users.id, NOT NULL | |
| name | String | NOT NULL | **Identity** — immutable after APPROVED |
| description | String | TEXT, nullable | Non-identity |
| logo_url | String | nullable | Non-identity |
| website | String | nullable | Non-identity |
| email | String | nullable | Non-identity |
| phone | String | nullable | Non-identity |
| verified | Boolean | default=false | `true` = APPROVED state |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

> `verified=false` = PENDING, `verified=true` = APPROVED. ARCHIVED is a separate soft-delete state.

## Relationships

- ManyToOne → `User` (owner)
- OneToMany → `Event`
- OneToMany → `OrganizationMember` (cascade ALL, orphanRemoval)

## Field classification (rules.md §2.3)

| Field | Classification | Editable after APPROVED? |
|---|---|---|
| name | Identity | No |
| logo_url, description, website, email, phone | Non-identity | Yes (organizer or admin) |
| members | Operational | Yes |

## State machine

```
PENDING  → APPROVED   (admin: verify action)
PENDING  → REJECTED   (admin action)
REJECTED → PENDING    (admin: re-review)
APPROVED → ARCHIVED   (admin: archive — not delete)
```

## Rules

- Must be `APPROVED` before creating or publishing events
- `APPROVED` organizations MUST NOT be deleted — admin may archive only
- `PENDING` with no events may be soft-deleted by owner or admin
- Admin MUST NOT archive if org has `PUBLISHED` events with active paid bookings
