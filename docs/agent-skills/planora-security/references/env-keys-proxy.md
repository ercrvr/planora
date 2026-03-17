# Environment Variables, API Keys & Proxy Management

How to handle secrets, configuration, and external API access securely in Planora.

## Table of Contents

1. [The Core Problem](#the-core-problem)
2. [Environment Variable Management](#environment-variable-management)
3. [API Key Security](#api-key-security)
4. [Proxy Patterns](#proxy-patterns)
5. [Build-Time Injection](#build-time-injection)
6. [Git History & Secret Rotation](#git-history--secret-rotation)

---

## The Core Problem

Planora is currently a single HTML file with no build step and no backend server. This creates a tension:

- **Client-side code is public.** Anything in the HTML file — API keys, tokens, passwords — is visible to anyone who opens DevTools or views source.
- **Some integrations require secrets.** API keys, OAuth tokens, and service credentials can't be safely embedded in client-side code.
- **The Frankfurter API doesn't require a key**, which is why it works today. But any future API integration that requires authentication will need a different approach.

### Current API Usage

```js
// Line ~8727 — Public API, no key needed
var url = 'https://api.frankfurter.app/latest?from=' + wc.defaultCurrency + '&to=' + codes;
var response = await fetch(url);
```

This is safe because Frankfurter is a free, keyless API. The moment Planora needs a keyed API (weather, maps, AI, payment processing), a different architecture is needed.

---

## Environment Variable Management

### For Development

Create a `.env` file in the project root (never committed to git):

```bash
# .env — DO NOT COMMIT
PLANORA_CURRENCY_API_KEY=sk_live_abc123
PLANORA_WEATHER_API_KEY=wx_def456
PLANORA_DEBUG_ENABLED=false
```

Add to `.gitignore`:
```
.env
.env.local
.env.*.local
```

### For Documentation

Create a `.env.example` file (committed to git) showing required variables without values:

```bash
# .env.example — Safe to commit
# Copy this to .env and fill in your values

# Currency API key (get from https://example.com/api)
PLANORA_CURRENCY_API_KEY=

# Weather API key (optional)
PLANORA_WEATHER_API_KEY=

# Enable debug panel (true/false)
PLANORA_DEBUG_ENABLED=false
```

### Naming Convention

Prefix all Planora env vars with `PLANORA_` to avoid conflicts:

```
PLANORA_{SERVICE}_{PURPOSE}

Examples:
PLANORA_CURRENCY_API_KEY
PLANORA_CURRENCY_API_URL
PLANORA_MAPS_API_KEY
PLANORA_PROXY_URL
PLANORA_DEBUG_ENABLED
```

---

## API Key Security

### Classification

| Type | Example | Client-Safe? | Approach |
|------|---------|-------------|----------|
| **Public keys** | Google Maps JS API key (with HTTP referrer restriction) | Yes, with restrictions | Embed in client, restrict by referrer/domain |
| **Secret keys** | Payment API secret, OAuth client secret | **Never** | Proxy through backend |
| **Rate-limited public keys** | Free-tier API keys with quotas | Depends | Proxy preferred, or embed with aggressive client-side caching |

### Decision Tree

```
Does the API provider offer a client-safe key type?
  → Yes: Use it with domain/referrer restrictions
  → No: Must use a proxy

Is the key rate-limited with cost implications?
  → Yes: Proxy through backend to control usage
  → No: If client-safe, embed with caching

Does the key provide access to user data or write operations?
  → Yes: Always proxy, always
  → No: Client embedding may be acceptable
```

### Restricting Client-Safe Keys

For keys that support domain restrictions (like Google Maps):

1. Create a separate key for each environment (dev, staging, prod)
2. Restrict each key to its domain (`planora.app`, `localhost:3000`, etc.)
3. Set usage quotas and billing alerts
4. Rotate keys periodically
5. Monitor usage dashboards for anomalies

---

## Proxy Patterns

A proxy server sits between Planora and external APIs, keeping secrets server-side.

### Pattern 1: Simple API Proxy (Cloudflare Worker)

A lightweight serverless function that forwards requests with the secret key:

```js
// Cloudflare Worker — proxy for currency API
export default {
  async fetch(request) {
    const url = new URL(request.url);
    
    // Only allow specific paths
    if (!url.pathname.startsWith('/api/currency')) {
      return new Response('Not found', { status: 404 });
    }
    
    // Rate limiting (use Cloudflare's built-in or implement your own)
    // ...
    
    // Forward to real API with secret key
    const apiUrl = `https://api.example.com/v1/rates${url.search}`;
    const response = await fetch(apiUrl, {
      headers: {
        'Authorization': `Bearer ${API_SECRET_KEY}`, // Set in Worker secrets
        'Content-Type': 'application/json',
      },
    });
    
    // Add CORS headers for Planora's domain
    const headers = new Headers(response.headers);
    headers.set('Access-Control-Allow-Origin', 'https://planora.app');
    headers.set('Access-Control-Allow-Methods', 'GET');
    
    return new Response(response.body, { status: response.status, headers });
  }
};
```

### Pattern 2: Netlify/Vercel Serverless Function

```js
// netlify/functions/currency.js
exports.handler = async (event) => {
  // Only allow GET
  if (event.httpMethod !== 'GET') {
    return { statusCode: 405, body: 'Method not allowed' };
  }
  
  const { from, to } = event.queryStringParameters || {};
  
  // Validate parameters
  if (!from || !to || !/^[A-Z]{3}$/.test(from)) {
    return { statusCode: 400, body: 'Invalid parameters' };
  }
  
  const response = await fetch(
    `https://api.example.com/rates?from=${from}&to=${to}`,
    { headers: { 'X-API-Key': process.env.CURRENCY_API_KEY } }
  );
  
  const data = await response.json();
  
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Cache-Control': 'public, max-age=3600', // Cache for 1 hour
    },
    body: JSON.stringify(data),
  };
};
```

### Pattern 3: GitHub Pages + External Proxy

Since Planora is hosted on GitHub Pages (static hosting), serverless functions can't run on the same domain. Options:

1. **Cloudflare Worker on a subdomain** — `api.planora.app` proxies to external APIs
2. **Netlify Functions** — If hosting is moved to Netlify
3. **Separate API service** — A tiny Express/Fastify server on a platform like Railway or Render

### Client-Side Configuration

```js
// In Planora, reference the proxy URL instead of the direct API
var CURRENCY_API = window.__PLANORA_CONFIG?.currencyApiUrl 
  || 'https://api.planora.app/currency';

async function fetchRates(from, to) {
  var url = CURRENCY_API + '?from=' + encodeURIComponent(from) + '&to=' + encodeURIComponent(to);
  var response = await fetch(url);
  if (!response.ok) throw new Error('API returned ' + response.status);
  return response.json();
}
```

### Proxy Security Checklist

1. **Validate all input parameters** — Don't blindly forward query strings
2. **Restrict allowed origins** (CORS) — Only allow requests from Planora's domain
3. **Rate limit** — Prevent abuse of your proxy (and your API quota)
4. **Cache responses** — Reduce API calls and improve performance
5. **Log requests** — For debugging and abuse detection
6. **Don't expose error details** — Return generic errors to the client
7. **Use HTTPS exclusively** — Never proxy over HTTP

---

## Build-Time Injection

If/when Planora adopts a build step, environment variables can be injected at build time:

### Simple Shell Script

```bash
#!/bin/bash
# build.sh — inject env vars into the HTML

# Read from .env
source .env

# Replace placeholders in the template
sed -e "s|__CURRENCY_API_URL__|${PLANORA_CURRENCY_API_URL}|g" \
    -e "s|__DEBUG_ENABLED__|${PLANORA_DEBUG_ENABLED}|g" \
    src/index.html > dist/index.html
```

### HTML Template Pattern

```html
<!-- In source template -->
<script>
  window.__PLANORA_CONFIG = {
    currencyApiUrl: '__CURRENCY_API_URL__',
    debugEnabled: __DEBUG_ENABLED__,
  };
</script>
```

### CI/CD Integration (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build with secrets
        env:
          PLANORA_CURRENCY_API_URL: ${{ secrets.CURRENCY_API_URL }}
          PLANORA_DEBUG_ENABLED: "false"
        run: bash build.sh
      - name: Deploy
        # ... deploy dist/index.html
```

---

## Git History & Secret Rotation

### If a Secret Was Committed

If an API key, password, or token was ever committed to git (even if later deleted):

1. **Assume it's compromised** — bots scrape GitHub for secrets within minutes
2. **Rotate the secret immediately** — Generate a new key from the API provider
3. **Revoke the old key** — Don't just generate a new one; kill the old one
4. **Clean git history** (optional but recommended):
   ```bash
   # Using git-filter-repo (recommended over filter-branch)
   pip install git-filter-repo
   git filter-repo --replace-text expressions.txt
   ```
5. **Force push** — This rewrites history, so coordinate with your team
6. **Add to .gitignore** — Prevent the same mistake from recurring

### The `__debugSecret` in Planora

The default `p@ssw0rd` on line 4955 has been in the repository since it was committed. While it's not an API key, it's a security-relevant credential that's now public. The fix is to remove the debug password mechanism from production (see `references/data-protection.md`).

### Secret Scanning

Enable GitHub's secret scanning on the repository:
1. Go to Settings → Code security and analysis
2. Enable "Secret scanning"
3. Enable "Push protection" to block commits containing secrets

This catches common patterns like API keys, tokens, and private keys before they enter the repo.
