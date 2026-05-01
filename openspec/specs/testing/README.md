# Evena — E2E Testing Spec Index

**Version:** 1.0 · **Date:** 2026-05-01

## Overview

End-to-end tests use **Playwright** against the staging environment (`https://evena.id.vn`).
Tests are located at `/home/haa900/DACN/e2e-testing/`.

## Structure

```
specs/testing/
├── README.md            ← this file (index)
├── e2e-setup.md         ← Playwright config, projects, credentials, environment
├── e2e-conventions.md   ← naming, Page Objects, helpers, cleanup rules
├── e2e-admin.md         ← admin test suites (category, venue, organizations, refunds)
├── e2e-organizer.md     ← organizer test suites (events, organizations)
└── e2e-auth.md          ← auth test suites (login, logout, register)
```

## Quick links

| Topic | File |
|---|---|
| How to run / configure | [e2e-setup.md](e2e-setup.md) |
| Naming + POM + cleanup conventions | [e2e-conventions.md](e2e-conventions.md) |
| Admin test coverage | [e2e-admin.md](e2e-admin.md) |
| Organizer test coverage | [e2e-organizer.md](e2e-organizer.md) |
| Auth test coverage | [e2e-auth.md](e2e-auth.md) |

## Test scope

| Role | Suites | File |
|---|---|---|
| Admin | category · venue · organizations · refunds | `tests/admin/` |
| Organizer | events · organizations | `tests/organizer/` |
| Auth | login · logout · register | `tests/auth/` |

## Running

```bash
cd e2e-testing
npx playwright test                           # all tests
npx playwright test tests/admin/             # admin only
npx playwright test tests/organizer/         # organizer only
npx playwright test tests/auth/              # auth only
npx playwright test --ui                     # interactive mode
npx playwright test --headed                 # visible browser
```

## Key rule

Tests run against **staging** (`https://evena.id.vn`), never production.
Every test cleans up its own data via API teardown in `afterEach`.
