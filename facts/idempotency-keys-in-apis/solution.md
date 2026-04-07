The absolute standard for resolving duplicate mutations in Senior API design is the **Idempotency Key**.

### The Concept
An idempotent operation means that executing it once has the exact same effect as executing it 10,000 times.
But processing a payment is fundamentally *not* idempotent. To fix this, you force it to be idempotent using a unique token.

### The Flow
1. **The Client** generates a unique `UUID` (e.g., `req-1234`) and attaches it to an HTTP Header called `Idempotency-Key` or `X-Idempotency-Key` when clicking "Pay".
2. **The Server** receives the request and immediately checks a fast database (like Redis or a dedicated SQL table) for `req-1234`.
3. **If it doesn't exist**: The server locks the key, processes the payment, saves the final JSON response payload to the cache under `req-1234`, and returns it to the client.
4. **If it DOES exist**: The server instantly bypasses the controller logic and replays the cached JSON response.

The second, duplicate click from the flaky 3G connection will just receive the `200 OK` success message from the first click—without hitting the payment gateway a second time!
