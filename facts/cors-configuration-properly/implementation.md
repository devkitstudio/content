# CORS Implementation Examples

## Express.js Configuration

```typescript
import cors from 'cors';
import express from 'express';

const app = express();

// Option 1: Simple allowlist
const allowedOrigins = [
  'https://example.com',
  'https://app.example.com',
  'http://localhost:3000',  // Development
  'http://localhost:3001'   // Development alt port
];

const corsOptions: cors.CorsOptions = {
  origin: allowedOrigins,
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-API-Key'],
  maxAge: 86400,  // 24 hours
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));

// Option 2: Dynamic origin validation
const corsOptionsWithDynamicOrigin: cors.CorsOptions = {
  origin: (origin, callback) => {
    // origin is undefined for same-origin requests
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST'],
  maxAge: 3600
};

app.options('/api/*', cors(corsOptionsWithDynamicOrigin));
app.get('/api/data', cors(corsOptionsWithDynamicOrigin), (req, res) => {
  res.json({ data: 'sensitive' });
});

// Option 3: Environment-based configuration
const getCorsConfig = () => {
  const env = process.env.NODE_ENV || 'development';

  const configs = {
    development: {
      origin: ['http://localhost:3000', 'http://localhost:3001'],
      credentials: true
    },
    production: {
      origin: ['https://example.com', 'https://app.example.com'],
      credentials: true
    },
    staging: {
      origin: ['https://staging.example.com', 'http://localhost:3000'],
      credentials: true
    }
  };

  return configs[env] || configs.production;
};

app.use(cors(getCorsConfig()));
```

## Nginx Configuration

```nginx
# Nginx CORS configuration
upstream api_backend {
  server localhost:3000;
}

map $http_origin $allow_origin {
  default "";
  "~^https://example\.com$" "https://example.com";
  "~^https://app\.example\.com$" "https://app.example.com";
  "~^http://localhost:300[01]$" $http_origin;
}

server {
  listen 80;
  server_name api.example.com;

  location /api {
    # Only set header if origin is allowed
    if ($allow_origin != "") {
      add_header 'Access-Control-Allow-Origin' $allow_origin;
      add_header 'Access-Control-Allow-Credentials' 'true';
      add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS';
      add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization, X-API-Key';
      add_header 'Access-Control-Max-Age' '86400';
    }

    # Handle preflight requests
    if ($request_method = 'OPTIONS') {
      return 204;
    }

    proxy_pass http://api_backend;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

## Cloudflare Workers

```typescript
// Cloudflare Worker CORS handling
const ALLOWED_ORIGINS = [
  'https://example.com',
  'https://app.example.com'
];

export default {
  async fetch(request: Request): Promise<Response> {
    const origin = request.headers.get('Origin');

    // Check if origin is allowed
    const isAllowed = ALLOWED_ORIGINS.includes(origin || '');
    const allowOrigin = isAllowed ? origin : null;

    // Handle preflight requests
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        status: 204,
        headers: {
          'Access-Control-Allow-Origin': allowOrigin || '',
          'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type, Authorization',
          'Access-Control-Max-Age': '86400',
          'Access-Control-Allow-Credentials': 'true'
        }
      });
    }

    // Forward request to origin
    const response = await fetch(request);

    // Add CORS headers to response
    const headers = new Headers(response.headers);
    if (allowOrigin) {
      headers.set('Access-Control-Allow-Origin', allowOrigin);
      headers.set('Access-Control-Allow-Credentials', 'true');
    }

    return new Response(response.body, {
      status: response.status,
      headers
    });
  }
};
```

## AWS API Gateway

```yaml
# CloudFormation template for API Gateway CORS
Resources:
  MyApiStage:
    Type: AWS::ApiGateway::Stage
    Properties:
      StageName: prod
      RestApiId: !Ref MyApi
      DeploymentId: !Ref MyDeployment
      MethodSettings:
        - ResourcePath: '/*'
          HttpMethod: '*'
          LoggingLevel: INFO
          DataTraceEnabled: true

  # Gateway Response for CORS
  ApiGatewayCorsResponse:
    Type: AWS::ApiGateway::GatewayResponse
    Properties:
      ResponseParameters:
        gatewayresponse.header.Access-Control-Allow-Origin: "'https://example.com'"
        gatewayresponse.header.Access-Control-Allow-Headers: "'Content-Type,Authorization'"
        gatewayresponse.header.Access-Control-Allow-Methods: "'GET,POST,PUT,DELETE,OPTIONS'"
        gatewayresponse.header.Access-Control-Max-Age: "'86400'"
      ResponseType: CORS_DEFAULT_4XX
      RestApiId: !Ref MyApi
