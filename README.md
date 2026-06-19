# Yogini Rakshita — Mobile App

A React Native (Expo) mobile app for Android + iOS. All backend services use **free** tiers:

| Service | Purpose | Free tier |
| --- | --- | --- |
| Firebase Auth | Phone-number OTP login | 10,000 verifications / month |
| Cloud Firestore | Users, payments, reviews, contact | 1 GB storage, 50k reads / 20k writes per day |
| Firebase Storage | UPI payment screenshots | 5 GB |
| Jitsi Meet (`meet.jit.si`) | In-app live video classes | Free, unlimited |
| WhatsApp via `wa.me` | Signup confirmation message | Free deep link (user taps "Send" once) |
| UPI (manual) | Payments | No gateway fees |

## What's inside

```
yoga-app/
├── app/                       # Expo Router screens
│   ├── _layout.tsx            # Auth gate, redirects
│   ├── (auth)/                # login, OTP verify, signup
│   ├── (app)/                 # Home, Online, Demo, Payment, History, Reviews, Contact, Admin
│   └── class/[room].tsx       # Embedded Jitsi class room
├── src/
│   ├── components/            # Screen, Field, PrimaryButton, JitsiVideo
│   ├── context/AuthContext.tsx
│   └── lib/                   # firebase, constants, types, access, whatsapp
├── firestore.rules            # DB security rules
├── storage.rules              # Storage security rules
└── app.json
```

## One-time setup (≈ 30 minutes)

### 1. Install Node + Expo CLI

```bash
brew install node
npm install -g eas-cli
```

### 2. Install dependencies

```bash
cd yoga-app
npm install
npx expo install --check   # auto-aligns versions to your Expo SDK
```

### 3. Create a Firebase project (free)

1. Go to <https://console.firebase.google.com/> and create a project named **Yogini Rakshita**.
2. **Authentication → Sign-in method → Phone**: enable it. Add your own phone as a test number for development.
3. **Firestore Database**: create in production mode, region `asia-south1` (Mumbai).
4. **Storage**: enable, same region.
5. Paste the contents of `firestore.rules` and `storage.rules` into the Rules tabs and publish.

### 4. Add the Android + iOS app to Firebase

- In project settings → **Add app → Android**, use package `com.yoginirakshita.app`. Download `google-services.json` and place it at the project root (same folder as `app.json`).
- **Add app → iOS**, use bundle ID `com.yoginirakshita.app`. Download `GoogleService-Info.plist` and place it at the project root.

> Both files are git-ignored already.

### 5. Fill in your details

Open `src/lib/constants.ts` and replace:

- `ADMIN_PHONES` — your phone number (with `+91`).
- `UPI.vpa` — your UPI ID (e.g. `yogini@oksbi`).
- `OWNER_WHATSAPP` — your number to receive signup notifications, or `null` to disable.
- `CONTACT.email`, `CONTACT.phone`, `CONTACT.address`.

### 6. Build a dev client (required for phone auth + Firebase)

The standard "Expo Go" app **does not** support native Firebase Auth. You need a one-time custom dev build (still free). EAS handles this for you:

```bash
eas login
eas build:configure
# Android dev build (free):
eas build --profile development --platform android
# iOS dev build (needs Apple Developer account, $99/yr to install on device — or use a simulator):
eas build --profile development --platform ios
```

Install the resulting `.apk` (Android) or run via simulator. Then:

```bash
npm start
```

…and scan the QR code from the dev client app. Hot reload works as normal.

### 7. First signup = first admin

1. Open the app, sign up with the phone number you put in `ADMIN_PHONES`.
2. Because that phone matches, your Firestore user doc will be created with `role: "admin"` and `freeAccess: true`. The **Admin** tab will appear.
3. Done — you can now approve payments and grant free access to anyone.

## How features work

### Phone login
- Login screen → enter `+91...` → Firebase sends an SMS via Phone Auth.
- Verify screen → enter 6-digit OTP.
- If no Firestore profile exists yet, the gate routes to **Signup**.

### Signup
- Collects Name, Age, Sex, Address, learning-for (self/kids/family/other) and a switch "this number is on WhatsApp".
- If the switch is on, after the doc is written we open `wa.me/<number>?text=...` so WhatsApp opens with the signup summary pre-filled. The user just taps Send.
- A copy can also be sent to `OWNER_WHATSAPP` if configured.

### Demo classes
- Each non-paid user can join up to `DEMO_LIMIT` (default 3) demo classes — counter lives in `users.demoClassesJoined`.
- Joining atomically increments the count (Firestore `FieldValue.increment(1)`).
- Once at 3, the join button becomes "Buy a plan".

### Online classes
- Gated by `hasPaidAccess(user)` = `role === "admin" || freeAccess === true || accessUntil > now`.
- If a non-paid user opens the Online tab, they're redirected to Demo after a short message.
- Paid users see today's room name and tap "Join class now" → opens the Jitsi WebView **inside** the app. No external app launch.

### Payment flow
- User picks a plan (₹1000 / ₹2700 / ₹5500).
- Tapping "Open my UPI app" launches a `upi://pay` intent (GPay / PhonePe / Paytm picks it up).
- User pays manually, returns to the app, attaches a screenshot, optionally adds the UPI reference, and submits.
- A `payments` doc is created with `status: "pending"`. Admin sees it in the Admin tab.

### Admin tab
- **Payments**: see all pending/approved/rejected payments. Tap **Approve** to extend the user's `accessUntil` by the plan's days (additive — extends past today and any existing expiry). Tap **Reject** to mark rejected.
- **Users**: list of all users with their access status. Two ways to bypass payment:
  1. Toggle **Free access** → flips `freeAccess: true`. The user gets unlimited access immediately, no payment required (great for family).
  2. Tap **+30d / +90d / +180d** → extends their `accessUntil` without any payment record.

### Reviews, Contact
- Reviews are world-readable, posted by signed-in users.
- Contact messages are admin-readable only.

## Running

```bash
npm start              # Metro bundler, scan QR from your dev client
npm run android        # build + run on connected Android
npm run ios            # build + run iOS simulator
```

## Cost reality check

- Backend / video / OTP: **₹0 / month** at this scale.
- Apple Developer Program: **$99 / year** (only if publishing on the App Store).
- Google Play Console: **$25 one-time** (only if publishing on the Play Store).
- WhatsApp Business API (optional, future): free for 1,000 service conversations/month, but requires business verification.

## Known limitations & "phase 2"

- **WhatsApp signup is semi-automatic** — the user taps "Send" inside WhatsApp. To make it fully automatic, switch to WhatsApp Business Cloud API (Meta) — still free up to 1k conversations/month but needs business verification.
- **Payments are admin-approved** — no automatic verification. To auto-verify, integrate Razorpay/Cashfree UPI Collect (2% per transaction).
- **`meet.jit.si`** is a public shared server. Sessions occasionally lag during peak hours. For a paid product, self-host Jitsi on a ₹500/month VM later.
- **Class scheduling** is currently a daily room (`yogini-rakshita-live-YYYY-MM-DD`). To support multiple classes per day with descriptions, add a `classes` collection later.
