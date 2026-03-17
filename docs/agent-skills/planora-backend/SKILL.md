---
name: planora-backend
description: Backend architecture, authentication, cloud sync, and data persistence for the Planora application using Firebase (free tier). Use this whenever adding server-side capabilities, implementing user authentication, building the sync engine between IndexedDB and Firestore, writing Firestore security rules, managing environment configuration, or making any change that involves cloud data, user accounts, or client-server communication — even small API additions. Also use when reviewing backend code for free-tier compliance, sync correctness, or security.
---

# Planora Backend

Planora is transitioning from a purely offline single-file app to an **offline-first app with cloud sync**. The backend uses **Firebase** (Spark free tier) as a Backend-as-a-Service — there is no custom server. All backend interaction happens through Firebase client SDKs loaded in the browser.

This skill covers how to architect, implement, and maintain the backend layer while preserving Planora's offline-first DNA.

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  Browser (GitHub Pages)                             │
│  ┌───────────────┐    ┌──────────────────────────┐  │
│  │  Planora UI   │◄──►│  Sync Engine             │  │
│  │  (vanilla JS) │    │  (conflict resolution,   │  │
│  └──────┬────────┘    │   offline queue,          │  │
│         │             │   merge logic)            │  │
│         ▼             └──────┬───────────────────┘  │
│  ┌───────────────┐           │                      │
│  │  IndexedDB    │◄──────────┘                      │
│  │  (primary)    │           │                      │
│  └───────────────┘           │                      │
└──────────────────────────────┼──────────────────────┘
                               │ Firebase JS SDK
                               ▼
┌──────────────────────────────────────────────────────┐
│  Firebase (Spark Free Tier)                          │
│  ┌────────────┐  ┌──────────────┐                    │
│  │  Auth       │  │  Firestore   │                    │
│  │  (Google    │  │  (1 GiB,     │                    │
│  │   sign-in)  │  │   50K R/day, │                    │
│  └────────────┘  │   20K W/day)  │                    │
│                  └──────────────┘                    │
└──────────────────────────────────────────────────────┘
```

### Core Principles

1. **IndexedDB is always the source of truth on-device.** The app works fully offline. Firestore is a sync target, not a dependency. If Firebase is unreachable, the app continues normally — sync resumes when connectivity returns.

2. **Eventual consistency, not real-time.** Planora is a personal productivity app, not a collaborative editor. Sync doesn't need to be instant — it needs to be correct. A few seconds of delay is fine; data loss is not.

3. **Respect the free tier.** Every design decision must account for Spark plan limits. Batch writes, minimize reads, avoid unnecessary listeners. Read `references/firebase-setup.md` for exact quotas.

4. **Authentication is optional.** Users who don't sign in keep the current fully-offline experience. Signing in unlocks cloud sync. The app never forces account creation.

5. **Preserve the single-file architecture.** Firebase SDK is loaded from CDN (like the existing QR/LZString libs). Backend logic lives in the second `<script>` block alongside existing app code. No build step required.

6. **Security rules are the only backend "code."** With no custom server, Firestore Security Rules are the entire authorization layer. They must be airtight — every document must be owned by the authenticated user.

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Hosting | GitHub Pages | Already set up, free, serves static HTML |
| Auth | Firebase Authentication | Free for all sign-in methods, Google sign-in integrates with Workspace |
| Database | Cloud Firestore | Free tier (1 GiB), offline support built into SDK, real-time listeners |
| Client SDK | Firebase JS SDK (modular v9+) | Tree-shakeable, loaded from CDN via ES module or compat bundle |
| Local storage | IndexedDB (`tz_planner_idb`) | Existing persistence layer — stays as primary |

### What's NOT in the stack

- **No custom server.** No Express, no Fastify, no Cloud Functions. Everything runs client-side or through Firebase's managed services.
- **No Firebase Hosting.** GitHub Pages remains the host. Firebase is used only for Auth + Firestore.
- **No Realtime Database.** Firestore is the better fit — richer queries, offline persistence, more generous free tier.

## Authentication

### Provider: Google Sign-In

Google sign-in is the primary (and initially only) auth method. It's the natural choice because:
- The user has Google Workspace Business Standard
- Zero password management burden
- Firebase Auth's Google provider is free with no user limits for the Spark plan
- `signInWithPopup` works on GitHub Pages without special configuration

### Google Workspace Domain Restriction (Optional)

Since the user has Google Workspace Business Standard, sign-in can optionally be restricted to the Workspace domain only. This prevents random Google accounts from creating Firestore data under your project:

```js
const provider = new firebase.auth.GoogleAuthProvider();
provider.setCustomParameters({ hd: 'your-domain.com' });  // Restrict to Workspace domain
```

This is a UI hint — it tells Google's consent screen to pre-select the Workspace account. For true enforcement, add a Firestore security rule check: `request.auth.token.email.matches('.*@your-domain\\.com')`. Skip this if you want any Google account to sign in.

### Auth Flow

```
User taps "Sign in" → signInWithPopup(GoogleAuthProvider)
  → Success: store user.uid, enable sync UI, trigger initial sync
  → Popup blocked (mobile): fall back to signInWithRedirect(provider)
  → Failure: show toast, continue in offline mode

