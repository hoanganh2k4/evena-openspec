## Why

The platform currently relies exclusively on ephemeral SSE toasts for personal notifications — when a user misses a toast (page reload, disconnect, not looking at screen), the notification is permanently lost. Critical cross-user events such as admin rejecting an organization, a member invitation, or a refund completion have no persistent record visible to the recipient. This creates trust and UX gaps that must be closed before the platform scales to real users.

## What Changes

- **New**: `Notification` entity persisted in the database (per-user inbox)
- **New**: `NotificationService` — business layer responsible for creating and managing notifications; emits `notification:new` SSE signal after each DB write
- **New**: `NotificationController` — REST API for listing, counting unread, and marking notifications as read
- **New**: `notification:new` SSE action on `user:{id}` channel — pure signal (no payload data), triggers frontend to refetch notification list
- **New**: `NotificationAPI` RTK Query service on the frontend
- **New**: `NotificationBell` UI component — bell icon with unread badge in header, dropdown notification list, mark-read interactions
- **Modified**: `SSEProvider` — handles `NOTIFICATION_NEW` event → invalidates `['Notification']` RTK tag
- **Modified**: `AdminService.reviewOrganization()` — now calls `NotificationService` for approve/reject outcomes (currently has **no SSE notification at all**)
- **Modified**: `OrganizationService` — calls `NotificationService` alongside existing SSE for org-level personal events
- **Modified**: `OrganizationMemberService` — fixes SSE-006: resolves invitee UUID and emits `user:{inviteeId}`; also persists invitation notification
- **Modified**: `OrderService`, `RefundRequestService` — calls `NotificationService` for personal-scope events to add persistence to existing SSE toasts

## Capabilities

### New Capabilities

- `notification-center`: Persistent in-app notification store with REST API and bell-icon UI. Covers entity model, service logic, REST endpoints, RTK Query integration, and NotificationBell component.
- `notification-sse-signal`: New `notification:new` SSE action on `user:{id}` channel. Pure transport signal — no payload beyond the action type — that tells the connected client to refetch its notification list.

### Modified Capabilities

- `notification-service`: Existing spec at `specs/services/notification-service.md` only covers SSENotificationService and EmailService. Adding the new `NotificationService` layer, its API contract, and the full notification scenario table.
- `sse-flow-spec`: Adding normative rule **SSE-021** — `notification:new` MUST be emitted only to `user:{id}` after a notification record is committed to the DB. MUST NOT carry entity data in payload (pure signal). Adding `notification:new` to the event taxonomy (Tier C).

## Impact

- **Backend**: New DB table `notifications`; new Spring Boot service + controller; modifications to `AdminService`, `OrganizationService`, `OrganizationMemberService`, `OrderService`, `RefundRequestService`
- **SSE service**: `actions.json` and `channels.json` updated with `notification:new` action and `notification` resource on `user:{id}` channel
- **Frontend**: New RTK Query service; new React component; `SSEProvider.tsx` and `sse.ts` updated
- **Database migration**: New Flyway/Liquibase migration for `notifications` table
- **No breaking changes** to existing SSE events — all existing cache-invalidation broadcasts remain unchanged
