# API: Monitor / Activity Log

**Base:** `/api/activity-log`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/stream` (SSE) | [Auth] | Open SSE connection for activity feed |
| GET | `/` | [Auth] | Query audit log with filters |

### Query params

| Param | Type | Description |
|---|---|---|
| `action` | LogAction | Filter by action type |
| `entityType` | EntityType | Filter by entity type |
| `from` | ISO datetime | Range start |
| `to` | ISO datetime | Range end |
| `page` | int | default 0 |
| `size` | int | default 20 |

### ActivityLogDTO
```json
{
  "id": 1, "actorId": "uuid", "actorRole": "ORGANIZER",
  "action": "EVENT_UPDATED", "entityType": "EVENT",
  "entityId": "uuid", "ownerUserId": "uuid", "eventId": "uuid",
  "oldValue": { ... }, "newValue": { ... },
  "description": "Title changed from X to Y",
  "ipAddress": "...", "createdAt": "..."
}
```
