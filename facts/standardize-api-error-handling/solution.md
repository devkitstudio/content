## RFC 7807 Problem Details Standard

Industry standard for HTTP error responses. Clients know exactly what to expect:

```json
{
  "type": "https://example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "The request body contains invalid data",
  "instance": "/api/users",
  "errors": [
    {
      "field": "email",
      "message": "Must be valid email format"
    }
  ]
}
```

**Why RFC 7807?**
- Standardized across industry
- Clients parse errors consistently
- Includes trace context (`instance`)
- Extensible with custom fields
- HTTP status code clarity

## Error Middleware Architecture

```typescript
// Categorize errors
class ValidationError extends Error {
  constructor(message: string, public details: any) {
    super(message);
    this.name = 'ValidationError';
  }
}

class AuthenticationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'AuthenticationError';
  }
}

// Map errors to RFC 7807
const errorTypeMap: Record<string, number> = {
  'ValidationError': 422,
  'AuthenticationError': 401,
  'AuthorizationError': 403,
  'NotFoundError': 404,
  'ConflictError': 409
};

// Express error handler
app.use((err: any, req: any, res: any, next: any) => {
  const status = errorTypeMap[err.name] || 500;

  const problemDetail = {
    type: `https://example.com/errors/${err.name}`,
    title: err.name,
    status,
    detail: err.message,
    instance: req.originalUrl,
    timestamp: new Date().toISOString(),
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
    ...(err.details && { errors: err.details })
  };

  res.status(status).json(problemDetail);
});
```

## Client-Friendly Responses

**Bad (inconsistent):**
```json
// Sometimes:
{ "error": "User not found" }

// Sometimes:
{ "message": "User not found" }

// Sometimes:
"User not found"
```

**Good (RFC 7807):**
```typescript
const response = {
  type: "https://api.example.com/errors/not-found",
  title: "Resource Not Found",
  status: 404,
  detail: "User with ID 123 does not exist",
  instance: "/api/users/123"
};
```

Clients can now:
```typescript
// Reliable error handling
if (error.status === 404) { /* retry or show not found */ }
if (error.status === 429) { /* backoff and retry */ }
if (error.status === 422) { /* show validation fields */ }
```

## Validation Error Details

```typescript
const validateUser = (data: any): ValidationError | null => {
  const errors = [];

  if (!data.email?.match(/@/)) {
    errors.push({ field: 'email', message: 'Invalid email format' });
  }
  if (!data.password || data.password.length < 8) {
    errors.push({ field: 'password', message: 'Must be at least 8 characters' });
  }

  if (errors.length > 0) {
    return new ValidationError('Validation failed', errors);
  }

  return null;
};

// Response:
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "Validation failed",
  "instance": "/api/users",
  "errors": [
    { "field": "email", "message": "Invalid email format" },
    { "field": "password", "message": "Must be at least 8 characters" }
  ]
}
```
