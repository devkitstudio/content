## Deployment Strategies

**The Anti-Pattern (All-or-Nothing):**

```text
0% ──────────────────▶ 100%
     │
     └─ Breaks for 100% of users instantly.
```

Result: Global outage. MTTR (Mean Time To Recovery) is bound by the time it takes to spin up the old infrastructure and update DNS/Routing.

**The Standard (Canary Rollout):**

```text
0% ──▶ Initial Subset ──▶ Expanded Subset ──▶ 100%
     │                │                 │
     ✓                ✓                 ✓
```

Result: Gradual risk expansion. Automatic rollback at the network tier (instant) if telemetry anomalies are detected.

---

## The Abstract Canary Lifecycle

Do not measure rollout stages in arbitrary "minutes". Measure them in **Telemetry Confidence**.

```text
Phase 1: The Blast Radius Test (e.g., 5% traffic)
→ Action: Route a minimal traffic subset to the new version.
→ Wait: "Initial Bake Period" (Sufficient time to accumulate statistically significant metrics).
→ Gate: If anomalies detected → Instant Auto-Rollback. If baseline matched → Proceed.

Phase 2: The Confidence Expansion (e.g., 25% -> 50% traffic)
→ Action: Step up traffic weight.
→ Wait: "Sustained Load Period" (Testing for slow burns like memory leaks or thread exhaustion).
→ Gate: Continual baseline comparison.

Phase 3: The Full Cutover
→ Action: Route 100% traffic. Deprecate legacy version.
```

---

## Infrastructure as Code: ArgoCD Rollouts

_Architectural Rule:_ The `pause.duration` and `analysis.interval` values below must be dynamically injected based on the service's SLA and traffic volume, not hardcoded. High-throughput services require very short bake times; low-throughput services require longer accumulation windows.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: "${INITIAL_BAKE_TIME}" } # e.g., 1m for High-RPS, 15m for Low-RPS
        - setWeight: 50
        - pause: { duration: "${SUSTAINED_LOAD_TIME}" }
      trafficRouting:
        istio:
          virtualService:
            name: myapp-vs
      analysis:
        interval: "${SCRAPE_INTERVAL}"
        # Analysis must evaluate RELATIVE degradation, not absolute values
        metrics:
          - name: relative-error-rate
            query: |
              (rate(requests_total{status=~"5..", version="v2"}[5m]) / rate(requests_total{version="v2"}[5m]))
              > 
              (rate(requests_total{status=~"5..", version="v1"}[5m]) / rate(requests_total{version="v1"}[5m])) * 1.5
```

---

## The Agnostic Metrics Contract

Never alert on absolute numbers during a Canary. Always compare `v2` directly against `v1` (The Baseline).

```text
Monitoring Setup:
├─ Reliability (Error Rate)
│  └─ Alert if: v2 Error Rate > (v1 Error Rate * Tolerance Multiplier)
│
├─ Performance (Latency)
│  └─ Alert if: v2 p95 Latency > (v1 p95 Latency + Acceptable Overhead Threshold)
│
├─ Resource Saturation
│  ├─ CPU: Alert if v2 shows a statistically significant deviation from v1 per request.
│  └─ Memory: Alert if v2 shows an upward growth trend (Memory Leak), regardless of the absolute % used.
```

---

## The Incident Timeline (State-Based)

```text
T+0: Rollout Initiated (v2 receives subset traffic).
T+1: Initial Bake completed. Metrics align with v1 baseline. Weight increased.
T+2: Sustained Load triggers anomaly. (e.g., v2 p95 latency degrades by 40%).
T+3: Automated Circuit Breaker trips. VirtualService patched.
T+4: Traffic safely isolated 100% to v1. Paging on-call engineer for post-mortem.
```
