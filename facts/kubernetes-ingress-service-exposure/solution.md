# Service Exposure Methods

## 1. ClusterIP (Internal Only)

**Default service type - no external access**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: myapp
spec:
  type: ClusterIP  # Default
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
```

**Use case:** Internal service-to-service communication
- Database service (only for app pods)
- Internal APIs (only for backend services)
- Microservices within cluster

**Access:** Only within cluster via DNS
```
curl http://api.myapp.svc.cluster.local
```

## 2. NodePort (Simple External Access)

**Exposes service on every node's IP**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: NodePort
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001  # 30000-32767
```

**Access:** `<NODE_IP>:30001`

**Pros:**
- Simple, no extra infrastructure
- Works in any Kubernetes environment
- Good for development/testing

**Cons:**
- Ugly port numbers (30001, etc.)
- Only works with node IPs
- Poor load balancing
- Every node is a potential entry point

**Cost:** Free (no cloud resources)

**Use case:** Development, testing, internal networks

## 3. LoadBalancer (Cloud Native)

**Cloud provider creates load balancer (ELB/ALB/LB)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 3000
```

**What happens:**
1. Kubernetes tells cloud provider to create LB
2. Cloud provider assigns external IP
3. LB routes traffic to NodePort service

**Access:** `<EXTERNAL_IP>:80`

**Pros:**
- Cloud provider handles load balancing
- Automatic failover
- Standard port (80/443)
- One IP per service

**Cons:**
- Expensive ($0.10-0.30/hour per LB)
- One IP per service (cost scales)
- No path-based routing
- Limited features (no headers, no hostnames)

**Cost:** $100-300/month per service

**Use case:** Production apps, simple exposures

## 4. Ingress (Recommended for Production)

**Kubernetes-native HTTP/HTTPS routing**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
```

**Multiple services under one domain:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 3000
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

**Pros:**
- One ingress controller = many services
- Path-based routing (/api, /web)
- Host-based routing (api.com, web.com)
- TLS/HTTPS termination
- Cost-effective

**Cons:**
- Requires Ingress Controller
- More setup than LoadBalancer
- Slower than bare LoadBalancer

**Cost:** $0.30/month (one controller for all) vs $3000+/month (LB per service)

**Use case:** Production with multiple services

## 5. Ingress Controller Installation

**Install NGINX Ingress Controller:**

```bash
# Using Helm (recommended)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Or using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
```

**Get ingress IP:**

```bash
kubectl get svc -n ingress-nginx
# NAME                       TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)
# nginx-ingress-controller   LoadBalancer   10.0.200.50   203.0.113.10     80:31379/TCP,443:31390/TCP

# Update DNS: *.example.com → 203.0.113.10
```

## 6. Comparison Table

| Aspect | ClusterIP | NodePort | LoadBalancer | Ingress |
|--------|-----------|----------|--------------|---------|
| **Type** | Internal | Node IP | Cloud LB | DNS routing |
| **Access** | Cluster only | Any node IP | External IP | Domain name |
| **Port** | Any | 30000-32767 | 80/443 | 80/443 |
| **Cost** | Free | Free | $0.30/hr | $0.02/hr |
| **TLS/HTTPS** | No | No | Cloud LB | Yes |
| **Path routing** | No | No | Limited | Yes |
| **Host routing** | No | No | No | Yes |
| **Scaling** | N/A | One per service | One per service | Many per controller |
| **Best for** | Internal | Dev/testing | Simple prod | Prod with multiple services |

## 7. Decision Matrix

```
Need external access?
├─ NO: ClusterIP
└─ YES:
   ├─ Development/Testing?
   │  ├─ YES: NodePort
   │  └─ NO: Production
   │     ├─ Multiple services/paths?
   │     │  ├─ YES: Ingress (cost-effective)
   │     │  └─ NO: LoadBalancer (simple)
   │     ├─ Need HTTPS?
   │     │  └─ YES: Ingress (automatic)
   │     ├─ Need path routing?
   │     │  └─ YES: Ingress
   │     └─ Budget?
   │        ├─ Limited: Ingress
   │        └─ Unlimited: LoadBalancer
```

## 8. Production Pattern: Ingress + LoadBalancer

**Best practice - combine both:**

```yaml
# 1. LoadBalancer exposes Ingress Controller
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: ingress-nginx
spec:
  type: LoadBalancer  # One cloud LB
  selector:
    app: nginx-ingress
  ports:
  - port: 80
    targetPort: 80
  - port: 443
    targetPort: 443

---
# 2. Multiple Ingresses route to different services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: tls-cert
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 3000
```

**Cost breakdown:**
- 1 LoadBalancer: $0.30/hour = $220/month
- Unlimited Ingress routes: Free
- Total: ~$220/month for any number of services

