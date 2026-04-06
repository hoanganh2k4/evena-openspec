# Object: PendingRegistration

**Table:** `pending_registrations` | **PK:** Long | **Module:** auth

Temporary row created during email-based registration.
Deleted when email is verified or token expires.

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| name | String | NOT NULL | |
| email | String | UNIQUE, NOT NULL | |
| phone | String | NOT NULL | |
| password_hash | String | NOT NULL | bcrypt |
| role_name | String | NOT NULL | `USER` or `ORGANIZER` |
| verification_token | String | UNIQUE, NOT NULL | UUID, sent by email |
| expires_at | LocalDateTime | NOT NULL | |
| created_at | LocalDateTime | auto | |
| version | Long | — | optimistic lock |

## Rules

- Token expires after a fixed TTL (e.g. 24 hours)
- On token expiry the row may be cleaned up by a scheduled task
- On successful verification: row is deleted, a `User` row is created
- Duplicate email in `pending_registrations` or `users` must be rejected at creation time