User taps "Sign out" → signOut()
  → Clear auth state, disable sync UI, app continues offline
```

### signInWithRedirect Fallback

Mobile browsers and strict popup blockers will prevent `signInWithPopup`. Always implement `signInWithRedirect` as a fallback — try popup first, catch `auth/popup-blocked`, then redirect. On page load, call `getRedirectResult()` to handle the return. See `references/firebase-setup.md` for the full implementation.

### Auth State Persistence

Firebase Auth defaults to `browserLocalPersistence` — the user stays signed in across browser sessions. On page load:

```js
onAuthStateChanged(auth, (user) => {
  if (user) {
    // User is signed in — enable sync
    initSyncEngine(user.uid);
  } else {
    // Not signed in — pure offline mode
    disableSyncEngine();
  }
});
```

### Auth in the UI

- Add a "Sign in with Google" button in Settings (not the welcome gate — keep that for audio activation)
- Show user avatar/email in settings when signed in
- Show sync status indicator (synced / syncing / offline / error)
- "Sign out" option in settings
- All auth UI follows the existing frontend design skill (glassmorphism, accent colors, bottom sheets)

### Security Considerations

- Never store the Firebase ID token in localStorage or IndexedDB — Firebase SDK manages token lifecycle automatically
- Use `getIdToken(user, true)` only when forced refresh is needed
- The `user.uid` is the key that ties all Firestore data to the user — treat it as the partition key
- See the security skill for client-side auth handling best practices

## Data Layer: Firestore Schema

The existing IDB structure maps to Firestore as **one document per IDB key, nested under the user's UID.** This keeps the model simple and minimizes read/write operations.

```
users/{uid}/
├── state/main          ← mirrors tz_main_state
├── state/alerts        ← mirrors tz_alert_state
├── state/notifications ← mirrors tz_notifications
└── meta/sync           ← sync metadata (timestamps, version)
```

Read `references/firestore-schema.md` for the complete document structure, field mappings, and index requirements.

### Why Documents, Not Collections-per-Entity?

Planora's data is relatively small (a single user's cards, events, expenses). Storing each IDB key as one Firestore document means:
- **1 read to get all state** (vs. N reads for N events in a subcollection)
- **1 write to save all state** (vs. N writes after batch editing)
- **Simpler security rules** (one ownership check per document)
- **Fits comfortably in Firestore's 1 MiB document limit** for typical usage

If a user's data grows beyond ~800 KiB in a single document, the sync engine should split it into subcollections. This is a future optimization — the current architecture handles it through size monitoring. See `references/firestore-schema.md` for the growth strategy.

## Sync Engine

The sync engine is the most complex piece. It reconciles IndexedDB (the local truth) with Firestore (the cloud copy) using timestamp-based versioning.

### Sync Triggers

| Trigger | Action |
|---------|--------|
| App load (user signed in) | Pull from Firestore, merge with local, push if local is newer |
| State saved to IDB | Mark as dirty, schedule push (debounced 3–5s) |
| Coming back online | Push any dirty state, pull latest from Firestore |
| Manual "Sync Now" button | Force bi-directional sync |
| Periodic background | Every 5 minutes if signed in and online |

### Conflict Resolution: Last-Write-Wins with Field-Level Merge

For most fields, **last-write-wins (LWW)** by `updatedAt` timestamp is sufficient. For array fields (cards, events, expenses), use **field-level merge**:

1. Compare local and remote arrays by item `id`
2. For items that exist in both: keep the one with the later `updatedAt`
3. For items only in local: add to merged result
4. For items only in remote: add to merged result
5. For items deleted locally (tracked in a deletion log): remove from merged result

Read `references/sync-engine.md` for the full algorithm, edge cases, deletion tracking, and the offline queue implementation.

### Sync Status

Track and display sync state in the UI:

```js
const SYNC_STATES = {
  IDLE: 'synced',        // All changes pushed, up to date
  SYNCING: 'syncing',    // Active sync in progress
  OFFLINE: 'offline',    // No network, changes queued
  ERROR: 'error',        // Sync failed (show retry)
  DISABLED: 'disabled'   // User not signed in
};
```

## Security Rules

Firestore Security Rules are the authorization layer. Every rule enforces:
1. **Authentication** — only signed-in users can read/write
2. **Ownership** — users can only access their own `users/{uid}/` subtree
3. **Validation** — document structure and field types are validated

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }
  }
}
```

