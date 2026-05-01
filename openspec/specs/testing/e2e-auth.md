# E2E Auth Tests

**Version:** 1.0 · **Date:** 2026-05-01  
**Project:** `auth-tests` · **Auth:** none (fresh context per test)  
**Location:** `tests/auth/`

---

## login.spec.ts

**Page:** `/login`  
**POM:** `LoginPage`

| Test | Steps | Assert |
|---|---|---|
| admin login | fill admin credentials → submit | redirect to `/dashboard/admin` |
| organizer login | fill organizer credentials → submit | redirect to `/dashboard/organizer` |
| wrong password | fill correct email + wrong password → submit | error message visible |
| empty email | submit empty form | required validation on email |
| empty password | fill email only → submit | required validation on password |
| invalid email format | fill `notanemail` → submit | format validation error |

---

## logout.spec.ts

**Page:** `/dashboard/organizer` (logged in as organizer)  
**POM:** none (direct actions)

| Test | Steps | Assert |
|---|---|---|
| logout from organizer sidebar | click Logout button in sidebar | redirect to `/` or `/login` |
| session cleared after logout | logout → navigate to `/dashboard/organizer` | redirect to `/login` |

---

## register.spec.ts

**Page:** `/register`  
**POM:** `RegisterPage`

| Test | Steps | Assert |
|---|---|---|
| successful registration | fill all fields with valid data → submit | redirect to login or confirmation page |
| duplicate email | register with existing email → submit | error: email already in use |
| password mismatch | fill mismatched confirm password → submit | validation error |
| required fields | submit empty form | required errors on all fields |
| weak password | fill password below policy → submit | password strength error |

**Cleanup:** successful registrations create real accounts — use unique generated emails  
(e.g. `e2e-test-${Date.now()}@evena.id.vn`) and note for manual cleanup if needed.

---

## Notes

- Auth tests use a **fresh browser context** with no stored session — never reuse admin/organizer sessions
- Login page selector: `data-testid` or role-based locators from `LoginPage` POM
- After successful login, wait for navigation to dashboard before asserting URL
