# Planora Animations, States & Layout Reference

> Extracted from `planora-prod.html` (~13,116 lines). All line numbers reference that file.

---

## Table of Contents

1. [Animation & Transitions (Section 8)](#8-animation--transitions)
   - [CSS Keyframe Animations (22 total)](#css-keyframe-animations-22-total)
   - [Signature Easing](#signature-easing)
   - [Transition Patterns](#transition-patterns)
   - [Active/Press Feedback](#activepress-feedback)
2. [State-Based Styling (Section 10)](#10-state-based-styling)
   - [Active/Selected State](#activeselected-state)
   - [Hover Patterns](#hover-patterns)
   - [Focus Patterns](#focus-patterns)
   - [Disabled States](#disabled-states)
   - [Loading States](#loading-states)
   - [Completed State](#completed-state)
   - [Dismissing State](#dismissing-state)
   - [Alert Glow States](#alert-glow-states)
3. [Responsive Design (Section 7)](#7-responsive-design)
   - [Breakpoints](#breakpoints-desktop-first-approach)
   - [Key Responsive Changes at 768px](#key-responsive-changes-at-768px)
   - [Touch-Friendly Sizing](#touch-friendly-sizing)
4. [Z-Index System (Section 11)](#11-z-index-system)

---

## 8. Animation & Transitions

### CSS Keyframe Animations (22 total)

| Animation | Usage | Duration/Easing |
|-----------|-------|-----------------| 
| `fadeInUp` | Cards entering | `400ms ease` — opacity + translateY |
| `fadeIn` | Pages entering | `400ms ease-out` — opacity only |
| `pulseGlow` | Subtle glow | `∞` — box-shadow pulse |
| `slideUp` | Elements sliding in | opacity + translateY(100%) |
| `clockPulse` | Clock separator blink | `∞` — opacity 1→0.6 |
| `shimmer` | Loading shimmer | background-position sweep |
| `iconRotateIn` | Icon rotation entrance | rotate(-90deg) + scale(0.7) → normal |
| `fabPulse` | FAB idle glow | `4s ease-in-out ∞` — box-shadow |
| `borderGlow` | Border color pulse | `∞` — border-color 0.15→0.3 |
| `schedPopIn` | Popup entrance | scale(0.92) → scale(1) |
| `alertSlideDown` | Alert toast entrance | `420ms spring` — translateY(-16px) + scale |
| `alertPopIn` | Alert popup entrance | `420ms spring` — scale(0.88) → 1 |
| `alertBellSwing` | Bell icon shake | multi-step rotation |
| `alertBadgePulse` | Notification badge | scale + opacity pulse |
| `alertAnnoyBg` | Annoying alert bg | background color flash |
| `alertAnnoyShake` | Annoying alert shake | multi-step translateX |
| `cardAlertPulse` | Active card glow (green) | border + box-shadow |
| `cardAlertPulseEnd` | Ending card glow (amber) | border + box-shadow |
| `toastIn` | Toast entrance | `.3s ease-out` — translateY + opacity |
| `toastOut` | Toast exit | `.3s ease-in` — translateY + opacity |
| `welcomeFadeUp` | Welcome screen entrance | `1s ease-out` — translateY(30px) |
| `welcomePulseRing` | Welcome icon ring | `2.5s ease-in-out ∞` |
| `welcomeTapPulse` | Welcome tap indicator | `2s ease-in-out ∞` |

### Signature Easing
```css
cubic-bezier(0.32, 0.72, 0, 1)  /* Used everywhere — spring-like feel */
```

### Transition Patterns
- **Default**: `var(--transition)` = `220ms cubic-bezier(0.32, 0.72, 0, 1)`
- **Spring**: `var(--transition-spring)` = `380ms` same curve
- **Slow**: `var(--transition-slow)` = `320ms` same curve
- **Common combo**: `all var(--transition)` for simple elements
- **Specific properties**: Usually `background`, `border-color`, `transform`, `opacity`
- **Sheets**: `420ms` for sliding in/out
- **Nav hide/show**: `300ms` for translateY + opacity

### Active/Press Feedback
Nearly all interactive elements have `:active { transform: scale(0.9–0.97) }`.

---

## 10. State-Based Styling

### Active/Selected State
```css
/* Nav button active */
.nav-btn.active {
  background: rgba(56, 189, 248, 0.12);
  color: var(--accent);
  box-shadow: inset 0 0 0 1px rgba(56, 189, 248, 0.2);
}

/* Tab active */
.cat-tab.active { background: var(--bg-primary); color: var(--text-primary); box-shadow: 0 1px 4px rgba(0,0,0,0.1); }

/* Segment active */
.segment-btn.active {
  background: rgba(56, 189, 248, 0.14); color: var(--accent);
  box-shadow: inset 0 0 0 1px rgba(56, 189, 248, 0.25), 0 2px 8px rgba(56, 189, 248, 0.08);
}

/* Alert mode active */
.alert-mode-card.active { border-color: var(--accent); background: rgba(56,189,248,0.06); }

/* Wallet period active */
.wallet-period-btn.active { background: var(--accent); color: #fff; box-shadow: 0 2px 8px rgba(56,189,248,0.3); }
```

### Hover Patterns
Common hover pattern: lighten background, strengthen border, elevate (transform).
```css
/* Card hover — elevation change */
.card:hover { border-color: rgba(148,163,184,0.14); box-shadow: var(--shadow-lg); transform: translateY(-2px) scale(1.005); }

/* Button hover — background fill */
.icon-btn:hover { background: rgba(255,255,255,0.08); color: var(--text); border-color: var(--border-strong); }

/* List item hover — subtle bg */
.settings-ctx-item:hover { background: var(--surface-hover); }
```

### Focus Patterns
```css
.input:focus {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(56, 189, 248, 0.12), 0 0 16px rgba(56, 189, 248, 0.06);
  background: rgba(0, 0, 0, 0.25);
}
```

### Disabled States
```css
.icon-btn:disabled { opacity: 0.3; cursor: not-allowed; transform: none; box-shadow: none; }
.ghost-btn:disabled { opacity: 0.3; cursor: not-allowed; }
.action-btn:disabled { opacity: 0.45; cursor: not-allowed; transform: none; }
.cat-item-name.disabled { opacity: 0.4; text-decoration: line-through; }
```

### Loading States
- Shimmer animation defined (`@keyframes shimmer`) for loading placeholders
- Clock pulse animation for live time displays

### Completed State
```css
.event-card.is-completed { opacity: 0.78; border-color: rgba(148,163,184,0.3); }
```

### Dismissing State
For toasts and notifications:
```css
.notif-item.dismissing { opacity: 0; transform: translateX(60px); transition: opacity 280ms ease, transform 280ms ease; }
.alert-toast.dismissing { opacity: 0; transform: translateX(100%) scale(0.95); }
.app-toast.dismissing { animation: toastOut .3s ease-in forwards; }
```

### Alert Glow States
```css
.card.card-alert-glow {
  border-color: #22c55e !important;
  animation: cardAlertPulse 2s ease-in-out infinite;
}
.card.card-alert-glow-end {
  border-color: #f59e0b !important;
  animation: cardAlertPulseEnd 2s ease-in-out infinite;
}
```

---

## 7. Responsive Design

### Breakpoints (Desktop-first approach)

| Breakpoint | Target | Lines |
|------------|--------|-------|
| `@media (max-width: 900px)` | Tablet — schedule layout adjustments | 841 |
| `@media (max-width: 768px)` | Mobile — main breakpoint, carousel cards | 1647 |
| `@media (max-width: 760px)` | Mobile — wallet layout adjustments | 2432 |
| `@media (max-width: 720px)` | Mobile — alert settings layout | 2360 |
| `@media (max-width: 640px)` | Small mobile — schedule further compacted | 851 |
| `@media (orientation: landscape) and (max-height: 540px)` | Landscape phones | 2053 |
| `@media (hover: none)` | Touch devices — show close buttons | 2671 |

### Key Responsive Changes at 768px
- Cards become horizontal snap carousel (swipeable):
  ```css
  .grid {
    display: flex; flex-wrap: nowrap; overflow-x: auto;
    scroll-snap-type: x mandatory; scroll-behavior: smooth;
    -webkit-overflow-scrolling: touch;
    margin: 0 -16px; padding: 8px 16px 24px 16px;
  }
  .card {
    flex: 0 0 calc(100vw - 64px);
    min-width: calc(100vw - 64px);
    scroll-snap-align: center;
    scroll-snap-stop: always;
  }
  ```
- FAB repositioned
- Input heights reduced (48px → 44px)
- Topbar restructured with more padding for touch

### Touch-Friendly Sizing
- Minimum touch targets: `40px × 40px` (icon buttons), `46px` min-height (nav buttons)
- `:active` states with `scale(0.9–0.97)` for tactile feedback
- `touch-action: manipulation` on html/body
- `overscroll-behavior: none` to prevent pull-to-refresh
- `-webkit-tap-highlight-color: transparent`
- `user-scalable=no` in viewport meta

---

## 11. Z-Index System

| Z-Index | Element | Purpose |
|---------|---------|---------|
| `-1` | Card `::before`, splash bg | Behind parent |
| `1–3` | Card internals, icon rows | Within card stacking |
| `4–10` | Schedule grid elements | Calendar layering |
| `40` | `.topbar` | Sticky header |
| `45` | `.bottom-nav` | Fixed bottom nav |
| `49` | `.bottom-nav-peek` | Peek button (above nav level) |
| `50` | `.fab` | Floating action button |
| `80` | `.scrim` | Overlay scrim behind sheets |
| `90` | `.sheet` | Bottom sheets |
| `200` | Schedule popup | Schedule action menu |
| `7999` | `.settings-ctx-overlay` | Behind context menu |
| `8000` | `.settings-ctx-menu` | Settings context menu |
| `9999` | `html::before` (noise), notifications | Noise texture & notification tray |
| `10000` | Alert popup overlay, QR modal | Modal overlays |
| `99998` | `.app-toast-tray` | Toast notifications |
| `99999` | Debug FAB / Debug panel | Debug tools (always on top) |
| `100000` | Welcome gate | Splash/welcome screen |