This baseline rule is simple but effective — it makes it structurally impossible for one user to read or modify another user's data. Read `references/security-rules.md` for field-level validation rules, rate limiting patterns, and testing.

## Error Handling

### Network Failures

Firebase SDK handles temporary network failures gracefully — writes queue locally and retry automatically. The sync engine should:
- Catch `FirebaseError` with code `unavailable` or `deadline-exceeded`
- Update sync status to `OFFLINE`
- Queue changes for retry (the SDK handles this for Firestore writes, but the sync engine should track pending state)
- Never show raw Firebase errors to the user — translate to friendly messages

### Auth Errors

| Error Code | User Message | Action |
|-----------|-------------|--------|
| `auth/popup-closed-by-user` | (none — silent) | User cancelled, no action |
| `auth/network-request-failed` | "Can't reach Google — check your connection" | Retry button |
| `auth/popup-blocked` | "Pop-up blocked — please allow pop-ups for this site" | Instruction toast |
| `auth/account-exists-with-different-credential` | "This email is linked to another sign-in method" | Guide to correct provider |

### Quota Errors

If nearing free-tier limits:
- `resource-exhausted`: "Sync paused — daily limit reached. Your data is safe locally."
- Log quota warnings to debug console
- Reduce sync frequency dynamically when quota is low

### General Error Pattern

```js
try {
  await syncToFirestore(data);
} catch (err) {
  if (err.code === 'unavailable') {
    setSyncStatus('offline');
  } else if (err.code === 'resource-exhausted') {
    setSyncStatus('error');
    showToast('Sync paused — daily limit reached', 'info');
  } else {
    console.error('[Sync]', err.code, err.message);
    setSyncStatus('error');
    showToast('Sync failed — your data is safe locally', 'error');
  }
}
```

## Free Tier Constraints

The Spark plan is generous for a personal app but has hard limits. Design every feature with these in mind:

| Resource | Free Limit | Planora Budget |
|----------|-----------|---------------|
| Firestore storage | 1 GiB | ~950 MiB (leave headroom) |
| Firestore reads | 50,000/day | ~40,000 (leave headroom) |
| Firestore writes | 20,000/day | ~15,000 (leave headroom) |
| Firestore deletes | 20,000/day | ~15,000 (leave headroom) |
| Auth users | No limit on Spark | N/A |
| Bandwidth | 10 GiB/month | Monitor monthly |

