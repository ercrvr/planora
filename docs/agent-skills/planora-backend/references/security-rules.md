# Firestore Security Rules

Firestore Security Rules are Planora's entire backend authorization layer. Without a custom server, these rules are the only thing preventing unauthorized data access. Every rule must be airtight.

## Table of Contents

1. [Security Model](#1-security-model)
2. [Base Rules](#2-base-rules)
3. [Field-Level Validation](#3-field-level-validation)
4. [Rate Limiting Patterns](#4-rate-limiting-patterns)
5. [Rules Deployment](#5-rules-deployment)
6. [Testing Security Rules](#6-testing-security-rules)
7. [Common Mistakes](#7-common-mistakes)

---

## 1. Security Model

### Principles

1. **User isolation:** Users can only read and write their own data. No cross-user access is possible.
2. **Authentication required:** Every read and write requires a valid Firebase Auth token.
3. **Structure validation:** Documents must match expected schemas — reject malformed writes.
4. **Size limits:** Prevent abuse by limiting document sizes.
5. **No admin access from client:** There is no admin SDK or service account in the client. All access is user-scoped.

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| User A reads User B's data | Rules enforce `request.auth.uid == userId` on every path |
| Unauthenticated access | Rules require `request.auth != null` |
| Oversized documents (DoS) | Rules check `request.resource.data` size |
| Malformed data injection | Rules validate field types and required fields |
| Deleted user's data accessed | Firestore rules still enforce auth — deleted auth = no access |
| Forged auth tokens | Firebase Auth verifies tokens server-side before rules evaluate |

---

## 2. Base Rules

### Minimal Viable Rules (Phase 1)

Start with this — it's simple, correct, and sufficient for launch:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Block all access by default
    match /{document=**} {
      allow read, write: if false;
    }

    // User data — only the authenticated owner can access
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null
                         && request.auth.uid == userId;
    }
  }
}
```

**Why the default deny?** The `/{document=**}` block denies all access except what's explicitly allowed. This ensures that if any new collections are added accidentally, they're locked down by default.

**Why `{document=**}` under users?** The recursive wildcard matches all subcollections (state/main, state/alerts, meta/sync, etc.) with a single rule. This is appropriate because the ownership check is the same for all subcollections under a user.

### Enhanced Rules (Phase 2)

Add field validation and document-specific constraints:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Default deny
    match /{document=**} {
      allow read, write: if false;
    }

    match /users/{userId} {
      // Helper: verify the request is from the document owner
      function isOwner() {
        return request.auth != null && request.auth.uid == userId;
      }

      // Helper: limit document size (approximate — Firestore doesn't expose exact size)
      function isReasonableSize() {
        return request.resource.size() < 950000;  // ~950 KiB
      }

      // User profile document (if needed in future)
      allow read, write: if isOwner();

      // State documents
      match /state/{docId} {
        allow read: if isOwner();
        // Size check only on create/update — request.resource is null on delete
        allow create, update: if isOwner()
                              && isReasonableSize()
                              && docId in ['main', 'alerts', 'notifications'];
        allow delete: if isOwner()
                      && docId in ['main', 'alerts', 'notifications'];
      }

      // Sync metadata
      match /meta/{docId} {
        allow read: if isOwner();
        allow create, update: if isOwner()
                              && docId in ['sync'];
        allow delete: if isOwner()
                      && docId in ['sync'];
      }

      // Future: per-entity subcollections (if document splitting is needed)
      match /data/{collection}/{entityId} {
        allow read: if isOwner();
        allow create, update: if isOwner()
                              && isReasonableSize()
                              && collection in ['cards', 'events', 'expenses'];
        allow delete: if isOwner()
                      && collection in ['cards', 'events', 'expenses'];
      }
    }
  }
}
```

### Key Design Decisions

**`isOwner()` function:** Centralizes the ownership check. If additional conditions are needed later (e.g., email verification), update one place.

**Document ID allowlist:** The `docId in ['main', 'alerts', 'notifications']` check prevents clients from creating arbitrary documents in the `state/` subcollection. Even if the client SDK is compromised, it can only write to expected documents.

**Size check:** `request.resource.size() < 950000` prevents a malicious or buggy client from writing enormous documents. The Firestore hard limit is 1 MiB, but catching it at 950 KiB gives headroom.

---

## 3. Field-Level Validation

### Validating `state/main` (Phase 3)

For production hardening, add field-level validation. This catches bugs and protects against tampered clients:

```
match /state/main {
  allow read: if isOwner();
  // Size + field validation on create/update only — request.resource is null on delete
  allow create, update: if isOwner()
                        && isReasonableSize()
                        && request.resource.data.keys().hasAny([
                             'baseZone', 'cards', 'events', '_schemaVersion', '_lastModified'
                           ])
                        && request.resource.data._schemaVersion is int
                        && request.resource.data._lastModified is string;
  allow delete: if isOwner();
}
```

### Validating Array Items

Firestore rules can validate array contents, but it's verbose and has limits (no looping). For Planora, the most practical approach is:
- Validate top-level field types and presence
- Validate array length bounds
- Leave per-item validation to the client (with the security skill's sanitization patterns)

```
// Example: validate cards array bounds
&& request.resource.data.cards.size() <= 100   // Max 100 cards
&& request.resource.data.events.size() <= 2000 // Max 2000 events
&& request.resource.data.expenses.size() <= 5000 // Max 5000 expenses
```

### When to Skip Deep Validation

Firestore rules execute on every read/write and have a 10-second timeout. Overly complex validation slows down writes and can timeout with large documents. The guideline:
- Validate structure and types: yes
- Validate string format (regex): only for critical fields
- Validate every array item: no (let the client handle it)

---

## 4. Rate Limiting Patterns

Firestore Security Rules don't have built-in rate limiting. However, you can implement lightweight throttling:

### Write Frequency Check

Track the last write timestamp in the document and require minimum intervals:

```
// In state/main rule (for create/update — safe because it only checks `resource`, not `request.resource`):
allow create, update: if isOwner()
                      && (
                        // First write (no existing data) — always allowed
                        !exists(resource)
                        ||
                        // Subsequent writes — at least 1 second between writes
                        request.time > resource.data._lastModified.toMillis() + duration.value(1, 's')
                      );
// Deletes don't need rate limiting — they're rare and idempotent
allow delete: if isOwner();
```

**Note:** This is a soft defense. The client controls `_lastModified`, so a malicious client could bypass it. For true rate limiting, you'd need Cloud Functions (not available on Spark plan). This pattern catches accidental rapid writes from buggy clients.

### Document Count Limiting

Prevent unbounded subcollection growth (relevant if you adopt the splitting strategy):

```
// In data/{collection}/{entityId} rule:
// Unfortunately, Firestore rules can't count documents in a collection.
// Enforce limits in client code and use the array size limits in the parent document.
```

---

## 5. Rules Deployment

### File Structure

```
project-root/
├── firestore.rules          ← Security rules file
├── firestore.indexes.json   ← Index definitions (usually empty for Planora)
└── firebase.json            ← Project configuration
```

### Deploying Rules

```bash
# Deploy rules only
firebase deploy --only firestore:rules

# Deploy everything (rules + indexes)
firebase deploy --only firestore
```

### Rules in Version Control

**Always commit `firestore.rules` to the repo.** Security rules are code — they need version control, code review, and change tracking.

Store in: `planora/firebase/firestore.rules`

### Pre-Deploy Checklist

Before deploying rule changes:

1. Test with the Firebase Emulator rules playground
2. Test with a signed-in user — verify own data access works
3. Test with a different UID — verify cross-user access is denied
4. Test with unauthenticated request — verify access is denied
5. Test document size limits with a large payload
6. Review the diff — never deploy rules you haven't tested

---

## 6. Testing Security Rules

### Using the Emulator

The Firebase Emulator includes a rules playground:

1. Start emulator: `firebase emulators:start`
2. Open Emulator UI: `http://localhost:4000`
3. Go to Firestore → Rules Playground
4. Test reads and writes with different auth states

### Test Cases

#### ✅ Authenticated user reads own data
```
Path: users/user123/state/main
Method: get
Auth: { uid: 'user123' }
Expected: ALLOWED
```

#### ❌ Authenticated user reads another user's data
```
Path: users/user456/state/main
Method: get
Auth: { uid: 'user123' }
Expected: DENIED
```

#### ❌ Unauthenticated user reads any data
```
Path: users/user123/state/main
Method: get
Auth: none
Expected: DENIED
```

#### ✅ Authenticated user writes own state
```
Path: users/user123/state/main
Method: set
Auth: { uid: 'user123' }
Data: { baseZone: 'Asia/Manila', cards: [], ... }
Expected: ALLOWED
```

#### ❌ Write to invalid document name
```
Path: users/user123/state/hacked
Method: set
Auth: { uid: 'user123' }
Expected: DENIED (if using Phase 2 rules with docId allowlist)
```

#### ✅ Authenticated user deletes own document
```
Path: users/user123/state/main
Method: delete
Auth: { uid: 'user123' }
Expected: ALLOWED
```

#### ❌ Oversized document write
```
Path: users/user123/state/main
Method: set
Auth: { uid: 'user123' }
Data: (> 950 KiB)
Expected: DENIED
```

### Automated Testing with @firebase/rules-unit-testing

For CI/CD integration (if you add a build step later):

```js
const { initializeTestEnvironment, assertSucceeds, assertFails } = require('@firebase/rules-unit-testing');

const testEnv = await initializeTestEnvironment({
  projectId: 'planora-test',
  firestore: {
    rules: fs.readFileSync('firestore.rules', 'utf8')
  }
});

// Test: owner can read
const ownerDb = testEnv.authenticatedContext('user123').firestore();
await assertSucceeds(ownerDb.doc('users/user123/state/main').get());

// Test: non-owner cannot read
const otherDb = testEnv.authenticatedContext('user456').firestore();
await assertFails(otherDb.doc('users/user123/state/main').get());

// Test: unauthenticated cannot read
const unauthedDb = testEnv.unauthenticatedContext().firestore();
await assertFails(unauthedDb.doc('users/user123/state/main').get());
```

---

## 7. Common Mistakes

1. **Using `allow read, write: if true` during development.** This gives everyone access to all data. Even temporarily, it's dangerous if you forget to revert. Use the emulator instead.

2. **Forgetting the default deny rule.** Without `match /{document=**} { allow read, write: if false; }`, new collections might be accessible by default (depending on Firestore version and configuration).

3. **Not testing with a different UID.** Always verify that User A cannot access User B's documents. It's the most critical security property.

4. **Overly complex rules that timeout.** Firestore rules have a 10-second execution limit. If rules involve deep field traversal on large documents, they may timeout and silently deny access. Keep rules simple.

5. **Deploying rules without testing.** A bad rule deployment can lock all users out of their data (if rules deny legitimate access) or expose everyone's data (if rules are too permissive). Always test in the emulator first.

6. **Assuming `request.resource.data` matches exactly what the client sent.** The `data` in rules includes all fields of the resulting document after the write — for `update()`, this includes existing fields not in the client's payload. Use `set()` with full documents for predictable validation.

7. **Not using `{document=**}` for recursive matching.** The rule `match /users/{userId}` only matches the user document itself, not subcollections. Use `match /users/{userId}/{document=**}` to match the entire subtree, or explicitly match each subcollection.

8. **Hardcoding test UIDs in production rules.** Never add `|| request.auth.uid == 'myDebugUid'` to production rules. Use the emulator for testing access.

9. **Using `allow write` with `request.resource` checks.** `request.resource` is **null on delete operations**. If you use `allow write: if isReasonableSize()`, deletes will always fail because `request.resource.size()` throws on null. Always separate delete rules from create/update rules when using `request.resource` or `request.resource.data` checks.
