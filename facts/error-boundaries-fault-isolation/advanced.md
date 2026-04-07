# Granular Boundaries & Recovery Strategies

## Hierarchical Error Boundaries

```javascript
// App level: Catch critical errors
class RootErrorBoundary extends React.Component {
  render() {
    if (this.state.hasError) {
      return <FullPageError />;
    }
    return this.props.children;
  }
}

// Feature level: Isolate feature failures
class FeatureErrorBoundary extends React.Component {
  render() {
    if (this.state.hasError) {
      return <FeaturePlaceholder />;
    }
    return this.props.children;
  }
}

// Widget level: Minimal UI fallback
class WidgetErrorBoundary extends React.Component {
  render() {
    if (this.state.hasError) {
      return <WidgetSkeletonLoader />;
    }
    return this.props.children;
  }
}

// Usage
function App() {
  return (
    <RootErrorBoundary>
      <Header />
      <FeatureErrorBoundary>
        <Dashboard>
          <WidgetErrorBoundary>
            <Analytics />
          </WidgetErrorBoundary>
          <WidgetErrorBoundary>
            <Charts />
          </WidgetErrorBoundary>
        </Dashboard>
      </FeatureErrorBoundary>
      <Footer />
    </RootErrorBoundary>
  );
}
```

## Recovery with Exponential Backoff

```javascript
class ErrorBoundaryWithRetry extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      hasError: false,
      retryCount: 0,
      retryDelay: 1000,
    };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error:', error);
    this.setState({ hasError: true });
  }

  handleRetry = async () => {
    const { retryCount, retryDelay } = this.state;

    // Wait before retrying (exponential backoff: 1s, 2s, 4s, 8s)
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay * Math.pow(2, retryCount))
    );

    this.setState({
      hasError: false,
      retryCount: retryCount + 1,
      retryDelay: Math.min(retryDelay * 2, 30000), // Cap at 30s
    });
  };

  render() {
    const { hasError, retryCount } = this.state;

    if (hasError) {
      return (
        <div className="error-container">
          <h2>Something went wrong</h2>
          <p>Attempt {retryCount}</p>
          <button onClick={this.handleRetry}>
            Retry
          </button>
          {retryCount > 5 && (
            <p>Too many retries. Contact support.</p>
          )}
        </div>
      );
    }

    return this.props.children;
  }
}
```

## Error Boundary with Analytics

```javascript
class AnalyticsErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // Track error metrics
    const errorData = {
      message: error.toString(),
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent,
      url: window.location.href,
      userId: getCurrentUserId(),
    };

    // Send to analytics
    window.analytics?.track('APP_ERROR', errorData);

    // Send to error tracking
    fetch('/api/errors', {
      method: 'POST',
      body: JSON.stringify(errorData),
    });

    // Alert team if critical
    if (error.message.includes('CRITICAL')) {
      fetch('/api/alert-team', {
        method: 'POST',
        body: JSON.stringify({ error: errorData }),
      });
    }
  }

  render() {
    if (this.state.hasError) {
      return <ErrorUI />;
    }
    return this.props.children;
  }
}
```

## Error Boundary with State Reset

```javascript
class ErrorBoundaryWithReset extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  componentDidCatch(error, errorInfo) {
    console.error(error);
    this.setState({ hasError: true });

    // Clear potentially corrupted state
    sessionStorage.clear();
    localStorage.removeItem('tempData');
  }

  handleReset = () => {
    this.setState({ hasError: false });
    // Optionally reload critical data
    this.props.onReset?.();
  };

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>App needs to restart</h2>
          <button onClick={this.handleReset}>Reset App</button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

## Error Isolation with Suspense

```javascript
function SafeAsyncBoundary({ loader: Loader }) {
  return (
    <ErrorBoundary
      fallback={<LoaderSkeletonError />}
      onError={(error) => console.error('Async error:', error)}
    >
      <Suspense fallback={<LoaderSkeleton />}>
        <Loader />
      </Suspense>
    </ErrorBoundary>
  );
}

// Use with any async component
function Dashboard() {
  return (
    <div>
      <SafeAsyncBoundary loader={AsyncAnalytics} />
      <SafeAsyncBoundary loader={AsyncReports} />
      <SafeAsyncBoundary loader={AsyncMetrics} />
    </div>
  );
}
```

## Error Boundary for Third-Party Components

```javascript
class ThirdPartyBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // Special handling for third-party errors
    if (error.message.includes('THIRD_PARTY')) {
      // Disable feature gracefully
      this.props.onThirdPartyError?.();
    }

    super.componentDidCatch(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="fallback">
          <p>Feature temporarily unavailable</p>
          <a href="/settings">View alternatives</a>
        </div>
      );
    }

    return this.props.children;
  }
}

// Usage with stripe, maps, chat widgets, etc.
function CheckoutFlow() {
  return (
    <ThirdPartyBoundary
      onThirdPartyError={() => alert('Stripe connection lost')}
    >
      <StripePaymentForm />
    </ThirdPartyBoundary>
  );
}
```

## Event Handler Error Catching

```javascript
// Error Boundaries can't catch event handler errors
// Use try-catch instead

function Form() {
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await submitForm();
      setError(null);
    } catch (err) {
      setError(err.message);
      // Log to error tracking
      logError(err);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" />
      {error && <div className="error">{error}</div>}
      <button>Submit</button>
    </form>
  );
}

// Or use error boundary HOC
function withErrorHandler(Component) {
  return class extends React.Component {
    constructor(props) {
      super(props);
      this.state = { error: null };
    }

    handleError = (error) => {
      this.setState({ error });
    };

    render() {
      if (this.state.error) {
        return <ErrorUI error={this.state.error} />;
      }

      return <Component {...this.props} onError={this.handleError} />;
    }
  };
}
```

## Error Boundary Best Practices Checklist

- Granular boundaries (App → Feature → Widget)
- Always provide fallback UI
- Log errors to tracking service (Sentry, LogRocket)
- Reset state on recovery
- Exponential backoff for retries
- Don't catch non-render errors (use try-catch)
- Special handling for third-party components
- Monitor error rate in analytics
- Test error paths in development
