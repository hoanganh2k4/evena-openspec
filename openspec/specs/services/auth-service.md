# Service: AuthService & UserService

---

## AuthService

### `register(RegisterRequest)` / `registerOrganizer(RegisterRequest)`
1. Validate email uniqueness across `users` and `pending_registrations`
2. Hash password (bcrypt)
3. Create `PendingRegistration` with UUID verification token
4. Send verification email
5. Return `UserResponse` (not yet in `users` table)

### `verifyEmail(String token)`
1. Find `PendingRegistration` by token; throw if not found or expired
2. Create `User` with `emailVerified=true`, `status=ACTIVE`
3. Assign role (`USER` or `ORGANIZER`)
4. Delete `PendingRegistration` row

### `login(LoginRequest)`
1. Find user by email
2. Verify bcrypt hash
3. Guard: `emailVerified == true`
4. Guard: `status == ACTIVE`
5. Generate access JWT + refresh JWT
6. Set refresh token in `HttpOnly` cookie

### `refreshAccessToken(String refreshToken)`
1. Validate signature and expiry
2. Load user from token subject
3. Generate new access JWT

### `requestPasswordReset(PasswordResetRequest)`
1. Find user by email
2. Generate reset token + expiry (`now + 1 hour`)
3. Persist on user; send email

### `resetPassword(PasswordChangeRequest)`
1. Find user by `passwordResetToken`
2. Guard: token not expired
3. Hash new password; clear token fields

---

## UserService

### `uploadAvatar(UUID userId, MultipartFile file)`
1. Upload to S3/MinIO
2. Delete previous avatar file if it exists
3. Update `user.avatarUrl`

### `deleteAvatar(UUID userId)`
1. Delete file from S3/MinIO
2. Set `user.avatarUrl = null`
