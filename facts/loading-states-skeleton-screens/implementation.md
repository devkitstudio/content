# Advanced Implementation & Suspense Boundaries

## TanStack Query with Skeleton

```javascript
import { useQuery } from '@tanstack/react-query';

function UserProfile({ userId }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
  });

  return (
    <div className="profile-container">
      {isLoading ? <SkeletonProfile /> : <Profile user={user} />}
    </div>
  );
}

// Skeleton exactly matches Profile component layout
function SkeletonProfile() {
  return (
    <div className="profile-container">
      <div className="skeleton-avatar" />
      <div className="skeleton-info">
        <div className="skeleton-title" />
        <div className="skeleton-text" />
      </div>
    </div>
  );
}
```

## Suspense Boundaries with React.lazy

```javascript
import { Suspense, lazy } from 'react';

const ArticleBody = lazy(() => import('./ArticleBody'));
const Comments = lazy(() => import('./Comments'));
const RelatedPosts = lazy(() => import('./RelatedPosts'));

function ArticlePage({ id }) {
  return (
    <article>
      <ArticleHeader id={id} />

      <Suspense fallback={<SkeletonBody />}>
        <ArticleBody id={id} />
      </Suspense>

      <Suspense fallback={<SkeletonComments />}>
        <Comments id={id} />
      </Suspense>

      <Suspense fallback={<SkeletonRelated />}>
        <RelatedPosts id={id} />
      </Suspense>
    </article>
  );
}
```

## Nested Suspense Boundaries

```javascript
function Dashboard() {
  return (
    <div>
      <Header /> {/* Renders immediately */}

      <Suspense fallback={<SkeletonSidebar />}>
        <Sidebar />
      </Suspense>

      <Suspense fallback={<SkeletonMainContent />}>
        <MainContent />
      </Suspense>
    </div>
  );
}

// Each component can have independent loading states
function MainContent() {
  return (
    <>
      <Suspense fallback={<SkeletonChart />}>
        <Chart />
      </Suspense>

      <Suspense fallback={<SkeletonTable />}>
        <DataTable />
      </Suspense>
    </>
  );
}
```

## CSS-in-JS Skeleton Animation

```javascript
import styled, { keyframes } from 'styled-components';

const shimmer = keyframes`
  0% {
    background-position: -1000px 0;
  }
  100% {
    background-position: 1000px 0;
  }
`;

const SkeletonWrapper = styled.div`
  background: linear-gradient(
    90deg,
    #f0f0f0 0%,
    #e0e0e0 50%,
    #f0f0f0 100%
  );
  background-size: 1000px 100%;
  animation: ${shimmer} 2s infinite;
  border-radius: 4px;
`;

function Skeleton({ width = '100%', height = '12px' }) {
  return <SkeletonWrapper style={{ width, height }} />;
}
```

## Avoiding CLS with Aspect Ratio

```javascript
// Image placeholder with fixed aspect ratio
function ImageSkeleton() {
  return (
    <div className="image-wrapper">
      <div className="image-skeleton" />
    </div>
  );
}

const styles = `
  .image-wrapper {
    aspect-ratio: 16 / 9;
    overflow: hidden;
  }

  .image-skeleton {
    width: 100%;
    height: 100%;
    background: linear-gradient(...);
    animation: shimmer 2s infinite;
  }
`;
```

## Staggered Skeleton Loading

```javascript
function UserListSkeleton({ count = 5 }) {
  return (
    <div className="user-list">
      {Array.from({ length: count }).map((_, i) => (
        <div
          key={i}
          style={{
            animation: `fadeIn 0.3s ease-in ${i * 0.1}s both`,
          }}
        >
          <Skeleton />
        </div>
      ))}
    </div>
  );
}

const fadeIn = keyframes`
  from { opacity: 0; }
  to { opacity: 1; }
`;
```

## Content Replacement Pattern

```javascript
function UseQueryWithSkeleton({ queryKey, queryFn, skeletonComponent: SkeletonComponent }) {
  const { data, isLoading, error } = useQuery({ queryKey, queryFn });

  if (isLoading) return <SkeletonComponent />;
  if (error) return <ErrorMessage error={error} />;
  return <div>{data}</div>;
}

// Usage
<UseQueryWithSkeleton
  queryKey={['user']}
  queryFn={() => fetchUser()}
  skeletonComponent={UserSkeleton}
/>
```

## Multi-Stage Loading

```javascript
function Post({ id }) {
  const [stage, setStage] = useState('skeleton'); // skeleton → content → full

  useEffect(() => {
    // Load header first
    fetchPostHeader(id).then(() => {
      setStage('header');
      // Then load body
      fetchPostBody(id).then(() => {
        setStage('full');
      });
    });
  }, [id]);

  return (
    <div>
      {stage === 'skeleton' && <SkeletonPost />}
      {stage === 'header' && <PostHeader id={id} />}
      {stage === 'full' && <FullPost id={id} />}
    </div>
  );
}
```

## Best Practices Checklist

- Match skeleton dimensions exactly to real content
- Use animation (shimmer) to indicate loading, not static skeleton
- Provide multiple skeleton variants for different content types
- Never show skeleton longer than 1-2 seconds (or prefetch)
- Use Suspense boundaries for independent content areas
- Prevent CLS by reserving space with fixed heights/aspect ratios
- Test skeleton appearance matches real content width/height
