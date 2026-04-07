## Solution 3: Query Result Caching

When the same N+1 query runs repeatedly for the same data, cache the results instead of hitting the database every time.

```typescript
// Redis cache layer
async function getUserWithPosts(userId: number) {
  const cacheKey = `user:${userId}:with_posts`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const user = await db.query(`
    SELECT u.*, json_agg(p.*) AS posts
    FROM users u
    LEFT JOIN posts p ON p.author_id = u.id
    WHERE u.id = $1
    GROUP BY u.id
  `, [userId]);

  await redis.setex(cacheKey, 300, JSON.stringify(user)); // 5 min TTL
  return user;
}
```

### Cache Invalidation Strategy

```typescript
// Invalidate when related data changes
async function createPost(authorId: number, data: PostData) {
  await db.query('INSERT INTO posts ...', [data]);

  // Bust the cache for this author
  await redis.del(`user:${authorId}:with_posts`);

  // Or use pattern-based invalidation
  const keys = await redis.keys(`user:${authorId}:*`);
  if (keys.length) await redis.del(...keys);
}
```

## Solution 4: Denormalization

For read-heavy workloads, store pre-computed data to avoid JOINs entirely.

### Embedding (Document-Style)

```typescript
// Instead of: users table + posts table + JOIN
// Store the count/summary directly on the user record

// PostgreSQL: materialized JSON column
await db.query(`
  ALTER TABLE users ADD COLUMN posts_summary jsonb DEFAULT '[]';
`);

// Update on write (trigger or application code)
async function onPostCreated(post: Post) {
  await db.query(`
    UPDATE users SET posts_summary = (
      SELECT jsonb_agg(jsonb_build_object('id', id, 'title', title, 'date', created_at))
      FROM posts WHERE author_id = $1
      ORDER BY created_at DESC
      LIMIT 10
    ) WHERE id = $1
  `, [post.author_id]);
}

// Read: zero JOINs, instant response
const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
// user.posts_summary is already there
```

### Materialized View

```sql
-- Create a pre-joined view refreshed periodically
CREATE MATERIALIZED VIEW user_post_stats AS
SELECT
  u.id, u.name, u.email,
  COUNT(p.id) AS post_count,
  MAX(p.created_at) AS last_post_at,
  array_agg(p.title ORDER BY p.created_at DESC) FILTER (WHERE p.id IS NOT NULL) AS recent_titles
FROM users u
LEFT JOIN posts p ON p.author_id = u.id
GROUP BY u.id;

-- Refresh every 5 minutes via cron
REFRESH MATERIALIZED VIEW CONCURRENTLY user_post_stats;

-- Query: instant, zero JOINs
SELECT * FROM user_post_stats WHERE id = 42;
```

## Interview Decision Matrix

When the interviewer asks "which approach would you use?", use this to pick:

| Scenario | Best Approach | Why |
|----------|--------------|-----|
| Simple REST API, SQL DB | **JOIN or IN clause** | Simplest, no extra infrastructure |
| GraphQL API | **DataLoader** | Works with resolver architecture |
| Same query repeated 100x/min | **Cache (Redis)** | Avoid hitting DB at all |
| Dashboard with complex aggregations | **Materialized View** | Pre-computed, instant reads |
| Write-heavy, read-light | **Just fix the query** | Don't over-optimize writes |
| Read-heavy, rarely changes | **Denormalize** | Trade storage for speed |
