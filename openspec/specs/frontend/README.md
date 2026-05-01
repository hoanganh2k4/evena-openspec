# Evena — Frontend Spec Index

**Version:** 1.0 · **Date:** 2026-05-01

## Structure

```
specs/frontend/
├── README.md                  ← this file
└── organizer-dashboard.md     ← organizer dashboard components + layout
```

## Stack

- **Framework:** Next.js 15 (App Router), TypeScript
- **UI library:** MUI v5
- **State:** RTK Query (server) + Redux (auth)
- **i18n:** react-i18next, locales at `src/i18n/locales/{en,vi}/translation.json`
- **Colors:** `BRAND` constants at `src/utils/constants/constant.ts` — no hardcoded hex
- **Routing:** `/dashboard/organizer/**` (organizer), `/dashboard/admin/**` (admin)

## Layout

```
LayoutWithSidebar (outer bg: #FFFFFF)
├── Sidebar (bg: #E4E6F5, width: 240px)
└── main content (bg: #F7F7F7, borderRadius: 20px, p: 3)
```

## Quick links

| Topic | File |
|---|---|
| Organizer sidebar, profile, event filters, ticket management | [organizer-dashboard.md](organizer-dashboard.md) |
| Global rules (no hardcoded colors, i18n) | [../rules.md](../rules.md) |
| Brand color constants | `src/utils/constants/constant.ts` |
