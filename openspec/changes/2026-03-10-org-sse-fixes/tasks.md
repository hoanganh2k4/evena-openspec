# Tasks: Organization SSE Fixes & Verified-Org Edit Block

**ID:** TASK-001
**Spec:** SPEC-001
**Author:** team
**Date:** 2026-03-10
**Status:** done

---

## Checklist

### backend

- [x] **T1** — `EventRepository.java`: add `countAllEventsByCategory` JPQL query
  counting events of ANY status (not just PUBLISHED)
- [x] **T2** — `CategoryService.java`: update `deleteCategory()` to use
  `countAllEventsByCategory`; throw `BadRequestException` with descriptive message
- [x] **T3** — `SSENotificationService.java`: change `notifyOrganizationVerified`
  signature to include `UUID ownerId`; emit to both `organizer,admin` channel
  and `user:{ownerId}` personal channel
- [x] **T4** — `OrganizationService.java`: add verified-org edit guard
  (`org.verified && !isAdmin()` → `BadRequestException`); update
  `verifyOrganization()` call to pass `owner.getId()` (UUID)

### sse-service

- [x] **T5** — `config/channels.json`: add `"organization"` to
  `user:{id}` pattern's `allowed_resources`

### frontend

- [x] **T6** — `src/stores/services/OrganizerApi.ts`: fix
  `updateOrganization.invalidatesTags` →
  `[{ type: 'Organizer', id }, 'Organizer']`
- [x] **T7** — `src/providers/SSEProvider.tsx`:
  - Add `notification` state + `clearNotification` callback
  - Add `channel` field to `SSEEvent` in `handleEvent`
  - Split `ORGANIZATION_VERIFIED` into dedicated case; set personal
    toast when `channel.startsWith('user:')`
  - Add global `Snackbar` + `Alert` render
- [x] **T8** — `src/stores/types/sse.ts`:
  - Add `channel: string` to `SSEEvent`
  - Add `SSENotification` interface
  - Add `notification` + `clearNotification` to `SSEContextType`
- [x] **T9** — `src/components/OrganizationCard/OrganizationRowCard.tsx`:
  - Edit button condition: `isOwner && !organization.verified`
  - Tooltip for verified state: "Verified organizations cannot be edited..."
- [x] **T10** — `app/dashboard/organizer/organizations/page.tsx`:
  - Guard in `handleOrganizationEdit`: show snackbar + return early if verified
- [x] **T11** — `src/components/OrganizationFilters/OrganizationFilters.tsx`:
  - Remove redundant manual `refetchInvitations()` SSE useEffect
  - Remove `console.log` calls
- [x] **T12** — `src/i18n/locales/en/translation.json`: add
  `messages.error.verifiedOrgCannotEdit`
- [x] **T13** — `src/i18n/locales/vi/translation.json`: add
  `messages.error.verifiedOrgCannotEdit`
