## ADDED Requirements

### Requirement: SSE-021 — notification:new MUST be emitted only to user:{id} after DB commit

The backend MUST emit `notification:new` exclusively to `user:{userId}` and NEVER to shared channels (`public`, `organizer`, `admin`). The emit MUST occur after the `Notification` record is committed to the database (SSE-008 applies). The payload MUST carry no entity data — it is a pure cache-invalidation signal.

#### Scenario: notification:new goes only to private channel
- **WHEN** NotificationService creates a notification for userId X
- **THEN** `notification:new` is emitted to `user:{X}` only — not to any shared channel

#### Scenario: notification:new carries no payload data
- **WHEN** `notification:new` is emitted
- **THEN** the SSE payload `data` object is empty — no `notificationId`, `title`, `body`, `entityId`, or other fields

#### Scenario: notification:new not emitted before DB commit
- **WHEN** a DB transaction creating a Notification record rolls back
- **THEN** no `notification:new` event reaches the SSE service

---

### Requirement: SSE event taxonomy — Tier C addition for notification:new

`notification:new` is added to the Tier C (Private user events) taxonomy.

| SSEAction | Trigger | Recipient |
|---|---|---|
| `notification:new` | NotificationService creates a Notification record | Target user only (`user:{userId}`) |

#### Scenario: notification:new classified as Tier C
- **WHEN** a new notification is created for user X
- **THEN** `notification:new` is emitted to `user:{X}` — same Tier C private channel pattern as `order:confirm`, `ticket:issue`, etc.

---

### Requirement: SSE invalidation model — NOTIFICATION_NEW invalidates only Notification tag

When `NOTIFICATION_NEW` is received in SSEProvider, the frontend MUST invalidate only `['Notification']`. It MUST NOT invalidate any other RTK Query tag.

#### Scenario: NOTIFICATION_NEW invalidates Notification tag only
- **WHEN** SSEProvider receives `NOTIFICATION_NEW`
- **THEN** `NotificationAPI.util.invalidateTags(['Notification'])` is dispatched
- **AND** no other API tags (`Order`, `Event`, `TicketType`, etc.) are invalidated

## MODIFIED Requirements

### Requirement: SSE-006 — Invitation creation MUST notify the invitee on their personal channel

`notifyInvitationCreated()` MUST emit to TWO channels simultaneously:
1. `organizer` — for badge/list refresh for the inviting org's members
2. `user:{inviteeId}` — so the invitee receives a real-time toast notification

The `inviteeId` (UUID) MUST be resolved from `inviteeEmail` via `userRepository.findByEmail()` at the `OrganizationMemberService` layer before calling `notifyInvitationCreated()`. If no user account is found for the email, the `user:{inviteeId}` emit is silently skipped (no error). The `organizer` channel emit MUST still fire regardless.

Additionally, `OrganizationMemberService` SHALL call `NotificationService.create()` with type `INVITATION_RECEIVED` for the invitee (if account exists), persisting the notification for the bell icon.

#### Scenario: Invitee with account receives SSE on private channel
- **WHEN** invitation created for a user with an existing account
- **THEN** `invitation:create` emits to `organizer` AND to `user:{inviteeId}`

#### Scenario: Invitee without account — organizer channel still fires
- **WHEN** invitation created for an email with no matching user account
- **THEN** `invitation:create` emits to `organizer` only; no `user:{inviteeId}` emit; no error

#### Scenario: inviteeId resolved from email at service layer
- **WHEN** `OrganizationMemberService.createInvitation(inviteeEmail, ...)` is called
- **THEN** `userRepository.findByEmail(inviteeEmail)` is called to resolve the userId before any SSE or notification call
