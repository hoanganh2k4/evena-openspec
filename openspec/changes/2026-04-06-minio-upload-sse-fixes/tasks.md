# Tasks: MinIO Image Upload + SSE Bug Fixes

**ID:** TASK-005
**Spec:** SPEC-005
**Author:** team
**Date:** 2026-04-06
**Status:** done

---

## Checklist

### backend

- [x] **T1** — `SSENotificationService.java`: Fix `notifyEventUpdated()` to route `ONGOING` and `COMPLETED` events to `public,organizer` (was: only `PUBLISHED`)
- [x] **T2** — `SSENotificationService.java`: Add `notifyOrderRefunded()` method emitting `order:refund` to `user:{userId}` with `orderId`, `eventId`, `eventName`, optional `refundAmount`
- [x] **T3** — `EventImageService.java`: `uploadCover()` — remove DB save; return URL only (two-stage pattern)
- [x] **T4** — `EventService.java`: In `updateEvent()`, delete old cover via `storageService.deleteFile()` when `coverUrl` changes; set new URL from request
- [x] **T5** — `StorageService.java`: Add null / non-MinIO-URL guard in `deleteFile()` to prevent S3Exception on external URLs
- [x] **T6** — `PaymentService.java`: After successful refund, call `sseNotificationService.notifyOrderRefunded()`
- [x] **T7** — `compose.yaml`: Add `extra_hosts: host.docker.internal:host-gateway` + `APP_SSE_URL=http://host.docker.internal:8000` so Docker container can reach host SSE service
- [x] **T8** — DB: ALTER `activity_log_action_check` constraint to include `ORGANIZATION_*` and `MEMBER_UPDATED` actions missing from original migration

### frontend

- [x] **T9** — `EventApi.ts`: Add `uploadEventCover`, `uploadGalleryImage`, `deleteGalleryImage`, `uploadEventFile`, `listEventFiles`, `deleteEventFile` mutations with correct `UploadResponse` type (not wrapped)
- [x] **T10** — `UserApi.ts`: Create new RTK Query API slice with `uploadAvatar`, `deleteAvatar`
- [x] **T11** — `store.ts`: Register `UserAPI` reducer and middleware
- [x] **T12** — `event.ts`: Add `UploadResponse` and `EventFileDTO` interfaces
- [x] **T13** — `CreateEventForm.tsx`: Add `eventId` prop; add `CoverImageUpload` and `GalleryUpload` inner components using `useFormikContext`; use `updateEventSchema` when `isEdit`
- [x] **T14** — `UpdateEventForm.tsx`: Pass `eventId={event.id}` to `<CreateEventForm>`
- [x] **T15** — `eventValidationSchema.ts`: Replace `.url()` with `new URL()` constructor test for `coverUrl` to accept MinIO dev URLs
- [x] **T16** — `SSEProvider.tsx`: Add `ORDER_REFUND` listener; fix TicketType SSE cache invalidation split (ACTIVATED/DEACTIVATED vs CREATED/UPDATED/DELETED)
- [x] **T17** — `sse.ts`: Add `ORDER_REFUND` action, `ORDER_REFUNDED` normalized type, `OrderRefundEventData` interface
- [x] **T18** — `next.config.ts`: Add `localhost:9000` to `images.remotePatterns`
