# Immutable Image Architecture

Deploying `myapp:latest` to a Kubernetes cluster is an architectural anti-pattern. `latest` is a mutable pointer; it guarantees nothing about the underlying code.

Modern deployment pipelines mandate **Immutable Artifacts**. Once an image is built and tested, its tag must never be overwritten.

### The Standard: Git SHA-1 Tagging

The most robust strategy is tying the container image directly to the specific Git commit hash that generated it.

- **Anti-Pattern:** `docker pull company/api:latest`
- **Standard:** `docker pull company/api:a1b2c3d`

### The Multi-Tag Pattern (Best Practice)

While the deployment manifest must strictly use the immutable Git SHA, the CI/CD pipeline should push multiple semantic tags to the registry for developer ergonomics:

1. **The Immutable Target:** `myapp:a1b2c3d` (Used by Kubernetes).
2. **The Semantic Release:** `myapp:v1.2.3` (For public tracking).
3. **The Human Pointer:** `myapp:latest` (Convenience for local developer testing only; never used in Production).
