# Planora — Gmail Calendar Sync Feature Spec

> **Feature:** Sync meeting/calendar events from multiple email accounts into Planora via Gmail API  
> **Status:** Draft  
> **Date:** March 15, 2026  
> **Branch:** `feature/gmail-sync`

---

## TABLE OF CONTENTS

1. [Overview & Goals](#1-overview--goals)
2. [User Flow](#2-user-flow)
3. [Prerequisites — Google Cloud Setup](#3-prerequisites--google-cloud-setup)
4. [Architecture & Technical Approach](#4-architecture--technical-approach)
5. [CDN Dependencies](#5-cdn-dependencies)
6. [OAuth2 Authentication Flow](#6-oauth2-authentication-flow)
7. [Gmail API — Email Discovery & Fetching](#7-gmail-api--email-discovery--fetching)
8. [ICS / Meeting Email Parsing](#8-ics--meeting-email-parsing)
9. [Data Mapping — Gmail → Planora Events](#9-data-mapping--gmail--planora-events)
10. [IndexedDB Schema Changes](#10-indexeddb-schema-changes)
11. [Sync Engine](#11-sync-engine)
12. [UI Design — Sync Button & Panel](#12-ui-design--sync-button--panel)
13. [New Event Type — Synced Meeting](#13-new-event-type--synced-meeting)
14. [Settings Integration](#14-settings-integration)
15. [Conflict & Duplicate Handling](#15-conflict--duplicate-handling)
16. [Error Handling](#16-error-handling)
17. [Security & Privacy](#17-security--privacy)
18. [Migration & Backward Compatibility](#18-migration--backward-compatibility)
19. [Testing Strategy](#19-testing-strategy)
20. [Future Enhancements](#20-future-enhancements)

---

## 1. Overview & Goals

### Problem

Users manage work across multiple companies/clients (reflected in Planora's company cards). Meeting invites arrive at different email accounts (work Outlook, client Google Workspace, etc.). Currently, users must manually create events in Planora for every meeting.

### Solution

Enable users to:
1. Set up auto-forwarding from their various email accounts to their personal Gmail
2. Press a **"Sync"** button inside Planora to pull meeting data from Gmail
3. Have meetings automatically parsed and plotted on the Planora calendar

### Goals

- **Zero backend** — all logic runs client-side in the browser (consistent with Planora's architecture)
- **Single HTML file** — no separate files; new code is added inline
- **User-initiated sync** — no background polling; the user controls when sync happens via a button
- **Smart deduplication** — avoid creating duplicate events for the same meeting
- **Source attribution** — synced events are clearly marked with their email source
- **Offline resilience** — synced events persist in IndexedDB and work offline after sync

### Non-Goals (v1)

- Real-time push sync (would require a backend)
- Google Calendar API direct integration (future enhancement)
- Two-way sync (Planora → Gmail)
- Support for non-Gmail providers directly (users forward to Gmail instead)

---

## 2. User Flow

### First-Time Setup (One-Time)

```
1. User opens Planora Settings → "Gmail Sync" option
2. Taps "Connect Gmail Account"
3. Google OAuth2 popup appears → user signs in and grants read-only Gmail access
4. Access token stored in IndexedDB (encrypted key)
5. Toast: "Gmail connected! Use the Sync button on the Calendar page to pull meetings."
```

### Ongoing Sync (Recurring)

```
1. User navigates to Calendar page
2. Taps the "Sync" ☁️ button in the calendar toolbar
3. Loading indicator appears: "Syncing meetings from Gmail..."
4. App queries Gmail API for new meeting emails since last sync
5. Parses .ics attachments and meeting invite content
6. Creates/updates Planora events for each meeting
7. Calendar grid refreshes with new synced events
8. Toast: "Synced 3 new meetings, updated 1"
```

### Sync from Events Page

```
1. User is on Events page
2. Long-press or use FAB action: "Sync from Gmail"
3. Same sync flow as above
4. Events list refreshes with new entries
```

---

## 3. Prerequisites — Google Cloud Setup

The user must configure a Google Cloud project before using this feature. Planora will guide them through this with an in-app setup wizard.

### Required Setup

| Step | Action | Details |
|------|--------|---------|
| 1 | Create Google Cloud Project | [console.cloud.google.com](https://console.cloud.google.com) |
| 2 | Enable Gmail API | APIs & Services → Library → Gmail API → Enable |
| 3 | Configure OAuth Consent Screen | External user type, app name "Planora", add scope `gmail.readonly` |
| 4 | Create OAuth2 Client ID | Type: Web Application. Add authorized JavaScript origins: `https://ercrvr.github.io` and optionally `http://localhost:8080` for development |
| 5 | Copy Client ID | Paste into Planora's Gmail Sync settings |

### OAuth2 Scopes Required

```
https://www.googleapis.com/auth/gmail.readonly
```

Only **read-only** access. Planora never sends, modifies, or deletes emails.

### Client ID Storage

The OAuth2 Client ID is stored in Planora's state (IndexedDB) and is NOT a secret — it's safe to store client-side per Google's documentation for public OAuth2 clients (SPA/browser apps).

---

## 4. Architecture & Technical Approach

### How It Fits Planora's Existing Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Planora (Single HTML)                  │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Company Cards │  │   Events     │  │  Calendar     │  │
│  │ (Home Page)   │  │  (Events Pg) │  │  (Cal Page)   │  │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  │
│         │                 │                   │          │
│         └────────┬────────┘                   │          │
│                  │                            │          │
│         ┌────────▼────────────────────────────▼───────┐  │
│         │              IndexedDB (kv store)           │  │
│         │  tz_main_state (events[], cards[], etc.)    │  │
│         │  tz_alert_state                             │  │
│         │  tz_notifications                           │  │
│         │  tz_gmail_sync  ◄── NEW: sync state         │  │
│         └────────────────────────────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Gmail Sync Engine (NEW)              │    │
│  │                                                    │    │
│  │  ┌─────────┐  ┌──────────┐  ┌────────────────┐   │    │
│  │  │ OAuth2  │  │ Gmail    │  │  ICS Parser    │   │    │
│  │  │ (GIS)   │→ │ API      │→ │  + Email       │   │    │
│  │  │         │  │ Client   │  │  Body Parser   │   │    │
│  │  └─────────┘  └──────────┘  └───────┬────────┘   │    │
│  │                                      │            │    │
│  │                              ┌───────▼────────┐   │    │
│  │                              │ Event Mapper   │   │    │
│  │                              │ (→ Planora     │   │    │
│  │                              │   Event Model) │   │    │
│  │                              └────────────────┘   │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌─────────────── CDN Scripts ──────────────────────┐    │
│  │  qrcode • lz-string • html5-qrcode              │    │
│  │  google GIS • google API loader  ◄── NEW         │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### Key Design Decisions

1. **No backend required** — OAuth2 uses the implicit grant flow via Google Identity Services (GIS), which is designed for SPAs/browser apps
2. **Token management** — Access tokens are short-lived (~1 hour). Refresh handled by re-prompting the user (GIS does not support refresh tokens for browser apps without a backend). Token stored in memory + IndexedDB for session persistence
3. **ICS parsing** — Built-in parser (no external library needed). The iCalendar format is well-defined text (RFC 5545) that can be parsed with regex and string operations
4. **Sync is user-initiated** — The sync button triggers the full flow. No timers, no background polling. This keeps the app simple and avoids API quota issues
5. **Forwarded email detection** — Gmail search query targets forwarded messages with calendar attachments or invite patterns

---

## 5. CDN Dependencies

### New Scripts (Added to `<head>`)

```html
<!-- Google Identity Services — OAuth2 login -->
<script src="https://accounts.google.com/gsi/client" async defer></script>

<!-- Google API Client — Gmail API calls -->
<script src="https://apis.google.com/js/api.js" async defer></script>
```

### Loading Strategy

Both scripts load asynchronously with `async defer`. The Gmail Sync engine initializes only when:
1. Both scripts have loaded (`gapi` and `google.accounts.oauth2` are available)
2. The user has configured their Client ID
3. The user clicks the Sync button

A readiness check prevents sync from running if scripts haven't loaded:

```javascript
function isGmailSyncReady() {
  return typeof gapi !== 'undefined' 
    && typeof google !== 'undefined' 
    && google.accounts 
    && google.accounts.oauth2;
}
```

---

## 6. OAuth2 Authentication Flow

### Token Client Initialization

```javascript
let gmailTokenClient = null;
let gmailAccessToken = null;

function initGmailAuth() {
  const syncState = getGmailSyncState();
  if (!syncState.clientId) return;

  gmailTokenClient = google.accounts.oauth2.initTokenClient({
    client_id: syncState.clientId,
    scope: 'https://www.googleapis.com/auth/gmail.readonly',
    callback: handleAuthResponse,
    error_callback: handleAuthError
  });
}
```

### Auth Response Handling

```javascript
function handleAuthResponse(tokenResponse) {
  if (tokenResponse.error) {
    __appToast('Gmail auth failed: ' + tokenResponse.error, 'error');
    return;
  }
  
  gmailAccessToken = tokenResponse.access_token;
  
  // Store token with expiry
  const syncState = getGmailSyncState();
  syncState.accessToken = tokenResponse.access_token;
  syncState.tokenExpiry = Date.now() + (tokenResponse.expires_in * 1000);
  saveGmailSyncState(syncState);
  
  // Continue with sync if triggered by sync button
  if (_pendingSyncAfterAuth) {
    _pendingSyncAfterAuth = false;
    performGmailSync();
  }
}
```

### Token Lifecycle

| State | Behavior |
|-------|----------|
| No token | User clicks Sync → OAuth popup → get token → sync |
| Valid token (< 1 hour old) | User clicks Sync → sync immediately using stored token |
| Expired token | User clicks Sync → GIS silently refreshes (or shows popup if needed) → sync |
| Revoked token | User clicks Sync → OAuth popup → re-authenticate → sync |

### Token Validation

Before any Gmail API call:

```javascript
function ensureValidToken() {
  return new Promise((resolve, reject) => {
    const syncState = getGmailSyncState();
    
    if (gmailAccessToken && syncState.tokenExpiry > Date.now() + 60000) {
      // Token valid (with 60s buffer)
      resolve(gmailAccessToken);
      return;
    }
    
    // Need new token
    _pendingSyncAfterAuth = true;
    gmailTokenClient.requestAccessToken({ prompt: '' }); // Try silent refresh
  });
}
```

---

## 7. Gmail API — Email Discovery & Fetching

### Search Strategy

Gmail's search operators allow precise targeting of meeting emails:

#### Primary Query (Calendar Invites with .ics)

```
has:attachment filename:ics newer_than:7d
```

This finds all emails with `.ics` calendar attachments from the past 7 days.

#### Secondary Query (Meeting Patterns without .ics)

```
(subject:(invite OR meeting OR "calendar event" OR "has been scheduled" OR "accepted" OR "rsvp") 
 OR from:(calendar-notification@google.com OR noreply@google.com)) 
newer_than:7d
```

This catches Google Calendar notifications and common meeting email patterns that may not have .ics attachments.

#### Forwarded Email Detection

```
in:anywhere (has:attachment filename:ics) newer_than:{days}d
```

The `in:anywhere` ensures forwarded emails (which may go to different labels) are included.

### API Call Flow

```javascript
async function fetchMeetingEmails(daysBack = 7) {
  const query = `has:attachment filename:ics newer_than:${daysBack}d`;
  
  // Step 1: List matching message IDs
  const listResponse = await gmailApiCall(
    `https://www.googleapis.com/gmail/v1/users/me/messages?q=${encodeURIComponent(query)}&maxResults=50`
  );
  
  if (!listResponse.messages || listResponse.messages.length === 0) {
    return [];
  }
  
  // Step 2: Fetch full message details (batched)
  const messages = [];
  for (const msg of listResponse.messages) {
    const detail = await gmailApiCall(
      `https://www.googleapis.com/gmail/v1/users/me/messages/${msg.id}?format=full`
    );
    messages.push(detail);
  }
  
  return messages;
}
```

### Gmail API Helper

```javascript
async function gmailApiCall(url) {
  const response = await fetch(url, {
    headers: {
      'Authorization': `Bearer ${gmailAccessToken}`,
      'Accept': 'application/json'
    }
  });
  
  if (response.status === 401) {
    // Token expired mid-sync
    throw new GmailAuthError('Token expired');
  }
  
  if (!response.ok) {
    throw new GmailApiError(`API error: ${response.status}`);
  }
  
  return response.json();
}
```

### Attachment Extraction

```javascript
async function extractIcsAttachment(messageId, parts) {
  for (const part of parts) {
    if (part.filename && part.filename.endsWith('.ics') && part.body?.attachmentId) {
      const attachment = await gmailApiCall(
        `https://www.googleapis.com/gmail/v1/users/me/messages/${messageId}/attachments/${part.body.attachmentId}`
      );
      // Gmail returns base64url-encoded data
      return atob(attachment.data.replace(/-/g, '+').replace(/_/g, '/'));
    }
    // Recurse into nested parts
    if (part.parts) {
      const result = await extractIcsAttachment(messageId, part.parts);
      if (result) return result;
    }
  }
  return null;
}
```

### Rate Limiting & Pagination

- **Max 50 messages per sync** (configurable in settings)
- **Pagination**: If `nextPageToken` exists, store it and continue on next sync
- **Rate limiting**: 250 quota units per user per second (Gmail API). Each `messages.get` = 5 units. 50 messages = 250 units = ~1 second. Well within limits
- **Batch requests**: Future optimization — use Gmail batch API to fetch multiple messages in one request

---

## 8. ICS / Meeting Email Parsing

### ICS Parser (Built-in)

The iCalendar format (RFC 5545) is structured text that can be parsed without external libraries.

#### Sample .ics Content

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//Google Inc//Google Calendar 70.9054//EN
BEGIN:VEVENT
DTSTART:20260316T090000Z
DTEND:20260316T100000Z
SUMMARY:Sprint Planning
DESCRIPTION:Weekly sprint planning session
LOCATION:Zoom - https://zoom.us/j/123456
ORGANIZER;CN=John Smith:mailto:john@company.com
ATTENDEE;CN=Eric Rivera;PARTSTAT=ACCEPTED:mailto:ercrivera.9192@gmail.com
ATTENDEE;CN=Jane Doe;PARTSTAT=NEEDS-ACTION:mailto:jane@company.com
UID:abc123@google.com
STATUS:CONFIRMED
SEQUENCE:0
END:VEVENT
END:VCALENDAR
```

#### Parser Implementation

```javascript
function parseICS(icsText) {
  const events = [];
  // Unfold continuation lines (RFC 5545: lines starting with space/tab are continuations)
  const unfolded = icsText.replace(/\r?\n[ \t]/g, '');
  const lines = unfolded.split(/\r?\n/);
  
  let currentEvent = null;
  
  for (const line of lines) {
    if (line === 'BEGIN:VEVENT') {
      currentEvent = {};
      continue;
    }
    if (line === 'END:VEVENT') {
      if (currentEvent) events.push(currentEvent);
      currentEvent = null;
      continue;
    }
    if (!currentEvent) continue;
    
    const colonIdx = line.indexOf(':');
    if (colonIdx === -1) continue;
    
    let key = line.substring(0, colonIdx);
    const value = line.substring(colonIdx + 1);
    
    // Strip parameters (e.g., DTSTART;TZID=America/New_York:20260316T090000)
    const semiIdx = key.indexOf(';');
    let params = {};
    if (semiIdx !== -1) {
      const paramStr = key.substring(semiIdx + 1);
      key = key.substring(0, semiIdx);
      // Parse params like TZID=America/New_York
      paramStr.split(';').forEach(p => {
        const [pk, pv] = p.split('=');
        if (pk && pv) params[pk] = pv;
      });
    }
    
    switch (key) {
      case 'SUMMARY':
        currentEvent.summary = unescapeICS(value);
        break;
      case 'DTSTART':
        currentEvent.dtStart = parseICSDateTime(value, params.TZID);
        break;
      case 'DTEND':
        currentEvent.dtEnd = parseICSDateTime(value, params.TZID);
        break;
      case 'DESCRIPTION':
        currentEvent.description = unescapeICS(value);
        break;
      case 'LOCATION':
        currentEvent.location = unescapeICS(value);
        break;
      case 'ORGANIZER':
        currentEvent.organizer = extractMailto(value);
        currentEvent.organizerName = params.CN || '';
        break;
      case 'ATTENDEE':
        if (!currentEvent.attendees) currentEvent.attendees = [];
        currentEvent.attendees.push({
          email: extractMailto(value),
          name: params.CN || '',
          status: params.PARTSTAT || 'NEEDS-ACTION'
        });
        break;
      case 'UID':
        currentEvent.uid = value;
        break;
      case 'STATUS':
        currentEvent.status = value;
        break;
      case 'RRULE':
        currentEvent.rrule = parseRRule(value);
        break;
      case 'SEQUENCE':
        currentEvent.sequence = parseInt(value, 10) || 0;
        break;
    }
  }
  
  return events;
}
```

#### DateTime Parsing

```javascript
function parseICSDateTime(value, tzid) {
  // Format: 20260316T090000Z (UTC) or 20260316T090000 (local/with TZID)
  const match = value.match(/^(\d{4})(\d{2})(\d{2})T(\d{2})(\d{2})(\d{2})(Z?)$/);
  if (!match) return null;
  
  const [, year, month, day, hour, minute, second, utcFlag] = match;
  
  return {
    year: parseInt(year),
    month: parseInt(month),
    day: parseInt(day),
    hour: parseInt(hour),
    minute: parseInt(minute),
    second: parseInt(second),
    isUTC: utcFlag === 'Z',
    tzid: tzid || null
  };
}

// Convert ICS datetime to Planora's format (YYYY-MM-DD and HH:MM)
function icsDateTimeToPlanora(icsdt, baseZone) {
  let date;
  if (icsdt.isUTC) {
    date = new Date(Date.UTC(icsdt.year, icsdt.month - 1, icsdt.day, icsdt.hour, icsdt.minute, icsdt.second));
  } else if (icsdt.tzid) {
    // Create date in the specified timezone, then convert to base timezone
    date = resolveZonedDate(icsdt, icsdt.tzid);
  } else {
    // Assume local time
    date = new Date(icsdt.year, icsdt.month - 1, icsdt.day, icsdt.hour, icsdt.minute, icsdt.second);
  }
  
  // Convert to base timezone for Planora display
  const inBase = convertToTimezone(date, baseZone);
  
  return {
    date: formatDateKey(inBase),   // 'YYYY-MM-DD'
    time: formatTimeKey(inBase)    // 'HH:MM'
  };
}
```

#### ICS Text Unescaping

```javascript
function unescapeICS(text) {
  return text
    .replace(/\\n/gi, '\n')
    .replace(/\\,/g, ',')
    .replace(/\\;/g, ';')
    .replace(/\\\\/g, '\\');
}

function extractMailto(value) {
  const match = value.match(/mailto:([^\s;]+)/i);
  return match ? match[1] : value;
}
```

### Email Body Fallback Parser

For emails without .ics attachments (e.g., Google Calendar notification emails), parse the HTML/text body:

```javascript
function parseMeetingFromEmailBody(headers, bodyText) {
  const meeting = {
    summary: null,
    dtStart: null,
    dtEnd: null,
    organizer: null,
    location: null,
    source: 'email-body'
  };
  
  // Extract from subject line
  const subject = getHeader(headers, 'Subject') || '';
  const subjectPatterns = [
    /Invitation:\s*(.+?)\s*@/,           // Google Calendar: "Invitation: Meeting @ ..."
    /Accepted:\s*(.+?)\s*@/,             // "Accepted: Meeting @ ..."
    /Updated invitation:\s*(.+?)\s*@/,   // "Updated invitation: Meeting @ ..."
    /(.+?)\s*-\s*Meeting\s*Invitation/   // Outlook style
  ];
  
  for (const pattern of subjectPatterns) {
    const match = subject.match(pattern);
    if (match) {
      meeting.summary = match[1].trim();
      break;
    }
  }
  
  // Extract date/time from body patterns
  const dateTimePatterns = [
    /When:\s*(.+?)(?:\n|<br|<\/)/i,
    /Date:\s*(.+?)(?:\n|<br|<\/)/i,
    /(\w+ \d{1,2}, \d{4})\s+(\d{1,2}:\d{2}\s*(?:AM|PM)?)\s*[–-]\s*(\d{1,2}:\d{2}\s*(?:AM|PM)?)/i
  ];
  
  // ... pattern matching logic ...
  
  return meeting;
}
```

---

## 9. Data Mapping — Gmail → Planora Events

### Mapping Table

| ICS / Email Field | Planora Event Field | Transform |
|-------------------|---------------------|-----------|
| `SUMMARY` | `title` | Direct copy, trim to 200 chars |
| `DTSTART` | `startDate` + `startTime` | Convert from UTC/TZID to base timezone |
| `DTEND` | `endDate` + `endTime` | Convert from UTC/TZID to base timezone |
| `UID` | `_gmailSyncId` | Used for deduplication (stored in new field) |
| `ORGANIZER` | `notes[0]` (auto-note) | "Organized by: {name} ({email})" |
| `LOCATION` | `notes[1]` (auto-note) | "Location: {location}" |
| `ATTENDEES` | `notes[2]` (auto-note) | "Attendees: {list}" |
| `DESCRIPTION` | `notesText` | Truncated to 500 chars |
| `STATUS` (CANCELLED) | `status` | Map: CONFIRMED→'active', CANCELLED→'cancelled' |
| `RRULE` | `recurrence` | Map: DAILY→'daily', WEEKLY→(everyXDays, 7) |
| `SEQUENCE` | `_gmailSyncSeq` | Track ICS sequence for update detection |
| Email `From` header | `_gmailSyncSource` | Original sender (forwarded from) |
| Gmail message ID | `_gmailMessageId` | Reference back to original email |

### New Event Fields (Gmail Sync Extensions)

```javascript
// These fields are added to Planora events created by Gmail sync
{
  // ... existing Event fields ...
  
  // Gmail Sync metadata
  _gmailSyncId: string,      // ICS UID — unique ID for dedup
  _gmailMessageId: string,   // Gmail message ID for reference
  _gmailSyncSource: string,  // Forwarded-from email address
  _gmailSyncSeq: number,     // ICS SEQUENCE number for version tracking
  _gmailSyncedAt: string,    // ISO timestamp of when this was synced
  _gmailSyncOrigin: string   // 'ics' | 'email-body' — how it was parsed
}
```

### Event Creation Logic

```javascript
function gmailToPlanoraEvent(parsedICS, emailMeta, baseZone) {
  const start = icsDateTimeToPlanora(parsedICS.dtStart, baseZone);
  const end = parsedICS.dtEnd 
    ? icsDateTimeToPlanora(parsedICS.dtEnd, baseZone)
    : { date: start.date, time: addMinutes(start.time, 60) }; // Default 1hr if no end
  
  const notes = [];
  if (parsedICS.organizerName || parsedICS.organizer) {
    notes.push({
      id: cryptoRandomId(),
      text: `📧 Organized by: ${parsedICS.organizerName || ''} (${parsedICS.organizer || 'unknown'})`,
      createdAt: new Date().toISOString()
    });
  }
  if (parsedICS.location) {
    notes.push({
      id: cryptoRandomId(),
      text: `📍 Location: ${parsedICS.location}`,
      createdAt: new Date().toISOString()
    });
  }
  if (parsedICS.attendees && parsedICS.attendees.length > 0) {
    const attendeeList = parsedICS.attendees
      .map(a => `${a.name || a.email} (${a.status.toLowerCase()})`)
      .join(', ');
    notes.push({
      id: cryptoRandomId(),
      text: `👥 Attendees: ${attendeeList}`,
      createdAt: new Date().toISOString()
    });
  }
  
  return {
    id: cryptoRandomId(),
    title: (parsedICS.summary || 'Untitled Meeting').substring(0, 200),
    typeId: 'synced_meeting',        // New event type (see Section 13)
    ownerCompanyId: '',              // User can assign later
    laneCompanyId: '',               // User can assign later
    startDate: start.date,
    startTime: start.time,
    endDate: end.date,
    endTime: end.time,
    status: parsedICS.status === 'CANCELLED' ? 'cancelled' : 'active',
    priority: 50,
    recurrence: mapICSRecurrence(parsedICS.rrule),
    notesText: (parsedICS.description || '').substring(0, 500),
    notes: notes,
    subtasks: [],
    
    // Sync metadata
    _gmailSyncId: parsedICS.uid || '',
    _gmailMessageId: emailMeta.messageId || '',
    _gmailSyncSource: emailMeta.from || '',
    _gmailSyncSeq: parsedICS.sequence || 0,
    _gmailSyncedAt: new Date().toISOString(),
    _gmailSyncOrigin: parsedICS._parseSource || 'ics'
  };
}
```

### Recurrence Mapping

```javascript
function mapICSRecurrence(rrule) {
  if (!rrule) return { mode: 'once', interval: 2 };
  
  switch (rrule.freq) {
    case 'DAILY':
      return { mode: 'daily', interval: rrule.interval || 1 };
    case 'WEEKLY':
      return { mode: 'everyXDays', interval: 7 * (rrule.interval || 1) };
    case 'MONTHLY':
      return { mode: 'everyXDays', interval: 30 * (rrule.interval || 1) };
    default:
      return { mode: 'once', interval: 2 };
  }
}
```

---

## 10. IndexedDB Schema Changes

### New IDB Key: `tz_gmail_sync`

```javascript
// Stored under: __idb.set('tz_gmail_sync', gmailSyncState)
{
  // OAuth Configuration
  clientId: string,              // Google OAuth2 Client ID
  
  // Token (short-lived, refreshed via GIS)
  accessToken: string | null,    // Current access token (or null)
  tokenExpiry: number | null,    // Token expiry timestamp (ms)
  
  // Sync State
  lastSyncTimestamp: number | null,  // Last successful sync (ms since epoch)
  lastSyncMessageCount: number,     // Number of messages processed in last sync
  syncHistory: [{                   // Last 10 sync operations
    timestamp: number,
    messagesFound: number,
    eventsCreated: number,
    eventsUpdated: number,
    errors: number
  }],
  
  // Settings
  syncDaysBack: number,          // How far back to search (default: 7)
  maxMessagesPerSync: number,    // Max messages to process (default: 50)
  autoAssignCompany: boolean,    // Try to auto-assign company from email domain (default: false)
  companyEmailMap: {             // Email domain → company card ID mapping
    // 'company.com': 'card_abc123'
  },
  
  // Processed Message Tracking (dedup)
  processedMessageIds: string[], // Gmail message IDs already processed (max 500, FIFO)
  syncedEventUIDs: string[]      // ICS UIDs already in Planora (for dedup)
}
```

### Default Gmail Sync State

```javascript
function defaultGmailSyncState() {
  return {
    clientId: '',
    accessToken: null,
    tokenExpiry: null,
    lastSyncTimestamp: null,
    lastSyncMessageCount: 0,
    syncHistory: [],
    syncDaysBack: 7,
    maxMessagesPerSync: 50,
    autoAssignCompany: false,
    companyEmailMap: {},
    processedMessageIds: [],
    syncedEventUIDs: []
  };
}
```

### Preload Addition

Add to the IDB preload sequence (Section 1 of main spec):

```javascript
// In async IIFE init:
// After existing preloads...
let gmailSyncState = await __idb.get('tz_gmail_sync') || defaultGmailSyncState();
```

### Schema Version Bump

- `DATA_SCHEMA_VERSION` → `2`
- Migration `1→2`:

```javascript
migrations[1] = function(data) {
  // v1 → v2: Add Gmail sync support
  // No changes to existing state needed — gmailSyncState is a separate IDB key
  // But ensure events with _gmailSync* fields are preserved on import
  if (data.state && data.state.events) {
    data.state.events.forEach(ev => {
      // Preserve sync metadata if present
      if (ev._gmailSyncId === undefined) ev._gmailSyncId = '';
    });
  }
  return data;
};
```

### Export/Import Extensions

Update `exportAllData()`:
```javascript
{
  // ... existing export fields ...
  gmailSyncSettings: {
    syncDaysBack: gmailSyncState.syncDaysBack,
    maxMessagesPerSync: gmailSyncState.maxMessagesPerSync,
    autoAssignCompany: gmailSyncState.autoAssignCompany,
    companyEmailMap: gmailSyncState.companyEmailMap
    // Note: tokens NOT exported for security
  }
}
```

Update `importDataFromFile()`:
```javascript
// Restore Gmail sync settings (but NOT tokens/auth)
if (data.gmailSyncSettings) {
  const syncState = getGmailSyncState();
  syncState.syncDaysBack = data.gmailSyncSettings.syncDaysBack || 7;
  syncState.maxMessagesPerSync = data.gmailSyncSettings.maxMessagesPerSync || 50;
  syncState.autoAssignCompany = data.gmailSyncSettings.autoAssignCompany || false;
  syncState.companyEmailMap = data.gmailSyncSettings.companyEmailMap || {};
  saveGmailSyncState(syncState);
}
```

---

## 11. Sync Engine

### Core Sync Function

```javascript
async function performGmailSync() {
  const syncState = getGmailSyncState();
  
  // 1. Validate prerequisites
  if (!syncState.clientId) {
    openGmailSyncSetupSheet();
    return;
  }
  
  if (!isGmailSyncReady()) {
    __appToast('Gmail sync scripts still loading. Please try again in a moment.', 'info');
    return;
  }
  
  // 2. Show sync indicator
  showSyncProgress('Connecting to Gmail...');
  
  try {
    // 3. Ensure valid token
    await ensureValidToken();
    
    // 4. Fetch meeting emails
    showSyncProgress('Searching for meeting emails...');
    const messages = await fetchMeetingEmails(syncState.syncDaysBack);
    
    if (messages.length === 0) {
      hideSyncProgress();
      __appToast('No new meeting emails found.', 'info');
      logSyncHistory(syncState, 0, 0, 0, 0);
      return;
    }
    
    // 5. Filter out already-processed messages
    const newMessages = messages.filter(m => 
      !syncState.processedMessageIds.includes(m.id)
    );
    
    showSyncProgress(`Processing ${newMessages.length} emails...`);
    
    // 6. Parse each message
    let created = 0, updated = 0, errors = 0;
    
    for (const msg of newMessages) {
      try {
        const result = await processMessageForSync(msg, syncState);
        if (result === 'created') created++;
        else if (result === 'updated') updated++;
        
        // Track as processed
        syncState.processedMessageIds.push(msg.id);
      } catch (err) {
        console.error('Sync error for message', msg.id, err);
        errors++;
      }
    }
    
    // 7. Prune old processed IDs (keep last 500)
    if (syncState.processedMessageIds.length > 500) {
      syncState.processedMessageIds = syncState.processedMessageIds.slice(-500);
    }
    
    // 8. Update sync state
    logSyncHistory(syncState, newMessages.length, created, updated, errors);
    syncState.lastSyncTimestamp = Date.now();
    syncState.lastSyncMessageCount = newMessages.length;
    saveGmailSyncState(syncState);
    
    // 9. Save main state (events were modified)
    saveState();
    
    // 10. Refresh UI
    render();
    hideSyncProgress();
    
    // 11. Show result toast
    const parts = [];
    if (created > 0) parts.push(`${created} new meeting${created > 1 ? 's' : ''}`);
    if (updated > 0) parts.push(`${updated} updated`);
    if (errors > 0) parts.push(`${errors} error${errors > 1 ? 's' : ''}`);
    
    __appToast(
      parts.length > 0 
        ? `Sync complete: ${parts.join(', ')}` 
        : 'Sync complete — no changes.',
      errors > 0 ? 'error' : 'success'
    );
    
  } catch (err) {
    hideSyncProgress();
    
    if (err instanceof GmailAuthError) {
      __appToast('Gmail authentication expired. Please try syncing again.', 'error');
      gmailAccessToken = null;
    } else {
      __appToast('Sync failed: ' + (err.message || 'Unknown error'), 'error');
    }
    console.error('Gmail sync failed:', err);
  }
}
```

### Process Single Message

```javascript
async function processMessageForSync(message, syncState) {
  const headers = message.payload.headers;
  const from = getHeader(headers, 'From') || '';
  const subject = getHeader(headers, 'Subject') || '';
  
  // Try to extract .ics attachment first
  let parsedEvents = [];
  const icsContent = await extractIcsAttachment(message.id, message.payload.parts || []);
  
  if (icsContent) {
    parsedEvents = parseICS(icsContent);
    parsedEvents.forEach(e => e._parseSource = 'ics');
  }
  
  // Fallback: parse email body
  if (parsedEvents.length === 0) {
    const bodyText = extractEmailBody(message.payload);
    const bodyEvent = parseMeetingFromEmailBody(headers, bodyText);
    if (bodyEvent && bodyEvent.summary) {
      bodyEvent._parseSource = 'email-body';
      parsedEvents.push(bodyEvent);
    }
  }
  
  if (parsedEvents.length === 0) return 'skipped';
  
  // Process each parsed event
  let result = 'skipped';
  
  for (const parsedEvent of parsedEvents) {
    // Skip CANCELLED events with no matching existing event
    if (parsedEvent.status === 'CANCELLED' && !findExistingSyncedEvent(parsedEvent.uid)) {
      continue;
    }
    
    const emailMeta = {
      messageId: message.id,
      from: extractForwardedFrom(headers, from),
      subject: subject
    };
    
    // Check for existing event (update vs create)
    const existingEvent = findExistingSyncedEvent(parsedEvent.uid);
    
    if (existingEvent) {
      // Update if sequence number is higher
      if ((parsedEvent.sequence || 0) > (existingEvent._gmailSyncSeq || 0)) {
        updateExistingEventFromSync(existingEvent, parsedEvent, emailMeta);
        result = 'updated';
      }
    } else {
      // Create new event
      const newEvent = gmailToPlanoraEvent(parsedEvent, emailMeta, state.baseZone);
      
      // Auto-assign company if enabled
      if (syncState.autoAssignCompany && emailMeta.from) {
        const domain = emailMeta.from.split('@')[1];
        if (domain && syncState.companyEmailMap[domain]) {
          newEvent.ownerCompanyId = syncState.companyEmailMap[domain];
          newEvent.laneCompanyId = syncState.companyEmailMap[domain];
        }
      }
      
      state.events.push(newEvent);
      
      // Track UID for dedup
      if (parsedEvent.uid) {
        syncState.syncedEventUIDs.push(parsedEvent.uid);
      }
      
      result = 'created';
    }
  }
  
  return result;
}
```

### Deduplication Logic

```javascript
function findExistingSyncedEvent(uid) {
  if (!uid) return null;
  return state.events.find(e => e._gmailSyncId === uid);
}

function updateExistingEventFromSync(existing, parsed, emailMeta) {
  const start = icsDateTimeToPlanora(parsed.dtStart, state.baseZone);
  const end = parsed.dtEnd 
    ? icsDateTimeToPlanora(parsed.dtEnd, state.baseZone) 
    : { date: start.date, time: addMinutes(start.time, 60) };
  
  existing.title = (parsed.summary || existing.title).substring(0, 200);
  existing.startDate = start.date;
  existing.startTime = start.time;
  existing.endDate = end.date;
  existing.endTime = end.time;
  existing.status = parsed.status === 'CANCELLED' ? 'cancelled' : existing.status;
  existing._gmailSyncSeq = parsed.sequence || 0;
  existing._gmailSyncedAt = new Date().toISOString();
  
  // Add update note
  existing.notes.push({
    id: cryptoRandomId(),
    text: `🔄 Updated via sync on ${new Date().toLocaleDateString()}`,
    createdAt: new Date().toISOString()
  });
}
```

---

## 12. UI Design — Sync Button & Panel

### Sync Button on Calendar Page

A new button is added to the **calendar navigation toolbar** (alongside the week navigation controls):

```
┌──────────────────────────────────────────────┐
│  ◀  Today  ▶     Mar 9 – Mar 15       ☁️ 🔄 │
│                                    [Sync btn] │
└──────────────────────────────────────────────┘
```

#### Button Specs

```html
<button id="gmailSyncBtn" class="icon-btn gmail-sync-btn" aria-label="Sync meetings from Gmail" title="Sync from Gmail">
  <svg class="icon-svg"><use href="#icon-mail"></use></svg>
</button>
```

#### Button States

| State | Appearance | Behavior |
|-------|-----------|----------|
| **Idle (not configured)** | Muted mail icon | Opens setup sheet |
| **Idle (configured)** | Accent-colored mail icon with small sync badge | Starts sync |
| **Syncing** | Spinning/pulsing animation | Disabled, shows progress |
| **Sync complete** | Brief green flash | Returns to idle after 2s |
| **Error** | Brief red flash | Shows error toast |

#### Sync Progress Indicator

```html
<div id="gmailSyncProgress" class="gmail-sync-progress hidden">
  <div class="sync-spinner"></div>
  <span class="sync-status-text">Syncing meetings...</span>
</div>
```

Positioned as a floating pill below the calendar toolbar, above the grid.

### CSS Styling

```css
.gmail-sync-btn {
  position: relative;
}

.gmail-sync-btn::after {
  content: '';
  position: absolute;
  bottom: 4px;
  right: 4px;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--accent);
  display: none;
}

.gmail-sync-btn.connected::after {
  display: block;
}

.gmail-sync-btn.syncing .icon-svg {
  animation: syncPulse 1.2s ease-in-out infinite;
}

@keyframes syncPulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(0.9); }
}

.gmail-sync-progress {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  font-size: 13px;
  color: var(--text-muted);
  backdrop-filter: blur(12px);
  animation: fadeIn 0.2s ease;
}

.sync-spinner {
  width: 16px;
  height: 16px;
  border: 2px solid var(--border-strong);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
```

### Sync Button on Events Page FAB

An additional entry in the Events page FAB action sheet:

```javascript
// In FAB action sheet for Events page:
{
  icon: 'mail',
  label: 'Sync from Gmail',
  action: () => performGmailSync()
}
```

### Gmail Sync Setup Sheet (First-Time)

A bottom sheet guiding the user through initial setup:

```
┌─────────────────────────────────────┐
│  ☁️  Connect Gmail                  │
│                                      │
│  Sync meeting invites from your      │
│  Gmail inbox to Planora.             │
│                                      │
│  ┌────────────────────────────────┐  │
│  │ Google Cloud Client ID         │  │
│  │ [___________________________ ] │  │
│  │ (Paste your OAuth2 Client ID)  │  │
│  └────────────────────────────────┘  │
│                                      │
│  📖 Setup Guide                     │
│  1. Go to console.cloud.google.com   │
│  2. Create a project & enable        │
│     Gmail API                        │
│  3. Create OAuth2 Web Client ID      │
│  4. Add origin:                      │
│     https://ercrvr.github.io         │
│  5. Copy Client ID & paste above     │
│                                      │
│  [ Save & Connect ]                  │
└─────────────────────────────────────┘
```

### Gmail Sync Status Panel

Accessible from Settings → Gmail Sync:

```
┌─────────────────────────────────────┐
│  ☁️  Gmail Sync Settings            │
│                                      │
│  Status: ✅ Connected                │
│  Last sync: 2 hours ago              │
│  Last result: 3 new, 1 updated      │
│                                      │
│  ── Sync Settings ──                 │
│  Days to look back: [7 ▼]           │
│  Max emails per sync: [50 ▼]        │
│                                      │
│  ── Company Mapping ──               │
│  Map email domains to companies:     │
│  ┌──────────────────────────────┐   │
│  │ company.com → Acme Corp      │   │
│  │ client.io   → Client Inc     │   │
│  │ [+ Add mapping]              │   │
│  └──────────────────────────────┘   │
│                                      │
│  ── Sync History ──                  │
│  Mar 15, 5:30 PM - 3 new, 1 upd    │
│  Mar 14, 9:00 AM - 0 new           │
│  Mar 13, 8:45 AM - 2 new           │
│                                      │
│  [ Disconnect Gmail ]                │
└─────────────────────────────────────┘
```

---

## 13. New Event Type — Synced Meeting

### Event Type Definition

Added to the default event types array:

```javascript
{
  id: 'synced_meeting',
  name: 'Synced Meeting',
  markerKind: 'icon',
  colorHex: '#f472b6',         // Pink — distinct from existing types
  shape: 'square',
  icon: 'mail',                // Uses existing mail icon
  notificationsEnabled: true,
  conflictAlertsEnabled: true,
  alertBeforeValue: 15,
  alertBeforeUnit: 'minutes',
  maxAlerts: 3
}
```

### Visual Distinction

Synced meetings appear on the calendar with:
- **Pink mail icon** marker (distinct from manual meetings' blue briefcase)
- **"↗" badge** in the event card showing it came from sync
- **Source label** in event details: "Synced from: john@company.com"

### User Customization

Users can:
- Change the synced meeting color, icon, and shape (like any event type)
- Reassign synced events to a different type (e.g., convert to a regular "Meeting")
- Manually edit any field of a synced event
- Assign synced events to company cards/lanes after sync

---

## 14. Settings Integration

### Settings Context Menu Addition

Add a fourth item to the settings gear menu:

```javascript
// Updated settings menu items:
1. Event Settings
2. Calendar Settings
3. Alert Settings
4. Gmail Sync        // ← NEW
```

### Gmail Sync Settings Entry Points

| Entry Point | Action |
|-------------|--------|
| Settings → Gmail Sync | Opens full Gmail Sync settings sheet |
| Calendar page Sync button (first time) | Opens setup sheet |
| Calendar page Sync button (configured) | Triggers sync |
| Events page FAB → Sync from Gmail | Triggers sync |

---

## 15. Conflict & Duplicate Handling

### Deduplication Strategy

| Level | Method | Purpose |
|-------|--------|---------|
| **Gmail Message ID** | `processedMessageIds[]` | Don't re-process the same email |
| **ICS UID** | `_gmailSyncId` field match | Don't create duplicate Planora events for same meeting |
| **ICS SEQUENCE** | `_gmailSyncSeq` comparison | Detect meeting updates (time change, cancellation) |
| **Title + Time** | Fuzzy match | Catch duplicates from email body parsing (no UID available) |

### Fuzzy Duplicate Detection (for email-body parsed events)

```javascript
function findFuzzyDuplicateEvent(title, startDate, startTime) {
  return state.events.find(e => {
    if (e.startDate !== startDate) return false;
    if (Math.abs(timeToMinutes(e.startTime) - timeToMinutes(startTime)) > 15) return false;
    
    // Normalize titles for comparison
    const normA = title.toLowerCase().replace(/[^a-z0-9]/g, '');
    const normB = e.title.toLowerCase().replace(/[^a-z0-9]/g, '');
    
    return normA === normB || normA.includes(normB) || normB.includes(normA);
  });
}
```

### Conflict Detection Integration

Synced meetings integrate with the existing conflict detection system:
- `conflictAlertsEnabled: true` on the `synced_meeting` event type
- Cross-lane conflicts detected when synced events overlap with company shift schedules
- Conflict indicators appear on the calendar grid as with any other event

---

## 16. Error Handling

### Error Types & Recovery

| Error | Cause | User-Facing Message | Recovery |
|-------|-------|---------------------|----------|
| `GmailAuthError` | Token expired/revoked | "Gmail authentication expired. Please try syncing again." | Re-triggers OAuth popup on next sync |
| `GmailApiError (403)` | API not enabled or quota exceeded | "Gmail API access denied. Please check your Google Cloud project setup." | Link to setup guide |
| `GmailApiError (429)` | Rate limited | "Too many requests. Please wait a minute and try again." | Auto-retry after 60s |
| `NetworkError` | No internet | "No internet connection. Synced events are still available offline." | Toast + retain existing data |
| `ParseError` | Malformed .ics or email | Silently skip + log error count | Show "X errors" in sync result |
| `QuotaError` | IDB storage full | "Storage full. Please clear some data." | Point to export/clear options |

### Error Logging

```javascript
class GmailAuthError extends Error {
  constructor(msg) { super(msg); this.name = 'GmailAuthError'; }
}

class GmailApiError extends Error {
  constructor(msg, status) { 
    super(msg); 
    this.name = 'GmailApiError'; 
    this.status = status; 
  }
}
```

### Debug Console Integration

All sync operations are logged to Planora's debug console:

```javascript
console.log(`[Gmail Sync] Starting sync, looking back ${days} days`);
console.log(`[Gmail Sync] Found ${count} messages`);
console.log(`[Gmail Sync] Parsed event: "${title}" on ${date}`);
console.warn(`[Gmail Sync] Could not parse .ics from message ${id}`);
console.error(`[Gmail Sync] API error:`, err);
```

---

## 17. Security & Privacy

### Token Security

- Access tokens are **short-lived** (~1 hour) and stored only in IndexedDB
- Tokens are **never exported** in data backups
- Tokens are **never included** in QR sync payloads
- On "Disconnect Gmail", all tokens and sync data are wiped

### Scope Minimization

- Only `gmail.readonly` scope — Planora **cannot** send, modify, or delete emails
- The app only reads message metadata and .ics attachments
- Email body content is parsed locally and never stored (only extracted meeting fields are kept)

### Client ID Handling

- The OAuth2 Client ID is a **public identifier** (not a secret) — safe to store in IndexedDB
- It's scoped to authorized origins (GitHub Pages URL) so it can't be misused from other domains

### Data Cleanup on Disconnect

```javascript
function disconnectGmail() {
  const syncState = getGmailSyncState();
  
  // Revoke token if active
  if (syncState.accessToken) {
    google.accounts.oauth2.revoke(syncState.accessToken);
  }
  
  // Clear all sync data
  syncState.accessToken = null;
  syncState.tokenExpiry = null;
  syncState.processedMessageIds = [];
  syncState.syncedEventUIDs = [];
  syncState.syncHistory = [];
  syncState.lastSyncTimestamp = null;
  // Keep clientId and settings — user might reconnect
  
  saveGmailSyncState(syncState);
  gmailAccessToken = null;
  
  __appToast('Gmail disconnected. Your synced events remain in Planora.', 'success');
}
```

### Note on Forwarded Emails

Users should be aware that auto-forwarding copies email content to their Gmail. This is a configuration they control outside Planora. Planora only reads what's already in their Gmail inbox.

---

## 18. Migration & Backward Compatibility

### Schema Version

- Bump `DATA_SCHEMA_VERSION` from `1` to `2`
- Migration `1→2` is a no-op for main state (gmail sync uses separate IDB key)
- Existing events are unaffected — they simply won't have `_gmailSync*` fields

### Import Compatibility

- Older exports (v1) import cleanly — no gmail sync fields expected
- Newer exports (v2+) include `gmailSyncSettings` block — older versions ignore unknown keys
- Synced events import/export like regular events (the `_gmail*` metadata fields are preserved)

### Feature Detection

```javascript
function isGmailSyncSupported() {
  return typeof gapi !== 'undefined' 
    && typeof google !== 'undefined' 
    && window.isSecureContext;  // Requires HTTPS (GitHub Pages)
}
```

If running on `file://` or `http://` (non-localhost), the Gmail Sync button shows a tooltip: "Gmail sync requires HTTPS. Deploy to GitHub Pages or use localhost."

---

## 19. Testing Strategy

### Manual Testing Checklist

#### OAuth Flow
- [ ] First-time setup: enter Client ID → OAuth popup → successful auth
- [ ] Token refresh: wait >1hr → sync → should re-authenticate seamlessly
- [ ] Invalid Client ID: should show clear error
- [ ] User denies permission: should handle gracefully
- [ ] Disconnect and reconnect

#### Sync Operations
- [ ] Sync with .ics attachments (Google Calendar invites)
- [ ] Sync with Outlook/Exchange invites (different .ics format)
- [ ] Sync email-body-only meetings (no .ics)
- [ ] Sync with 0 results (no meeting emails)
- [ ] Sync with 50+ results (pagination)
- [ ] Deduplication: run sync twice, verify no duplicates
- [ ] Update detection: modify meeting in source → re-sync → verify update
- [ ] Cancellation: cancel meeting in source → re-sync → verify status change

#### UI/UX
- [ ] Sync button states: idle, syncing, complete, error
- [ ] Progress indicator during sync
- [ ] Toast messages for all outcomes
- [ ] Synced events visible on calendar grid
- [ ] Synced events visible on events page
- [ ] Synced event details show source info
- [ ] Company mapping works (auto-assign lane)

#### Edge Cases
- [ ] Sync offline → error toast
- [ ] Sync with expired token → re-auth flow
- [ ] Very long meeting title (>200 chars) → truncated
- [ ] All-day events (no specific time in .ics)
- [ ] Multi-day events
- [ ] Recurring meetings (RRULE in .ics)
- [ ] Meeting in different timezone → correct conversion
- [ ] Forwarded email with original sender preserved

#### Data Integrity
- [ ] Export with synced events → import on fresh instance → events preserved
- [ ] Synced events persist across page reloads
- [ ] Disconnect Gmail → synced events remain
- [ ] Clear all data → sync state also cleared

### Test .ics Files

Create sample .ics files for testing different providers:
- Google Calendar invite
- Outlook/Exchange invite
- Apple Calendar invite
- Simple VEVENT (minimal fields)
- Complex VEVENT (all fields, RRULE, multiple attendees)
- VCALENDAR with multiple VEVENTs
- Cancelled event (STATUS:CANCELLED)

---

## 20. Future Enhancements

### v2 — Google Calendar API Direct Integration

Instead of parsing Gmail, connect directly to Google Calendar API:
- Scope: `calendar.readonly`
- Direct access to structured event data
- No need for email forwarding
- Real-time sync via calendar webhooks

### v2 — Microsoft Outlook/Graph API

Add Outlook calendar sync:
- Scope: `Calendars.Read`
- Microsoft Identity Platform OAuth2
- Direct calendar access for Office 365 users

### v2 — Two-Way Sync

- Events created in Planora pushed to Google Calendar
- Requires `calendar.events` scope (write)
- Conflict resolution for bi-directional changes

### v2 — Background Sync

- Service Worker-based background sync
- Push notifications for new meetings
- Periodic sync (every 15–30 minutes)

### v2 — Multi-Account Gmail

- Support multiple Gmail accounts simultaneously
- Each with its own OAuth token
- Unified sync across all accounts

### v2 — Smart Company Assignment

- ML-based company detection from email patterns
- Learn from user corrections
- Automatic domain → company mapping suggestions

---

## Appendix: File Changes Summary

### Modified Sections in `index.html`

| Section | Change |
|---------|--------|
| `<head>` | Add 2 CDN script tags (GIS + gapi) |
| SVG symbols | Add `#icon-sync` if not using existing icon |
| CSS | Add ~80 lines for sync button, progress, setup sheet |
| Constants | Add `DATA_SCHEMA_VERSION = 2` |
| IDB preload | Add `tz_gmail_sync` key loading |
| Default event types | Add `synced_meeting` type |
| Settings menu | Add "Gmail Sync" option |
| Calendar page | Add sync button to toolbar |
| Events page FAB | Add "Sync from Gmail" action |
| Export/Import | Handle `gmailSyncSettings` |
| Migration chain | Add `1→2` migration |
| New code block | ~500–700 lines: OAuth, Gmail API, ICS parser, sync engine, UI |

### Estimated Impact

- **New code**: ~600–800 lines (JavaScript + CSS + HTML)
- **Modified code**: ~50 lines (existing functions updated)
- **Total file size increase**: ~25–35 KB (from ~658 KB to ~690 KB)
- **New CDN dependencies**: 2 scripts (~100 KB combined, loaded async)
- **New IDB key**: 1 (`tz_gmail_sync`)

---

*End of Gmail Calendar Sync Feature Spec*
