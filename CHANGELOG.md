# Changelog — Security, Trust, Payment & Hotel Analytics Update

All existing functionality (customer/hotel/admin auth, subscriptions, meal
balance/reservation logic, QR flow, dashboards, payouts, reviews, etc.) is
preserved as-is. Everything below is additive or a targeted fix.

## 1. CAPTCHA verification (backend-validated)

- **New**: `server/src/models/Captcha.js`, `services/captchaService.js`,
  `controllers/captchaController.js`, `routes/captchaRoutes.js` — generates
  a random 6-character CAPTCHA, stores only its bcrypt hash server-side
  (TTL-expiring, max 5 attempts), and verifies submissions against that
  hash. The plaintext is never trusted from the client.
- **New**: `mobile/src/components/CaptchaChallenge.js` — shared distorted
  SVG CAPTCHA widget with a refresh button, used everywhere a CAPTCHA is
  needed.
- Wired into:
  - **Customer & hotel registration** — always required
    (`authController.registerCustomer` / `registerHotel`).
  - **Login** — required only after 5+ recent failed attempts on that
    account (see "Login attempt limits" below).
  - **Account verification** (`VerifyContactScreen.js`) — replaced the
    previous client-only CAPTCHA with the same backend-validated one.

## 2. Email/phone verification temporarily disabled

- OTP send/verify code (`otpController.js`, routes, `authController.js`
  triggers) is commented out, not deleted, and documented for how to
  restore it.
- `email` and `phone` fields remain on the User model and in the DB.
- `.env.example` documents `EMAIL_VERIFICATION_ENABLED` /
  `PHONE_VERIFICATION_ENABLED` flags for when this is turned back on.

## 3-6. Rate limiting, login attempt limits, registration/IP abuse controls

- **`server/src/middleware/rateLimiters.js`**: added `captchaLimiter`,
  `registerLimiter`, `paymentLimiter`, `reservationLimiter`, `qrLimiter`,
  `accountDeletionLimiter` on top of the existing `generalLimiter` /
  `authLimiter`. Applied to: CAPTCHA generation, registration, subscription
  create-order/verify-payment, reservation creation, hotel QR verify/confirm,
  and account deletion.
- **Login brute-force protection** (`authController.loginByRole`, new
  `User.failedLoginAttempts` / `User.lockUntil` fields):
  - 1-4 failed attempts: normal login.
  - 5+: a valid CAPTCHA must accompany the login request (verified
    server-side) before the password is even checked.
  - 8+: temporary escalating cooldown (doubles per extra failure, capped
    at 30 minutes) — never a permanent lock.
  - All counters reset on a successful login.
  - Unknown-email attempts return the same generic "Invalid credentials"
    as a wrong password, without revealing account existence.

## 7. Duplicate email/phone handling

- `authController.js` now explicitly checks for duplicate email AND phone
  (normalized) before creating an account, with friendly messages:
  *"This email is already registered 😊 Please log in instead."* /
  *"This mobile number is already registered 😊 Please use another number."*
- Added a unique index on `User.phone` as a defense-in-depth backstop
  against race conditions (see `bootstrapService.repairIndexes`).

## 8. Security/audit logging

- **New**: `models/SecurityLog.js` + `services/securityLogService.js` —
  fire-and-forget, auto-expiring (90 days) log of login success/failure,
  lockouts, CAPTCHA failures, duplicate-registration attempts, registration
  success, payment-blocked-while-disabled events, and account deletions.
  Never logs passwords, tokens, OTP/CAPTCHA answers, or payment secrets.
  Ready for a future admin view; no admin UI changes made in this pass.

## 9-10. Payments temporarily disabled (real AND mock/test)

- **New env flag**: `PAYMENTS_ENABLED` (default `false`). While not exactly
  `"true"`, `subscriptionService.initiateSubscriptionPurchase` (and
  defensively, `confirmSubscriptionPurchase`) throw a friendly
  `PAYMENTS_DISABLED` error before anything — order, pending Payment row,
  subscription — is created. No fake transactions, no gateway call, no mock
  auto-success.
- `PlansScreen.js` shows the exact friendly message from the spec when this
  happens; the full plans/pricing browsing experience is untouched.
- Turning payments back on: set `PAYMENTS_ENABLED=true` and the existing
  `RAZORPAY_KEY_ID`/`RAZORPAY_KEY_SECRET` env vars — no code changes needed.

## 11-13. Hotel visibility bug fix

- **Fixed**: `customerController.listHotels` no longer filters hotels by
  plan-tier eligibility — every approved hotel is now returned to every
  customer, with a non-price-revealing `eligibleForYourPlan` flag and
  `upgradeMessage` hint per hotel.