### Strategies to Stay Under Limits

1. **Batch state into single documents** — 1 read/write per sync instead of per-entity
2. **Debounce writes** — Don't write on every keystroke; batch saves with 3–5s debounce
3. **Avoid real-time listeners for sync** — Use `getDoc()` for pull-based sync instead of `onSnapshot()`. Listeners generate reads on every remote change, which burns quota fast
4. **Cache read timestamps** — Don't re-read from Firestore if local data hasn't changed
5. **Track operations** — Maintain a daily counter in memory; reduce sync frequency when approaching limits
6. **Compress data** — Use LZString (already in the app) to compress large state before storing in Firestore. Reduces storage usage and bandwidth

## Quick Decision Guides

### Adding a New Feature with Backend Involvement

- **Data must survive device loss?** → Store in Firestore via the sync engine. Otherwise IDB only.
- **Needs real-time collaboration?** → No (personal app) → Pull-based sync is fine.
- **Involves sensitive data?** → Encrypt client-side before Firestore. See security skill.

### What Gets Synced vs. What Stays Local

| Data | Storage | Synced? |
|------|---------|---------|
| Cards, events, expenses, budgets, categories, alert templates | IDB + Firestore | ✅ Yes |
| Calendar settings, event settings, wallet currencies | IDB + Firestore | ✅ Yes |
| Notification history | IDB + Firestore | ✅ Yes |
| Theme preference, welcome gate skip, debug secret | localStorage | ❌ No (device-specific) |
| firedAlerts, snoozed alerts | Memory only | ❌ No (session-scoped) |
| Virtual recurring entries (`_isGenerated: true`) | Runtime only | ❌ Never write to Firestore |

### Handling a Sync Conflict

```
Two devices edited the same card's label?
  → Last-write-wins by updatedAt timestamp

Device A added an event, Device B added a different event?
  → Both events kept (union merge on arrays by id)

Device A deleted a card, Device B edited the same card?
  → Delete wins (deletion log has explicit entry with timestamp)
  → The deletion log prevents "zombie" items from reappearing

Same expense edited on two devices?
  → Last-write-wins by updatedAt
  → Log the conflict in debug console for transparency
```

## Import/Export & QR Sync Interaction

The existing import/export and QR sync features must be updated to work with the Firebase backend.

### File Import → Trigger Push

When data is imported from a JSON file (`importDataFromFile()`), after writing to IDB:
1. Mark all sync dirty flags as true (`main`, `alerts`, `notifications`)
2. If the user is signed in, trigger an immediate push to Firestore (skip debounce)
3. The push overwrites Firestore data entirely — imported data is the new source of truth
4. If not signed in, dirty flags are set so the next sign-in will push

### File Export → Include Sync Metadata

When exporting via `exportAllData()`, include sync-related fields:
- `_schemaVersion`, `_lastModified`, `_deletedItems` — include in export so reimports preserve sync state
- Do NOT include `meta/sync` metadata (device IDs, push timestamps) — that's device-specific

### QR Import → Same as File Import

QR scan import follows the same pattern as file import: write to IDB, mark dirty, trigger push if signed in.

### QR Export → Same as File Export

QR export uses the same data as file export. Sync metadata is included.

### Key Rule

After any import (file, QR, or future methods), always call:
```js
syncEngine.dirty.main = true;
syncEngine.dirty.alerts = true;
syncEngine.dirty.notifications = true;
if (syncEngine.uid && navigator.onLine) {
  pushToFirestore();  // Immediate push, no debounce
}
```

## Account Deletion & Data Cleanup

If a user wants to delete their account and all cloud data:

1. Delete all Firestore documents under `users/{uid}/` (state/main, state/alerts, state/notifications, meta/sync) using a batch delete
2. Optionally call `firebase.auth().currentUser.delete()` to remove the Auth account
3. Local IDB data remains intact — the user keeps their offline data

