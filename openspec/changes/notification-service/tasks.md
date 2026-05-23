## 1. Backend — DB Migration & Entity

- [ ] 1.1 Tạo Flyway migration SQL: `notifications` table (id BIGSERIAL, user_id, type VARCHAR(64), title VARCHAR(255), body TEXT, entity_type VARCHAR(64), entity_id VARCHAR(64), is_read BOOLEAN DEFAULT FALSE, created_at TIMESTAMP)
- [ ] 1.2 Tạo enum `NotificationType` tại `enums/NotificationType.java` với values: ORG_APPROVED, ORG_REJECTED, ORG_PENDING_REVIEW, INVITATION_RECEIVED, ORDER_CONFIRMED, REFUND_COMPLETED, REFUND_REQUEST_RECEIVED, FLEXPASS_APPROVED, FLEXPASS_REJECTED
- [ ] 1.3 Tạo JPA entity `InAppNotification` tại `module/notification/entity/InAppNotification.java` (ánh xạ bảng `notifications`)
- [ ] 1.4 Tạo `InAppNotificationRepository` extends `JpaRepository<InAppNotification, Long>` với query `findByUserIdOrderByCreatedAtDesc(Long userId, Pageable pageable)` và `countByUserIdAndIsReadFalse(Long userId)`

## 2. Backend — InAppNotificationService

- [ ] 2.1 Tạo interface `InAppNotificationService` tại `module/notification/service/InAppNotificationService.java` với methods: `create(Long userId, NotificationType type, String title, String body, String entityType, String entityId)`, `getNotifications(Long userId)`, `getUnreadCount(Long userId)`, `markRead(Long notificationId, Long userId)`, `markAllRead(Long userId)`
- [ ] 2.2 Tạo `InAppNotificationServiceImpl` implement interface trên; `create()` annotate `@Transactional`; sau khi save gọi `sseNotificationService.emitNotificationNew(userId)` via `@TransactionalEventListener(AFTER_COMMIT)` hoặc sau khi save bằng cách publish Spring ApplicationEvent
- [ ] 2.3 Thêm method `emitNotificationNew(Long userId)` vào `SSENotificationService.java` — emit `notification:new` to `user:{userId}` với empty data map
- [ ] 2.4 `markRead` MUST kiểm tra `notification.userId == requestingUserId`, throw `AccessDeniedException` nếu không khớp

## 3. Backend — DTOs & Controller

- [ ] 3.1 Tạo DTO `NotificationResponse.java` tại `module/notification/dto/` với fields: id, type, title, body, entityType, entityId, isRead, createdAt
- [ ] 3.2 Tạo `NotificationController` tại `module/notification/controller/NotificationController.java` với các endpoints:
  - `GET /api/notifications` → list (limit 30, newest first)
  - `GET /api/notifications/unread-count` → `{ "count": N }`
  - `PUT /api/notifications/{id}/read` → mark one read
  - `PUT /api/notifications/read-all` → mark all read
- [ ] 3.3 Tất cả endpoints dùng `@AuthenticationPrincipal` để lấy userId, không cần role annotation (user chỉ thấy notification của mình)

## 4. Backend — Wire vào AdminService (ORG_APPROVED / ORG_REJECTED)

- [ ] 4.1 Đọc `AdminService.java` để hiểu flow `reviewOrganization(orgId, status)`
- [ ] 4.2 Inject `InAppNotificationService` vào `AdminService`
- [ ] 4.3 Sau khi update org status thành công:
  - Nếu status = "ACTIVE": gọi `inAppNotificationService.create(org.owner.id, ORG_APPROVED, "Tổ chức đã được duyệt", "Tổ chức {name} đã được admin phê duyệt.", "ORGANIZATION", org.id)`
  - Nếu status khác (reject): gọi `inAppNotificationService.create(org.owner.id, ORG_REJECTED, "Tổ chức bị từ chối", reason, "ORGANIZATION", org.id)`

## 5. Backend — Wire vào OrganizationService (ORG_PENDING_REVIEW)

- [ ] 5.1 Inject `InAppNotificationService` và `UserRepository` vào `OrganizationService`
- [ ] 5.2 Trong `createOrganization()`, sau khi save org: query `userRepository.findAllByRole(ADMIN)` rồi gọi `inAppNotificationService.create(adminUser.id, ORG_PENDING_REVIEW, ...)` cho mỗi admin

## 6. Backend — Wire vào OrganizationMemberService (SSE-006 fix + INVITATION_RECEIVED)

- [ ] 6.1 Đọc `OrganizationMemberService.java` để hiểu flow `createInvitation()`
- [ ] 6.2 Inject `UserRepository` vào `OrganizationMemberService`
- [ ] 6.3 Trong `createInvitation()`: gọi `userRepository.findByEmail(inviteeEmail)` để resolve `inviteeId`
- [ ] 6.4 Nếu user tồn tại: gọi `sseNotificationService.emitEvent("invitation:create", "user:{inviteeId}", ...)` (fix SSE-006) VÀ `inAppNotificationService.create(inviteeId, INVITATION_RECEIVED, ...)`
- [ ] 6.5 Nếu user không tồn tại: skip SSE và notification, không throw error

## 7. Backend — Wire vào OrderService (ORDER_CONFIRMED)

