## Protocol Comparison

### HTTP/1.1 (1997)

**Problem: Sequential requests on single connection**

```
Connection 1:
Request 1 ──────→ Response 1 ──────→
                  Request 2 ──────→ Response 2 ──────→
                                    Request 3 ──────→ Response 3
Timeline: R1(500ms) + R2(300ms) + R3(200ms) = 1000ms
```

**Workarounds:**
- Domain sharding (a.example.com, b.example.com)
- Inline CSS/JS/images
- Minification
- Connection pooling (creates overhead)

### HTTP/2 (2015)

**Solution: Multiplexing on single connection**

```
Connection 1:
Stream 1: Request 1 ─→
Stream 3:           Request 2 ─→
Stream 5:                        Request 3 ─→

All streams interleaved on single connection
Timeline: max(500ms, 300ms, 200ms) = 500ms
```

**Benefits:**
- Single TCP connection (1 handshake, 1 TLS negotiation)
- Header compression
- Server push
- Stream prioritization

**Issues:**
- TCP head-of-line blocking (packet loss blocks all streams)
- TLS renegotiation overhead for long connections

### HTTP/3 (2022)

**Improvement: UDP instead of TCP**

```
UDP (no guaranteed order):
- Packet 1 lost → Only Stream 1 blocked
- Streams 2, 3, 5 continue

TCP (all-or-nothing):
- Packet 1 lost → ALL streams blocked until retransmitted
```

**Key features:**
- QUIC protocol (UDP-based)
- Each stream has independent packet loss recovery
- Faster connection establishment (0-RTT resumption)
- Connection migration (switch networks seamlessly)

## Performance Impact

### Scenario: 10 Requests (100ms each)

| Protocol | Latency | Connections | TLS Handshakes | Notes |
|----------|---------|-------------|----------------|-------|
| HTTP/1.1 | 1000ms | 6 (limit) | 6 | Sequential + sharding |
| HTTP/2 | 100ms | 1 | 1 | Multiplexing on single connection |
| HTTP/3 | 50ms | 1 | 0 (0-RTT) | QUIC 0-RTT resumption |

### Real-World: Page Load

```
HTTP/1.1:
[HTML] → [CSS] → [JS] → [Image1] → [Image2] → [Font]
1000ms   500ms   400ms   300ms     250ms     200ms
Total: 2650ms

HTTP/2:
All requested in parallel on streams
Total: max(1000ms, 500ms, 400ms, 300ms, 250ms, 200ms) = 1000ms
Speedup: 2.65x

HTTP/3:
Same as HTTP/2 + 0-RTT handshake
Total: ~950ms
Speedup: 2.8x
```

## When to Use

| Protocol | Use Case |
|----------|----------|
| HTTP/1.1 | Legacy systems, simple APIs |
| HTTP/2 | Production web apps, APIs (most common) |
| HTTP/3 | Mobile apps, high-latency networks, competitive advantage |

## Browser Support

```javascript
// HTTP/2: 98% of modern browsers
// HTTP/3: 95% (modern Chrome, Safari, Firefox, Edge)

// Fallback chain:
// Try HTTP/3 → HTTP/2 → HTTP/1.1
```

## Nginx Configuration

```nginx
# Enable HTTP/2 and HTTP/3
server {
    listen 443 ssl http2;
    listen 443 ssl quic reuseport;  # HTTP/3

    server_name example.com;

    ssl_certificate /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;

    # Push critical resources
    http2_push /css/style.css;
    http2_push /js/app.js;

    location / {
        proxy_pass http://backend;
    }
}
```

## Node.js Performance

```javascript
// HTTP/1.1: ~5,000 req/sec
// HTTP/2: ~8,000 req/sec (1.6x faster)
// HTTP/3: ~10,000 req/sec (2x faster)

// With 100 concurrent connections:
// HTTP/1.1: Creates 100 sockets → memory intensive
// HTTP/2: Single socket with 100 streams → efficient
// HTTP/3: Single QUIC connection → most efficient
```
