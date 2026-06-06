# Evena Mobile App — Deploy Plan

> **Stack:** React Native + Expo SDK 54  
> **Target Platforms:** Android (APK/AAB via Google Play), iOS (IPA via App Store)  
> **Updated:** 2026-05-06

---

## 1. Architecture Overview

```
┌────────────────────────────────────────────────┐
│               Evena Mobile App                 │
│          (React Native / Expo SDK 54)          │
├──────────────────────┬─────────────────────────┤
│  Customer Role       │  Organizer Role          │
│  • Event Browse      │  • Event Management      │
│  • Ticket Purchase   │  • QR Scan & Check-in   │
│  • My Tickets (QR)  │  • Orders & Revenue      │
│  • Orders / Refunds  │  • Refund Review         │
│  • Profile           │  • Organizations         │
├──────────────────────┴─────────────────────────┤
│              Admin Role                        │
│  • Dashboard Stats (real-time)                 │
│  • Users / Events / Orgs Management           │
│  • Activity Logs                               │
├────────────────────────────────────────────────┤
│          API Gateway (Nginx / Port 80)         │
│  /api/*  →  Spring Boot :8080                 │
│  /sse/*  →  SSE Service :8000                 │
└────────────────────────────────────────────────┘
```

---

## 2. Environment Configuration

### 2.1 app.json (Expo Config)

```json
{
  "expo": {
    "name": "Evena",
    "slug": "evena-mobile",
    "version": "1.0.0",
    "android": {
      "package": "com.evena.mobile",
      "versionCode": 1
    },
    "ios": {
      "bundleIdentifier": "com.evena.mobile",
      "buildNumber": "1"
    },
    "extra": {
      "apiBaseUrl": "https://api.evena.com",
      "sseBaseUrl": "https://api.evena.com/sse"
    }
  }
}
```

### 2.2 Environment Profiles

| Environment | `apiBaseUrl`                  | Notes                        |
|-------------|-------------------------------|------------------------------|
| Development | `http://192.168.x.x` (LAN IP) | Use `npm run update-ip`      |
| Staging     | `https://staging.evena.com`   | Internal testing              |
| Production  | `https://api.evena.com`       | Live gateway                  |

> **Android Emulator:** Localhost is automatically rewritten to `10.0.2.2` by `client.ts`.

---

## 3. Build Process

### 3.1 Prerequisites

```bash
# Install EAS CLI globally
npm install -g eas-cli

# Login to Expo account
eas login

# Configure project (first time only)
eas build:configure
```

### 3.2 eas.json

```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "android": { "buildType": "apk" },
      "env": { "API_BASE_URL": "http://192.168.1.x" }
    },
    "staging": {
      "distribution": "internal",
      "android": { "buildType": "apk" },
      "env": { "API_BASE_URL": "https://staging.evena.com" }
    },
    "production": {
      "distribution": "store",
      "android": { "buildType": "app-bundle" },
      "ios": { "buildConfiguration": "Release" },
      "env": { "API_BASE_URL": "https://api.evena.com" }
    }
  }
}
```

### 3.3 Build Commands

```bash
# Development build (internal testing APK)
eas build --platform android --profile development

# Staging build (QA testing)
eas build --platform all --profile staging

# Production build (store submission)
eas build --platform all --profile production
```

---

## 4. Feature Checklist (Pre-Deploy)

### 4.1 Customer Features
- [x] Auth: Login / Register / Forgot Password
- [x] Event Browse with search, category filter, sort
- [x] Event Detail with ticket types & pricing
- [x] Ticket Purchase flow (quantity selection)
- [x] Checkout with MoMo / VNPay integration
- [x] Payment Return handling (success / failure)
- [x] My Tickets with QR code display
- [x] My Orders with status tracking
- [x] Order Detail with refund request
- [x] Profile view and edit
- [x] Change password

### 4.2 Organizer Features
- [x] QR Scan & Check-in (camera-based)
- [x] Scan Result (valid / used / invalid)
- [x] Scan History per event
- [x] Event List with status filtering
- [x] Create Event (title, description, date, venue, category, org)
- [x] Event Management (stats, ticket types, quick actions)
- [x] Publish event from DRAFT
- [x] Organizer Orders with revenue summary
- [x] Refund Review (approve / reject with note)
- [x] Organization Management (create, view, status)

