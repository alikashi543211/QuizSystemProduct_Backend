# Racing Eye Pro — Flutter app status and API requirements

**For:** Racing Eye client / backend team  
**App:** Racing Eye Pro (`com.racingeyepro.re`)  
**Date:** 10 September 2026

This note covers what is already built in the Flutter app, which APIs it calls today, and what the backend must add or change so Google/Apple sign-in, the four membership plans, and Dubai Pay work end to end.

---

## 1. What is done in the app

### Account

- Sign in with email + password, mobile number + OTP, and create account
- Forgot password / reset password
- Profile (name, email, phone, country, photo)
- Sign out (Settings), with confirmation: “Are you sure you want to sign out?”
- **Google Sign-In** on Android (Firebase project **Racing Eye Pro**)
- **Sign in with Apple** on iOS
- Push notifications via **Firebase Cloud Messaging** (not raw APNs)

### Plans (Membership)

The Plans screen shows the **original four plans**, not a single Basic package:

| Plan (shown in app) | Internal id | Price |
| --- | --- | --- |
| Racecard | `free` | Free |
| Form | `form` | USD 6.99 / month (USD 59 / year) |
| Paddock | `paddock` | USD 14.99 / month (USD 129 / year) |
| Owner’s Box | `bloodstock` | USD 49 / month (USD 490 / year) |

- **Racecard** still switches on the AI API (`POST /api/plan` with `tier=free`).
- **Form, Paddock, Owner’s Box** require a Racing Eye account, then open a **Dubai Pay webview**.

### Dubai Pay (in app)

1. User taps **Pay … with Dubai Pay** on a paid plan.
2. If they are not signed in to `app.racingeye.ae`, the sign-in screen opens first.
3. App calls `POST /api/v3/package_subscribe` with `package_id`, `plan`, `plan_name`, and `user_id`.
4. App opens a **webview** (`webview_flutter`):
   - If subscribe returns a `url` / `payment_url`, that URL is loaded.
   - Otherwise the app loads  
     `https://racingeye.ae/pay?package_id={id}&user_id={userId}&plan={form|paddock|bloodstock}`
5. When the browser reaches  
   `https://racingeye.ae/payment_response`  
   the app closes the screen and refreshes the plan.

**Important:** use a **full app restart** (`flutter run`) after payment/WebView changes — **hot restart** can leave stale native WebView state and cause crashes on Android.

The pay page and the payment-response page must be implemented/hosted on the existing Racing Eye site (same flow as the current website Dubai Pay checkout).

### Other features already calling the account API

- Favourites (list / add / remove)
- Register FCM device token after sign-in

---

## 2. Two backends the app uses

| Role | Base URL | Auth |
| --- | --- | --- |
| Racing / AI (cards, Ghayath, session plan) | `https://ai.racingeye.ae` | `X-RE-Key` + session |
| Account, profile, packages, Dubai Pay user | `https://app.racingeye.ae/api/v3` | Header `x-api-key` + `Authorization: Bearer {token}` after login |

Dubai Pay pages (not JSON):

- Checkout: `https://racingeye.ae/pay`
- Return / success: `https://racingeye.ae/payment_response`

---

## 3. Account APIs already used (keep as they are)

Unless noted in section 5, these already exist and the app is integrated:

| Method | Endpoint | When |
| --- | --- | --- |
| POST | `/api/v3/login` | Email or phone sign-in |
| POST | `/api/v3/register-user` | Create account |
| POST | `/api/v3/forgot_pass` | Forgot password |
| POST | `/api/v3/resend-otp` | Resend OTP |
| POST | `/api/v3/confirm_code` | Verify OTP |
| POST | `/api/v3/verify-otp` | Verify OTP (alternate) |
| POST | `/api/v3/password_reset` | New password |
| GET | `/api/v3/user-profile` | Load profile |
| POST | `/api/v3/update-user-profile` | Save profile (+ optional photo) |
| POST | `/api/v3/send-profile-otp` | OTP when changing email/phone |
| GET | `/api/v3/get-packages-list` | Package catalogue |
| GET | `/api/v3/get_package_detail` | Current subscription |
| POST | `/api/v3/package_subscribe` | Start paid checkout |
| POST | `/api/v3/cancel_subscription` | Cancel paid plan |
| GET | `/api/v3/fetch-favourite-horses` | Favourites |
| POST | `/api/v3/add-favourite-horse` | Add favourite |
| POST | `/api/v3/remove-favourite-horse` | Remove favourite |
| POST | `/api/v3/registertoken` | FCM token |

