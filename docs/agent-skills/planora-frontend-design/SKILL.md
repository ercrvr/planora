---
name: planora-frontend-design
description: How to create and modify UI components in the Planora monolithic HTML application. Use this whenever implementing new features, fixing UI bugs, adding components, tweaking styles, or making any visual change to Planora — even for small CSS adjustments. Also use when reviewing designs for consistency with the existing design system.
---

# Planora Frontend Design

Planora is a single-file (~647KB, ~13,400 lines) offline-first web application with a dark-first glassmorphism design system. Every visual element follows a cohesive set of patterns — new components must blend in seamlessly.

This skill covers how to create, modify, and style UI components that match Planora's design language.

## Architecture Quick Facts

- **Single HTML file** containing all CSS (`<style>` in `<head>`) and JS (two `<script>` blocks)
- **No framework** — vanilla HTML/CSS/JS
- **CSS custom properties** for theming (25 root variables)
- **Dark theme default**, light theme via `html[data-theme="light"]` overrides
- **SVG sprite sheet** for icons (40+ symbols referenced via `<use href="#icon-name">`)
- **Inter** font family, weights 400–900

## The Planora Look — Core Design Principles

1. **Dark-first glassmorphism** — Surfaces are translucent with blur effects. In light mode, surfaces become solid white instead.
2. **Signature easing** — `cubic-bezier(0.32, 0.72, 0, 1)` is used on virtually every transition and animation. This gives the app its distinctive spring-like feel. Use it consistently.
3. **Sky blue accent** — `var(--accent)` (#38bdf8) is the single primary accent color. Used sparingly for active states, focus rings, and primary CTAs. Don't introduce new accent colors.
4. **Minimal borders, heavy blur** — Visual separation comes from glassmorphism (backdrop-filter + translucent backgrounds) rather than hard borders.
5. **Mobile-native feel** — Bottom sheets, snap-scroll carousels, FAB, press-scale animations. Everything feels like a native mobile app.
6. **Progressive disclosure** — Content is revealed through bottom sheets and expandable sections, not crammed into views.
7. **Emoji as decoration** — Category icons and some section headers use emoji rather than SVG icons.

## Design Tokens

All tokens are in `:root` (lines 21–57) with light theme overrides at lines 60–75.

### Colors
```css
/* Backgrounds */
--bg: #080d1a;                              /* Page background */
--surface: rgba(22, 33, 55, 0.55);         /* Card/panel backgrounds (translucent!) */
--surface-hover: rgba(35, 50, 78, 0.72);   /* Hover state for surfaces */
--surface-solid: #131c2e;                   /* Opaque surfaces (sheets, modals) */

/* Text — three levels of hierarchy */
--text: #f1f5f9;                            /* Primary */
--text-muted: #94a3b8;                      /* Secondary */
--text-subtle: #64748b;                     /* Tertiary / placeholder */

/* Accent & Semantic */
--accent: #38bdf8;                          /* Sky blue — THE accent color */
--accent-hover: #7dd3fc;                    /* Lighter accent for hover */
--accent-gradient: linear-gradient(135deg, #38bdf8, #22d3ee);
--danger: #f87171;                          /* Error/destructive */
--danger-soft: rgba(248, 113, 113, 0.12);  /* Danger background tint */

/* Also used but not as variables */
/* #22c55e — success/income green */
/* #f59e0b — warning/amber */
/* #8b5cf6 — purple accent (budgets) */
```

### Borders, Shadows, Radius
```css
--border: rgba(148, 163, 184, 0.07);       /* Subtle — default borders */
--border-strong: rgba(15, 23, 42, 0.11);   /* Prominent — sheets, inputs */

--shadow-sm: 0 2px 8px -2px rgba(0,0,0,0.2), 0 4px 16px -4px rgba(0,0,0,0.15);
--shadow-lg: 0 12px 40px -8px rgba(0,0,0,0.4), 0 4px 16px -4px rgba(0,0,0,0.25);
--shadow-glow: 0 0 24px rgba(56,189,248,0.25), 0 0 48px rgba(56,189,248,0.08);

--radius: 20px;           /* Cards, large containers */
--radius-sm: 14px;        /* Inputs, smaller elements */
--radius-pill: 9999px;    /* Buttons, badges, pills */
```

### Transitions (always use the signature easing)
```css
--transition: 220ms cubic-bezier(0.32, 0.72, 0, 1);         /* Default */
--transition-spring: 380ms cubic-bezier(0.32, 0.72, 0, 1);  /* Larger movements */
--transition-slow: 320ms cubic-bezier(0.32, 0.72, 0, 1);    /* Medium pace */
```

### Safe Areas
```css
--safe-top: env(safe-area-inset-top, 0px);
--safe-bottom: env(safe-area-inset-bottom, 0px);
```
Use `var(--safe-bottom)` in `calc()` for all bottom-positioned fixed elements.

## Creating New Components

Follow this checklist for every new UI element:

### Surface / Container
```css
.my-component {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);          /* or var(--radius) for larger containers */
  padding: 14px;                            /* 14-20px typical */
  backdrop-filter: blur(20px) saturate(1.2);
  -webkit-backdrop-filter: blur(20px) saturate(1.2);
  box-shadow: var(--shadow-sm);
  transition: all var(--transition);
}
```

### Interactive States
```css
/* Hover — lighten, strengthen border, slight lift */
.my-component:hover {
  background: var(--surface-hover);
  border-color: var(--border-strong);
  transform: translateY(-1px);
  box-shadow: var(--shadow-lg);
}

/* Active/press — always scale down for tactile feedback */
.my-component:active {
  transform: scale(0.95);
}

/* Focus — accent border with glow ring */
.my-component:focus {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(56, 189, 248, 0.12), 0 0 16px rgba(56, 189, 248, 0.06);
  outline: none;
}

/* Disabled */
.my-component:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
```

### Entry Animation
New elements entering the view should use `fadeInUp` (already defined):
```css
.my-component {
  animation: fadeInUp 400ms cubic-bezier(0.32, 0.72, 0, 1) both;
}
```
For staggered lists, use `animation-delay` with increments (e.g., `calc(var(--i, 0) * 60ms)`).

### Typography
```css
/* Labels (above inputs, section kickers) */
.my-label {
  color: var(--text-muted);
  font-size: 0.78rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
}

/* Section headings */
.my-heading {
  color: var(--text);
  font-size: 1.08rem;
  font-weight: 800;
  letter-spacing: -0.01em;
}

/* Body text */
.my-body {
  color: var(--text);
  font-size: 0.9rem;
  font-weight: 600;
  line-height: 1.4;
}

/* Helper/description text */
.my-helper {
  color: var(--text-muted);
  font-size: 0.78rem;
  font-weight: 500;
  line-height: 1.4;
}
```

### Touch Targets
- Minimum touch target: **40px × 40px** (icon buttons)
- Navigation buttons: **min-height: 46px**
- Form inputs: **height: 54px** (48px on mobile)
- Always add `cursor: pointer` to interactive elements

### Light Theme Handling
Every component must work in both themes. Common overrides to add:

```css
html[data-theme="light"] .my-component {
  background: rgba(255, 255, 255, 0.78);   /* Translucent white instead of --surface */
  border-color: rgba(15, 23, 42, 0.06);
  backdrop-filter: none;                     /* Glass effect disabled in light mode */
}
```

Key light theme behaviors:
- `backdrop-filter` is disabled on most elements
- Surfaces become solid/near-white instead of translucent dark
- Noise texture overlay is hidden
- Background gradient is replaced with flat `#f0f4f8`
- Accent color stays the same (#38bdf8)

### Responsive Design
Desktop-first approach. Main breakpoint is **768px**.

At 768px, the card grid transforms into a horizontal snap carousel:
```css
@media (max-width: 768px) {
  .grid {
    display: flex; flex-wrap: nowrap;
    overflow-x: auto; scroll-snap-type: x mandatory;
  }
  .card {
    flex: 0 0 calc(100vw - 64px);
    scroll-snap-align: center;
  }
}
```

Other breakpoints: 900px (tablet), 760px (wallet), 720px (alerts), 640px (small mobile).

Check your component at 768px. Inputs shrink to 44px height on mobile.

## Existing Component Patterns

Read `references/component-catalog.md` for exact HTML structure and CSS for every existing component:
- Cards (company timezone cards, event cards)
- Bottom sheets (iOS-style slide-up panels)
- Buttons (icon, FAB, ghost, action, segment)
- Form inputs and toggle switches
- Tabs (category tabs, segment buttons)
- Bottom navigation bar
- Toasts and notifications
- Modals and context menus
- Color swatches, chips/badges
- Empty states

Always match the existing patterns when creating similar components. Don't reinvent — extend.

## Icons

Use the SVG sprite sheet (defined at line 3708). Reference icons with:
```html
<svg><use href="#icon-name"></use></svg>
```

Available icons include: `icon-plus`, `icon-edit`, `icon-check`, `icon-close`, `icon-trash`, `icon-globe`, `icon-calendar`, `icon-bell`, `icon-wallet`, `icon-dollar`, `icon-user`, `icon-star`, and 30+ more.

Icon styling in buttons:
```css
svg { width: 20px; height: 20px; fill: none; stroke: currentColor; stroke-width: 2; }
```

If you need a new icon, add it as a `<symbol>` in the sprite sheet and follow the existing naming pattern (`icon-{name}`).

## Z-Index Hierarchy

Respect the existing stacking order:
- **40** — Topbar (sticky header)
- **45** — Bottom nav
- **50** — FAB
- **80** — Scrim (sheet backdrop)
- **90** — Bottom sheets
- **200** — Popups within pages
- **8000** — Context menus
- **10000** — Modal overlays
- **99998** — Toast notifications
- **100000** — Welcome/splash

Place new components at the appropriate level. Don't use arbitrary z-index values.

## Naming Conventions

### CSS Classes
Lowercase, hyphen-separated. Follow the component-element pattern:
- Component: `.my-widget`
- Element: `.my-widget-head`, `.my-widget-body`
- State: `.active`, `.open`, `.hidden`, `.is-completed`, `.dismissing`
- Variant: `.primary`, `.danger`, `.compact`, `.mini`

Group-prefix classes by page/feature: `event-*`, `alert-*`, `wallet-*`, `schedule-*`, etc.

### IDs
camelCase: `myWidgetTitle`, `editBtn`, `addNewForm`

### Data Attributes
`data-{entity}-id` for entity references, `data-action` for click handlers, `data-field` for form binding.

## Layout Patterns

- Main container: `width: min(1000px, 100%); margin: 0 auto`
- Body padding: `0 16px calc(112px + var(--safe-bottom))` (accounts for bottom nav)
- Pages are `<section class="page" id="pageName">` elements, toggled with `hidden` attribute
- Flexbox is the primary layout tool (176 uses vs 13 grid uses)
- Grid is used for specific structured layouts (card grid, event data, time editors, bottom nav)
- Common gap values: 4, 6, 8, 10, 12, 14, 16, 20px

## Common Pitfalls

1. **Forgetting `-webkit-backdrop-filter`** — Safari requires the prefixed version. Always include both.
2. **Not handling light theme** — Every new component needs light theme overrides or it will look broken.
3. **Breaking the z-index hierarchy** — Don't use values outside the established ranges.
4. **Ignoring safe areas** — Fixed bottom elements must include `var(--safe-bottom)` in their positioning.
5. **Wrong easing** — Using `ease` or `ease-in-out` instead of the signature `cubic-bezier(0.32, 0.72, 0, 1)`. The app's personality comes from its consistent spring feel.
6. **Hard-coded colors** — Use CSS variables. The only exception is semantic colors (#22c55e, #f59e0b, #8b5cf6) which don't have variables yet.
7. **Missing :active scale** — Every tappable element needs press feedback via `transform: scale()`.
8. **Oversized touch targets on desktop** — The 44px minimum applies to mobile; desktop can use 34-40px.

## Reference Files

- `references/component-catalog.md` — Exact HTML and CSS for every existing component type. Read this when you need to match or extend an existing component pattern.
- `references/design-tokens.md` — Complete color tables, typography scale, all CSS variables with light/dark values. Read this when you need exact values.
- `references/animations.md` — All 22 keyframe animations, transition patterns, and state change effects. Read this when working on animations or page transitions.
