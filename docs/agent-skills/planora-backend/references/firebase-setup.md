# Firebase Setup

Step-by-step guide for setting up Firebase for Planora. This covers project creation, SDK initialization, environment configuration, and the local development workflow.

## Table of Contents

1. [Project Creation](#1-project-creation)
2. [SDK Installation (CDN Approach)](#2-sdk-installation-cdn-approach)
3. [Firebase Initialization in Planora](#3-firebase-initialization-in-planora)
4. [Environment Configuration](#4-environment-configuration)
5. [Firebase Emulator Setup](#5-firebase-emulator-setup)
6. [Free Tier Quotas](#6-free-tier-quotas)
7. [Monitoring Usage](#7-monitoring-usage)

---

## 1. Project Creation

### Steps

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Click "Create a project" (or "Add project")
3. Enter project name: `planora` (or `planora-prod`)
4. **Disable Google Analytics** — not needed, reduces complexity
5. Click "Create project"

### Enable Authentication

1. In Firebase Console → Authentication → Get Started
2. Sign-in providers → Enable **Google**
3. Set the project's public-facing name (shown on consent screen): `Planora`
4. Set the support email to your Google Workspace email
5. Save

### Enable Firestore

1. Firebase Console → Firestore Database → Create Database
2. Choose **Start in production mode** (you'll deploy security rules explicitly)
3. Select a Firestore location close to your primary users:
   - `asia-southeast1` (Singapore) — good for Philippines-based users
   - `us-central1` — good default for global
   - **This cannot be changed later** — choose carefully
4. Click "Enable"

### Register a Web App

1. Firebase Console → Project Settings → General → Your apps
2. Click the web icon (`</>`) to add a web app
3. Nickname: `planora-web`
4. **Do NOT** enable Firebase Hosting (you're using GitHub Pages)
5. Click "Register app"
6. Copy the Firebase config object — you'll need it for initialization

The config looks like this:

```js
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "planora-xxxxx.firebaseapp.com",
  projectId: "planora-xxxxx",
  storageBucket: "planora-xxxxx.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef123456"
};
```

This config is safe to embed in client code — it identifies your project but doesn't grant access. Security is enforced by Firestore Security Rules, not by hiding this config.

### Authorized Domains

Firebase Auth needs to know which domains can trigger sign-in flows:

1. Firebase Console → Authentication → Settings → Authorized domains
2. Verify these are listed (usually auto-added):
   - `localhost`
   - `planora-xxxxx.firebaseapp.com`
3. **Add your GitHub Pages domain:**
   - `your-username.github.io`
   - If using a custom domain, add that too

---

## 2. SDK Installation (CDN Approach)

Planora is a single-file app with no build step. Load the Firebase SDK from CDN, matching the existing pattern of loading libraries via `<script>` tags.

### Using the Compat Bundle

The compat bundle provides the `firebase` global namespace (v8-style API) which works cleanly in a no-build-step environment:

```html
<!-- Add after existing CDN scripts (qrcode, lz-string, html5-qrcode) -->
<script src="https://www.gstatic.com/firebasejs/11.4.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/11.4.0/firebase-auth-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/11.4.0/firebase-firestore-compat.js"></script>
```

Pin to an exact version. When updating, update all three to the same version.

#### SRI Hashes

Generate SRI hashes for each script to comply with the security skill's requirements:

```bash
# Generate SRI hash for each CDN script
curl -s https://www.gstatic.com/firebasejs/11.4.0/firebase-app-compat.js | openssl dgst -sha384 -binary | openssl base64 -A
```

Add the `integrity` and `crossorigin` attributes:

```html
<script src="https://www.gstatic.com/firebasejs/11.4.0/firebase-app-compat.js"
        integrity="sha384-XXXXX"
        crossorigin="anonymous"></script>
```

### Alternative: Modular SDK via ES Modules

If in the future Planora adds a build step or uses `<script type="module">`:

```js
import { initializeApp } from 'https://www.gstatic.com/firebasejs/11.4.0/firebase-app.js';
import { getAuth, signInWithPopup, GoogleAuthProvider, onAuthStateChanged }
  from 'https://www.gstatic.com/firebasejs/11.4.0/firebase-auth.js';
import { getFirestore, doc, getDoc, setDoc }
  from 'https://www.gstatic.com/firebasejs/11.4.0/firebase-firestore.js';
```

The modular SDK is more tree-shakeable but requires either a bundler or native ES module support. Stick with compat for now.

### Load Order

```html
<!-- 1. Existing CDN deps -->
<script src="https://cdn.jsdelivr.net/npm/qrcode@1.4.4/build/qrcode.min.js" ...></script>
<script src="https://cdn.jsdelivr.net/npm/lz-string@1.5.0/libs/lz-string.min.js" ...></script>
<script src="https://cdn.jsdelivr.net/npm/html5-qrcode@2.3.8/html5-qrcode.min.js" ...></script>

<!-- 2. Firebase SDK (load async — non-blocking) -->
<script async src=".../firebase-app-compat.js" ...></script>
<script async src=".../firebase-auth-compat.js" ...></script>
<script async src=".../firebase-firestore-compat.js" ...></script>

<!-- 3. Planora scripts (existing) -->
<script>/* First script block - debug console, toast */</script>
<script>/* Second script block - app logic */</script>
```

When loading Firebase scripts with `async`, check that the SDK is available before initializing:

```js
function waitForFirebase() {
  return new Promise((resolve) => {
    if (typeof firebase !== 'undefined' && firebase.app) {
      resolve();
      return;
    }
    const check = setInterval(() => {
      if (typeof firebase !== 'undefined' && firebase.app) {
        clearInterval(check);
        resolve();
      }
    }, 50);
    // Timeout after 10 seconds — proceed in offline-only mode
    setTimeout(() => { clearInterval(check); resolve(); }, 10000);
  });
}
```

---

## 3. Firebase Initialization in Planora

Add this at the beginning of the second `<script>` block (the async IIFE), after IDB initialization:

```js
// ── Firebase Init ──────────────────────────────
let fbApp = null;
let fbAuth = null;
let fbDb = null;
let fbUser = null;

async function initFirebase() {
  await waitForFirebase();
  if (typeof firebase === 'undefined') {
    console.warn('[Firebase] SDK not loaded — running in offline-only mode');
    return;
  }

  try {
    fbApp = firebase.initializeApp(FIREBASE_CONFIG);
    fbAuth = firebase.auth();
    fbDb = firebase.firestore();

    // Enable Firestore offline persistence (caches data locally)
    // Note: enablePersistence() is deprecated in Firebase JS SDK v10+.
    // For compat SDK (used here), it still works. If migrating to modular SDK later, use:
    //   initializeFirestore(app, { localCache: persistentLocalCache({
    //     tabManager: persistentMultipleTabManager()
    //   }) });
    await fbDb.enablePersistence({ synchronizeTabs: true })
      .catch((err) => {
        if (err.code === 'unimplemented') {
          console.warn('[Firebase] Persistence not supported in this browser');
        }
        // 'failed-precondition' means multiple tabs open — only one can enable persistence
        // This is fine — the other tabs use memory cache
      });

    // Connect to emulator in development
    if (location.hostname === 'localhost' || location.hostname === '127.0.0.1') {
      fbAuth.useEmulator('http://localhost:9099');
      fbDb.useEmulator('localhost', 8080);
      console.log('[Firebase] Connected to local emulators');
    }

    // Check for redirect sign-in result (if signInWithRedirect was used on previous page load)
    fbAuth.getRedirectResult().catch((err) => {
      if (err.code !== 'auth/credential-already-in-use') {
        console.error('[Auth] Redirect result error:', err.message);
      }
    });

    // Listen for auth state changes
    fbAuth.onAuthStateChanged((user) => {
      fbUser = user;
      if (user) {
        console.log('[Auth] Signed in as', user.email);
        initSyncEngine(user.uid);
        renderAuthUI(user);
      } else {
        console.log('[Auth] Not signed in');
        disableSyncEngine();
        renderAuthUI(null);
      }
    });

  } catch (err) {
    console.error('[Firebase] Init failed:', err.message);
    // App continues in offline-only mode
  }
}
```

### Firebase Config Object

Store the config as a constant in the second script block:

```js
const FIREBASE_CONFIG = {
  apiKey: "your-api-key",
  authDomain: "your-project.firebaseapp.com",
  projectId: "your-project-id",
  storageBucket: "your-project.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abcdef"
};
```

As stated in the SKILL.md: this config is intentionally public. It identifies the project. Authorization is handled by Firestore Security Rules + Firebase Auth tokens.

---

## 4. Environment Configuration

### Dev vs. Prod

Since there's no build step, use hostname detection:

```js
const IS_DEV = location.hostname === 'localhost'
            || location.hostname === '127.0.0.1'
            || location.hostname === '';  // file:// protocol

const FIREBASE_CONFIG = IS_DEV
  ? { /* dev project config — or same project, emulator handles it */ }
  : {
      apiKey: "...",
      authDomain: "...",
      projectId: "...",
      storageBucket: "...",
      messagingSenderId: "...",
      appId: "..."
    };
```

For most cases, one Firebase project is enough — use the emulator for development and the real project for production. Two separate Firebase projects are only needed if you want completely isolated data.

### Sign-In with Popup + Redirect Fallback

```js
async function signIn() {
  const provider = new firebase.auth.GoogleAuthProvider();
  // Optional: restrict to Workspace domain
  // provider.setCustomParameters({ hd: 'your-domain.com' });
  try {
    await fbAuth.signInWithPopup(provider);
  } catch (err) {
    if (err.code === 'auth/popup-blocked' || err.code === 'auth/popup-closed-by-user') {
      // Mobile fallback — full page redirect to Google, then back
      await fbAuth.signInWithRedirect(provider);
    } else {
      console.error('[Auth] Sign-in error:', err.code, err.message);
      if (err.code === 'auth/network-request-failed') {
        window.__appToast("Can't reach Google — check your connection", 'error');
      } else if (err.code === 'auth/account-exists-with-different-credential') {
        window.__appToast('This email is linked to another sign-in method', 'error');
      }
    }
  }
}
```

On page load (already in `initFirebase()`), the `getRedirectResult()` check handles the return from redirect sign-in. The `onAuthStateChanged` listener fires automatically for both popup and redirect flows.

### CSP Updates

The security skill requires CSP headers. Add Firebase domains to the CSP policy:

```
script-src 'self' https://www.gstatic.com/firebasejs/ https://apis.google.com;
connect-src 'self' https://firestore.googleapis.com https://identitytoolkit.googleapis.com https://securetoken.googleapis.com https://www.googleapis.com https://api.frankfurter.app;
frame-src https://planora-xxxxx.firebaseapp.com https://accounts.google.com;
```

Update the CSP whenever Firebase SDK or auth domains change.

---

## 5. Firebase Emulator Setup

The Firebase Emulator Suite simulates Auth and Firestore locally. Zero production quota usage, fast iteration, and testable security rules.

### Installation

```bash
# Install Firebase CLI (one-time)
npm install -g firebase-tools

# Login to Firebase
firebase login

# Initialize Firebase in the project root
firebase init
# Select: Firestore, Emulators
# Choose your project
# Accept default Firestore rules file: firestore.rules
# Select emulators: Authentication, Firestore
# Accept default ports (9099 for Auth, 8080 for Firestore)
# Enable emulator UI (usually port 4000)
```

### Running Emulators

```bash
# Start emulators
firebase emulators:start

# Or with data persistence (saves emulator state across restarts)
firebase emulators:start --export-on-exit=./emulator-data --import=./emulator-data
```

### Emulator UI

Access the emulator UI at `http://localhost:4000` to:
- View and edit Firestore documents
- View authenticated users
- Test security rules
- Monitor operations

### Testing Security Rules

The emulator includes a rules playground:

1. Open Emulator UI → Firestore → Rules Playground
2. Set "Authenticated" and enter a UID
3. Test read/write operations against different document paths
4. Verify that users can only access their own data

---

## 6. Free Tier Quotas

### Firebase Spark Plan — Hard Limits

| Resource | Daily Limit | Monthly Limit | Notes |
|----------|------------|---------------|-------|
| Firestore document reads | 50,000/day | ~1.5M/month | Each `getDoc()` = 1 read |
| Firestore document writes | 20,000/day | ~600K/month | Each `setDoc()`/`updateDoc()` = 1 write |
| Firestore document deletes | 20,000/day | ~600K/month | Each `deleteDoc()` = 1 delete |
| Firestore storage | 1 GiB total | — | Includes documents + indexes |
| Firestore bandwidth | 10 GiB/month | — | Data transferred out |
| Auth sign-ins | No limit | — | Google sign-in is free on Spark |
| Auth user accounts | No limit | — | But daily email sends are limited |

### Planora's Expected Usage (Single User)

| Operation | Estimated Daily | Well Under Limit? |
|-----------|----------------|-------------------|
| Sync reads (every 5 min while active) | ~300 reads | ✅ (0.6% of limit) |
| Sync writes (on save, debounced) | ~50 writes | ✅ (0.25% of limit) |
| Auth token refreshes | ~5 reads | ✅ negligible |
| Document size (typical user) | ~50-200 KiB | ✅ (0.02% of 1 GiB) |

Even with heavy usage on multiple devices, a single user won't approach free tier limits. The constraints matter more if the project ever supports multiple users.

---

## 7. Monitoring Usage

### Firebase Console

Firebase Console → Usage and billing → shows daily read/write/delete counts and storage.

### In-App Monitoring

Track operations in memory for awareness:

```js
const _fbOps = { reads: 0, writes: 0, deletes: 0, day: new Date().toDateString() };

function trackOp(type) {
  const today = new Date().toDateString();
  if (_fbOps.day !== today) {
    _fbOps.reads = 0;
    _fbOps.writes = 0;
    _fbOps.deletes = 0;
    _fbOps.day = today;
  }
  _fbOps[type]++;

  // Warn at 80% of safe budget
  if (type === 'reads' && _fbOps.reads >= 32000) {
    console.warn('[Quota] Approaching daily read limit — reducing sync frequency');
  }
  if (type === 'writes' && _fbOps.writes >= 12000) {
    console.warn('[Quota] Approaching daily write limit — reducing sync frequency');
  }
}
```

Show this data in the debug console's Tools tab so you can monitor during development.
