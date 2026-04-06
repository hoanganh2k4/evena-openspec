# API: Events, Categories & Venues

---

## `/api/events`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create event (status=DRAFT) |
| PUT | `/{eventId}` | [ORGANIZER\|ADMIN] | Update event |
| GET | `/{eventId}` | [Public] | Get by ID |
| GET | `/` | [Public] | Upcoming events (PUBLISHED + ONGOING) |
| GET | `/search` | [Public] | Search with filters |
| GET | `/organizer/{organizerId}` | [Public] | Events by organizer |
| GET | `/my-events` | [ORGANIZER\|ADMIN] | Caller's events |
| PATCH | `/{eventId}/publish` | [ORGANIZER\|ADMIN] | Publish (DRAFT → PUBLISHED) |
| PATCH | `/{eventId}/cancel` | [ORGANIZER\|ADMIN] | Cancel |
| DELETE | `/{eventId}` | [ORGANIZER\|ADMIN] | Delete DRAFT |

### EventSearchRequest (query params)

| Param | Description |
|---|---|
| `keyword` | Full-text on title + description |
| `categoryId` | Filter by category |
| `city` | Filter by venue city |
| `startDate` | Range start (ISO date) |
| `endDate` | Range end (ISO date) |
| `status` | Explicit status (organizer/admin only; customers see PUBLISHED+ONGOING) |
| `page` | default 0 |
| `size` | default 10 |

### CreateEventRequest
```json
{
  "title": "string (3-200)",
  "description": "string (10+)",
  "startAt": "2026-06-01T09:00:00",
  "endAt": "2026-06-01T18:00:00",
  "organizationId": 1,
  "categoryId": 1,
  "venueId": 1,
  "coverUrl": "https://... (optional)",
  "imageUrls": ["https://..."]
}
```

### EventResponse
```json
{
  "id": "uuid", "title": "...", "description": "...",
  "startAt": "...", "endAt": "...", "status": "PUBLISHED",
  "coverUrl": "...", "eventVersion": 1,
  "organization": { "id": 1, "name": "..." },
  "category": { "id": 1, "name": "..." },
  "venue": { "id": 1, "name": "...", "city": "..." },
  "images": [], "ticketTypes": [],
  "createdAt": "...", "updatedAt": "..."
}
```

---

## `/api/events/{eventId}/images`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/cover` (multipart) | [ORGANIZER\|ADMIN] | Upload/replace cover |
| POST | `/gallery` (multipart) | [ORGANIZER\|ADMIN] | Add gallery image |
| DELETE | `/gallery?url=` | [ORGANIZER\|ADMIN] | Remove gallery image |
| DELETE | `/` | [ADMIN] | Delete all images |

---

## `/api/events/{eventId}/files`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` (multipart) | [ORGANIZER\|ADMIN] | Upload file |
| GET | `/` | [Public] | List files |
| DELETE | `/{fileId}` | [ORGANIZER\|ADMIN] | Delete file |

---

## `/api/categories`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ADMIN] | Create |
| PUT | `/{categoryId}` | [ADMIN] | Update |
| GET | `/{categoryId}` | [Public] | Get |
| GET | `/` | [Public] | List all |
| DELETE | `/{categoryId}` | [ADMIN] | Delete |

---

## `/api/venues`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create |
| PUT | `/{venueId}` | [ORGANIZER\|ADMIN] | Update |
| GET | `/{venueId}` | [Public] | Get |
| GET | `/` | [Public] | List (paginated) |
| GET | `/search?keyword=&page=&size=` | [Public] | Search |
| GET | `/city/{city}` | [Public] | Venues in city |
| GET | `/cities` | [Public] | All city names |
| DELETE | `/{venueId}` | [ADMIN] | Delete |
