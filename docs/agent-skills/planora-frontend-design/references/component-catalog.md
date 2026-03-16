# Planora Component Catalog

> Extracted from `planora-prod.html` (~13,116 lines). All line numbers reference that file.

---

## Table of Contents

1. [5.1 Cards](#51-cards-lines-910950)
2. [5.2 Event Cards](#52-event-cards-lines-21522172)
3. [5.3 Bottom Sheets](#53-bottom-sheets-lines-15001552)
4. [5.4 Buttons](#54-buttons)
   - [Icon Buttons](#icon-buttons-lines-9951035)
   - [FAB — Floating Action Button](#fab--floating-action-button-lines-13261365)
   - [Ghost Buttons](#ghost-buttons-lines-12961323)
   - [Action Buttons](#action-buttons-lines-15551625)
   - [Segment Buttons / Toggle Buttons](#segment-buttons--toggle-buttons-lines-11511184)
5. [5.5 Form Inputs](#55-form-inputs-lines-11151150)
6. [5.6 Toggle Switch](#56-toggle-switch-lines-12121249)
7. [5.7 Tabs](#57-tabs-multiple-variants)
8. [5.8 Bottom Navigation](#58-bottom-navigation-lines-13671484)
9. [5.9 Toasts](#59-toasts-lines-33313350)
10. [5.10 Modals / Overlays](#510-modals--overlays)
11. [5.11 Context Menu](#511-context-menu-lines-24602508)
12. [5.12 Color Swatch Picker](#512-color-swatch-picker-lines-18301860)
13. [5.13 Chips / Badges](#513-chips--badges-lines-21852195)
14. [5.14 Empty States](#514-empty-states-lines-884908-22142232)

---

## 5.1 Cards (Lines 910–950)

The primary content container. Used for company timezone cards.

### HTML Structure
```html
<article class="card" data-card-id="${id}">
  <div class="card-head card-head-view">
    <div class="card-live-block">
      <div class="card-live-label">Label</div>
      <div class="card-live-clock">12:45 PM</div>
      <div class="card-live-date">Mon, Mar 17</div>
      <div class="card-live-zone"><span>America/New_York</span></div>
    </div>
    <div class="icon-row">
      <button class="icon-btn" type="button"><svg>...</svg></button>
    </div>
  </div>
  <!-- additional content -->
</article>
```

### CSS
```css
.card {
  position: relative;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);          /* 20px */
  padding: 20px;
  backdrop-filter: blur(24px) saturate(1.2);
  box-shadow: var(--shadow-sm);
  transition: transform 300ms cubic-bezier(0.32, 0.72, 0, 1),
              box-shadow 300ms cubic-bezier(0.32, 0.72, 0, 1),
              border-color 300ms cubic-bezier(0.32, 0.72, 0, 1);
  display: flex;
  flex-direction: column;
  gap: 14px;
  animation: fadeInUp 400ms cubic-bezier(0.32, 0.72, 0, 1) both;
  overflow: hidden;
}
```

### State: Hover
```css
.card:hover {
  border-color: rgba(148, 163, 184, 0.14);
  box-shadow: var(--shadow-lg);
  transform: translateY(-2px) scale(1.005);
}
```

### Gradient Glow Pseudo-element
```css
.card::before {
  content: '';
  position: absolute; inset: -1px;
  border-radius: var(--radius);
  background: linear-gradient(135deg, rgba(56, 189, 248, 0.08), transparent 40%, transparent 60%, rgba(139, 92, 246, 0.06));
  pointer-events: none; z-index: -1;
  opacity: 0; transition: opacity 300ms ease;
}
.card:hover::before { opacity: 1; }
```

### State: Edit Mode
Edit mode adds class `.card-edit` with more padding for toolbar and edit form fields.

### State: Alert Glow
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

## 5.2 Event Cards (Lines 2152–2172)

### HTML Structure
```html
<article class="event-card" data-event-id="${id}" style="--owner-color:${color}; --event-marker-color:${markerColor}">
  <div class="event-card-head">
    <div class="event-title-wrap">
      <div class="event-title-row">
        <span class="event-title">Title</span>
        <span class="event-status-chip">Status</span>
      </div>
      <span class="event-meta">metadata</span>
    </div>
    <div class="event-card-actions">
      <button class="event-compact-btn">Edit</button>
    </div>
  </div>
  <div class="event-card-body">
    <div class="event-data"><label>FIELD</label><strong>Value</strong></div>
    <!-- grid of 2 columns -->
  </div>
</article>
```

### CSS
```css
.event-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 18px;
  padding: 14px;
  box-shadow: var(--shadow-sm);
  backdrop-filter: blur(20px) saturate(1.2);
  display: flex; flex-direction: column; gap: 10px;
  transition: transform 200ms cubic-bezier(0.32, 0.72, 0, 1),
              border-color 200ms ease, box-shadow 200ms ease;
}
```

### State: Hover
```css
.event-card:hover {
  transform: translateY(-1px);
  border-color: var(--border-strong);
  box-shadow: var(--shadow-lg);
}
```

### State: Completed
```css
.event-card.is-completed { opacity: 0.78; border-color: rgba(148,163,184,0.3); }
```

---

## 5.3 Bottom Sheets (Lines 1500–1552)

Sheets slide up from bottom, iOS-style.

### HTML Structure
```html
<section class="sheet" id="sheetName" aria-hidden="true">
  <div class="sheet-scrim" data-close-sheet></div>
  <div class="sheet-panel">
    <div class="sheet-handle"></div>
    <div class="sheet-head">
      <h2 class="sheet-title">Title</h2>
      <button class="icon-btn" type="button" aria-label="Close">
        <svg><use href="#icon-close"></use></svg>
      </button>
    </div>
    <div class="sheet-body">
      <!-- content -->
    </div>
  </div>
</section>
```

### CSS
```css
.sheet {
  position: fixed; inset: auto 0 0 0;
  z-index: 90;
  padding: 0 12px calc(12px + var(--safe-bottom));
  transform: translateY(100%);
  transition: transform 420ms cubic-bezier(0.32, 0.72, 0, 1);
  pointer-events: none;
}
```

### State: Open
```css
.sheet.open { transform: translateY(0); pointer-events: auto; }
```

### Sheet Panel
```css
.sheet-panel {
  width: min(600px, 100%);
  margin: 0 auto;
  max-height: 85vh;
  display: flex; flex-direction: column;
  border-radius: 28px 28px 20px 20px;
  border: 1px solid var(--border-strong);
  background: var(--surface-solid);
  box-shadow: 0 -12px 48px rgba(0,0,0,0.4), 0 0 1px rgba(255,255,255,0.05);
  overflow: hidden;
}
```

### Sheet Handle
```css
.sheet-handle {
  width: 40px; height: 4px;
  border-radius: 4px;
  /* drag handle indicator at top of sheet */
}
```

---

## 5.4 Buttons

### Icon Buttons (Lines 995–1035)

```css
.icon-btn {
  width: 40px; height: 40px;
  display: flex; align-items: center; justify-content: center;
  border-radius: var(--radius-pill);
  border: 1px solid var(--border);
  background: rgba(255,255,255,0.04);
  color: var(--text-muted);
  cursor: pointer;
  transition: all var(--transition);
}
```

#### State: Hover
```css
.icon-btn:hover { background: rgba(255,255,255,0.08); color: var(--text); border-color: var(--border-strong); }
```

#### State: Active (Press)
```css
.icon-btn:active { transform: scale(0.9); }
```

#### State: Disabled
```css
.icon-btn:disabled { opacity: 0.3; cursor: not-allowed; }
```

#### Variants
```css
.icon-btn.primary { border-color: rgba(56, 189, 248, 0.3); color: var(--accent); }
.icon-btn.danger { border-color: rgba(248, 113, 113, 0.3); color: var(--danger); }
```

---

### FAB — Floating Action Button (Lines 1326–1365)

#### HTML Structure
```html
<button class="fab" id="mainFab" type="button" aria-label="Actions">
  <svg><use href="#icon-plus"></use></svg>
</button>
```

#### CSS
```css
.fab {
  position: fixed; right: 20px; bottom: calc(80px + var(--safe-bottom));
  width: 64px; height: 64px;
  border-radius: var(--radius-pill);
  border: none;
  background: var(--accent);
  background-image: var(--accent-gradient);
  color: #0b0f19;
  display: flex; align-items: center; justify-content: center;
  box-shadow: var(--shadow-lg), 0 0 24px rgba(56, 189, 248, 0.25);
  z-index: 50;
  cursor: pointer;
  transition: transform 360ms cubic-bezier(0.32, 0.72, 0, 1), opacity 300ms ease, bottom var(--transition), box-shadow 300ms ease;
  animation: fabPulse 4s ease-in-out infinite;
}
```

#### State: Hover
```css
.fab:hover {
  transform: scale(1.08) translateY(-3px);
  box-shadow: var(--shadow-lg), 0 0 36px rgba(56, 189, 248, 0.4);
  animation: none;
}
.fab:hover svg { transform: rotate(90deg); }
```

#### State: Active (Press)
```css
.fab:active { transform: scale(0.93); }
```

#### State: Hidden
```css
.fab.hidden { opacity: 0; transform: scale(0.6) translateY(24px); pointer-events: none; }
```

---

### Ghost Buttons (Lines 1296–1323)

```css
.ghost-btn {
  height: 34px;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: transparent;
  color: var(--text-muted);
  font-weight: 600; font-size: 0.82rem;
  padding: 0 12px;
  cursor: pointer;
  transition: all var(--transition);
}
```

#### State: Hover
```css
.ghost-btn:hover:not(:disabled) { background: rgba(255,255,255,0.05); border-color: var(--border-strong); color: var(--text); }
```

#### State: Active (Press)
```css
.ghost-btn:active:not(:disabled) { transform: scale(0.96); }
```

#### Variant: Danger
```css
.ghost-btn.danger { color: var(--danger); border-color: rgba(248,113,113,0.3); }
```

#### State: Disabled
```css
.ghost-btn:disabled { opacity: 0.3; cursor: not-allowed; }
```

---

### Action Buttons (Lines 1555–1625)

Large, full-width list-style buttons used in sheets for primary actions.

#### HTML Structure
```html
<button class="action-btn primary" type="button">
  <div class="action-btn-text"><strong>Save</strong><span>Description text.</span></div>
  <div class="action-btn-icon" aria-hidden="true"><svg><use href="#icon-check"></use></svg></div>
</button>
```

#### CSS
```css
.action-btn {
  display: flex; align-items: center; justify-content: space-between; gap: 14px;
  width: 100%; padding: 16px 18px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--border);
  background: rgba(255,255,255,0.03);
  transition: background var(--transition), border-color var(--transition), transform 150ms ease;
  cursor: pointer; text-align: left;
}
```

#### State: Hover
```css
.action-btn:hover { background: rgba(255,255,255,0.06); border-color: var(--border-strong); }
```

#### State: Active (Press)
```css
.action-btn:active { transform: scale(0.985); }
```

#### Variant: Primary Hover
```css
.action-btn.primary:hover { background: rgba(56, 189, 248, 0.08); }
```

#### Variant: Danger Hover
```css
.action-btn.danger:hover { background: rgba(248, 113, 113, 0.08); }
```

#### State: Disabled
```css
.action-btn:disabled { opacity: 0.45; cursor: not-allowed; transform: none; }
```

---

### Segment Buttons / Toggle Buttons (Lines 1151–1184)

#### HTML Structure
```html
<div class="segmented">
  <button class="segment-btn active" type="button">Option A</button>
  <button class="segment-btn" type="button">Option B</button>
</div>
```

#### CSS
```css
.segmented {
  display: inline-flex; width: 100%; padding: 4px; gap: 4px;
  border-radius: var(--radius-pill);
  border: 1px solid var(--border);
  background: rgba(255,255,255,0.03);
}
.segment-btn {
  flex: 1; height: 38px; border: none; border-radius: var(--radius-pill);
  background: transparent; color: var(--text-muted);
  font-size: 0.92rem; font-weight: 700;
  transition: all 250ms cubic-bezier(0.32, 0.72, 0, 1);
}
```

#### State: Active
```css
.segment-btn.active {
  background: rgba(56, 189, 248, 0.14);
  color: var(--accent);
  box-shadow: inset 0 0 0 1px rgba(56, 189, 248, 0.25), 0 2px 8px rgba(56, 189, 248, 0.08);
}
```

---

## 5.5 Form Inputs (Lines 1115–1150)

### CSS
```css
.input, .picker-btn {
  width: 100%; height: 54px;
  border-radius: var(--radius-sm);      /* 14px */
  border: 1px solid var(--border-strong);
  background: rgba(0,0,0,0.2);
  color: #fff;
  font-size: 1rem; font-family: inherit;
  padding: 0 16px;
  outline: none;
  transition: border-color var(--transition), box-shadow var(--transition), background var(--transition);
}
```

### State: Focus
```css
.input:focus {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(56, 189, 248, 0.12), 0 0 16px rgba(56, 189, 248, 0.06);
  background: rgba(0, 0, 0, 0.25);
}
```

### Placeholder
```css
.input::placeholder { color: var(--text-subtle); }
```

### Light Theme Override
```css
html[data-theme="light"] .input { background: rgba(255,255,255,0.9); color: var(--text); border-color: rgba(15, 23, 42, 0.1); }
```

### Labels (above inputs)
```css
.field label {
  color: var(--text-muted);
  font-size: 0.78rem; font-weight: 700;
  text-transform: uppercase; letter-spacing: 0.06em;
}
```

---

## 5.6 Toggle Switch (Lines 1212–1249)

### HTML Structure
```html
<label class="switch-row" for="toggleId">
  <span class="switch-copy"><strong>Label</strong><span>Description text.</span></span>
  <input class="switch-input" id="toggleId" type="checkbox" />
</label>
```

### CSS
```css
.switch-input {
  appearance: none; -webkit-appearance: none;
  width: 52px; height: 30px;
  border-radius: 999px;
  border: 1px solid var(--border-strong);
  background: rgba(255,255,255,0.06);
  position: relative; cursor: pointer;
  transition: background 250ms cubic-bezier(0.32, 0.72, 0, 1), border-color 250ms ..., box-shadow 250ms ...;
}
.switch-input::after {
  content: '';
  position: absolute; top: 3px; left: 3px;
  width: 22px; height: 22px; border-radius: 50%;
  background: rgba(255,255,255,0.9);
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  transition: transform 300ms cubic-bezier(0.32, 0.72, 0, 1), background 250ms ease, box-shadow 250ms ease;
}
```

### State: Checked
```css
.switch-input:checked {
  background: rgba(56, 189, 248, 0.2);
  border-color: rgba(56, 189, 248, 0.4);
  box-shadow: 0 0 12px rgba(56, 189, 248, 0.1);
}
.switch-input:checked::after {
  transform: translateX(22px);
  background: var(--accent);
  box-shadow: 0 2px 8px rgba(56, 189, 248, 0.3);
}
```

---

## 5.7 Tabs (Multiple Variants)

### Category/Settings Tabs (Lines 3585–3591)

#### HTML Structure
```html
<div class="cat-tabs">
  <button class="cat-tab active">Tab 1</button>
  <button class="cat-tab">Tab 2</button>
</div>
```

#### CSS
```css
.cat-tabs { display: flex; gap: 0; margin: 0 0 16px; background: var(--bg-secondary); border-radius: 10px; padding: 3px; }
.cat-tab {
  flex: 1; border: none; background: transparent; color: var(--text-muted);
  font-size: 0.78rem; font-weight: 700; padding: 9px 14px; border-radius: 8px;
  cursor: pointer; transition: all 200ms; white-space: nowrap;
}
```

#### State: Active
```css
.cat-tab.active { background: var(--bg-primary); color: var(--text-primary); box-shadow: 0 1px 4px rgba(0,0,0,0.1); }
```

> Same pattern used for `.alert-tabs / .alert-tab-btn` and `.cal-set-tabs / .cal-set-tab`.

---

## 5.8 Bottom Navigation (Lines 1367–1484)

### HTML Structure
```html
<nav class="bottom-nav" aria-label="Bottom navigation">
  <div class="bottom-nav-inner">
    <button class="nav-btn active" id="navHome" type="button" aria-label="Home">
      <svg><use href="#icon-home"></use></svg>
      Home
    </button>
    <!-- 3 more nav-btns: Calendar, Events, Wallet -->
  </div>
</nav>
```

### CSS
```css
.bottom-nav-inner {
  display: grid; grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 4px;
  padding: 8px 8px calc(10px + var(--safe-bottom));
  border-radius: 24px 24px 0 0;
  border: 1px solid rgba(148, 163, 184, 0.06); border-bottom: none;
  background: rgba(8, 13, 26, 0.78);
  box-shadow: 0 -4px 32px rgba(0, 0, 0, 0.15);
  backdrop-filter: blur(40px) saturate(1.4);
}
.nav-btn {
  border: none; background: transparent;
  color: var(--text-muted);
  border-radius: var(--radius-pill);
  min-height: 46px;
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  gap: 2px;
  font-size: 0.68rem; font-weight: 700;
}
```

### State: Active
```css
.nav-btn.active {
  background: rgba(56, 189, 248, 0.12);
  color: var(--accent);
  box-shadow: inset 0 0 0 1px rgba(56, 189, 248, 0.2);
}
```

### State: Hidden on Scroll
```css
body.nav-hidden .bottom-nav { transform: translateY(calc(100% + 8px)); opacity: 0; pointer-events: none; }
```

A **peek button** (`.bottom-nav-peek`) appears as a small FAB-like circle when nav is hidden.

### Light Theme Override
`backdrop-filter: none` applied to bottom nav inner (line 1442).

---

## 5.9 Toasts (Lines 3331–3350)

### CSS
```css
.app-toast-tray {
  position: fixed; top: env(safe-area-inset-top, 12px); left: 12px; right: 12px;
  z-index: 99998;
  display: flex; flex-direction: column; gap: 8px;
  pointer-events: none;
}
.app-toast {
  pointer-events: auto; padding: 14px 16px; border-radius: 12px;
  font-size: 14px; font-weight: 500; line-height: 1.4;
  box-shadow: 0 4px 16px rgba(0,0,0,.3);
  animation: toastIn .3s ease-out;
  display: flex; align-items: flex-start; gap: 10px;
}
```

### Variants
```css
.app-toast.success { background: #065f46; color: #d1fae5; }
.app-toast.error { background: #7f1d1d; color: #fecaca; }
.app-toast.info { background: #1e3a5f; color: #bfdbfe; }
```

### State: Dismissing
```css
.app-toast.dismissing { animation: toastOut .3s ease-in forwards; }
```

---

## 5.10 Modals / Overlays

### Category Modal
```html
<div class="cat-modal-overlay" id="catEditModal" hidden>
  <div class="cat-modal-box">
    <!-- form content -->
  </div>
</div>
```

### Alert Popup Overlay
```css
.alert-popup-overlay {
  position: fixed; inset: 0; z-index: 10000;
  display: flex; align-items: center; justify-content: center;
  padding: 24px; background: rgba(0,0,0,0.55);
  backdrop-filter: blur(10px);
  opacity: 0; pointer-events: none; transition: opacity 300ms ease;
}
```

### State: Active
```css
.alert-popup-overlay.active { opacity: 1; pointer-events: auto; }
```

---

## 5.11 Context Menu (Lines 2460–2508)

### CSS
```css
.settings-ctx-menu {
  position: fixed; top: calc(var(--safe-top) + 56px); right: 12px;
  z-index: 8000; min-width: 220px;
  background: var(--surface-solid);
  border: 1px solid var(--border-strong);
  border-radius: var(--radius-sm);
  box-shadow: var(--shadow-lg);
  padding: 6px 0;
  opacity: 0; transform: translateY(-8px) scale(0.96);
  pointer-events: none; transition: opacity 180ms ease, transform 180ms ease;
}
```

### State: Open
```css
.settings-ctx-menu.open { opacity: 1; transform: translateY(0) scale(1); pointer-events: auto; }
```

### Menu Item Hover
```css
.settings-ctx-item:hover { background: var(--surface-hover); }
```

---

## 5.12 Color Swatch Picker (Lines 1830–1860)

### CSS
```css
.color-swatch {
  --swatch: #38bdf8;
  position: relative; width: 26px; height: 26px;
  border-radius: 999px;
  border: 1.5px solid rgba(255,255,255,0.2);
  background: var(--swatch);
  box-shadow: inset 0 0 0 2px rgba(255,255,255,0.35), 0 2px 8px rgba(0,0,0,0.15), 0 0 0 1px var(--swatch);
  overflow: hidden; cursor: pointer;
  transition: transform 200ms ease, box-shadow 200ms ease;
}
/* Hidden native color input */
.color-swatch input { position: absolute; inset: 0; opacity: 0; cursor: pointer; width: 100%; height: 100%; }
```

---

## 5.13 Chips / Badges (Lines 2185–2195)

### Event Chip
```css
.event-chip {
  display: inline-flex; align-items: center; gap: 6px;
  min-height: 28px; padding: 4px 10px;
  border-radius: 999px;
  background: rgba(255,255,255,0.04);
  border: 1px solid var(--border);
  font-size: 0.78rem; font-weight: 700;
  transition: background var(--transition), border-color var(--transition);
}
```

### Status Pills
```css
.wallet-entry-status-pill { display: inline-flex; align-items: center; font-size: 0.65rem; font-weight: 700; padding: 2px 8px; border-radius: 20px; }
.wallet-entry-status-pill.status-paid { background: rgba(34,197,94,0.15); color: #22c55e; }
.wallet-entry-status-pill.status-pending { background: rgba(249,115,22,0.15); color: #f97316; }
.wallet-entry-status-pill.status-overdue { background: rgba(239,68,68,0.15); color: #ef4444; }
```

---

## 5.14 Empty States (Lines 884–908, 2214–2232)

### Dashed Border Pattern with Radial Glow
```css
.empty {
  margin-top: 16px; padding: 40px 24px;
  border: 1.5px dashed var(--border-strong);
  border-radius: var(--radius);
  color: var(--text-muted);
  text-align: center;
  background: rgba(255,255,255,0.02);
  position: relative;
}
.empty::before {
  content: '';
  position: absolute; inset: -1px;
  border-radius: var(--radius);
  background: radial-gradient(ellipse at 50% 0%, rgba(56, 189, 248, 0.04), transparent 60%);
  pointer-events: none;
}
.empty svg { width: 52px; height: 52px; opacity: 0.4; color: var(--accent); }
```

### Wallet / Simpler Empty States
```css
.wallet-empty { text-align: center; padding: 60px 20px; color: var(--text-muted); }
.wallet-empty-icon { font-size: 3rem; margin-bottom: 16px; }
```
