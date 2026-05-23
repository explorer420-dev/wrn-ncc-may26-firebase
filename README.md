# Event Check-in System

A real-time, multi-day event check-in system built with Firebase. Designed for events with pre-registered attendees across multiple sessions, supporting both registered and walk-in flows with live attendance tracking.

## Live Demo

- **Check-in Desk** — `index.html` (desk operator view)
- **Admin Dashboard** — `admin.html` (organiser view)

---

## Features

- **Phone-based lookup** — attendees identified by mobile number; prefetch optimisation loads the record as the 10th digit is typed
- **7 check-in scenarios** — handles registered attendees, walk-ins, multi-day missed sessions, bonus visits, and outside-event-window cases
- **Real-time dashboard** — live attendance log with Firestore snapshot listeners; no refresh needed
- **Role-based access** — separate Firebase Auth credentials for desk operators and admins
- **CSV export** — one-click export of registered attendees or full attendance log
- **Test mode** — day-override flag in Firestore config lets you simulate any event day without changing dates
- **Firestore security rules** — reads require authentication; unauthenticated Apps Script import supported via time-scoped write rules

---

## Architecture

```
Google Sheets  ──(Apps Script import)──▶  Firestore /master
                                                │
                                         index.html (desk)
                                         ├── phone lookup
                                         ├── scenario logic
                                         └── writes /attendance_log

                                         admin.html (admin)
                                         ├── live /attendance_log listener
                                         ├── /master overview
                                         └── CSV export
```

**Hosting:** GitHub Pages (static HTML — no server, no build step)  
**Backend:** Firebase (Auth + Firestore)  
**Import pipeline:** Google Sheets → Google Apps Script → Firestore REST API  

---

## Tech Stack

| Layer | Tool |
|---|---|
| Frontend | Vanilla HTML / CSS / JS |
| Database | Cloud Firestore |
| Auth | Firebase Authentication (Email/Password) |
| Hosting | GitHub Pages |
| Import | Google Apps Script |
| Data source | Google Sheets |

---

## Firestore Data Model

### `master/{phone}`
Attendee registry — written by the morning import, read by the desk app.

| Field | Type | Description |
|---|---|---|
| `name` | string | Attendee full name |
| `registered_days` | number[] | Days the attendee registered for (1–6) |
| `attended_days` | number[] | Days actually checked in (appended on check-in) |
| `sponsored` | boolean | Whether a sponsored kit was collected |

### `attendance_log/{auto-id}`
Append-only audit trail — one document per check-in event.

| Field | Type | Description |
|---|---|---|
| `phone` | string | Attendee phone number |
| `name` | string | Attendee name at time of check-in |
| `day_number` | number | Event day (1–6) |
| `scenario` | string | Check-in scenario code (outcome1–outcome3) |
| `payment_flag` | string | Walk-in payment status note |
| `test_mode` | boolean | True if logged during a test session |
| `timestamp` | timestamp | Server timestamp |

### `config/event`
Single document for runtime configuration.

| Field | Type | Description |
|---|---|---|
| `test_day_override` | number | Force a specific day (1–6); 0 = use real date |

---

## Check-in Scenarios

| Code | Scenario |
|---|---|
| `outcome1` | Registered attendee, checking in on a registered day |
| `outcome2a` | Registered attendee, checking in early (day not yet registered) |
| `outcome2b` | Registered attendee, missed a previous day |
| `outcome2c` | Registered attendee, bonus visit (all registered days done) |
| `outcome3` | Walk-in — not in the registry |

---

## Auth Roles

Two Firebase Auth users, no custom claims needed:

| Role | Credential | Access |
|---|---|---|
| Desk | `desk@<domain>` | Check-in app (`index.html`) |
| Admin | `admin@<domain>` | Dashboard (`admin.html`) |

Firestore rules enforce `request.auth != null` for all reads. Writes to `master` are additionally permitted for unauthenticated Apps Script imports.

---

## Project Structure

```
├── index.html          # Desk check-in app
├── admin.html          # Admin dashboard
├── firestore.rules     # Firestore security rules
├── seed-test-data.js   # Script to populate test data
└── README.md
```

---

## Setup

1. Create a Firebase project, enable **Firestore** and **Email/Password Auth**
2. Create two users: `desk@<domain>` and `admin@<domain>`
3. Deploy `firestore.rules` via Firebase Console or CLI
4. Add Firebase config to both HTML files
5. Host on GitHub Pages (Settings → Pages → Deploy from main branch)
6. Run the Apps Script import to populate `/master` from your Google Sheet

---

## Key Design Decisions

- **No framework** — plain HTML/JS keeps the project zero-dependency and deployable anywhere as a static file
- **Prefetch on 10 digits** — Firestore read fires as the last digit is typed, shaving ~300ms off lookup latency at busy desks
- **Append-only log** — `attendance_log` has no update/delete rules, making it a reliable audit trail
- **Config-driven test mode** — day override in Firestore means testers never need to touch code or wait for a specific date
