# Object: User

**Table:** `users` | **PK:** UUID (UUIDv7) | **Module:** auth

## Schema

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | UUID | PK | UUIDv7 |
| name | String | NOT NULL | |
| email | String | UNIQUE, NOT NULL | |
| password_hash | String | NOT NULL | bcrypt |
| phone | String | nullable | |
| status | UserStatus | NOT NULL, default=ACTIVE | see [enums/user-status.md](../../enums/user-status.md) |
| avatar_url | String | nullable | S3/MinIO URL |
| email_verified | Boolean | default=false | |
| email_verification_token | String | nullable | cleared on verify |
| password_reset_token | String | nullable | |
| password_reset_expires_at | LocalDateTime | nullable | |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Relationships

- ManyToMany → `Role` via `user_roles` (EAGER)
- OneToMany → `Organization` (as owner)
- OneToMany → `OrganizationMember`
- OneToMany → `Order`
- OneToMany → `Ticket`
- OneToMany → `ScanLog` (scanned_by)

## Rules

- Email must be unique across both `users` and `pending_registrations` tables
- Password stored as bcrypt hash — never plaintext
- `email_verified` must be `true` before login is allowed
- `status == ACTIVE` required to log in
- Avatar replacement deletes the previous S3/MinIO file before uploading new one
