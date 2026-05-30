# CLAUDE.md — Meeting Room Booking System

## Project Overview

**会議室予約フォーム** (Meeting Room Booking Form) is a single-file, zero-build web application for managing multi-tenant office conference room reservations with monthly billing tracking.

- **Language**: Japanese UI, JSX/ES6+ JavaScript
- **Deployment**: Drop `index.html` on any static web server
- **Backend**: Firebase Firestore (real-time sync, no custom server)
- **Current state**: `index.html` was deleted in the latest commit (`4f3eed0`). The full source is recoverable from commit `aa4f819`.

---

## Architecture

The entire application is a **single `index.html` file** (~575 lines). There is no build step, no package manager, and no module bundler. All dependencies are loaded from CDN at runtime.

```
index.html
├── <head> — CDN script tags (React, ReactDOM, Babel)
├── <script type="module"> — Firebase initialization (ES module)
│     Exposes: window.__db, window.__fs, window.__firebaseReady
├── <style> — Global CSS reset + scrollbar styles
└── <script type="text/babel"> — Full React application
      ├── Constants & utility functions
      ├── Style constants (JS objects)
      ├── UI components
      └── ReactDOM.createRoot(...).render(<App />)
```

### Firebase ↔ React bridge

Because Firebase uses ES modules and Babel standalone cannot import them, Firebase is initialized in a `type="module"` script tag and then exposed on `window`:

```js
window.__db = db;           // Firestore instance
window.__fs = { collection, getDocs, addDoc, deleteDoc, doc, onSnapshot, setDoc };
window.__firebaseReady = true;
window.dispatchEvent(new Event("firebaseReady"));
```

The React app waits for the `firebaseReady` event before mounting Firestore listeners.

---

## Tech Stack

| Layer | Technology | Version | Source |
|-------|-----------|---------|--------|
| UI framework | React | 18.2.0 | cdnjs |
| DOM rendering | ReactDOM | 18.2.0 | cdnjs |
| JSX transpilation | Babel Standalone | 7.23.2 | cdnjs |
| Database | Firebase Firestore | 10.12.0 | gstatic CDN |
| Styling | Inline CSS-in-JS | — | — |
| Build system | None | — | — |
| Package manager | None | — | — |

---

## Application Constants

All tuneable values are defined at the top of the `<script type="text/babel">` block:

```js
const ROOMS = ["3F商談室", "4F商談室"];  // Conference room names
const OPEN_HOUR = 9;        // Bookable from 09:00
const CLOSE_HOUR = 19;      // Bookable until 19:00
const FREE_HOURS = 40;      // Free hours included per tenant per month
const EXTRA_RATE = 2000;    // Yen per hour over the free quota
const SLOT_COUNT = 30;      // Number of tenant slots
const CANCEL_MINUTES = 20;  // Minutes before start after which cancellation is locked
const OWNER_PASSWORD = "owner1234";  // Plaintext password for owner view
```

---

## Firebase Data Model

### Collection: `bookings`

Each document represents one booking:

```
{
  room:     string,   // e.g. "3F商談室"
  date:     string,   // "YYYY-MM-DD"
  start:    number,   // hour integer, e.g. 10
  end:      number,   // hour integer, e.g. 12
  tenantId: string    // e.g. "t1" … "t30"
}
```

### Collection: `tenants`

Document ID = tenant ID (`t1`…`t30`):

```
{
  name: string   // Display name, editable by owner
}
```

On first load, if the `tenants` collection is empty, the app seeds it with 30 default documents (`テナント 1` … `テナント 30`).

---

## React Components

### Stateless / pure components

| Component | Props | Purpose |
|-----------|-------|---------|
| `Label` | `children` | Bold section heading |
| `SubLabel` | `children` | Smaller sub-heading |
| `Field` | `label, children` | Form field with label |
| `Empty` | `children` | Centered placeholder text |
| `Loading` | — | Full-screen Firebase connecting screen |

### `BookingRow`

Displays one booking entry with conditional delete/cancel button.

- **Tenant view**: shows "キャンセル" if `canCancel(booking)` returns true, "締切" otherwise
- **Owner view**: always shows "削除" regardless of time

### `AvailabilityCalendar`

Month/week availability overview. Local state only (no Firestore writes).

- **Month mode**: 7-column grid, color-coded cells
- **Week mode**: hour×day grid with `○ △ ×` labels
- Colors: green (`#dcfce7`) = empty, yellow (`#fef9c3`) = partial, red (`#fee2e2`) = full

### `OwnerLogin`

Full-screen password prompt. Compares against `OWNER_PASSWORD` constant. On success, calls `onLogin()`. Error state auto-clears after 2 seconds.

### `TenantEditor`

Owner-only. Inline editing of tenant display names with search filter. Highlights tenants exceeding `FREE_HOURS` in red. Saves to Firestore via `renameTenant()`.

