# Planora Dead Code Audit Report

**Generated:** 2026-03-15
**File:** `planora-audit.html` (13,406 lines, 657,840 bytes)

## Summary

| Metric | Count |
|--------|-------|
| Total JS functions cataloged | 375 |
| IIFEs (self-executing, not dead) | 4 |
| **Unused functions (high confidence)** | **4** |
| Possibly unused functions (needs review) | 0 |
| Total CSS classes defined | 571 |
| **Unused CSS classes** | **41** |
| Total CSS IDs defined | 15 |
| Unused CSS IDs | 0 (9 false positives excluded — hex colors) |
| Commented-out code blocks (≥5 lines) | 0 |
| Unreachable code instances | 0 |
| Unused top-level variables | 1 |
| HTML IDs not referenced in JS | 52 |
| Duplicate function groups | 1 |
| **Estimated total dead code** | **~146 lines** |

## 1. Unused JavaScript Functions

### Definitely Unused (High Confidence)

These functions are defined but their name appears **only once** in the entire file (the definition itself). No call sites, no event handler references, no callback usage detected.

| # | Function Name | Line | Body Lines | Context |
|---|--------------|------|-----------|---------|
| 1 | `openCalendarSettingsSheet` | 7135 | 5 | `function openCalendarSettingsSheet() {` |
| 2 | `addCard` | 7511 | 10 | `function addCard() {` |
| 3 | `snapScheduleGridToNearestDay` | 7701 | 3 | `function snapScheduleGridToNearestDay() {` |
| 4 | `populateEventTypeCreateOptions` | 8591 | 5 | `function populateEventTypeCreateOptions() {` |

### IIFEs (Self-Executing — NOT Dead Code)

These immediately-invoked function expressions execute at load time. They don't need external callers.

| # | Function Name | Line | Context |
|---|--------------|------|---------|
| 1 | `autoFetchRatesOnInit` | 5751 | `(function autoFetchRatesOnInit() {` |
| 2 | `initTplVarPills` | 12682 | `(function initTplVarPills() {` |
| 3 | `initSettingsGear` | 12872 | `(function initSettingsGear() {` |
| 4 | `setupWelcomeGate` | 12978 | `(function setupWelcomeGate() {` |

## 2. Unused CSS Classes/Selectors

Found **41** CSS classes with no detected reference outside `<style>` blocks.

These classes are not found in HTML `class=` attributes, JavaScript `classList` operations, `className` assignments, string literals, or template literals.

| # | Class Name | Defined at Line |
|---|-----------|----------------|
| 1 | `.topbar-title` | 92 |
| 2 | `.card-title` | 97 |
| 3 | `.micro-chip` | 120 |
| 4 | `.page-placeholder` | 305 |
| 5 | `.schedule-date-group` | 626 |
| 6 | `.today-left` | 805 |
| 7 | `.today-right` | 811 |
| 8 | `.card-sub` | 1071 |
| 9 | `.time-value` | 1164 |
| 10 | `.time-date` | 1174 |
| 11 | `.meta-row` | 1195 |
| 12 | `.picker-meta` | 1272 |
| 13 | `.card-current-row` | 1419 |
| 14 | `.editor-actions` | 1431 |
| 15 | `.is-warning` | 2330 |
| 16 | `.is-danger` | 2334 |
| 17 | `.event-comment-composer` | 2380 |
| 18 | `.event-mini-actions` | 2401 |
| 19 | `.event-subtask-list` | 2465 |
| 20 | `.event-subtask-row` | 2466 |
| 21 | `.event-note-row` | 2467 |
| 22 | `.marker-icon` | 2514 |
| 23 | `.event-type-inline-grid` | 2545 |
| 24 | `.event-type-grid` | 2616 |
| 25 | `.alert-popup-emoji` | 2946 |
| 26 | `.alert-popup-title` | 2947 |
| 27 | `.alert-popup-message` | 2948 |
| 28 | `.alert-popup-meta` | 2951 |
| 29 | `.alert-settings-body` | 2993 |
| 30 | `.lt-sublabel` | 3060 |
| 31 | `.alert-company-card` | 3087 |
| 32 | `.alert-company-header` | 3092 |
| 33 | `.alert-company-name` | 3096 |
| 34 | `.co-dot` | 3100 |
| 35 | `.alert-company-fields` | 3103 |
| 36 | `.alert-tpl-editor` | 3167 |
| 37 | `.alert-tpl-editor-vars` | 3182 |
| 38 | `.alert-tpl-editor-var` | 3183 |
| 39 | `.alert-tpl-editor-actions` | 3192 |
| 40 | `.wallet-gear-btn` | 3603 |
| 41 | `.w3` | 3706 |

### Unused CSS IDs

No unused CSS ID selectors found. *(Note: 9 hex color codes like `#f8fafc`, `#fbbf24` were initially detected as ID selectors but are actually CSS color values — false positives excluded.)*

## 3. Commented-Out Code Blocks

No significant commented-out code blocks (≥5 lines) found. The codebase is clean of large comment blocks.

## 4. Duplicate/Redundant Code

Found **1** groups of structurally identical functions that could potentially be consolidated.

### Duplicate Group 1
These 2 functions have identical structure:

- **`resolveEventOwnerColor`** at line 8558 (7 lines)
- **`resolveEventMarkerColor`** at line 8566 (7 lines)

<details><summary>Sample body</summary>

