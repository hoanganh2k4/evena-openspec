# Change: Organizer UI Redesign

**Date:** 2026-05-01  
**Branch:** develop  
**Status:** Merged to staging, deployed to Vercel

---

## Summary

A batch of UI fixes and redesigns across the organizer dashboard.  
No API changes; all changes are frontend-only except the `EventListResponse` type correction.

---

## Changes

### 1. FileUploadManager — same-file re-upload

**Problem:** Uploading the same file twice in a row silently did nothing.  
**Root cause:** HTML `<input type="file">` does not fire `onChange` when the selected path hasn't changed.  
**Fix:** Reset `e.target.value = ''` after reading files.

---

### 2. EventFilters — layout, i18n, stat cards

**Problems:**
- Input height / font size inconsistent with OrganizationFilters
- Status display was a row of chips — not intuitive
- Labels showed English instead of Vietnamese (`Category`, `All Time`)
- Category filter always returned zero results

**Root cause of broken filter:**  
`GET /api/events/my-events` returns `categoryName` (string), not `categoryId` (number).  
The TypeScript type was wrong. Client-side filtering was comparing `event.categoryId === selectedCategoryId`, which always failed because `categoryId` doesn't exist in the response.

**Fixes:**
- Set input size to `medium`, `borderRadius: 12px` (matches OrganizationFilters)
- Replaced status chips with 5 clickable stat cards (color-coded per status)
- Added `useTranslation` for all filter labels
- Fixed category filter: look up name from `initialCategories`, then filter `event.categoryName === name`
- Fixed `InputProps` deprecation → `slotProps`
- Fixed `SelectChangeEvent` type: cast value with `Number()`

---

### 3. OrganizationFilters — stat cards

**Problem:** Total / Verified / Pending display was plain bordered card, low visual hierarchy.  
**Fix:** Each stat shows a colored icon box (36×36px, `borderRadius: 10px`) + number + label.

---

### 4. Sidebar — user info box

**Problem:** User info box blended into sidebar background (`rgba(42,51,99,0.06)` tint of `#E4E6F5`).  
**Fix:** Changed to `background: #fff, border: 1px solid ${BRAND.divider}`.

---

### 5. LayoutWithSidebar — outer background

**Problem:** Outer wrapper was changed to `#F7F7F7` in a previous session; content area is also `#F7F7F7` — no contrast.  
**Fix:** Revert outer wrapper to `#FFFFFF`.

---

### 6. DashboardHeader — user name resolution

**Problem:** Pages that omit the `userName` prop (Orders, Refunds, FlexPass) displayed the fallback string `'User'`.  
**Fix:** Component now resolves internally via `useAuth()` + `useGetMeQuery()` before falling back to `'User'`.

---

### 7. EventListResponse type correction

**Problem:** TypeScript type declared `categoryId: number` but the API never returns this field.  
**Fix:** Type updated to declare `categoryName: string` instead of `categoryId: number`.

---

### 8. CI/CD — Vercel deployment workflow

**Problem:** `amondnet/vercel-action@v25` bundles a Vercel CLI version below the required minimum (≥47.2.2).  
**Fix:** Replaced with `npm install --global vercel@latest` + direct `vercel` CLI invocation.

---

## Affected files

| File | Change |
|---|---|
| `src/components/FileUploadManager/FileUploadManager.tsx` | Same-file re-upload reset |
| `src/components/EventFilters/EventFilters.tsx` | Full redesign |
| `src/components/EventsManagement/EventsManagement.tsx` | Category filter fix |
| `src/components/OrganizationFilters/OrganizationFilters.tsx` | Stat card redesign |
| `src/components/Sidebar/Sidebar.tsx` | User info box background |
| `src/components/layout/LayoutWithSidebar.tsx` | Outer background revert |
| `src/components/DashboardHeader/DashboardHeader.tsx` | Internal auth resolution |
| `src/types/event.ts` (or similar) | `categoryName` field correction |
| `.github/workflows/deploy.yml` | Vercel CLI update |
