# Services: SSENotificationService, EmailService & InAppNotificationService

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

---

## InAppNotificationService

Persistent in-app notification store. Business layer — distinct from `SSENotificationService` (transport). After each `create()`, emits `notification:new` to `user:{userId}` via `SSENotificationService.emitNotificationNew()` using `@TransactionalEventListener(AFTER_COMMIT)`.

### Entity: InAppNotification (table: `notifications`)

| Field | Type | Notes |
|---|---|---|
| `id` | BIGSERIAL PK | Auto-generated |
| `userId` | UUID NOT NULL | Foreign key to `users.id` |
| `type` | VARCHAR(64) | `NotificationType` enum stored as string |
| `title` | VARCHAR(255) | Short display title |
| `body` | TEXT | Full message body |
| `entityType` | VARCHAR(64) | e.g. `ORGANIZATION`, `ORDER`, `REFUND_REQUEST`, `INVITATION` |
| `entityId` | VARCHAR(64) | String ID of the referenced entity |
| `isRead` | BOOLEAN DEFAULT FALSE | Read status |
| `createdAt` | TIMESTAMP | Auto-set on creation |

Index: `idx_notifications_user_unread (user_id, is_read)`

### NotificationType enum

| Value | Trigger |
|---|---|
| `ORG_APPROVED` | Admin approves an organization |
| `ORG_REJECTED` | Admin rejects an organization |
| `ORG_PENDING_REVIEW` | New organization created (notifies admins) |
| `INVITATION_RECEIVED` | Invitation sent to an existing user (SSE-006 fix) |
| `ORDER_CONFIRMED` | Payment confirmed — customer notified |
| `REFUND_COMPLETED` | Refund processed successfully — customer notified |
| `REFUND_REQUEST_RECEIVED` | Customer submits refund request — organizer notified |
| `INVITATION_ACCEPTED` | Invitation accepted — org owner notified |
| `INVITATION_REJECTED` | Invitation rejected — org owner notified |
| `REFUND_REJECTED` | Refund request rejected — customer notified |
| `REFUND_FAILED` | Refund processing failed — organizer notified |
| `FLEXPASS_APPROVED` | FlexPass listing approved — seller notified |
| `FLEXPASS_REJECTED` | FlexPass listing rejected — seller notified |
| `FLEXPASS_LISTING_PENDING_REVIEW` | New FlexPass listing created — org owner notified |
| `FLEXPASS_PRICE_LOCKED` | Sale window opened, price locked — seller notified |
| `FLEXPASS_EXPIRED` | Sale window closed without buyer — seller notified |

### REST API

All endpoints require authentication. Users can only access their own notifications (enforced at service layer).

| Method | Path | Description |
|---|---|---|
| GET | `/api/notifications` | List last 30 notifications, newest first |
| GET | `/api/notifications/unread-count` | `{ "count": N }` |
| PUT | `/api/notifications/{id}/read` | Mark single notification as read |
| PUT | `/api/notifications/read-all` | Mark all notifications as read |

### SSE signal

After `create()` commits, emits `notification:new` to `user:{userId}` — pure signal, empty `data` object. Frontend refetches notification list via REST. See SSE-021.

### Services calling InAppNotificationService

| Service | Notification type | Recipient |
|---|---|---|
| `AdminService.reviewOrganization()` | `ORG_APPROVED` or `ORG_REJECTED` | Org owner |
| `OrganizationService.createOrganization()` | `ORG_PENDING_REVIEW` | All admin users |
| `OrganizationService.verifyOrganization()` | `ORG_APPROVED` | Org owner |
| `OrganizationMemberService.inviteMember()` | `INVITATION_RECEIVED` | Invitee (if has account) |
| `OrganizationMemberService.acceptInvitation()` | `INVITATION_ACCEPTED` | Org owner |
| `OrganizationMemberService.rejectInvitation()` | `INVITATION_REJECTED` | Org owner |
| `OrderInternalService.sendOrderConfirmedNotifications()` | `ORDER_CONFIRMED` | Customer |
| `RefundRequestService.createRefundRequest()` | `REFUND_REQUEST_RECEIVED` | Event organizer |
| `RefundRequestService.reviewRefundRequest()` (reject) | `REFUND_REJECTED` | Customer |
| `RefundRequestService.markRefundCompleted()` | `REFUND_COMPLETED` | Customer |
| `RefundRequestService.markRefundFailed()` | `REFUND_FAILED` | Event organizer |
| `FlexPassService.createListing()` | `FLEXPASS_LISTING_PENDING_REVIEW` | Event org owner |
| `FlexPassService.approveListing()` | `FLEXPASS_APPROVED` | Seller |
| `FlexPassService.rejectListing()` | `FLEXPASS_REJECTED` | Seller |
| `FlexPassService.openSaleWindow()` (per listing) | `FLEXPASS_PRICE_LOCKED` | Seller |
| `FlexPassService.closeSaleWindow()` (per listing) | `FLEXPASS_EXPIRED` | Seller |

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
