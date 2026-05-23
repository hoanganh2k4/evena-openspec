## ADDED Requirements

### Requirement: Notification entity persisted per user

The system SHALL store a `Notification` record in the database for every personal notification event. Each record SHALL contain: `id`, `userId`, `type`, `title`, `body`, `entityType`, `entityId`, `isRead`, `createdAt`.

#### Scenario: Notification saved on org approval
- **WHEN** admin approves an organization
- **THEN** a `Notification` record with `type=ORG_APPROVED` is saved for the org owner's userId

#### Scenario: Notification saved on org rejection
- **WHEN** admin rejects an organization
- **THEN** a `Notification` record with `type=ORG_REJECTED` is saved for the org owner's userId

#### Scenario: Notification saved when new org is pending
- **WHEN** an organizer creates a new organization (status=PENDING)
- **THEN** a `Notification` record with `type=ORG_PENDING_REVIEW` is saved for every admin user

#### Scenario: Notification saved on invitation
- **WHEN** an organizer sends an invitation to a user who has an account
- **THEN** a `Notification` record with `type=INVITATION_RECEIVED` is saved for the invitee's userId

#### Scenario: Notification saved on order confirmation
- **WHEN** a payment is confirmed and an order transitions to CONFIRMED
- **THEN** a `Notification` record with `type=ORDER_CONFIRMED` is saved for the customer's userId

#### Scenario: Notification saved on refund completion
- **WHEN** a refund request is processed successfully (REFUNDED)
- **THEN** a `Notification` record with `type=REFUND_COMPLETED` is saved for the customer's userId

#### Scenario: Notification saved on refund request received
- **WHEN** a customer submits a refund request
- **THEN** a `Notification` record with `type=REFUND_REQUEST_RECEIVED` is saved for the event organizer's userId

#### Scenario: Notification saved on FlexPass listing approved
- **WHEN** an organizer approves a FlexPass listing
- **THEN** a `Notification` record with `type=FLEXPASS_APPROVED` is saved for the seller's userId

#### Scenario: Notification saved on FlexPass listing rejected
- **WHEN** an organizer rejects a FlexPass listing
- **THEN** a `Notification` record with `type=FLEXPASS_REJECTED` is saved for the seller's userId

---

### Requirement: Notification REST API — list user notifications

The system SHALL expose `GET /api/notifications` returning the authenticated user's notifications, newest first, limited to 30 records. Response SHALL include: `id`, `type`, `title`, `body`, `entityType`, `entityId`, `isRead`, `createdAt`.

#### Scenario: User retrieves their notifications
- **WHEN** authenticated user calls `GET /api/notifications`
- **THEN** system returns at most 30 notifications belonging to that user, sorted by `createdAt` DESC

#### Scenario: User cannot access other users' notifications
- **WHEN** authenticated user calls `GET /api/notifications`
- **THEN** system returns ONLY notifications where `userId` matches the authenticated user's id

---

### Requirement: Notification REST API — unread count

The system SHALL expose `GET /api/notifications/unread-count` returning `{ "count": N }` where N is the number of unread notifications for the authenticated user.

#### Scenario: Unread count reflects unread notifications
- **WHEN** user has 3 unread notifications
- **THEN** `GET /api/notifications/unread-count` returns `{ "count": 3 }`

#### Scenario: Unread count is zero when all are read
- **WHEN** user has no unread notifications
- **THEN** `GET /api/notifications/unread-count` returns `{ "count": 0 }`

---

### Requirement: Notification REST API — mark single notification read

The system SHALL expose `PUT /api/notifications/{id}/read` that sets `isRead=true` for the specified notification. The system SHALL reject requests where the notification does not belong to the authenticated user.

#### Scenario: Mark own notification as read
- **WHEN** user calls `PUT /api/notifications/{id}/read` for their own notification
- **THEN** `isRead` is set to `true` and 200 OK is returned

#### Scenario: Cannot mark another user's notification as read
- **WHEN** user calls `PUT /api/notifications/{id}/read` for a notification owned by a different user
- **THEN** system returns 403 Forbidden

---

### Requirement: Notification REST API — mark all read

The system SHALL expose `PUT /api/notifications/read-all` that sets `isRead=true` for all unread notifications of the authenticated user.

#### Scenario: All notifications marked read
- **WHEN** user calls `PUT /api/notifications/read-all`
- **THEN** all notifications for that user have `isRead=true`

---

### Requirement: NotificationBell UI component

The frontend SHALL display a bell icon in the application header showing the unread notification count as a badge. Clicking the bell SHALL open a dropdown listing recent notifications. Clicking a notification SHALL mark it as read. The UI SHALL include a "Mark all as read" action.

#### Scenario: Badge shows unread count
- **WHEN** user has N unread notifications (N > 0)
- **THEN** bell icon displays badge with number N

#### Scenario: Badge hidden when no unread
- **WHEN** user has 0 unread notifications
- **THEN** bell icon displays no badge

#### Scenario: Badge updates in real-time via SSE
- **WHEN** a new `notification:new` SSE event is received
- **THEN** the unread count query is invalidated and badge updates without page reload

#### Scenario: Dropdown shows notification list
- **WHEN** user clicks the bell icon
- **THEN** a dropdown displays the latest notifications with title, body, and relative time

#### Scenario: Clicking notification marks it read
- **WHEN** user clicks an individual notification item
- **THEN** `PUT /api/notifications/{id}/read` is called and the item displays as read

#### Scenario: Mark all read action
- **WHEN** user clicks "Mark all as read" in the dropdown
- **THEN** `PUT /api/notifications/read-all` is called and all items display as read

---

### Requirement: NotificationService is the sole write point for persistent notifications

All notification creation SHALL go through `NotificationService.create(userId, type, title, body, entityType, entityId)`. No other service SHALL write directly to the `notifications` table.

#### Scenario: NotificationService saves then signals
- **WHEN** `NotificationService.create()` is called
- **THEN** a `Notification` row is saved to DB AND `notification:new` SSE is emitted to `user:{userId}` AFTER the DB transaction commits

#### Scenario: SSE emit does not fire if DB save fails
- **WHEN** the DB transaction containing `NotificationService.create()` rolls back
- **THEN** no `notification:new` SSE event is emitted
