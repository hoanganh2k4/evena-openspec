# Object: Venue

**Table:** `venues` | **PK:** Long | **Module:** event

## Schema

| Field | Type | Constraints |
|---|---|---|
| id | Long | PK, auto-increment |
| name | String | NOT NULL |
| address | String | NOT NULL |
| city | String | NOT NULL |
| lat | Double | nullable |
| lng | Double | nullable |
| capacity | Integer | nullable |
| description | String | TEXT, nullable |

## Rules

- Managed by **Admin only** (deletion)
- Organizers may create venues
- `city` is used for location-based event filtering
- `venue` is a contractual field on Event — cannot change after event is PUBLISHED
