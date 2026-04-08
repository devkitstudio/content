## The Cardinality Trap in Loki

Loki is cheap because it does not index the log text. However, this architecture has a fatal flaw if misused: **High Cardinality Labels**.

### The Anti-Pattern

Never extract highly unique fields (like `user_id`, `trace_id`, or `ip_address`) into Loki labels in your Promtail configuration.

```yaml
# ❌ FATAL ANTI-PATTERN: Will crash Loki's index
pipeline_stages:
  - json:
      expressions:
        user_id: user_id
  - labels:
      user_id: # Creates millions of unique index streams
```

If you create a label for `user_id`, Loki creates a separate log stream for every single user. This destroys the index, causes extreme memory usage, and crashes the ingester.

### The Correct Approach

Store unique IDs inside the JSON log line. Use static labels (like `app`, `env`, `level`) to narrow down the search space, then use LogQL to parse the JSON dynamically at query time:

```logql
# ✅ CORRECT: Filter by static label, then parse JSON dynamically
{app="api", env="production"} | json | user_id="123456"
```
