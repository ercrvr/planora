---
name: planora-security
description: Security practices for the Planora monolithic HTML application. Use this whenever writing or reviewing code, handling user data, adding external dependencies, working with API keys or environment variables, configuring proxies, performing security audits, or making any change that touches data storage, DOM rendering, network requests, or third-party resources — even seemingly small changes. Also use when the user asks about hardening, secrets management, or protecting user data.
---

# Planora Security

Planora is a single-file (~647KB) offline-first vanilla HTML/CSS/JS application. All user data lives in the browser (localStorage/IndexedDB). There is no backend server — but the app does make external API calls (currency rates), loads third-party scripts from CDNs, and has a QR-based data transfer feature.

This means the security surface is narrower than a full-stack app, but the stakes are different: there's no server-side safety net. If malicious data gets into localStorage or the DOM, the client has no second line of defense. Every piece of rendering code and every data import path is a potential entry point.

## Security Principles

1. **Treat localStorage as untrusted input.** Anyone with DevTools (or a malicious browser extension) can modify localStorage. Data read from storage should be validated and sanitized before rendering or processing.

2. **Prefer safe DOM APIs over innerHTML.** Use `textContent`, `createElement`, and `setAttribute` for dynamic content. When innerHTML is necessary (complex templates), always pass user-originated values through `escapeHtml()` first.

3. **Never ship secrets in client-side code.** API keys, debug passwords, tokens — none of these belong in the HTML file. Use environment-variable injection at build/deploy time, or proxy sensitive API calls through a backend.

4. **Pin and verify external resources.** Every `<script src>` and `<link href>` from a CDN must have a Subresource Integrity (SRI) hash. Pin exact versions. Compromised CDNs are a real supply-chain attack vector.

5. **Defense in depth with CSP.** A Content Security Policy restricts what the browser is allowed to execute. Even if an XSS payload lands in the DOM, a tight CSP can prevent it from running.

6. **Validate imports, sanitize exports.** The QR-scan import and JSON file import paths accept external data. Validate schemas strictly before merging into app state. Sanitize exported data to avoid leaking debug/internal state.

7. **Guard against prototype pollution.** Never merge, spread, or `Object.assign` untrusted external data directly. Use safe JSON parsing (with a reviver that strips `__proto__`, `constructor`, `prototype` keys) and always reconstruct objects from an allowlist of known fields.

8. **Handle errors securely.** Catch and handle errors without exposing internal details (stack traces, data structures, storage contents) to the user. Fail gracefully — leave the app in a consistent state and give the user clear feedback about what went wrong.

9. **Monitor and log security events.** Use CSP violation reporting, detect localStorage tampering on load, and log validation failures to the console for debugging — but never log sensitive user data (financial records, encryption keys, full state dumps).

10. **Prevent DOM clobbering.** HTML elements with `id` attributes auto-create `window.{id}` globals that can shadow JavaScript variables. Never rely on bare global lookups for important values. Always use `document.getElementById()` and namespaced ID prefixes.

11. **Use cryptographic randomness for identifiers.** Always use `crypto.randomUUID()` (or the existing `cryptoRandomId()` helper) for generating IDs. Never use `Math.random()` for identifiers — it's predictable.

12. **Validate inputs at every layer.** Check file size before reading, decompressed size before parsing, schema before merging. Each layer catches what the previous one missed.

## Current Security Posture

Understanding where Planora stands today helps you prioritize fixes and avoid introducing regressions.

