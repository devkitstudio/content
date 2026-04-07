# SQL Injection Examples and Prevention

## Vulnerable Code (What NOT to do)

```javascript
// VULNERABLE - String concatenation
app.get('/user/:id', (req, res) => {
  const id = req.params.id;
  const query = `SELECT * FROM users WHERE id = ${id}`;
  // Attack: /user/1 OR 1=1 -- returns all users
  db.query(query, (err, result) => res.json(result));
});

// VULNERABLE - Template strings don't help
const userId = req.query.id;
const query = `SELECT * FROM users WHERE id = ${userId}`;
// Still injectable!

// VULNERABLE - Concatenation in WHERE clause
const email = req.body.email;
const query = `SELECT * FROM users WHERE email = '${email}'`;
// Attack: admin'--' bypasses password check
```

## Secure Implementation - Parameterized Queries

```javascript
// SECURE - Using parameterized queries
app.get('/user/:id', async (req, res) => {
  const id = req.params.id;
  try {
    const result = await db.query(
      'SELECT * FROM users WHERE id = ?',
      [id]  // Parameter separated from query
    );
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: 'Database error' });
  }
});

// With named parameters (some drivers)
const result = await db.query(
  'SELECT * FROM users WHERE id = :id AND email = :email',
  { id: userId, email: userEmail }
);

// Attack: /user/1 OR 1=1 -- is treated as literal string "1 OR 1=1 --"
```

## ORM Approach - Prisma

```typescript
// SECURE - Using Prisma ORM
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function getUser(id: string) {
  // Type-safe, automatically parameterized
  const user = await prisma.user.findUnique({
    where: { id: parseInt(id) },  // Type coercion enforced
    select: { id: true, email: true, name: true }
  });
  return user;
}

// Prisma generates this SQL automatically:
// SELECT "public"."User"."id", "public"."User"."email",
//        "public"."User"."name"
// FROM "public"."User"
// WHERE "public"."User"."id" = $1

async function searchUsers(query: string) {
  // Parameterized search
  return await prisma.user.findMany({
    where: {
      OR: [
        { email: { contains: query } },
        { name: { contains: query } }
      ]
    }
  });
}

// Prisma prevents: query = "'; DROP TABLE users; --"
// Treated as literal search string
```

## ORM Approach - TypeORM

```typescript
import { getRepository } from 'typeorm';
import { User } from './entities/User';

class UserRepository {
  async findUserById(id: number) {
    // Safe query builder
    return await getRepository(User)
      .createQueryBuilder('user')
      .where('user.id = :id', { id })
      .getOne();
  }

  async searchUsers(searchTerm: string) {
    // Parameterized LIKE queries
    return await getRepository(User)
      .createQueryBuilder('user')
      .where('user.email LIKE :search', { search: `%${searchTerm}%` })
      .orWhere('user.name LIKE :search', { search: `%${searchTerm}%` })
      .getMany();
  }

  async findByEmail(email: string) {
    // Direct method with parameters
    return await getRepository(User)
      .findOne({ where: { email } });
  }
}

// TypeORM generates parameterized SQL automatically
// SELECT "User"."id", "User"."email" FROM "User"
// WHERE "User"."id" = $1
```

## Input Validation Layer

```typescript
import Joi from 'joi';
import validator from 'validator';

// Schema validation
const userSearchSchema = Joi.object({
  id: Joi.number().integer().min(1).max(2147483647).required(),
  email: Joi.string().email().max(255).required(),
  name: Joi.string().max(100).pattern(/^[a-zA-Z\s'-]+$/).required()
});

// Whitelist validation
function validateSearchInput(input: string): string {
  // Only allow alphanumeric + common characters
  if (!/^[a-zA-Z0-9\s@\.\-_]*$/.test(input)) {
    throw new Error('Invalid characters in search');
  }

  // Length limits
  if (input.length > 100) {
    throw new Error('Search term too long');
  }

  return input;
}

// Email validation
const email = req.body.email;
if (!validator.isEmail(email)) {
  return res.status(400).json({ error: 'Invalid email' });
}

// Middleware usage
app.post('/api/search', async (req, res) => {
  const { error, value } = userSearchSchema.validate(req.body);

  if (error) {
    return res.status(400).json({ error: error.details[0].message });
  }

  // value is validated and type-safe
  const results = await searchUsers(value);
  res.json(results);
});
```

## Least Privilege Database Users

