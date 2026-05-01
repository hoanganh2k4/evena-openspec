# E2E Organizer Tests

**Version:** 1.0 · **Date:** 2026-05-01  
**Project:** `organizer` · **Auth:** `.auth/organizer.json`  
**Location:** `tests/organizer/`

---

## organizations.spec.ts

**Page:** `/dashboard/organizer/organizations`  
**POM:** `OrganizerOrganizationsPage`

| Test | Precondition | Steps | Assert |
|---|---|---|---|
| create | none | click Create Org → fill name + desc → submit | success toast + org in list |
| edit | create org via API | find card → click edit → change name → submit | success toast + updated name |
| delete | create org via API | find card → click delete → confirm | success toast + org NOT in list |
| search | create org via API | type name in search input | org card visible |
| search — no match | create org via API | type unrelated string | org NOT visible |
| verified org — cannot edit | create org via API + admin verify | find card → click edit button | button disabled OR error shown |
| access control | no session | navigate to page | redirect to /login |

**Setup for events test:**
```typescript
beforeAll: createTestOrganization(organizerApi, name) → orgId
           verifyOrganization(adminApi, orgId)      // needed for event creation
```

**Teardown:** `cleanupOrganization(organizerApi, orgId)` in `afterEach`

---

## events.spec.ts

**Page:** `/dashboard/organizer/events`  
**POM:** `OrganizerEventsPage`

### beforeAll setup
```typescript
orgId      = createTestOrganization(organizerApi, name)
             verifyOrganization(adminApi, orgId)
categoryId = getFirstCategoryId(adminApi)
venueId    = getFirstVenueId(adminApi)
```

Organization must be verified before events can be created.

| Test | Precondition | Steps | Assert |
|---|---|---|---|
| smoke — page loads | org verified | navigate | page title "Events" visible |
| create | org verified | click Create Event → fill all fields → submit | success toast + event card in list |
| view detail | create event via API | click event card | event detail page loads, title visible |
| edit title | create event via API | click edit → change title → submit | success toast + new title shown |
| delete | create event via API | click delete on card → confirm | success toast + card NOT in list |
| search | create event via API | type event title in search | card visible |
| search — no match | create event via API | type unrelated string | card NOT visible |
| filter — category | create event via API | select category from dropdown | event card visible |
| access control | no session | navigate to page | redirect to /login |

### Create event form required fields

| Field | Type | Value in tests |
|---|---|---|
| title | text | uniqueName('E2E Event') |
| description | text | 'Test event description' |
| startAt | datetime-local | 30 days from now |
| endAt | datetime-local | 31 days from now |
| organization | select | orgId from beforeAll |
| category | select | categoryId from beforeAll |
| venue | select | venueId from beforeAll |

### Category filter behaviour

The `/events/my-events` API returns `categoryName` (string), **not** `categoryId`.  
Frontend filters client-side by matching `event.categoryName === selectedCategoryName`.  
Test must verify that selecting a category correctly shows/hides matching events.

**Teardown:** `cleanupEvent(organizerApi, eventId)` in `afterEach`  
`cleanupOrganization(organizerApi, orgId)` in `afterAll`
