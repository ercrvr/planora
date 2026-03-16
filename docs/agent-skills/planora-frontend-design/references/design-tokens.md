# Planora Design Tokens

> Extracted from `planora-prod.html` (~13,116 lines). All line numbers reference that file.

---

## Table of Contents

1. [CSS Custom Properties](#1-css-custom-properties-design-tokens)
   - [Backgrounds & Surfaces](#backgrounds--surfaces)
   - [Text Colors](#text-colors)
   - [Accent & Semantic Colors](#accent--semantic-colors)
   - [Borders](#borders)
   - [Shadows](#shadows)
   - [Border Radius](#border-radius)
   - [Transitions](#transitions)
   - [Safe Area Insets](#safe-area-insets-ios)
   - [Component-Specific Variables](#component-specific-variables)
   - [Dynamic Per-Element Variables](#dynamic-per-element-variables)
2. [Color System](#2-color-system)
   - [Dark vs Light Theme Comparison](#dark-vs-light-theme-comparison)
   - [Hard-Coded Semantic Colors](#hard-coded-semantic-colors)
   - [Theme Switching Mechanism](#theme-switching-mechanism)
   - [Background Effects](#background-effects-dark-mode-only)
3. [Typography](#3-typography)
   - [Font Family](#font-family-line-128)
   - [Font Size Scale](#font-size-scale)
   - [Font Weight Usage](#font-weight-usage)
   - [Line Height Patterns](#line-height-patterns)
   - [Letter Spacing Patterns](#letter-spacing-patterns)
   - [Text Gradient Effect](#text-gradient-effect-line-280)
4. [Naming Conventions](#12-naming-conventions)
   - [CSS Class Naming](#css-class-naming)
   - [ID Naming](#id-naming)
   - [Data Attributes](#data-attributes-55-unique)

---

## 1. CSS Custom Properties (Design Tokens)

All design tokens are declared in `:root` (lines 21–57) and overridden for light theme via `html[data-theme="light"]` (lines 60–75).

### Backgrounds & Surfaces
```css
:root {
  --bg: #080d1a;
  --bg-gradient: radial-gradient(ellipse at 20% 0%, rgba(56, 189, 248, 0.06) 0%, transparent 50%),
                  radial-gradient(ellipse at 80% 100%, rgba(139, 92, 246, 0.05) 0%, transparent 50%),
                  radial-gradient(circle at 50% 20%, #151d2e, #080d1a 70%);
  --surface: rgba(22, 33, 55, 0.55);
  --surface-hover: rgba(35, 50, 78, 0.72);
  --surface-solid: #131c2e;
}
```

### Text Colors
```css
:root {
  --text: #f1f5f9;
  --text-muted: #94a3b8;
  --text-subtle: #64748b;
}
```

### Accent & Semantic Colors
```css
:root {
  --accent: #38bdf8;           /* Sky blue - primary accent */
  --accent-hover: #7dd3fc;
  --accent-gradient: linear-gradient(135deg, #38bdf8, #22d3ee);
  --danger: #f87171;
  --danger-soft: rgba(248, 113, 113, 0.12);
}
```

### Borders
```css
:root {
  --border: rgba(148, 163, 184, 0.07);
  --border-strong: rgba(15, 23, 42, 0.11);  /* Note: in dark, slate-based alpha */
}
```

### Shadows
```css
:root {
  --shadow-sm: 0 2px 8px -2px rgba(0, 0, 0, 0.2), 0 4px 16px -4px rgba(0, 0, 0, 0.15);
  --shadow-lg: 0 12px 40px -8px rgba(0, 0, 0, 0.4), 0 4px 16px -4px rgba(0, 0, 0, 0.25);
  --shadow-glow: 0 0 24px rgba(56, 189, 248, 0.25), 0 0 48px rgba(56, 189, 248, 0.08);
}
```

### Border Radius
```css
:root {
  --radius: 20px;        /* Primary radius for cards */
  --radius-sm: 14px;     /* Secondary radius for inputs, smaller elements */
  --radius-pill: 9999px; /* Pill/circle shape */
}
```

### Transitions
```css
:root {
  --transition: 220ms cubic-bezier(0.32, 0.72, 0, 1);
  --transition-spring: 380ms cubic-bezier(0.32, 0.72, 0, 1);
  --transition-slow: 320ms cubic-bezier(0.32, 0.72, 0, 1);
}
```
> **Key Pattern**: All three transitions use the same custom easing `cubic-bezier(0.32, 0.72, 0, 1)` — a "spring-like" curve that overshoots slightly. This is the app's signature easing.

### Safe Area Insets (iOS)
```css
:root {
  --safe-top: env(safe-area-inset-top, 0px);
  --safe-bottom: env(safe-area-inset-bottom, 0px);
}
```

### Component-Specific Variables
The schedule/calendar page defines its own CSS custom properties (lines 477–487, 843–845):
```css
--schedule-time-col-width: 92px;      /* 74px on mobile */
--schedule-company-col-width: 48px;
--schedule-date-row-height: 0px;      /* 40px on mobile */
--schedule-company-row-height: 110px;
--schedule-slot-row-height: 28px;     /* 26px on mobile */
--schedule-fab-safe: 116px;
--schedule-today-color: var(--accent);
--schedule-today-thickness: 3px;
--schedule-now-color: var(--accent);
--schedule-now-thickness: 2px;
```

### Dynamic Per-Element Variables
Set via inline styles in JS:
- `--swatch` — color swatch value for color pickers
- `--marker-color` — schedule marker colors
- `--company-color` — company-specific colors
- `--slot-color` — slot lane colors
- `--owner-color` — event owner card color
- `--event-marker-color` — event type marker color
- `--ci-dot-color` — conflict indicator dot color

---

## 2. Color System

### Dark vs Light Theme Comparison

| Token | Dark (Default, Lines 21–57) | Light (Lines 60–75) |
|-------|---------------------------|-------------------|
| `--bg` | `#080d1a` (deep navy) | `#f0f4f8` (light cool gray) |
| `--surface` | `rgba(22, 33, 55, 0.55)` (translucent navy) | `rgba(255, 255, 255, 0.78)` (translucent white) |
| `--surface-hover` | `rgba(35, 50, 78, 0.72)` | `rgba(255, 255, 255, 0.95)` |
| `--surface-solid` | `#131c2e` (opaque navy) | `#f8fafc` (near-white) |
| `--text` | `#f1f5f9` (near-white) | `#0f172a` (near-black) |
| `--text-muted` | `#94a3b8` (slate) | `#475569` (darker slate) |
| `--text-subtle` | `#64748b` (dim slate) | `#64748b` (same) |
| `--accent` | `#38bdf8` (sky blue) | `#38bdf8` (same) |
| `--accent-hover` | `#7dd3fc` | `#7dd3fc` (same) |
| `--danger` | `#f87171` (red) | `#f87171` (same) |
| `--border` | `rgba(148, 163, 184, 0.07)` | `rgba(15, 23, 42, 0.06)` |
| `--border-strong` | `rgba(15, 23, 42, 0.11)` | `rgba(15, 23, 42, 0.11)` (same) |

### Hard-Coded Semantic Colors
Used across both themes — not controlled by CSS custom properties:
```css
/* Success/Income */ #22c55e (green-500)
/* Danger/Expense */ #ef4444 (red-500)
/* Warning */        #f59e0b (amber-500), #f97316 (orange-500)
/* Purple accent */  #8b5cf6 (violet-500), #764ba2
/* Budget section */ #8b5cf6 (purple)
/* Reimbursable */   #f59e0b (amber)
```

### Theme Switching Mechanism
- **Data attribute**: `html[data-theme="dark"|"light"]`
- **Script at line 11–18**: Applies theme **before CSS paints** to prevent flash:
```javascript
try {
  var t = localStorage.getItem('timezone_planner_theme_v1') === 'light' ? 'light' : 'dark';
  document.documentElement.setAttribute('data-theme', t);
} catch (_) {
  document.documentElement.setAttribute('data-theme', 'dark');
}
```
- Default is **dark** theme
- Toggle button: `#themeToggle` (line 3771)
- Light theme forces flat background and kills noise texture:
```css
html[data-theme="light"] {
  background: #f0f4f8 !important;
  background-image: none !important;
  background-attachment: scroll !important;
}
html[data-theme="light"]::before {
  display: none !important; /* kills noise texture */
}
```

### Background Effects (Dark Mode Only)
- Multi-layer radial gradient background (`--bg-gradient`)
- Subtle SVG noise texture overlay at 1.8% opacity (line 139–148)

---

## 3. Typography

### Font Family (Line 128)
```css
font-family: "Inter", -apple-system, BlinkMacSystemFont, "SF Pro Text", "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
```
Google Fonts import: `Inter` weights 400, 500, 600, 700, 800, 900 (line 10).

**Exception**: Debug console uses `monospace` font families (line 3313).

### Font Size Scale
The app uses a mix of `rem`, `px`, and `clamp()` values. Key sizes:

| Context | Size | Weight |
|---------|------|--------|
| Topbar time | `clamp(2.6rem, 9.6vw, 4.2rem)` | 800 |
| Topbar date | `clamp(1.05rem, 2.8vw, 1.28rem)` | 700 |
| Page titles | `1.4rem` | 800 |
| Section titles | `1.08rem–1.15rem` | 800 |
| Card live clock | Uses gradient text | 800 |
| Body text | `0.9rem–1rem` | 600 |
| Labels/kickers | `0.76rem–0.78rem` | 700, uppercase, letter-spacing: 0.06–0.12em |
| Helper text | `0.78rem–0.82rem` | 500 |
| Small text | `0.72rem–0.75rem` | 600–700 |
| Chip/badge text | `0.65rem–0.78rem` | 700 |
| Nav button labels | `0.68rem` | 700 |

### Font Weight Usage
| Weight | Usage |
|--------|-------|
| 400 | Welcome subtitle, minor body text |
| 500 | Helper text, context menu items, toast text |
| 600 | Body text, labels, nav buttons |
| 700 | Headings, buttons, labels, tags |
| 800 | Primary headings, card clocks, values, page titles |
| 900 | Topbar time display |

### Line Height Patterns
- Tight: `0.92` (topbar time)
- Compact: `1.1`–`1.3` (headings, small text)
- Default: `1.35`–`1.4` (body text)
- Relaxed: `1.5`–`1.55` (notes, longer text)

### Letter Spacing Patterns
- Tight: `-0.06em` (large headings), `-0.02em` to `-0.01em` (titles)
- Normal: `0` (body)
- Loose: `0.04em`–`0.06em` (uppercase labels)
- Wide: `0.1em`–`0.12em` (topbar date, kickers)

### Text Gradient Effect (Line 280)
```css
.topbar-time {
  background: linear-gradient(180deg, var(--text) 40%, var(--text-muted) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}
```

---

## 12. Naming Conventions

### CSS Class Naming
**Pattern**: Lowercase, hyphen-separated — a custom convention (not strict BEM but BEM-inspired).

- **Component**: `.card`, `.sheet`, `.fab`, `.input`
- **Component-element**: `.card-head`, `.card-toolbar`, `.sheet-panel`, `.sheet-handle`
- **Component-modifier**: `.card-edit`, `.card-head-view`, `.ghost-btn.mini`
- **State**: `.active`, `.open`, `.hidden`, `.is-completed`, `.is-end`, `.dismissing`, `.has-count`, `.has-errors`
- **Variant**: `.primary`, `.danger`, `.compact`

**Prefixed groups**:
- `event-*` (35 classes) — Event cards & event-related UI
- `alert-*` (56 classes) — Alert engine UI
- `card-*` (17 classes) — Company cards
- `cat-*` (19 classes) — Category management
- `wallet-*` (many) — Wallet page
- `schedule-*` (many) — Calendar/schedule
- `notif-*` (15 classes) — Notification center
- `compact-*` (11 classes) — Compact variants
- `debug-*` (9 classes) — Debug console

### ID Naming
- **camelCase**: `pageHome`, `mainFab`, `themeToggle`, `walletSummaryCard`
- **Pattern**: `{section}{Element}` or `{element}{Action}Btn`
- Examples: `navCalendar`, `alertBellBtn`, `closePicker`, `expAmountInput`

### Data Attributes (55 unique)
Common patterns:
- `data-card-id`, `data-event-id`, `data-entry-id` — Entity IDs
- `data-action`, `data-sap-action` — Click action handlers
- `data-field` — Form field binding
- `data-live-clock`, `data-live-date`, `data-live-zone` — Live-updating elements
- `data-section`, `data-period` — Section toggle identifiers
- `data-cal-tab`, `data-alert-tab`, `data-debug-tab` — Tab system identifiers
- `data-theme` — Theme indicator on `<html>`
