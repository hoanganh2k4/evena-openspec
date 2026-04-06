# Enum: EventStatus

| Value | Meaning |
|---|---|
| `DRAFT` | Not published; invisible to customers |
| `PUBLISHED` | Public; bookings accepted |
| `ONGOING` | Event has started; no new bookings |
| `COMPLETED` | Event ended or sold out |
| `CANCELLED` | Cancelled; all paid bookings must be refunded |

## Transitions

```
DRAFT     → PUBLISHED   (organizer/admin publish)
PUBLISHED → ONGOING     (auto: EventScheduler when startAt reached)
PUBLISHED → CANCELLED   (organizer/admin)
ONGOING   → COMPLETED   (auto: EventScheduler when endAt reached)
ONGOING   → CANCELLED   (organizer/admin)
COMPLETED → PUBLISHED   (auto: when booking cancellation frees capacity)
COMPLETED → CANCELLED   (organizer/admin)
```

## Notes

- Customer-visible statuses: `PUBLISHED`, `ONGOING` (rules.md §1.1)
- Auto-transitions run every 5 minutes via `EventScheduler`
- `CANCELLED` is terminal for data purposes — event cannot be deleted
