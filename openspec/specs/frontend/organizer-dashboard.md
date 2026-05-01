# Evena — Organizer Dashboard Components

**Version:** 1.0 · **Date:** 2026-05-01

---

## Sidebar

**File:** `src/components/Sidebar/Sidebar.tsx`

| Property | Value |
|---|---|
| Background | `#E4E6F5` |
| Width | `240px` |
| User info box background | `#FFFFFF` |
| User info box border | `1px solid ${BRAND.divider}` |

The user info box must have a **white background** with a divider border so it stands out against the sidebar's `#E4E6F5` background. Do not use `rgba(42,51,99,0.06)` or any tint of the sidebar color.

---

## DashboardHeader

**File:** `src/components/DashboardHeader/DashboardHeader.tsx`

### Auth resolution rule

The component resolves user display name and avatar from multiple sources in priority order:

```typescript
const { auth } = useAuth();
const { data: meData } = useGetMeQuery();

const resolvedName   = userName   || meData?.data?.name      || auth.user?.name      || 'User';
const resolvedAvatar = userAvatar || meData?.data?.avatarUrl  || auth.user?.avatarUrl || undefined;
```

**Why:** Pages that omit the `userName` prop (Orders, Refunds, FlexPass) used to display "User". The component now falls back to the auth context and API data rather than defaulting to a hard-coded string.

**Rule:** Never render the literal string `'User'` as the display name unless all auth sources are empty.

---

## EventFilters

**File:** `src/components/EventFilters/EventFilters.tsx`

### Layout rules

| Property | Value |
|---|---|
| MUI `TextField` size | `medium` (no `size="small"`) |
| MUI `FormControl` size | `medium` |
| `borderRadius` on inputs | `12px` |
| `borderRadius` on Create button | `12px` |
| Search icon | Left `startAdornment` (not right) |
| Button font size | `16px` |
| Button padding | `12px 24px` |

Layout must match OrganizationFilters visually (same height, same font size).

### i18n keys

| UI element | i18n key |
|---|---|
| Category select placeholder | `event.filter.category` |
| Time range — All Time | `event.filter.allTime` |
| Time range — This Week | `event.filter.thisWeek` |
| Time range — This Month | `event.filter.thisMonth` |
| Time range — This Year | `event.filter.thisYear` |
| Status — DRAFT | `common.status.draft` |
| Status — PUBLISHED | `common.status.published` |
| Status — ONGOING | `common.status.ongoing` |
| Status — COMPLETED | `common.status.completed` |
| Status — CANCELLED | `common.status.cancelled` |

Never hardcode status label strings in this component.

### Stat card row (status summary)

Displays 5 clickable stat cards (one per status). Clicking a card sets that status as the active filter.

```typescript
const statusMeta: Record<string, { color: string; bg: string }> = {
  DRAFT:     { color: '#888888', bg: '#F5F5F5' },
  PUBLISHED: { color: '#2E7D32', bg: '#E8F5E9' },
  ONGOING:   { color: '#283593', bg: '#E8EAF6' },
  COMPLETED: { color: '#1565C0', bg: '#E3F2FD' },
  CANCELLED: { color: '#C62828', bg: '#FFEBEE' },
};
```

Active card: `border: 2px solid <color>`, inactive: `border: 1px solid transparent`.

### Category filter — API field note

`GET /api/events/my-events` returns `categoryName` (string) in the response, **not** `categoryId`.  
The TypeScript `EventListResponse` type previously declared `categoryId: number` — this was incorrect.

Client-side filtering by category must:
1. Find the category name from the `initialCategories` array using the selected `categoryId`
2. Filter events by comparing `event.categoryName === categoryName`

```typescript
if (selectedCategory) {
  const categoryName = initialCategories.find((c) => c.id === selectedCategory)?.name;
  if (categoryName) {
    filtered = filtered.filter((event) => event.categoryName === categoryName);
  }
}
```

Using `event.categoryId` for filtering will always return empty results.

### MUI v5 deprecation

Use `slotProps` (not `InputProps`) for input adornments:

```tsx
slotProps={{ input: { startAdornment: <InputAdornment position="start"><Search /></InputAdornment> } }}
```

### Select onChange type

`SelectChangeEvent` always yields a `string` value at runtime regardless of the `MenuItem`'s `value` type. Cast explicitly:

```typescript
const handleCategoryChange = (e: SelectChangeEvent) => {
  setSelectedCategory(Number(e.target.value) || null);
};
```

---

## OrganizationFilters

**File:** `src/components/OrganizationFilters/OrganizationFilters.tsx`

### Stat cards

Three stat cards with icon + count + label layout:

| Stat | Icon | Icon color | Background |
|---|---|---|---|
| Total Organizations | `<Business />` | `#5C6BC0` | `#EEF0FA` |
| Verified | `<CheckCircle />` | `#2E7D32` | `#E8F5E9` |
| Pending Verification | `<HourglassEmpty />` | `#E65100` | `#FFF3E0` |

Icon container: `36×36px`, `borderRadius: 10px`.  
Icon font size: `18px`.

---

## FileUploadManager

**File:** `src/components/FileUploadManager/FileUploadManager.tsx`

### Same-file re-upload fix

HTML `<input type="file">` does not fire `onChange` if the same file is selected again (value unchanged). Reset the input value immediately after reading files:

```tsx
onChange={(e) => {
  handleFiles(e.target.files);
  e.target.value = '';
}}
```

This must be applied to every file input in the component.

### File name display

Long file names are truncated with `overflow: 'hidden', textOverflow: 'ellipsis', whiteSpace: 'nowrap'` and a fixed `maxWidth`.

---

## TicketTypeManagement

**File:** `src/components/TicketTypeManagement/TicketTypeManagement.tsx`

Ticket type cards use a flex row layout with status-colored badge. Follows the same stat-card visual pattern as OrganizationFilters (colored icon container, label, count).

---

## Layout — LayoutWithSidebar

**File:** `src/components/layout/LayoutWithSidebar.tsx`

| Region | Background |
|---|---|
| Outer wrapper (`Box` containing sidebar + content) | `#FFFFFF` |
| Main content area | `#F7F7F7` |
| Main content `borderRadius` | `20px` |
| Main content padding | `p: 3` |

The outer wrapper must be `#FFFFFF` so the `#F7F7F7` content area has visible contrast. If both are the same color, the layout loses visual depth.