**Note:** Firestore on Spark plan doesn't have Cloud Functions for automatic cleanup on user deletion. If the user deletes their Google account externally, their Firestore data remains orphaned but inaccessible (security rules deny access since the auth token no longer exists).

## Migration Strategy

Adding the backend to Planora is incremental — no big bang rewrite.

### Phase 1: Firebase Setup & Auth
1. Add Firebase SDK via CDN (compat bundle for simplicity with the single-file architecture)
2. Initialize Firebase app with project config
3. Add Google sign-in button in Settings
4. Wire up `onAuthStateChanged` for auth state management
5. **Test:** Sign in works, auth state persists across reloads

### Phase 2: Basic Sync (Push Only)
1. After every IDB save, push state to Firestore (debounced)
2. Implement sync status indicator in UI
3. Deploy Firestore security rules
4. **Test:** Data appears in Firestore after local save

### Phase 3: Full Bidirectional Sync
1. Implement pull-on-load (merge Firestore state with local state)
2. Add conflict resolution logic
3. Implement deletion tracking
4. Add periodic background sync
5. Add "Sync Now" manual button
6. **Test:** Data syncs correctly between two browser windows/devices

### Phase 4: Polish & Edge Cases
1. Handle quota exhaustion gracefully
2. Add data size monitoring
3. Implement document splitting (if data exceeds ~800 KiB)
4. Add sync history/log in debug console
5. **Test:** Offline edits sync correctly when coming back online

## Testing

### Firebase Local Emulator Suite

Use the Firebase Emulator for local testing — it simulates Auth and Firestore without hitting production:

```bash
npm install -g firebase-tools   # One-time setup
firebase emulators:start --only auth,firestore
```

Connect the app to emulators during development:

```js
if (location.hostname === 'localhost') {
  connectAuthEmulator(auth, 'http://localhost:9099');
  connectFirestoreEmulator(db, 'localhost', 8080);
}
```

### What to Test

1. **Auth flow** — Sign in, sign out, page reload persistence, popup blocked handling
2. **Sync correctness** — Edit on device A, verify it appears on device B after sync
3. **Conflict resolution** — Edit same item on two devices offline, bring both online
4. **Offline resilience** — Edit while offline, verify sync after reconnect
5. **Deletion sync** — Delete on one device, verify removed on the other
6. **Quota handling** — Simulate `resource-exhausted` error, verify graceful degradation
7. **Large data** — Test with 500+ events and 1000+ expenses to verify document size
8. **Security rules** — Use the Firebase Rules Playground to test that users can't access others' data

## Common Pitfalls

1. **Using `onSnapshot` for everything.** Real-time listeners generate a read for every remote change. For sync, use `getDoc()` on a schedule instead — it's far more quota-friendly for a personal app.

2. **Storing Firebase config as a "secret."** Firebase client config (apiKey, projectId, etc.) is designed to be public — it's embedded in the HTML. Security comes from Firestore Security Rules, not hidden config. Don't waste time trying to hide it.

3. **Writing to Firestore on every keystroke.** Debounce writes with a 3–5 second delay. The user edits a card label, waits, then the sync engine batches and pushes.

4. **Ignoring the 1 MiB document limit.** A user with 500 expenses at ~500 bytes each = ~250 KiB. Safe today, but monitor document size and implement splitting before it becomes a problem.

5. **Not tracking deletions.** If Device A deletes a card and Device B syncs, the card reappears because it's still in B's data. Maintain a `_deletedItems` log with timestamps so deletions propagate correctly.

6. **Forcing authentication.** Planora's identity is offline-first. Users who don't want cloud sync should have the exact same experience as today. Never gate features behind sign-in unless they inherently require it.

7. **Forgetting Firebase SDK adds to page load.** The Firebase JS SDK is ~50-80 KiB gzipped. Load it asynchronously so it doesn't block the existing app initialization. The app should be usable before Firebase finishes loading.

8. **Not handling `auth/popup-blocked`.** Mobile browsers and strict popup blockers will prevent `signInWithPopup`. Implement `signInWithRedirect` as a fallback.

