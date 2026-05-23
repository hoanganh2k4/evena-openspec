## ADDED Requirements

### Requirement: notification:new SSE action on user:{id} channel

The system SHALL emit a `notification:new` SSE event to `user:{userId}` whenever a new `Notification` record is created for that user. The payload MUST be empty (no `data` fields beyond the action type). This is a pure cache-invalidation signal — the frontend fetches notification content via REST API.

#### Scenario: Signal emitted after notification DB write
- **WHEN** `NotificationService.create()` saves a notification to DB and the transaction commits
- **THEN** `notification:new` event is emitted to `user:{userId}` channel on the SSE service

#### Scenario: Signal carries no notification data
- **WHEN** `notification:new` event is emitted
- **THEN** the SSE payload contains only the action type — no `title`, `body`, `notificationId`, or any entity fields

#### Scenario: Signal only on user private channel
- **WHEN** `notification:new` is emitted
- **THEN** the event is sent exclusively to `user:{userId}` — NEVER to `public`, `organizer`, or `admin` shared channels

#### Scenario: Frontend invalidates Notification RTK tag
- **WHEN** SSEProvider receives `notification:new`
- **THEN** `NotificationAPI.util.invalidateTags(['Notification'])` is dispatched, triggering refetch of unread count and notification list

---

### Requirement: notification resource declared in SSE service config

The SSE service `actions.json` SHALL contain a `notification:new` entry with `resource: "notification"`. The `channels.json` `user:{id}` pattern SHALL include `notification` in its `allowed_resources` list.

#### Scenario: notification:new action is valid for user:{id} channel
- **WHEN** backend emits `notification:new` to `user:{someUUID}` channel
- **THEN** SSE service `channel_manager.py` validates the action as allowed and routes it to connected clients

#### Scenario: notification:new action rejected on shared channels
- **WHEN** backend attempts to emit `notification:new` to `public`, `organizer`, or `admin`
- **THEN** SSE service rejects the emit (action not in allowed_resources for those channels)
