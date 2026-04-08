## Visibility: Verbose Logs vs. Distributed Traces

"But how do we debug issues without all those logs?"

You do not lose visibility; you simply change the medium. Traditional logging requires developers to manually piece together procedural events using correlation IDs. Distributed tracing (via OpenTelemetry) groups these events natively into parent-child spans.

One trace effectively replaces dozens of disjointed log lines while consuming a fraction of the index storage.

### The Old Way: Disjointed Logs (Expensive)

```text
// 22 separate lines to index, parse, and search
[INFO] POST /api/orders received
[INFO] Validating order payload
[DEBUG] Checking inventory for SKU-789
[DEBUG] Inventory query: 23ms
[INFO] Creating payment intent
[DEBUG] Stripe API call: 340ms
[INFO] Order created: ORD-123
```

### The New Way: A Single Trace (Efficient)

```text
// 1 structured event containing the entire lifecycle
Trace: ORD-123 (Total Execution Time: 580ms)
├─ validate_payload (12ms)
├─ check_inventory (23ms) → [Tags: sku=789, stock=5]
├─ charge_payment (340ms) → [Tags: provider=stripe, status=ok]
└─ create_order (45ms) → [Tags: order_id=ORD-123]
```

When an error occurs, the `ERROR` log in your hot storage contains the `trace_id`. Engineers simply copy that ID, paste it into the tracing UI, and see the exact visual waterfall of the failing request.
