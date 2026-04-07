## Round-Robin Problem

```
3 servers: A (fast), B (slow), C (medium)

Requests:  1    2    3    4    5    6
Route to:  A    B    C    A    B    C

Timeline:
A: ─ ─ ─ → completes (20ms each)
B: ─────── ─────── → completes (400ms each, slow database)
C: ─── ─── → completes (100ms each)

Server B is 20x slower but gets same load!
```

**Why?** Round-robin ignores **actual response time**.

## Better Algorithms

### 1. Least Connections

Routes to server with fewest active connections.

```python
class LoadBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.active_connections = {s: 0 for s in servers}

    def route(self, request):
        least_busy = min(
            self.servers,
            key=lambda s: self.active_connections[s]
        )
        self.active_connections[least_busy] += 1

        try:
            return least_busy.handle(request)
        finally:
            self.active_connections[least_busy] -= 1

# Works better for mixed request durations
```

**Problem:** Still doesn't account for actual load.

### 2. Weighted Round-Robin

```python
class WeightedLoadBalancer:
    def __init__(self, servers):
        # weights = { server: capacity }
        # Higher weight = more powerful
        self.servers = servers
        self.weights = {
            'server-a-fast': 10,
            'server-b-slow': 2,
            'server-c-medium': 5
        }
        self.current = 0

    def route(self):
        # Request distribution:
        # A: 10/(10+2+5) = 59%
        # B: 2/(10+2+5) = 12%
        # C: 5/(10+2+5) = 29%

        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server
```

**Better:** Manual tuning required.

### 3. Least Response Time (Adaptive)

```python
class AdaptiveLoadBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.response_times = {s: [] for s in servers}
        self.window_size = 100

    def route(self):
        avg_times = {}
        for server in self.servers:
            times = self.response_times[server][-self.window_size:]
            avg_times[server] = sum(times) / len(times) if times else 0

        # Route to server with fastest avg response time
        best_server = min(self.servers, key=lambda s: avg_times[s])
        return best_server

    def record_time(self, server, response_time):
        self.response_times[server].append(response_time)

# Example:
lb = AdaptiveLoadBalancer(['A', 'B', 'C'])

# Over time, learns:
# A: avg 20ms → gets 60%
# B: avg 400ms → gets 5%
# C: avg 100ms → gets 35%
```

**Best for:** Dynamic conditions

### 4. Consistent Hashing

```python
import hashlib

class ConsistentHashLB:
    def __init__(self, servers):
        self.servers = servers
        self.ring = {}
        self._build_ring()

    def _build_ring(self):
        for server in self.servers:
            for replica in range(160):  # Virtual nodes
                key = f"{server}:{replica}"
                hash_val = int(hashlib.md5(key.encode()).hexdigest(), 16)
                self.ring[hash_val] = server

    def route(self, user_id):
        if not self.ring:
            return None

        hash_val = int(hashlib.md5(str(user_id).encode()).hexdigest(), 16)

        # Find smallest hash >= user_id's hash (circular)
        sorted_hashes = sorted(self.ring.keys())
        for h in sorted_hashes:
            if h >= hash_val:
                return self.ring[h]

        # Wrap around
        return self.ring[sorted_hashes[0]]

    def add_server(self, server):
        """Add server without disrupting existing routing"""
        self.servers.append(server)
        self._build_ring()

# Benefits:
# - User 1 always routes to same server
# - Adding/removing servers only affects ~1/n of requests
# - Perfect for distributed caches, sessions
```

**Best for:** Caching, session persistence, distributed systems

### 5. IP Hash

```python
class IPHashLB:
    def __init__(self, servers):
        self.servers = servers

    def route(self, client_ip):
        hash_val = hash(client_ip) % len(self.servers)
        return self.servers[hash_val]

# Same client always routes to same server
# Useful for session persistence without sticky sessions
```

## Nginx Configuration

```nginx
upstream backend {
    # Round-robin (default)
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}

upstream backend_weighted {
    # Weighted round-robin
    server server1.example.com weight=5;  # 5/(5+2+3) = 50%
    server server2.example.com weight=2;  # 20%
    server server3.example.com weight=3;  # 30%
}

upstream backend_leastconn {
    # Least connections
    least_conn;
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}

upstream backend_hash {
    # Consistent hash by IP
    hash $remote_addr consistent;
    server server1.example.com;
    server server2.example.com;
    server server3.example.com;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend_leastconn;
    }
}
```

## Performance Comparison

| Algorithm | Scenario | Effectiveness |
|-----------|----------|---------------|
| Round-robin | Equal servers | 10/10 |
| Round-robin | Mixed speeds | 3/10 |
| Least Connections | Long requests | 8/10 |
| Weighted | Stable load | 9/10 |
| Adaptive | Dynamic | 10/10 |
| Consistent Hash | Session stickiness | 10/10 |
| IP Hash | Session stickiness | 9/10 |
