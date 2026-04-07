# Code Splitting & Lazy Loading with React.lazy & Suspense

## Dynamic Import with React.lazy

```javascript
// Before: Single bundle
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';

// After: Code splitting
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const Settings = React.lazy(() => import('./pages/Settings'));
```

## Suspense Boundaries

```javascript
import { Suspense } from 'react';

function Router() {
  return (
    <Routes>
      <Route
        path="/dashboard"
        element={
          <Suspense fallback={<LoadingSpinner />}>
            <Dashboard />
          </Suspense>
        }
      />
      <Route
        path="/settings"
        element={
          <Suspense fallback={<LoadingSpinner />}>
            <Settings />
          </Suspense>
        }
      />
    </Routes>
  );
}
```

## Component-Level Code Splitting

```javascript
// Lazy load components within pages
const Chart = React.lazy(() => import('./Chart'));
const DataTable = React.lazy(() => import('./DataTable'));

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<div>Loading chart...</div>}>
        <Chart />
      </Suspense>
      <Suspense fallback={<div>Loading table...</div>}>
        <DataTable />
      </Suspense>
    </div>
  );
}
```

## Named Exports with Code Splitting

```javascript
// Use React.lazy with named exports
const Overview = React.lazy(() =>
  import('./pages/Dashboard').then(mod => ({ default: mod.Overview }))
);

const Details = React.lazy(() =>
  import('./pages/Dashboard').then(mod => ({ default: mod.Details }))
);
```

## Error Boundary with Fallback

```javascript
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return <h1>Failed to load module</h1>;
    }
    return this.props.children;
  }
}

function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={<LoadingSpinner />}>
        <Dashboard />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## Expected Results
- Initial bundle reduced from 2.5MB to ~300KB
- First load on 3G: 8s → 2-3s
- Route transitions with Suspense provide visual feedback
