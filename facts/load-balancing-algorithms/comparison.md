## Algorithm Comparison

| Algorithm | Latency | Fairness | Sticky Sessions | Complexity |
|-----------|---------|----------|-----------------|-----------|
| Round-robin | Medium | Poor | No | Low |
| Weighted | Good | Good | No | Low |
| Least Conn | Good | Good | No | Medium |
| Consistent Hash | Good | Good | Yes | Medium |
| Adaptive | Excellent | Excellent | No | High |
| IP Hash | Good | Fair | Yes | Low |

## Decision Tree

```
Do you need session persistence?
├─ Yes (shopping cart, auth tokens)
│  ├─ Use Consistent Hashing
│  └─ Or IP-based Hashing
│
└─ No
   Do you know server capacities?
   ├─ Yes (stable infrastructure)
   │  ├─ Use Weighted Round-robin
   │  └─ Or Least Connections
   │
   └─ No (dynamic/unknown)
      ├─ Use Least Connections (real-time)
      └─ Or Adaptive (learns over time)
```

## Real-World Example

```
Scenario: E-commerce checkout (200 req/s, mixed request times)

Round-robin:
- Server A (fast, 20ms): 67 req/s, avg 20ms → 1.3s total
- Server B (slow, 500ms): 67 req/s, avg 500ms → 33s total
- Server C (medium, 100ms): 66 req/s, avg 100ms → 6.6s total
Problem: B is overloaded, A is idle

Weighted Round-robin (weight by specs):
- Server A (8 cores): 100 req/s, avg 20ms → OK
- Server B (2 cores): 34 req/s, avg 500ms → 17s total
- Server C (4 cores): 66 req/s, avg 100ms → 6.6s total
Better: Closer to balanced

Least Connections:
- Server A (20ms): 150 req/s active
- Server B (500ms): 30 req/s active
- Server C (100ms): 80 req/s active
LB routes to C (80 < 150)
Result: All servers get optimal load
```

## Implementation Complexity

**Round-robin:** 5 lines
```python
index = (index + 1) % len(servers)
return servers[index]
```

**Weighted:** 10 lines
```python
weighted_servers = []
for server, weight in weights.items():
    weighted_servers.extend([server] * weight)
return random.choice(weighted_servers)
```

**Least Connections:** 15 lines
```python
min_conns = min(servers, key=lambda s: connections[s])
connections[min_conns] += 1
```

**Consistent Hashing:** 50+ lines (requires ring structure)

**Adaptive:** 100+ lines (metrics collection, moving averages)
