# Tasks: SSE Integrity Compliance

**ID:** TASK-002
**Spec:** SPEC-002
**Author:** team
**Date:** 2026-03-10
**Status:** open

---

## Checklist

### backend

- [ ] **T1** — `SSENotificationService.java`: Change `notifyEventCreated` signature
  from `(Long organizationId, Object payload)` to
  `(UUID eventId, String eventName, Long organizationId, String eventStatus)`.
  Inside the method: choose `"organizer,admin"` if `DRAFT`, `"public,organizer"` if
  `PUBLISHED`. Build minimal `{eventId, eventName, organizationId}` data map.

- [ ] **T2** — `SSENotificationService.java`: Change `notifyEventUpdated` to accept
  `String eventStatus` parameter and choose channel accordingly (same logic as T1).

- [ ] **T3** — `SSENotificationService.java`: Change `notifyTicketTypeCreated`,
  `notifyTicketTypeUpdated`, `notifyTicketTypeDeleted` to accept
  `boolean isEventPublished` parameter. Choose `"organizer,admin"` if `!isEventPublished`,
  `"public,organizer"` if published.

- [ ] **T4** — `EventService.java`: Update calls to `notifyEventCreated()` and
  `notifyEventUpdated()` to pass `savedEvent.getStatus().name()` (or
  `event.isPublished()`).

- [ ] **T5** — `TicketTypeService.java`: Update calls to `notifyTicketTypeCreated()`,
  `notifyTicketTypeUpdated()`, `notifyTicketTypeDeleted()` to pass
  `event.isPublished()`.

### frontend

- [ ] **T6** — `src/providers/SSEProvider.tsx`: In `ORDER_CONFIRMED` case,
  remove `TicketTypeAPI.util.invalidateTags` and `EventAPI.util.invalidateTags`
  calls. Keep only `OrderAPI.util.invalidateTags(['Order'])` and
  `OrderAPI.util.invalidateTags(['Ticket'])`.

- [ ] **T7** — `src/providers/SSEProvider.tsx`: In `ORDER_CANCELLED` and
  `ORDER_EXPIRED` cases, verify they also don't invalidate Event/TicketType
  unnecessarily. Fix if needed.

- [ ] **T8** — `src/stores/services/EventApi.ts`: Add `status: 'PUBLISHED'` param
  to `getPublicEvents` query. Remove the comment "No status filter - show all
  events to customers".

- [ ] **T9** — `app/dashboard/customer/cart/[id]/page.tsx`: Remove all
  `console.log` calls (lines ~58, ~65).

- [ ] **T10** — `app/dashboard/customer/events/[id]/page.tsx`: Remove all
  `console.log` calls (lines ~38, ~44, ~54).

---

## Notes

- T1–T5 are backend changes that REQUIRE coordinated deployment with T6–T10
  (frontend). If backend is deployed first, DRAFT event SSE will stop reaching
  customers before the frontend is updated — this is safe.
- T8 (status filter) must be verified against the backend `/events` endpoint.
  If the backend does not enforce PUBLISHED filter, it must be fixed in a
  separate backend task.
- After T6, monitor SSE-triggered refetch count in browser devtools. The number
  of Event/TicketType refetches after a payment should drop to zero.
