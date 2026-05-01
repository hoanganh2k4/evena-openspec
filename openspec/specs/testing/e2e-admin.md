# E2E Admin Tests

**Version:** 1.0 · **Date:** 2026-05-01  
**Project:** `chromium` · **Auth:** `.auth/admin.json`  
**Location:** `tests/admin/`

---

## category.spec.ts

**Page:** `/dashboard/admin/content` (Categories tab)  
**POM:** `AdminContentPage`

| Test | Precondition | Steps | Assert |
|---|---|---|---|
| create | none | fill name + desc → submit | success toast + name in table |
| edit | create via form | find row → click edit → change name → submit | success toast + updated name in table |
| delete | create via form | find row → click delete → confirm dialog | success toast + name NOT in table |
| validation — required | none | submit empty form | required error on name field |
| validation — min length | none | submit 1-char name | min-length error |
| access control | no session | navigate to page | redirect to /login |

**Teardown:** `cleanupCategory(api, name)` in `afterEach`

---

## venue.spec.ts

**Page:** `/dashboard/admin/content` (Venues tab)  
**POM:** `AdminContentPage`

| Test | Precondition | Steps | Assert |
|---|---|---|---|
| create | none | fill all required fields → submit | success toast + name in table |
| edit | create via form | find row → click edit → change name → submit | success toast + updated name |
| delete | create via form | find row → click delete → confirm | success toast + name NOT in table |
| validation — required fields | none | submit empty form | required errors shown |
| access control | no session | navigate to page | redirect to /login |

**Required fields:** name, address, city, latitude, longitude, capacity  
**Teardown:** `cleanupVenue(api, name)` in `afterEach`

---

## organizations.spec.ts

**Page:** `/dashboard/admin/organizations`  
**POM:** `AdminOrganizationsPage`

| Test | Precondition | Steps | Assert |
|---|---|---|---|
| search | org exists | type org name in search | org visible |
| filter — unverified | org exists (unverified) | click Unverified filter | only unverified orgs shown |
| verify | create org via organizer API | admin clicks Verify | success toast + org shows Verified badge |
| delete | create org via organizer API | admin clicks delete → confirm | success toast + org not in list |
| access control | no session | navigate to page | redirect to /login |

**Setup:** `createTestOrganization(organizerApi, name)` + `verifyOrganization(adminApi, id)` in `beforeAll`  
**Teardown:** `cleanupOrganization(adminApi, id)` in `afterEach`

---

## refunds.spec.ts

**Page:** `/dashboard/admin/refunds`  
**POM:** `AdminRefundsPage`

| Test | Precondition | Steps | Assert |
|---|---|---|---|
| display pending refunds | pending refund exists | navigate | refund request listed |
| search | pending refund exists | type order/event text | matching refund shown |
| approve | pending refund exists | click Approve → confirm | success toast + status updated |
| reject | pending refund exists | click Reject → fill reason → confirm | success toast + status updated |
| access control | no session | navigate to page | redirect to /login |

**Note:** Refund precondition requires a real order in REFUND_REQUESTED state.  
`findPendingRefundId(api)` is used to locate an existing one; skip test if none found.
