# Service: AuthService & UserService

---

## AuthService

### `register(RegisterRequest)` / `registerOrganizer(RegisterRequest)`
1. Validate email uniqueness via `assertEmailAvailable(email)`:
   - If a `users` row already exists → throw `BadRequestException`
   - If a `PendingRegistration` exists and is **not yet expired** → throw (verification email was already sent)
   - If a `PendingRegistration` exists but **expired** → delete it and allow re-registration
2. Hash password (bcrypt)
3. Create `PendingRegistration` with UUID verification token, `expiresAt = now + 24h`
4. Send verification email
5. Return `UserResponse` (user not yet in `users` table)

### `verifyEmail(String token)`
1. Find `PendingRegistration` by token; throw if not found
2. Guard: `expiresAt` not before now; if expired delete the record and throw
3. Guard: email not already in `users` table (concurrent registrations)
4. Assign role from `pendingRegistration.roleName` (`USER` or `ORGANIZER`)
5. Create `User` with `emailVerified=true`, `status=ACTIVE`
6. Delete `PendingRegistration` row

### `login(LoginRequest)`
1. Authenticate via Spring `AuthenticationManager` (email + bcrypt)
2. Guard: `emailVerified == true`
3. Guard: `status == ACTIVE`
4. Generate access JWT (short-lived) + refresh JWT (30 days)
5. Set refresh token in `HttpOnly; SameSite=Lax; Path=/` cookie
6. **Clear legacy path cookies** — two `Set-Cookie MaxAge=0` headers per legacy path (`/api/auth`, `/api`):
   - With `Domain=<cookieDomain>` (e.g. `.evena.id.vn`) — clears wildcard-domain cookies
   - Without `Domain` attribute — clears host-only cookies (`api.evena.id.vn`) set by older backend versions that had no `cookie-domain` configured
7. Return `LoginResponse` (refresh token omitted from body)

### `refreshAccessToken(String refreshToken)`
1. Validate JWT signature and expiry; throw if invalid
2. Assert token type is `"refresh"` (not `"access"`)
3. Load user by UUID subject from token
4. Generate new access JWT + **new refresh JWT** (token rotation)
5. Set new refresh cookie (`HttpOnly; SameSite=Lax; Path=/`, 30 days)
6. **Clear legacy path cookies** (`Path=/api/auth`, `Path=/api`, `MaxAge=0`)
7. Return `RefreshTokenResponse` (new refresh token omitted from body)

> **Cookie path rule**: The refresh cookie MUST always be set with `Path=/`.  
> Omitting the path causes the browser to default to the URL directory path  
> (e.g. `/api/auth`), which creates a second cookie with higher path-specificity  
> that shadows the canonical cookie on subsequent requests → 400 on refresh.

### `logout`
1. Set `refreshToken` cookie with `MaxAge=0; Path=/` (clears canonical cookie)
2. Clear legacy path cookies (`Path=/api/auth`, `Path=/api`, `MaxAge=0`)

### `requestPasswordReset(PasswordResetRequest)`
1. Find user by email
2. Generate reset token + expiry (`now + 24h`)
3. Persist on user; send email

### `resetPassword(PasswordChangeRequest)`
1. Find user by `passwordResetToken`
2. Guard: token not expired
3. Hash new password; clear token fields

---

## Frontend: AuthInitializer

`AuthInitializer` runs once per page lifecycle (root layout, never unmounts in production).

### Rules
- Uses `useRef(false)` as a guard (`initialized`). The ref is **never reset in cleanup**.
- No `isActive` flag — abandoning the in-flight fetch via cleanup would leave auth permanently uninitialized under React Strict Mode (cleanup fires before the response arrives → `setAuthFromInit` never called → infinite redirect loop).
- React Strict Mode fires `effect → cleanup → effect` in development. Because `initialized.current` stays `true` after the first invocation, the second effect invocation returns early with no second fetch.
- On page load: calls `POST /api/auth/refresh` (credentials: include). On success → sets access token + user in Redux. On failure → clears Redux auth + resets RTK Query caches.
- Skips the refresh call if `sessionStorage.__evena_logout` is set (user explicitly logged out).

### Why token rotation can cause logout (if guard is removed)
Two concurrent calls with the same refresh token both validate correctly. Each generates a new token. Both set `Set-Cookie` with `Path=/` — but due to browser cookie-handling race conditions, the losing response's cookie can shadow the winning one, or path-specific cookies from older sessions can take priority, resulting in a stale token being sent on the next refresh → 400.

---

## UserService

### `uploadAvatar(UUID userId, MultipartFile file)`
1. Upload to S3/MinIO
2. Delete previous avatar file if it exists
3. Update `user.avatarUrl`

### `deleteAvatar(UUID userId)`
1. Delete file from S3/MinIO
2. Set `user.avatarUrl = null`
