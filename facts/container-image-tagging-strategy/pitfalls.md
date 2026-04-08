## Production Disasters & Emergency Recovery

### The Cache Inconsistency Trap

If you deploy `myapp:latest` with `imagePullPolicy: IfNotPresent`, Kubernetes will not pull the image if it already exists on the Node.

- **Node A** pulled `latest` yesterday (v1.0.0).
- **Node B** was scaled up today and pulled `latest` just now (v1.0.1).
- **Result:** Your cluster is simultaneously running two different versions of your application under the exact same tag. Bugs will appear intermittently depending on which Node receives the load balancer traffic.

### Emergency Fix: Recovering a Broken `latest` Deployment

If you inherit a broken production environment using `latest`, you cannot just "rollback", because the previous state was overwritten. You must pin the current running state by its underlying `RepoDigest`.

```bash
# Step 1: Find the exact SHA-256 digest of the image currently running in the pod
kubectl get deployment myapp -o yaml | grep image

# Step 2: Query the registry for that exact digest
docker inspect docker.io/company/myapp:latest | jq -r '.[0].RepoDigests[0]'
# Output: docker.io/company/myapp@sha256:7f8a9b...

# Step 3: Explicitly tag that digest locally to secure it
docker tag docker.io/company/myapp@sha256:7f8a9b... myapp:stable-recovery

# Step 4: Force the deployment to use the pinned digest instead of 'latest'
kubectl set image deployment/myapp myapp=docker.io/company/myapp@sha256:7f8a9b...
```
