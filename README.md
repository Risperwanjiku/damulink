# DamuLink

A mobile app that connects blood donors with the people who need them, built for the Kenyan context.

When someone needs blood, the people most able to help are often nearby strangers who simply never hear about the need in time. DamuLink closes that gap: a requester posts a blood need, and the app automatically notifies compatible, available donors in the same city — without ever exposing the patient's private details to the public.

Built with Flutter and Firebase. Android, feature-complete, and pre-launch, publishing is pending data-protection registration (see Status below).

---

## What it does

**For people who need blood (requesters):**
- Post a blood request with the patient's blood type, the hospital, and an urgency level.
- Choose how much to reveal sensitive details stay private until a donor offers to help.
- See offers come in and contact donors who have stepped forward.

**For donors:**
- Get notified automatically when a compatible request appears in their city no need to go looking.
- Browse open requests that match their blood type.
- Offer to help, which reveals the requester's contact details so they can coordinate directly.

The core idea is that **matching happens automatically and privately**. Donors don't search a public directory of patients, and patients aren't listed anywhere for strangers to browse. The app does the compatibility matching server-side and reaches out only to the people who can actually help.

## Why it's built this way

A blood-donation app handles some of the most sensitive data there is health information, names, and phone numbers of people in a medical emergency. Most of the non-obvious decisions in this project come from taking that seriously:

- **No public donor directory.** Listing donors and their blood types publicly would expose health data to anyone. Instead, donor matching happens server side, and donors are contacted privately. There is no browsable list of people and their blood types.

- **Patient details are split across two documents.** Public, donor safe fields (blood type, hospital, urgency, patient initials) live in one place; identifying details (full name, contact phone) live in a separate, locked down document that's only revealed through the offer to help flow. This split is enforced by security rules, not just by the UI.

- **Notifications carry only public-safe content.** When the app fans out a notification to compatible donors, the message contains initials, hospital, and blood type never the patient's full name or phone number.

These choices are driven by Kenya's Data Protection Act (2019), which treats health data as a special category requiring extra care. The architecture is designed to make leaking that data hard by default.

## Architecture

DamuLink is a Flutter client backed entirely by Firebase, with the privacy-critical logic pushed to the server (Cloud Functions) so it can't be bypassed by a tampered client.

```
┌─────────────────┐         ┌──────────────────────────────┐
│  Flutter app    │         │  Firebase                    │
│  (Android)      │         │                              │
│                 │         │  ┌────────────────────────┐  │
│  Requester ─────┼────────▶│  │ Firestore              │  │
│  posts request  │  write  │  │  blood_requests (public)│  │
│                 │         │  │  blood_request_private  │  │
│  Donor browses ◀┼─────────┤  │  public_profiles        │  │
│  & offers help  │  read   │  │  responses              │  │
│                 │         │  │  notifications          │  │
│                 │         │  └───────────┬────────────┘  │
│                 │         │              │ triggers       │
│  Donor gets ◀───┼─────────┤  ┌───────────▼────────────┐  │
│  push notif     │   FCM   │  │ Cloud Functions (v2)   │  │
│                 │         │  │  fan-out, cascade-delete,│ │
│                 │         │  │  response counting,      │ │
│                 │         │  │  places proxy            │ │
│                 │         │  └────────────────────────┘  │
│                 │         │  Auth · Storage · FCM         │
└─────────────────┘         └──────────────────────────────┘
```

**The flow when a request is posted:**

1. The requester writes a public `blood_requests` document and a matching private `blood_request_private` document in a single atomic batch (so the privacy split can never end up half-written).
2. A Cloud Function (`onBloodRequestCreated`) fires on the new public document.
3. It computes which donor blood types are compatible with the patient, then queries for available donors in the same city with a matching type.
4. For each eligible donor, it writes an in app notification and sends an FCM push using only public safe content.
5. When a donor offers to help, the requester's private contact details are revealed to that donor through the response flow.

Putting the matching and the cascade cleanup on the server means a modified client can't pull data it shouldn't, and deleting a request reliably cleans up everything attached to it.

## Tech stack

