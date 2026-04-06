# Change Spec: MinIO Image Upload + SSE Bug Fixes

**ID:** SPEC-005
**Date:** 2026-04-06
**Status:** done

---

## Summary

Three independent bug fixes shipped together:

1. **MinIO image upload flow** — Frontend two-stage upload pattern for event cover and gallery images. Upload via `/api/events/{id}/images/cover` → receive URL → save on form submit. Prevents `@Version` conflicts from mid-session DB writes.

2. **SSE ONGOING/COMPLETED routing bug** — `notifyEventUpdated()` only routed `PUBLISHED` events to `public,organizer`; ONGOING and COMPLETED events were incorrectly routed to `organizer,admin`. Attendees watching live/completed events did not receive real-time updates.

3. **`order:refund` SSE implementation** — When a paid order is refunded, the backend now emits `order:refund` to `user:{userId}` with `orderId`, `eventId`, `eventName`, and `refundAmount`. Frontend handles the event: invalidates Order cache and shows a toast with the refund amount.

---

## Spec changes

### SSE-001 (sse-flow-spec.md §SSE-001)

Updated routing rule for `notifyEventUpdated()`:

| Before | After |
|--------|-------|
| `PUBLISHED` → `public,organizer`; else `organizer,admin` | `PUBLISHED \| ONGOING \| COMPLETED` → `public,organizer`; else `organizer,admin` |

**Rationale:** ONGOING and COMPLETED events are publicly visible (customers can view them). Attendees actively viewing an ONGOING event must receive `event:update` notifications (e.g., updated description, cover image). COMPLETED events should also propagate updates to public since they remain browsable. CANCELLED events are immutable post-transition so their updates (if any) should not reach public.

### Table update (sse-flow-spec.md §Tier A / §Tier B)

- Tier A row: `event:update` trigger updated to "PUBLISHED, ONGOING, or COMPLETED event"
- Tier B row: `event:update` trigger updated to "DRAFT or CANCELLED event"

---

## Backend changes

### `SSENotificationService.java`
- `notifyEventUpdated()`: routing now uses `isPubliclyVisible = PUBLISHED || ONGOING || COMPLETED`
- `notifyOrderRefunded()`: new method — emits `order:refund` to `user:{userId}` channel with `orderId`, `eventId`, `eventName`, optional `refundAmount`

### `EventImageService.java`
- `uploadCover()`: changed from "upload + save to DB" to "upload only, return URL". Saving moved to `EventService.updateEvent()` to avoid `@Version` increment during upload.

### `EventService.java`
- `updateEvent()`: when `coverUrl` in request differs from stored value, calls `storageService.deleteFile(old URL)` then sets new URL. This replaces the former implicit save in `EventImageService`.

### `StorageService.java`
- `deleteFile()`: added guard — skips deletion if URL is null or does not start with `app.storage.public-url`. Prevents S3Exception when trying to delete external/placeholder URLs (e.g., `https://picsum.photos/...`).

### `PaymentService.java`
- `refundPayment()`: after successful refund commit, calls `sseNotificationService.notifyOrderRefunded(orderId, userId, eventId, eventTitle, refundAmount)`.

### `compose.yaml`
- Added `extra_hosts: host.docker.internal:host-gateway` to `app` service so the Spring Boot container can reach the SSE service running on the Docker host.
- Added `APP_SSE_URL: http://host.docker.internal:8000` and `APP_SSE_ENABLED: "true"` environment variables.

### Database
- `activity_log_action_check` constraint updated to include: `ORGANIZATION_ADDED`, `ORGANIZATION_UPDATED`, `ORGANIZATION_DELETED`, `ORGANIZATION_REJECTED`, `ORGANIZATION_APPROVED`, `ORGANIZATION_VERIFIED`, `MEMBER_UPDATED`, `ORDER_SUCCESSFULL`.

---

## Frontend changes

### `EventApi.ts`
- Added mutations: `uploadEventCover`, `uploadGalleryImage`, `deleteGalleryImage`, `uploadEventFile`, `listEventFiles`, `deleteEventFile`
- Response types: `UploadResponse` (raw, not wrapped in `ApiResponse`) matching backend `ResponseEntity<UploadResponse>` directly

### `UserApi.ts` (new)
- `uploadAvatar`, `deleteAvatar` mutations for user profile image

### `store.ts`
- Registered `UserAPI` reducer and middleware

### `event.ts` (types)
- Added `UploadResponse` and `EventFileDTO` interfaces

### `CreateEventForm.tsx`
- Added `eventId?: string` prop
- Inner `CoverImageUpload` component using `useFormikContext` — uploads file, sets `coverUrl` field value, does NOT submit form
- Inner `GalleryUpload` component — uploads file, appends URL to gallery state
- Conditional UI: file upload inputs in edit mode (`isEdit && !!eventId`), URL text inputs in create mode
- Uses `updateEventSchema` (not `eventSchema`) when `isEdit` to avoid past-date `startAt` validation blocking submit

### `UpdateEventForm.tsx`
- Passes `eventId={event.id}` to `<CreateEventForm>`

### `eventValidationSchema.ts`
- Replaced Yup `.url()` with `new URL()` constructor test for `coverUrl` field — accepts `http://localhost:9000/...` URLs (MinIO dev URLs without TLD fail Yup's regex)

### `SSEProvider.tsx`
- Added `ORDER_REFUND` listener → `ORDER_REFUNDED` normalized type
- `ORDER_REFUNDED` case: invalidates `Order` tags, shows toast with formatted `refundAmount` if present
- TicketType event routing fixed: CREATED/UPDATED/DELETED only invalidate `TicketType`; ACTIVATED/DEACTIVATED invalidate both `TicketType` and `Event`

### `sse.ts` (types)
- Added `ORDER_REFUND = 'order:refund'` to `SSEAction`
- Added `ORDER_REFUNDED = 'ORDER_REFUNDED'` to `SSENormalizedType`
- Added `OrderRefundEventData` interface: `{ orderId: number; eventId: string; eventName: string; refundAmount?: number }`
- Added to `SSEEventData` discriminated union

### `next.config.ts`
- Added `localhost:9000` (MinIO dev server) to `images.remotePatterns`
