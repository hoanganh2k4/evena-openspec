# Spec: Organization SSE Fixes & Verified-Org Edit Block

**ID:** SPEC-001
**Author:** team
**Date:** 2026-03-10
**Tier:** light
**Status:** approved

---

## Overview

Three related fixes shipped together:

1. **Category delete FK violation** — deleting a category still referenced by
   any event (not just published) threw a raw `DataIntegrityViolationException`.
   Backend now pre-checks all events and returns a friendly 400.

2. **SSE flow for org verification** — when an admin verifies an organization,
   the org owner received no real-time notification. Backend now emits to both
   the broadcast channel (`organizer,admin`) and the personal channel
   (`user:{ownerId}`). Frontend SSEProvider renders a toast.

3. **Verified-org edit block + list sync** — verified organizations could still
   be edited (no server-side guard). Also, org list did not refresh after
   create/update because RTK Query only invalidated the item tag, not the list
   tag.

## User Stories

- As an organizer, I want a clear error message when I try to delete a category
  that has events, so that I understand what to fix.
- As an org owner, I want to be notified in real time when my organization is
  verified by admin.
- As an org owner, I should not be able to edit a verified organization to
  prevent accidental changes to audited data.
- As an organizer, I want the org list to update immediately after I create or
  edit an organization.

## Out of Scope

- Admin editing verified organizations (allowed by design)
- Batch category reassignment UI
- SSE for admin unverify notification (separate change)

---

## API Changes

| Method | Path | Change | Auth |
|--------|------|--------|------|
| DELETE | /api/categories/{id} | Returns 400 with message when events exist | ADMIN |
| PUT | /api/organizations/{id} | Returns 400 when org is verified (non-admin) | ORGANIZER |

## Database Changes

none

## SSE Events

| Action | Channel(s) | Payload |
|--------|-----------|---------|
| `organization:verify` | `organizer,admin` + `user:{ownerId}` | `{ organizationId, organizationName, ownerId }` |

## Frontend Changes

- `SSEProvider.tsx` — add `notification` state, Snackbar toast, split
  `ORGANIZATION_VERIFIED` case to handle personal channel
- `OrganizationRowCard.tsx` — disable Edit button when `org.verified`
- `OrganizerApi.ts` — fix `updateOrganization.invalidatesTags` to include list tag
- `organizations/page.tsx` — guard in `handleOrganizationEdit` for verified orgs
- `OrganizationFilters.tsx` — remove redundant manual SSE refetch

---

## Error Cases

| Scenario | HTTP Status | Error message |
|----------|-------------|---------------|
| Delete category with events | 400 | "Cannot delete category '...': it is still used by N event(s)." |
| Edit verified org (non-admin) | 400 | "Organization '...' is verified and cannot be edited." |

## i18n Keys Required

```
messages.error.verifiedOrgCannotEdit
  EN: "This organization is verified and cannot be edited. Contact admin for changes."
  VI: "Tổ chức này đã được xác minh và không thể chỉnh sửa. Liên hệ quản trị viên để thay đổi."
```

## Security Notes

- Edit guard checks `organization.verified && !isAdmin()` in Spring service layer.
- Frontend enforces the same rule in both the UI (disabled button) and the
  handler (`handleOrganizationEdit` early return), but the backend is the
  authoritative gate.
