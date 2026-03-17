# Data Protection

How to handle sensitive data storage, encryption, and data transfer in Planora.

## Table of Contents

1. [localStorage Security Model](#localstorage-security-model)
2. [The Debug Secret Problem](#the-debug-secret-problem)
3. [Encrypting Sensitive Data](#encrypting-sensitive-data)
4. [Data Export Security](#data-export-security)
5. [Data Import Validation](#data-import-validation)
6. [QR Transfer Security](#qr-transfer-security)
7. [Secure Data Migration](#secure-data-migration)
8. [Privacy and Data Minimization](#privacy-and-data-minimization)

---

## localStorage Security Model

### Current State

Planora stores all user data across 7 localStorage keys:

| Key | Content | Sensitivity |
|-----|---------|-------------|
| `timezone_planner_v7_events` | User events and schedules | Medium |
| `timezone_planner_alerts_v1` | Alert/notification settings | Low |
| `timezone_planner_theme` | Theme preference (light/dark) | None |
| `timezone_planner_theme_v1` | Theme preference (v2) | None |
| `tz_skip_welcome` | UI state flag | None |
| `__debugSecret` | Debug mode password | Medium |
| `idb_*` | IndexedDB fallback data | Medium |

Additionally, wallet/expense data (financial information) is stored via the IndexedDB abstraction layer using `idb_*` prefixed keys.

### Threats to localStorage

1. **Browser extensions** — Any extension with `storage` or `tabs` permissions can read localStorage for any origin. Users may not realize a coupon-finder extension can see their financial data.

2. **Physical access** — Anyone with access to the device can open DevTools and read all localStorage.

3. **XSS** — A successful XSS attack can exfiltrate all localStorage data in a single `JSON.stringify(localStorage)`.

4. **Shared devices** — Public or family computers where multiple people use the same browser profile.

### Defense Strategy

For theme preferences and UI flags, plain text storage is fine. For user events, financial data, and any PII, consider encryption at rest. See [Encrypting Sensitive Data](#encrypting-sensitive-data).

---

## The Debug Secret Problem

**Line 4955:**
```js
window.__debugSecret = localStorage.getItem('__debugSecret') || 'p@ssw0rd';
```

This is problematic for several reasons:

1. **Default password in source code** — `p@ssw0rd` is visible to anyone viewing the HTML source. It's not really a secret.
2. **Stored on window** — `window.__debugSecret` is accessible from any script on the page, including injected extensions.
3. **localStorage persistence** — The secret persists across sessions and is readable in DevTools.
4. **No rate limiting** — The input listener (line 4966) checks every keystroke against the secret with no lockout.

### Recommended Fix

For a client-side app, debug mode can't be truly "secured" since anyone can read the JS. But we can make it less casually accessible:

```js
// Option 1: Remove debug mode from production builds entirely
// Use a build step to strip the debug panel and related code

// Option 2: If debug must ship, use a per-session activation
// that doesn't persist and doesn't have a visible default password
(function() {
  var debugActivated = false;
  
  // Require a URL parameter: ?debug=1
  // This is still not "secure" but doesn't leak a password in source
  if (new URLSearchParams(location.search).has('debug')) {
    debugActivated = true;
  }
  
  window.__toggleDebug = function() {
    if (!debugActivated) return;
    // ... toggle logic
  };
})();
```

---

## Encrypting Sensitive Data

For protecting financial and schedule data at rest, use the Web Crypto API (already partially present — `crypto.randomUUID()` at line 10450).

### Encryption Pattern with Web Crypto

```js
// Key derivation from a user passphrase
async function deriveKey(passphrase, salt) {
  var encoder = new TextEncoder();
  var keyMaterial = await crypto.subtle.importKey(
    'raw', encoder.encode(passphrase), 'PBKDF2', false, ['deriveKey']
  );
  return crypto.subtle.deriveKey(
    { name: 'PBKDF2', salt: salt, iterations: 100000, hash: 'SHA-256' },
    keyMaterial,
    { name: 'AES-GCM', length: 256 },
    false,
    ['encrypt', 'decrypt']
  );
}

// Encrypt data before storing
async function encryptAndStore(key, data, storageKey) {
  var encoder = new TextEncoder();
  var iv = crypto.getRandomValues(new Uint8Array(12));
  var encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv: iv },
    key,
    encoder.encode(JSON.stringify(data))
  );
  var payload = {
    iv: Array.from(iv),
    data: Array.from(new Uint8Array(encrypted))
  };
  localStorage.setItem(storageKey, JSON.stringify(payload));
}

// Decrypt data when reading
async function decryptFromStorage(key, storageKey) {
  var raw = localStorage.getItem(storageKey);
  if (!raw) return null;
  var payload = JSON.parse(raw);
  var decrypted = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv: new Uint8Array(payload.iv) },
    key,
    new Uint8Array(payload.data)
  );
  return JSON.parse(new TextDecoder().decode(decrypted));
}
```

### When to Encrypt

| Data | Encrypt? | Reason |
|------|----------|--------|
| Financial data (expenses, budgets) | Yes | Sensitive personal information |
| Event schedules | Optional | Depends on user sensitivity needs |
| Theme preferences | No | Not sensitive |
| UI state flags | No | Not sensitive |
| Alert settings | No | Low sensitivity |

### Considerations

- Encryption adds complexity and a key-management burden (if the user forgets their passphrase, data is unrecoverable)
- Consider making encryption opt-in with a "Lock with password" feature
- Store the salt alongside the encrypted data (salts don't need to be secret)
- Never store the passphrase itself

---

## Data Export Security

The backup export (line ~10112) creates a JSON blob of all app state:

```js
var blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
```

### Security Checklist for Exports

1. **Strip debug state** — Don't include `__debugSecret` or internal debug data in exports
2. **Validate data completeness** — Ensure the export contains what the user expects, nothing more
3. **Consider encrypted exports** — For sensitive data, offer password-protected export (encrypt the JSON blob)
4. **Include a version field** — For forward compatibility and import validation
5. **Sanitize before export** — Strip any HTML tags from data fields that shouldn't contain them

```js
function prepareExportData(state) {
  var exportData = {
    version: '1.0',
    exportedAt: new Date().toISOString(),
    cards: state.cards.filter(isPersistableCard),
    events: state.events || [],
    // Explicitly list what's included — don't spread the whole state
  };
  // Strip debug/internal fields
  delete exportData.__debug;
  return exportData;
}
```

---

## File Input Hardening

The `<input type="file" accept=".json">` on line ~3830 only provides a UI hint — it does **not** enforce file type. A user (or attacker with physical access) can select any file.

### Pre-Read Validation

Before calling `FileReader.readAsText()`, validate the file object itself:

```js
var MAX_IMPORT_FILE_SIZE = 2 * 1024 * 1024; // 2 MB

function handleFileImport(file) {
  // 1. Check file size before reading
  if (file.size > MAX_IMPORT_FILE_SIZE) {
    showError('File too large. Maximum import size is 2 MB.');
    return;
  }

  // 2. Check file extension (MIME types are unreliable for .json)
  if (!file.name.toLowerCase().endsWith('.json')) {
    showError('Please select a .json file.');
    return;
  }

  // 3. Optionally check MIME type (not all browsers set it correctly)
  if (file.type && file.type !== 'application/json' && file.type !== 'text/plain') {
    showError('Unexpected file type. Please select a JSON file.');
    return;
  }

  // 4. Read with size and parse safety
  var reader = new FileReader();
  reader.onload = function(e) {
    var content = e.target.result;

    // Double-check content length (belt and suspenders)
    if (content.length > MAX_IMPORT_FILE_SIZE) {
      showError('File content exceeds maximum allowed size');
      return;
    }

    try {
      var data = safeJSONParse(content);  // Prototype-safe parsing
      var validated = validateImportPayload(data);  // Schema validation
      applyImport(validated);
    } catch (err) {
      showError('Invalid file format. Please ensure this is a valid Planora export.');
    }
  };
  reader.onerror = function() {
    showError('Failed to read file.');
  };
  reader.readAsText(file);
}
```

### Why `accept=".json"` Is Not Enough

The `accept` attribute:
- ✅ Filters the file picker UI (most files are hidden)
- ❌ Does NOT prevent drag-and-drop of arbitrary files
- ❌ Does NOT prevent manual file type selection ("All Files")
- ❌ Does NOT validate file contents

Always treat the file as untrusted input regardless of the `accept` attribute.

---

## Data Import Validation

The import paths (JSON file upload at line ~10128, QR scan at line ~12926) accept external data. This is the highest-risk surface in the application.

### Schema Validation Pattern

```js
function validateImportedEvent(obj) {
  // Type checks
  if (typeof obj !== 'object' || obj === null) return null;
  if (typeof obj.title !== 'string') return null;
  if (typeof obj.id !== 'string') return null;

  // Length limits — prevent memory exhaustion and rendering issues
  var MAX_TITLE_LENGTH = 200;
  var MAX_NOTE_LENGTH = 5000;

  return {
    id: obj.id.slice(0, 64),
    title: obj.title.slice(0, MAX_TITLE_LENGTH),
    note: typeof obj.note === 'string' ? obj.note.slice(0, MAX_NOTE_LENGTH) : '',
    // Validate dates
    startDate: isValidISODate(obj.startDate) ? obj.startDate : null,
    endDate: isValidISODate(obj.endDate) ? obj.endDate : null,
    // Validate enums
    type: VALID_EVENT_TYPES.includes(obj.type) ? obj.type : 'default',
    // Strip unexpected fields entirely
  };
}

function validateImportPayload(data) {
  if (!data || typeof data !== 'object') throw new Error('Invalid import format');
  if (!data.version) throw new Error('Missing version field');

  // Cap array sizes
  var MAX_EVENTS = 10000;
  var MAX_CARDS = 500;

  var events = Array.isArray(data.events) ? data.events.slice(0, MAX_EVENTS) : [];
  var cards = Array.isArray(data.cards) ? data.cards.slice(0, MAX_CARDS) : [];

  return {
    events: events.map(validateImportedEvent).filter(Boolean),
    cards: cards.map(validateImportedCard).filter(Boolean),
  };
}
```

### Key Principles

1. **Allowlist fields, don't blocklist** — Only copy known-good fields from imported objects. Never spread or merge unknown properties into state.
2. **Enforce length limits** — A malicious import could contain a 100MB string in a title field.
3. **Validate types strictly** — A number where a string is expected (or vice versa) can cause unexpected behavior.
4. **Wrap in try/catch** — JSON.parse and validation can throw. Handle errors gracefully with user feedback.
5. **Reject prototype-polluting keys** — External data may contain `__proto__`, `constructor`, or `prototype` keys designed to pollute the global Object prototype. Always use safe JSON parsing (see `references/secure-coding.md` → Prototype Pollution Prevention) and field allowlisting.

### Prototype-Safe JSON Parsing

Always parse external JSON (from file imports, QR scans, or any external source) using a reviver that strips dangerous keys:

```js
var DANGEROUS_KEYS = new Set(['__proto__', 'constructor', 'prototype']);

function safeJSONParse(jsonString) {
  return JSON.parse(jsonString, function(key, value) {
    if (DANGEROUS_KEYS.has(key)) return undefined;
    return value;
  });
}

// ✅ Use this instead of raw JSON.parse for all external data
var importedData = safeJSONParse(fileContents);
var qrData = safeJSONParse(LZString.decompressFromEncodedURIComponent(qrPayload));
```

This is the first line of defense. The second is the `validateImportedEvent` allowlist pattern above, which ensures only known fields are copied into state.

---

## QR Transfer Security

The QR share/scan feature (lines ~12800-12960) transfers app data between devices by encoding JSON → LZString compression → QR code.

### Current Risks

1. **No encryption** — Data is compressed but not encrypted. Anyone who photographs the QR code can decompress and read the data.
2. **No authentication** — There's no way to verify the QR code came from a trusted source.
3. **Schema validation gaps** — The scan-to-import path should validate imported data as strictly as the file import path.

### Recommended Improvements

1. **Add a warning to users** — "QR codes contain your unencrypted data. Only share in private."
2. **Offer encrypted QR transfer** — Encrypt the payload with a shared passphrase before compression. Both devices enter the same passphrase.
3. **Validate scanned data** — Apply the same `validateImportPayload` schema validation to QR-scanned data.
4. **Limit QR payload size** — The current MAX_QR_BYTES of 2200 is reasonable, but validate that decompressed data doesn't exceed a sane limit.
5. **Guard against decompression bombs** — LZString can decompress a small payload into a very large string. Always check the decompressed size before parsing.

### Decompression Bomb Prevention

The QR scan path (~line 13058) decompresses and immediately parses without size validation:

```js
// ❌ CURRENT — no size check after decompression
var raw = LZString.decompressFromBase64(decoded);
var data = JSON.parse(raw);  // raw could be megabytes
```

A crafted QR payload of ~2200 bytes could decompress to a multi-megabyte string, causing `JSON.parse()` to consume excessive memory and potentially crash the browser tab.

```js
// ✅ SAFE — check decompressed size before parsing
var MAX_DECOMPRESSED_SIZE = 512 * 1024; // 512 KB — generous for event data

var raw = LZString.decompressFromBase64(decoded);
if (!raw) {
  showError('Invalid QR code: decompression failed');
  return;
}
if (raw.length > MAX_DECOMPRESSED_SIZE) {
  showError('QR code data exceeds maximum allowed size');
  return;
}

var data = safeJSONParse(raw); // Use safe parser (see above)
var validated = validateImportPayload(data); // Schema validation
```

**Why 512 KB?** Planora's typical full data export (events + wallet) compresses to ~1-2 KB. Even a power user with thousands of events would be well under 512 KB decompressed. Anything larger is almost certainly malicious.

Apply the same pattern to file imports — check `reader.result.length` before `JSON.parse()`:

```js
reader.onload = function(e) {
  var content = e.target.result;
  if (content.length > MAX_IMPORT_FILE_SIZE) {
    showError('Import file exceeds maximum allowed size');
    return;
  }
  try {
    var data = safeJSONParse(content);
    // ...validate and import
  } catch (err) {
    showError('Invalid JSON format');
  }
};
```

```js
// Encrypted QR payload
async function encryptForQR(data, passphrase) {
  var salt = crypto.getRandomValues(new Uint8Array(16));
  var key = await deriveKey(passphrase, salt);
  var iv = crypto.getRandomValues(new Uint8Array(12));
  var encrypted = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    new TextEncoder().encode(JSON.stringify(data))
  );
  return {
    s: Array.from(salt),
    i: Array.from(iv),
    d: Array.from(new Uint8Array(encrypted))
  };
}
```

---

## Secure Data Migration

### The Challenge

Planora uses versioned localStorage keys (e.g., `timezone_planner_v7_events`, `timezone_planner_alerts_v1`). When the data schema changes across app versions, a migration step reads old data, transforms it, and writes it to the new key. This is a security-sensitive operation because:

1. **Old data may not conform to new validation rules** — Migrated data bypasses the import validation that new data goes through.
2. **Migration bugs can corrupt or lose data** — A failed migration may silently drop events or financial records.
3. **Old keys may linger** — If old localStorage keys aren't cleaned up, stale sensitive data remains accessible.

### Migration Safety Checklist

```
Before writing migration code:
1. □ Define the exact schema difference (old shape → new shape)
2. □ Write a validation function for the NEW schema
3. □ Run migrated data through the same validation as fresh imports
4. □ Handle migration failure gracefully:
     - Keep the old data intact as a fallback
     - Show the user a clear error message
     - Log details for debugging (without leaking sensitive values)
5. □ Clean up old keys ONLY after confirming new data is valid and written
6. □ Test with real-world data volumes (not just one or two events)
```

### Migration Pattern

```js
function migrateEventsV7toV8() {
  var OLD_KEY = 'timezone_planner_v7_events';
  var NEW_KEY = 'timezone_planner_v8_events';

  // 1. Check if migration is needed
  if (localStorage.getItem(NEW_KEY)) return; // already migrated
  var raw = localStorage.getItem(OLD_KEY);
  if (!raw) return; // nothing to migrate

  try {
    // 2. Parse with prototype-safety
    var oldData = safeJSONParse(raw);

    // 3. Transform to new schema
    var newData = oldData.map(function(event) {
      return {
        id: String(event.id || crypto.randomUUID()),
        title: String(event.title || '').slice(0, 200),
        // ... map old fields to new fields
        migratedFrom: 'v7' // track provenance
      };
    });

    // 4. Validate against new schema
    var validated = newData.map(validateEventV8).filter(Boolean);

    // 5. Write new data
    localStorage.setItem(NEW_KEY, JSON.stringify(validated));

    // 6. Clean up old key only after successful write
    localStorage.removeItem(OLD_KEY);

    console.log('Migration v7→v8: ' + validated.length + '/' + oldData.length + ' events migrated');
  } catch (err) {
    // Don't delete old data on failure — keep it as fallback
    console.error('Migration v7→v8 failed — old data preserved:', err.message);
    // Show user-facing warning
    showMigrationWarning('Some data could not be migrated. Your original data is preserved.');
  }
}
```

### Version Tracking

Consider storing a schema version marker:

```js
var SCHEMA_VERSION = 8;
var storedVersion = parseInt(localStorage.getItem('planora_schema_version') || '0', 10);

if (storedVersion < SCHEMA_VERSION) {
  runMigrations(storedVersion, SCHEMA_VERSION);
  localStorage.setItem('planora_schema_version', String(SCHEMA_VERSION));
}
```

---

## Privacy and Data Minimization

### Principles

Even though Planora is offline-first and data stays on the user's device, privacy-by-design principles still apply — especially for financial data in the wallet feature.

1. **Collect only what's needed.** Don't store metadata or derived data that can be recomputed on the fly (e.g., don't cache currency conversion results with timestamps — recompute from rates).

2. **Provide real deletion.** When a user deletes an event, expense, or card:
   - Remove it from the data array (don't just mark as `deleted: true`)
   - If encryption is enabled, the deleted data is unrecoverable after key rotation — which is the desired behavior
   - Clear it from any in-memory caches or undo stacks after a reasonable window

3. **Clear all data completely.** A "Delete All Data" action should:
   ```js
   function clearAllUserData() {
     // Remove all Planora localStorage keys
     var keys = Object.keys(localStorage).filter(function(k) {
       return k.startsWith('timezone_planner') || k.startsWith('tz_') || 
              k.startsWith('idb_') || k === '__debugSecret';
     });
     keys.forEach(function(k) { localStorage.removeItem(k); });

     // Clear IndexedDB if used
     if (window.indexedDB) {
       indexedDB.databases().then(function(dbs) {
         dbs.forEach(function(db) { indexedDB.deleteDatabase(db.name); });
       });
     }

     // Clear in-memory state
     // ... reset all state variables
   }
   ```

4. **Minimize export scope.** When exporting data, include only what's necessary:
   - Don't include UI state, debug flags, or internal metadata
   - Don't include computed/cached values
   - Let the user choose which data categories to export (events only, wallet only, etc.)

5. **Warn about data in QR codes.** QR codes are visual — anyone who photographs the screen captures the data. Show a clear warning before generating a data-sharing QR code: *"This QR code contains your unencrypted data. Only share in a private setting."*

### Data Retention Awareness

| Data Type | Retention Consideration |
|-----------|------------------------|
| Events/schedules | Keep until user deletes; offer bulk cleanup of past events |
| Financial records | User may need for tax purposes; don't auto-delete, but offer archival |
| Theme preferences | Keep indefinitely (not sensitive) |
| Debug/error logs | Don't persist to localStorage; keep in-memory only during session |
| Currency rates cache | Short-lived; overwrite on each fetch, don't accumulate history |

---

## Clipboard Data Sensitivity

Lines ~4988-4992 write schedule data to the clipboard via `navigator.clipboard.writeText()`. While low-risk for a web app, be aware:

### Risks

1. **Other applications can read the clipboard** — Clipboard managers, remote desktop software, and other apps can access copied text. If the copied data includes financial records or personal notes, it's exposed.
2. **Clipboard persists after tab close** — Data stays in the clipboard until the user copies something else.
3. **Social engineering** — An attacker could trick a user into pasting clipboard contents into a form on another site.

### Guidelines

1. **Minimize what's copied.** When copying schedule data to clipboard, include only the essential information (event titles, dates/times). Don't include wallet data, notes with sensitive content, or internal IDs.

2. **Warn users about sensitive data.** If the copy action includes financial information, show a brief toast: *"Schedule copied to clipboard. Clear your clipboard after pasting."*

3. **Don't auto-copy sensitive data.** Never automatically copy API keys, passwords, or encryption keys to clipboard without explicit user action.

4. **Consider a "copy and clear" pattern** for sensitive data:

```js
async function copyWithAutoExpiry(text, expiryMs) {
  await navigator.clipboard.writeText(text);
  showToast('Copied to clipboard');

  // Attempt to clear clipboard after a timeout
  // Note: This only works if the tab is still focused
  setTimeout(async function() {
    try {
      var current = await navigator.clipboard.readText();
      if (current === text) {
        await navigator.clipboard.writeText('');
      }
    } catch (e) {
      // Clipboard read may be blocked — that's okay
    }
  }, expiryMs || 30000); // 30 seconds default
}
```

**Note:** The Clipboard API requires focus/permissions, so auto-clear is best-effort. Don't rely on it as a security guarantee.
