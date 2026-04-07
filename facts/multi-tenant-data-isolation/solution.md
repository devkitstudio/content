## Isolation Strategies

### 1. Row-Level Security (RLS) - Shared Database

**Best for**: Horizontal scaling, simple SaaS

```sql
-- Add tenant_id to all tables
CREATE TABLE users (
  id UUID PRIMARY KEY,
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  email VARCHAR,
  created_at TIMESTAMP
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their tenant's data
CREATE POLICY tenant_isolation ON users
  FOR SELECT
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

CREATE POLICY tenant_isolation_insert ON users
  FOR INSERT
  WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

**Application enforcement**:
```typescript
// Set tenant context for every query
const setTenantContext = async (tenantId: string, cb: any) => {
  const client = await pool.connect();
  try {
    await client.query(`SET app.current_tenant_id = '${tenantId}'`);
    return await cb(client);
  } finally {
    client.release();
  }
};

// Usage
const getUsers = (tenantId: string) =>
  setTenantContext(tenantId, async (client) => {
    return client.query('SELECT * FROM users');
  });
```

### 2. Schema-Level Isolation - Separate Schemas

**Best for**: Regulatory isolation, compliance

```sql
-- Create schema per tenant
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_bigcorp;

-- Create identical table structure in each schema
CREATE TABLE tenant_acme.users (
  id UUID PRIMARY KEY,
  email VARCHAR,
  created_at TIMESTAMP
);

CREATE TABLE tenant_bigcorp.users (
  id UUID PRIMARY KEY,
  email VARCHAR,
  created_at TIMESTAMP
);
```

**Application routing**:
```typescript
const getTenantSchema = (tenantId: string): string => {
  const schemaMap: Record<string, string> = {
    'acme': 'tenant_acme',
    'bigcorp': 'tenant_bigcorp'
  };
  return schemaMap[tenantId] || 'public';
};

const queryTenantData = async (tenantId: string, query: string) => {
  const schema = getTenantSchema(tenantId);
  return pool.query(`SET search_path TO ${schema}; ${query}`);
};
```

### 3. Database-Level Isolation - Separate DBs

**Best for**: High-security, enterprise

```typescript
const tenantDatabases: Record<string, string> = {
  'acme': 'postgresql://user:pass@db1:5432/acme_db',
  'bigcorp': 'postgresql://user:pass@db2:5432/bigcorp_db',
  'default': 'postgresql://user:pass@db:5432/default_db'
};

const getTenantPool = (tenantId: string) => {
  const connString = tenantDatabases[tenantId] || tenantDatabases['default'];
  return new Pool({ connectionString: connString });
};

const getOrCreatePool = (tenantId: string) => {
  if (!pools[tenantId]) {
    pools[tenantId] = getTenantPool(tenantId);
  }
  return pools[tenantId];
};
```

## Critical: Enforce at Every Layer

```typescript
// Middleware: Extract tenant from token
export const tenantMiddleware = (req: any, res: any, next: any) => {
  const token = req.headers.authorization?.split(' ')[1];
  const decoded = jwt.verify(token, SECRET);

  // Store tenant in request
  req.tenantId = decoded.tenantId;
  next();
};

// Repository: Always filter by tenant
export class UserRepository {
  async getById(tenantId: string, userId: string) {
    return db.query(
      'SELECT * FROM users WHERE id = $1 AND tenant_id = $2',
      [userId, tenantId]
    );
  }

  async list(tenantId: string) {
    return db.query(
      'SELECT * FROM users WHERE tenant_id = $1',
      [tenantId]
    );
  }
}

// Route: Use tenant from request
app.get('/users/:id', tenantMiddleware, async (req, res) => {
  const user = await userRepo.getById(req.tenantId, req.params.id);

  if (!user) return res.status(404).json({ error: 'Not found' });
  res.json(user);
});
```

## Testing Isolation

```typescript
test('User cannot access other tenant data', async () => {
  const acmeUser = await createUser('acme', 'user@acme.com');
  const bigcorpUser = await createUser('bigcorp', 'user@bigcorp.com');

  // Access with acme tenant
  const result = await getUser('acme', bigcorpUser.id);

  expect(result).toBeNull(); // Should not find other tenant's user
});

test('Tenant isolation in schema', async () => {
  await queryTenantData('acme', 'INSERT INTO users VALUES (...)');
  const bigcorpResult = await queryTenantData('bigcorp', 'SELECT * FROM users');

  expect(bigcorpResult.rows).toHaveLength(0); // Isolated data
});
```
