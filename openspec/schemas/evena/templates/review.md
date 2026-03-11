# Review: {{title}}

**ID:** REV-XXX
**Tasks:** TASK-XXX
**Reviewer:** {{reviewer}}
**Date:** {{YYYY-MM-DD}}
**Status:** <!-- pass | fail | needs-changes -->

---

## Review Checklist

| # | Check | Result | Notes |
|---|-------|--------|-------|
| RC-001 | All spec requirements implemented | ✅ / ❌ | |
| RC-002 | RTK Query mutations invalidate both item + list tags | ✅ / ❌ | |
| RC-003 | All user-visible strings use t() + i18n keys in EN+VI | ✅ / ❌ | |
| RC-004 | No hardcoded hex colors — BRAND constants or theme tokens used | ✅ / ❌ | |
| RC-005 | Backend service throws domain exceptions (not JPA violations) | ✅ / ❌ | |
| RC-006 | SSE personal channel used for owner-targeted events | ✅ / ❌ | |
| RC-007 | UUID vs Long types verified in Java | ✅ / ❌ | |
| RC-008 | Verified org edit block enforced in both backend + frontend | ✅ / ❌ | |
| RC-009 | No console.log in components | ✅ / ❌ | |
| RC-010 | No new files created unless necessary | ✅ / ❌ | |

---

## Summary

<!-- 1–3 sentences describing the overall quality of the implementation -->

## Blocking Issues

<!-- Must be fixed before merge. Leave empty if none. -->
-

## Suggestions (non-blocking)

<!-- Style, refactor, or future improvement ideas -->
-

---

> Status: **PASS** — ready to merge
> Status: **NEEDS-CHANGES** — address blocking issues and re-review
