# Planora — Complete Application Specification

> **Purpose:** A single-page, offline-first, mobile-friendly web application for managing work schedules across multiple companies/clients in different time zones. It combines timezone conversion, shift planning, event management, a weekly calendar, a full alert/notification engine, and a **full expense/income tracker with multi-currency support** — all in one self-contained HTML file (~13,400 lines) using **IndexedDB** for persistence (no backend).

---

## TABLE OF CONTENTS

1. [Architecture & Tech Stack](#1-architecture--tech-stack)
2. [Theme System (Dark/Light)](#2-theme-system)
3. [Navigation & Page Structure](#3-navigation--page-structure)
4. [Welcome Gate](#4-welcome-gate)
5. [Home Page — Company Cards (Timezone Cards)](#5-home-page--company-cards)
6. [Calendar Page — Weekly Schedule Grid](#6-calendar-page--weekly-schedule-grid)
7. [Events Page — Event Management](#7-events-page--event-management)
8. [Settings Context Menu](#8-settings-context-menu)
9. [Alert / Notification Engine](#9-alert--notification-engine)
10. [Notification Center](#10-notification-center)
11. [Data Import / Export](#11-data-import--export)
12. [Data Models](#12-data-models)
13. [Timezone Engine](#13-timezone-engine)
14. [Mobile & Responsive Behavior](#14-mobile--responsive-behavior)
15. [Accessibility & UX Details](#15-accessibility--ux-details)
16. [CSS Design System & Styling](#16-css-design-system--styling)
17. [Expense / Income Tracker (Wallet Page)](#17-expense--income-tracker-wallet-page)
18. [Category Management](#18-category-management)
19. [Currency Management System](#19-currency-management-system)
20. [Recurring Expense/Income Entries](#20-recurring-expenseincome-entries)
21. [Breakdown View](#21-breakdown-view)
22. [Company Net Summary](#22-company-net-summary)
23. [Reimbursable View](#23-reimbursable-view)
24. [Budget Management](#24-budget-management)
25. [Alert Engine — Expense Integration](#25-alert-engine--expense-integration)
26. [Sound Engine](#26-sound-engine)
27. [Debug Console](#27-debug-console)
28. [Toast System](#28-toast-system)
29. [QR Sync](#29-qr-sync)
30. [SVG Icon & Shape System](#30-svg-icon--shape-system)
31. [Marker System](#31-marker-system)
32. [Conflict Detection](#32-conflict-detection)

---

## 1. Architecture & Tech Stack

- **Single HTML file** (~13,400 lines) — all HTML, CSS, and JavaScript inline
- **Google Fonts**: Inter font family (weights 400–900) loaded from `fonts.googleapis.com`
- **External CDN dependencies**:
  - `qrcode@1.4.4` — QR code generation for data sharing
  - `lz-string@1.5.0` — data compression for QR payloads
  - `html5-qrcode@2.3.8` — camera-based QR code scanning
- **Pure vanilla JavaScript** (ES6+, but uses `var` in some places for legacy compat)
- **IndexedDB** for all persistence:
  - Database name: `tz_planner_idb`
  - Single object store: `kv` (key-value pattern)
  - Three IDB keys:
    - `tz_main_state` — main app state (cards, events, calendar settings, wallet data, etc.)
    - `tz_alert_state` — alert engine configuration (excludes `firedAlerts` and `snoozed`)
    - `tz_notifications` — notification center entries
  - **localStorage fallback**: if IndexedDB unavailable, uses `idb_` prefixed keys in localStorage
  - **Migration from localStorage**: on first IDB load, auto-migrates from:
    - `timezone_planner_v7_events` → `tz_main_state`
    - `timezone_planner_alerts_v1` → `tz_alert_state`
- **Theme storage**: `timezone_planner_theme_v1` in **localStorage** (NOT IDB — needed for sync page load before IDB async init)
- **Schema versioning**: `DATA_SCHEMA_VERSION = 1` with migration chain support
- **Intl.DateTimeFormat** for all timezone calculations (no moment.js or similar)
- **Web Audio API** for alert sounds (synthesized, not audio files)
- **SpeechSynthesis API** for text-to-speech alerts
- **Vibration API** for haptic feedback
- **Two `<script>` blocks**:
  1. First block (synchronous): Debug console, toast system, secret password handler
  2. Second block (async IIFE): All app logic, IDB init, state management, rendering

### Preload & Init Flow

```
1. First <script> block executes synchronously:
   - Debug console interceptors (console.error, console.warn, console.log)
   - window.onerror handler
   - Debug FAB setup
   - Secret password detection on input fields
   - Toast system (window.__appToast)
2. Second <script> async IIFE:
   a. Initialize __idb key-value store
   b. Preload from IDB: tz_main_state, tz_alert_state, tz_notifications
   c. Migrate from localStorage if IDB empty
   d. Set constants (STORAGE_KEY, DATA_SCHEMA_VERSION, THEME_KEY, TZ_FALLBACK, TZ_ALIASES)
   e. Build DOM references
   f. Build timezone choice data
   g. Apply saved theme
   h. Initialize UI state
   i. loadState() → state
   j. loadAlertState() → alertState
   k. render(), attachEvents(), startClock()
   l. Start alert check interval (15s)
   m. Welcome gate logic
```

---

## 2. Theme System

Two themes: **Dark** (default) and **Light**. Controlled via `data-theme` attribute on `<html>`.

### Dark Theme (default — `data-theme="dark"`)

```css
:root {
  color-scheme: dark;
  --bg: #080d1a;
  --bg-gradient: radial-gradient(ellipse at 20% 0%, rgba(56, 189, 248, 0.06) 0%, transparent 50%),
                 radial-gradient(ellipse at 80% 100%, rgba(139, 92, 246, 0.05) 0%, transparent 50%),
                 radial-gradient(circle at 50% 20%, #151d2e, #080d1a 70%);
  --surface: rgba(22, 33, 55, 0.55);
  --surface-hover: rgba(35, 50, 78, 0.72);
  --surface-solid: #131c2e;
  --border: rgba(148, 163, 184, 0.07);
  --border-strong: rgba(148, 163, 184, 0.14);
  --text: #f1f5f9;
  --text-muted: #94a3b8;
  --text-subtle: #64748b;
  --accent: #38bdf8;
  --accent-hover: #7dd3fc;
  --accent-gradient: linear-gradient(135deg, #38bdf8, #22d3ee);
  --danger: #f87171;
  --danger-soft: rgba(248, 113, 113, 0.12);
  --shadow-sm: 0 2px 8px -2px rgba(0, 0, 0, 0.2), 0 4px 16px -4px rgba(0, 0, 0, 0.15);
  --shadow-lg: 0 12px 40px -8px rgba(0, 0, 0, 0.4), 0 4px 16px -4px rgba(0, 0, 0, 0.25);
  --shadow-glow: 0 0 24px rgba(56, 189, 248, 0.25), 0 0 48px rgba(56, 189, 248, 0.08);
  --radius: 20px;
  --radius-sm: 14px;
  --radius-pill: 9999px;
  --transition: 220ms cubic-bezier(0.32, 0.72, 0, 1);
  --transition-spring: 380ms cubic-bezier(0.32, 0.72, 0, 1);
  --transition-slow: 320ms cubic-bezier(0.32, 0.72, 0, 1);
  --safe-top: env(safe-area-inset-top, 0px);
  --safe-bottom: env(safe-area-inset-bottom, 0px);
}
```

- Background noise texture applied via `body::before` pseudo-element (SVG noise filter, subtle grain effect)
- Background uses `var(--bg-gradient)` for layered radial gradients

### Light Theme (`data-theme="light"`)

```css
[data-theme="light"] {
  color-scheme: light;
  --bg: #f0f4f8;
  --bg-gradient: none;
  --surface: rgba(255, 255, 255, 0.78);
  --surface-hover: rgba(255, 255, 255, 0.95);
  --surface-solid: #f8fafc;
  --border: rgba(15, 23, 42, 0.06);
  --border-strong: rgba(15, 23, 42, 0.11);
  --text: #0f172a;
  --text-muted: #475569;
  --text-subtle: #64748b;
  --danger-soft: rgba(248, 113, 113, 0.1);
  --shadow-sm: 0 2px 12px rgba(15, 23, 42, 0.06), 0 4px 24px rgba(15, 23, 42, 0.04);
  --shadow-lg: 0 12px 48px rgba(15, 23, 42, 0.1), 0 4px 16px rgba(15, 23, 42, 0.06);
  --shadow-glow: 0 0 24px rgba(56, 189, 248, 0.12);
}
```

- Light mode **kills noise texture**: `body::before { display: none }` 
- Light mode forces **flat background** (no gradient via `--bg-gradient: none`)
- `--accent`, `--accent-hover`, `--accent-gradient`, `--danger` stay the same in both themes

### Theme Toggle

- Moon/Sun icon button in topbar left side
- Toggles `data-theme` between `dark` and `light`
- Saves to `localStorage` key: `timezone_planner_theme_v1`
- Theme loaded on page init via `loadTheme()` function (synchronous, before render)

### Accent Color

- Primary accent: **`#38bdf8`** (sky blue/cyan) — used throughout for buttons, links, highlights, glow effects
- This replaces the older purple (`#6c5ce7`) accent

---

## 3. Navigation & Page Structure

### Pages (4 total)

| Page | ID | Nav Icon | Description |
|------|----|----------|-------------|
| Home | `pageHome` | House (`icon-home`) | Company cards / timezone list |
| Calendar | `pageCalendar` | Calendar (`icon-calendar`) | Weekly schedule grid |
| Events | `pageEvents` | Notes (`icon-notes`) | Event list & management |
| Wallet | `pageWallet` | Wallet (`icon-wallet`) | Expense/income tracker |

### Bottom Navigation Bar

- Fixed bottom bar with 4 tab buttons
- Each tab: icon (inline SVG `<use>`) + label text
- Active tab highlighted with accent color
- **Auto-hide on calendar scroll**: bottom nav hides when scrolling the calendar grid
- **Peek strip** (`#bottomNavPeek`): a small strip visible when nav is hidden, tap to show nav

### Header / Topbar

- Fixed top bar with three sections:
  - **Left side**: Theme toggle button (moon icon → dark, sun icon → light)
  - **Right side**: Settings gear button + Alert bell button with notification badge
  - **Center/below**: Clock display area
- **Live clock**: shows current time with seconds, 12-hour format (lowercase am/pm)
  - Format: `hh:mm:ss am/pm`
  - Updated every second via `startClock()` → `setInterval` 
- **Date display**: `{weekday}, {dateLong}` (e.g., "Saturday, March 15, 2026")
- **Base timezone info**: pin icon + `{timezone} · {UTC offset} · {abbreviation}`
- Title "**Planora**" displayed with gradient text effect in dark mode

### FAB (Floating Action Button)

- Bottom-right floating button (`#mainFab`)
- Plus icon
- Contextual: opens action sheet for current page
- On Home: shows home actions (add card, import/export)
- On Calendar: shows calendar actions (add event)
- On Events: shows events actions  
- On Wallet: shows wallet actions (add expense/income, manage categories)

### Scrim & Action Sheets

- Modal overlay (`#scrim`) shown behind sheets
- Bottom sheets slide up from bottom with backdrop blur
- All sheets have close button and drag-to-dismiss support

---

## 4. Welcome Gate

Full-screen overlay shown on first visit to activate audio context.

### Structure

```html
<div id="welcomeGate" class="welcome-gate">
  <div class="welcome-gate-bg"></div>
  <div class="welcome-content">
    <div class="welcome-icon-ring">
      <!-- Smiley face SVG icon -->
    </div>
    <h1 class="welcome-greeting">Good Morning</h1>
    <p class="welcome-sub">Tap anywhere to start your day</p>
    <div class="welcome-tap-ring">
      <div class="welcome-tap-dot"></div>
    </div>
    <p class="welcome-hint">This activates sound &amp; voice alerts</p>
  </div>
</div>
```

### Behavior

- **Time-of-day greeting**: 
  - Before 12pm: "Good Morning"
  - 12pm–5pm: "Good Afternoon"  
  - After 5pm: "Good Evening"
- **Animated smiley face** in a pulsing ring (`welcomePulseRing` animation)
- **Tap anywhere to dismiss**: 
  - Adds `.leaving` class (opacity 0, scale 1.05 transition over 0.7s)
  - Removes element after transition
  - Activates AudioContext for sound alerts
- **Skip mechanism**: `localStorage.setItem('tz_skip_welcome', '1')` bypasses gate
- After data import, `tz_skip_welcome` is automatically set

### Animations

- `welcomeFadeUp`: content slides up from 30px with opacity fade (1s)
- `welcomePulseRing`: ring scales 1→1.05 with box-shadow glow (2.5s infinite)
- `welcomeTapPulse`: tap ring scales 1→1.15 (2s infinite)
- `welcomeDotPulse`: dot opacity 0.5→1 and scale 1→1.3 (2s infinite)

### Theme Variants

- **Dark**: gradient background `linear-gradient(135deg, #0f0c29, #302b63, #24243e)`
- **Light**: gradient background `linear-gradient(135deg, #667eea, #764ba2, #f093fb)`
- Light mode ring: more opaque white background, stronger border

---

## 5. Home Page — Company Cards

### Card Data Model

```javascript
{
  id: string,              // cryptoRandomId() — crypto.getRandomValues based
  label: string,           // Company/client name
  targetZone: string,      // IANA timezone (e.g., "America/New_York")
  targetDisplay: string,   // Human-readable display name
  inputMode: 'target' | 'local',  // Whether times entered in target or local tz
  ignoreDst: boolean,      // Fixed-offset mode (ignore DST transitions)
  colorHex: string,        // Card accent color (e.g., "#38bdf8")
  startTime: string,       // Shift start "HH:MM" (24h format)
  endTime: string          // Shift end "HH:MM" (24h format)
}
```

### Card Creation (New Card Sheet)

- `createBlankCard()` generates defaults:
  - `id`: `cryptoRandomId()`
  - `startTime`: current time in base timezone
  - `endTime`: startTime + 60 minutes
  - `inputMode`: `'target'`
  - `ignoreDst`: `false`
  - `colorHex`: `''` (defaults to accent `#38bdf8` when empty)
  - Internal `_alertSettings`: `{ enabled: false, timeRef: 'target', startLead: 15, endLead: 10 }`
  - Internal `_alertTplStart`: `'tpl_default'`
  - Internal `_alertTplEnd`: `'tpl_default'`

### Card Display (View Mode)

Each card shows:
- **Live clock header**: real-time clock with seconds for the card's timezone
- **Company label** with color accent
- **Target timezone times**: start → end with offset and DST info
- **Local timezone times**: converted start → end with day delta if applicable
- **Offset difference**: e.g., "+8h" or "-5h 30m"
- **DST notes**: if DST transition detected
- Card glow effect when alert is within lead time (`.card-glow-start`, `.card-glow-end`, `.card-glow-both`)

### Card Editing (Edit Mode)

Inline edit mode per card with:
- Label input
- Timezone picker button (opens timezone picker sheet)
- Color picker (hex input with swatch preview)
- Input mode toggle (Target / Local)
- Ignore DST checkbox
- Start/end time inputs
- Alert settings section
- Save/Cancel buttons

### Computed Card View

`computeCard(card)` returns:
- Target start/end times formatted
- Local start/end times (converted)
- Offset difference string
- Day delta (if times span midnight)
- DST notes
- Card pattern for calendar grid

### Card Persistence

- `isPersistableCard(card)`: requires `targetZone` + valid `startTime` + valid `endTime`
- Sanitized via `sanitizeCard()` before save
- Alert settings stored separately in `alertState.companyAlerts[cardId]`
- Template assignments in `alertState.templateAssignments[cardId + '_start']` and `[cardId + '_end']`

### Home Page Search

- Search bar (`#homeSearchInput`) at top of home page (hidden when no cards)
- Filters cards by label and targetDisplay (case-insensitive substring match)

---

## 6. Calendar Page — Weekly Schedule Grid

### Calendar Settings

```javascript
{
  slotInterval: 5 | 10 | 15 | 20 | 30 | 45 | 60,  // default: 30
  weekMode: 'sunday' | 'monday' | 'custom',          // default: 'sunday'
  weekOrder: ['sun','mon','tue','wed','thu','fri','sat'], // day order
  conflictNotify: boolean,                             // default: true
  todayBorderColor: string,                            // default: '#38bdf8'
  todayBorderThickness: 2-6,                           // default: 3 (px)
  nowLineColor: string,                                // default: '#38bdf8'
  nowLineThickness: 1-4,                               // default: 2 (px)
  conflictIndicator: {
    style: 'label' | 'dot',        // default: 'label'
    labelText: string,              // default: 'CONFLICT' (max 12 chars, uppercased)
    color: string | null,           // null = auto from event color
    dotSize: 2-20                   // default: 8 (px)
  }
}
```

### Navigation Controls

**Week navigation**:
- Previous week / Today / Next week buttons
- Week range label: e.g., "Mar 9 – Mar 15"
- Today button disabled when `calendarWeekOffset === 0`

**Day navigation** (within selected week):
- First day / Previous day / Next day / Last day buttons
- Current day display: short day name + month/day (e.g., "Sat" / "Mar 15")
- First/Prev disabled at start of week, Next/Last disabled at end

### Schedule Grid

- CSS Grid layout: time column + company columns
- `gridTemplateColumns`: `var(--schedule-time-col-width) repeat(N, var(--schedule-company-col-width))`
- `gridTemplateRows`: `var(--schedule-company-row-height) repeat(M, var(--schedule-slot-row-height))`
- Responsive layout computed by `computeScheduleLayout()`:
  - Compact mode (≤900px): `timeColWidth: 92px`, `slotRowHeight: 26px`
  - Normal mode: `timeColWidth: 98px`, `slotRowHeight: 28px`
  - Company column width: `max(26, availableWidth / cardsCount)`
  - Company row height: `max(92/100, min(360, 22 + labelLength * 11.5))`

### Schedule Grid Features

- **Company header columns**: vertical labels with accent color, rotated text
- **Time slots**: rows showing time ranges (formatted based on slotInterval)
- **Today column highlighting**: colored borders on today's column (customizable color & thickness)
- **Now line**: horizontal line at current time position (customizable color & thickness)
- **Now label**: shows current time next to now line
- **Schedule slot events**: rendered as markers within cells
- **Slot action popup**: tap a cell to see events, create new event, or open existing
- **Conflict indicators**: `has-conflict` and `conflict-danger` CSS classes on cells
- **Auto-fit conflict labels**: `autoFitConflictLabels()` shrinks label text to fit cell width
- **Legend card**: shows event type markers with labels

### Calendar Settings Sheet

- Slot interval dropdown (5/10/15/20/30/45/60 min)
- Week start mode: Sunday / Monday / Custom buttons
- Custom week order: drag-reorder UI with up/down buttons per day
- Conflict notifications toggle
- Today border: color picker + thickness slider (2-6px)
- Now line: color picker + thickness slider (1-4px)
- Conflict indicator settings:
  - Style: Label / Dot toggle
  - Label text input (max 12 chars, auto-uppercased)
  - Color: Auto (from event) / Custom picker
  - Dot size slider (2-20px)
- Settings summary displayed at bottom

### Custom Week Order

- Week days constant: `WEEK_DAYS = [{id:'sun', label:'Sunday'}, ... {id:'sat', label:'Saturday'}]`
- `sundayWeekOrder()` returns `['sun','mon','tue','wed','thu','fri','sat']`
- `mondayWeekOrder()` returns `['mon','tue','wed','thu','fri','sat','sun']`
- Custom: user reorders via drag/up/down controls

---

## 7. Events Page — Event Management

### Event Data Model

```javascript
{
  id: string,                    // cryptoRandomId()
  title: string,                 // Event name
  typeId: string,                // References event type id
  ownerCompanyId: string,        // Company that owns this event (or '')
  laneCompanyId: string,         // Calendar lane to display in (or '')
  startDate: 'YYYY-MM-DD',      // Start date
  startTime: 'HH:MM',           // Start time (24h)
  endDate: 'YYYY-MM-DD',        // End date
  endTime: 'HH:MM',             // End time (24h)
  status: 'active' | 'completed' | 'cancelled',  // default: 'active'
  priority: 1-100,               // default: 50
  recurrence: {
    mode: 'once' | 'daily' | 'everyXDays',  // default: 'once'
    interval: 1-365              // default: 2 (only for everyXDays)
  },
  notesText: string,             // Current notes input text
  notes: [{                      // Persisted note history
    id: string,
    text: string,
    createdAt: ISO string
  }],
  subtasks: [{                   // Checklist items
    id: string,
    text: string,
    done: boolean
  }]
}
```

### Event Creation (Add Event Sheet)

- `createBlankEventDraft()` generates defaults:
  - `id`: `''` (empty = new event)
  - `typeId`: first event type id or `'meeting'`
  - `startDate`: from context dayKey, or current page date
  - `startTime`: from context slotStart, or `'09:00'` (540 minutes)
  - `endTime`: startTime + slotInterval
  - `priority`: 50
  - `recurrence`: `{ mode: 'once', interval: 2 }`

### Event Settings

```javascript
{
  markerStyle: 'mixed',           // Currently always 'mixed'
  types: EventType[],             // Array of event types
  hideCompleted: boolean          // default: true — hide completed events in list
}
```

### Default Event Types

```javascript
[
  { id: 'meeting', name: 'Meeting', markerKind: 'icon', colorHex: '#38bdf8', shape: 'square', icon: 'briefcase', notificationsEnabled: true, conflictAlertsEnabled: true, alertBeforeValue: 30, alertBeforeUnit: 'minutes', maxAlerts: 3 },
  { id: 'task', name: 'Task', markerKind: 'shape', colorHex: '#a78bfa', shape: 'diamond', icon: 'task', notificationsEnabled: false, conflictAlertsEnabled: true, alertBeforeValue: 15, alertBeforeUnit: 'minutes', maxAlerts: 2 },
  { id: 'reminder', name: 'Reminder', markerKind: 'icon', colorHex: '#14b8a6', shape: 'circle', icon: 'bell', notificationsEnabled: true, conflictAlertsEnabled: false, alertBeforeValue: 10, alertBeforeUnit: 'minutes', maxAlerts: 3 },
  { id: 'headsup', name: 'Heads-up', markerKind: 'shape', colorHex: '#f59e0b', shape: 'triangle', icon: 'alert', notificationsEnabled: true, conflictAlertsEnabled: true, alertBeforeValue: 60, alertBeforeUnit: 'minutes', maxAlerts: 4 }
]
```

### Event Type Fields

```javascript
{
  id: string,                          // Unique id (auto-generated from name or cryptoRandomId)
  name: string,                        // Display name
  markerKind: 'icon' | 'shape',       // How to render the marker
  colorHex: string,                    // Hex color code
  shape: string,                       // Shape from SHAPE_OPTIONS
  icon: string,                        // Icon from ICON_OPTIONS
  notificationsEnabled: boolean,       // Enable notifications for this type
  conflictAlertsEnabled: boolean,      // Enable conflict detection for this type
  alertBeforeValue: number,            // Lead time before event (0-9999)
  alertBeforeUnit: 'seconds' | 'minutes' | 'hours',  // Lead time unit
  maxAlerts: number                    // Max alert repetitions (1-20)
}
```

### Events Page Features

- **Context row**: shows current filter context (company, day, lane)
- **Search bar**: filter events by text
- **Event cards**: expandable cards showing event details
- **Conflict banner**: shows conflict summaries at top
- **Add comments**: inline note input per event (Enter to save)
- **Toggle complete**: mark events as completed
- **Delete events**: with confirmation modal
- **Subtasks**: checklist items within events

### Event Occurrence System

- `doesEventOccurOnStartDay(event, dayKey)`:
  - `once`: only on `event.startDate`
  - `daily`: every day after startDate
  - `everyXDays`: every N days after startDate (using date diff modulo interval)
- `getEventOccurrenceOnDay(event, dayKey)`: handles overnight events, returns segments with startMinute/endMinute
- Overnight events: spans midnight, checked via comparing start/end dates or endMin ≤ startMin

---

## 8. Settings Context Menu

A popup menu triggered by the gear button in the topbar.

### Menu Items

1. **Event Settings** — Opens event settings sheet (event type management)
2. **Calendar Settings** — Opens calendar settings sheet
3. **Alert Settings** — Opens alert settings sheet

### Overlay

- `.settings-ctx-overlay` covers screen behind menu
- `.settings-ctx-menu` positioned near gear button
- Click overlay or any item to dismiss

---

## 9. Alert / Notification Engine

### Alert State

```javascript
{
  enabled: boolean,              // default: true — master on/off for popup/sound/TTS
  mode: 'normal' | 'annoying' | 'minimal' | 'visual',  // default: 'normal'
  userName: string,              // default: '' — for template personalization
  globalEventLeadMinutes: 15,    // Fallback lead time
  companyAlerts: {},             // Per-company alert config
  // companyAlerts[cardId] = { enabled: true, timeRef: 'target'|'local', startLead: 15, endLead: 10, templateId: 'default' }
  eventTypeLeadTimes: {},        // Per-event-type lead times
  // eventTypeLeadTimes[typeId] = { startLead: 10 }
  templates: AlertTemplate[],    // Array of alert message templates
  templateAssignments: {},       // Template assignments per card/event type
  // templateAssignments[companyId_start | companyId_end | eventTypeId] = templateId
  soundEnabled: boolean,         // default: true
  soundType: 'chime' | 'bell' | 'pulse' | 'digital' | 'siren',  // default: 'chime'
  soundVolume: 0.0-1.0,         // default: 0.7
  vibrationEnabled: boolean,     // default: true
  ttsEnabled: boolean,           // default: true
  ttsRate: number,               // default: 1.0
  firedAlerts: {},               // MEMORY-ONLY (never persisted) — tracks fired alert keys this session
  snoozed: {},                   // MEMORY-ONLY (never persisted) — snoozed alert expiry timestamps
  expenseAlerts: {               // Expense/budget alert settings
    enabled: boolean,              // default: false
    recurringDueEnabled: boolean,  // default: true
    overdueEnabled: boolean,       // default: true
    budgetThresholdEnabled: boolean, // default: true
    budgetExceededEnabled: boolean,  // default: true
    reimbursementReminderEnabled: boolean, // default: true
    reimbursementReminderDays: 7   // Days after submission to remind
  }
}
```

### Built-in Alert Templates

```javascript
[
  { id: 'tpl_default', name: 'Default Reminder', text: 'Hey {name}, your {eventType} "{eventTitle}" with {company} starts in {leadTime}!', builtIn: true },
  { id: 'tpl_shift', name: 'Shift Start', text: 'Hey {name}, your shift with {company} will start in {leadTime}. Get ready!', builtIn: true },
  { id: 'tpl_shift_end', name: 'Shift Ending', text: '{name}, your shift with {company} ends in {leadTime}. Wrap things up!', builtIn: true },
  { id: 'tpl_meeting', name: 'Meeting Notice', text: '{name}, heads up — "{eventTitle}" is in {leadTime}.', builtIn: true },
  { id: 'tpl_recurring_due', name: 'Recurring Payment Due', text: 'Hey {name}, your recurring {expenseCategory} payment of {expenseCurrency}{expenseAmount} is due today!', builtIn: true },
  { id: 'tpl_overdue', name: 'Payment Overdue', text: '{name}, your {expenseCategory} payment of {expenseCurrency}{expenseAmount} from {expenseDate} is still pending.', builtIn: true },
  { id: 'tpl_budget_threshold', name: 'Budget Threshold', text: '{name}, heads up — your {expenseCategory} budget is at {budgetPercent}% ({budgetRemaining} remaining).', builtIn: true },
  { id: 'tpl_budget_exceeded', name: 'Budget Exceeded', text: '⚠️ {name}, your {expenseCategory} budget has been exceeded! Now at {budgetPercent}%.', builtIn: true },
  { id: 'tpl_reimb_reminder', name: 'Reimbursement Reminder', text: '{name}, your {expenseCategory} reimbursement of {expenseCurrency}{expenseAmount} has been submitted for {leadTime} — follow up!', builtIn: true }
]
```

### Template Variables

`TEMPLATE_VARS`: `['name', 'eventTitle', 'eventType', 'company', 'leadTime', 'startTime', 'endTime', 'date', 'expenseType', 'expenseAmount', 'expenseCurrency', 'expenseCategory', 'expenseDescription', 'expenseDate', 'expenseStatus', 'budgetPercent', 'budgetRemaining']`

Template rendering: `text.replace(/\{(\w+)\}/g, (m, k) => vars[k] !== undefined ? vars[k] : m)`

### Alert Check Engine

`checkUpcomingAlerts()` runs every **15 seconds** via `setInterval`.

**Part 1: Card Shift Alerts**
- For each card with `companyAlerts[cardId].enabled`:
  - Determines today in card's timezone
  - Checks START alert: key = `shift-{cardId}-{todayKey}-{startTime}-start`
  - Checks END alert: key = `shift-{cardId}-{todayKey}-{endTime}-end`
  - If `diffMinutes > 0 && diffMinutes <= leadTime` and not already fired → fire alert
  - Snooze support: if snoozed and expired, re-fire
  - Card glow: sets `cardGlows[cardId]` = `'start'`, `'end'`, or `'both'`

**Part 2: Event Alerts**
- For each event with startDate and startTime:
  - For recurring events: checks `doesEventOccurOnStartDay()` for today
  - Key: `{eventId}-{dateKey}-start`
  - Same logic as card alerts for firing and snoozing
  - Also checks END alerts for events with companyAlerts enabled

**Cleanup**: fired alerts and snoozed entries older than 2 hours are pruned (memory-only, session-scoped).

### Alert State Persistence

- `firedAlerts` and `snoozed` are **never persisted** — they are memory-only per session
- On `loadAlertState()`, `firedAlerts` and `snoozed` are always reset to `{}`
- On `saveAlertState()`, these fields are deleted from the save payload
- Templates are merged with defaults on load (new built-in templates added if missing)

### Alert Modes

| Mode | Sound | TTS | Vibration | Popup |
|------|-------|-----|-----------|-------|
| normal | Once | Once (1.5s delay) | Normal pattern | Yes |
| annoying | Once | Looping (1.8s gap) | Annoying pattern | Yes |
| minimal | Once | None | None | Yes |
| visual | None | None | None | Yes |

### Alert Popup

- Full-screen overlay with card showing:
  - Emoji icon
  - Title and message (rendered from template)
  - Meta HTML (event type, time details)
  - Dismiss button
  - Snooze button (snoozes for 5 minutes)

---

## 10. Notification Center

Separate from alert popup — a persistent log of all notifications.

### Storage

- IDB key: `tz_notifications`
- Max 200 notifications (FIFO pruning when exceeded)
- Saved via `saveNotifications()` → `__idb.set('tz_notifications', notifStore)`

### Notification Model

```javascript
{
  id: string,              // 'n_' + Date.now() + '_' + random5chars
  type: 'shift' | 'event' | 'conflict' | 'budget' | 'expense',
  title: string,
  body: string,
  timestamp: number,       // Date.now()
  read: boolean,
  eventId: string,         // Associated event (or '')
  companyId: string        // Associated company (or '')
}
```

### UI

- Opens as bottom sheet from bell button in topbar
- **Badge**: shows unread count (max display "99+"), `.has-count` class adds styling
- **Mark all as read**: all notifications marked read when sheet opens
- **Individual dismiss**: swipe/tap to remove with animation
- **Individual mark-as-read**: tap to mark single notification read
- **Clear all**: button to remove all notifications

### Relative Timestamps

```javascript
formatNotifTime(timestamp):
  < 60s:      "Just now"
  < 1h:       "{X}m ago"
  < 24h:      "{X}h ago"
  < 7d:       "{X}d ago"
  >= 7d:      "Mon DD" format
```

---

## 11. Data Import / Export

### Export

```javascript
exportAllData() → JSON file download:
{
  _schemaVersion: DATA_SCHEMA_VERSION,  // Currently 1
  _exportedAt: ISO timestamp,
  _appName: 'Planora',
  state: {
    baseZone, baseDisplay,
    calendarSettings: sanitized,
    events: sanitized array,
    eventSettings: sanitized,
    expenseCategories, incomeCategories,
    expenses, budgets,
    walletCurrencies,
    cards: sanitized array
  },
  alertState: {
    // Full alertState MINUS firedAlerts and snoozed
  },
  theme: 'dark' | 'light'
}
```

- Filename: `timezone-planner-backup-YYYY-MM-DD.json`
- Uses `Blob` + `URL.createObjectURL` for download

### Import

```javascript
importDataFromFile(file):
1. Parse JSON
2. Validate data.state exists
3. Run migrateImportedData(data, version) migration chain
4. Build importedState with sanitized data
5. Write to IDB: __idb.set('tz_main_state', importedState)
6. Write alert state to IDB (if present)
7. Save theme to localStorage (if present)
8. Verify write by re-reading from IDB
9. Set localStorage('tz_skip_welcome', '1')
10. Show success toast with counts
11. Reload page after 1.5s delay
```

### Migration Chain

```javascript
migrateImportedData(data, fromVersion):
  migrations = {
    0: function(d) {
      // 0 → 1: Add subtasks array to events, notes array, calendarSettings, eventSettings defaults
      events.forEach(ev => {
        if (!ev.subtasks) ev.subtasks = [];
        if (!ev.notes) ev.notes = [];
      });
      if (!d.state.calendarSettings) d.state.calendarSettings = {};
      if (!d.state.eventSettings) d.state.eventSettings = {};
      return d;
    }
    // Future: 1: function(d) { /* v1→v2 */ return d; }
  };
  // Runs migrations sequentially: 0→1→...→current
```

---

## 12. Data Models

### Main State Structure

```javascript
{
  baseZone: string,                    // IANA timezone (e.g., "Asia/Manila")
  baseDisplay: string,                 // Display name for base timezone
  cards: Card[],                       // Company/timezone cards
  calendarSettings: CalendarSettings,  // Calendar configuration
  events: Event[],                     // All user events
  eventSettings: EventSettings,        // Event type configuration
  expenseCategories: Category[],       // Expense categories
  incomeCategories: Category[],        // Income categories
  expenses: Expense[],                 // All expense/income entries
  budgets: Budget[],                   // Budget definitions
  walletCurrencies: WalletCurrencies | null  // Multi-currency config
}
```

### Default State

```javascript
{
  baseZone: Intl.DateTimeFormat().resolvedOptions().timeZone,
  baseDisplay: humanizeZone(baseZone),
  cards: [],
  calendarSettings: defaultCalendarSettings(),
  events: [],
  eventSettings: defaultEventSettings(),
  expenseCategories: DEFAULT_EXPENSE_CATEGORIES (with createdAt),
  incomeCategories: DEFAULT_INCOME_CATEGORIES (with createdAt),
  expenses: [],
  budgets: [],
  walletCurrencies: null  // initialized lazily to { defaultCurrency: 'PHP', ... }
}
```

### Category Model

```javascript
{
  id: string,              // 'cat_...' for expense, 'icat_...' for income
  name: string,            // Category display name
  emoji: string,           // Single emoji character
  enabled: boolean,        // Whether category is active
  createdAt: ISO string,   // When created
  updatedAt?: ISO string   // When last modified
}
```

### Default Expense Categories

```javascript
[
  { id: 'cat_food', name: 'Food & Dining', emoji: '🍔', enabled: true },
  { id: 'cat_transport', name: 'Transportation', emoji: '🚗', enabled: true },
  { id: 'cat_housing', name: 'Housing/Rent', emoji: '🏠', enabled: true },
  { id: 'cat_utilities', name: 'Utilities', emoji: '💡', enabled: true },
  { id: 'cat_groceries', name: 'Groceries', emoji: '🛒', enabled: true },
  { id: 'cat_entertainment', name: 'Entertainment', emoji: '🎮', enabled: true },
  { id: 'cat_health', name: 'Health/Medical', emoji: '💊', enabled: true },
  { id: 'cat_education', name: 'Education', emoji: '📚', enabled: true },
  { id: 'cat_work', name: 'Work Supplies', emoji: '👔', enabled: true },
  { id: 'cat_subscriptions', name: 'Subscriptions', emoji: '📱', enabled: true },
  { id: 'cat_travel', name: 'Travel', emoji: '✈️', enabled: true },
  { id: 'cat_misc', name: 'Miscellaneous', emoji: '🔧', enabled: true }
]
```

### Default Income Categories

```javascript
[
  { id: 'icat_salary', name: 'Salary', emoji: '💼', enabled: true },
  { id: 'icat_freelance', name: 'Freelance/Contract', emoji: '💰', enabled: true },
  { id: 'icat_bonus', name: 'Bonus', emoji: '🎁', enabled: true },
  { id: 'icat_reimbursement', name: 'Reimbursement', emoji: '💵', enabled: true },
  { id: 'icat_investment', name: 'Investment', emoji: '📈', enabled: true },
  { id: 'icat_sideproject', name: 'Side Project', emoji: '🔧', enabled: true },
  { id: 'icat_other', name: 'Other', emoji: '📦', enabled: true }
]
```

### Expense/Income Entry Model

```javascript
{
  id: string,                    // 'exp_' + Date.now()
  type: 'expense' | 'income',   // Entry type
  amount: number,                // Monetary amount
  currency: string,              // Currency symbol (e.g., '₱')
  exchangeRate: number,          // Snapshot rate at time of entry
  companyId: string | null,      // Associated company card id
  categoryId: string,            // Category id
  description: string,           // Entry description
  notes: string,                 // Additional notes
  date: 'YYYY-MM-DD',           // Entry date
  status: 'paid' | 'pending',   // Payment status
  isRecurring: boolean,          // Whether this recurs
  recurFrequency: 'daily' | 'weekly' | 'biweekly' | 'monthly' | 'yearly' | null,
  recurEndDate: 'YYYY-MM-DD' | null,  // End date for recurrence
  isReimbursable: boolean,       // Whether expense is reimbursable
  reimbursementStatus: 'submitted' | 'approved' | 'reimbursed' | 'denied' | null,
  createdAt: ISO string,
  updatedAt: ISO string
}
```

### Budget Model

```javascript
{
  id: string,                    // 'bud_' + Date.now()
  categoryId: string | null,     // null = overall budget
  amount: number,                // Budget limit amount
  period: 'daily' | 'weekly' | 'biweekly' | 'monthly' | 'yearly',
  thresholdPercent: 1-100,       // Alert threshold percentage (default: 80)
  currency: string               // Currency symbol at time of creation
}
```

### Wallet Currencies Model

```javascript
{
  defaultCurrency: string,         // Currency code (default: 'PHP')
  enabled: [{
    code: string,                  // Currency code
    symbol: string,                // Currency symbol
    rate: number,                  // Exchange rate to default currency
    mode: 'manual' | 'auto'       // How rate was set
  }],
  lastFetched: number | null       // Timestamp of last rate fetch
}
```

---

## 13. Timezone Engine

### Zone Discovery

- Primary: `Intl.supportedValuesOf('timeZone')` (modern browsers)
- Fallback: `TZ_FALLBACK` — hardcoded array of ~400 IANA zones
- Validation: `isZoneSupported(zone)` — tries creating `Intl.DateTimeFormat` with zone

### Zone Aliases (`TZ_ALIASES`)

Extensive array mapping common city names to IANA zones:
- San Diego, San Francisco, Seattle, Las Vegas, Portland → America/Los_Angeles
- Salt Lake City → America/Denver
- Phoenix, Arizona → America/Phoenix
- Dallas, Houston, Austin, New Orleans, Minneapolis → America/Chicago
- Detroit → America/Detroit
- Boston, Miami, Atlanta, Washington DC → America/New_York
- And many more international cities...

### Choice Data

```javascript
{
  key: string,         // Unique key for choice
  zone: string,        // IANA timezone
  display: string,     // Human-readable display
  isAlias: boolean,    // Whether this is an alias
  displayNorm: string, // Normalized for search
  zoneNorm: string,    // Normalized zone for search
  humanZone: string,   // Humanized zone display
  searchNorm: string,  // Combined search string
  sortText: string     // For alphabetical sorting
}
```

### Fuzzy Search Scoring

```
Exact display match: score 1
Exact zone match: score 2
Display starts with query: score 3
Zone starts with query: score 4
Token match: score 20+ (higher = later match)
```

### Suggested Choices

- Browser timezone (auto-detected)
- UTC
- Asia/Manila
- America/Los_Angeles
- America/New_York
- Europe/London
- Asia/Tokyo
- Australia/Sydney

### DST Handling

- `resolveZonedDateTime()`: resolves a date/time in a zone, handling DST gaps and ambiguous times
- `resolveTargetSourceDateTime()`: for `ignoreDst` mode — uses standard (non-DST) offset
- `getStandardOffsetMinutes()`: calculates standard offset by checking January offset

### Formatter Cache

- `formatterCache`: `Map` caching `Intl.DateTimeFormat` instances by options key
- Avoids recreating formatters for repeated timezone operations

---

## 14. Mobile & Responsive Behavior

- **Viewport meta**: `width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover`
- **Safe area insets**: `--safe-top` and `--safe-bottom` CSS variables for notch handling
- **Touch optimized**: `touch-action: manipulation`, `-webkit-tap-highlight-color: transparent`
- **Zoom prevention**: `preventZoomInteractions()` blocks pinch-zoom and double-tap zoom
- **Bottom nav auto-hide**: on calendar page scroll, nav slides down; peek strip appears
- **PWA-ready**: `apple-mobile-web-app-capable`, `mobile-web-app-capable` meta tags
- **Status bar**: `apple-mobile-web-app-status-bar-style="black-translucent"`

### Responsive Breakpoints

- **Compact mode**: `max-width: 900px` — tighter calendar grid, smaller slot rows
- **Calendar grid**: fluid column widths based on available space and card count
- **Card grid**: full-width cards on mobile
- **Font sizing**: uses `clamp()` for responsive typography

---

## 15. Accessibility & UX Details

- **Semantic HTML**: proper `button`, `input`, `select` elements
- **ARIA labels**: `aria-label` on icon-only buttons
- **ARIA expanded**: toggle buttons use `aria-expanded`
- **Focus management**: keyboard navigable picker lists
- **Color contrast**: `contrastColorForHex()` computes WCAG-compliant text colors using relative luminance
  - Threshold: luminance > 0.35 → dark text (#111827), else white (#ffffff)
- **Escape key**: closes sheets and popups
- **Click outside**: closes dropdowns and popups
- **Haptic feedback**: vibration on debug toggle and alert actions
- **Animations**: respects `prefers-reduced-motion` (implicit via transition durations)

---

## 16. CSS Design System & Styling

### Typography

- **Font family**: `'Inter', -apple-system, BlinkMacSystemFont, 'SF Pro Display', 'Segoe UI', sans-serif`
- **Font weights**: 400 (normal), 500 (medium), 600 (semibold), 700 (bold), 800 (extrabold), 900 (black)
- **Base font size**: 16px
- **Line height**: 1.4 default

### Surface Treatment

- **Glass morphism**: `backdrop-filter: blur(20px)` + semi-transparent backgrounds
- **Noise texture**: SVG filter-based grain overlay on body (dark mode only)
- **Gradient backgrounds**: layered radial gradients in dark mode
- **Border radius**: `--radius: 20px` for cards, `--radius-sm: 14px` for smaller elements, `--radius-pill: 9999px` for pills

### Transitions

- **Default**: `220ms cubic-bezier(0.32, 0.72, 0, 1)` — smooth deceleration
- **Spring**: `380ms cubic-bezier(0.32, 0.72, 0, 1)` — for opening animations
- **Slow**: `320ms cubic-bezier(0.32, 0.72, 0, 1)` — for complex transitions

### Shadows

- **Small**: `--shadow-sm` — subtle depth for cards
- **Large**: `--shadow-lg` — elevated elements, modals
- **Glow**: `--shadow-glow` — accent-colored glow for highlights

### Button Styles

- **Icon buttons** (`.icon-btn`): 36×36px, rounded, transparent background
- **Primary buttons**: accent gradient background, white text
- **Danger buttons**: red background
- **Ghost buttons**: transparent with border

### Sheet Animations

- Bottom sheets slide up with `transform: translateY(100%) → translateY(0)`
- Scrim fades in with `opacity: 0 → 1`
- Spring-like timing function for natural feel

---

## 17. Expense / Income Tracker (Wallet Page)

### Page Structure

- **Period bar**: Day / Week / Month toggle buttons
- **Period navigation**: Previous / Next with period label
- **Summary card**: Total Income, Total Expense, Net Balance
- **Pending section**: shown if any pending entries (pending income / pending expense)
- **Action toggles**: Breakdown / Company / Reimb / Budget section toggles
- **Transaction list**: chronological list of entries for current period
- **Search bar**: filter transactions by description, category, company, type, status, amount
- **Empty state**: shown when no expenses exist

### Entry Types

- **Expense**: money going out
- **Income**: money coming in
- Both share the same data model and storage array (`state.expenses`)

### Wallet Period Ranges

- **Day**: single day with offset from today
- **Week**: Sunday–Saturday week with offset
- **Month**: calendar month with offset

### Currency Conversion

- `convertWithStoredRate(amount, entry)`: uses stored exchange rate from entry, falls back to current rate
- `convertToDefaultCurrency(amount, fromCurrency)`: converts using enabled currency rates
- All summaries displayed in default currency

---

## 18. Category Management

### Category Sheet

- Tabs: Expense / Income / Currencies
- List of categories with emoji, name, enable/disable toggle
- Edit modal: change emoji and name
- Delete with confirmation (checks for entries using category)
- Add new category button

### Operations

- Create: generates `cat_` or `icat_` prefixed id + `Date.now()`
- Edit: updates name and emoji, sets `updatedAt`
- Delete: removes category, confirmation if entries exist
- Toggle: enable/disable category visibility

---

## 19. Currency Management System

### Currencies Data

40 currencies supported (`CURRENCIES_DATA`):

```
PHP, USD, EUR, GBP, JPY, AUD, CAD, CHF, CNY, HKD, SGD, KRW, INR, THB, MYR, IDR, VND, TWD, NZD, SEK, NOK, DKK, MXN, BRL, ZAR, AED, SAR, PLN, CZK, HUF, TRY, ILS, RUB, CLP, COP, PEN, ARS, BGN, RON, ISK
```

Each entry: `{ code, symbol, name }`

### Default Currency

- Default: **PHP** (Philippine Peso, ₱)
- `CURRENCY_MAP`: quick lookup object by currency code

### Wallet Currencies Initialization

When `state.walletCurrencies` is null (lazy init):
```javascript
{
  defaultCurrency: 'PHP',
  enabled: [{ code: 'PHP', symbol: '₱', rate: 1, mode: 'manual' }],
  lastFetched: null
}
```

### Rate Management

- Manual rate entry per currency
- Auto-fetch from external API (max once per hour, tracked by `lastFetched`)
- Snapshot rate stored with each expense entry at time of creation

---

## 20. Recurring Expense/Income Entries

### Recurrence Frequencies

- `daily`, `weekly`, `biweekly`, `monthly`, `yearly`

### Occurrence Generation

`generateRecurringOccurrences(entry, rangeStart, rangeEnd)`:
- Generates virtual entries for date range based on frequency
- Max 1000 iterations safety limit
- Future occurrences default to `'pending'` status
- Virtual entries have: `_isGenerated: true`, `_parentId: entry.id`
- IDs: `{parentId}_r_{YYYY-MM-DD}`

---

## 21. Breakdown View

- Toggleable section in wallet page
- Company filter dropdown (all companies or specific)
- Category-by-category breakdown showing:
  - Income per category
  - Expense per category
  - Reimbursable amounts
- Visual bars showing relative amounts

---

## 22. Company Net Summary

- Toggleable section in wallet page
- Per-company income vs expense summary
- Shows net balance per company
- Includes "Other" category for entries without company

---

## 23. Reimbursable View

- Toggleable section in wallet page
- Lists all reimbursable expense entries
- Shows reimbursement status (submitted/approved/reimbursed/denied)
- Summary totals

---

## 24. Budget Management

### Budget View

- Toggleable section in wallet page
- List of budgets with:
  - Category name (or "Overall")
  - Period label
  - Spent vs budget amounts
  - Progress bar with percentage
  - Visual indicator: green (under threshold), yellow (near threshold), red (exceeded)

### Budget Edit Modal

- Category dropdown (nullable = overall)
- Amount input
- Period dropdown (daily/weekly/biweekly/monthly/yearly)
- Threshold percentage input (1-100, default: 80)
- Duplicate prevention: can't have same category + period combo

---

## 25. Alert Engine — Expense Integration

`checkExpenseAlerts()` runs as part of the alert check cycle.

### Alert Types

1. **Recurring Due**: recurring expense due today
2. **Overdue**: pending expenses past their date
3. **Budget Threshold**: spending reached threshold percentage
4. **Budget Exceeded**: spending exceeded budget
5. **Reimbursement Reminder**: reimbursable expense pending for N days

---

## 26. Sound Engine

All sounds synthesized via **Web Audio API** — no audio files.

### Sound Types (5)

#### Chime
- 3-note arpeggio: C5 (523.25Hz), E5 (659.25Hz), G5 (783.99Hz)
- Sine wave oscillator
- Notes staggered: 0s, 0.15s, 0.3s
- Each note: 1.4s exponential decay
- Individual gain per note: `soundVolume * 0.5`

#### Bell
- Base frequency: 440Hz
- Harmonics: 880Hz, 1318Hz, 1760Hz (in addition to base)
- Double strike: first at t=0, second at t=0.3 (attenuated)
- ~2s total decay
- Each harmonic at progressively lower gain

#### Pulse
- 3 rhythmic pulses at 0.3s intervals
- Sine wave sweeping 330→440Hz per pulse
- 0.15s duration per pulse
- Gain: `soundVolume * 0.5`

#### Digital
- Two-tone beep pattern: 880Hz / 660Hz
- Square wave oscillator
- Pattern: high-low alternating beeps
- Short durations (0.08-0.12s per beep)

#### Siren
- Part 1 (0-1s): Sawtooth wave 440→880→440Hz sweep
  - Gain: `soundVolume * 0.6`
- Part 2 (1.1-2.1s): Square wave 600→1000→600Hz sweep
  - Gain: `soundVolume * 0.5`
- Total: ~2.1s

### Sound Control

- `AlertSound.play(type)`: plays specified sound type
- `AlertSound.stop()`: stops all active oscillators and zeros gains
- `AlertSound.getCtx()`: lazily creates/resumes AudioContext
- Active oscillators tracked in `activeOscillators[]` array
- Gain nodes tracked in `gainNodes[]` array
- Legacy aliases: `playChime()` → `_playChime()`, `playAlarm()` → `_playSiren()`

---

## 27. Debug Console

Hidden developer console activated by secret password.

### Activation

- **Secret password**: type the secret in any input field to toggle
- Default secret: `p@ssw0rd`
- Secret stored in: `localStorage.getItem('__debugSecret')`
- Secret change: available in debug Tools tab
- Detection: monitors `input` events, checks if value ends with secret string
  - Strips secret from input value after detection
  - Calls `window.__toggleDebug()`

### State

- `window.__debugMode`: boolean flag
- `window.__debugSecret`: current secret string
- Debug FAB (🐛): fixed button at bottom-left, visible only when debug mode on
  - `.visible` class shows FAB
  - `.has-errors` class: red styling when errors exist

### Debug Panel

Full-screen panel with:
- **Header**: title + Close/Clear/Copy buttons
- **Tabs**: All, Errors, Warns, Logs, Tools
  - Each tab shows count badge
  - Active tab styling: green (all/logs), red (errors), orange (warns)
- **Body**: scrollable log entries
  - `.log-entry`: green text
  - `.error-entry`: red text
  - `.warn-entry`: orange text
  - Entry format: `[HH:MM:SS.mmm] message`
- **Font**: `'SF Mono', 'Fira Code', 'Courier New', monospace` at 12px

### Interceptors

- `console.error` → captured + original called
- `console.warn` → captured + original called
- `console.log` → captured + original called
- `window.onerror` → captured as error entry

### Tools Tab

1. **Export Data** — triggers `exportAllData()`
2. **Import Data** — triggers file input for import
3. **Share via QR** — compresses data and generates QR code
4. **Scan QR** — opens camera scanner to import data
5. **Delete All Events** — clears all events
6. **Change Debug Password** — input field with save button
7. **Clear All Data** — wipes IDB and localStorage
8. **Exit Debug Mode** — turns off debug mode

---

## 28. Toast System

In-app notification toasts replacing `alert()`.

### API

```javascript
window.__appToast(message, type, duration)
```

- **message**: string (HTML escaped, `\n` → `<br>`)
- **type**: `'success'` | `'error'` | `'info'` (default: `'info'`)
- **duration**: milliseconds (default: 4000)

### Display

- Tray: `#appToastTray` — fixed at top of screen, below safe area
- Each toast:
  - Icon: ✅ (success), ❌ (error), ℹ️ (info)
  - Text content
  - Auto-dismiss after duration
  - `.dismissing` class triggers exit animation

### Styling

- **Success**: `background: #065f46; color: #d1fae5`
- **Error**: `background: #7f1d1d; color: #fecaca`
- **Info**: `background: #1e3a5f; color: #bfdbfe`

### Animations

- `toastIn`: slide down from -20px + fade in (0.3s)
- `toastOut`: slide up to -20px + fade out (0.3s)

---

## 29. QR Sync

Share and import data via QR codes using CDN libraries.

### Dependencies

- `qrcode@1.4.4` — generates QR code images
- `lz-string@1.5.0` — compresses/decompresses data for QR payload
- `html5-qrcode@2.3.8` — camera-based QR code scanner

### Share via QR

1. Serialize data to JSON
2. Compress with `LZString.compressToEncodedURIComponent()`
3. Generate QR code image from compressed string
4. Display QR code for scanning

### Scan QR

1. Open camera scanner via `html5-qrcode`
2. Read QR code content
3. Decompress with `LZString.decompressFromEncodedURIComponent()`
4. Parse JSON and import data

### Data Chunking

Large datasets may be split into multiple QR codes:
- Expenses chunked separately with metadata: `{ type: 'expenses', part: X, total: Y, items: [...] }`

---

## 30. SVG Icon & Shape System

All icons are inline SVG `<symbol>` elements in a hidden `<svg>` container, referenced via `<use href="#icon-...">` or `<use href="#shape-...">`.

### Icon Symbols (42 total)

| Category | Icons |
|----------|-------|
| Actions | plus, edit, check, close, trash |
| Navigation | globe, up, down, chevrons-up |
| Theme | moon, sun |
| Pages | home, calendar, notes, wallet |
| UI | sliders, bell, repeat, list |
| Event Types | briefcase, task, alert |
| Communication | clock2, flag, message, phone, video, mail |
| Objects | star, file, dollar, link |
| People | user, users |
| Misc | target, check-circle, spark, pin, wrench, rocket, shield-check, bug, headset, document |

### Shape Symbols (12 total)

```
square, circle, diamond, triangle, hexagon, octagon, pill, bookmark, shield, tag, star, burst
```

Each shape is a filled SVG path within a 24×24 viewBox.

### Shape Options Array

```javascript
const SHAPE_OPTIONS = [
  ['square', 'Square'], ['circle', 'Circle'], ['diamond', 'Diamond'],
  ['triangle', 'Triangle'], ['hexagon', 'Hexagon'], ['octagon', 'Octagon'],
  ['pill', 'Pill'], ['bookmark', 'Bookmark'], ['shield', 'Shield'],
  ['tag', 'Tag'], ['star', 'Star'], ['burst', 'Burst']
];
```

### Icon Options Array

```javascript
const ICON_OPTIONS = [
  ['briefcase', 'Briefcase'], ['task', 'Task'], ['bell', 'Bell'],
  ['alert', 'Alert'], ['calendar', 'Calendar'], ['clock2', 'Clock'],
  ['flag', 'Flag'], ['message', 'Message'], ['phone', 'Phone'],
  ['video', 'Video'], ['mail', 'Mail'], ['star', 'Star'],
  ['file', 'File'], ['dollar', 'Dollar'], ['link', 'Link'],
  ['user', 'User'], ['target', 'Target'], ['globe', 'Globe'],
  ['home', 'Home'], ['list', 'List'], ['check-circle', 'Check circle'],
  ['spark', 'Spark'], ['pin', 'Pin'], ['wrench', 'Wrench'],
  ['rocket', 'Rocket'], ['shield-check', 'Shield'], ['users', 'Users'],
  ['bug', 'Bug'], ['headset', 'Headset'], ['document', 'Document']
];
```

---

## 31. Marker System

Event markers are rendered in the calendar grid and event list to identify event types.

### Marker Rendering

```javascript
renderEventMarker(type, colorHex, eventLike, mode, multiSlotNumber):
  mode: 'grid' | 'popup' | 'preview'
  markerKind: 'shape' | 'icon'  (from event type)
```

### Marker Behavior

- **Shape markers**: filled SVG shape with `--marker-color` CSS variable
- **Icon markers**: stroked SVG icon with `--marker-color` CSS variable
- **Completed events**: forced gray color (`#94a3b8`)
- **Own-lane events**: special `.on-own-lane` class for contrast handling
- **Multi-slot numbering**: events spanning multiple time slots get sequential numbers
  - Uses `_multiSlotNumbers` Map
  - Numbered markers show `data-number` attribute with shape background

### Conflict Severity Indicators

Applied to markers in the calendar grid:
- **`warning`**: single conflict (one overlapping event)
- **`danger`**: multiple conflicts (two or more overlapping events)

### Color Handling

- `effectiveColorHex(value)`: ensures valid hex color, defaults to `#38bdf8`
- `contrastColorForHex(hex)`: returns `#111827` (dark) or `#ffffff` (white) based on luminance
- `computeInverseColor(hex)`: simpler contrast check using weighted brightness

---

## 32. Conflict Detection

### Algorithm

`computeConflictPairs(items)`:
- Compares all event pairs in a time slot
- Skips completed/cancelled events
- Checks overlap: `a.sortStart < b.sortEnd && b.sortStart < a.sortEnd`
- Same-day events only (`a.dayKey === b.dayKey`)
- Detects cross-lane conflicts: `a.laneCompanyId !== b.laneCompanyId`
- Returns pairs with: `{ ids, primary, secondary, range, crossLane }`

### Conflict Display

- `getConflictSummaries(items)`: returns top 6 conflict pairs for display
- Events page: conflict banner at top showing human-readable conflict descriptions
- Calendar cells: `has-conflict` and `conflict-danger` CSS classes
- Conflict indicator: label text or dot (configurable in calendar settings)
- `autoFitConflictLabels()`: dynamically shrinks conflict label text to fit available cell width

### Cross-Lane Detection

`getScheduleSlotEvents()`:
- Collects ALL events in a time slot (across all lanes)
- Runs `computeConflictPairs()` on all slot events
- Assigns `conflictSeverity` per event: `'danger'` (>1 conflict), `'warning'` (1 conflict), `''` (none)

---

## Appendix: UI State Object

```javascript
const ui = {
  editing: Set,            // Card IDs currently in edit mode
  drafts: {},              // Edited card drafts by ID
  unsavedIds: Set,         // Cards with unsaved changes
  currentPage: 'home',    // Current active page
  eventsContext: null,     // Events page filter context
  eventDraft: null,        // Current event being edited
  pageDate: '',            // Current date string
  sheet: '',               // Currently open sheet name
  picker: {                // Timezone picker state
    type: '',              // 'base' | 'card' | 'new'
    cardId: '',
    activeIndex: 0
  },
  pendingDeleteId: '',     // Item pending deletion
  pendingDeleteType: '',   // Type of pending deletion
  newDraft: null,          // New card draft
  calendarDraft: null,     // Calendar settings draft
  autoSwipeTimer: null,    // Card auto-swipe interval
  autoSwipeResumeTimer: null,
  lastScrollY: 0,          // For nav auto-hide
  navPeekStartY: null,     // Touch tracking for nav peek
  calendarWeekOffset: 0,   // Week navigation offset
  calendarDayId: '',       // Selected day in calendar
  scheduleSnapTimer: null,
  scheduleLastScrollLeft: 0,
  scheduleLastScrollTop: 0,
  walletCatTab: 'expense', // Category tab
  catEditId: null,         // Category being edited
  walletPeriod: 'month',   // Wallet period mode
  walletPeriodOffset: 0,   // Period navigation offset
  expenseEditId: null,     // Expense being edited
  walletBreakdownOpen: false,
  walletBreakdownCompany: '',
  walletCompanySummaryOpen: false,
  walletReimbOpen: false,
  walletBudgetOpen: false,
  expenseType: 'expense'   // Current expense form type
};
```

---

## Appendix: Vibration Patterns

```javascript
AlertVibrate = {
  normal:   [200, 100, 200, 100, 200],                    // 3 short buzzes
  annoying: [500, 100, 500, 100, 500, 100, 500, 100, 500] // 5 long buzzes
};
```

---

## Appendix: TTS Engine

```javascript
AlertTTS = {
  speak(text, loop):
    - Creates SpeechSynthesisUtterance
    - Rate: alertState.ttsRate (default: 1.0)
    - Pitch: 1.0
    - Volume: min(1, alertState.soundVolume + 0.3)
    - Loop mode: 1800ms gap between repetitions
  stop():
    - Cancels speech synthesis
    - Clears loop timer
};
```

TTS start is delayed by 1.5s after alert popup to allow sound to play first (`_ttsStartTimer`).

---

## Appendix: Key localStorage Items

| Key | Purpose | Notes |
|-----|---------|-------|
| `timezone_planner_theme_v1` | Theme preference | `'dark'` or `'light'` |
| `tz_skip_welcome` | Skip welcome gate | `'1'` to skip |
| `__debugSecret` | Debug console password | Default: `'p@ssw0rd'` |

## Appendix: Key IDB Items

| Key | Store | Purpose |
|-----|-------|---------|
| `tz_main_state` | `kv` | All app state (cards, events, settings, wallet) |
| `tz_alert_state` | `kv` | Alert engine configuration |
| `tz_notifications` | `kv` | Notification center entries |
