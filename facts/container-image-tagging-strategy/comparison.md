## The Cost of Mutable Deployments

Using `latest` creates untraceable environments. When production breaks, MTTR (Mean Time To Recovery) skyrockets because engineers cannot answer the simple question: _"What exact code is currently running?"_

### Feature Comparison

| Scenario                | Mutable (`:latest`)                           | Immutable (`:a1b2c3d`)                    |
| :---------------------- | :-------------------------------------------- | :---------------------------------------- |
| **New pod starts**      | Might download new image, might use old cache | Downloads exact SHA, perfectly repeatable |
| **Rollback triggered**  | Fails. "Rollback to what?"                    | Instant. Reverts to previous SHA          |
| **Debugging**           | Zero traceability                             | `git log a1b2c3d` shows exact code        |
| **Cluster Consistency** | Node A and Node B might run different code    | 100% Guaranteed Consistency               |

### The ROI of Immutable Architecture

**The Baseline (Mutable Tags):**

- **Incident Rate:** High vulnerability to "ghost" bugs (cache mismatches, race conditions across nodes).
- **MTTR (Mean Time To Recovery):** Measured in hours (Engineers must hunt down and manually verify the running state).
- **Engineering Waste:** Massive loss of FTE (Full-Time Equivalent) hours debugging infrastructure instead of shipping features.

**The Optimized State (Immutable Git SHA Tags):**

- **Incident Rate:** **Eliminates 100%** of infrastructure-induced deployment anomalies.
- **MTTR:** **Reduced by > 90%** (A simple `kubectl rollout undo` takes seconds, restoring a mathematically guaranteed known-good state).
- **Engineering Waste:** **Reduced to near zero** for version-mismatch triage.