| Area | Status | Risk |
|------|--------|------|
| DOM rendering | 73 innerHTML usages, `escapeHtml()` used in ~40% of dynamic cases | Medium |
| Code execution | No eval/Function/string-setTimeout — clean | Low |
| Data storage | 7 localStorage keys, all plain text, includes `__debugSecret` | Medium |
| Network/API | 1 fetch call (Frankfurter currency API), no hardcoded API keys | Low |
| CSP | None defined | **High** |
| External deps | 3 CDN scripts, 1 Google Fonts — zero SRI hashes | Medium |
| Input sanitization | `escapeHtml()` exists but not universally applied | Medium |
| Data transfer | QR share/scan with LZString compression, no encryption | Medium |
| Prototype pollution | No guards on JSON import merge paths; `Object.assign`/spread from external data | **High** |
| Error handling | Inconsistent try/catch usage; some silent swallowing, some raw errors shown | Medium |
| URL validation | No URL scheme validation for dynamic href/src attributes | Medium |
| Blob URL cleanup | `URL.createObjectURL` used without `revokeObjectURL` | Low |
| Data migration | Versioned keys exist (`_v7`, `_v1`) but no formal migration safety pattern | Low |
| DOM clobbering | 420+ element IDs, many with risky names that could shadow JS variables | Medium |
| Trusted Types | Not implemented — browser-level innerHTML XSS prevention not yet used | Medium |
| Permissions Policy | Only `interest-cohort=()` set — camera required for QR, all others unblocked | Low |
| Decompression safety | LZString output not size-checked before JSON.parse | Medium |
| File input validation | `accept=".json"` is UI hint only — no size/type enforcement pre-read | Medium |
| ID generation | Inconsistent — 2 of 3 sites use `Math.random()` instead of `cryptoRandomId()` | Low |
| Clipboard handling | Schedule data written to clipboard without sensitivity awareness | Low |

### The `escapeHtml()` Function (line ~10451)

Planora has a single HTML-escaping utility:

```js
function escapeHtml(value) {
  return String(value ?? '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}
```

This covers the standard five characters and is adequate for inserting values into HTML text content and quoted attribute values. Use it consistently for all user-originated data rendered via innerHTML. It does **not** protect against injection into unquoted attributes, `javascript:` URLs, CSS contexts, or `<script>` blocks — avoid those patterns entirely.

---

## Quick Decision Guides

### Adding Dynamic Content to the DOM

```
Is the content from a trusted, hardcoded source (no user/storage data)?
  → innerHTML with static strings is acceptable
  
Does it contain user-originated or storage-originated values?
  → Prefer textContent / createElement + setAttribute
  → If innerHTML is required (complex markup):
      1. Pass every dynamic value through escapeHtml()
      2. Never interpolate into unquoted attributes
      3. Never interpolate into href, src, style, or event handler attributes
      4. Comment why innerHTML is needed (for future auditors)
```

### Adding an External Dependency

```
1. Is there a way to avoid the dependency? (fewer deps = smaller attack surface)
2. Pin to an exact version (not a range, not "latest")
3. Generate an SRI hash and add integrity + crossorigin attributes
4. Add the domain to the CSP connect-src / script-src allowlist
5. Document what the dependency does and why it's needed
```

### Working with API Keys or Sensitive Configuration

```
Does this need to be in the client at all?
  → No: proxy through a backend service
  → Yes (unavoidable): inject at build time via environment variables
      - Never hardcode in the HTML source
      - Use a .env file excluded from version control
      - Document the required env vars in README
      - Rotate keys if they've ever been committed to git history
```

### Performing a Data Import (QR Scan / File Upload)

```
1. Parse the input using safeJSONParse (NOT raw JSON.parse) inside try/catch
2. Check for prototype-polluting keys (__proto__, constructor, prototype)
3. Validate the schema — check expected fields, types, and value ranges
4. Reconstruct objects from an allowlist of known fields (never spread/assign)
5. Sanitize string values before they hit innerHTML or DOM
6. Cap array lengths to prevent memory exhaustion
7. Log import events for debugging (without logging sensitive data values)
```

### Handling Errors Securely

