## Real Disasters from Using `latest`

### Disaster 1: Silent Production Failure

```
Timeline:
─────────
Monday 2 PM:  Deploy myapp:latest (v2.0.0)
              Production is stable

Monday 3 PM:  Developer pushes fix to bug-branch
              CI rebuilds "latest" tag (now v2.0.1)
              But v2.0.1 has incomplete feature

Monday 4 PM:  Pod crashes and restart
              Kubernetes pulls myapp:latest → Gets v2.0.1
              Production broken
              No one knows why (code unchanged)

Monday 5 PM:  Incident response
              Doesn't understand issue
              Not in any git commit
              Rollback? To what? No one knows what was running
              30 minutes down
```

### Disaster 2: Cache Inconsistency

```
Scenario: Different Kubernetes nodes have different `latest`

Node A:
└─ Cached: myapp:latest (v2.0.0, pulled 1 hour ago)

Node B:
└─ Cached: myapp:latest (v2.0.1, pulled now)

Result:
Same app, different code on different nodes
Race conditions, inconsistent behavior
Impossible to debug
```

### Disaster 3: Rollback Nightmare

```
Production is broken. Rollback!
But how? What to rollback to?

$ kubectl get pod myapp-12345 -o yaml | grep image
image: myapp:latest

What is "latest"? No one knows.
Was it v2.0.0? v2.0.1? v2.1.0-beta?

No way to track down what was running.
```

## Comparison: `latest` vs SHA Tags

| Scenario | With `latest` | With SHA Tags |
|----------|---|---|
| **New pod starts** | Downloads image (cache miss or old cache) | Downloads exact SHA, repeatable |
| **Rollback needed** | "Rollback to what?" | Rollback to specific SHA |
| **Debugging** | No traceability | `git log` shows exact code |
| **Concurrent deploys** | Chaos (race conditions) | Clean, isolated |
| **Version mismatch** | Possible across nodes | Impossible |

## Kubernetes imagePullPolicy Impact

```yaml
# Default: IfNotPresent
imagePullPolicy: IfNotPresent

# With "latest" + IfNotPresent:
# "Did pull latest 1 hour ago, using cache"
# → Old version might run
# → Inconsistent across cluster

# With SHA tag + IfNotPresent:
# "Did pull abc1234, using cache"
# → Same image everywhere
# → Immutable guarantee
```

## Cost of Not Using Immutable Tags

```
Company: 500 engineers, weekly deployments

Without immutable tags:
├─ Incidents/week: 2-3 (mysterious failures)
├─ MTTR (mean time to recovery): 45 min (includes debugging)
├─ Cost: 2 incidents × 45 min × $150/hour = $225/week
├─ Engineering time spent debugging: 4 hours = $600/week
└─ Total: ~$825/week = $42,900/year

With immutable tags (one-time setup 4 hours):
├─ Incidents/week: 0.1 (only real bugs)
├─ MTTR: 5 min (quick rollback)
├─ Cost: 0.1 incidents × 5 min × $150/hour = $12.50/week
├─ Confidence in deployments: High
└─ Total: ~$650/year (vs $42,900/year)

ROI: Break even in < 1 week
```

## Testing: Verify Your Tags Are Immutable

```bash
#!/bin/bash
# Test that tag doesn't change

TAG="myapp:abc1234"

# Get image digest
DIGEST1=$(docker inspect --format='{{.RepoDigests}}' $TAG)

# Wait a week
# ... time passes ...

# Check digest again
DIGEST2=$(docker inspect --format='{{.RepoDigests}}' $TAG)

if [ "$DIGEST1" == "$DIGEST2" ]; then
  echo "✓ Tag is immutable (same digest)"
else
  echo "✗ Tag changed! Someone overwrote it"
  exit 1
fi
```

## Fix a Broken `latest` Deployment

If you're stuck with `latest`:

```bash
# Step 1: Find what's actually running
kubectl get deployment myapp -o yaml | grep image

# Step 2: Get its digest
docker inspect docker.io/company/myapp:latest | jq -r '.[0].RepoDigests[0]'
# Output: docker.io/company/myapp@sha256:abc123...

# Step 3: Immediately tag it with explicit version
docker tag docker.io/company/myapp:latest myapp:stable-2024-04-06

# Step 4: Update deployment to use explicit tag
kubectl set image deployment/myapp myapp=myapp:stable-2024-04-06

# Step 5: Implement proper tagging going forward
```

## Prevention Checklist

- [ ] Never use `latest` in production deployments
- [ ] Use git SHA, semver, or build ID
- [ ] Registry enforces immutability rules
- [ ] All images tagged with multiple strategies
- [ ] Deployment manifests specify exact tags
- [ ] CI/CD builds and pushes with correct tags
- [ ] Rollback procedure documented and tested
- [ ] No manual tag overwrites allowed
