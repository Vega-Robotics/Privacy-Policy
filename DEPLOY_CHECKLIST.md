# Deploy checklist

## 1. Backend — fixes the OTP email issue
The code was already correct; emails were only ever going out if these were set.
On your deployed server's environment (not in this zip — set these where the
backend actually runs: Render/Railway/EC2/etc.):

```
EMAIL_HOST=smtp.yourprovider.com
EMAIL_PORT=587
EMAIL_USER=your@email.com
EMAIL_PASSWORD=your_app_password   # Gmail: use an App Password, not your login password
EMAIL_FROM=your@email.com
```

Restart the backend after setting these. No code change needed — `server/src/services/otpService.js`
auto-detects these and switches from console-log "mock mode" to real sending.

Phone/SMS OTP is now disabled by request (email-only verification). The Twilio
code is still in `otpService.js`, `otpValidators.js`, `server/package.json`, and
`.env.example` — commented out, not deleted — so it can be switched back on later.

## 2. Mobile — fixes the "red banner" / notification permission log
That banner and log line are Expo Go's own behavior on Android (SDK 53 dropped
push-notification support there) — not a bug, and not fixable from inside Expo Go.
It disappears once you run the app as a real build instead of Expo Go.

This zip now has the mobile project's missing scaffolding
(`package.json`, `app.json`, `eas.json`, `babel.config.js`, `index.js`) added, since
they weren't in the original upload. Before building:

1. `cd mobile && npm install`
2. Copy `.env.example` to `.env` and set `EXPO_PUBLIC_API_URL` to your **deployed**
   backend's HTTPS URL (not localhost).
3. In `app.json`, change `"com.messclub.app"` (both `ios.bundleIdentifier` and
   `android.package`) to your own reverse-domain package name if you don't
   already own that one.
4. `npx eas login` (create a free Expo account if you don't have one), then
   `npx eas init` — this fills in the real `extra.eas.projectId` in `app.json`
   for you.
5. Build the production Android App Bundle:
   ```
   eas build --platform android --profile production
   ```
   This runs on Expo's build servers and requires network access — it can't be
   run inside this environment. It generates and manages the signing keystore
   for you on first run.

I couldn't run `eas build` myself or hand you a compiled `.aab` — it needs your
own Expo account and (for Play Store submission) your own Google Play signing
identity, neither of which exist in this sandbox. Everything else needed to make
that command work on the first try is now in this zip.
