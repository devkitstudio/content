## Execution: Dynamic CORS Configuration

**Architectural Rule:** Never hardcode allowed origins or cache times in application code. These parameters must be injected via environment variables or infrastructure templates to maintain environment parity (Dev, Staging, Prod).

### 1. Application-Level: Express.js (Node.js)

```typescript
import cors from "cors";
import express from "express";

const app = express();

// INJECTED CONFIGURATION: Read from environment, fallback to secure defaults
const allowedOrigins = (process.env.CORS_ALLOWED_ORIGINS || "").split(",");
const maxAge = parseInt(process.env.CORS_MAX_AGE || "86400", 10);

const corsOptions: cors.CorsOptions = {
  origin: (origin, callback) => {
    // Allow requests with no origin (like mobile apps or curl requests)
    // AND check against the strict environment allowlist
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error("Not allowed by CORS Policy"));
    }
  },
  credentials: true,
  methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  allowedHeaders: ["Content-Type", "Authorization", "X-Idempotency-Key"],
  maxAge: maxAge, // Cache preflight
};

app.use(cors(corsOptions));
```

### 2. Infrastructure-Level: Nginx Reverse Proxy

Enforcing CORS at the edge (API Gateway / Reverse Proxy) centralizes security and prevents application-level misconfigurations.

```nginx
# INJECTED CONFIGURATION: Maps are safer than complex 'if' statements
map $http_origin $cors_allowed_origin {
    default "";
    "~^https://app\.mycompany\.com$" $http_origin;
    "~^https://admin\.mycompany\.com$" $http_origin;
    # DO NOT use broad regex like ".*\.mycompany\.com" (See Pitfalls)
}

server {
    location /api/ {
        if ($cors_allowed_origin != "") {
            add_header 'Access-Control-Allow-Origin' $cors_allowed_origin always;
            add_header 'Access-Control-Allow-Credentials' 'true' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
            add_header 'Access-Control-Max-Age' '86400' always;
        }

        # Intercept and answer preflight requests immediately at the edge
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' $cors_allowed_origin always;
            add_header 'Access-Control-Allow-Credentials' 'true' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        proxy_pass http://backend_upstream;
    }
}
```
