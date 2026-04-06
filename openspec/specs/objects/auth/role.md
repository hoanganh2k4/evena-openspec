# Object: Role

**Table:** `roles` | **PK:** Long | **Module:** auth

## Schema

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| name | String | UNIQUE, NOT NULL |
| description | String | nullable |

## Seed values

| Name | Description |
|---|---|
| `USER` | Regular customer |
| `ORGANIZER` | Event organizer |
| `ADMIN` | System administrator |

## Rules

- Roles are seeded at startup and not managed via API
- A user may have multiple roles (ManyToMany)
- Role names are used in `@PreAuthorize` annotations
