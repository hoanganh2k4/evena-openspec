# Services: SSENotificationService & EmailService

---

## SSENotificationService

Manages SSE emitters by channel name. See [sse-flow-spec.md](../sse-flow-spec.md) for full rules.

### Channel naming

| Channel | Audience |
|---|---|
| `public` | All users (including unauthenticated) |
| `organizer` | All organizers |
| `admin` | Admin users |
| `user:{userId}` | Specific user (private) |

### Events emitted

| Event | Channel(s) | Trigger |
|---|---|---|
| `event:publish` | `public`, `organizer` | Event published |
| `event:cancel` | `public`, `organizer` | Event cancelled |
| `event:update` | `public`, `organizer` | Metadata updated |
| `order:confirm` | `user:{userId}` | Payment confirmed |
| `order:cancel` | `user:{userId}` | Order cancelled |
| `order:refund` | `user:{userId}` | Refund processed |
| `tickettype:update` | `public` | TicketType capacity changed |
| `venue:update` | `public`, `admin` | Venue updated |
| `flexpass:sold` | `user:{sellerId}`, `user:{buyerId}` | Transfer completed |
| `flexpass:failed` | `user:{sellerId}`, `user:{buyerId}` | Transfer failed |

---

## EmailService

All emails are sent **asynchronously**. Failures are logged and do not roll back the triggering transaction.

| Trigger | Template |
|---|---|
| Registration | Email verification link |
| Forgot password | Password reset link |
| Order confirmed | Receipt + ticket QR codes |
| Order cancelled | Cancellation notice |
| Event cancelled | Notice with refund info |
| Org member invite | Invitation link |
| FlexPass transfer complete | Confirmation to seller + buyer |