Login / register already send `device_type` (`Android` or `Ios` / `IOS`).

---

## 4. New API required (does not exist yet — app already calls it)

### `POST /api/v3/social-login`

Used for **Google (Android)** and **Apple (iOS)**. Please add this endpoint (or confirm the real path if it already exists under another name).

**Headers**

- `x-api-key` (same as other v3 routes)
- `Accept: application/json`  
- No Bearer token (user is not signed in yet)

**Body (form fields)**

| Field | Google | Apple |
| --- | --- | --- |
| `provider` | `google` | `apple` |
| `token` | Google ID token | Apple identity token |
| `id_token` | same as `token` | same as `token` |
| `email` | from Google (when present) | from Apple (when present) |
| `name` | display name | given + family name (first tap only) |
| `device_type` | `Android` | `Ios` |
| `apple_user_id` | — | Apple user identifier |
| `authorization_code` | — | Apple authorization code |

**Expected success (HTTP 200 or 201)** — same shape as email login:

```json
{
  "token": "<Bearer token>",
  "user": {
    "id": 123,
    "name": "…",
    "email": "…"
  },
  "message": "Signed in"
}
```

Create the user if they do not exist; return the existing user if the Google/Apple account is already linked.

---

## 5. APIs / pages that need modification

### 5.1 `GET /api/v3/get-packages-list`

**Today:** the app received only one **Basic** package. The UI no longer shows that list as the main Plans screen.

**Needed:** return the four membership products so `package_id` in Dubai Pay is a real catalogue id:

| name | type / slug | price | notes |
| --- | --- | --- | --- |
| Racecard | `free` | 0 | optional; app treats free locally |
| Form | `form` | 6.99 | required |
| Paddock | `paddock` | 14.99 | required |
| Owner’s Box | `bloodstock` | 49 | required (`bloodstock` is the stored id) |

Match names (`Form`, `Paddock`, `Owner's Box`) **or** `type` (`form`, `paddock`, `bloodstock`). Until this is live, the app still sends `package_id=form|paddock|bloodstock`.

### 5.2 `POST /api/v3/package_subscribe`

**App sends**

| Field | Example |
| --- | --- |
| `package_id` | numeric id from the list, **or** `form` / `paddock` / `bloodstock` |
| `plan` | `form` / `paddock` / `bloodstock` |
| `plan_name` | `Form` / `Paddock` / `Owner's Box` |
| Bearer token | logged-in user |

**Needed**

- Accept slug ids, not only integers.
- Create / reuse a Dubai Pay session for that user + plan.
- Return a checkout URL when possible:

```json
{
  "url": "https://racingeye.ae/pay?package_id=…&user_id=…",
  "message": "OK"
}
```

(`url` may also sit under `data`.)

If `url` is missing, the app still opens `https://racingeye.ae/pay` with query params.

### 5.3 Dubai Pay page `GET https://racingeye.ae/pay`

**Query params the app sends**

| Param | Meaning |
| --- | --- |
| `package_id` | Catalogue id or `form` / `paddock` / `bloodstock` |
| `user_id` | Account user id from login |
| `plan` | `form` / `paddock` / `bloodstock` |

**Verified issue (Sep 2026):** a test request to  
`https://racingeye.ae/pay?package_id=form&user_id=1&plan=form`  
returned **HTTP 302 → `https://app.racingeye.ae`** (homepage), **not** a Dubai Pay checkout.  
Until this is fixed, the in-app webview cannot show payment even when the Flutter WebView is stable.

**Needed**

