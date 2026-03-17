# Security Audit Checklist

Step-by-step workflow for performing a security audit of Planora, producing findings, and prioritizing remediation.

## Table of Contents

1. [Audit Workflow](#audit-workflow)
2. [Checklist by Category](#checklist-by-category)
3. [Severity Classification](#severity-classification)
4. [Report Template](#report-template)
5. [Remediation Prioritization](#remediation-prioritization)

---

## Audit Workflow

### Pre-Audit

1. **Pull the latest code** from the `main` branch
2. **Read the changelog** since the last audit to focus on new/changed areas
3. **Set up tooling:**
   - Browser DevTools for runtime inspection
   - A text editor with regex search for static analysis
   - Network monitoring (DevTools Network tab)

### During Audit

1. Run through each category in the checklist below
2. Document every finding with:
   - Location (file, line number)
   - Description of the issue
   - Severity (Critical / High / Medium / Low / Info)
   - Recommended fix
   - Evidence (screenshot, code snippet)
3. Test the application while monitoring the console and network tabs
4. Try injecting payloads through every input path

### Post-Audit

1. Compile findings into a report using the template below
2. Prioritize remediation by severity and effort
3. Create issues/tasks for each finding
4. Schedule a follow-up audit after fixes are applied

---

## Checklist by Category

### 1. Cross-Site Scripting (XSS)

- [ ] **Search for all innerHTML assignments** — `grep -n "\.innerHTML\s*=" index.html`
- [ ] **For each innerHTML, verify** all dynamic values pass through `escapeHtml()` or equivalent
- [ ] **Check for innerHTML in dangerous contexts** — href, src, style, event handlers
- [ ] **Test XSS payloads in every input field:**
  ```
  <img src=x onerror=alert(1)>
  <svg onload=alert(1)>
  "><script>alert(1)</script>
  javascript:alert(1)
  ```
- [ ] **Test XSS via localStorage** — Manually modify localStorage values with payloads, then reload
- [ ] **Test XSS via data import** — Create a JSON backup with payloads in string fields, import it
- [ ] **Test XSS via QR scan** — Generate a QR code with a malicious payload
- [ ] **Check for DOM clobbering** — Elements with IDs matching global variable names
- [ ] **Verify textContent is used** for all simple text rendering

### 2. Data Storage

- [ ] **List all localStorage keys** and classify their sensitivity
- [ ] **Check for sensitive data in plain text** — financial data, personal information
- [ ] **Verify no credentials** are stored in localStorage (API keys, passwords, tokens)
- [ ] **Check `__debugSecret`** — Is it still using a default password?
- [ ] **Verify data deletion** — Does "delete all data" actually clear everything?
- [ ] **Check IndexedDB** for stored data and its sensitivity
- [ ] **Test localStorage quota handling** — What happens when storage is full?

### 3. Network & API Security

- [ ] **List all fetch/XHR calls** and their destinations
- [ ] **Check for hardcoded API keys or tokens** — `grep -n "api[_-]key\|token\|secret\|password\|Bearer" index.html`
- [ ] **Verify HTTPS** is used for all external requests
- [ ] **Check for open redirects** — Any `window.location = userInput` patterns
- [ ] **Monitor network traffic** during normal use — are there unexpected requests?
- [ ] **Test API error handling** — What happens when the currency API is down?

### 4. Content Security Policy

- [ ] **Check for CSP meta tag** — Is one present? Is it effective?
- [ ] **List all inline scripts** and evaluate if they can be externalized
- [ ] **List all inline event handlers** (onclick, onsubmit, etc.)
- [ ] **Test CSP violations** — Open the console and check for CSP-related errors
- [ ] **Verify CSP covers all directives** — script-src, style-src, img-src, connect-src, etc.

### 5. Third-Party Dependencies

- [ ] **List all external scripts** with their CDN URLs and versions
- [ ] **Check for SRI hashes** on all external script/link tags
- [ ] **Verify versions are pinned** (exact, not ranges)
- [ ] **Check for known vulnerabilities** — Search npm advisories for each package
- [ ] **Verify crossorigin attribute** is present on external resources
- [ ] **Check for unused dependencies** — Is everything loaded actually used?

### 6. Data Import/Export

- [ ] **Test JSON import validation** — Try importing malformed JSON
- [ ] **Test with oversized imports** — 100MB+ JSON files
- [ ] **Test with unexpected field types** — Numbers where strings are expected, etc.
- [ ] **Test with extra fields** — Does the import blindly merge unknown properties?
- [ ] **Test prototype pollution via import** — Import JSON with `__proto__`, `constructor`, and `prototype` keys:
  ```json
  {
    "events": [{"__proto__": {"polluted": true}}],
    "__proto__": {"isAdmin": true},
    "constructor": {"prototype": {"injected": 1}}
  }
  ```
  After import, check: `({}).polluted`, `({}).isAdmin`, `({}).injected` — all should be `undefined`
- [ ] **Verify safe JSON parsing** — Is `safeJSONParse` (with reviver) used instead of raw `JSON.parse` for external data?
- [ ] **Verify field allowlisting** — Are imported objects reconstructed from known fields only (not spread/assigned)?
- [ ] **Verify export doesn't leak internal state** — Debug data, secret keys
- [ ] **Test QR import** with crafted payloads
- [ ] **Test blob URL cleanup** — After exporting, check `performance.memory` or DevTools for blob URL leaks
- [ ] **Test decompression bomb** — Create a QR payload that compresses to <2200 bytes but decompresses to >1MB. Verify the app rejects it before `JSON.parse()`
- [ ] **Verify decompressed size check** — Is `raw.length` checked against a maximum before parsing? Should be ≤512KB
- [ ] **Test file input hardening** — Try uploading: a 100MB file, a .txt file renamed to .json, a valid .json file with nested objects >10 levels deep
- [ ] **Verify file size check before read** — Is `file.size` validated before `FileReader.readAsText()`?

### 7. Environment & Configuration

- [ ] **Search for hardcoded credentials** — Passwords, API keys, tokens anywhere in source
- [ ] **Check .gitignore** — Are .env files excluded?
- [ ] **Verify no secrets in git history** — Use `git log -p --all -S "password\|secret\|api_key"`
- [ ] **Check deployment configuration** — Are environment variables properly injected?
- [ ] **Verify debug mode** can't be accidentally enabled in production

### 8. Error Handling & Information Disclosure

- [ ] **Check all try/catch blocks** — Do they handle errors securely (no stack traces shown to users)?
- [ ] **Test with invalid data in every input** — Does the app fail gracefully or show raw error details?
- [ ] **Check console output** — Run through the app normally and check console for leaked sensitive data (localStorage contents, full state objects, API responses)
- [ ] **Verify API failure handling** — Disconnect the network and check how the currency API failure is displayed. It should show a user-friendly message, not a raw error object
- [ ] **Check for silent error swallowing** — `catch(e) {}` blocks that hide failures and leave the app in an inconsistent state
- [ ] **Verify error boundaries** — Critical operations (save, import, export) should catch errors and inform the user rather than silently failing
- [ ] **Test localStorage quota exceeded** — Fill localStorage to capacity and verify the app handles `QuotaExceededError` gracefully

### 9. Security Logging & Monitoring

- [ ] **Check for CSP violation reporting** — Is `report-uri` or `report-to` configured in the CSP?
- [ ] **Verify localStorage integrity checks** — Does the app detect tampered/malformed localStorage data on load?
- [ ] **Check for security-relevant console warnings** — Are prototype pollution attempts, validation failures, or schema mismatches logged?
- [ ] **Verify no sensitive data in logs** — Console output should not contain passwords, full financial records, or encryption keys
- [ ] **Test tampered localStorage** — Manually corrupt a localStorage key and verify the app recovers or warns rather than crashing

### 10. General

- [ ] **Check for `eval()`** and `Function()` usage
- [ ] **Check for `document.write()`**
- [ ] **Check for `setTimeout`/`setInterval` with string arguments**
- [ ] **Check for console.log statements** that might leak sensitive data
- [ ] **Test on different browsers** — Security behavior can vary
- [ ] **Check for timing-safe comparisons** — Are secret values (like debug passwords) compared using constant-time methods?
- [ ] **Check URL validation** — Are user-provided URLs validated before use in `href`, `src`, or navigation?
- [ ] **Verify blob URL cleanup** — Are `URL.revokeObjectURL()` calls present after downloads/previews?
- [ ] **Check for DOM clobbering risks** — Search for element IDs that match JavaScript variable names: `grep -oP 'id="([^"]+)"' index.html | sort -u` and cross-reference with bare global variable lookups
- [ ] **Verify namespaced ID prefixes** — New elements should use a prefix like `pl-` to avoid variable collision
- [ ] **Check ID generation consistency** — Are all ID generation sites using `cryptoRandomId()`? Search for `Math.random()` and verify none are used for identifiers
- [ ] **Check Permissions Policy** — Is `Permissions-Policy` header set? Does it allow `camera=(self)` and deny all other sensitive features?
- [ ] **Check Trusted Types readiness** — If CSP Phase 3+ is deployed, is `require-trusted-types-for 'script'` in report-only mode? Are there violations to fix?
- [ ] **Verify clipboard data sensitivity** — Does the "copy to clipboard" feature include only non-sensitive schedule data? No financial records or internal IDs?
- [ ] **Check CSS injection safety** — Are user-controlled values (theme colors) validated with strict regex before insertion into style properties?

---

## Severity Classification

| Severity | Description | Examples | SLA |
|----------|-------------|---------|-----|
| **Critical** | Immediate exploitation possible, data at risk | Stored XSS with no escaping, exposed API key with write access | Fix within 24 hours |
| **High** | Exploitation possible with some effort | Missing CSP allowing injected scripts, unvalidated data imports | Fix within 1 week |
| **Medium** | Potential risk requiring specific conditions | innerHTML without escaping (data from localStorage), missing SRI | Fix within 1 month |
| **Low** | Minor risk, defense-in-depth concern | Missing security headers, debug mode accessible | Fix in next release |
| **Info** | Observation, not directly exploitable | Outdated dependency (no known vuln), code quality concern | Track and review |

---

## Report Template

Save audit reports to `dev/audit/reports/security-audit-YYYY-MM-DD.md`:

```markdown
# Security Audit Report — [Date]

## Summary

| Severity | Count |
|----------|-------|
| Critical | 0 |
| High | 0 |
| Medium | 0 |
| Low | 0 |
| Info | 0 |

**Auditor:** [Name]
**Codebase version:** [Git commit hash]
**Scope:** [Full / Partial — specify which areas]

## Findings

### [SEVERITY]-001: [Title]

**Location:** `index.html`, line [N]
**Description:** [What the issue is]
**Impact:** [What an attacker could do]
**Evidence:**
```
[Code snippet or screenshot]
```
**Recommendation:** [How to fix it]
**Status:** Open / In Progress / Fixed

---

[Repeat for each finding]

## Recommendations Summary

1. [Prioritized list of actions]
2. ...

## Next Steps

- [ ] [Action items with owners and dates]
```

---

## PR / Code Review Security Quick-Check

Use this lightweight checklist for every pull request or code change that touches security-sensitive areas. This is NOT a replacement for a full audit — it's a fast sanity check for day-to-day development.

### When to Use This Checklist

Any PR that modifies:
- DOM rendering code (innerHTML, createElement, textContent)
- Data storage/retrieval (localStorage, IndexedDB)
- Data import/export paths
- External resource loading (scripts, stylesheets, fonts, APIs)
- URL handling or navigation
- Error handling or logging

### The Quick-Check (5 minutes)

```
□ innerHTML safety
  - Every innerHTML with dynamic values uses escapeHtml()
  - No unescaped user data in href, src, style, or event handler attributes

□ Prototype pollution
  - No Object.assign/spread from untrusted external data
  - Import paths use safeJSONParse and field allowlisting

□ URL safety
  - User-provided URLs validated with sanitizeURL() before use in href/src/navigation
  - No open redirects (window.location = userInput)

□ Secret hygiene
  - No hardcoded API keys, passwords, or tokens
  - No new console.log() calls that output sensitive data
  - No sensitive data added to localStorage without encryption consideration

□ Dependency safety
  - New external resources have SRI hashes
  - CDN versions are pinned (exact, not ranges)
  - CSP updated if new external origins are referenced

□ Error handling
  - New try/catch blocks don't swallow errors silently
  - Error messages shown to users don't include stack traces or internal details
  - Failed operations leave the app in a consistent state

□ Blob cleanup
  - Any new URL.createObjectURL() has a matching URL.revokeObjectURL()

□ DOM clobbering
  - New element IDs use namespaced prefix (pl-*)
  - No bare global variable lookups that could collide with element IDs

□ Randomness
  - All new ID generation uses cryptoRandomId() — no Math.random()

□ File/data input
  - File inputs check file.size before reading
  - Decompressed data checked for size before JSON.parse()
  - Clipboard writes don't include sensitive data (financial, internal IDs)

□ Data changes
  - New localStorage keys documented in data-protection.md
  - Deletion logic updated if new data types are introduced
```

---

## Remediation Prioritization

When multiple findings exist, prioritize by:

### Priority Matrix

| | Low Effort | High Effort |
|---|---|---|
| **High Impact** | **Do first** — Quick wins that significantly reduce risk | **Plan next** — Important but requires architecture changes |
| **Low Impact** | **Batch together** — Do in a cleanup sprint | **Defer** — Track but don't prioritize |

### Planora-Specific Priority Order

Based on the current security posture:

1. **Add prototype pollution guards to import paths** — **Critical** impact, low effort. Add `safeJSONParse` reviver + field allowlisting to JSON import and QR scan paths.
2. **Add decompression size check before JSON.parse** — **Critical** impact, low effort. One `if (raw.length > MAX)` check on the QR scan path (~line 13058).
3. **Add file input hardening (pre-read validation)** — High impact, low effort. Add `file.size` and extension checks before `FileReader.readAsText()`.
4. **Add CSP meta tag (Phase 1)** — High impact, low effort. Copy the recommended CSP into the `<head>`.
5. **Add SRI hashes to CDN scripts** — High impact, low effort. Generate hashes and add `integrity` attributes.
6. **Add URL validation for dynamic href/src** — High impact, low effort. Add `sanitizeURL()` utility and apply to all user-originated URLs.
7. **Standardize ID generation to cryptoRandomId()** — Medium impact, low effort. Replace 2 `Math.random()` call sites (~lines 8829, 10600).
8. **Audit innerHTML for missing escapeHtml()** — Medium impact, medium effort. Systematic grep-and-review.
9. **Remove/secure `__debugSecret`** — Medium impact, low effort. Remove the default password.
10. **Add schema validation to data import** — Medium impact, medium effort. Write validation functions.
11. **Add blob URL cleanup to export paths** — Low impact, low effort. Add `URL.revokeObjectURL()` after downloads.
12. **Standardize error handling** — Medium impact, medium effort. Audit try/catch blocks, add user-friendly messages, remove raw error display.
13. **Add Permissions Policy header** — Medium impact, low effort. Add `camera=(self)` and deny all other sensitive permissions.
14. **Add CSP violation reporting** — Low impact, low effort. Add `report-uri` or `report-to` directive.
15. **Encrypt sensitive localStorage data** — Medium impact, high effort. Requires key management UX.
16. **Move JS to external file** — Medium impact, high effort. Enables stricter CSP (Phase 2→3).
17. **Deploy Trusted Types (CSP Phase 4)** — High impact, high effort. Requires wrapping all 73 innerHTML sites in policy. Start with report-only mode.
18. **Namespace element IDs** — Low impact, high effort. Rename 420+ IDs to use `pl-` prefix. Do incrementally with other refactoring.
19. **Set up API proxy** — Impact depends on future API integrations. Plan when needed.
20. **Add formal data migration framework** — Low impact, medium effort. Implement versioned migration pipeline for future schema changes.
