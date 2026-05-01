# E2E Conventions — Naming, Page Objects, Helpers, Cleanup

**Version:** 1.0 · **Date:** 2026-05-01

---

## Test file layout

```
tests/
├── admin/
│   ├── category.spec.ts
│   ├── venue.spec.ts
│   ├── organizations.spec.ts
│   └── refunds.spec.ts
├── organizer/
│   ├── events.spec.ts
│   └── organizations.spec.ts
└── auth/
    ├── login.spec.ts
    ├── logout.spec.ts
    └── register.spec.ts
```

---

## Page Object Model (POM)

All page interaction is encapsulated in `pages/`. Tests never use raw selectors.

```
pages/
├── LoginPage.ts
├── RegisterPage.ts
├── AdminContentPage.ts          # /dashboard/admin/content
├── AdminOrganizationsPage.ts    # /dashboard/admin/organizations
├── AdminRefundsPage.ts          # /dashboard/admin/refunds
├── OrganizerOrganizationsPage.ts
└── OrganizerEventsPage.ts
```

### POM contract

Each page object must expose:
- `goto()` — navigate to the page and wait for content
- Locators for all interactive elements (inputs, buttons, dialogs)
- `expectSuccessToast(message)` — wait for and assert toast

---

## UI text constants

All UI strings used in selectors/assertions live in `constants/ui-text.ts`.
Never hard-code UI strings in test files — use `UI_TEXT.admin.toast.categoryCreated` etc.

**Why:** when i18n keys change, update one file only.

---

## Unique test data naming

Always suffix test data names with a timestamp to avoid conflicts between runs:

```typescript
function uniqueName(prefix: string): string {
  return `${prefix} ${Date.now()}`;
}
// e.g. "E2E Category 1746123456789"
```

---

## Cleanup rule

Every test that creates data **must** clean it up via API in `afterEach`, even on failure.

```typescript
afterEach(async () => {
  if (createdId) await cleanupOrganization(adminApi, createdId);
});
```

Cleanup uses the helper functions in `helpers/api-client.ts`, not UI actions.

---

## API client helpers (`helpers/api-client.ts`)

| Function | Description |
|---|---|
| `createAdminApiClient()` | Returns APIRequestContext with admin JWT |
| `createOrganizerApiClient()` | Returns APIRequestContext with organizer JWT |
| `createTestOrganization(api, name)` | POST /api/organizations → returns id |
| `verifyOrganization(adminApi, id)` | PATCH /api/organizations/{id}/verify |
| `cleanupOrganization(api, id)` | DELETE /api/organizations/{id} |
| `createTestEvent(api, params)` | POST /api/events → returns event id (string UUID) |
| `cleanupEvent(api, id)` | DELETE /api/events/{id} |
| `getFirstCategoryId(api)` | GET /api/categories → returns first id |
| `getFirstVenueId(api)` | GET /api/venues → returns first id |
| `findCategoryIdByName(api, name)` | Searches category list by name |
| `cleanupCategory(api, name)` | Finds by name + deletes |
| `findVenueIdByName(api, name)` | Searches venue list by name |
| `cleanupVenue(api, name)` | Finds by name + deletes |
| `findPendingRefundId(api)` | GET /api/refund-requests/organizer → first PENDING |

---

## Test structure template

```typescript
test.describe('Feature (role)', () => {
  let api: APIRequestContext;
  let createdId: number | string | null = null;

  test.beforeAll(async () => {
    api = await createAdminApiClient();
  });

  test.afterAll(async () => {
    await api.dispose();
  });

  test.afterEach(async () => {
    if (createdId) {
      await cleanupResource(api, createdId);
      createdId = null;
    }
  });

  test('create', async ({ page }) => { ... });
  test('edit', async ({ page }) => { ... });
  test('delete', async ({ page }) => { ... });
  test('search', async ({ page }) => { ... });
  test('access control — unauthenticated', async ({ page }) => { ... });
});
```

---

## Access control test pattern

Every suite must include an access control test verifying unauthenticated users are redirected:

```typescript
test('access control — unauthenticated', async ({ browser }) => {
  const ctx = await browser.newContext(); // no storageState
  const page = await ctx.newPage();
  await page.goto('/dashboard/admin/...');
  await expect(page).toHaveURL(/\/login/);
  await ctx.close();
});
```

---

## Selector priority

1. `data-id` attribute (preferred — set on cards/rows)
2. `data-testid` attribute
3. ARIA role + accessible name
4. UI text from `constants/ui-text.ts`
5. CSS class (last resort — fragile)
