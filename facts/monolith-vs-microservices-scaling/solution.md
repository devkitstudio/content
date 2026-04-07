# Scaling Monoliths vs Microservices

**The honest answer:** Most monoliths scale vertically to 100M+ users. Microservices are about team scale, not user scale.

## Decision Framework

### Phase 1: Scale the Monolith (10K → 10M users)

**Vertical scaling works:**
- Database optimization (indexes, partitioning, read replicas)
- Caching (Redis, memcached)
- Load balancing (multiple monolith instances)
- Async jobs (background workers for heavy lifting)
- CDN for static assets

**Example monolith architecture:**
```
                    ┌─────────────────┐
                    │   Load Balancer │
                    └────────┬────────┘
                 ┌──────────┼──────────┐
            ┌────▼────┐ ┌─────▼────┐ ┌─▼─────┐
            │Monolith │ │Monolith  │ │Mono...│ (horizontal scaling)
            │Instance1│ │Instance2 │ │Inst3  │
            └────┬────┘ └────┬─────┘ └───┬───┘
                 └──────────┬─────────────┘
              ┌─────────────▼───────────────┐
              │   PostgreSQL (read replicas)│
              │   + Redis Cache Layer       │
              └─────────────────────────────┘
```

## Phase 2: Modular Monolith (When needed)

Before jumping to microservices, split your monolith into **logical modules** with clear boundaries:

```python
# Monolithic structure remains, but organized by domain
app/
  ├── payments/
  │   ├── models.py
  │   ├── services.py
  │   ├── api.py
  │   └── tests/
  ├── notifications/
  │   ├── models.py
  │   ├── services.py
  │   ├── api.py
  │   └── tests/
  ├── users/
  │   ├── models.py
  │   ├── services.py
  │   ├── api.py
  │   └── tests/
```

**Benefits:** Clear separation of concerns, easier testing, reduced merge conflicts.

## Phase 3: Microservices (Only if needed)

Extract microservices ONLY when:
- **Team scale:** 8+ engineers need independent deployment cycles
- **Domain reason:** Payment service needs PCI compliance isolated from others
- **Technology reason:** One service needs Java + spring-boot, another needs Python
- **Scaling reason:** Payments spike to 10x while users stay flat (rare)

```
┌─────────────────────────────────────────────────┐
│            API Gateway / Load Balancer          │
└────┬──────────┬──────────┬──────────┬───────────┘
     │          │          │          │
┌────▼──┐   ┌───▼───┐  ┌──▼───┐  ┌──▼────┐
│Payment│   │Notify │  │Users │  │Orders │  (independent services)
│Service│   │Service│  │Service   │Service│
└────┬──┘   └───┬───┘  └──┬───┘  └──┬────┘
     │          │         │         │
┌────▼──────────▼─────────▼─────────▼────┐
│  Event Bus (Kafka/RabbitMQ) - async     │
└─────────────────────────────────────────┘
     ▲
     │ (database per service)
┌────┴──────┬──────────┬──────────┐
│ Payments  │Notif DB  │Users DB  │ Orders DB
│   DB      │          │          │
```

## Comparison: What breaks first?

| Bottleneck | Monolith Fix | Effort |
|-----------|-----------|--------|
| Database CPU | Read replicas + caching | 2 weeks |
| Memory per process | Load balance more instances | 1 week |
| Deployment speed | Modular monolith + fast tests | 3 weeks |
| Team coordination | Feature flags + clear ownership | 2 weeks |
| One service slow | Async jobs + worker processes | 2 weeks |

## Red Flags for "We need microservices"

- "We have 30K users and need microservices"
- "We want to use Kubernetes from day one"
- "Each team should own a service" (but teams are <5 people)

**Real reasons for microservices:**
- Payments team needs PCI scope isolation (regulatory)
- Video processing needs Go, rest is Python (tech diversity)
- Payments load: 1K req/s, Orders: 100 req/s (extreme skew)
- Company has 50+ engineers across 10+ teams

## Recommendation

```
Users:        Scale monolith with:
10K - 100K    → Vertical scaling, caching
100K - 1M     → Horizontal monolith, async workers
1M - 10M      → Modular monolith, read replicas, CDN
10M+          → If still monolith: feature flags, internal services
              → If split: only because of TEAM/REGULATORY reasons
```

**Most companies at 1M users still run a core monolith** + specialized services (search, analytics, media processing).
