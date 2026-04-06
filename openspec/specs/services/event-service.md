# Service: EventService

### `createEvent(CreateEventRequest)`
1. Verify caller has org access
2. Guard: `organization.verified == true`
3. Set `status=DRAFT`, `eventVersion=1`
4. Persist; log `EVENT_CREATED`

### `updateEvent(UUID eventId, UpdateEventRequest)`
1. Load event; verify ownership
2. If `status ∈ {PUBLISHED, ONGOING, COMPLETED, CANCELLED}`: block contractual field changes
3. Increment `eventVersion` if contractual fields changed
4. Persist; log `EVENT_UPDATED`

### `publishEvent(UUID eventId)`
1. Guard: `organization.verified == true`
2. Guard: `status == DRAFT`
3. Set `status=PUBLISHED`
4. Emit SSE `event:publish` → `public` + `organizer`
5. Log `EVENT_APPROVED`

### `cancelEvent(UUID eventId)`
1. Guard: caller has org access
2. Set `status=CANCELLED`
3. Cascade: all TicketTypes → `DEACTIVATED`
4. Cascade: all `PENDING` orders → `CANCELLED`, release sold counts, emit `order:cancel` per user
5. Emit SSE `event:cancel` → `public` + `organizer`
6. Log cancel action

> `CONFIRMED` orders are NOT cancelled — they require explicit refund flow.

### `searchEvents(EventSearchRequest)`
- Default customer filter: `status IN (PUBLISHED, ONGOING)`
- Organizer/admin: can filter by any status
- Supports: keyword, categoryId, city, dateRange
