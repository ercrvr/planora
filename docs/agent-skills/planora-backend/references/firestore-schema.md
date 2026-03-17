# Firestore Schema

Complete data model for Planora's Firestore backend. Every document maps directly to an existing IndexedDB key, making the sync engine straightforward.

## Table of Contents

1. [Collection Structure](#1-collection-structure)
2. [Document: state/main](#2-document-statemain)
3. [Document: state/alerts](#3-document-statealerts)
4. [Document: state/notifications](#4-document-statenotifications)
5. [Document: meta/sync](#5-document-metasync)
6. [Field-Level updatedAt Tracking](#6-field-level-updatedat-tracking)
7. [Deletion Log](#7-deletion-log)
8. [Document Size Management](#8-document-size-management)
9. [Index Requirements](#9-index-requirements)
10. [Data Migration from IDB](#10-data-migration-from-idb)

---

## 1. Collection Structure

```
firestore-root/
└── users/                              ← top-level collection
    └── {uid}/                          ← user document (may be empty or contain profile)
        ├── state/                      ← subcollection for app state
        │   ├── main                    ← mirrors IDB: tz_main_state
        │   ├── alerts                  ← mirrors IDB: tz_alert_state
        │   └── notifications           ← mirrors IDB: tz_notifications
        └── meta/                       ← subcollection for sync metadata
            └── sync                    ← sync timestamps and version info
```

Each user gets their own subtree under `users/{uid}`. The `{uid}` comes from Firebase Auth (`user.uid`).

### Why Subcollections Instead of Nested Documents?

Firestore charges per document read. Having `state/main`, `state/alerts`, and `state/notifications` as separate documents means the sync engine can read/write only the documents that changed — rather than reading one massive combined document every time.

---

## 2. Document: `state/main`

**Path:** `users/{uid}/state/main`
**Mirrors IDB key:** `tz_main_state`
**Max expected size:** 200–800 KiB (well under 1 MiB limit)

### Fields

```js
{
  // ── Top-level settings ──
  baseZone: string,             // IANA timezone, e.g. "Asia/Manila"
  baseDisplay: string,          // Human-readable display name

  // ── Array data (sync engine uses field-level merge on these) ──
  cards: [{                     // Company/timezone cards
    id: string,
    label: string,
    targetZone: string,
    targetDisplay: string,
    inputMode: 'target' | 'local',
    ignoreDst: boolean,
    colorHex: string,
    startTime: string,          // "HH:MM"
    endTime: string,            // "HH:MM"
    updatedAt: string           // ISO timestamp — ADDED for sync
  }],

  events: [{                    // All user events
    id: string,
    title: string,
    typeId: string,
    ownerCompanyId: string,
    laneCompanyId: string,
    startDate: string,          // "YYYY-MM-DD"
    startTime: string,          // "HH:MM"
    endDate: string,
    endTime: string,
    status: string,
    priority: number,
    recurrence: { mode: string, interval: number },
    notesText: string,
    notes: [{ id: string, text: string, createdAt: string }],
    subtasks: [{ id: string, text: string, done: boolean }],
    updatedAt: string           // ISO timestamp — ADDED for sync
  }],

  expenses: [{                  // All expense/income entries
    id: string,
    type: 'expense' | 'income',
    amount: number,
    currency: string,
    exchangeRate: number,
    companyId: string | null,
    categoryId: string,
    description: string,
    notes: string,
    date: string,               // "YYYY-MM-DD"
    status: 'paid' | 'pending',
    isRecurring: boolean,
    recurFrequency: string | null,
    recurEndDate: string | null,
    isReimbursable: boolean,
    reimbursementStatus: string | null,
    createdAt: string,
    updatedAt: string           // ISO timestamp — already exists
  }],

  budgets: [{
    id: string,
    categoryId: string | null,
    amount: number,
    period: string,
    thresholdPercent: number,
    currency: string,
    updatedAt: string           // ISO timestamp — ADDED for sync
  }],

  // ── Settings objects (sync engine uses LWW on these) ──
  calendarSettings: {
    slotInterval: number,
    weekMode: string,
    weekOrder: [string],
    conflictNotify: boolean,
    todayBorderColor: string,
    todayBorderThickness: number,
    nowLineColor: string,
    nowLineThickness: number,
    conflictIndicator: { style: string, labelText: string, color: string | null, dotSize: number },
    updatedAt: string
  },

  eventSettings: {
    markerStyle: string,
    types: [{ id: string, name: string, markerKind: string, colorHex: string, shape: string, icon: string, notificationsEnabled: boolean, conflictAlertsEnabled: boolean, alertBeforeValue: number, alertBeforeUnit: string, maxAlerts: number }],
    hideCompleted: boolean,
    updatedAt: string
  },

  expenseCategories: [{
    id: string, name: string, emoji: string, enabled: boolean, createdAt: string, updatedAt: string
  }],

  incomeCategories: [{
    id: string, name: string, emoji: string, enabled: boolean, createdAt: string, updatedAt: string
  }],

  walletCurrencies: {
    defaultCurrency: string,
    enabled: [{ code: string, symbol: string, rate: number, mode: string }],
    lastFetched: number | null,
    updatedAt: string
  } | null,

  // ── Sync metadata (within this document) ──
  _schemaVersion: number,       // DATA_SCHEMA_VERSION (currently 1)
  _lastModified: string,        // ISO timestamp of last local modification
  _deletedItems: [{             // Deletion log for conflict resolution
    collection: string,         // 'cards' | 'events' | 'expenses' | 'budgets' | 'expenseCategories' | 'incomeCategories'
    id: string,                 // The deleted item's id
    deletedAt: string           // ISO timestamp
  }]
}
```

### What's NOT in Firestore

These are explicitly excluded from the `state/main` document in Firestore:

1. **Virtual recurring entries** (`_isGenerated: true`): The expense tracker generates virtual entries at runtime for recurring expenses. These have `_isGenerated: true` and `_parentId`. They must NEVER be written to Firestore — only the parent `isRecurring: true` entry is stored. Filter before push: `expenses.filter(e => !e._isGenerated)`.

2. **Theme preference**: Stored in `localStorage` (`timezone_planner_theme_v1`), not IDB. Theme is device-specific (user may prefer dark on phone, light on desktop). Do NOT sync to Firestore.

3. **Welcome gate skip flag**: `localStorage` key `tz_skip_welcome`. Device-specific, not synced.

4. **Debug secret**: `localStorage` key `__debugSecret`. Device-specific, not synced.

### Key Design Decisions

**`updatedAt` on every entity:** The existing IDB model doesn't have `updatedAt` on cards, events, or budgets. Add it. Every time an item is modified locally, set `updatedAt = new Date().toISOString()`. This is the timestamp the sync engine uses for last-write-wins conflict resolution.

**`_deletedItems` array:** When an item is deleted locally, instead of just removing it from the array, also push an entry to `_deletedItems`. The sync engine uses this to propagate deletions to other devices. Prune entries older than 30 days to prevent unbounded growth.

**`_lastModified`:** The outermost timestamp, updated on every save. The sync engine compares local and remote `_lastModified` to decide whether a full merge is needed.

---

## 3. Document: `state/alerts`

**Path:** `users/{uid}/state/alerts`
**Mirrors IDB key:** `tz_alert_state`
**Expected size:** 5–20 KiB

### Fields

```js
{
  enabled: boolean,
  mode: string,                 // 'normal' | 'annoying' | 'minimal' | 'visual'
  userName: string,
  globalEventLeadMinutes: number,
  companyAlerts: {              // Map of cardId → alert config
    [cardId]: {
      enabled: boolean,
      timeRef: 'target' | 'local',
      startLead: number,
      endLead: number,
      templateId: string
    }
  },
  eventTypeLeadTimes: {
    [typeId]: { startLead: number }
  },
  templates: [{                 // Custom templates (built-in are merged on load)
    id: string,
    name: string,
    text: string,
    builtIn: boolean
  }],
  templateAssignments: {
    [key]: string               // templateId assignments
  },
  soundEnabled: boolean,
  soundType: string,
  soundVolume: number,
  vibrationEnabled: boolean,
  ttsEnabled: boolean,
  ttsRate: number,
  expenseAlerts: {
    enabled: boolean,
    recurringDueEnabled: boolean,
    overdueEnabled: boolean,
    budgetThresholdEnabled: boolean,
    budgetExceededEnabled: boolean,
    reimbursementReminderEnabled: boolean,
    reimbursementReminderDays: number
  },

  // Sync metadata
  _lastModified: string
}
```

### What's NOT Synced

Per the spec and security skill:
- `firedAlerts` — memory-only, session-scoped. Never persisted to IDB or Firestore.
- `snoozed` — memory-only, session-scoped. Never persisted to IDB or Firestore.

These are explicitly stripped from the save payload in `saveAlertState()`. The sync engine must respect this — syncing them would cause duplicate alerts on other devices.

---

## 4. Document: `state/notifications`

**Path:** `users/{uid}/state/notifications`
**Mirrors IDB key:** `tz_notifications`
**Expected size:** 10–50 KiB (max 200 notifications)

### Fields

```js
{
  items: [{
    id: string,                 // 'n_' + timestamp + '_' + random
    type: string,               // 'shift' | 'event' | 'conflict' | 'budget' | 'expense'
    title: string,
    body: string,
    timestamp: number,
    read: boolean,
    eventId: string,
    companyId: string
  }],

  _lastModified: string
}
```

### Sync Considerations

Notifications are informational and device-context-specific (e.g., "your shift starts in 15 minutes" fires per-device). Syncing them is lower priority than main state. Consider:
- Sync only `read` status (so marking all read on one device applies everywhere)
- Don't generate duplicate notifications on other devices during sync

---

## 5. Document: `meta/sync`

**Path:** `users/{uid}/meta/sync`
**Purpose:** Sync engine metadata — not user data

### Fields

```js
{
  lastPushAt: string,           // ISO timestamp of last successful push to Firestore
  lastPullAt: string,           // ISO timestamp of last successful pull from Firestore
  schemaVersion: number,        // For future migration detection
  deviceId: string,             // Unique per-device ID (crypto.randomUUID)
  devices: [{                   // Registry of known devices
    deviceId: string,
    lastSeenAt: string,
    userAgent: string            // Truncated to browser name + OS for identification
  }]
}
```

### Device Registry

Each device that syncs registers itself in the `devices` array. This helps:
- Identify which device made the latest change
- Prune stale devices (no sync in 90+ days)
- Show "last synced from" info in the UI

---

## 6. Field-Level `updatedAt` Tracking

Every entity that can be individually edited needs an `updatedAt` timestamp. This is the foundation of conflict resolution.

### Adding `updatedAt` to Existing Entities

The current IDB model doesn't have `updatedAt` on cards, events, or budgets. Add it incrementally:

```js
function ensureUpdatedAt(item) {
  if (!item.updatedAt) {
    item.updatedAt = item.createdAt || new Date().toISOString();
  }
  return item;
}

// On every save/edit operation:
function updateItem(item, changes) {
  Object.assign(item, changes);
  item.updatedAt = new Date().toISOString();
  return item;
}
```

### Timestamp Format

Use ISO 8601 strings (`new Date().toISOString()`) everywhere. Firestore Timestamps are more powerful but ISO strings work in both IDB and Firestore, keeping the sync simple.

Ensure device clocks are reasonably accurate. Significant clock skew (minutes or more) can cause last-write-wins to pick the wrong "winner." This is an inherent limitation of LWW — acceptable for a personal app where devices are typically owned by the same person and have NTP-synced clocks.

---

## 7. Deletion Log

### Structure

```js
// In state/main document:
_deletedItems: [
  { collection: 'cards', id: 'card_abc123', deletedAt: '2026-03-17T09:00:00.000Z' },
  { collection: 'events', id: 'evt_def456', deletedAt: '2026-03-17T09:15:00.000Z' }
]
```

### Lifecycle

1. **On delete:** Push entry to `_deletedItems` array, remove item from its array
2. **On sync push:** Include `_deletedItems` in the Firestore document
3. **On sync pull:** Process `_deletedItems` from remote — remove matching local items
4. **On prune:** Every sync cycle, remove entries older than 30 days

### Why 30 Days?

If a device goes offline for more than 30 days and then syncs, deleted items may reappear. 30 days is a reasonable window — devices offline longer than that should do a full reset sync.

### Conflict: Delete vs. Edit

If Device A deletes a card and Device B edits the same card before syncing:
- The delete entry has a `deletedAt` timestamp
- The edit has an `updatedAt` timestamp
- **If `deletedAt` > `updatedAt`:** delete wins (the deletion happened after the edit)
- **If `updatedAt` > `deletedAt`:** edit wins (the edit happened after the deletion — the user re-created or re-edited it on another device)
- Log these conflicts in the debug console for transparency

---

## 8. Document Size Management

### Firestore Limits

- Maximum document size: **1 MiB** (1,048,576 bytes)
- Recommended working limit for Planora: **800 KiB** (leave headroom)

### Estimating Size

```js
function estimateDocSize(obj) {
  return new Blob([JSON.stringify(obj)]).size;
}
```

This is approximate — Firestore's internal encoding adds overhead for field names and types. For safety, treat JSON size * 1.3 as the Firestore size.

### Typical User Sizes

| Component | Per-Item Size | 100 Items | 500 Items |
|-----------|--------------|-----------|-----------|
| Cards | ~300 bytes | 30 KiB | 150 KiB |
| Events | ~500 bytes | 50 KiB | 250 KiB |
| Expenses | ~400 bytes | 40 KiB | 200 KiB |
| Budgets | ~150 bytes | 15 KiB | 75 KiB |
| Categories | ~100 bytes | 10 KiB | 50 KiB |

A power user with 500 events + 500 expenses ≈ ~500 KiB. Well within limits.

### Growth Strategy: Subcollection Splitting

If `state/main` exceeds 800 KiB, split arrays into subcollections:

```
users/{uid}/
├── state/main              ← settings + metadata only
├── state/alerts
├── state/notifications
├── data/cards/{cardId}     ← one document per card
├── data/events/{eventId}   ← one document per event
└── data/expenses/{expId}   ← one document per expense
```

This is a future optimization. Implement only when monitoring shows a user approaching the limit. The sync engine should be designed with this migration path in mind — use an abstraction layer for reading/writing entity arrays so the underlying storage can switch from embedded-in-document to subcollection without changing sync logic.

---

## 9. Index Requirements

### Default Indexes

Firestore automatically creates single-field indexes for every field. For Planora's current access patterns (read entire user document by path), **no custom composite indexes are needed.**

### Future Indexes (If Subcollections Are Adopted)

If events or expenses move to subcollections, add composite indexes for common queries:

```
// If querying events by date range
Collection: users/{uid}/data/events
Fields: startDate (ASC), startTime (ASC)

// If querying expenses by date and type
Collection: users/{uid}/data/expenses
Fields: date (DESC), type (ASC)
```

Define these in `firestore.indexes.json` and deploy with `firebase deploy --only firestore:indexes`.

---

## 10. Data Migration from IDB

### First Sync (No Firestore Data)

When a user signs in for the first time, their Firestore is empty. The sync engine should:

1. Read all data from IDB (existing local state)
2. Add `updatedAt` to any entities missing it
3. Initialize `_deletedItems` as empty array
4. Set `_schemaVersion` and `_lastModified`
5. Push entire state to Firestore as a single write
6. Write sync metadata to `meta/sync`

### Adding `updatedAt` to Legacy Data

Existing IDB data won't have `updatedAt` on cards, events, or budgets. During the first sync:

```js
function prepareForFirstSync(state) {
  const now = new Date().toISOString();

  state.cards = (state.cards || []).map(c => ({
    ...c,
    updatedAt: c.updatedAt || now
  }));

  state.events = (state.events || []).map(e => ({
    ...e,
    updatedAt: e.updatedAt || e.createdAt || now
  }));

  state.expenses = (state.expenses || []).map(e => ({
    ...e,
    updatedAt: e.updatedAt || e.createdAt || now
  }));

  state.budgets = (state.budgets || []).map(b => ({
    ...b,
    updatedAt: b.updatedAt || now
  }));

  state._deletedItems = [];
  state._schemaVersion = DATA_SCHEMA_VERSION;
  state._lastModified = now;

  return state;
}
```

### Schema Version Checks

On every sync pull, compare `_schemaVersion` from Firestore with the local `DATA_SCHEMA_VERSION`. If they differ, run the existing `migrateImportedData()` migration chain before merging.
