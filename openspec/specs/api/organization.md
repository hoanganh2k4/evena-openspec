# API: Organizations & Members

---

## `/api/organizations`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/` | [ORGANIZER\|ADMIN] | Create organization |
| PUT | `/{organizationId}` | [ORGANIZER\|ADMIN] | Update organization |
| GET | `/{organizationId}` | [Public] | Get by ID |
| GET | `/my-organizations` | [Auth] | Caller's organizations |
| GET | `/` | [Public] | List all (paginated) |
| GET | `/search?keyword=&page=&size=` | [Public] | Search |
| PATCH | `/{organizationId}/verify` | [ADMIN] | Approve organization |
| DELETE | `/{organizationId}` | [ORGANIZER\|ADMIN] | Delete (PENDING only) |
| GET | `/{organizerId}/details` | [Auth] | Detail with members |
| GET | `/user/{userId}/organizations` | [Auth] | All orgs for a user |

### CreateOrganizationRequest
```json
{
  "name": "string (2-200)",
  "description": "string (10+)",
  "logoUrl": "string (optional)",
  "website": "https://... (optional)",
  "email": "org@example.com (optional)",
  "phone": "string (optional)"
}
```

---

## `/api/organizations/{organizationId}/members`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/invite` | [ORGANIZER\|ADMIN] | Invite member |
| POST | `/{memberId}/accept` | [Auth] | Accept invitation |
| POST | `/{memberId}/reject` | [Auth] | Reject invitation |
| PATCH | `/{memberId}/role` | [ORGANIZER\|ADMIN] | Change role |
| DELETE | `/{memberId}` | [ORGANIZER\|ADMIN] | Remove member |
| GET | `/` | [Auth] | List members |
| GET | `/paged?page=&size=` | [Auth] | List members (paginated) |

---

## `/api/invitations`

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/pending` | [Auth] | Pending invitations for current user |
| POST | `/{invitationId}/accept` | [Auth] | Accept |
| POST | `/{invitationId}/reject` | [Auth] | Reject |
