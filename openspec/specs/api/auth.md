# API: Auth & User

---

## `/api/auth`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/login` | [Public] | Email/password login |
| POST | `/register` | [Public] | Register as customer (USER role) |
| POST | `/registerOrganizer` | [Public] | Register as organizer (ORGANIZER role) |
| POST | `/forgot-password` | [Public] | Send password reset email |
| POST | `/reset-password` | [Public] | Consume reset token, set new password |
| GET | `/verify-email?token=` | [Public] | Verify email address |
| GET | `/me` | [Auth] | Get current user profile |
| POST | `/refresh` | [Public] | Refresh access token via cookie |
| POST | `/logout` | [Auth] | Invalidate refresh token cookie |

### POST `/api/auth/login`

**Request:**
```json
{ "email": "user@example.com", "password": "••••••••" }
```

**Response:**
```json
{
  "accessToken": "eyJ...", "refreshToken": "eyJ...",
  "tokenType": "Bearer", "expiresIn": 3600,
  "user": {
    "id": "uuid", "name": "Jane", "email": "user@example.com",
    "status": "ACTIVE", "emailVerified": true,
    "roles": ["USER"], "createdAt": "...", "updatedAt": "..."
  }
}
```

### RegisterRequest
```json
{
  "name": "string",
  "email": "user@example.com",
  "phone": "string",
  "password": "Min 6 chars, complex",
  "confirmPassword": "same"
}
```

---

## `/api/users`

| Method | Path | Auth | Description |
|---|---|---|---|
| PUT | `/avatar` (multipart) | [Auth] | Upload/replace avatar |
| DELETE | `/avatar` | [Auth] | Delete avatar |