```sql
-- Application read-only user
CREATE USER 'app_read'@'localhost' IDENTIFIED BY 'secure_password';
GRANT SELECT ON production.* TO 'app_read'@'localhost';

-- API writer user (limited permissions)
CREATE USER 'app_write'@'localhost' IDENTIFIED BY 'secure_password';
GRANT SELECT, INSERT, UPDATE ON production.users TO 'app_write'@'localhost';
REVOKE DELETE ON production.users FROM 'app_write'@'localhost';

-- No DDL permissions for any application user
-- Cannot CREATE, DROP, ALTER tables

-- Audit/sensitive data masked
CREATE USER 'app_safe'@'localhost' IDENTIFIED BY 'secure_password';
CREATE VIEW safe_users AS
  SELECT id, email, name FROM users;
GRANT SELECT ON production.safe_users TO 'app_safe'@'localhost';
```

```javascript
// Node.js: Use role-specific connections
const readPool = mysql.createPool({
  host: process.env.DB_HOST,
  user: 'app_read',  // Read-only user
  password: process.env.DB_READ_PASS,
  database: 'production'
});

const writePool = mysql.createPool({
  host: process.env.DB_HOST,
  user: 'app_write',  // Limited write user
  password: process.env.DB_WRITE_PASS,
  database: 'production'
});

app.get('/api/users/:id', async (req, res) => {
  const connection = await readPool.getConnection();
  try {
    const [rows] = await connection.execute(
      'SELECT id, email, name FROM users WHERE id = ?',
      [req.params.id]
    );
    res.json(rows[0]);
  } finally {
    connection.release();
  }
});
```

## WAF Configuration - CloudFlare/AWS

```yaml
# Cloudflare WAF rule to block SQL injection
{
  "expression": "(http.request.body.string contains \"union\") or (http.request.query_string contains \"or 1=1\") or (http.request.uri.path contains \"'--\")",
  "action": "block",
  "description": "Block SQL injection attempts"
}

# AWS WAF SQLiProtection rule set
Rules:
  - Name: AWSManagedRulesSQLiRuleSet
    Priority: 1
    Statement:
      ManagedRuleGroupStatement:
        VendorName: AWS
        Name: AWSManagedRulesSQLiRuleSet
    OverrideAction:
      Count: {}
    VisibilityConfig:
      SampledRequestsEnabled: true
      CloudWatchMetricsEnabled: true
      MetricName: SQLiProtectionMetric
```

## Error Handling - Safe Responses

```typescript
app.get('/api/user/:id', async (req, res) => {
  try {
    const id = parseInt(req.params.id, 10);
    if (isNaN(id)) {
      return res.status(400).json({ error: 'Invalid ID format' });
    }

    const user = await db.query(
      'SELECT id, email FROM users WHERE id = ?',
      [id]
    );

    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    // SAFE: Generic message to user
    res.status(500).json({ error: 'Database operation failed' });

    // VERBOSE: Detailed logging on server
    logger.error('Database error', {
      query: 'SELECT id, email FROM users WHERE id = ?',
      params: [req.params.id],
      error: error.message,
      stack: error.stack,
      userId: req.user?.id
    });
  }
});
```

## Monitoring for Injection Attempts

```typescript
import * as ml from 'ml.js';

// Detect anomalous SQL patterns
class SQLInjectionDetector {
  private suspiciousKeywords = [
    'UNION', 'EXEC', 'EXECUTE', 'DROP', 'DELETE',
    'INSERT', 'UPDATE', 'ALTER', 'DECLARE', 'CAST'
  ];

  detect(userInput: string): boolean {
    const upper = userInput.toUpperCase();

    // Check for SQL keywords
    if (this.suspiciousKeywords.some(kw => upper.includes(kw))) {
      return true;
    }

    // Check for comment syntax
    if (userInput.includes('--') || userInput.includes('/*')) {
      return true;
    }

    // Check for quote manipulation
    if ((userInput.match(/'/g) || []).length % 2 !== 0) {
      return true;
    }

    return false;
  }

  logAttempt(userInput: string, userId?: string) {
    logger.warn('Possible SQL injection attempt', {
      input: userInput,
      userId,
      timestamp: new Date(),
      ip: process.env.USER_IP
    });

    // Alert if multiple attempts
    metrics.increment('sql_injection_attempts');
    if (metrics.get('sql_injection_attempts') > 5) {
      alerts.notifySecurityTeam('High SQL injection attempt rate');
    }
  }
}
```
