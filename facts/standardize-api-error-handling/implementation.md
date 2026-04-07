## Complete Express Error Middleware

```typescript
import express from 'express';

// Custom error classes
export class ApiError extends Error {
  constructor(
    public statusCode: number,
    public type: string,
    message: string,
    public details?: any
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export class ValidationError extends ApiError {
  constructor(message: string, details: any) {
    super(422, 'https://api.example.com/errors/validation-error', message, details);
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends ApiError {
  constructor(message: string) {
    super(404, 'https://api.example.com/errors/not-found', message);
    this.name = 'NotFoundError';
  }
}

// Error handler middleware
export const errorHandler = (
  err: any,
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
) => {
  const statusCode = err.statusCode || 500;
  const type = err.type || 'https://api.example.com/errors/internal-server-error';

  const problemDetail = {
    type,
    title: err.name || 'Internal Server Error',
    status: statusCode,
    detail: err.message || 'An unexpected error occurred',
    instance: req.originalUrl,
    timestamp: new Date().toISOString(),
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
    ...(err.details && { errors: err.details })
  };

  res.status(statusCode).json(problemDetail);
};

// Usage in routes
const app = express();

app.post('/api/users', async (req, res, next) => {
  try {
    const { email, password } = req.body;

    // Validation
    if (!email || !password) {
      throw new ValidationError('Missing required fields', [
        { field: 'email', message: email ? null : 'Required' },
        { field: 'password', message: password ? null : 'Required' }
      ].filter(e => e.message));
    }

    // Find user
    const user = await db.users.findByEmail(email);
    if (!user) {
      throw new NotFoundError(`User with email ${email} not found`);
    }

    res.json(user);
  } catch (err) {
    next(err);
  }
});

// Attach error handler
app.use(errorHandler);

export default app;
```

## Async Route Wrapper

Avoid repetitive try/catch:

```typescript
const asyncHandler = (fn: any) => (req: any, res: any, next: any) =>
  Promise.resolve(fn(req, res, next)).catch(next);

// Before: Verbose
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await db.users.findById(req.params.id);
    if (!user) throw new NotFoundError('User not found');
    res.json(user);
  } catch (err) {
    next(err);
  }
});

// After: Clean
app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await db.users.findById(req.params.id);
  if (!user) throw new NotFoundError('User not found');
  res.json(user);
}));
```

## Response Format Consistency

```typescript
// Middleware to ensure all responses follow a pattern
app.use((req, res, next) => {
  const originalJson = res.json.bind(res);

  res.json = function(data: any) {
    // Errors handled by error middleware
    if (res.statusCode >= 400) {
      return originalJson(data);
    }

    // Success responses
    return originalJson({
      data,
      status: 'success',
      timestamp: new Date().toISOString()
    });
  };

  next();
});

// Client expects:
// Success: { data: {...}, status: 'success', timestamp: '...' }
// Error: { type: '...', title: '...', status: 400, ... }
```
