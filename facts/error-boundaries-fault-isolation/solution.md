# Error Boundaries Implementation

## Basic Error Boundary Class Component

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorInfo: null,
    };
  }

  static getDerivedStateFromError(error) {
    // Update state so next render will show fallback UI
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    // Log error to error reporting service
    console.error('Error caught by boundary:', error, errorInfo);
    this.setState({
      error,
      errorInfo,
    });

    // Send to error tracking (Sentry, LogRocket, etc.)
    logErrorToService({
      error: error.toString(),
      componentStack: errorInfo.componentStack,
    });
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary-fallback">
          <h2>Something went wrong</h2>
          <details style={{ whiteSpace: 'pre-wrap' }}>
            {this.state.error && this.state.error.toString()}
            <br />
            {this.state.errorInfo?.componentStack}
          </details>
        </div>
      );
    }

    return this.props.children;
  }
}

export default ErrorBoundary;
```

## Usage: Wrap Components

```javascript
function App() {
  return (
    <ErrorBoundary>
      <Header />
      <ErrorBoundary>
        <Dashboard />
      </ErrorBoundary>
      <ErrorBoundary>
        <Sidebar />
      </ErrorBoundary>
      <Footer />
    </ErrorBoundary>
  );
}
```

## Isolated Error Boundaries

```javascript
// Granular error boundaries - only affected component crashes
function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>

      <ErrorBoundary>
        <Analytics />
      </ErrorBoundary>

      <ErrorBoundary>
        <Charts />
      </ErrorBoundary>

      <ErrorBoundary>
        <DataTable />
      </ErrorBoundary>
    </div>
  );
}

// If Analytics crashes, only Analytics shows error
// Charts and DataTable continue working
```

## Advanced Error Boundary with Reset

```javascript
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      hasError: false,
      error: null,
      errorCount: 0,
    };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    const errorCount = this.state.errorCount + 1;

    console.error('Error:', error);
    this.setState({ error, errorCount });

    // Reload app if too many errors
    if (errorCount > 3) {
      window.location.reload();
    }
  }

  resetError = () => {
    this.setState({
      hasError: false,
      error: null,
    });
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-page">
          <h1>Something went wrong</h1>
          <p>{this.state.error?.message}</p>
          <button onClick={this.resetError}>Try Again</button>
          {this.state.errorCount > 3 && (
            <p>Multiple errors detected. Reload page.</p>
          )}
        </div>
      );
    }

    return this.props.children;
  }
}
```

## Error Boundary with Fallback UI

```javascript
function SafeThirdPartyWidget({ widgetId }) {
  return (
    <ErrorBoundary fallback={<WidgetPlaceholder />}>
      <ThirdPartyWidget id={widgetId} />
    </ErrorBoundary>
  );
}

function WidgetPlaceholder() {
  return (
    <div style={{ padding: '20px', textAlign: 'center', background: '#f0f0f0' }}>
      <p>Widget could not load</p>
      <button onClick={() => window.location.reload()}>Reload</button>
    </div>
  );
}
```

## Catching Async Errors

```javascript
// Error Boundary CANNOT catch async errors
// Need to use try-catch or Promise.catch

function AsyncComponent() {
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/data')
      .then((r) => r.json())
      .catch((err) => {
        // Handle error manually
        setError(err.message);
      });
  }, []);

  if (error) {
    return <div>Error: {error}</div>;
  }

  return <div>Data loaded</div>;
}

// Or with Suspense for async boundaries (React 18+)
import { Suspense } from 'react';

function AsyncBoundary() {
  return (
    <ErrorBoundary fallback={<ErrorUI />}>
      <Suspense fallback={<LoadingUI />}>
        <AsyncComponent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## Third-Party Script Isolation

```javascript
// Isolate third-party ads, analytics, plugins in iframe
function SafeThirdPartyContent() {
  return (
    <iframe
      title="Third Party Content"
      srcDoc={`
        <script src="https://third-party.com/script.js"></script>
      `}
      sandbox="allow-scripts"
      style={{ border: 'none', width: '100%', height: '300px' }}
    />
  );
}

// Or lazy load with fallback
const ThirdPartyChat = lazy(() =>
  import('./ThirdPartyChat').catch(() => ({
    default: () => <div>Chat unavailable</div>,
  }))
);

function App() {
  return (
    <ErrorBoundary>
      <Suspense fallback={null}>
        <ThirdPartyChat />
      </Suspense>
    </ErrorBoundary>
  );
}
```

## Error Reporting Integration

```javascript
import * as Sentry from '@sentry/react';

// Initialize Sentry
Sentry.init({
  dsn: process.env.REACT_APP_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0,
});

class ErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // Automatically sent to Sentry
    Sentry.captureException(error, {
      contexts: {
        react: {
          componentStack: errorInfo.componentStack,
        },
      },
    });

    console.error(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback />;
    }

    return this.props.children;
  }
}

export default Sentry.withErrorBoundary(App, {
  fallback: <ErrorFallback />,
  showDialog: true,
});
```

## Key Points

- Error Boundaries catch component render errors
- They do NOT catch: async errors, event handlers, SSR
- Granular boundaries prevent full app crashes
- Always have fallback UI
- Integrate with error tracking services
- Use async error handling for data fetching