- `getHotelDetails` no longer 404s a hotel the customer's plan doesn't
  cover; it returns full details plus the same eligibility hint.
- `resolveHotelQr` and `reservationService.reserveMeal` (the actual
  point-of-use gate) now return a specific friendly message naming the
  required tier (e.g. *"This hotel requires the Silver plan or above..."*)
  instead of a generic block or a raw price/percentage.
- `getTodayMeals` already showed all hotels — unchanged, confirmed correct.

## 14-18. Hotel view analytics

- **New**: `models/HotelView.js` — one document per (hotel, customer,
  calendar day), unique-indexed so repeated same-day views only count once.
- `customerController.getHotelDetails` records a view (fire-and-forget,
  never blocks the page load) each time a customer opens a hotel's details.
- **New**: `hotelController.getViewAnalytics` (`GET /api/hotels/analytics/views`)
  returns today/this-week/this-month aggregated counts for the
  authenticated hotel owner's own hotel only — identity always derived
  server-side, never from a client-supplied hotelId. No per-customer data
  is ever exposed to hotels.
- **New UI**: "Hotel Views 👀" card on `HotelDashboardScreen.js`.

## Plan-tier labels on every hotel

- Every hotel now carries a `planLabel` (`Basic` / `Silver` / `Golden` /
  `Premium`) — the tier its own mess price falls into — returned from
  `listHotels`, `getHotelDetails`, `resolveHotelQr`, and `getTodayMeals`.
  The underlying price itself is still never sent to the client.
- Shown as a colored badge on `HotelListCard`, `TodayMealCard`, and in the
  `HotelDetailsScreen` header, so customers can tell at a glance which
  hotels match their subscription.

## 19-21. Account deletion

- **New**: `DELETE /api/customers/account` and `DELETE /api/hotels/account`
  (`controllers/accountController.js`, shared logic). Requires the current
  password; identity always comes from the authenticated session, never the
  request body. Admin accounts are excluded.
- **Soft delete**: the User doc is anonymized (name/email/phone/images
  scrubbed, `isActive=false` — which already blocks future login) rather
  than hard-deleted, so Payment/Subscription/MealTicket/HotelPayout history
  stays intact for accounting/audit integrity and hotels can still see a
  customer name on an already-issued ticket. A hotel owner's Hotel profile
  is set to `suspended` (hidden from customers) on deletion.
- **New UI**: `DeleteAccountScreen.js` (shared, password-confirmation +
  strong warning dialog), reachable from Profile → Settings → Delete
  Account on both the customer and hotel apps.

## 22-24. MongoDB backup & recovery

- **New**: `BACKUP_AND_RECOVERY.md` — Atlas automated-backup instructions,
  manual `mongodump`/`mongorestore` procedure, backup frequency/storage
  guidance, verification steps, and a corruption/recovery runbook.

## 25-26. Data integrity / meal-balance logic

- No changes — audited the existing reservation/meal-usage/idempotency
  logic and subscription remaining-meals calculation; no vulnerabilities or
  regressions found or introduced by this update.

## 27-28. Plans preserved, no internal percentages shown

- All 4 plans (Basic/Silver/Golden/Premium), their durations/prices/meals,
  and the exact spec-provided description text remain unchanged and fully
  browsable regardless of the payments-disabled state.

## 29. Error handling

- All new error paths return friendly, non-technical messages (see
  `rateLimiters.js`, `accountController.js`, `subscriptionService.js`,
  `customerController.js` messages above) — no stack traces, Mongo errors,
  or secrets ever reach the client.

## 32. Environment variables

- `server/.env.example` updated with `PAYMENTS_ENABLED`,
  `EMAIL_VERIFICATION_ENABLED`, `PHONE_VERIFICATION_ENABLED`,
  `CAPTCHA_ENABLED` (documentation flag), and clarifying comments. No real
  secrets included.

## Files touched (backend)

`app.js`, `server.js`(unchanged), `middleware/rateLimiters.js`,
`controllers/authController.js`, `controllers/otpController.js`,
`controllers/customerController.js`, `controllers/hotelController.js`,
`controllers/captchaController.js` (new), `controllers/accountController.js`
(new), `routes/authRoutes.js`, `routes/otpRoutes.js`, `routes/captchaRoutes.js`
(new), `routes/customerRoutes.js`, `routes/hotelRoutes.js`,
`services/captchaService.js` (new), `services/securityLogService.js` (new),
`services/subscriptionService.js`, `services/reservationService.js`,
`services/bootstrapService.js`, `models/Captcha.js` (new),
`models/SecurityLog.js` (new), `models/HotelView.js` (new), `models/User.js`,
`config/planTiers.js`, `validators/authValidators.js`, `.env.example`.

