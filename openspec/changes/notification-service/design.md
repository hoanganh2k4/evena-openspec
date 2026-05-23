## Context

The platform uses an SSE-based real-time notification layer (SSENotificationService → SSE service → SSEProvider) and a separate EmailService. All SSE events are ephemeral: toasts shown once and lost on reload. There is no persistent per-user notification store. Several critical personal events (org rejection, invitations, FlexPass outcomes) either have no notification at all or only broadcast to shared channels.

The existing `SSENotificationService` already functions correctly as a transport layer — it POSTs to the SSE service and routes events to the correct channels. This design adds a `NotificationService` layer on top without touching the transport layer's responsibilities.

**Current critical gaps:**
- `AdminService.reviewOrganization()` emits no SSE at all (approve or reject)
- `OrganizationMemberService` does not emit to `user:{inviteeId}` (SSE-006 gap)
- No notification history — users who miss a toast lose the information permanently

## Goals / Non-Goals

**Goals:**
- Persistent per-user notification store (DB-backed)
- REST API for listing, unread count, and marking read
- `notification:new` SSE signal on `user:{id}` to update bell badge in real time
- Fix `AdminService` gap (approve/reject org notifications)
- Fix SSE-006 (invitation to invitee)
- NotificationBell UI component with dropdown in the app header
- All existing SSE cache-invalidation signals remain unchanged

**Non-Goals:**
- Push notifications (mobile/browser)
- Email notification changes
- WebSocket migration
- Notification preferences / mute settings
- Pagination beyond simple offset (v1: latest 30 notifications)

## Decisions

### D-1: Two-layer architecture — SSENotificationService (transport) vs NotificationService (business)

**Decision:** Business services continue calling `SSENotificationService` for Type 1 cache-invalidation broadcasts. For Type 2 personal notifications, business services additionally call `NotificationService.create(...)`, which saves to DB and then calls `SSENotificationService.emitNotificationNew(userId)` to emit `notification:new`.

**Why not replace SSE events with `notification:new` entirely?** The existing SSE events carry semantic cache-invalidation information (`order:confirm` → invalidate `['Order', 'Ticket']`). Replacing them with a generic `notification:new` would break RTK Query targeted cache invalidation (SSE-009, SSE-010). The two signals serve different purposes.

**Why not have SSENotificationService call NotificationService?** SSENotificationService is infrastructure — it should not have knowledge of business concepts (which events are "personal" vs "cache-invalidation"). Keeping NotificationService as the business layer calling the transport is cleaner.

**Alternative considered:** Single `NotificationService` that absorbs SSENotificationService. Rejected: breaks separation between broadcast cache signals and personal persistent notifications; would require rewriting all existing service call sites.

---

### D-2: notification:new carries no payload — pure signal

**Decision:** The `notification:new` SSE event MUST carry only `{ type: "notification:new" }` — no `notificationId`, no `title`, no `body`. Frontend refetches via REST API on receipt.

**Why:** Consistent with SSE P-1 (cache-invalidation signal, not data transport). Avoids duplicating notification content in two places. Prevents PII leakage in SSE logs.

---

### D-3: Notification entity schema

```sql
CREATE TABLE notifications (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT NOT NULL REFERENCES users(id),
  type        VARCHAR(64) NOT NULL,   -- e.g. ORG_APPROVED, ORDER_CONFIRMED
  title       VARCHAR(255) NOT NULL,
  body        TEXT,
  entity_type VARCHAR(64),            -- e.g. ORGANIZATION, ORDER
  entity_id   VARCHAR(64),            -- UUID or Long as string
  is_read     BOOLEAN NOT NULL DEFAULT FALSE,
  created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_notifications_user_unread ON notifications(user_id, is_read);
```

**Why BIGSERIAL not UUID for PK:** Internal entity, not exposed externally, sequential PK is fine for ordering.  
**Why VARCHAR for entity_id:** Entities use both Long and UUID — a string column avoids two separate FK columns.