```
Is this a user-facing operation (save, import, export, fetch)?
  → Wrap in try/catch
  → Show a user-friendly message ("Could not save — please try again")
  → Do NOT show: stack traces, raw error objects, localStorage contents, or internal state
  → Log details to console for debugging: console.error('Save failed:', err.message)
  → Leave the app in a consistent state (don't half-write data)

Is this a background operation (auto-save, scheduled fetch, migration)?
  → Catch errors silently but LOG them — don't swallow with empty catch blocks
  → Retry with exponential backoff if appropriate
  → If retries fail, show a non-intrusive notification

Is this a security-relevant failure (invalid import, tampered data, CSP violation)?
  → Log a clear warning: console.warn('[Security] Import contained __proto__ key — rejected')
  → Reject the operation entirely (don't partially process)
  → Never expose WHY it was rejected in user-facing messages
    (say "Invalid data format" not "Prototype pollution attempt detected")
```

### Using URLs from User Data

```
Is the URL displayed in an href, src, or used for navigation?
  → Validate with sanitizeURL() — block javascript:, data:, vbscript: schemes
  → Only allow http:, https:, mailto:, tel:, and relative paths
  → ALSO escapeHtml() the validated URL when inserting into innerHTML

Is the URL used in fetch() or XHR?
  → Validate against an allowlist of known API domains
  → Never fetch a user-provided arbitrary URL from client code
```

### Creating Blob URLs

```
1. Create: var url = URL.createObjectURL(blob)
2. Use: assign to href/src, trigger download, display preview
3. Revoke: URL.revokeObjectURL(url) — immediately after use
4. Safety net: setTimeout revoke as backup (e.g., 60 seconds)
```

### Handling File Uploads

```
1. Check file.size against MAX_IMPORT_FILE_SIZE (2 MB) BEFORE reading
2. Verify file.name ends with .json (accept attribute is just a UI hint)
3. Read with FileReader.readAsText() inside try/catch
4. Check content.length after reading (belt-and-suspenders)
5. Parse with safeJSONParse() — NOT raw JSON.parse()
6. Validate schema with validateImportPayload()
7. Handle FileReader.onerror — don't let it silently fail
```

### Decompressing External Data (QR Scan)

```
1. Decompress: var raw = LZString.decompressFromBase64(encoded)
2. Check decompression success: if (!raw) → reject
3. Check decompressed size: if (raw.length > 512KB) → reject
4. Parse with safeJSONParse(raw)
5. Validate schema with validateImportPayload()
6. Never skip step 3 — a 2KB QR payload can decompress to megabytes
```

### Adding HTML Elements with IDs

```
Does the new element have an id attribute?
  → Does any JavaScript variable use the same name as a bare global?
      → Yes: rename the ID or the variable to avoid DOM clobbering
      → No: still prefer a namespaced prefix (pl-*) for safety
  → Is the ID used in document.getElementById()? ✅ Safe
  → Is the ID accessed as a bare window.{name} global? ❌ Refactor
```

---

## Reference Files

Read these for detailed guidance on specific security domains:

- **`references/secure-coding.md`** — XSS prevention patterns, safe DOM manipulation, innerHTML audit guide, converting innerHTML to safe alternatives, **prototype pollution prevention**, **URL validation patterns**, **blob URL lifecycle management**, **DOM clobbering prevention** (threat model, 420 IDs in Planora, namespacing patterns), **secure randomness** (`cryptoRandomId()` vs `Math.random()`), **timing-safe comparison**, **CSS injection & exfiltration**, and **postMessage security** (future-proofing). Read when writing or reviewing rendering code, handling external data, working with URLs, or adding element IDs.

- **`references/data-protection.md`** — **File input hardening** (pre-read size/type checks), localStorage encryption patterns, sensitive data handling, the `__debugSecret` issue, data export/import validation, QR transfer security with **decompression bomb prevention**, **prototype-safe JSON parsing**, **secure data migration patterns**, **privacy & data minimization guidelines**, and **clipboard data sensitivity**. Read when working with file uploads, data storage, transfer, import/export, migration, or clipboard features.

- **`references/csp-and-hardening.md`** — Content Security Policy configuration (Phases 1–4 including **Trusted Types**), SRI hash generation, third-party dependency management, **expanded Permissions Policy** (camera allowed, all others denied), recommended hardening headers for Planora, and **CORS awareness guide**. Read when adding dependencies, configuring deployment, working with browser permissions, or making cross-origin requests.

