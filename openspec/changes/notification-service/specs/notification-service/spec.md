## ADDED Requirements

### Requirement: NotificationService — business layer for personal notifications

The system SHALL introduce a `NotificationService` class (Spring `@Service`) responsible for creating, listing, and managing persistent notifications. This service is distinct from `SSENotificationService` (transport layer). `NotificationService` SHALL call `SSENotificationService.emitNotificationNew(userId)` after every successful `create()` to trigger the `notification:new` SSE signal.

#### Scenario: NotificationService.create persists and signals
- **WHEN** `NotificationService.create(userId, type, title, body, entityType, entityId)` is called after a business transaction commits
- **THEN** a `Notification` row is saved and `notification:new` is emitted to `user:{userId}`

---

### Requirement: SSENotificationService gains emitNotificationNew method

`SSENotificationService` SHALL expose a new method `emitNotificationNew(Long userId)` that emits `notification:new` to `user:{userId}` with no payload data fields. This method follows the same error-handling pattern as existing `emitEvent()` calls (log failure, do not throw).

#### Scenario: emitNotificationNew emits correct event
- **WHEN** `emitNotificationNew(userId)` is called
- **THEN** a POST is sent to the SSE service with `type="notification:new"`, `channel="user:{userId}"`, and empty `data` object

---

### Requirement: AdminService.reviewOrganization triggers notifications

`AdminService.reviewOrganization()` SHALL call `NotificationService.create()` after updating organization status:
- On approval (`ACTIVE`): create `ORG_APPROVED` notification for the org owner
- On rejection: create `ORG_REJECTED` notification for the org owner, including rejection reason in `body`

#### Scenario: Org approved — owner notified
- **WHEN** admin approves an organization via `AdminService.reviewOrganization(orgId, "ACTIVE")`
- **THEN** an `ORG_APPROVED` notification is created for `org.owner.id`

#### Scenario: Org rejected — owner notified with reason
- **WHEN** admin rejects an organization via `AdminService.reviewOrganization(orgId, "REJECTED")`
- **THEN** an `ORG_REJECTED` notification is created for `org.owner.id` with rejection reason in `body`

---

### Requirement: OrganizationService.createOrganization notifies admins

`OrganizationService.createOrganization()` SHALL call `NotificationService.create()` for each admin user with type `ORG_PENDING_REVIEW` after the org is saved.

#### Scenario: All admins notified on new org creation
- **WHEN** organizer creates a new organization
- **THEN** each admin user receives an `ORG_PENDING_REVIEW` notification

---

### Requirement: OrganizationMemberService fixes SSE-006 and persists invitation notification

`OrganizationMemberService.createInvitation()` SHALL:
1. Resolve the invitee's userId from their email via `userRepository.findByEmail(inviteeEmail)`
2. If found: emit `invitation:create` to `user:{inviteeId}` (fixing SSE-006) AND create an `INVITATION_RECEIVED` notification
3. If not found (user doesn't have an account yet): skip SSE and notification silently (no error)

#### Scenario: Invitee has account — receives SSE and notification
- **WHEN** invitation is created for a user who has an account
- **THEN** `invitation:create` SSE emits to `user:{inviteeId}` AND `INVITATION_RECEIVED` notification is persisted

#### Scenario: Invitee has no account — no error
- **WHEN** invitation is created for an email with no matching user account
- **THEN** no SSE is emitted, no notification is created, and no error is thrown

---

### Requirement: Notification type enum

The system SHALL define a `NotificationType` enum with values covering all notification scenarios:
`ORG_APPROVED`, `ORG_REJECTED`, `ORG_PENDING_REVIEW`, `INVITATION_RECEIVED`, `ORDER_CONFIRMED`, `REFUND_COMPLETED`, `REFUND_REQUEST_RECEIVED`, `FLEXPASS_APPROVED`, `FLEXPASS_REJECTED`.

#### Scenario: Type stored as string in DB
- **WHEN** a notification is saved
- **THEN** `type` column stores the enum name as a VARCHAR (e.g., `"ORG_APPROVED"`)
