# Service: EventScheduler

Spring `@Scheduled` tasks for automatic event status transitions.

| Task | Condition | Transition | Interval |
|---|---|---|---|
| `updateOngoingEvents()` | `status=PUBLISHED AND startAt < now AND endAt > now` | PUBLISHED → ONGOING | 5 min |
| `updateCompletedEvents()` | `status IN (ONGOING, PUBLISHED) AND endAt ≤ now` | ONGOING/PUBLISHED → COMPLETED | 5 min |

> **Note:** `updateCompletedEvents()` covers both the normal `ONGOING → COMPLETED` path and the edge case where a PUBLISHED event's `endAt` passes without the scheduler having observed the ONGOING window (scheduler downtime, or event published after its end time).

Both tasks:
- Loop through results and call `eventRepository.save()` per event
- Catch and log errors per event without stopping the full run

> **Sold-out transition** (`ACTIVE → SOLD_OUT` when `sold >= total`) is driven by
> `OrderService` at booking time — not by the scheduler.

> **FlexPass expiry** (`expireListings()`) runs on a separate schedule in `FlexPassService`.
