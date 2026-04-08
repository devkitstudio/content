# API Versioning Strategy Comparison

## Side-by-Side Comparison

| Aspect | URL Path | Header | Query Parameter |
|--------|----------|--------|-----------------|
| **Visibility** | Clear | Hidden | Visible |
| **URL Length** | Longer | Shorter | Slightly longer |
| **Caching** | Easy | Hard | Depends |
| **Browser Testing** | Easy | Hard | Easy |
| **Mobile Apps** | Easy | Harder | Easy |
| **Convention** | REST standard | HTTP spec | Uncommon |
| **Learning Curve** | Easiest | Medium | Easy |
| **Code Organization** | Separate files | Conditional logic | Conditional logic |
| **Deprecation** | Clear in URL | Via headers | Via params |

## When to Use Each

### Use URL Path Versioning (/api/v1/) if:
- Supporting multiple major versions simultaneously
- Need clear separation of code
- Want explicit version visibility
- Building public APIs
- Clients need to upgrade intentionally

```javascript
// Good for public APIs with long-term version support
// /api/v1/users
// /api/v2/users
// /api/v3/users (current)

// All versions coexist
// Clear which version is being used
```

### Use Header Versioning if:
- Supporting mostly one version at a time
- Want cleaner URL structure
- Building internal/partner APIs
- Can maintain single codebase with conditionals

```javascript
// Good for internal APIs
// Accept-Version: 2
// Cleaner URL: /api/users
// Single endpoint with conditional logic
```

### Use Query Parameter Versioning if:
- Version is optional or default-heavy
- Need easy testing across versions
- Building mobile APIs with feature flags
- Version is secondary concern

```javascript
// Good for feature flags and experiments
// /api/users?v=2&experimental=true
// Easy to test different versions in browser
```

## Real-World Examples

### Stripe (URL Path Versioning)
```
GET /v1/customers        # API version in path
GET /v1/charges          # All endpoints versioned
GET /v1/invoices
```
- Multiple major versions coexist
- Clear deprecation timeline
- Clients choose when to upgrade

### GitHub API (Header Versioning)
```
GET /repos/user/repo
Accept-Version: 2022-11-28  # Version as date in header
```
- Single endpoint structure
- Rolled out gradually
- Easier deprecation

### Twitter API (URL + Parameter)
```
GET /2/tweets?tweet.fields=author_id,created_at
GET /1.1/statuses/update
```
- Mix of path and parameter versioning
- Flexible, but more complex

## Deprecation Timeline Example

```javascript
// Track usage and set sunset dates
const versions = {
  '1': {
    status: 'deprecated',
    sunsetDate: '2024-12-31',
    usage: 2.5,  // percentage of requests
    message: 'Please upgrade to v2'
  },
  '2': {
    status: 'stable',
    usage: 70,
    message: 'Current version'
  },
  '3': {
    status: 'beta',
    usage: 27.5,
    message: 'Latest version, send feedback'
  }
};

// Middleware warns clients about deprecation
app.use((req, res, next) => {
  const version = req.headers['accept-version'] || '1';
  const versionInfo = versions[version];

  if (versionInfo?.status === 'deprecated') {
    res.set('Deprecation', 'true');
    res.set('Sunset', versionInfo.sunsetDate);
    res.set('X-API-Warn', `API v${version} is deprecated. Upgrade to v${latestVersion()}`);
  }

  next();
});
```

## Migration Path for Clients

### For Mobile App (On App Store)
```javascript
// Detect app version and API version
const appVersion = getAppVersion(); // e.g., "1.5.2"

// Graceful degradation
function selectAPIVersion() {
  if (appVersion >= '2.0.0') return 'v3';     // Newest version
  if (appVersion >= '1.5.0') return 'v2';     // Mid version
  return 'v1';                                // Legacy version
}

// Make request with appropriate version
const version = selectAPIVersion();
const url = `/api/${version}/users`;
```

### For Web Client (Auto-Updates)
```javascript
// Can always use latest version
// Web apps update automatically
fetch('/api/v3/users'); // Always use latest
```


