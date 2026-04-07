# CORS: Origins, Dangers, and Proper Configuration

## What is CORS?

Cross-Origin Resource Sharing (CORS) allows browsers to make requests from one origin to another.

**Origin** = scheme + domain + port
- `https://example.com` (port 443 implicit)
- `https://api.example.com` (different subdomain = different origin)
- `http://example.com` (different scheme = different origin)

## Why `Access-Control-Allow-Origin: *` is Dangerous

### Attack Vector 1: Malicious Website Access

```
Attacker's site: attacker.com
Victim's site: bank.com

Victim visits attacker.com
Attacker's JavaScript runs in victim's browser
Browser makes fetch to bank.com (with victim's cookies)
With CORS: *, the request succeeds
Attacker reads bank data (CSRF + XSS combination)
```

### Attack Vector 2: Credential Leakage

```
- Requests with credentials (cookies) inherit user's authority
- With wildcard CORS, ANY origin can trigger authenticated requests
- Attacker doesn't see the response (browsers block it) but can:
  - Cause state-changing actions (fund transfers)
  - Monitor timing to infer data (timing-based attack)
```

### Attack Vector 3: Exposing Sensitive Headers

```
- CORS: * allows all headers except Authorization
- But CORS doesn't apply to simple requests (GET with basic headers)
- Non-browser tools (curl, Postman) ignore CORS entirely
- Internal network attackers can abuse wildcard origins
```

## Proper CORS Configuration

### 1. Explicit Allowlist

Only allow known, trusted origins:

```
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Origin: https://app.example.com
```

### 2. Environment-Specific

```
Development:  http://localhost:3000
Staging:      https://staging.example.com
Production:   https://example.com
```

### 3. Controlled Credentials

```
Access-Control-Allow-Credentials: true

# Must be used WITH explicit origin (never with *)
# Allows cookies to be sent with CORS requests
```

### 4. Restricted Methods

```
Access-Control-Allow-Methods: GET, POST
# Excludes: PUT, DELETE, PATCH

# Default safe methods:
# - GET (retrieval only)
# - HEAD (like GET, no body)
# - POST (with restrictions)
```

### 5. Restricted Headers

```
Access-Control-Allow-Headers: Content-Type, X-API-Key
# Excludes sensitive headers: Authorization, Cookie, etc.
```

### 6. Preflight Caching

```
Access-Control-Max-Age: 86400
# Browser caches preflight response for 24 hours
# Reduces unnecessary OPTIONS requests
```

## CORS Request Flow

### Simple Requests (No Preflight)

```
GET, POST, HEAD with basic headers
- Browser sends request directly
- Server responds with CORS headers
- Browser checks CORS policy
```

### Preflight Requests (Complex)

```
PUT, DELETE, PATCH
Custom headers, Content-Type: application/json
Authorization header

1. Browser sends OPTIONS request (preflight)
2. Server responds with allowed methods/headers
3. If allowed, browser sends actual request
4. Server responds with CORS headers again
```

## CORS vs Other Protections

| Protection | What it does |
|-----------|------------|
| **CORS** | Browser enforces origin checks on cross-domain requests |
| **CSRF Tokens** | Prevents state-changing requests without valid token |
| **SOP (Same-Origin Policy)** | Foundation - blocks all cross-origin access by default |
| **Content-Security-Policy** | Restricts resource loading (scripts, images, etc.) |

## Typical Attack Flow (Wildcard CORS)

```
1. Victim authenticated to bank.com
2. Victim visits attacker.com (background tab)
3. Attacker's script executes:
   fetch('https://bank.com/api/transfer', {
     method: 'POST',
     credentials: 'include',
     body: JSON.stringify({ to: 'attacker', amount: 1000 })
   })
4. Bank's server receives request with victim's cookies
5. Request passes authentication (cookies valid)
6. Bank transfers $1000 to attacker
7. Browser blocks response from being read (CORS)
   but damage is already done (state changed)
```

Solution: Bank requires CSRF token + CORS: explicit origin only
