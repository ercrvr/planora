# CSP & Hardening

Content Security Policy configuration, Subresource Integrity, and browser hardening for Planora.

## Table of Contents

1. [Content Security Policy](#content-security-policy)
2. [Recommended CSP for Planora](#recommended-csp-for-planora) (Phases 1–4, including Trusted Types)
3. [Subresource Integrity](#subresource-integrity)
4. [Third-Party Dependency Management](#third-party-dependency-management)
5. [Additional Hardening Headers](#additional-hardening-headers) (including Permissions Policy)
6. [CORS Awareness](#cors-awareness)

---

## Content Security Policy

CSP is a browser security mechanism that restricts what resources a page can load and execute. It's the single most effective defense against XSS after proper output encoding.

### Why Planora Needs CSP

Currently, Planora has **no CSP**. This means if an attacker injects `<script>alert(1)</script>` through any vector (localStorage tampering, QR import, future API responses), the browser will execute it without question.

A properly configured CSP would block that script from running, even if the XSS payload lands in the DOM.

### The Challenge: Inline Scripts

Planora's entire JavaScript (~8,000+ lines) lives inside a `<script>` block in the HTML file. A strict CSP like `script-src 'self'` would block this. Options:

1. **Move JS to an external file** — The cleanest solution. Put the script in a `.js` file and reference it with `<script src="app.js">`. This allows `script-src 'self'` without `unsafe-inline`.

2. **Use a nonce** — Add a random nonce to the script tag and the CSP header. This works for inline scripts but requires the nonce to change on each page load (easy with a server, harder for a static file).

3. **Use a hash** — Calculate the SHA-256 hash of the entire inline script and include it in the CSP. This works for static files but the hash must be updated whenever the script changes.

4. **Use `unsafe-inline` as a stepping stone** — Start with a CSP that allows inline scripts but restricts everything else. This still provides protection against injected external scripts and resource loading. Tighten later.

---

## Recommended CSP for Planora

### Phase 1: Baseline Protection (implement now)

This CSP allows the existing inline scripts but restricts external resource loading:

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'none';
  script-src 'self' 'unsafe-inline'
    https://cdn.jsdelivr.net/npm/qrcode@1.4.4/
    https://cdn.jsdelivr.net/npm/lz-string@1.5.0/
    https://unpkg.com/html5-qrcode@2.3.8/;
  style-src 'self' 'unsafe-inline'
    https://fonts.googleapis.com;
  font-src https://fonts.gstatic.com;
  img-src 'self' data: blob:;
  connect-src 'self'
    https://api.frankfurter.app;
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
">
```

**What this does:**
- Blocks scripts from any domain not explicitly listed
- Blocks object/embed/applet elements entirely
- Restricts API calls to only the Frankfurter currency API
- Blocks the page from being embedded in iframes (clickjacking protection)
- Restricts form submissions to same origin

**What it doesn't do yet:**
- Doesn't block inline scripts (requires `unsafe-inline`)
- Doesn't block inline styles (requires `unsafe-inline`)

### Phase 2: Hash-Based CSP (after script externalization or when rebuilding)

When the JS is moved to an external file:

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'none';
  script-src 'self'
    'sha256-{hash-of-inline-bootstrap}'
    https://cdn.jsdelivr.net/npm/qrcode@1.4.4/
    https://cdn.jsdelivr.net/npm/lz-string@1.5.0/
    https://unpkg.com/html5-qrcode@2.3.8/;
  style-src 'self'
    https://fonts.googleapis.com;
  font-src https://fonts.gstatic.com;
  img-src 'self' data: blob:;
  connect-src 'self'
    https://api.frankfurter.app;
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
">
```

### Phase 3: Strict CSP (target state)

After migrating inline event handlers to addEventListener:

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'none';
  script-src 'self' 'strict-dynamic' 'sha256-{bootstrap-hash}';
  style-src 'self' 'sha256-{style-hash}';
  font-src https://fonts.gstatic.com;
  img-src 'self' data: blob:;
  connect-src 'self' https://api.frankfurter.app;
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
">
```

### Phase 4: Trusted Types (advanced target)

After Phase 3 is stable, add Trusted Types to eliminate DOM XSS structurally. Trusted Types make it impossible to forget `escapeHtml()` — the browser itself blocks any raw string assignment to innerHTML and other injection sinks.

**Browser support:** Chrome/Edge (stable since v83), Firefox (behind `dom.security.trusted_types.enabled` flag). Safari: not yet supported. Use the Trusted Types polyfill for broader coverage, or deploy as a report-only header first.

#### Step 1: Report-Only Mode (observe without breaking)

```
Content-Security-Policy-Report-Only: require-trusted-types-for 'script'; report-uri /csp-reports
```

This logs every innerHTML/eval/etc. assignment without blocking it. Review the reports to find all sink sites that need migration.

#### Step 2: Define a Planora Trusted Types Policy

```js
// Create a single policy for the entire app
var escapePolicy = trustedTypes.createPolicy('planora-escape', {
  createHTML: function(dirty) {
    // Use the existing escapeHtml() function as the sanitizer
    return escapeHtml(dirty);
  }
});

// All innerHTML assignments must now go through the policy:
// ❌ BLOCKED by Trusted Types
element.innerHTML = '<div>' + userInput + '</div>';

// ✅ ALLOWED — policy applies escapeHtml automatically
element.innerHTML = escapePolicy.createHTML(userInput);
```

#### Step 3: Handle Intentional HTML (Safe Sinks)

Some innerHTML assignments intentionally set markup (e.g., rendering icons, layout templates). For these, create a separate "trusted markup" policy:

```js
var trustedMarkupPolicy = trustedTypes.createPolicy('planora-markup', {
  createHTML: function(html) {
    // This policy is for developer-authored markup only.
    // NEVER pass user input through this policy.
    return html;
  }
});

// For trusted, developer-authored markup:
container.innerHTML = trustedMarkupPolicy.createHTML(
  '<div class="icon">' + SAFE_ICON_SVG + '</div>'
);
```

**Audit rule:** Every use of `trustedMarkupPolicy` must be reviewed — it's a bypass. Grep for it during security audits.

#### Step 4: Enforce

```html
<meta http-equiv="Content-Security-Policy" content="
  default-src 'none';
  script-src 'self' 'strict-dynamic' 'sha256-{bootstrap-hash}';
  style-src 'self' 'sha256-{style-hash}';
  font-src https://fonts.gstatic.com;
  img-src 'self' data: blob:;
  connect-src 'self' https://api.frankfurter.app;
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
  require-trusted-types-for 'script';
  trusted-types planora-escape planora-markup;
">
```

#### Migration Effort Estimate

With 73 innerHTML assignments in Planora:
- ~60 likely use `escapeHtml()` already → wrap in `escapePolicy.createHTML()`
- ~10-15 set trusted developer markup → wrap in `trustedMarkupPolicy.createHTML()`
- ~1-3 may need refactoring to `textContent` instead

This is a **medium-effort, high-reward** change — once deployed, new DOM XSS vulnerabilities become structurally impossible.

---

## Subresource Integrity

SRI ensures that files loaded from CDNs haven't been tampered with. If the CDN serves a different file than expected, the browser refuses to execute it.

### Current External Resources Without SRI

```html
<!-- Line 4831 — NO integrity hash -->
<script src="https://cdn.jsdelivr.net/npm/qrcode@1.4.4/build/qrcode.min.js" crossorigin="anonymous"></script>

<!-- Line 4832 — NO integrity hash -->
<script src="https://cdn.jsdelivr.net/npm/lz-string@1.5.0/libs/lz-string.min.js" crossorigin="anonymous"></script>

<!-- Line 4833 — NO integrity hash -->
<script src="https://unpkg.com/html5-qrcode@2.3.8/html5-qrcode.min.js" crossorigin="anonymous"></script>
```

### How to Generate SRI Hashes

```bash
# Using curl + openssl
curl -s https://cdn.jsdelivr.net/npm/qrcode@1.4.4/build/qrcode.min.js | \
  openssl dgst -sha384 -binary | openssl base64 -A

# Or using the srihash.org website
# Or using Node.js:
# node -e "const crypto=require('crypto');const fs=require('fs');console.log('sha384-'+crypto.createHash('sha384').update(fs.readFileSync(process.argv[1])).digest('base64'))" file.js
```

### Corrected Script Tags

```html
<script src="https://cdn.jsdelivr.net/npm/qrcode@1.4.4/build/qrcode.min.js"
  integrity="sha384-{GENERATED_HASH}"
  crossorigin="anonymous"></script>

<script src="https://cdn.jsdelivr.net/npm/lz-string@1.5.0/libs/lz-string.min.js"
  integrity="sha384-{GENERATED_HASH}"
  crossorigin="anonymous"></script>

<script src="https://unpkg.com/html5-qrcode@2.3.8/html5-qrcode.min.js"
  integrity="sha384-{GENERATED_HASH}"
  crossorigin="anonymous"></script>
```

Generate the actual hashes before deploying. SRI hashes must be regenerated if you change the version.

---

## Third-Party Dependency Management

### Current Dependencies

| Dependency | Version | CDN | Purpose |
|-----------|---------|-----|---------|
| qrcode | 1.4.4 | jsdelivr | QR code generation |
| lz-string | 1.5.0 | jsdelivr | Data compression for QR transfer |
| html5-qrcode | 2.3.8 | unpkg | QR code scanner (camera) |
| Inter font | latest | Google Fonts | Typography |

### Dependency Security Checklist

When adding or updating a dependency:

1. **Check for known vulnerabilities** — Search the npm advisory database and Snyk for the package
2. **Pin exact versions** — Use `@1.4.4` not `@^1.4.4` or `@latest`
3. **Generate SRI hash** — Before deploying the updated reference
4. **Review the bundle** — For small dependencies, read the minified source to verify it does what it claims
5. **Consider self-hosting** — Download the file and serve it locally to eliminate CDN dependency. This trades CDN-compromise risk for self-hosting maintenance.
6. **Update CSP** — If the CDN domain changes, update the CSP allowlist
7. **Document the dependency** — Why it's needed, what it does, who maintains it

### Self-Hosting Pattern

For maximum supply-chain security, download CDN dependencies and bundle them:

```html
<!-- Instead of CDN -->
<script src="https://cdn.jsdelivr.net/npm/qrcode@1.4.4/build/qrcode.min.js"></script>

<!-- Self-hosted (requires serving the file alongside index.html) -->
<script src="vendor/qrcode-1.4.4.min.js"
  integrity="sha384-{HASH}"
  crossorigin="anonymous"></script>
```

Since Planora is a single-file app, self-hosting means either inlining the library (increases file size) or moving to a multi-file structure with a simple file server.

---

## Additional Hardening Headers

If Planora is served via a web server (not just `file://`), add these response headers:

```
# Prevent MIME type sniffing
X-Content-Type-Options: nosniff

# Prevent clickjacking (CSP frame-ancestors is preferred, but this is a fallback)
X-Frame-Options: DENY

# Control referrer information
Referrer-Policy: strict-origin-when-cross-origin

# Permissions Policy — restrict browser features
# Camera is REQUIRED for QR scanner (html5-qrcode); allow self only
# Deny all other sensitive permissions explicitly
Permissions-Policy: camera=(self), microphone=(), geolocation=(), payment=(), usb=(), magnetometer=(), gyroscope=(), accelerometer=(), autoplay=(), fullscreen=(self), interest-cohort=()

# Force HTTPS (if served over HTTPS)
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

For a static file deployment (GitHub Pages, Netlify, etc.), these can usually be configured in the hosting platform's settings or a `_headers` file.

### Permissions Policy Deep Dive

The Permissions Policy header above deserves explanation:

| Permission | Setting | Why |
|-----------|---------|-----|
| `camera` | `(self)` | **Required** — html5-qrcode accesses the camera for QR scanning. Restricting to `self` prevents third-party iframes from accessing it. |
| `microphone` | `()` | Denied — Planora doesn't use audio input |
| `geolocation` | `()` | Denied — No location features |
| `payment` | `()` | Denied — Payment Request API not used (wallet feature is manual tracking) |
| `usb`, `magnetometer`, `gyroscope`, `accelerometer` | `()` | Denied — No hardware sensor access needed |
| `autoplay` | `()` | Denied — No media playback |
| `fullscreen` | `(self)` | Allowed for self — may be useful for calendar views |
| `interest-cohort` | `()` | Denied — Opt out of FLoC/Topics tracking |

**⚠️ Important:** If you add new features that need permissions (e.g., geolocation-based scheduling, voice input), you must update this header. Forgetting to do so will cause silent failures — the browser simply won't grant the permission with no error message.

**Testing:** Check permissions in browser DevTools → Application → Permissions Policy (Chrome) to verify which features are enabled/blocked.

---

## CORS Awareness

### Planora's CORS Context

Planora makes one cross-origin request: fetching currency exchange rates from `https://api.frankfurter.app`. The Frankfurter API returns appropriate CORS headers (`Access-Control-Allow-Origin: *`), so this works from any origin.

If Planora is served from `file://` (opened directly as a local file), CORS behavior varies by browser — some treat file origins as `null` and block cross-origin requests entirely.

### CORS Pitfalls to Avoid

**1. Don't set `Access-Control-Allow-Origin: *` on your own proxy if it handles authenticated requests.**

If you set up a proxy (see `references/env-keys-proxy.md`) that forwards requests with API keys, the proxy must NOT use `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`. This would let any website use your proxy (and your API key) for free.

```js
// ❌ DANGEROUS proxy response headers
{
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Credentials": "true"  // This + wildcard = open proxy
}

// ✅ SAFE — restrict to your origin
{
  "Access-Control-Allow-Origin": "https://your-planora-domain.com",
  "Access-Control-Allow-Credentials": "true"
}
```

**2. Don't disable CORS checks in production.**

Browser extensions like "CORS Unblock" or proxy-all-requests workarounds mask CORS errors during development but don't fix the underlying issue. If a CORS error occurs in production, the fix is:
- Add proper CORS headers on the server/API
- Or route the request through a proxy you control (see `references/env-keys-proxy.md`)

**3. Don't use `no-cors` fetch mode to suppress errors.**

```js
// ❌ This suppresses the error but gives you an opaque response you can't read
fetch('https://api.example.com/data', { mode: 'no-cors' });

// ✅ Fix the server-side CORS headers or use a proxy instead
fetch('https://your-proxy.workers.dev/data');
```

**4. Keep `connect-src` in CSP aligned with CORS needs.**

CSP's `connect-src` directive controls which origins the browser will allow `fetch`/`XHR` to contact. If you add a new API endpoint, you need to update both:
- The CSP `connect-src` allowlist
- The proxy's CORS origin allowlist (if proxied)

### When Adding a New External API

```
1. Does the API support CORS from your origin?
   → Yes: Add it to CSP connect-src and use directly
   → No: Route through a proxy (see env-keys-proxy.md)

2. Does the API require authentication?
   → Yes: MUST use a proxy — never expose API keys to the client
   → No: Direct fetch is acceptable if CORS headers are present

3. Update CSP connect-src to include the new origin (API or proxy)

4. Test from the actual production origin (not localhost, not file://)
```
