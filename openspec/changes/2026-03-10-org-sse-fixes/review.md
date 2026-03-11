# Review: Organization SSE Fixes & Verified-Org Edit Block

**ID:** REV-001
**Tasks:** TASK-001
**Reviewer:** team
**Date:** 2026-03-10
**Status:** pass

---

## Review Checklist

| # | Check | Result | Notes |
|---|-------|--------|-------|
| RC-001 | All spec requirements implemented | ✅ | All 13 tasks done |
| RC-002 | RTK Query mutations invalidate both item + list tags | ✅ | Fixed in OrganizerApi.ts |
| RC-003 | All user-visible strings use t() + i18n keys in EN+VI | ✅ | verifiedOrgCannotEdit added to both locales |
| RC-004 | No hardcoded hex colors | ✅ | No new hex literals introduced |
| RC-005 | Backend throws domain exceptions (not JPA violations) | ✅ | BadRequestException in both CategoryService + OrganizationService |
| RC-006 | SSE personal channel for owner-targeted events | ✅ | notifyOrganizationVerified emits to user:{ownerId} |
| RC-007 | UUID vs Long types verified | ✅ | SSENotificationService.notifyOrganizationVerified(UUID ownerId) |
| RC-008 | Verified org edit block in backend + frontend | ✅ | Service guard + disabled button + handler guard |
| RC-009 | No console.log in components | ✅ | Removed from OrganizationFilters |
| RC-010 | No new files created unless necessary | ✅ | Only existing files modified |

---

## Summary

All three issues were root-caused and fixed correctly. The category delete guard
now catches all event statuses (not just PUBLISHED), preventing the FK violation.
The SSE org-verify personal notification uses a clean two-channel emit pattern
that will serve as the template for future owner-targeted events. The verified-org
edit block is enforced at both the API layer (BadRequestException) and the UI
layer (disabled button + handler guard), with the backend remaining authoritative.

## Blocking Issues

none

## Suggestions (non-blocking)

- Consider extracting BRAND color constants as a shared MUI theme extension
  instead of repeated hex literals across component `sx` props.
- The `isAdmin()` helper in OrganizationService could be extracted to a
  `SecurityUtils` class to avoid duplication across services.

---

> Status: **PASS** — all changes merged
