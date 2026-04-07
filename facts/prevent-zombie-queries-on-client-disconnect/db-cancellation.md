## Database-Specific Query Cancellation

When your ORM doesn't support `AbortSignal`, you need to cancel queries at the database level directly.

### PostgreSQL: `pg_cancel_backend` / `pg_terminate_backend`

```typescript
import { Pool } from 'pg';

const pool = new Pool();

async function executeWithCancellation(sql: string, signal: AbortSignal) {
  const client = await pool.connect();

  try {
    // Get the backend PID for this connection
    const { rows } = await client.query('SELECT pg_backend_pid() AS pid');
    const pid = rows[0].pid;

    // Listen for abort → cancel the query on this PID
    const onAbort = () => {
      // Use a SEPARATE connection to cancel (can't cancel on the busy one)
      pool.query('SELECT pg_cancel_backend($1)', [pid])
        .catch(err => console.error('Cancel failed:', err));
    };
    signal.addEventListener('abort', onAbort, { once: true });

    const result = await client.query(sql);
    signal.removeEventListener('abort', onAbort);
    return result;
  } catch (err: any) {
    if (err.code === '57014') {
      // 57014 = query_canceled in PostgreSQL
      console.log('Query was cancelled successfully');
      return null;
    }
    throw err;
  } finally {
    client.release();
  }
}
```

**Key difference**: `pg_cancel_backend(pid)` sends a cancel signal (graceful), `pg_terminate_backend(pid)` kills the entire connection (forced). Always try cancel first.

### MySQL: `KILL QUERY`

```typescript
import mysql from 'mysql2/promise';

const pool = mysql.createPool({ /* config */ });

async function executeWithCancellation(sql: string, signal: AbortSignal) {
  const conn = await pool.getConnection();

  try {
    const [rows] = await conn.query('SELECT CONNECTION_ID() AS id');
    const threadId = (rows as any)[0].id;

    const onAbort = () => {
      // KILL QUERY only kills the running query, not the connection
      pool.query(`KILL QUERY ${threadId}`)
        .catch(err => console.error('Kill failed:', err));
    };
    signal.addEventListener('abort', onAbort, { once: true });

    const result = await conn.query(sql);
    signal.removeEventListener('abort', onAbort);
    return result;
  } finally {
    conn.release();
  }
}
```

**MySQL variants**: `KILL QUERY <id>` kills only the current query. `KILL <id>` kills the entire connection. Always prefer `KILL QUERY`.

### PostgreSQL: `statement_timeout` as Safety Net

Even if your code handles cancellation perfectly, always set a maximum query time:

```sql
-- Per connection (set in your pool config)
SET statement_timeout = '30s';

-- Per query (wrap critical queries)
SET LOCAL statement_timeout = '5s';
SELECT * FROM massive_table WHERE ...;
RESET statement_timeout;

-- Global default (postgresql.conf)
statement_timeout = 60000  -- 60 seconds in ms
```

```typescript
// In your pool config
const pool = new Pool({
  connectionTimeoutMillis: 5000,   // wait 5s for a connection
  statement_timeout: 30000,         // kill queries after 30s
  idle_in_transaction_session_timeout: 60000, // kill idle transactions
});
```

### Prisma Workaround

Prisma doesn't natively support AbortSignal yet. Use `$queryRawUnsafe` + `statement_timeout`:

```typescript
// Set a per-query timeout via raw SQL
async function prismaWithTimeout<T>(
  prisma: PrismaClient,
  query: string,
  timeoutMs: number = 10000
): Promise<T> {
  return prisma.$transaction(async (tx) => {
    await tx.$executeRawUnsafe(`SET LOCAL statement_timeout = '${timeoutMs}'`);
    return tx.$queryRawUnsafe(query) as T;
  });
}

// With manual cancellation via AbortSignal
async function prismaWithCancellation(
  prisma: PrismaClient,
  signal: AbortSignal
) {
  // Get the PID from Prisma's underlying connection
  const [{ pid }] = await prisma.$queryRaw<[{ pid: number }]>`
    SELECT pg_backend_pid() AS pid
  `;

  const onAbort = () => {
    prisma.$executeRaw`SELECT pg_cancel_backend(${pid})`.catch(() => {});
  };
  signal.addEventListener('abort', onAbort, { once: true });

  try {
    const result = await prisma.hugeTable.findMany({ /* ... */ });
    return result;
  } finally {
    signal.removeEventListener('abort', onAbort);
  }
}
```
