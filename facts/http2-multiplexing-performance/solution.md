## HTTP/1.1 Problem: Head-of-Line Blocking

```
Single TCP Connection (HTTP/1.1 keep-alive)

Request 1: GET /api/users (500ms)  ━━━━━━━━━━━━━━━━━━━━━━
Request 2: GET /api/posts          (waits...) ━━━━━━━━━
Request 3: GET /api/comments       (waits...)          ━━

Timeline: 500ms + 300ms + 200ms = 1000ms total
```

**Solution: Multiple TCP Connections**

```
Connection 1: GET /api/users (500ms) ━━━━━━━━━━━━━━━━━━━━
Connection 2: GET /api/posts (300ms) ━━━━━━━━
Connection 3: GET /api/comments (200ms) ━━

Timeline: max(500ms, 300ms, 200ms) = 500ms total
```

**Problem:** Creating new connections is expensive (TCP handshake, TLS negotiation)

## HTTP/2 Solution: Multiplexing

```
Single TCP Connection (HTTP/2)

Stream 1: GET /api/users
Stream 3: GET /api/posts       ← interleaved!
Stream 5: GET /api/comments    ← no blocking!

Both streams can send/receive simultaneously
Timeline: ~500ms (fastest request wins)
```

### How Multiplexing Works

HTTP/2 breaks requests/responses into **frames** on a single connection:

```
TCP Connection
├── Stream 1 (Request 1)
│   ├── Headers Frame: GET /api/users
│   ├── (server processing)
│   └── Data Frame: [response body] [response body]
├── Stream 3 (Request 2)
│   ├── Headers Frame: GET /api/posts
│   └── Data Frame: [partial response]
└── Stream 5 (Request 3)
    ├── Headers Frame: GET /api/comments
    └── (interleaved with other streams)
```

**Key difference:** Requests and responses don't block each other.

## Real-World Example

```javascript
// HTTP/1.1: Creates 6 TCP connections (connection pooling limit)
const http1Results = await Promise.all([
  fetch('/api/users'),      // Connection 1: 500ms
  fetch('/api/posts'),      // Connection 2: 300ms
  fetch('/api/comments'),   // Connection 3: 200ms
  fetch('/api/analytics'),  // Connection 4: 400ms
  fetch('/api/settings'),   // Connection 5: 150ms
  fetch('/api/notifications') // Connection 6: 100ms
]);
// Total: 500ms (all in parallel due to 6 connections)
// But: 6 TCP handshakes, 6 TLS negotiations, memory overhead

// HTTP/2: Single TCP connection with streams
// Same code, but server responds on single connection
// Total: 500ms (fastest request)
// But: 1 TCP handshake, 1 TLS negotiation, lower memory
```

## Server-Side: HTTP/2 Push

```javascript
// Express with spdy (HTTP/2)
const spdy = require('spdy');
const fs = require('fs');

const options = {
  key: fs.readFileSync('./server.key'),
  cert: fs.readFileSync('./server.cert'),
};

app.get('/', (req, res) => {
  // Push critical resources before client asks
  res.push('/css/style.css', {
    status: 200,
    method: 'GET',
    request: {
      accept: '*/*'
    },
    response: {
      'content-type': 'text/css'
    }
  }, (err, pushRes) => {
    if (err) return;
    pushRes.end(fs.readFileSync('./style.css'));
  });

  // Send HTML
  res.end('<html>...');
});

spdy.createSecureServer(options, app).listen(3000);
```

**Push eliminates round-trips:**
```
HTTP/1.1:
GET /index.html → Response: <link rel="stylesheet" href="style.css">
GET /style.css → Response: CSS content

HTTP/2 Push:
GET /index.html → Server proactively sends style.css (stream 3)
                  Response: HTML + CSS inline in push
```

## Performance Comparison

| Metric | HTTP/1.1 | HTTP/2 | Improvement |
|--------|----------|--------|------------|
| 6 requests (60ms each) | 360ms | 60ms | 6x faster |
| Connection setup | 6 × TLS | 1 × TLS | 6x faster |
| Memory per connection | Medium | Low | 6x less |
| Large response blocking | Yes | No | N/A |
| Request priority | FIFO | Weighted streams | Better |

## Node.js HTTP/2 Server

```javascript
const http2 = require('http2');
const fs = require('fs');

const options = {
  key: fs.readFileSync('./server.key'),
  cert: fs.readFileSync('./server.cert'),
};

const server = http2.createSecureServer(options, (req, res) => {
  res.setHeader('content-type', 'application/json');

  switch (req.url) {
    case '/api/users':
      // Simulate slow endpoint
      setTimeout(() => {
        res.end(JSON.stringify([{ id: 1, name: 'Alice' }]));
      }, 500);
      break;

    case '/api/posts':
      res.end(JSON.stringify([{ id: 1, title: 'Hello' }]));
      break;

    default:
      res.statusCode = 404;
      res.end('Not found');
  }
});

server.listen(3000);
// Single connection can handle multiple streams simultaneously
```

## Browser Compatibility

```javascript
// Modern browsers support HTTP/2 automatically
// Just use https:// and serve from HTTP/2 server

// Detect HTTP/2
const protocol = performance.getEntriesByType('navigation')[0].nextHopProtocol;
console.log(protocol); // "h2" for HTTP/2, "http/1.1" for HTTP/1.1
```
