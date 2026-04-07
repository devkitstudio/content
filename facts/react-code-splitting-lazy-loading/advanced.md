# Route-Based vs Component-Based Splitting & Prefetching

## Route-Based Splitting (Recommended)

```javascript
// Routes.js
import { lazy } from 'react';
import { createBrowserRouter } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Profile = lazy(() => import('./pages/Profile'));
const Settings = lazy(() => import('./pages/Settings'));

export const router = createBrowserRouter([
  { path: '/', element: <Home /> },
  { path: '/dashboard', element: <Dashboard /> },
  { path: '/profile', element: <Profile /> },
  { path: '/settings', element: <Settings /> },
]);
```

**Benefits:**
- Easy to implement with routing libraries
- Clear separation of concerns
- Optimal chunk sizes
- Natural user journey

## Component-Based Splitting

```javascript
// Dashboard.js - split heavy features
const Analytics = lazy(() => import('./features/Analytics'));
const Reports = lazy(() => import('./features/Reports'));
const Export = lazy(() => import('./features/Export'));

function Dashboard({ tab }) {
  return (
    <Suspense fallback={<Skeleton />}>
      {tab === 'analytics' && <Analytics />}
      {tab === 'reports' && <Reports />}
      {tab === 'export' && <Export />}
    </Suspense>
  );
}
```

## Prefetching Strategies

### Link Prefetch

```javascript
function Navigation() {
  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/dashboard" prefetch="intent">Dashboard</Link>
      <Link to="/settings" prefetch="render">Settings</Link>
    </nav>
  );
}
```

### Programmatic Prefetch

```javascript
function usePrefetch() {
  const prefetchRoute = (path) => {
    // Preload chunk before route change
    import(path);
  };
  return prefetchRoute;
}

function Dashboard() {
  const prefetch = usePrefetch();

  return (
    <button
      onMouseEnter={() => prefetch('./pages/Settings')}
    >
      Go to Settings
    </button>
  );
}
```

### Request Idle Callback Prefetch

```javascript
function usePrefetchIdle(importPath) {
  useEffect(() => {
    if ('requestIdleCallback' in window) {
      requestIdleCallback(() => {
        import(importPath);
      });
    } else {
      setTimeout(() => {
        import(importPath);
      }, 2000);
    }
  }, [importPath]);
}

// Usage
usePrefetchIdle('./pages/Profile');
```

## Bundle Analysis

```bash
# Install analyzer
npm install --save-dev webpack-bundle-analyzer

# In webpack config
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer')
  .BundleAnalyzerPlugin;

plugins: [
  new BundleAnalyzerPlugin(),
],

# Build and view
npm run build
```

## Chunk Naming for Cache Busting

```javascript
// webpack.config.js
output: {
  filename: '[name].[contenthash].js',
  chunkFilename: '[name].[contenthash].chunk.js',
}
```

## Prefetching Comparison

| Strategy | Use Case | Benefits | Drawbacks |
|----------|----------|----------|-----------|
| Link Prefetch | Common navigation paths | Predictable, works with routers | May waste bandwidth |
| Intersection Observer | Below-the-fold components | Only loads when needed | Complex setup |
| Request Idle Callback | Non-critical chunks | Doesn't block main thread | Not all browsers |
| Manual onClick | User-triggered features | Full control | Requires planning |
