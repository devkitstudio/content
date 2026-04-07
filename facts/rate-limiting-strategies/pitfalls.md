## Mistakes That Break Rate Limiting in Production

### 1. Rate Limiting by IP Only
Behind a corporate NAT or mobile carrier, thousands of users share one IP. You'll block legitimate users.

**Fix**: Combine IP + API key + User ID for granular limits:
```
Key = hash(IP + API_KEY + USER_ID)
```

### 2. Forgetting Distributed State
If you have 5 app servers, each with its own in-memory counter, a user gets `5x` the actual limit.

**Fix**: Use a centralized store (Redis) with atomic operations:
```redis
-- Atomic increment + expire in one round-trip
MULTI
INCR rate:user:123
EXPIRE rate:user:123 60
EXEC
```

### 3. Not Returning Proper Headers
Clients need to know their limit status. Without headers, they can't self-throttle.

**Always return**:
```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1712435260
Retry-After: 30
```

### 4. Applying the Same Limit Everywhere
- Read endpoints (GET) can tolerate higher limits
- Write endpoints (POST/PUT/DELETE) need stricter limits
- Authentication endpoints need the strictest limits

```typescript
const LIMITS = {
  'GET /api/search':   { rate: 60, window: 60 },
  'POST /api/payment': { rate: 5,  window: 60 },
  'POST /api/login':   { rate: 3,  window: 300 },
}
```