9. **Syncing firedAlerts and snoozed data.** These are memory-only by design (session-scoped). Syncing them to Firestore would cause ghost alerts on other devices. Respect the existing architecture.

10. **Exceeding free tier without monitoring.** Add a simple daily operation counter. When reads hit 40K or writes hit 15K, reduce sync frequency to once per hour instead of every 5 minutes.

11. **Not using the Firebase Emulator for development.** Testing against production Firestore burns your free quota and risks data corruption. Always develop against the local emulator.

12. **Assuming Firestore offline persistence replaces IndexedDB.** Firestore's built-in offline cache is a convenience layer, not a replacement for Planora's IndexedDB persistence. IDB remains the primary store — Firestore is the sync target.

13. **Syncing virtual recurring entries.** The expense tracker generates virtual entries with `_isGenerated: true` and `_parentId` for recurring occurrences. These are computed at runtime and must NEVER be written to Firestore. Only the parent entry (the original `isRecurring: true` expense) is synced. Filter them out before push: `expenses.filter(e => !e._isGenerated)`.

14. **Forgetting to sanitize before Firestore writes.** The security skill defines sanitization pipelines (`sanitizeCard()`, `sanitizeEvent()`, etc.) for IDB writes. Apply the SAME sanitization before pushing to Firestore. Malformed data that passes IDB but fails Firestore validation will silently break sync.

15. **Multi-tab sync race conditions.** If two browser tabs are open and both have sync engines running, they can race — both pushing simultaneously or one overwriting the other's changes. Mitigate by using Firestore's `enablePersistence({ synchronizeTabs: true })` (compat SDK) or `persistentMultipleTabManager()` (modular SDK). Additionally, only one tab should run the periodic sync timer — use `BroadcastChannel` or the `visibilitychange` event to let only the active tab sync.

16. **Not handling theme sync explicitly.** Theme (`timezone_planner_theme_v1`) is in localStorage, not IDB. The export includes theme, but it's a device preference (dark mode on phone, light on desktop). Do NOT sync theme to Firestore — keep it localStorage-only and device-specific. If a user imports data with a different theme, apply it locally but don't push.

17. **Currency API stays client-side.** The Frankfurter API (`api.frankfurter.app`) is fetched directly from the browser. Adding Firebase doesn't change this — there's no server to proxy through. The security skill's CSP `connect-src` already allows this domain. Cache exchange rates in Firestore (via `walletCurrencies.lastFetched`) to reduce redundant API calls across devices.

18. **Using deprecated `enablePersistence()`.** In Firebase JS SDK v10+, `enablePersistence()` is deprecated. Use `initializeFirestore()` with cache settings instead. For the compat SDK (recommended for Planora's no-build setup), `enablePersistence()` still works but watch for deprecation in future versions. If migrating to modular SDK: `initializeFirestore(app, { localCache: persistentLocalCache({ tabManager: persistentMultipleTabManager() }) })`.

## Reference Files

Read these for detailed implementation guidance:

- **`references/firebase-setup.md`** — Step-by-step Firebase project creation, SDK initialization (CDN approach for single-file app), environment configuration for dev/prod, Firebase Emulator setup, and free-tier quota details. Read when setting up Firebase for the first time or configuring environments.

- **`references/firestore-schema.md`** — Complete Firestore document structure mapping every IDB key to Firestore documents, field types, subcollection strategy for large datasets, index requirements, and data size monitoring. Read when designing or modifying the Firestore data model.

- **`references/sync-engine.md`** — Full bidirectional sync algorithm, conflict resolution implementation, deletion tracking, offline queue, debounce strategy, merge functions, sync status management, and edge case handling. Read when implementing or debugging the sync engine.

- **`references/security-rules.md`** — Firestore Security Rules with field-level validation, auth enforcement patterns, rate-limiting strategies, rules testing with the emulator, and deployment workflow. Read when writing or reviewing security rules.
