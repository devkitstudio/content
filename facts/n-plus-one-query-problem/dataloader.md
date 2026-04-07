## The DataLoader Pattern (Facebook's Approach)

When using GraphQL or any resolver-based architecture, JOINs aren't always possible because each field resolver fetches data independently. DataLoader solves N+1 by **batching and caching** within a single request lifecycle.

### How It Works

```
Without DataLoader:
  resolve user(1) → SELECT * FROM users WHERE id = 1
  resolve user(2) → SELECT * FROM users WHERE id = 2
  resolve user(3) → SELECT * FROM users WHERE id = 3
  = 3 queries (N+1 pattern)

With DataLoader:
  resolve user(1) → loader.load(1)  ← queued
  resolve user(2) → loader.load(2)  ← queued
  resolve user(3) → loader.load(3)  ← queued
  // End of event loop tick:
  → SELECT * FROM users WHERE id IN (1, 2, 3)
  = 1 query (batched)
```

### Implementation

```typescript
import DataLoader from 'dataloader';

// Create a loader per request (not global — avoids stale cache across requests)
function createLoaders() {
  return {
    userLoader: new DataLoader(async (ids: readonly number[]) => {
      const users = await db.query(
        'SELECT * FROM users WHERE id = ANY($1)', [ids]
      );
      // IMPORTANT: return results in the SAME ORDER as input ids
      const userMap = new Map(users.map(u => [u.id, u]));
      return ids.map(id => userMap.get(id) || null);
    }),

    postsByAuthorLoader: new DataLoader(async (authorIds: readonly number[]) => {
      const posts = await db.query(
        'SELECT * FROM posts WHERE author_id = ANY($1)', [authorIds]
      );
      // Group by author_id, return array for each
      const grouped = new Map<number, Post[]>();
      posts.forEach(p => {
        const list = grouped.get(p.author_id) || [];
        list.push(p);
        grouped.set(p.author_id, list);
      });
      return authorIds.map(id => grouped.get(id) || []);
    }),
  };
}

// In GraphQL resolvers:
const resolvers = {
  Post: {
    author: (post, _, { loaders }) => loaders.userLoader.load(post.author_id),
  },
  User: {
    posts: (user, _, { loaders }) => loaders.postsByAuthorLoader.load(user.id),
  },
};
```

### DataLoader vs JOIN vs IN Clause

| Approach | Best For | Limitation |
|----------|----------|-----------|
| **JOIN** | SQL-based APIs, simple relations | Cartesian explosion with multiple JOINs, not usable in resolver architectures |
| **IN clause** | Known batch of IDs upfront | Need to manually collect IDs before querying |
| **DataLoader** | GraphQL, resolver patterns, nested data | Per-request only, adds library dependency |
| **Prisma `include`** | Prisma ORM projects | Prisma-specific, generates JOINs or IN under the hood |

### ORM-Specific Solutions

```typescript
// Prisma: built-in eager loading (generates optimized queries)
const posts = await prisma.post.findMany({
  include: {
    author: true,        // 1 extra query with IN clause
    comments: {
      include: { user: true }  // 1 more query, not N
    }
  }
});

// Drizzle: explicit joins
const result = await db
  .select()
  .from(posts)
  .leftJoin(users, eq(posts.authorId, users.id));

// TypeORM: relation loading
const posts = await postRepo.find({
  relations: ['author', 'comments', 'comments.user'],
});
```
