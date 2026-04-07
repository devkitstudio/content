# Skeleton Screens Pattern

## Basic Skeleton Component

```javascript
// Skeleton.jsx
function Skeleton() {
  return (
    <div className="skeleton">
      <div className="skeleton-line skeleton-title" />
      <div className="skeleton-line skeleton-text" />
      <div className="skeleton-line skeleton-text" />
      <div className="skeleton-line skeleton-text-short" />
    </div>
  );
}

export default Skeleton;
```

## CSS Skeleton Animation

```css
.skeleton {
  padding: 16px;
  background: #f5f5f5;
  border-radius: 8px;
}

.skeleton-line {
  height: 12px;
  background: linear-gradient(
    90deg,
    #f0f0f0 0%,
    #e0e0e0 50%,
    #f0f0f0 100%
  );
  background-size: 200% 100%;
  animation: shimmer 2s infinite;
  margin-bottom: 12px;
  border-radius: 4px;
}

.skeleton-title {
  height: 24px;
  width: 60%;
}

.skeleton-text {
  height: 14px;
  width: 100%;
}

.skeleton-text-short {
  width: 40%;
}

@keyframes shimmer {
  0% {
    background-position: 200% 0;
  }
  100% {
    background-position: -200% 0;
  }
}
```

## UserCard with Skeleton

```javascript
function UserCard({ user, isLoading }) {
  if (isLoading) {
    return <Skeleton />;
  }

  return (
    <div className="user-card">
      <img src={user.avatar} alt={user.name} />
      <h3>{user.name}</h3>
      <p>{user.bio}</p>
    </div>
  );
}
```

## Suspense-Based Loading

```javascript
import { Suspense } from 'react';

function UserCardSuspense({ userId }) {
  const user = use(fetchUser(userId));

  return (
    <div className="user-card">
      <img src={user.avatar} alt={user.name} />
      <h3>{user.name}</h3>
    </div>
  );
}

function App() {
  return (
    <Suspense fallback={<Skeleton />}>
      <UserCardSuspense userId="123" />
    </Suspense>
  );
}
```

## Multiple Skeleton Loaders

```javascript
function UserListSkeleton() {
  return (
    <div className="user-list">
      {Array.from({ length: 5 }).map((_, i) => (
        <Skeleton key={i} />
      ))}
    </div>
  );
}

function UserList() {
  const [users, setUsers] = useState([]);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetchUsers().then(data => {
      setUsers(data);
      setIsLoading(false);
    });
  }, []);

  if (isLoading) {
    return <UserListSkeleton />;
  }

  return (
    <div className="user-list">
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}
```

## Skeleton Content Mapping

```javascript
// Exact shape match of real content
function SkeletonPost() {
  return (
    <div className="post-container">
      <div className="skeleton-avatar" />
      <div className="skeleton-header">
        <div className="skeleton-title" />
        <div className="skeleton-date" />
      </div>
      <div className="skeleton-body">
        <div className="skeleton-line" />
        <div className="skeleton-line" />
        <div className="skeleton-line-short" />
      </div>
      <div className="skeleton-actions">
        <div className="skeleton-button" />
        <div className="skeleton-button" />
      </div>
    </div>
  );
}

function Post({ post, isLoading }) {
  return isLoading ? <SkeletonPost /> : <RealPost post={post} />;
}
```

## Avoiding Layout Shift

```javascript
// Good: Fixed height container
function UserCardContainer() {
  return (
    <div style={{ minHeight: '200px' }}>
      {/* Skeleton or content fits in same space */}
    </div>
  );
}

// Bad: Different heights cause shift
function BadUserCardContainer() {
  return (
    <div>
      {/* Skeleton is 100px, real content is 300px */}
    </div>
  );
}
```

## Progressive Content Loading

```javascript
function Article() {
  const { data: article, isLoading } = useQuery(['article']);

  return (
    <article>
      {/* Header always loads first */}
      {isLoading ? <SkeletonHeader /> : <Header data={article} />}

      {/* Content loads with lazy loading */}
      <Suspense fallback={<SkeletonContent />}>
        <ArticleBody data={article} />
      </Suspense>
    </article>
  );
}
```

## Key Points

- Match skeleton shape exactly to real content
- Use fixed heights to prevent CLS (Cumulative Layout Shift)
- Animate shimmer effect for perceived performance
- Load critical content first (header before body)