- [ ] 7.1 Inject `InAppNotificationService` vào `OrderService` (hoặc `OrderInternalService`)
- [ ] 7.2 Trong method xử lý payment confirmed: sau khi gọi `sseNotificationService.notifyOrderConfirmed()`, gọi thêm `inAppNotificationService.create(customerId, ORDER_CONFIRMED, "Đơn hàng đã xác nhận", "Vé cho sự kiện {eventName} đã được xác nhận.", "ORDER", orderId)`

## 8. Backend — Wire vào RefundRequestService (REFUND_COMPLETED + REFUND_REQUEST_RECEIVED)

- [ ] 8.1 Inject `InAppNotificationService` vào `RefundRequestService`
- [ ] 8.2 Khi customer tạo refund request: `inAppNotificationService.create(organizerId, REFUND_REQUEST_RECEIVED, "Yêu cầu hoàn tiền mới", "Khách hàng {name} yêu cầu hoàn tiền cho đơn hàng {orderId}.", "REFUND_REQUEST", refundRequestId)`
- [ ] 8.3 Khi refund hoàn thành (REFUNDED): `inAppNotificationService.create(customerId, REFUND_COMPLETED, "Hoàn tiền thành công", "Yêu cầu hoàn tiền cho sự kiện {eventName} đã được xử lý.", "REFUND_REQUEST", refundRequestId)`

## 9. SSE Service — Config Update

- [ ] 9.1 Thêm vào `sse-service/config/actions.json`:
  ```json
  "notification:new": {
    "resource": "notification",
    "description": "New in-app notification available for user"
  }
  ```
- [ ] 9.2 Thêm `"notification"` vào `allowed_resources` của pattern channel `user:{id}` trong `sse-service/config/channels.json`

## 10. Frontend — SSE Types & SSEProvider

- [ ] 10.1 Thêm vào `SSEAction` enum trong `stores/types/sse.ts`: `NOTIFICATION_NEW = 'notification:new'`
- [ ] 10.2 Thêm vào `SSENormalizedType` enum: `NOTIFICATION_NEW = 'NOTIFICATION_NEW'`
- [ ] 10.3 Trong `SSEProvider.tsx`, thêm listener: `on(SSEAction.NOTIFICATION_NEW, SSENormalizedType.NOTIFICATION_NEW)`
- [ ] 10.4 Trong switch-case cache invalidation của SSEProvider: case `NOTIFICATION_NEW` → `dispatch(NotificationAPI.util.invalidateTags(['Notification']))` — KHÔNG invalidate bất kỳ tag nào khác

## 11. Frontend — NotificationAPI RTK Query

- [ ] 11.1 Tạo `stores/services/NotificationApi.ts` với baseApi và tag `'Notification'`
- [ ] 11.2 Endpoint `getNotifications`: `GET /api/notifications` → list, provides tag `[{ type: 'Notification', id: 'LIST' }]`
- [ ] 11.3 Endpoint `getUnreadCount`: `GET /api/notifications/unread-count` → `{ count: number }`, provides tag `[{ type: 'Notification', id: 'COUNT' }]`
- [ ] 11.4 Mutation `markRead`: `PUT /api/notifications/{id}/read`, invalidates `['Notification']`
- [ ] 11.5 Mutation `markAllRead`: `PUT /api/notifications/read-all`, invalidates `['Notification']`
- [ ] 11.6 Thêm `NotificationAPI` vào `stores/services/index.ts` và vào `store` reducer

## 12. Frontend — NotificationBell Component & Header Integration

- [ ] 12.1 Tạo `components/NotificationBell/NotificationBell.tsx` — bell icon với Popover/Dropdown hiển thị danh sách notification
- [ ] 12.2 Dùng `useGetUnreadCountQuery()` để lấy count, truyền vào `Badge.badgeContent` thay hardcoded `3` trong `DashboardHeader`
- [ ] 12.3 Khi click bell: open Popover, gọi `useGetNotificationsQuery()` để render danh sách (title, body, relative time)
- [ ] 12.4 Mỗi notification item: click gọi `markRead(id)`, item hiển thị khác màu nếu `isRead=false`
- [ ] 12.5 Thêm nút "Đánh dấu tất cả đã đọc" ở đầu dropdown, gọi `markAllRead()`
- [ ] 12.6 Cập nhật `DashboardHeader.tsx`: thay `<IconButton onClick={onNotificationClick}>` bằng `<NotificationBell />`  — component tự quản lý state, không cần prop `onNotificationClick` nữa
- [ ] 12.7 Cập nhật `DashboardHeader/types.ts`: đánh dấu `onNotificationClick` là optional (không xóa để backward compat)

## 13. Spec Update

- [ ] 13.1 Cập nhật `evena-openspec/openspec/specs/services/notification-service.md` — thêm section `InAppNotificationService` với entity schema, API endpoints, và notification scenarios table
- [ ] 13.2 Thêm SSE-021 vào `evena-openspec/openspec/specs/sse-flow-spec.md` — normative rule cho `notification:new`
- [ ] 13.3 Cập nhật SSE-006 compliance status trong `sse-flow-spec.md` section 16 từ `NOT IMPLEMENTED` → `COMPLIANT`
