# Service: TicketTypeService

### `createTicketType(UUID eventId, CreateTicketTypeRequest)`
1. Verify event exists and caller has access
2. Set `status=DRAFT`
3. Persist; log `TICKET_TYPE_ADDED`

### `activateTicketType(UUID eventId, Long ticketTypeId)`
Guards:
- `price ≥ 0` and meets CurrencyMinimum if paid
- `total > 0`
- `salesStart` and `salesEnd` set, `salesStart < salesEnd`
- Event not CANCELLED

Transition `DRAFT → ACTIVE`; log `TICKET_TYPE_UPDATED`.

### `updateTicketType(UUID eventId, Long ticketTypeId, UpdateTicketTypeRequest)`
1. If `status == ACTIVE`: reject changes to price, currency, capacity decrease, benefits, refundPolicy
2. Capacity increase after ACTIVE allowed but logged
3. `name`, `description`, `visible`: always editable
4. Persist; log `TICKET_TYPE_UPDATED`

### `deactivateTicketType(UUID eventId, Long ticketTypeId)`
1. Set `status=DEACTIVATED`
2. MUST NOT delete if has ever been ACTIVE or has bookings
3. Log `TICKET_TYPE_DEACTIVATED`

### `deleteTicketType(UUID eventId, Long ticketTypeId)`
Allowed only if:
- `status == DRAFT`
- Zero `OrderItem` records linked