- **Do not redirect** `/pay` to the marketing homepage when `package_id`, `user_id`, and `plan` are present.
- Render (or redirect to) the **Dubai Pay / gateway checkout** for that plan and amount (6.99 / 14.99 / 49 USD, or AED if configured).
- Create a **fresh payment session per checkout** (one order per user + plan tap); do not reuse an expired session.
- Bind the payment to `user_id` + `package_id` + `plan`.
- After success or failure, **redirect the webview** to  
  `https://racingeye.ae/payment_response`  
  (the app detects this URL and closes the webview).

**Recommended:** `POST /api/v3/package_subscribe` should return the **exact** URL to load (including any signed `encRequest` / `access_code` fields), for example:

```json
{
  "data": {
    "url": "https://racingeye.ae/pay?package_id=3&user_id=42&plan=form&order_id=abc123",
    "encRequest": "...",
    "access_code": "..."
  },
  "message": "OK"
}
```

The mobile app opens `data.url` when present; otherwise it builds the `/pay` query string above.

### 5.4 `https://racingeye.ae/payment_response`

**Needed**

- Show a short success/failure page (the app waits ~2 seconds then pops).
- On **success**, mark the user subscribed on `app.racingeye.ae` **and** set the racing session plan on `ai.racingeye.ae` to `form`, `paddock`, or `bloodstock` so Ghayath / Paddock locks unlock.
- On **failure / cancel**, do not change the plan.

### 5.5 `GET /api/v3/get_package_detail`

Should reflect the plan after Dubai Pay success (`package_subscribed`, `package_name`, `package_expiry`, remaining predictions if used).

### 5.6 AI API `POST /api/plan` (existing)

Still used for **Racecard / free** only (`tier=free`). Paid upgrades should come from Dubai Pay success, not the old sandbox checkout.

If the AI API currently only accepts `free` for `post/plan`, that is fine. Paid entitlements must be applied when Dubai Pay confirms (section 5.4).

---

## 6. Payment flow (sequence)

```
User taps Pay on Form / Paddock / Owner’s Box
        │
        ├─ not signed in → Racing Eye account login / Google / Apple
        │
        ▼
POST /api/v3/package_subscribe
  package_id, plan, plan_name  + Bearer token
        │
        ▼
Webview → https://racingeye.ae/pay?package_id=&user_id=&plan=
        │
        ▼
Dubai Pay (card / gateway)
        │
        ▼
Redirect → https://racingeye.ae/payment_response
        │
        ├─ backend activates plan on account API + AI API
        └─ app closes webview and refreshes membership
```

---

## 7. Firebase (already configured in the app)

| Item | Value |
| --- | --- |
| Console project | **Racing Eye Pro** |
| Project id | `testing-faf2c` |
| Android / iOS app id | `com.racingeyepro.re` |

**Still required on the client side (console / Apple / Google Cloud), not in Flutter:**

1. Authentication → Google is enabled (done for development).
2. Add the **release** Android SHA-1 / SHA-256 (Play Store keystore) next to the debug fingerprints.
3. Upload an **APNs Auth Key** under Firebase → Project settings → Cloud Messaging (iOS push). Do not send APNs from the app directly; FCM uses that key.
4. Apple Developer: enable **Sign in with Apple** and **Push Notifications** on App ID `com.racingeyepro.re`.

---

## 8. What the client backend team should do first

Priority order:

1. **Add `POST /api/v3/social-login`** (Google + Apple) and return the same `token` + `user` as email login.
2. **Put Form, Paddock, and Owner’s Box** in `get-packages-list` (stop serving only Basic as the catalogue).
3. **Accept `form` / `paddock` / `bloodstock`** on `package_subscribe` and on `/pay`.
4. **Dubai Pay success** must hit `/payment_response` and **activate the matching plan** on both `app.racingeye.ae` and `ai.racingeye.ae`.
5. Confirm amounts and currency with Dubai Pay (USD list prices vs AED settlement).

Until steps 2–4 are live, the app will still open the webview, but payment may not complete or the plan may not unlock after pay.

---

## 9. Out of scope / not in this app build

- Sandbox “approve / decline card” checkout is no longer used for paid membership.
- Direct APNs from the phone (FCM only).
- Web or desktop builds.

---

Please send any different live paths (for example if social login is already `/api/v3/google-login`) and we will point the app at them.
