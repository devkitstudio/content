| Solution | RTO | RPO | Cost | Setup | Failover |
|----------|-----|-----|------|-------|----------|
| **Patroni + etcd** | 10-30s | <1s | Medium | Complex | Auto |
| **RDS Multi-AZ** | 2 min | <1s | High | Easy | Auto |
| **CockroachDB** | <10s | 0s | High | Complex | Auto |
| **Citus (PostgreSQL)** | 10-30s | <1s | Medium | Medium | Auto |
| **Manual + DNS** | 10+ min | 30s+ | Low | Easy | Manual |

## Patroni (Open Source HA)

**Pros:**
- Free, open source
- True PostgreSQL (not fork)
- Fine-grained control
- Can use any storage (local, EBS, etc)

**Cons:**
- Requires etcd/Consul cluster
- More operational overhead
- Must monitor yourself
- Complex troubleshooting

**Cost:** ~$0/month (self-managed)

## RDS Multi-AZ (AWS)

**Pros:**
- Completely managed
- Automatic backups + PITR
- AWS integrations
- Professional support

**Cons:**
- Expensive (~2x single instance)
- Limited to AWS
- Less control over parameters
- Vendor lock-in

**Cost:** ~$4,000-20,000/month (db.r6i.xlarge Multi-AZ)

## CockroachDB (Distributed)

**Pros:**
- Built-in replication (3+ nodes)
- No single point of failure
- ACID + distributed
- Geo-distributed failover

**Cons:**
- Not PostgreSQL (compatibility layer)
- Higher cost
- More complex querying
- Geo-latency considerations

**Cost:** ~$5,000+/month (managed service)

## Recommendation by Scale

```
Startup/Small (< 1M req/day):
└─ Single primary + RDS backup
   └─ Acceptable downtime, fast recovery

Growing (1M-10M req/day):
└─ Patroni + 1 streaming replica
   └─ ~10s RTO, zero data loss

Scale (> 10M req/day):
├─ Patroni + 2+ replicas + WAL archiving
├─ Or RDS Multi-AZ
└─ Or CockroachDB (if distributed needed)
```

## Production Setup (Recommended)

```
Primary (us-east-1a)
├─ Patroni + PostgreSQL + etcd
│
Replica 1 (us-east-1b)
├─ Patroni + PostgreSQL (streaming replication)
│
Replica 2 (us-west-1)
├─ Patroni + PostgreSQL (streaming replication)
│
Monitoring:
├─ Prometheus (metrics)
├─ AlertManager (alerts)
└─ Grafana (dashboards)

Application:
├─ Connects to db.internal (VIP)
├─ VIP maintained by keepalived
└─ Automatic reconnect on failover
```

## Feature Comparison Table

| Feature | Patroni | RDS Multi-AZ | CockroachDB |
|---------|---------|-------------|-----------|
| Automatic failover | Yes | Yes | Yes |
| Data loss | <1s | <1s | 0s |
| Maintenance windows | You | AWS | You |
| Point-in-time recovery | Yes | Yes (35 days) | Yes |
| Multi-region | Yes | Yes (RDS Global) | Yes (native) |
| PostgreSQL compatibility | 100% | 99% | 70% |
| Horizontal scaling | No | Read replicas | Yes |
| Cost visibility | Transparent | Opaque | Transparent |
