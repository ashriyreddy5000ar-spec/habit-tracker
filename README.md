# My Habits · PRO

> A production-grade, multi-user habit tracking SaaS application with real-time Firebase sync, advanced analytics, and a premium dark-mode UI — delivered as a zero-dependency single-file web app.

[![Live Demo](https://img.shields.io/badge/Live%20Demo-GitHub%20Pages-black?style=flat-square&logo=github)](https://ashriyreddy5000ar-spec.github.io/habit-tracker/)
[![Firebase](https://img.shields.io/badge/Backend-Firebase-orange?style=flat-square&logo=firebase)](https://firebase.google.com)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](#15-license)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Features](#2-features-detailed)
3. [System Architecture](#3-system-architecture)
4. [Tech Stack](#4-tech-stack)
5. [Project Structure](#5-project-structure)
6. [Internal Working](#6-internal-working)
7. [Data Architecture](#7-data-architecture)
8. [API Documentation](#8-api-documentation)
9. [Installation & Setup](#9-installation--setup)
10. [Environment Variables](#10-environment-variables)
11. [Build & Deployment](#11-build--deployment)
12. [Scaling & Future Improvements](#12-scaling--future-improvements)
13. [Troubleshooting](#13-troubleshooting)
14. [Contributing Guidelines](#14-contributing-guidelines)
15. [License](#15-license)

---

## 1. Project Overview

### What It Does

**My Habits · PRO** is a personal habit tracking application that lets users build, monitor, and analyse their daily habits across multiple devices. It combines a rich, analytics-driven UI with a real-time cloud backend so that every toggle, addition, or deletion is immediately persisted and available on any device the user logs into.

### Problem It Solves

Most habit trackers are either:
- **Too simple** — no analytics, no streaks, no scoring
- **Too heavy** — native apps requiring installation, sign-up friction, or subscription paywalls
- **Not truly multi-device** — relying on `localStorage` which is browser/device-bound

This app solves all three by being a fully capable web app, accessible from any browser, with real-time cross-device sync powered by Firebase — no installation, no native build pipeline, no per-device data silos.

### Real-World Use Case

A user opens the app on their phone in the morning, checks off "Meditation" and "Drink 3L water", then opens it on their laptop at night to log "Read 10 pages". Both sessions see the same up-to-date state instantly. Their 7-day score, category breakdowns, and streak data are consistent everywhere.

### Target Users

- Individuals building or maintaining personal habits
- People who track health, productivity, mindfulness, and finance goals
- Users who want data-driven insight into their behaviour patterns
- Developers looking for a Firebase + vanilla JS SaaS reference implementation

---

## 2. Features (Detailed)

### 🔐 Authentication System

| Method | Implementation |
|---|---|
| Google OAuth | `signInWithPopup` via `GoogleAuthProvider` — one-tap login with existing Google account |
| Email / Password | `createUserWithEmailAndPassword` / `signInWithEmailAndPassword` — traditional credential flow |
| Session Persistence | Firebase Auth handles JWT token refresh automatically across browser sessions |
| Logout | `signOut()` clears the auth state; `onAuthStateChanged` listener redirects to the login screen immediately |

Internally, `onAuthStateChanged` is the single source of truth for the entire app lifecycle. When it fires with a user object, the app loads that user's Firestore document and mounts the main UI. When it fires with `null`, the login screen is shown and all state is cleared.

### 💾 Per-User Cloud Data

Every user's data is stored in a dedicated Firestore document at `users/{uid}`. This means:
- Complete data isolation between users — no shared collections, no leakage
- New users automatically receive the 14 default habits on first login
- Data is fetched once on login (`getDoc`) and written on every mutation (`setDoc`)

### ✅ Habit Management

- **Add habits** with a name, emoji icon, and category (Health / Mind / Productivity / Finance / Custom)
- **Toggle completion** for the current day — stored as a date-keyed boolean in the habit's `history` map
- **Delete habits** with an animated swipe-out transition
- **Streak calculation** — computed client-side on every render from the `history` map, walking backwards day by day

### 📊 Scoring Engine

The app runs a multi-factor scoring system entirely in the browser:

- **Habit Score (0–100)** — per-habit score combining:
  - 7-day consistency (40% weight)
  - All-time completion rate (30% weight)
  - Streak-to-best-streak ratio (30% weight)

- **User Score (0–100)** — overall user score combining:
  - Average of 7-day daily scores (50% weight)
  - Average of all habit scores (50% weight)

- **Tier Badges**: Starter → Rising → Pro → Elite (mapped to score thresholds 0 / 35 / 60 / 80)

### 📈 Analytics

- **Monthly trend chart** — 30-day canvas line chart showing daily score and completion % (animated on render)
- **Category breakdown** — grid showing today's completion % per category
- **Performance insights** — algorithmically generated text cards identifying best/worst days, hot streaks, momentum changes, and perfect days
- **Streak bar** — visual 14-dot bar showing today's per-habit completion status

### 🔁 Real-Time Sync Indicator

A non-intrusive top bar shows **"Saving…"** (pulsing amber dot) during a Firestore write and **"Synced ✓"** (gold dot) on success, then auto-hides after 1.8 seconds. This gives users confidence their data is persisted without being disruptive.

---

## 3. System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                      │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  Auth Screen │    │   App Shell  │    │  Scoring &   │  │
│  │  (Login UI)  │    │  (Habit UI)  │    │  Analytics   │  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│         │                   │                   │           │
│         └───────────────────┴───────────────────┘           │
│                             │                               │
│                    ┌────────▼────────┐                      │
│                    │  Firebase SDK   │                      │
│                    │  (ES Module CDN)│                      │
│                    └────────┬────────┘                      │
└─────────────────────────────┼───────────────────────────────┘
                              │ HTTPS
              ┌───────────────┼───────────────┐
              │               │               │
     ┌────────▼───────┐  ┌────▼─────┐  ┌─────▼──────┐
     │ Firebase Auth  │  │Firestore │  │  Firebase  │
     │  (OAuth + JWT) │  │   DB     │  │  Hosting   │
     │                │  │ (NoSQL)  │  │ (optional) │
     └────────────────┘  └──────────┘  └────────────┘
```

### Data Flow — Step by Step

```
1. User opens the app
       │
       ▼
2. Firebase SDK initialises → onAuthStateChanged fires
       │
       ├─ No user → Show Auth Screen
       │
       └─ User exists → showApp(user)
              │
              ▼
3. loadUserData(uid)
       │
       ├─ getDoc(users/{uid}) → document exists
       │       └─ Parse habits + history → populate state S
       │
       └─ Document does NOT exist (new user)
               └─ Seed S with 14 default habits → saveToFirestore()
                      │
                      ▼
4. renderAll() → DOM updated with user's habits

5. User toggles a habit
       │
       ▼
6. toggle(id, card)
       ├─ Update S.habits[n].history[today] = true/false
       ├─ recalcStreak(habit) — walk history backwards
       ├─ saveToFirestore() → setDoc(users/{uid}, S)
       │       └─ showSync("saving") → showSync("synced")
       └─ updateAll() → re-render scores, chart, insights

7. User adds a habit
       │
       ▼
8. addHabit()
       ├─ Push new habit object to S.habits
       ├─ saveToFirestore()
       └─ renderAll()

9. User logs out
       │
       ▼
10. signOut() → onAuthStateChanged fires with null
        └─ showAuth() → clear S → return to login screen
```

---

## 4. Tech Stack

### Firebase Authentication
- **Why**: Provides production-grade OAuth (Google) and email/password auth with zero server-side code. Handles JWT issuance, refresh, and session persistence automatically.
- **Role**: Identity layer — every Firestore read/write is gated behind `request.auth.uid` in security rules.
- **Alternative**: Auth0, Supabase Auth, Clerk

### Firebase Firestore
- **Why**: Serverless NoSQL document database with a generous free tier. Supports real-time listeners via `onSnapshot`, offline persistence, and atomic writes.
- **Role**: Single source of truth for all user data. Each user maps to one document containing their full habit and history state.
- **Alternative**: Supabase (PostgreSQL), PlanetScale, MongoDB Atlas

### Vanilla JavaScript (ES Modules)
- **Why**: No build pipeline, no bundler, no framework overhead. The app is a single HTML file — deployable by dropping it anywhere. ES module `import` is used to load Firebase from the CDN.
- **Role**: All UI rendering, state management, scoring logic, and DOM interaction.
- **Alternative**: React, Svelte, Vue (all require a build step)

### HTML5 Canvas
- **Why**: The monthly trend chart requires a custom animated dual-line chart. No chart library dependency is needed — Canvas 2D API is sufficient and keeps the file self-contained.
- **Role**: Renders the 30-day score and completion % line chart with frame-based animation.
- **Alternative**: Chart.js, D3.js (would add ~200KB of dependencies)

### Google Fonts (Syne + Instrument Sans)
- **Why**: Syne is a geometric display font that gives the app a premium, editorial feel for headings and numbers. Instrument Sans provides excellent legibility for body text.
- **Role**: Typography system loaded via CDN.

### GitHub Pages
- **Why**: Free static hosting with HTTPS, zero configuration, and automatic deployment on push to `main`.
- **Role**: Production host for the single HTML file.
- **Alternative**: Vercel, Netlify, Firebase Hosting

---

## 5. Project Structure

```
habit-tracker/
│
├── index.html                  # Entire application — UI, styles, and logic
│
└── README.md                   # Project documentation
```

### Why a Single File?

This is an intentional architectural decision. The app has:
- No build step (no Webpack, Vite, or Rollup)
- No package.json or node_modules
- No server — it's purely static

Firebase is loaded as an ES module from Google's CDN. CSS is inlined in `<style>`. JavaScript runs in a single `<script type="module">` block.

This means the entire application can be:
- Deployed by uploading one file to any static host
- Opened directly from the filesystem (`file://`) for local development
- Forked and customised with a single-file edit

### Internal File Sections (within `index.html`)

```
index.html
│
├── <head>
│   ├── Meta tags (viewport, theme-color, PWA)
│   └── Google Fonts link
│
├── <style>
│   ├── CSS custom properties (:root variables)
│   ├── Auth screen styles
│   ├── App shell & header styles
│   ├── Score hero & progress ring styles
│   ├── Habit card styles
│   ├── Form & FAB styles
│   └── Animation keyframes
│
├── <body>
│   ├── #syncBar          — Save/sync status indicator
│   ├── #authScreen       — Login UI (Google + Email/Password)
│   ├── #appScreen        — Main app (hidden until auth)
│   │   ├── .header       — Greeting, score hero, user chip
│   │   ├── .section ×5   — Progress, Habits, Insights, Scores, Categories, Chart
│   │   └── .add-section  — Habit creation form
│   └── #fabWrap          — Floating action button
│
└── <script type="module">
    ├── Firebase imports (CDN)
    ├── Firebase initialisation
    ├── Constants (EMOJIS, DAYS, CAT_COLOR, CAT_LBL, MOTIVES, DEF habits)
    ├── State (S object — habits[], dh{})
    ├── Date helpers (dk, today, k7, k30)
    ├── Sync UI (showSync)
    ├── Firestore helpers (userDoc, loadUserData, saveToFirestore)
    ├── Auth state handler (onAuthStateChanged)
    ├── Auth event handlers (Google, Email, Toggle, Logout)
    ├── Scoring engine (habitScore, dayScore, userScore, etc.)
    ├── Streak calculator (recalcStreak)
    ├── Toggle handler
    ├── Render functions (renderHabits, updateProgress, updateUserScore, etc.)
    ├── Chart renderer (renderChart — Canvas 2D)
    ├── Habit CRUD (remove, addHabit)
    ├── Form handlers (openForm, closeForm, buildPicker)
    ├── Spark animation
    └── Init (greeting, date chip, motivational line)
```

---

## 6. Internal Working

### State Object

The entire app state lives in a single object `S`:

```js
S = {
  habits: [
    {
      id: 1,
      emoji: "✍️",
      name: "Journal",
      cat: "mind",
      streak: 0,          // computed on render, not stored
      bestStreak: 3,       // stored — persists in Firestore
      history: {
        "2025-03-28": true,
        "2025-03-29": true,
        "2025-03-30": false
      }
    },
    // ...
  ],
  dh: {
    "2025-03-30": { percent: 71, score: 71 }
  }
}
```

`S` is the single source of truth. Every render function reads from `S`. Every mutation writes `S` back to Firestore entirely (full document replace, not partial merge).

### Streak Calculation

`recalcStreak(habit)` walks backwards from today through the `history` map:

```
Today = 2025-03-30

Walk: 2025-03-30 → done? yes (count=1)
      2025-03-29 → done? yes (count=2)
      2025-03-28 → done? no  → STOP

streak = 2
bestStreak = max(bestStreak, 2)
```

If today is not yet completed, the function counts yesterday's consecutive run as the "pending" streak. This avoids resetting the streak display at the start of each new day before the user has had a chance to complete their habits.

### Scoring Engine

```
habitScore(h):
  cons  = count of last 7 days completed / 7        → weight 40%
  cr    = total completed / total logged days        → weight 30%
  ss    = current streak / best streak               → weight 30%
  score = round((cons×0.4 + cr×0.3 + ss×0.3) × 100)

userScore():
  avg7  = average of dayScore() for last 7 days     → weight 50%
  ha    = average habitScore() across all habits     → weight 50%
  score = round(avg7×0.5 + ha×0.5)
```

### Chart Rendering

`renderChart()` uses the HTML5 Canvas 2D API with a frame-based animation loop:

```
1. Size canvas to container width × devicePixelRatio (for retina)
2. Compute 30 daily score values and 30 completion % values
3. Draw grid lines and axis labels
4. Animate both lines from left → right over ~25 frames (prog: 0 → 1)
5. Fill area under each line with 8% opacity of line colour
```

The animation is driven by `requestAnimationFrame` and cancels any in-flight animation before starting a new one (prevents stacked RAF loops on rapid re-renders).

### Firestore Write Strategy

All writes use `setDoc` with full document replacement (no `merge: true`). This is simpler and safer for a document that is always fully loaded before any mutation — there is no risk of partial state. The tradeoff is that writes transfer the entire habits array, which is acceptable given typical habit counts (10–50 habits, < 50KB per document).

### Error Handling

| Layer | Strategy |
|---|---|
| Auth errors | `friendlyError(e)` maps Firebase error codes to human-readable messages displayed in `#authErr` |
| Firestore load | `try/catch` around `getDoc` — silent failure with console log |
| Firestore save | `try/catch` around `setDoc` — sync indicator stays in "saving" state on failure |
| Form validation | Inline border flash on empty name input, 700ms timeout to reset |
| Canvas resize | `window.addEventListener('resize', renderChart)` re-renders chart on layout change |

---

## 7. Data Architecture

### Firestore Collection Structure

```
Firestore
└── users/                          ← Collection
    └── {uid}/                      ← Document (one per user)
        ├── habits: Array           ← All habits
        └── dh: Map                 ← Daily history metadata
```

### Habit Object Schema

```json
{
  "id": 7,
  "emoji": "🏋️",
  "name": "Gym workout",
  "cat": "health",
  "streak": 4,
  "bestStreak": 12,
  "history": {
    "2025-03-27": true,
    "2025-03-28": true,
    "2025-03-29": true,
    "2025-03-30": true
  }
}
```

| Field | Type | Description |
|---|---|---|
| `id` | Number | Auto-incrementing integer, unique within user's habits |
| `emoji` | String | Single emoji character used as visual icon |
| `cat` | String | One of: `health`, `mind`, `productivity`, `finance`, `custom` |
| `streak` | Number | Current consecutive day streak (recomputed on render) |
| `bestStreak` | Number | All-time best streak (persisted) |
| `history` | Map | Date string (`YYYY-MM-DD`) → boolean keys |

### Daily History Object (`dh`)

```json
{
  "2025-03-30": { "percent": 71, "score": 71 },
  "2025-03-29": { "percent": 57, "score": 57 }
}
```

Used as a cached snapshot of daily aggregate scores, enabling faster chart rendering without recomputing from scratch on each load.

### Category Values

```
health        → Health
mind          → Mind
productivity  → Productivity
finance       → Finance
custom        → Custom
```

---

## 8. API Documentation

This app uses **Firebase SDKs directly** — there is no custom REST API. All data operations go through the Firestore SDK and Firebase Auth SDK.

### Auth Operations

| Operation | SDK Call | Description |
|---|---|---|
| Google Sign-In | `signInWithPopup(auth, GoogleAuthProvider)` | Opens OAuth popup |
| Email Sign-Up | `createUserWithEmailAndPassword(auth, email, password)` | Creates new account |
| Email Sign-In | `signInWithEmailAndPassword(auth, email, password)` | Authenticates existing user |
| Sign Out | `signOut(auth)` | Invalidates session |
| Auth State | `onAuthStateChanged(auth, callback)` | Persistent auth listener |

### Firestore Operations

| Operation | SDK Call | Path | Description |
|---|---|---|---|
| Load user data | `getDoc(doc(db, 'users', uid))` | `users/{uid}` | Fetch full user document on login |
| Save user data | `setDoc(doc(db, 'users', uid), S)` | `users/{uid}` | Full document replace on any mutation |
| Realtime listener | `onSnapshot(doc(db, 'users', uid), cb)` | `users/{uid}` | Live updates (connected but not yet used for UI sync) |

### Firestore Security Rules

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null
                         && request.auth.uid == uid;
    }
  }
}
```

This rule ensures:
- Unauthenticated users cannot read or write any data
- Authenticated users can only access their own `users/{uid}` document
- No user can read or modify another user's data

### Example Firestore Document Payload (Write)

```json
{
  "habits": [
    {
      "id": 1,
      "emoji": "✍️",
      "name": "Journal",
      "cat": "mind",
      "streak": 2,
      "bestStreak": 5,
      "history": {
        "2025-03-29": true,
        "2025-03-30": true
      }
    }
  ],
  "dh": {
    "2025-03-30": { "percent": 100, "score": 100 }
  }
}
```

---

## 9. Installation & Setup

### Prerequisites

- A modern browser (Chrome, Firefox, Safari, Edge — ES Modules support required)
- A [Firebase project](https://console.firebase.google.com) with Auth and Firestore enabled
- A static file host (GitHub Pages, Vercel, Netlify, or just a local file server)

### Step 1 — Clone the Repository

```bash
git clone https://github.com/ashriyreddy5000ar-spec/habit-tracker.git
cd habit-tracker
```

### Step 2 — Create a Firebase Project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add project** → name it (e.g. `habits-tracker`)
3. Enable **Google Analytics** (optional)

### Step 3 — Enable Authentication

1. Firebase Console → **Build** → **Authentication**
2. Click **Get started**
3. **Sign-in method** tab:
   - Enable **Google** → set support email → Save
   - Enable **Email/Password** → Save
4. **Settings** tab → **Authorized domains** → Add your domain (e.g. `yourusername.github.io`)

### Step 4 — Create Firestore Database

1. Firebase Console → **Build** → **Firestore Database**
2. Click **Create database**
3. Choose a region close to your users
4. Start in **test mode** (update rules before going to production)

### Step 5 — Set Security Rules

In Firestore → **Rules** tab, replace the default rules:

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null
                         && request.auth.uid == uid;
    }
  }
}
```

Click **Publish**.

### Step 6 — Add Firebase Config to the App

1. Firebase Console → **Project Settings** (gear icon) → **Your apps**
2. Click **Add app** → Web (`</>`) → register it
3. Copy the `firebaseConfig` object
4. Open `index.html` and replace the config block:

```js
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT.firebasestorage.app",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID",
  measurementId: "YOUR_MEASUREMENT_ID"  // optional
};
```

### Step 7 — Run Locally

No build step required. Serve the file with any static server:

```bash
# Python
python3 -m http.server 8080

# Node.js (npx)
npx serve .

# VS Code
# Use the "Live Server" extension
```

Open `http://localhost:8080` in your browser.

---

## 10. Environment Variables

This app does not use `.env` files — it is a static frontend with no server-side code. Firebase configuration is embedded directly in `index.html`.

| Config Key | Description | Where to Find |
|---|---|---|
| `apiKey` | Firebase project API key | Project Settings → Your apps |
| `authDomain` | OAuth redirect domain | Project Settings → Your apps |
| `projectId` | Firestore project identifier | Project Settings → General |
| `storageBucket` | Firebase Storage bucket URL | Project Settings → Your apps |
| `messagingSenderId` | FCM sender ID | Project Settings → Your apps |
| `appId` | Unique Firebase app identifier | Project Settings → Your apps |
| `measurementId` | Google Analytics measurement ID | Optional — Analytics settings |

> **Security note**: The Firebase API key for a web app is safe to expose in client-side code. It is not a secret — access control is enforced entirely through Firebase Security Rules and Auth, not by keeping the API key private.

---

## 11. Build & Deployment

### Local Development

No build step. Edit `index.html` and refresh the browser.

```bash
# Serve locally
python3 -m http.server 8080
```

### Deploy to GitHub Pages

```bash
# Ensure index.html is at the root of your repo
git add index.html README.md
git commit -m "deploy: update app"
git push origin main
```

Then in GitHub → Repository → **Settings** → **Pages**:
- Source: `Deploy from a branch`
- Branch: `main` / `/ (root)`
- Click **Save**

Your app will be live at:
```
https://{your-username}.github.io/{repo-name}/
```

Remember to add this URL to **Firebase Auth → Authorized domains**.

### Deploy to Vercel

```bash
npm i -g vercel
vercel --prod
```

### Deploy to Netlify

Drag and drop `index.html` into [app.netlify.com/drop](https://app.netlify.com/drop).

### Deploy to Firebase Hosting

```bash
npm install -g firebase-tools
firebase login
firebase init hosting
# Set public directory to: . (current folder)
# Configure as single-page app: No
firebase deploy
```

---

## 12. Scaling & Future Improvements

### Current Limitations

| Limitation | Impact |
|---|---|
| Full document write on every mutation | Inefficient for users with large history maps (> 365 days × 50 habits) |
| Streak computed client-side on every render | CPU cost grows linearly with habit count and history depth |
| No offline support | App requires network on first load to fetch data |
| Single HTML file | Hard to unit test individual functions |

### Scaling Improvements

**1. Subcollection per habit**
Move each habit to `users/{uid}/habits/{habitId}` with a subcollection `history/{date}`. This enables:
- Partial writes (only the toggled habit is updated)
- Real-time per-habit listeners
- Efficient range queries on history

**2. Firestore `arrayUnion` / `arrayRemove`**
Replace full `setDoc` with targeted `updateDoc` calls using atomic array operators.

**3. Service Worker + IndexedDB**
Add a service worker for offline support. Cache the app shell and queue Firestore writes when offline, syncing when connectivity is restored.

**4. Modular JS Architecture**
Split into ES modules:
```
src/
├── auth.js         ← Auth logic
├── db.js           ← Firestore operations
├── scoring.js      ← Scoring engine
├── render.js       ← DOM rendering
├── chart.js        ← Canvas chart
└── state.js        ← State management
```

**5. Push Notifications**
Use Firebase Cloud Messaging (FCM) to send daily habit reminders at a user-configured time.

**6. Social Features**
- Public profile with score leaderboards
- Habit challenges between friends
- Shared habit templates

**7. Export**
CSV/PDF export of habit history for personal records.

**8. Progressive Web App (PWA)**
Add `manifest.json` and a service worker to enable:
- Add to Home Screen
- Offline functionality
- App-like fullscreen experience on mobile

---

## 13. Troubleshooting

### `auth/api-key-not-valid`
**Cause**: The `firebaseConfig` in `index.html` still contains placeholder values.
**Fix**: Replace all `"YOUR_*"` values with your actual Firebase project config from the Firebase Console.

---

### `auth/configuration-not-found`
**Cause**: Google Sign-In is not enabled in Firebase Authentication.
**Fix**: Firebase Console → Authentication → Sign-in method → Enable **Google** → set support email → Save.

---

### `auth/unauthorized-domain`
**Cause**: The domain you're opening the app from is not in Firebase's authorized domains list.
**Fix**: Firebase Console → Authentication → Settings → Authorized domains → Add your domain (e.g. `yourusername.github.io`).

---

### Habits not loading after login
**Cause**: Firestore database has not been created, or security rules are blocking access.
**Fix**:
1. Ensure Firestore database is created in Firebase Console
2. Check that security rules allow `read/write` for authenticated users (see [Section 8](#8-api-documentation))

---

### Google sign-in popup closes immediately
**Cause**: Browser is blocking popups, or the app is being opened via `file://` protocol.
**Fix**: Use a local server (`python3 -m http.server`) instead of opening the file directly. Enable popups for `localhost` in browser settings.

---

### Chart not rendering
**Cause**: Canvas element has zero dimensions (usually on first paint before layout is complete).
**Fix**: `renderChart()` checks for zero width/height and returns early. It will re-render on `window.resize`. Trigger manually by resizing the window once, or wrap the call in `requestAnimationFrame`.

---

### Data not syncing across devices
**Cause**: Both devices are logged into different accounts, or the Firestore write failed silently.
**Fix**: Confirm both sessions use the same email/Google account. Check the browser console for Firestore write errors. Verify security rules are correctly published.

---

## 14. Contributing Guidelines

Contributions are welcome. Please follow these steps:

### Getting Started

```bash
git clone https://github.com/ashriyreddy5000ar-spec/habit-tracker.git
cd habit-tracker
```

Add your Firebase config to `index.html` for local testing (do not commit your real API keys — use a separate dev Firebase project).

### Contribution Types

| Type | Description |
|---|---|
| 🐛 Bug fix | Fix a reproducible issue — include steps to reproduce in the PR description |
| ✨ Feature | Discuss in an Issue first before opening a large PR |
| 📝 Docs | README improvements, inline code comments |
| 🎨 UI/UX | Visual improvements — keep the dark aesthetic and CSS variable system |
| ⚡ Performance | Scoring, rendering, or Firestore efficiency improvements |

### Code Style

- Vanilla JS — no frameworks, no build tools
- Keep CSS variables in `:root` — do not hardcode colours
- All DOM IDs use camelCase (`habitList`, `scoreNum`)
- Date keys always in `YYYY-MM-DD` format (ISO 8601)
- Keep the single-file architecture unless proposing a full modular refactor (open an issue first)

### Pull Request Process

1. Fork the repository
2. Create a branch: `git checkout -b feature/your-feature-name`
3. Make your changes
4. Test in Chrome, Firefox, and Safari
5. Submit a PR with a clear description of what changed and why

### Reporting Issues

Open a GitHub Issue with:
- Browser and OS
- Steps to reproduce
- Expected vs actual behaviour
- Console error output (if any)

---

## 15. License

```
MIT License

Copyright (c) 2025 Ashriy Reddy

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

<div align="center">
  <strong>My Habits · PRO</strong> — Built with Firebase, Vanilla JS, and zero dependencies.<br>
  <a href="https://ashriyreddy5000ar-spec.github.io/habit-tracker/">Live Demo</a> ·
  <a href="https://github.com/ashriyreddy5000ar-spec/habit-tracker/issues">Report Bug</a> ·
  <a href="https://github.com/ashriyreddy5000ar-spec/habit-tracker/issues">Request Feature</a>
</div>
