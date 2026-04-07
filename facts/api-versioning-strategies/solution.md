# API Versioning: Three Main Approaches

## Strategy 1: URL Path Versioning (Most Common)

```javascript
// Separate routes for different API versions
app.get('/api/v1/users/:id', handleV1GetUser);
app.get('/api/v2/users/:id', handleV2GetUser);
app.get('/api/v3/users/:id', handleV3GetUser);

// V1: Older response format
function handleV1GetUser(req, res) {
  const user = getUser(req.params.id);
  res.json({
    id: user.id,
    name: user.name,
    email: user.email
    // No address, no subscription info
  });
}

// V2: Added new fields
function handleV2GetUser(req, res) {
  const user = getUser(req.params.id);
  res.json({
    id: user.id,
    name: user.name,
    email: user.email,
    address: user.address,  // NEW
    created_at: user.createdAt // NEW
  });
}

// V3: Renamed fields, new structure
function handleV3GetUser(req, res) {
  const user = getUser(req.params.id);
  res.json({
    id: user.id,
    fullName: user.name,       // Renamed from 'name'
    emailAddress: user.email,  // Renamed from 'email'
    address: user.address,
    subscription: {            // NEW nested object
      tier: user.subscriptionTier,
      renewalDate: user.subscriptionRenewal
    },
    createdAt: user.createdAt  // camelCase in v3
  });
}
```

**Pros:**
- Clear API versioning in URL
- Easy for clients to understand which version they use
- Browser can access directly

**Cons:**
- Code duplication across routes
- Multiple endpoints to maintain

## Strategy 2: Header Versioning

```javascript
// Single endpoint, version via Accept-Version header
app.get('/api/users/:id', (req, res) => {
  const version = req.headers['accept-version'] || '1';
  const user = getUser(req.params.id);

  switch (version) {
    case '1':
      return res.json({
        id: user.id,
        name: user.name,
        email: user.email
      });

    case '2':
      return res.json({
        id: user.id,
        name: user.name,
        email: user.email,
        address: user.address,
        createdAt: user.createdAt
      });

    case '3':
      return res.json({
        id: user.id,
        fullName: user.name,
        emailAddress: user.email,
        address: user.address,
        subscription: {
          tier: user.subscriptionTier,
          renewalDate: user.subscriptionRenewal
        },
        createdAt: user.createdAt
      });

    default:
      return res.status(400).json({
        error: `API version ${version} not supported`
      });
  }
});

// Client usage:
// curl -H "Accept-Version: 2" https://api.example.com/api/users/123
```

**Pros:**
- Single endpoint, cleaner URL structure
- Version negotiation like content negotiation
- Easy to deprecate old versions

**Cons:**
- Not visible in URL or browser
- Requires understanding of HTTP headers

## Strategy 3: Query Parameter Versioning

```javascript
// Version via query parameter
app.get('/api/users/:id', (req, res) => {
  const version = req.query.v || '1';
  const user = getUser(req.params.id);

  // Respond based on version
  const responses = {
    '1': () => ({
      id: user.id,
      name: user.name,
      email: user.email
    }),
    '2': () => ({
      id: user.id,
      name: user.name,
      email: user.email,
      address: user.address,
      createdAt: user.createdAt
    }),
    '3': () => ({
      id: user.id,
      fullName: user.name,
      emailAddress: user.email,
      address: user.address,
      subscription: {
        tier: user.subscriptionTier,
        renewalDate: user.subscriptionRenewal
      },
      createdAt: user.createdAt
    })
  };

  const handler = responses[version];
  if (!handler) {
    return res.status(400).json({
      error: `API version ${version} not supported`
    });
  }

  res.json(handler());
});

// Client usage:
// https://api.example.com/api/users/123?v=2
```

**Pros:**
- Visible in URL and browser
- Easy for mobile apps to test different versions

**Cons:**
- Less conventional than path versioning
- Can complicate caching

## Minimize Code Duplication with Shared Logic

```javascript
// Shared base logic
class UserSerializer {
  constructor(user) {
    this.user = user;
  }

  // Base fields used by all versions
  baseFields() {
    return {
      id: this.user.id
    };
  }

  // Version-specific transformations
  v1() {
    return {
      ...this.baseFields(),
      name: this.user.name,
      email: this.user.email
    };
  }

  v2() {
    return {
      ...this.v1(),
      address: this.user.address,
      created_at: this.user.createdAt
    };
  }

  v3() {
    return {
      id: this.user.id,
      fullName: this.user.name,
      emailAddress: this.user.email,
      address: this.user.address,
      subscription: {
        tier: this.user.subscriptionTier,
        renewalDate: this.user.subscriptionRenewal
      },
      createdAt: this.user.createdAt
    };
  }

  serialize(version = '1') {
    const method = this[`v${version}`];
    if (!method) {
      throw new Error(`Version ${version} not supported`);
    }
    return method.call(this);
  }
}

// Usage
app.get('/api/users/:id', (req, res) => {
  const user = getUser(req.params.id);
  const version = req.headers['accept-version'] || '1';
  const serializer = new UserSerializer(user);
  res.json(serializer.serialize(version));
});
```

## Deprecation Strategy

```javascript
// Track API version usage
const versionMetrics = {};

function trackVersionUsage(version) {
  versionMetrics[version] = (versionMetrics[version] || 0) + 1;
}

// Middleware to add deprecation headers
function deprecationMiddleware(req, res, next) {
  const version = req.headers['accept-version'] || '1';

  trackVersionUsage(version);

  // Add deprecation warnings to responses
  if (version === '1') {
    res.set('Deprecation', 'true');
    res.set('Sunset', '2026-12-31T00:00:00Z');
    res.set('Link', '<https://docs.example.com/v2>; rel="successor-version"');
  }

  next();
}

app.use(deprecationMiddleware);

// Version usage endpoint (admin only)
app.get('/admin/api-version-stats', authenticateAdmin, (req, res) => {
  res.json(versionMetrics);
});
```