## Files touched (mobile)

`context/AuthContext.js`, `api/captchaApi.js` (new), `api/otpApi.js`,
`api/customerApi.js`, `api/hotelApi.js`, `components/CaptchaChallenge.js`
(new), `components/HotelListCard.js`, `components/TodayMealCard.js`,
`screens/VerifyContactScreen.js`, `screens/CustomerRegisterScreen.js`,
`screens/HotelRegisterScreen.js`, `screens/CustomerLoginScreen.js`,
`screens/HotelLoginScreen.js`, `screens/PlansScreen.js`,
`screens/HotelDetailsScreen.js`, `screens/HotelDashboardScreen.js`,
`screens/CustomerProfileScreen.js`, `screens/HotelProfileScreen.js`,
`screens/DeleteAccountScreen.js` (new), `navigation/ProfileStack.js`,
`navigation/HotelProfileStack.js`.

## New documentation

`BACKUP_AND_RECOVERY.md`, `CHANGELOG.md` (this file).

## Follow-up fixes (this round)

- **CAPTCHA rendering bug fixed**: the distorted CAPTCHA letters' rotation/
  position/size were being recomputed with a bare `Math.random()` call on
  every re-render of `CaptchaChallenge.js` - including every keystroke
  typed into the answer field, since that updates parent state and
  re-renders the image too. The letters were visibly reshuffling/jumping as
  you typed, which read as the CAPTCHA "clipping"/flickering rapidly. Fixed
  by memoizing all per-character distortion values together, keyed only on
  the actual CAPTCHA text - the image is now stable and only changes on an
  actual refresh. Also hardened `load()` against a stale-response race (a
  newer request in flight now always wins over an older one that resolves
  late).
- **CAPTCHA removed from registration**: `CustomerRegisterScreen.js` and
  `HotelRegisterScreen.js` no longer show or require a CAPTCHA;
  `authController.registerCustomer`/`registerHotel` and
  `authValidators.js` no longer validate/require one either. CAPTCHA is
  still required right after, on the account-verification step
  (`VerifyContactScreen.js` / `otpController.verifyCaptcha`), and
  automatically on login after 5+ recent failed attempts - unchanged.
- **Admin hotel-views table**: new "Views" tab on `AdminDashboardScreen.js`
  (new `AdminHotelViewsSection.js` component) showing every hotel in a
  table - Hotel Name / Today / This Week / This Month - sorted by
  most-viewed-this-month, backed by the new `GET /api/admin/hotel-views`
  endpoint (`adminController.getHotelViewsOverview`). Meant for admin to
  show a hotel owner their reach on the platform, for trust-building.
- **Hotel views history (browsable)**: `hotelController.getViewAnalytics`
  now also returns `history.daily` (last 30 days, zero-filled) and
  `history.weekly` (last 12 calendar weeks) alongside the existing today/
  week/month totals. New `HotelViewsHistoryScreen.js`, reachable via a
  "See history →" link on the dashboard's Hotel Views card, lets a hotel
  owner toggle between day-by-day and week-by-week and page back through
  previous periods.

## Testing performed

All backend (`server/src/**/*.js`) and mobile (`mobile/src/**/*.js`) files
were syntax-validated (`node --check` for backend, Babel parse with the
React preset for JSX) after every change. This confirms the code is
syntactically correct and will load; it does **not** replace running the
app end-to-end. Before deploying, please run through:

- Register (customer + hotel) with a correct and an incorrect CAPTCHA
- Log in normally, then deliberately fail 5+ times to confirm the CAPTCHA
  requirement and (8+) the temporary cooldown appear, and that a correct
  login afterward clears them
- Duplicate email and duplicate phone registration attempts
- Browse hotels on a Basic-plan account and confirm higher-tier hotels are
  still visible, with an upgrade message on reservation attempts
- Click Subscribe/Pay with `PAYMENTS_ENABLED=false` and confirm the
  friendly alert appears and no subscription/payment record is created;
  then flip it to `true` (with real/test Razorpay keys) and confirm normal
  checkout still works
- View a hotel's details as a customer, refresh several times same day
  (view count should not increase), then check the hotel dashboard's
  "Hotel Views" card
- Delete a customer account and a hotel account, confirm login afterward
  fails and existing payment/reservation history is preserved for
  admin/accounting purposes