---

### D-4: API design — no full pagination in v1

```
GET  /api/notifications          → last 30, newest first
GET  /api/notifications/unread-count  → { count: N }
PUT  /api/notifications/{id}/read     → mark one read
PUT  /api/notifications/read-all      → mark all read for current user
```

**Why no pagination in v1:** Bell dropdown typically shows ≤ 20 items. Adding cursor pagination adds complexity without immediate value. Limit 30 is sufficient. Can add in v2.

**Auth:** All endpoints use existing Spring Security — `@AuthenticationPrincipal UserDetails` resolves the current user. Users can only access their own notifications (enforced at service layer: `WHERE user_id = currentUserId`).

---

### D-5: NotificationService is the single write point

All notification creation goes through `NotificationService.create(userId, type, title, body, entityType, entityId)`. This method:
1. Saves `Notification` to DB
2. Calls `sseNotificationService.emitNotificationNew(userId)` — outside any `@Transactional` boundary to comply with SSE-008

**Why not emit inside `@Transactional`?** SSE-008 requires emit after commit. `NotificationService.create()` is itself `@Transactional`; the SSE emit is placed after the save returns by using `@TransactionalEventListener(AFTER_COMMIT)` on a domain event, or by the caller ensuring create() is called after the enclosing business transaction.

---

### D-6: Frontend — RTK tag `'Notification'`

`NotificationAPI` uses RTK Query tag `'Notification'`. SSEProvider on `NOTIFICATION_NEW` → `invalidateTags(['Notification'])`. This follows SSE-014 (single authoritative cache invalidation location). The `NotificationBell` subscribes via `useGetNotificationsQuery` and `useGetUnreadCountQuery` — auto-refetches when tag is invalidated.

---

### D-7: Admin notifications (ORG_PENDING_REVIEW) — target admin users

When an org is created (PENDING), `OrganizationService.createOrganization()` should notify admin users. Since the system may have multiple admins, we query `userRepository.findByRole(ADMIN)` and call `NotificationService.create(adminUserId, ...)` for each.

**Risk:** If there are many admins, this creates multiple notifications. Acceptable for v1 (admin count is small in this system).

## Risks / Trade-offs

| Risk | Mitigation |
|---|---|
| SSE emit fires before DB transaction commits | Use `@TransactionalEventListener(AFTER_COMMIT)` or place `NotificationService.create()` call in controller layer after service method returns |
| Multiple admins → N notification rows per org creation | Acceptable for v1; can add a role-targeted notification type later |
| Notification table grows unbounded | Add a scheduled cleanup job later (out of scope for v1); index on `user_id + created_at` keeps queries fast |
| `entity_id` is VARCHAR — no FK constraint | Accepted trade-off for flexibility; referential integrity enforced at service layer |
| SSE-006 fix (inviteeId resolution) requires DB lookup | `userRepository.findByEmail(inviteeEmail)` at service layer; invitee may not exist if invited by email before account creation — handle gracefully (no SSE, no notification if user not found) |

## Migration Plan

1. Add Flyway migration: `V{next}__create_notifications_table.sql`
2. Deploy backend (new table, new service, new controller) — backward compatible
3. Deploy SSE service config update (`actions.json`, `channels.json`) — additive, no existing actions removed
4. Deploy frontend (new component, SSEProvider updated) — additive

**Rollback:** Drop `notifications` table; revert SSE config; revert frontend. No data loss risk (notifications are supplementary, not authoritative).

## Open Questions

- **Q1:** Should `ORG_PENDING_REVIEW` notifications go to ALL admin users or only a designated reviewer? → Decision: all admins for v1.
- **Q2:** Should we notify the organizer when their org is first created (confirmation)? → Decision: no, the creation itself is their action; only admin-action outcomes notify organizer.
- **Q3:** Cleanup / TTL for old notifications? → Out of scope for v1; add scheduled cleanup in a follow-up change.
