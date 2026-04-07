## All JWT Storage Options Compared

### Option 1: HttpOnly Cookie (Recommended)

```typescript
// Server: Set JWT as HttpOnly cookie
res.cookie('access_token', jwt, {
  httpOnly: true,    // JS cannot read it
  secure: true,      // HTTPS only
  sameSite: 'lax',   // CSRF protection
  path: '/',
  maxAge: 15 * 60 * 1000, // 15 min
});

// Client: Token is sent automatically on every request
// No JS code needed — the browser handles it
fetch('/api/data', { credentials: 'include' });
```

**Pros**: Immune to XSS, automatic attachment.
**Cons**: Vulnerable to CSRF (mitigated by SameSite + CSRF token), doesn't work cross-domain easily.

### Option 2: BFF Pattern (Backend-for-Frontend)

The BFF holds the token server-side. The browser never sees a JWT at all:

```
Browser ←→ BFF (same domain, session cookie) ←→ API (JWT)

1. User logs in → BFF gets JWT from auth server
2. BFF stores JWT in server-side session (Redis)
3. BFF gives browser a session ID cookie (HttpOnly)
4. Browser makes requests to BFF → BFF attaches JWT → forwards to API
```

```typescript
// BFF middleware
app.use(async (req, res, next) => {
  const sessionId = req.cookies.session_id;
  const jwt = await redis.get(`session:${sessionId}`);

  if (!jwt) return res.status(401).json({ error: 'Unauthorized' });

  // Forward to upstream API with JWT
  req.headers.authorization = `Bearer ${jwt}`;
  next();
});
```

**Pros**: JWT never touches the browser (immune to XSS AND JavaScript-based attacks). Works with any API.
**Cons**: Extra infrastructure (BFF server + session store). More complexity.

### Option 3: In-Memory Variable (SPA-Only)

```typescript
// Store token in a JavaScript variable (not localStorage)
let accessToken: string | null = null;

async function login(credentials: LoginData) {
  const res = await fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify(credentials),
  });
  const { token } = await res.json();
  accessToken = token; // memory only, not persisted
}

// Attach to requests manually
async function apiCall(url: string) {
  return fetch(url, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });
}
```

**Pros**: Survives XSS better than localStorage (attacker needs to execute JS in the right scope, not just `localStorage.getItem`).
**Cons**: Lost on page refresh (need silent re-auth via refresh token in HttpOnly cookie).

### Option 4: Service Worker Token Broker

Advanced pattern — the service worker acts as a secure vault:

```typescript
// service-worker.ts
let token: string | null = null;

self.addEventListener('message', (event) => {
  if (event.data.type === 'SET_TOKEN') {
    token = event.data.token;
  }
});

self.addEventListener('fetch', (event: FetchEvent) => {
  const url = new URL(event.request.url);

  // Only attach token to your API domain
  if (url.origin === 'https://api.example.com') {
    const modifiedRequest = new Request(event.request, {
      headers: {
        ...Object.fromEntries(event.request.headers),
        Authorization: `Bearer ${token}`,
      },
    });
    event.respondWith(fetch(modifiedRequest));
  }
});
```

**Pros**: Token isolated from page context (harder for XSS to access). Centralized token management.
**Cons**: Complex setup. Service worker scope limitations.

## Refresh Token Rotation

Regardless of storage method, use short-lived access tokens + refresh token rotation:

```typescript
// Server: Issue token pair
function issueTokens(userId: string) {
  const accessToken = jwt.sign({ sub: userId }, SECRET, { expiresIn: '15m' });
  const refreshToken = crypto.randomUUID();

  // Store refresh token hash in DB (not the token itself)
  db.refreshTokens.create({
    tokenHash: hash(refreshToken),
    userId,
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
    familyId: crypto.randomUUID(), // token family for rotation detection
  });

  return { accessToken, refreshToken };
}

// Server: Refresh endpoint with rotation
app.post('/api/refresh', async (req, res) => {
  const { refreshToken } = req.cookies;
  const tokenRecord = await db.refreshTokens.findByHash(hash(refreshToken));

  if (!tokenRecord || tokenRecord.expiresAt < new Date()) {
    return res.status(401).json({ error: 'Invalid refresh token' });
  }

  // Detect token reuse (stolen token being used)
  if (tokenRecord.used) {
    // Revoke ALL tokens in this family — likely stolen
    await db.refreshTokens.revokeFamily(tokenRecord.familyId);
    return res.status(401).json({ error: 'Token reuse detected' });
  }

  // Mark old token as used
  await db.refreshTokens.markUsed(tokenRecord.id);

  // Issue new token pair
  const tokens = issueTokens(tokenRecord.userId);
  res.cookie('refreshToken', tokens.refreshToken, {
    httpOnly: true, secure: true, sameSite: 'strict', path: '/api/refresh',
  });
  res.json({ accessToken: tokens.accessToken });
});
```

## Storage Comparison Matrix

| Method | XSS Safe | CSRF Safe | Cross-Domain | Survives Refresh | Complexity |
|--------|----------|-----------|-------------|-----------------|------------|
| **HttpOnly Cookie** | Yes | Needs SameSite + token | Hard | Yes | Low |
| **BFF Pattern** | Yes | Yes (session) | Yes | Yes | High |
| **In-Memory** | Partial | Yes | Yes | No (needs refresh) | Medium |
| **Service Worker** | Mostly | Yes | Yes | Depends | High |
| **localStorage** | **NO** | Yes | Yes | Yes | Low |
| **sessionStorage** | **NO** | Yes | Yes | Tab only | Low |
