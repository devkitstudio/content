## The Cost of Mutable Deployments

Using `latest` creates untraceable environments. When production breaks, MTTR (Mean Time To Recovery) skyrockets because engineers cannot answer the simple question: _"What exact code is currently running?"_

### Feature Comparison

| Scenario                | Mutable (`:latest`)                           | Immutable (`:a1b2c3d`)                    |
| :---------------------- | :-------------------------------------------- | :---------------------------------------- |
| **New pod starts**      | Might download new image, might use old cache | Downloads exact SHA, perfectly repeatable |
| **Rollback triggered**  | Fails. "Rollback to what?"                    | Instant. Reverts to previous SHA          |
| **Debugging**           | Zero traceability                             | `git log a1b2c3d` shows exact code        |
| **Cluster Consistency** | Node A and Node B might run different code    | 100% Guaranteed Consistency               |

### The Financial ROI (At 500 Engineers)

**Without immutable tags:**

- Incidents: ~2/week (mysterious cache failures, race conditions).
- MTTR: 45+ minutes (hunting down the ghost version).
- Engineering Waste: ~$42,000/year in lost productivity.

**With immutable Git SHA tags (4-hour one-time setup):**

- Incidents: Drastically reduced (infrastructure-induced failures eliminated).
- MTTR: < 5 minutes (One-click `kubectl rollout undo`).
- Engineering Waste: Near zero.