```javascript
      function resolveEventOwnerColor(event, type = null) {
        if (event?.status === 'completed') return '#94a3b8';
        const ownerColor = resolveCompanyColor(event?.ownerCompanyId || '');
        if (ownerColor) return ownerColor;
        const eventType = type || findEventType(event?.type
```
</details>

## 5. Suspicious Dead Code Patterns

### Unused Top-Level Variables (1)

| # | Variable | Line | Context |
|---|----------|------|---------|
| 1 | `ALERT_STORAGE_KEY` | 10773 | `const ALERT_STORAGE_KEY = 'timezone_planner_alerts_v1';` |

### HTML IDs Not Referenced in JavaScript (52)

These HTML elements have `id` attributes with no direct JS reference found. However, **most of these are SVG `<symbol>` elements** used as an icon sprite sheet. They are referenced dynamically via patterns like `href="#icon-${escapeHtml(icon)}"` and `href="#shape-${escapeHtml(shape)}"` in JS-generated HTML. These are **NOT dead code** — they are used dynamically.

Only `eventsSearchBar` and `walletPeriodNav` at the bottom are potentially unreferenced non-SVG IDs worth investigating.

| # | ID | HTML Line |
|---|-----|----------|
| 1 | `icon-plus` | 3961 |
| 2 | `icon-globe` | 3966 |
| 3 | `icon-chevrons-up` | 3969 |
| 4 | `icon-moon` | 3970 |
| 5 | `icon-sun` | 3971 |
| 6 | `icon-home` | 3972 |
| 7 | `icon-calendar` | 3973 |
| 8 | `icon-notes` | 3974 |
| 9 | `icon-sliders` | 3975 |
| 10 | `icon-bell` | 3976 |
| 11 | `icon-briefcase` | 3977 |
| 12 | `icon-task` | 3978 |
| 13 | `icon-alert` | 3979 |
| 14 | `icon-repeat` | 3980 |
| 15 | `icon-list` | 3981 |
| 16 | `icon-clock2` | 3982 |
| 17 | `icon-flag` | 3983 |
| 18 | `icon-message` | 3984 |
| 19 | `icon-phone` | 3985 |
| 20 | `icon-video` | 3986 |
| 21 | `icon-mail` | 3987 |
| 22 | `icon-star` | 3988 |
| 23 | `icon-file` | 3989 |
| 24 | `icon-wallet` | 3990 |
| 25 | `icon-dollar` | 3991 |
| 26 | `icon-link` | 3992 |
| 27 | `icon-user` | 3993 |
| 28 | `icon-target` | 3994 |
| 29 | `shape-square` | 3996 |
| 30 | `shape-circle` | 3997 |
| 31 | `shape-diamond` | 3998 |
| 32 | `shape-triangle` | 3999 |
| 33 | `shape-hexagon` | 4000 |
| 34 | `shape-octagon` | 4001 |
| 35 | `shape-pill` | 4002 |
| 36 | `shape-bookmark` | 4003 |
| 37 | `shape-shield` | 4004 |
| 38 | `shape-tag` | 4005 |
| 39 | `shape-star` | 4006 |
| 40 | `shape-burst` | 4007 |
| 41 | `icon-check-circle` | 4008 |
| 42 | `icon-spark` | 4009 |
| 43 | `icon-pin` | 4010 |
| 44 | `icon-wrench` | 4011 |
| 45 | `icon-rocket` | 4012 |
| 46 | `icon-shield-check` | 4013 |
| 47 | `icon-users` | 4014 |
| 48 | `icon-bug` | 4015 |
| 49 | `icon-headset` | 4016 |
| 50 | `icon-document` | 4017 |
| 51 | `eventsSearchBar` | 4184 |
| 52 | `walletPeriodNav` | 4213 |

## 6. Recommendations

### 🔴 High Priority — Safe to Remove

1. **Definitely unused functions** (4 functions, ~23 lines)
   - These have zero call sites anywhere in the file. Safe to delete.
   - Functions: `openCalendarSettingsSheet`, `addCard`, `snapScheduleGridToNearestDay`, `populateEventTypeCreateOptions`

### 🟡 Medium Priority — Review Before Removing

3. **Possibly unused functions** (0 functions, ~0 lines)
   - Verify these aren't called via dynamic patterns before removing.

4. **Unused CSS** (41 classes, ~123 lines)
   - These class names appear only in CSS `<style>` blocks with zero references in HTML or JS (including JS-generated innerHTML strings). Some may be applied via dynamic class name construction — review before removing.

### 🟢 Low Priority — Investigate

5. **Unreachable code** (0 instances)
   - Code after return/break statements. May be intentionally disabled.

6. **Duplicate functions** (1 groups)
   - Could be consolidated into shared utility functions.

7. **Unreferenced HTML IDs** (~2 non-SVG IDs; the other 50 SVG symbols are dynamically referenced)
   - May be CSS-only anchors or leftovers. Low risk but worth cleaning up.

### Total Estimated Savings

| Category | Est. Lines |
|----------|-----------|
| Unused functions | ~23 |
| Possibly unused functions | ~0 |
| Commented-out code | ~0 |
| Unused CSS rules | ~123 |
| Unreachable code | ~0 |
| **Total** | **~146** |

This represents approximately **1.1%** of the total 13,406-line file.

---

*Note: This analysis uses static regex-based pattern matching. Functions called dynamically via*
*string concatenation, `eval()`, computed property names, or framework conventions may appear*
*as unused. IIFEs are NOT dead code — they self-execute at load time. Always verify before deleting.*