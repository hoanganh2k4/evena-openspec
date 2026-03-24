# Spec: Venue Delete Lock — Form Locking on Concurrent Delete

**ID:** SPEC-003
**Author:** team
**Date:** 2026-03-17
**Tier:** light
**Status:** approved

---

## Overview

When a venue is **deleted** while another browser has the venue's edit form open,
the update form MUST lock (submit button disabled, warning shown) — exactly as it
does today for concurrent **updates** via `useOptimisticLocking`.

This is required by `sse-checklist.md` Case 3.1:
> "Expected result: Success delete venue, and lock the update form"

---

## Root Cause

`ENTITY_EVENT_TYPES.VENUE` in `useOptimisticLocking.ts` only includes
`SSENormalizedType.VENUE_UPDATED`. A `venue:delete` SSE event therefore never
triggers `hasConflict = true`, leaving the submit button enabled.

---

## Change

### `frontend/src/hooks/useOptimisticLocking.ts`

Add `SSENormalizedType.VENUE_DELETED` to `ENTITY_EVENT_TYPES.VENUE`:

```typescript
VENUE: [
  SSENormalizedType.VENUE_UPDATED,
  SSENormalizedType.VENUE_DELETED,   // lock form when venue is concurrently deleted
],
```

No other files need to change. The `VenueEventData` shape already carries
`venueId: number`, so the ID-matching logic in the hook works without modification.

The existing conflict message ("This venue has been modified by another user or
session. Please close this form and reopen it to get the latest data before making
changes.") is acceptable for a delete event — the user closes the form and sees
the venue is gone from the list.

---

## Spec References

- `sse-checklist.md` Case 3.1 — delete venue while update form is open → lock form
- `patterns.json` DELETE_LOCK — "The modal must lock"
