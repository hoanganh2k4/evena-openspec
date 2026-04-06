# Service: OrganizationService & OrganizationMemberService

---

## OrganizationService

### `createOrganization(CreateOrganizationRequest)`
1. Set `owner = currentUser`, `verified = false`
2. Persist and return

### `updateOrganization(Long id, UpdateOrganizationRequest)`
1. Verify caller owns org or is ADMIN
2. If `verified=true`: block identity field changes (`name`, etc.)
3. Apply non-identity updates; persist

### `verifyOrganization(Long id)` — ADMIN only
Sets `verified=true`. Enables event creation for the org.

### `deleteOrganization(Long id)`
1. Only `PENDING` orgs with no events may be deleted
2. `APPROVED` orgs MUST NOT be deleted — archive only

---

## OrganizationMemberService

### `inviteMember(Long organizationId, InviteMemberRequest)`
Creates `OrganizationMember` with `invitationAccepted=false`. Sends invitation email.

### `acceptInvitation(Long invitationId)` / `rejectInvitation(Long invitationId)`
Sets `invitationAccepted=true` or deletes the row.

### `updateMemberRole(Long organizationId, Long memberId, UpdateMemberRoleRequest)`
Updates `role`; OWNER role cannot be changed.

### `removeMember(Long organizationId, Long memberId)`
Deletes the `OrganizationMember` row.