### `TenantSummary`

Owner-only. Shows usage bar charts per tenant with overage charge calculation. Filterable by name or "over quota only" toggle.

### `TimeGrid` (inline in `App`)

Today's hour-by-hour grid per room. Defined inside `App` to close over `bookings` and `tenants` state.

### `App`

Root component. Owns all state:

| State | Type | Purpose |
|-------|------|---------|
| `firebaseReady` | bool | Gate until Firebase event fires |
| `tenants` | array | Real-time from Firestore |
| `bookings` | array | Real-time from Firestore |
| `view` | `"tenant"` \| `"owner"` \| `"owner-login"` | Current screen |
| `ownerAuthed` | bool | Owner password validated |
| `currentId` | string | Active tenant ID (`"t1"`…`"t30"`) |
| `form` | object | New booking form values |
| `error` / `success` | string | Inline feedback messages |

---

## Utility Functions

```js
generateId()              // Random 7-char alphanumeric (unused in Firestore path, Firestore auto-IDs)
today()                   // Returns "YYYY-MM-DD" for current date
fmt(h)                    // Formats integer hour → "H:00"
toYMD(date)               // Date object → "YYYY-MM-DD"
addMonths(date, n)        // Returns new Date n months ahead
addWeeks(date, n)         // Returns new Date n weeks ahead
calcUsedHours(bookings, tid)  // Sum of (end-start) for a tenant's bookings
calcExtraCharge(h)        // ¥2,000 × max(0, h - FREE_HOURS)
canCancel(booking)        // Returns false if booking starts within CANCEL_MINUTES
```

---

## Development Workflow

### Running locally

No build step required. Open `index.html` directly in a browser, or serve it:

```bash
# Python
python3 -m http.server 8080

# Node.js
npx serve .

# Or any static file server
```

### Editing the application

All code lives in `index.html`. Edit sections in order:
1. Constants block (top of Babel script)
2. Utility functions
3. Style constants (`card`, `sel`, `inp`, `btnP`)
4. Components (stateless first, then stateful)
5. `App` component

### Making changes to Firebase config

The Firebase project config is hardcoded inside the `type="module"` script tag. To point at a different project, replace the entire `firebaseConfig` object.

### Testing

There is no automated test suite. Test manually by:
1. Opening the app in a browser
2. Verifying the tenant view (booking creation, cancellation, usage display)
3. Verifying the owner view (login, tenant editing, timeline, billing summary)
4. Checking real-time sync by opening two browser tabs

### Deployment

Upload `index.html` to any static host (Firebase Hosting, GitHub Pages, Netlify, Vercel, S3, etc.). No environment variables or server configuration needed.

---

## Key Conventions

- **Inline styles only**: All styling is via JS style objects. No external CSS classes, no Tailwind, no CSS modules.
- **Style constants**: Reusable style objects (`card`, `sel`, `inp`, `btnP`) are defined once at the top of the Babel script and spread where needed.
- **Japanese UI**: All user-facing text is in Japanese. Keep new UI text in Japanese.
- **Firestore via globals**: Never `import` Firebase inside the Babel script. Always access via `window.__db` and `window.__fs`.
- **No ID generation**: Firestore auto-generates document IDs via `addDoc`. The `generateId()` function exists but is not used for Firestore paths.
- **Billing is month-agnostic**: `calcUsedHours` sums all bookings ever, not filtered by month. If monthly reset is needed, it must be added explicitly.
- **Conflict detection is client-side**: Overlap checking happens in `addBooking()` before writing to Firestore. There is no Firestore security rule enforcing this — concurrent submissions could theoretically create overlaps.

---

## Security Notes

These are known issues in the current implementation:

1. **Exposed Firebase config**: The API key and project ID are visible in the HTML source. For Firestore, the API key is not a secret (it identifies the project), but Firestore Security Rules should be configured in the Firebase Console to restrict read/write access appropriately.
2. **Plaintext owner password**: `OWNER_PASSWORD = "owner1234"` is hardcoded. Owner authentication is purely client-side with no server validation.
3. **No tenant authentication**: Any user can select any tenant from the dropdown. There is no login/session for tenants.
4. **Race condition on booking**: Two simultaneous bookings for the same slot will both succeed since conflict detection is not atomic.

---

## Git History Reference

| Commit | Message | Notes |
|--------|---------|-------|
| `d085b0f` | Add files via upload | Original upload with Japanese filename |
| `aa4f819` | Rename file to `index.html` | Complete source available here |
| `4f3eed0` | Delete `index.html` | Current HEAD — working tree is empty |

To recover the full source:
```bash
git show aa4f819:index.html > index.html
```

---

## Branch Convention

Development for this documentation task: `claude/claude-md-docs-aES6u`
