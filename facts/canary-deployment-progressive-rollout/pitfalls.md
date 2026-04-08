Canary deployments operate purely at the Application Layer (stateless routing) and will fail catastrophically in these specific scenarios:

* **Database Schema Breaking Changes:** If `v2` includes a database migration that drops a column or alters a data type, the 95% of traffic still routed to `v1` will instantly crash due to a schema mismatch. *(This requires a Zero-Downtime Database Migration pattern instead).*
* **Stateful In-Memory Sessions:** If your application stores session state in the container's RAM rather than a centralized store (like Redis or Memcached), user sessions will be destroyed when the load balancer shifts their subsequent requests between `v1` and `v2`.
* **Overkill for Low-Traffic Tools:** For an internal admin dashboard with low traffic, provisioning Istio, Prometheus, and Argo Rollouts is textbook over-engineering. A standard Kubernetes `RollingUpdate` strategy is more than sufficient.
