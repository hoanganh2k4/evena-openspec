# E2E Setup — Playwright Configuration

**Version:** 1.0 · **Date:** 2026-05-01

---

## Location

```
/home/haa900/DACN/e2e-testing/
├── playwright.config.ts
├── global.setup.ts          # saves admin session → .auth/admin.json
├── organizer.setup.ts       # saves organizer session → .auth/organizer.json
├── fixtures/auth.fixture.ts # credentials + login helpers
└── .auth/                   # git-ignored session state files
```

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `BASE_URL` | `https://evena.id.vn` | Frontend base URL |
| `API_URL` | `https://api.evena.id.vn` | Backend base URL |
| `TEST_ORGANIZER_EMAIL` | `anh.nguyenhaa2652004@hcmut.edu.vn` | Organizer login |
| `TEST_CUSTOMER_EMAIL` | `test.customer@evena.id.vn` | Customer login |

---

## Playwright projects

| Project | Auth state | Test match | Dependencies |
|---|---|---|---|
| `setup` | none | `global.setup.ts` | — |
| `setup-organizer` | none | `organizer.setup.ts` | — |
| `auth-tests` | none | `tests/auth/**` | — |
| `chromium` | `.auth/admin.json` | `tests/admin/**` | `setup` |
| `organizer` | `.auth/organizer.json` | `tests/organizer/**` | `setup-organizer` |

---

## Global settings

```typescript
timeout:    60_000          // per-test timeout
retries:    2 (CI), 0 (local)
fullyParallel: false        // tests within a file run sequentially
trace:      on-first-retry
screenshot: only-on-failure
video:      retain-on-failure
```

---

## Credentials

| Role | Email | Password | Source |
|---|---|---|---|
| Admin | `admin@eventtickets.com` | `Admin@123` | hardcoded in `auth.fixture.ts` |
| Organizer | `$TEST_ORGANIZER_EMAIL` | `Admin@123` | env var with fallback |
| Customer | `$TEST_CUSTOMER_EMAIL` | `Test@123456` | env var with fallback |

---

## Session setup flow

```
npm run test
  → setup project: global.setup.ts
      loginViaUI(admin) → saves .auth/admin.json
  → setup-organizer: organizer.setup.ts
      loginViaUI(organizer) → saves .auth/organizer.json
  → chromium project: storageState: .auth/admin.json
  → organizer project: storageState: .auth/organizer.json
```

`loginViaUI` fills the login form and waits for `networkidle` + dashboard load.
Sessions are reused across all tests in that project without re-login.

---

## Installation

```bash
cd e2e-testing
npm install
npx playwright install chromium
```