- **`references/env-keys-proxy.md`** — Environment variable management, API key handling, proxy patterns for hiding keys from the client, `.env` file conventions, and build-time injection. Read when adding API integrations or managing secrets.

- **`references/audit-checklist.md`** — Step-by-step security audit workflow, **error handling & information disclosure checks**, **security logging & monitoring checks**, **prototype pollution testing**, report template, severity classification, and **PR/code review security quick-check**. Read when performing a security review or reviewing pull requests.

---

## Common Pitfalls

1. **"It's just a local app, security doesn't matter."** Browser extensions, shared computers, malicious QR codes, and XSS through imported data are all real threats. Offline-first doesn't mean risk-free.

2. **Using innerHTML for convenience.** Every innerHTML with an unescaped variable is a potential XSS vector. The five minutes saved writing a template literal instead of createElement could cost hours debugging an exploit.

3. **Committing secrets to git.** Even if you delete the file later, git history preserves it forever. The `__debugSecret` default of `p@ssw0rd` on line 4955 is an example of this — it's public to anyone viewing the source.

4. **Loading CDN scripts without SRI.** If `cdn.jsdelivr.net` or `unpkg.com` serves a compromised file, Planora will execute it. SRI hashes make this impossible.

5. **Trusting QR-scanned data.** The QR import path deserializes LZString-compressed JSON and merges it into app state. Without schema validation, a crafted QR code could inject malicious data that triggers XSS when rendered.

6. **Forgetting to update CSP after adding resources.** Any new external URL must be added to the CSP allowlist, or it will be silently blocked — or worse, you'll weaken the CSP with wildcards to "fix" it.

7. **Merging imported JSON with `Object.assign` or spread.** This is the prototype pollution express lane. A crafted `__proto__` key in imported data pollutes every object in the app. Always parse with `safeJSONParse` and reconstruct from an allowlist of known fields.

8. **Swallowing errors with empty catch blocks.** `catch(e) {}` hides bugs and leaves the app in inconsistent states. Always log the error and either recover gracefully or inform the user.

9. **Showing raw errors to users.** `alert(err.stack)` or rendering error objects into the DOM exposes internal implementation details. Show friendly messages; log details to console.

10. **Using user-provided strings as URLs without validation.** `escapeHtml()` doesn't protect against `javascript:` URLs in `href` attributes. Always validate URL schemes with `sanitizeURL()` before use.

11. **Forgetting to revoke blob URLs.** Every `URL.createObjectURL()` should have a matching `URL.revokeObjectURL()`. Unreleased blobs leak memory and keep data accessible longer than necessary.

12. **Migrating data without re-validating.** When localStorage schema changes, migrated data bypasses import validation. Always run migrated data through the same validation pipeline as new imports.

13. **Relying on element ID globals.** With 420+ `id` attributes, bare `window.debugPanel` lookups can be clobbered by DOM elements. Always use `document.getElementById()` and namespaced prefixes (`pl-*`).

14. **Skipping decompression size checks.** `LZString.decompressFromBase64()` can turn a 2KB QR payload into megabytes. Always check `raw.length` before `JSON.parse()`.

15. **Trusting the file input `accept` attribute.** `accept=".json"` only filters the file picker UI — it doesn't prevent arbitrary files. Always validate file size, extension, and content before processing.

16. **Using `Math.random()` for IDs.** Two of three ID generation sites in Planora use `Math.random()` instead of the existing `cryptoRandomId()`. Always use the crypto-safe helper.

17. **Not restricting browser permissions.** Without a Permissions Policy, any embedded content can request camera, microphone, geolocation, etc. Explicitly deny all unused permissions and allow only `camera=(self)` for QR scanning.

18. **Ignoring Trusted Types.** Once CSP Phase 3 is stable, Trusted Types (Phase 4) can structurally prevent DOM XSS — making it impossible to forget `escapeHtml()`. Plan for it early.