```

## TypeScript/Node.js with Middleware

```typescript
import express, { Request, Response, NextFunction } from 'express';

// Custom CORS middleware
class CORSMiddleware {
  private allowedOrigins: Set<string>;
  private allowedMethods: string[];
  private allowedHeaders: string[];
  private maxAge: number;

  constructor(origins: string[]) {
    this.allowedOrigins = new Set(origins);
    this.allowedMethods = ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'];
    this.allowedHeaders = ['Content-Type', 'Authorization', 'X-API-Key'];
    this.maxAge = 86400;
  }

  middleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const origin = req.get('origin');

      // Set CORS headers if origin is allowed
      if (origin && this.allowedOrigins.has(origin)) {
        res.set('Access-Control-Allow-Origin', origin);
        res.set('Access-Control-Allow-Credentials', 'true');
        res.set(
          'Access-Control-Allow-Methods',
          this.allowedMethods.join(', ')
        );
        res.set(
          'Access-Control-Allow-Headers',
          this.allowedHeaders.join(', ')
        );
        res.set('Access-Control-Max-Age', String(this.maxAge));
      }

      // Handle preflight requests
      if (req.method === 'OPTIONS') {
        return res.send(200);
      }

      next();
    };
  }

  // Dynamic origin checker (e.g., regex patterns)
  isDomainAllowed(origin: string): boolean {
    // Exact match
    if (this.allowedOrigins.has(origin)) {
      return true;
    }

    // Pattern match for subdomains
    const regex = /^https:\/\/[\w-]+\.example\.com$/;
    return regex.test(origin);
  }
}

// Usage
const app = express();
const corsMiddleware = new CORSMiddleware([
  'https://example.com',
  'https://app.example.com',
  'http://localhost:3000'
]);

app.use(corsMiddleware.middleware());

app.get('/api/protected', (req, res) => {
  res.json({ data: 'protected data' });
});
```

## React Frontend Integration

```typescript
// Frontend: Proper credential handling
const fetchWithCORS = async (url: string, options?: RequestInit) => {
  try {
    const response = await fetch(url, {
      ...options,
      // Critical: Include credentials with CORS
      credentials: 'include',  // Sends cookies
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers
      }
    });

    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    console.error('Request failed:', error);
    throw error;
  }
};

// Usage
const getUser = async (id: string) => {
  return fetchWithCORS(`https://api.example.com/users/${id}`);
};

const updateUser = async (id: string, data: Record<string, any>) => {
  return fetchWithCORS(`https://api.example.com/users/${id}`, {
    method: 'PUT',
    body: JSON.stringify(data)
  });
};
```

## Testing CORS Configuration

```bash
# Test CORS headers with curl
curl -H "Origin: https://example.com" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: Content-Type" \
     -X OPTIONS \
     https://api.example.com/data \
     -v

# Expected response headers:
# Access-Control-Allow-Origin: https://example.com
# Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
# Access-Control-Allow-Headers: Content-Type
# Access-Control-Max-Age: 86400

# Test with credentials
curl -H "Origin: https://example.com" \
     -H "Cookie: sessionid=abc123" \
     https://api.example.com/auth/profile \
     -v
```

## Common Mistakes to Avoid

```typescript
// WRONG: Dynamic origin without validation
app.use(cors({
  origin: (origin, callback) => {
    callback(null, true);  // Accepts ANY origin!
  }
}));

// WRONG: Hardcoded wildcard
app.use(cors({
  origin: '*',
  credentials: true  // Contradictory: can't use * with credentials
}));

// WRONG: Over-permissive headers
app.use(cors({
  allowedHeaders: '*'  // Allows Authorization header from any origin
}));

// CORRECT: Explicit allowlist
const corsOptions = {
  origin: ['https://example.com', 'https://app.example.com'],
  credentials: true,
  methods: ['GET', 'POST'],
  allowedHeaders: ['Content-Type']
};
app.use(cors(corsOptions));
```
