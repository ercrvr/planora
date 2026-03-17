# Sync Engine

The sync engine is the bridge between IndexedDB (local source of truth) and Firestore (cloud copy). It handles bidirectional synchronization with conflict resolution, offline queuing, and quota-aware operation.

## Table of Contents

1. [Architecture](#1-architecture)
2. [Sync Lifecycle](#2-sync-lifecycle)
3. [Push: Local → Firestore](#3-push-local--firestore)
4. [Pull: Firestore → Local](#4-pull-firestore--local)
5. [Merge Algorithm](#5-merge-algorithm)
6. [Conflict Resolution](#6-conflict-resolution)
7. [Deletion Propagation](#7-deletion-propagation)
8. [Offline Queue](#8-offline-queue)
9. [Debounce and Batching](#9-debounce-and-batching)
10. [Sync Status Management](#10-sync-status-management)
11. [Quota-Aware Sync](#11-quota-aware-sync)
12. [Edge Cases](#12-edge-cases)
13. [Testing the Sync Engine](#13-testing-the-sync-engine)

---

## 1. Architecture

```
┌──────────────────────────────────────────────────┐
│  Sync Engine                                      │
│                                                   │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐ │
│  │ Push       │  │ Pull       │  │ Merge       │ │
│  │ Manager    │  │ Manager    │  │ Logic       │ │
│  └─────┬──────┘  └──────┬─────┘  └──────┬──────┘ │
│        │                │               │         │
│  ┌─────┴────────────────┴───────────────┴──────┐  │
│  │              State Tracker                   │  │
│  │  (dirty flags, timestamps, offline queue)    │  │
│  └──────────────────────────────────────────────┘  │
│        │                            │              │
│        ▼                            ▼              │
│  ┌───────────┐              ┌──────────────┐       │
│  │  IndexedDB │              │  Firestore   │       │
│  │  (local)   │              │  (cloud)     │       │
│  └───────────┘              └──────────────┘       │
└──────────────────────────────────────────────────┘
```

### Core State

```js
const syncEngine = {
  uid: null,                    // Firebase user ID
  status: 'disabled',          // See Sync Status section
  dirty: {                     // Which IDB keys have unsaved changes
    main: false,
    alerts: false,
    notifications: false
  },
  lastPush: {                  // Last successful push timestamps
    main: null,
    alerts: null,
    notifications: null
  },
  lastPull: null,              // Last successful pull timestamp
  pushTimer: null,             // Debounce timer for push
  periodicTimer: null,         // Interval for periodic sync
  offlineQueue: [],            // Changes made while offline
  opsToday: { reads: 0, writes: 0, day: '' }  // Quota tracking
};
```

---

## 2. Sync Lifecycle

### Initialization (on sign-in)

```js
async function initSyncEngine(uid) {
  syncEngine.uid = uid;
  syncEngine.status = 'syncing';

  // 1. Pull latest from Firestore
  await pullFromFirestore();

  // 2. Push any local changes that exist
  if (hasLocalChanges()) {
    await pushToFirestore();
  }

  // 3. Start periodic sync
  syncEngine.periodicTimer = setInterval(periodicSync, 5 * 60 * 1000);  // 5 min

  // 4. Listen for online/offline events
  window.addEventListener('online', onOnline);
  window.addEventListener('offline', onOffline);

  syncEngine.status = 'synced';
  renderSyncStatus();
}
```

### Shutdown (on sign-out)

```js
function disableSyncEngine() {
  syncEngine.uid = null;
  syncEngine.status = 'disabled';
  clearInterval(syncEngine.periodicTimer);
  clearTimeout(syncEngine.pushTimer);
  window.removeEventListener('online', onOnline);
  window.removeEventListener('offline', onOffline);
  renderSyncStatus();
}
```

### Periodic Sync

```js
async function periodicSync() {
  if (!syncEngine.uid || syncEngine.status === 'syncing') return;
  if (!navigator.onLine) return;

  // Check if quota allows
  if (!checkQuota('reads', 3)) return;  // 3 reads for 3 docs

  try {
    syncEngine.status = 'syncing';
    renderSyncStatus();

    await pullFromFirestore();
    if (syncEngine.dirty.main || syncEngine.dirty.alerts || syncEngine.dirty.notifications) {
      await pushToFirestore();
    }

    syncEngine.status = 'synced';
  } catch (err) {
    handleSyncError(err);
  }
  renderSyncStatus();
}
```

---

## 3. Push: Local → Firestore

Push takes the current IDB state and writes it to Firestore.

```js
async function pushToFirestore() {
  if (!syncEngine.uid || !navigator.onLine) {
    syncEngine.offlineQueue.push({ action: 'push', timestamp: Date.now() });
    return;
  }

  const batch = fbDb.batch();
  const userRef = fbDb.collection('users').doc(syncEngine.uid);
  const now = new Date().toISOString();

  if (syncEngine.dirty.main) {
    const state = await __idb.get('tz_main_state');
    if (state) {
      // Filter out virtual recurring entries — only sync parent entries
      if (state.expenses) {
        state.expenses = state.expenses.filter(e => !e._isGenerated);
      }
      state._lastModified = now;
      state._schemaVersion = DATA_SCHEMA_VERSION;
      const mainRef = userRef.collection('state').doc('main');
      batch.set(mainRef, state);
      trackOp('writes');
    }
  }

  if (syncEngine.dirty.alerts) {
    const alertState = await __idb.get('tz_alert_state');
    if (alertState) {
      // Strip session-only fields
      const clean = { ...alertState };
      delete clean.firedAlerts;
      delete clean.snoozed;
      clean._lastModified = now;
      const alertRef = userRef.collection('state').doc('alerts');
      batch.set(alertRef, clean);
      trackOp('writes');
    }
  }

  if (syncEngine.dirty.notifications) {
    const notifications = await __idb.get('tz_notifications');
    const notifRef = userRef.collection('state').doc('notifications');
    batch.set(notifRef, { items: notifications || [], _lastModified: now });
    trackOp('writes');
  }

  // Update sync metadata
  const syncRef = userRef.collection('meta').doc('sync');
  batch.set(syncRef, {
    lastPushAt: now,
    lastPullAt: syncEngine.lastPull || now,
    schemaVersion: DATA_SCHEMA_VERSION,
    deviceId: getDeviceId()
  }, { merge: true });

  await batch.commit();

  // Reset dirty flags
  syncEngine.dirty.main = false;
  syncEngine.dirty.alerts = false;
  syncEngine.dirty.notifications = false;
  syncEngine.lastPush.main = now;
  syncEngine.lastPush.alerts = now;
  syncEngine.lastPush.notifications = now;
}
```

### Marking State as Dirty

Hook into the existing save functions:

```js
// Wrap or extend the existing saveState() function
const _originalSaveState = saveState;
async function saveState() {
  await _originalSaveState.apply(this, arguments);
  syncEngine.dirty.main = true;
  schedulePush();
}

// Similarly for saveAlertState() and saveNotifications()
```

---

## 4. Pull: Firestore → Local

Pull reads Firestore documents and merges them with local state.

```js
async function pullFromFirestore() {
  if (!syncEngine.uid || !navigator.onLine) return;

  const userRef = fbDb.collection('users').doc(syncEngine.uid);

  // Read all state documents
  const [mainSnap, alertSnap, notifSnap] = await Promise.all([
    userRef.collection('state').doc('main').get(),
    userRef.collection('state').doc('alerts').get(),
    userRef.collection('state').doc('notifications').get()
  ]);
  trackOp('reads');
  trackOp('reads');
  trackOp('reads');

  // Merge main state
  if (mainSnap.exists) {
    const remoteMain = mainSnap.data();
    const localMain = await __idb.get('tz_main_state');

    if (localMain) {
      const merged = mergeMainState(localMain, remoteMain);
      await __idb.set('tz_main_state', merged);
      state = merged;  // Update in-memory state
    } else {
      // No local state — use remote as-is
      await __idb.set('tz_main_state', remoteMain);
      state = remoteMain;
    }
  }

  // Merge alert state
  if (alertSnap.exists) {
    const remoteAlerts = alertSnap.data();
    const localAlerts = await __idb.get('tz_alert_state');

    if (localAlerts) {
      const merged = mergeAlertState(localAlerts, remoteAlerts);
      await __idb.set('tz_alert_state', merged);
      alertState = merged;
      // Re-add session-only fields
      alertState.firedAlerts = alertState.firedAlerts || {};
      alertState.snoozed = alertState.snoozed || {};
    } else {
      remoteAlerts.firedAlerts = {};
      remoteAlerts.snoozed = {};
      await __idb.set('tz_alert_state', remoteAlerts);
      alertState = remoteAlerts;
    }
  }

  // Merge notifications
  if (notifSnap.exists) {
    const remoteNotifs = notifSnap.data();
    const localNotifs = await __idb.get('tz_notifications') || [];
    const mergedNotifs = mergeNotifications(localNotifs, remoteNotifs.items || []);
    await __idb.set('tz_notifications', mergedNotifs);
    notifStore = mergedNotifs;
  }

  syncEngine.lastPull = new Date().toISOString();

  // Re-render the UI to reflect merged state
  render();
}
```

---

## 5. Merge Algorithm

### Overview

The merge algorithm handles three types of fields differently:

1. **Scalar fields** (baseZone, settings objects): Last-write-wins by `_lastModified`
2. **Array fields** (cards, events, expenses): Field-level merge by item `id`
3. **Deletion log**: Union merge, then apply deletions

### Main State Merge

```js
function mergeMainState(local, remote) {
  // If one side has no _lastModified, the other wins entirely
  if (!local._lastModified) return remote;
  if (!remote._lastModified) return local;

  const merged = {};

  // ── Scalar fields: LWW ──
  const localNewer = local._lastModified > remote._lastModified;
  merged.baseZone = localNewer ? local.baseZone : remote.baseZone;
  merged.baseDisplay = localNewer ? local.baseDisplay : remote.baseDisplay;

  // ── Settings objects: LWW by their own updatedAt ──
  merged.calendarSettings = mergeByUpdatedAt(
    local.calendarSettings, remote.calendarSettings
  );
  merged.eventSettings = mergeByUpdatedAt(
    local.eventSettings, remote.eventSettings
  );
  merged.walletCurrencies = mergeByUpdatedAt(
    local.walletCurrencies, remote.walletCurrencies
  );

  // ── Merge deletion logs first ──
  const deletions = mergeDeletedItems(
    local._deletedItems || [], remote._deletedItems || []
  );

  // ── Array fields: field-level merge ──
  merged.cards = mergeArrayById(local.cards, remote.cards, deletions, 'cards');
  merged.events = mergeArrayById(local.events, remote.events, deletions, 'events');
  merged.expenses = mergeArrayById(local.expenses, remote.expenses, deletions, 'expenses');
  merged.budgets = mergeArrayById(local.budgets, remote.budgets, deletions, 'budgets');
  merged.expenseCategories = mergeArrayById(
    local.expenseCategories, remote.expenseCategories, deletions, 'expenseCategories'
  );
  merged.incomeCategories = mergeArrayById(
    local.incomeCategories, remote.incomeCategories, deletions, 'incomeCategories'
  );

  // ── Metadata ──
  merged._schemaVersion = Math.max(
    local._schemaVersion || 1, remote._schemaVersion || 1
  );
  merged._lastModified = new Date().toISOString();
  merged._deletedItems = deletions;

  return merged;
}
```

### Array Merge by ID

```js
function mergeArrayById(localArr, remoteArr, deletions, collectionName) {
  localArr = localArr || [];
  remoteArr = remoteArr || [];

  // Build lookup maps
  const localMap = new Map(localArr.map(item => [item.id, item]));
  const remoteMap = new Map(remoteArr.map(item => [item.id, item]));

  // Build set of deleted IDs for this collection
  const deletedIds = new Set(
    deletions
      .filter(d => d.collection === collectionName)
      .map(d => d.id)
  );

  const merged = [];
  const seenIds = new Set();

  // Process all IDs from both sides
  const allIds = new Set([...localMap.keys(), ...remoteMap.keys()]);

  for (const id of allIds) {
    // Skip if deleted
    if (deletedIds.has(id)) continue;

    const localItem = localMap.get(id);
    const remoteItem = remoteMap.get(id);

    if (localItem && remoteItem) {
      // Exists in both — LWW by updatedAt
      const localTime = localItem.updatedAt || '';
      const remoteTime = remoteItem.updatedAt || '';
      merged.push(localTime >= remoteTime ? localItem : remoteItem);
    } else if (localItem) {
      // Only in local — keep (new item created locally)
      merged.push(localItem);
    } else {
      // Only in remote — keep (new item from another device)
      merged.push(remoteItem);
    }

    seenIds.add(id);
  }

  return merged;
}
```

### Settings Merge (LWW by updatedAt)

```js
function mergeByUpdatedAt(local, remote) {
  if (!local && !remote) return null;
  if (!local) return remote;
  if (!remote) return local;

  const localTime = local.updatedAt || '';
  const remoteTime = remote.updatedAt || '';
  return localTime >= remoteTime ? local : remote;
}
```

### Deletion Log Merge

```js
function mergeDeletedItems(localDeletions, remoteDeletions) {
  const map = new Map();

  // Union — keep the latest deletion timestamp per item
  for (const d of [...localDeletions, ...remoteDeletions]) {
    const key = `${d.collection}:${d.id}`;
    const existing = map.get(key);
    if (!existing || d.deletedAt > existing.deletedAt) {
      map.set(key, d);
    }
  }

  // Prune entries older than 30 days
  const cutoff = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();
  const result = [];
  for (const d of map.values()) {
    if (d.deletedAt >= cutoff) result.push(d);
  }

  return result;
}
```

---

## 6. Conflict Resolution

### Strategy: Last-Write-Wins (LWW) with Merge

LWW is simple, predictable, and sufficient for a personal app where the same user owns all devices. The user always knows what they changed last.

### Resolution Rules (priority order)

1. **Deletion vs. Edit:** Compare `deletedAt` and `updatedAt` timestamps — later one wins
2. **Edit vs. Edit (same item):** Later `updatedAt` wins
3. **Addition vs. Addition (different items):** Both kept (union)
4. **Edit vs. No change:** The edit wins (obvious)

### Conflict Logging

Log every conflict resolution in the debug console for transparency:

```js
function logConflict(itemType, itemId, resolution, detail) {
  console.log(`[Sync] Conflict: ${itemType} ${itemId} — ${resolution}`, detail);
}

// Example usage:
logConflict('card', 'card_abc', 'local wins (newer updatedAt)', {
  localUpdatedAt: '2026-03-17T10:00:00Z',
  remoteUpdatedAt: '2026-03-17T09:55:00Z'
});
```

### Clock Skew Tolerance

If two timestamps are within 1 second of each other, treat them as simultaneous. In this case, prefer the local version (the user's current device is the one they're actively using):

```js
function isNewer(a, b) {
  if (!a) return false;
  if (!b) return true;
  const diff = new Date(a).getTime() - new Date(b).getTime();
  if (Math.abs(diff) < 1000) return true;  // Within 1s — prefer first (local)
  return diff > 0;
}
```

---

## 7. Deletion Propagation

### Recording Deletions

When a user deletes an item, add to the deletion log before removing from the array:

```js
function deleteItem(collection, itemId) {
  // 1. Record the deletion
  if (!state._deletedItems) state._deletedItems = [];
  state._deletedItems.push({
    collection: collection,
    id: itemId,
    deletedAt: new Date().toISOString()
  });

  // 2. Remove from the array
  state[collection] = state[collection].filter(item => item.id !== itemId);

  // 3. Save and trigger sync
  saveState();
}
```

### Applying Remote Deletions

During pull/merge, iterate the deletion log and remove matching local items:

```js
function applyDeletions(state, deletions) {
  const collectionNames = ['cards', 'events', 'expenses', 'budgets', 'expenseCategories', 'incomeCategories'];

  for (const collName of collectionNames) {
    const deletedIds = new Set(
      deletions
        .filter(d => d.collection === collName)
        .map(d => d.id)
    );
    if (deletedIds.size > 0 && state[collName]) {
      state[collName] = state[collName].filter(item => !deletedIds.has(item.id));
    }
  }
}
```

### Preventing Zombie Items

A "zombie" is an item that was deleted on one device but reappears after syncing with another device that still has it. The deletion log prevents this by explicitly tracking deletions, so the merge algorithm removes the item from both sides.

---

## 8. Offline Queue

When the device is offline, sync operations are queued and replayed when connectivity returns.

```js
function onOffline() {
  syncEngine.status = 'offline';
  renderSyncStatus();
}

async function onOnline() {
  syncEngine.status = 'syncing';
  renderSyncStatus();

  // Replay queued operations
  try {
    // Simple approach: just do a full bidirectional sync
    await pullFromFirestore();
    await pushToFirestore();
    syncEngine.status = 'synced';
  } catch (err) {
    handleSyncError(err);
  }
  renderSyncStatus();
}
```

### Import-Triggered Sync

When data is imported (file or QR), bypass the normal debounce and push immediately:

```js
async function onDataImported() {
  // Mark everything dirty
  syncEngine.dirty.main = true;
  syncEngine.dirty.alerts = true;
  syncEngine.dirty.notifications = true;

  // Push immediately if signed in
  if (syncEngine.uid && navigator.onLine) {
    syncEngine.status = 'syncing';
    renderSyncStatus();
    try {
      await pushToFirestore();
      syncEngine.status = 'synced';
    } catch (err) {
      handleSyncError(err);
    }
    renderSyncStatus();
  }
}
```

Call `onDataImported()` at the end of `importDataFromFile()` and after QR scan import, before the page reload. The push must complete before the reload — use `await`.

### Why Not a Detailed Queue?

For Planora's use case (single user, document-level sync), a full bidirectional sync on reconnection is simpler and equally effective as replaying individual operations. The merge algorithm handles conflicts correctly regardless of how many offline edits were made.

A detailed operation queue (CRDT-style) would be needed for multi-user collaboration, which isn't in scope.

---

## 9. Debounce and Batching

### Write Debounce

Don't push to Firestore on every save. Debounce with a 3-second delay:

```js
function schedulePush() {
  if (syncEngine.pushTimer) clearTimeout(syncEngine.pushTimer);
  syncEngine.pushTimer = setTimeout(async () => {
    if (syncEngine.uid && navigator.onLine) {
      try {
        syncEngine.status = 'syncing';
        renderSyncStatus();
        await pushToFirestore();
        syncEngine.status = 'synced';
      } catch (err) {
        handleSyncError(err);
      }
      renderSyncStatus();
    }
  }, 3000);  // 3 second debounce
}
```

### Batched Writes

Use Firestore batched writes (`batch.set()` / `batch.commit()`) to write all dirty documents in a single network round-trip. A batch counts as one write per document in the batch, but commits atomically — either all succeed or all fail.

### Read Batching

Use `Promise.all()` to read all state documents in parallel (as shown in the pull function). This reduces latency but still counts as individual reads.

---

## 10. Sync Status Management

### States

```js
const SYNC_STATUS = {
  SYNCED:   'synced',     // ✅ All changes saved to cloud
  SYNCING:  'syncing',    // 🔄 Sync in progress
  OFFLINE:  'offline',    // 📴 No connection, changes saved locally
  ERROR:    'error',      // ❌ Sync failed (with retry)
  DISABLED: 'disabled'    // ⬜ User not signed in
};
```

### UI Indicator

Show a small status icon near the settings gear or in the topbar:

```js
function renderSyncStatus() {
  const el = document.getElementById('syncStatusIndicator');
  if (!el) return;

  const icons = {
    synced:   '☁️',
    syncing:  '🔄',
    offline:  '📴',
    error:    '⚠️',
    disabled: ''     // Hide when disabled
  };

  el.textContent = icons[syncEngine.status] || '';
  el.title = {
    synced:   'All changes synced',
    syncing:  'Syncing...',
    offline:  'Offline — changes saved locally',
    error:    'Sync error — tap to retry',
    disabled: ''
  }[syncEngine.status] || '';

  el.style.display = syncEngine.status === 'disabled' ? 'none' : '';
}
```

### Error Status with Retry

```js
function handleSyncError(err) {
  console.error('[Sync]', err.code || '', err.message);

  if (err.code === 'unavailable' || err.code === 'deadline-exceeded') {
    syncEngine.status = 'offline';
  } else if (err.code === 'resource-exhausted') {
    syncEngine.status = 'error';
    window.__appToast('Sync paused — daily limit reached. Your data is safe locally.', 'info', 6000);
  } else if (err.code === 'permission-denied' || err.code === 'unauthenticated') {
    syncEngine.status = 'error';
    window.__appToast('Please sign in again to sync.', 'error');
    // May need to re-authenticate
  } else {
    syncEngine.status = 'error';
    window.__appToast('Sync failed — your data is safe locally.', 'error');
  }
}
```

---

## 11. Quota-Aware Sync

### Tracking Operations

```js
function checkQuota(type, count) {
  const today = new Date().toDateString();
  if (syncEngine.opsToday.day !== today) {
    syncEngine.opsToday = { reads: 0, writes: 0, day: today };
  }

  const limits = { reads: 40000, writes: 15000 };
  if (syncEngine.opsToday[type] + count > limits[type]) {
    console.warn(`[Quota] Would exceed ${type} limit (${syncEngine.opsToday[type]}/${limits[type]})`);
    return false;
  }
  return true;
}

function trackOp(type) {
  const today = new Date().toDateString();
  if (syncEngine.opsToday.day !== today) {
    syncEngine.opsToday = { reads: 0, writes: 0, day: today };
  }
  syncEngine.opsToday[type]++;
}
```

### Dynamic Frequency Reduction

When approaching quota limits, reduce sync frequency:

```js
function getSyncInterval() {
  const { reads, writes } = syncEngine.opsToday;
  if (reads > 35000 || writes > 12000) return 60 * 60 * 1000;  // 1 hour
  if (reads > 25000 || writes > 8000)  return 30 * 60 * 1000;  // 30 min
  if (reads > 15000 || writes > 5000)  return 15 * 60 * 1000;  // 15 min
  return 5 * 60 * 1000;  // Default: 5 min
}
```

---

## 12. Edge Cases

### First-Time Sync (Empty Firestore)

When `getDoc()` returns `!exists`, the user has never synced before. Push all local data to Firestore. Don't try to merge with nonexistent data.

### New Device (Empty IDB, Data in Firestore)

When local IDB has no `tz_main_state` but Firestore has data, pull everything and write to IDB. This is the "restore from cloud" flow.

### Both Empty

No sync needed. The user is starting fresh.

### Simultaneous Edits on Same Device

Not a sync issue — it's a local state management issue. The existing app already handles this (single-threaded JS, sequential saves).

### Tab Synchronization

Firestore's `enablePersistence({ synchronizeTabs: true })` handles cross-tab sync for the Firestore cache. For IDB, the existing app doesn't handle cross-tab communication — this remains a future enhancement (could use `BroadcastChannel` or `storage` events).

### Large Arrays (500+ items)

The merge algorithm iterates all items in both arrays. For 500 items × 2 sides = 1000 iterations — this is fast (< 10ms). No optimization needed until item counts reach 10,000+.

### Schema Version Mismatch

If a newer version of Planora writes to Firestore with `_schemaVersion: 2` but an older version pulls it:
- The older version should detect the higher schema version
- Run migrations if available
- If the version is too new (no migration path), show a toast: "Please update Planora to sync"
- Continue in offline mode with local data

---

## 13. Testing the Sync Engine

### Unit Tests (in debug console or test script)

```js
// Test mergeArrayById
const local = [
  { id: '1', name: 'A', updatedAt: '2026-03-17T10:00:00Z' },
  { id: '2', name: 'B', updatedAt: '2026-03-17T09:00:00Z' }
];
const remote = [
  { id: '1', name: 'A-remote', updatedAt: '2026-03-17T09:30:00Z' },
  { id: '3', name: 'C', updatedAt: '2026-03-17T08:00:00Z' }
];
const deletions = [];
const result = mergeArrayById(local, remote, deletions, 'test');
// Expected: [{ id: '1', name: 'A' }, { id: '2', name: 'B' }, { id: '3', name: 'C' }]
// id '1': local wins (10:00 > 09:30), id '2': local only, id '3': remote only
```

### Integration Tests (with emulator)

1. **Push-pull roundtrip:** Save locally → push → clear local → pull → verify data matches
2. **Conflict resolution:** Edit same card on two browser windows → sync both → verify LWW
3. **Deletion propagation:** Delete card in window A → sync A → sync B → verify card gone in B
4. **Offline resilience:** Go offline → edit → come back online → verify sync completes
5. **First-time sync:** Sign in with empty Firestore → verify push creates all documents
6. **New device sync:** Clear IDB → sign in with existing Firestore data → verify pull restores all data

### Manual Testing Checklist

- [ ] Sign in with Google on Device/Browser A
- [ ] Create a card and event on A
- [ ] Verify data appears in Firestore (Firebase Console)
- [ ] Sign in on Device/Browser B
- [ ] Verify card and event appear on B
- [ ] Edit the card label on B
- [ ] Wait for sync or press "Sync Now" on A
- [ ] Verify updated label appears on A
- [ ] Delete the event on A
- [ ] Sync B → verify event is gone
- [ ] Turn off network on A, make edits, turn network back on
- [ ] Verify edits sync to Firestore and to B
