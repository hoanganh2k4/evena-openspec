# API: Monitor / Activity Log

**Base:** `/api/activity-log`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/stream` (SSE) | [Auth] | Open SSE connection for activity feed |
| GET | `/` | [Auth] | Query audit log with filters |
| GET | `/stats` | ADMIN | Summary stats for the last 24 h vs previous 24 h |
| GET | `/hourly` | ADMIN | Per-hour event counts for the last 24 h |

### Query params — `GET /`

| Param | Type | Description |
|---|---|---|
| `action` | LogAction | Filter by action type |
| `entityType` | EntityType | Filter by entity type |
| `entityId` | String | Filter by entity UUID / ID |
| `actorId` | UUID | Filter by actor UUID |
| `from` | ISO datetime | Range start |
| `to` | ISO datetime | Range end |
| `sort` | `ASC` \| `DESC` | Sort direction on `createdAt` (default `DESC`) |
| `page` | int | default 0 |
| `size` | int | default 20 |

### ActivityLogDTO
```json
{
  "id": 1, "actorId": "uuid", "actorName": "Nguyen Minh",
  "actorRole": "ORGANIZER",
  "action": "EVENT_UPDATED", "entityType": "EVENT",
  "entityId": "uuid", "eventId": "uuid",
  "oldValue": { ... }, "newValue": { ... },
  "description": "Title changed from X to Y",
  "createdAt": "2026-05-07T08:11:00"
}
```

### ActivityLogStatsDTO — `GET /stats`
```json
{
  "activitiesToday": 142,
  "criticalToday": 7,
  "activeActors": 23,
  "topAction": "EVENT_UPDATED",
  "topActionCount": 38,
  "yesterdayCount": 120,
  "criticalYesterday": 4
}
```

### HourlyCountDTO — `GET /hourly`
Returns array of 24 entries (one per hour bucket), oldest first.
```json
[
  { "hourLabel": "00:00", "count": 3, "isSpike": false },
  { "hourLabel": "02:00", "count": 21, "isSpike": true },
  ...
]
```
`isSpike` is `true` when `count > mean * 2.0` across all 24 buckets.
