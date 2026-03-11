# Spec: SSE Integrity Compliance — Fix Non-Compliant Flows

**ID:** SPEC-002
**Author:** team
**Date:** 2026-03-10
**Tier:** light
**Status:** approved

---

## Overview

Five non-compliant SSE behaviors were identified by audit against Core Rules.
This spec covers the fixes required to bring all SSE flows into compliance with
`specs/sse-flow-spec.md` (SSE-001 through SSE-020) and `rules.md`.

## User Stories

- As a customer, I should NOT receive SSE notifications about DRAFT events
  (which I'm not allowed to see), so that my event cache doesn't get polluted
  with draft data.
- As an admin, I should be confident that confirming an order does not
  accidentally cause all customer event caches to be wiped out.
- As a customer, my paid order detail must always show what I paid, not
  a live refetch triggered by unrelated TicketType events.

## Out of Scope

- Adding `order:refund` SSE (separate change, requires refund flow implementation)
- Adding personal invitation notification to invitee (separate change)
- SSE service Redis migration (separate change)
- Per-organization channels (future architecture decision)

---

## API Changes

none — SSE channel routing changes are internal; no REST API contract change

## Database Changes

none

## SSE Events Changed

| Rule | Current behavior | New behavior |
|------|-----------------|--------------|
| SSE-001 | `event:create` → `public,organizer` always | → `organizer,admin` if DRAFT; `public,organizer` if PUBLISHED |
| SSE-002 | `ticket_type:create/update` → `public,organizer` always | → `organizer,admin` if parent DRAFT |
| SSE-003 | `notifyEventCreated` passes full `EventResponse` DTO | Passes minimal `{eventId, eventName, organizationId}` |
| SSE-004 | `ORDER_CONFIRMED` invalidates `Event`+`TicketType` | Removes those invalidations |
| SSE-009 | `getPublicEvents` has no status filter | Adds `status: 'PUBLISHED'` |

## Frontend Changes

- `src/providers/SSEProvider.tsx` — remove Event/TicketType invalidation from ORDER_CONFIRMED/CANCELLED/EXPIRED
- `src/stores/services/EventApi.ts` — add status filter to `getPublicEvents`
- `app/dashboard/customer/cart/[id]/page.tsx` — remove console.log
- `app/dashboard/customer/events/[id]/page.tsx` — remove console.log

## Backend Changes

- `SSENotificationService.java` — fix `notifyEventCreated` signature (accept eventId, eventName, orgId instead of Object payload)
- `EventService.java` — pass event status to notification methods; choose channel accordingly
- `TicketTypeService.java` — check parent event status before choosing channel

---

## Error Cases

none — these are internal channel routing changes with no user-facing error impact

## Security Notes

- Fixing SSE-001 ensures DRAFT event IDs are not broadcast to public channel
- Fixing SSE-009 ensures the REST endpoint also enforces PUBLISHED filter server-side
  (backend must be verified separately)
