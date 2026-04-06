# Object: Category

**Table:** `categories` | **PK:** Long | **Module:** event

## Schema

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| name | String | UNIQUE, NOT NULL |
| description | String | nullable |
| icon_url | String | nullable |

## Rules

- Managed by **Admin only**
- Cannot be deleted if events are assigned to the category