### 4.3 Admin Features
- [x] Dashboard with real-time stats (users, events, orders, revenue)
- [x] Pending organization & refund alerts
- [x] Users Management (search, view, lock/unlock)
- [x] Events Oversight (filter by status)
- [x] Organizations Review (approve / suspend)
- [x] Activity Logs (paginated, with action icons)

### 4.4 Security
- [x] JWT tokens stored in `expo-secure-store` (encrypted)
- [x] Auto token refresh on 401 via interceptor
- [x] HTTPS-only in production (`https://api.evena.com`)
- [x] Deep link URL scheme: `evena://` for payment callbacks
- [x] No sensitive data in AsyncStorage or logs (production)

---

## 5. Security Hardening

### 5.1 Token Storage
- Access token: `SecureStore` (iOS Keychain / Android Keystore)
- Refresh token: `SecureStore`
- Never stored in `AsyncStorage` or global state logs

### 5.2 Network Security
```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<network-security-config>
  <domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="true">api.evena.com</domain>
  </domain-config>
  <!-- Allow cleartext for dev only -->
  <debug-overrides>
    <trust-anchors>
      <certificates src="user"/>
    </trust-anchors>
  </debug-overrides>
</network-security-config>
```

### 5.3 Permissions (Android `AndroidManifest.xml`)
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CAMERA" />
<!-- No location, contacts, or unnecessary permissions -->
```

### 5.3b QR Screenshot Protection

| Platform | Mechanism | Effect |
|---|---|---|
| Android | `expo-screen-capture.preventScreenCaptureAsync()` | `FLAG_SECURE` — screenshots and app-switcher produce black image |
| iOS | `enableAppSwitcherProtectionAsync(0.8)` | Blurs content in app-switcher |
| iOS | `addScreenshotListener` | Alert shown after screenshot taken |

Applied via `ScreenGuard` component wrapping `useScreenshotProtection` hook. Active on all return states of `TicketDetailScreen`.

### 5.3c Force Logout on Session Expiry

When JWT refresh fails (refresh token missing or expired):
1. `client.ts` interceptor calls `authEvents.emitForceLogout()`
2. `AuthContext` subscriber clears all tokens + sets `isAuthenticated = false`
3. `RootNavigator` re-renders → `AuthNavigator` → Login screen

Time-window QR token on screen becomes invalid within 5 minutes regardless.

### 5.4 iOS `Info.plist` Keys
```xml
<key>NSCameraUsageDescription</key>
<string>Evena uses the camera to scan QR codes for event check-in.</string>
```

---

## 6. CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/mobile-build.yml
name: Mobile Build

on:
  push:
    branches: [main]
    paths: ['mobile_app/**']

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
        working-directory: mobile_app
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas build --platform android --profile production --non-interactive
        working-directory: mobile_app

  build-ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
        working-directory: mobile_app
      - uses: expo/expo-github-action@v8
        with:
          eas-version: latest
          token: ${{ secrets.EXPO_TOKEN }}
      - run: eas build --platform ios --profile production --non-interactive
        working-directory: mobile_app
```

**Required secrets:**
- `EXPO_TOKEN`: EAS / Expo access token
- `APPLE_ID`, `APPLE_TEAM_ID`: For iOS signing (managed via EAS credentials)

---

## 7. Store Submission

### 7.1 Google Play Store

```bash
# Submit AAB directly via EAS
eas submit --platform android --profile production

# Or manual: upload AAB at play.google.com/console
# Track: Internal Testing → Closed Testing → Production
```

**Google Play requirements:**
- Target API level: Android 14 (API 34+)
- Privacy Policy URL required
- Data safety form: camera, no location, no contacts

### 7.2 Apple App Store

```bash
# Submit IPA via EAS
eas submit --platform ios --profile production

# Or via Transporter / Xcode Organizer
# Review: TestFlight → App Review → Production
```

**App Store requirements:**
- iOS 15.0+ deployment target
- Camera permission description in `Info.plist`
- Payment gateway disclosure (MoMo, VNPay)
- Privacy policy URL
- Age rating: 4+

---

## 8. OTA Updates (Expo Updates)

For hotfixes that don't require store review (JS-only changes):

```bash
# Publish OTA update (no native changes)
eas update --branch production --message "Hotfix: payment callback handling"
```