- **Flutter / Dart** - cross platform UI (Android target)
- **Firebase Auth** - email/password authentication
- **Cloud Firestore** - primary datastore, with security rules enforcing the public/private split
- **Cloud Functions (2nd gen, Node.js)** — server side matching, notification fan-out, cascade deletes, response counting, and a Google Places proxy
- **Firebase Cloud Messaging (FCM)** - push notifications to donors
- **Firebase Storage** - profile photos
- **GetX** - state management, routing, and dependency injection
- **Google Places API** - hospital autocomplete (proxied through a Cloud Function so the API key never ships in the app)

## Project structure

```
lib/
  configs/        Theme, design tokens, shared utilities, blood-type compatibility
  services/       Notification service and other app-wide services
  view/
    screen/       App screens (dashboard, requests, donor browse, profile, etc.)
    widgets/      Reusable UI widgets
  main.dart       App entry point
functions/
  index.js        All Cloud Functions
android/          Android platform code
```

## Getting started

> This is a Firebase app, so running it requires your own Firebase project. The repo does not include any Firebase config or API keys you'll generate your own.

### Prerequisites

- [Flutter SDK](https://docs.flutter.dev/get-started/install) (stable channel)
- A [Firebase](https://console.firebase.google.com/) project on the Blaze plan (Cloud Functions require it)
- [Node.js](https://nodejs.org/) (for Cloud Functions)
- The [Firebase CLI](https://firebase.google.com/docs/cli): `npm install -g firebase-tools`

### 1. Clone and install

```bash
git clone https://github.com/Risperwanjiku/damulink.git
cd damulink
flutter pub get
```

### 2. Connect your own Firebase project

Install the FlutterFire CLI and configure the app against your project. This generates `firebase_options.dart` locally (it is not committed):

```bash
dart pub global activate flutterfire_cli
flutterfire configure
```

In the Firebase console, enable: **Authentication** (Email/Password), **Cloud Firestore**, **Storage**, and **Cloud Messaging**.

### 3. Deploy the Cloud Functions

```bash
cd functions
npm install
cd ..
firebase deploy --only functions
```

The hospital autocomplete uses a Google Places API key stored as a Functions secret (not in the code):

```bash
firebase functions:secrets:set GOOGLE_PLACES_API_KEY
```

### 4. Run

```bash
flutter run
```

## Cloud Functions

All functions live in `functions/index.js`:

| Function | Trigger | Purpose |
|---|---|---|
| `onBloodRequestCreated` | New `blood_requests` doc | Match compatible, available donors in the same city and fan out notifications (in-app + FCM) using public-safe content only. |
| `onBloodRequestDeleted` | `blood_requests` doc deleted | Cascade-delete the private companion doc, related responses, and notifications. |
| `onResponseCreated` | New `responses` doc | Atomically increment the request's response count. |
| `onResponseDeleted` | `responses` doc deleted | Atomically decrement the response count (guarding against a deleted parent). |
| `placesAutocomplete` | Callable | Proxy hospital searches to the Google Places API so the key stays server-side; restricted to signed-in users and Kenyan hospitals. |

Notification IDs are deterministic (`requestId_donorUid`) so that function retries overwrite rather than duplicate.

## Data model

| Collection | Holds | Visibility |
|---|---|---|
| `blood_requests` | Public request fields: blood type, hospital, urgency, patient initials, response count | Readable by donors |
| `blood_request_private` | Identifying fields: full name, contact phone | Locked down; revealed through the offer-to-help flow |
| `public_profiles` | Donor-visible fields the matcher queries: blood type, city, availability | Server-queried for matching |
| `users` | Full user profile, FCM token, notification preference | Owner only |
| `responses` | A donor's offer to help on a request | Scoped to the parties involved |
| `notifications` | Per-donor in-app notifications | Recipient only |

Blood type compatibility follows standard medical rules (O− universal donor, AB+ universal recipient, Rh− recipients receive only Rh− blood) and is kept in sync between the client and the server-side matcher.

## Status

DamuLink is **feature-complete and working**, but **not yet published** to the Play Store. Because it handles health data, launching responsibly in Kenya means completing data-protection steps first — registration with the Office of the Data Protection Commissioner (ODPC) and a published privacy policy — before it goes live to real users.
