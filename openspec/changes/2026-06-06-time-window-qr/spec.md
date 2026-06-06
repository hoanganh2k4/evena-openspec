# Change: Time-Window QR Token + Mobile Security

**Date:** 2026-06-06  
**Status:** Implemented  
**Repos:** backend-evena, frontend-evena, mobile_app

---

## Problem

Static `qrPayload` stored in DB never rotates — a screenshot of the QR code remains valid forever, bypassing physical ticket verification.

---

## Solution

### 1. Time-Window QR Token (backend)

Replace the static `qrPayload` in all ticket API responses with a **time-window token** that rotates every 5 minutes.

**Token format:**
```
TW:{ticketId}:{userId}:{timeSlot}:{hmac}
```

| Part | Description |
|---|---|
| `TW` | Literal prefix — distinguishes from static QR (4-part) |
| `ticketId` | Long — DB primary key |
| `userId` | UUID — ticket owner |
| `timeSlot` | `floor(epochSeconds / 300)` — changes every 5 minutes |
| `hmac` | 64-char hex — `HMAC-SHA256("TW:ticketId:userId:timeSlot", QR_SECRET)` |

**Security guarantee:**
- Screenshot valid for at most **5 minutes** (current slot only — no grace period)
- Cannot forge without server secret
- Bound to specific owner (userId)

**Performance:**
- Computed on-the-fly during `toTicketResponse()` — ~1μs HMAC, no DB write
- No scheduled job or cron required
- Clock sync via NTP required for multi-instance deployments

### 2. Scanner backward compatibility

`scanTicket()` and `validateTicket()` verify in priority order:

```
1. TW: prefix → verifyTimeWindowToken() → current slot only
2. 4-part format → verifyAndExtractTicketId() → static QR fallback
3. Anything else → INVALID_QR
```

Static QR fallback preserves compatibility with tickets issued before this change.

### 3. Client polling (mobile + web)

- Poll `GET /api/orders/my-tickets` or `GET /api/orders/tickets/{id}` every **5 minutes**
- Each response returns the current slot's token as `qrPayload`
- QR rendered client-side from `qrPayload` text
- No new endpoints required

### 4. Mobile screenshot protection

| Platform | Mechanism |
|---|---|
| Android | `expo-screen-capture.preventScreenCaptureAsync()` → `FLAG_SECURE` — screenshots and app-switcher produce black image |
| iOS | `enableAppSwitcherProtectionAsync(0.8)` blurs app-switcher; `addScreenshotListener` shows alert after capture |

**Implementation:** `ScreenGuard` component wraps `useScreenshotProtection` hook. Applied to all 3 return states of `TicketDetailScreen` (loading, error, main).

### 5. Mobile force-logout on expired session

**Problem:** When JWT refresh token fails, `tokenStorage.clearTokens()` was called but `AuthContext` state (`isAuthenticated`) was not updated → app stayed on customer screens.

**Fix:** Module-level EventEmitter pattern:

```
client.ts (interceptor)
  → 401 + refresh fail (or no refresh token)
  → authEvents.emitForceLogout()

AuthContext (useEffect subscriber)
  → tokenStorage.clearAll()
  → setState({ isAuthenticated: false })
  → RootNavigator re-renders → AuthNavigator → Login screen
```

---

## Files Changed

### backend-evena
| File | Change |
|---|---|
| `QRCodeService.java` | Add `TimeWindowResult` record, `generateTimeWindowToken()`, `verifyTimeWindowToken()` |
| `OrderMapper.java` | `toTicketResponse()` uses `generateTimeWindowToken()` instead of `ticket.getQrPayload()` |
| `TicketService.java` | `scanTicket()` + `validateTicket()` try time-window first, fall back to static QR |

### frontend-evena
| File | Change |
|---|---|
| `app/dashboard/customer/cart/page.tsx` | QR modal renders `QRCodeSVG` from `qrPayload`; `useGetMyTicketsQuery` polls every 5 min via `pollingInterval` |

### mobile_app
| File | Change |
|---|---|
| `src/services/api/authEvents.ts` | New — lightweight EventEmitter for auth lifecycle |
| `src/services/api/client.ts` | Emit `forceLogout` when refresh token missing or refresh fails |
| `src/contexts/AuthContext.tsx` | Subscribe to `forceLogout` event, clear state |
| `src/screens/customer/TicketDetailScreen.tsx` | QR from `ticket.qrPayload`, poll every 5 min, all states wrapped in `ScreenGuard` |
| `src/hooks/useScreenshotProtection.ts` | Screenshot protection hook (FLAG_SECURE + iOS listener) |
| `src/components/ui/ScreenGuard.tsx` | Wrapper component applying screenshot protection |

---

## Security Layers (combined)

```
1. API auth:       @PreAuthorize("isAuthenticated()") on all ticket endpoints
2. JWT lifecycle:  interceptor auto-refresh; force-logout on refresh failure
3. QR expiry:      time-window token valid max 5 minutes
4. HMAC integrity: cannot forge without QR_SECRET
5. Ownership:      userId embedded + verified in token
6. Screenshot:     FLAG_SECURE (Android) + iOS listener
```
