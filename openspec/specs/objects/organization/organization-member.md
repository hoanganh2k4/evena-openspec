# Object: OrganizationMember

**Table:** `organization_members` | **PK:** Long | **Module:** event

## Schema

**Unique constraint:** `(organization_id, user_id)`

| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | Long | PK, auto-increment | |
| organization_id | Long | FK → organizations.id, NOT NULL | |
| user_id | UUID | FK → users.id, NOT NULL | |
| role | OrganizationRole | NOT NULL | see [enums/organization-role.md](../../enums/organization-role.md) |
| invitation_accepted | Boolean | NOT NULL, default=false | |
| created_at | LocalDateTime | auto | |
| updated_at | LocalDateTime | auto | |

## Rules

- A user may hold different roles in different organizations
- `invitation_accepted=false` = pending invitation (visible only to invitee)
- Invitee must accept before gaining access to the organization
- OWNER role cannot be removed or changed — owner is the original creator
