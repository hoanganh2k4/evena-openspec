# Service: EventScheduler

Spring `@Scheduled` tasks for automatic event status transitions.

| Task | Condition | Transition | Interval |
|---|---|---|---|
| `updateOngoingEvents()` | `status=PUBLISHED AND startAt < now AND endAt > now` | PUBLISHED → ONGOING | 5 min |
| `updateCompletedEvents()` | `status=ONGOING AND endAt ≤ now` | ONGOING → COMPLETED | 5 min |

Both tasks:
- Loop through results and call `eventRepository.save()` per event
- Catch and log errors per event without stopping the full run

> **Sold-out transition** (`ACTIVE → SOLD_OUT` when `sold >= total`) is driven by
> `OrderService` at booking time — not by the scheduler.

> **FlexPass expiry** (`expireListings()`) runs on a separate schedule in `FlexPassService`.