**OTA eligibility:**
- ✅ Bug fixes in JS/TS screens
- ✅ API endpoint changes
- ✅ UI text / styling tweaks
- ❌ New native modules / permissions
- ❌ Expo SDK version bumps
- ❌ Native code changes (camera, secure store)

**`app.json` OTA config:**
```json
{
  "updates": {
    "url": "https://u.expo.dev/<project-id>",
    "enabled": true,
    "fallbackToCacheTimeout": 0,
    "checkAutomatically": "ON_LOAD"
  },
  "runtimeVersion": {
    "policy": "sdkVersion"
  }
}
```

---

## 9. Monitoring & Crash Reporting

### 9.1 Recommended Tools

| Tool                    | Purpose                          |
|-------------------------|----------------------------------|
| **Sentry** (Expo)       | Crash reports, error tracking    |
| **Expo Updates**        | OTA deployment tracking          |
| **Firebase Analytics**  | Screen views, user flows         |

### 9.2 Sentry Setup

```bash
npx @sentry/wizard@latest -i reactNative
```

```ts
// App.tsx
import * as Sentry from '@sentry/react-native';
Sentry.init({
  dsn: 'https://<key>@sentry.io/<project>',
  environment: __DEV__ ? 'development' : 'production',
  enabled: !__DEV__,
});
```

---

## 10. Rollout Strategy

| Phase        | Target          | Duration  | Success Criteria                      |
|--------------|-----------------|-----------|---------------------------------------|
| Alpha        | Internal team   | 1 week    | No critical crashes, all flows work   |
| Beta (closed)| 100 test users  | 2 weeks   | < 0.5% crash rate, payment confirmed  |
| Staged roll  | 10% production  | 3 days    | Metrics stable, no 5xx spikes         |
| Full release | 100% production | —         | App store rating ≥ 4.0               |

---

## 11. API Dependency Matrix

| Mobile Feature          | Backend Endpoint                        | Auth Required |
|-------------------------|-----------------------------------------|---------------|
| Login / Register        | `POST /api/auth/login|register`         | No            |
| Forgot Password         | `POST /api/auth/forgot-password`        | No            |
| Browse Events           | `GET /api/events?status=PUBLISHED`      | No            |
| Event Detail            | `GET /api/events/{id}`                  | No            |
| Create Order            | `POST /api/orders`                      | CUSTOMER      |
| Checkout                | `POST /api/orders/checkout`             | CUSTOMER      |
| My Tickets              | `GET /api/orders/my-tickets`            | CUSTOMER      |
| My Orders               | `GET /api/orders/my-orders`             | CUSTOMER      |
| Refund Request          | `POST /api/refund-requests`             | CUSTOMER      |
| Update Profile          | `PUT /api/users/profile`                | CUSTOMER      |
| Organizer Events        | `GET /api/events/my-events`             | ORGANIZER     |
| Scan QR                 | `POST /api/orders/tickets/scan`         | ORGANIZER     |
| Scan History            | `GET /api/orders/events/{id}/scan-logs` | ORGANIZER     |
| Refund Review           | `PUT /api/refund-requests/{id}/review`  | ORGANIZER     |
| Admin Stats             | `GET /api/admin/dashboard/stats`        | ADMIN         |
| Admin Users             | `GET /api/admin/users`                  | ADMIN         |
| Admin Events            | `GET /api/admin/events`                 | ADMIN         |
| Admin Organizations     | `GET /api/admin/organizations`          | ADMIN         |
| Activity Logs           | `GET /api/admin/activity-logs`          | ADMIN         |

---

## 12. Known Limitations & Future Work

| Item                             | Priority | Notes                                          |
|----------------------------------|----------|------------------------------------------------|
| Push notifications (FCM/APNs)   | High     | Real-time order/event updates                  |
| FlexPass marketplace             | Medium   | Ticket resale — frontend only currently        |
| Offline ticket viewing           | Medium   | Cache QR code for offline access               |
| Biometric login (Face ID/Touch)  | Medium   | SecureStore + expo-local-authentication        |
| i18n (Vietnamese/English toggle) | Low      | i18next is already in frontend                 |
| Dark mode support                | Low      | Design system already has dark theme tokens    |
| Social login (Google/Apple/FB)   | Low      | OAuth2 flow required                           |
| Image upload (avatar/event)      | Low      | expo-image-picker + S3 presigned URL           |
