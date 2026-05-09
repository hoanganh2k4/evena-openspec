# API: Relation Explorer (Admin)

**Base:** `/api/admin/relations`
**Auth:** ADMIN role required for all endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/event/{eventId}` | Full relation context for a given event |
| GET | `/event/search` | Search events by keyword for entity picker |

---

## GET /event/{eventId}

Returns a composite relation summary for the given event UUID.

### EventRelationSummaryDTO
```json
{
  "event": {
    "id": "uuid", "title": "Hanoi Acoustic Night 2026",
    "organizationId": 4, "organizationName": "Hanoi Music Co."
  },
  "directCustomers": 347,
  "coPurchasedEvents": [
    { "eventId": "uuid", "title": "Saigon Acoustic Night", "sharedCustomers": 236, "overlapPct": 68 }
  ],
  "customerSegments": [
    { "name": "Loyal fans",       "count": 142, "description": "≥3 events from this organizer", "color": "purple" },
    { "name": "Casual attendees", "count": 98,  "description": "1–2 events, low engagement",    "color": "blue"   },
    { "name": "First-time buyers","count": 52,  "description": "First event on the platform",   "color": "green"  },
    { "name": "Group buyers",     "count": 55,  "description": "Bought 4+ tickets in one order","color": "orange" }
  ],
  "networkNodes": [
    { "id": "event-<uuid>",  "label": "EVENT",  "type": "EVENT",  "size": 48, "cx": 300, "cy": 220 },
    { "id": "org-4",         "label": "Hanoi Music Co.", "type": "ORG", "size": 32, "cx": 300, "cy": 60 }
  ],
  "networkEdges": [
    { "source": "event-<uuid>", "target": "org-4", "weight": 1.0, "label": "owned by" }
  ],
  "suspiciousPatterns": [
    { "type": "MULTI_ACCOUNT_IP", "description": "5 accounts shared IP 10.0.0.1, 47 tickets total", "severity": "HIGH", "count": 5 },
    { "type": "RAPID_CO_PURCHASE", "description": "12 users bought 2 events within 1 minute", "severity": "MEDIUM", "count": 12 }
  ]
}
```

### Node types
| type | Color | Description |
|---|---|---|
| `EVENT` | Purple | The explored event (center) |
| `ORG` | Blue | Owning organization |
| `SEGMENT_LOYAL` | Purple light | Loyal fans cluster |
| `SEGMENT_CASUAL` | Blue light | Casual attendees cluster |
| `SEGMENT_FIRST` | Green | First-time buyers cluster |
| `SEGMENT_GROUP` | Orange | Group buyers cluster |
| `CO_EVENT` | Teal | Co-purchased event |
| `SUSPICIOUS` | Red-orange | Suspicious cluster |

### Suspicious pattern types
| type | Trigger |
|---|---|
| `MULTI_ACCOUNT_IP` | ≥2 distinct actorIds with same IP in ActivityLog for this event, with ≥5 tickets total |
| `RAPID_CO_PURCHASE` | ≥5 users who bought this event AND another event within 60 seconds |

---

## GET /event/search

Search events to use in the entity picker.

| Param | Type | Description |
|---|---|---|
| `q` | String | Keyword to match event title |
| `size` | int | default 10, max 20 |

Returns `List<EventSearchResultDTO>`:
```json
[{ "id": "uuid", "title": "Hanoi Acoustic Night 2026", "organizationName": "Hanoi Music Co.", "status": "PUBLISHED" }]
```
