# CORS Architecture & Security Mandate

Cross-Origin Resource Sharing (CORS) is not a feature to be "bypassed" during development; it is a fundamental browser security mechanism built on top of the Same-Origin Policy (SOP).

Misconfiguring CORS is a critical vulnerability that allows malicious domains to execute unauthorized state-changing actions using the victim's authenticated session.

## The Architectural Standard

A production-grade API must enforce strict boundary controls using the following mandate:

1.  **Explicit Allowlisting:** Never use the `*` (wildcard) wildcard for the `Access-Control-Allow-Origin` header in an authenticated API. Origins must be strictly matched against a dynamic, environment-injected allowlist.
2.  **Controlled Credentials:** The `Access-Control-Allow-Credentials: true` header must only be emitted if the requesting origin strictly matches the allowlist. (The CORS spec explicitly forbids combining `Credentials: true` with `Origin: *`).
3.  **Method & Header Restriction:** Do not reflect arbitrary headers requested by the client. Explicitly define allowed methods (`GET, POST, PUT, DELETE`) and strictly necessary headers (`Content-Type, Authorization`).
4.  **Preflight Caching:** Preflight (`OPTIONS`) requests add a full round-trip of latency to complex requests. Always configure `Access-Control-Max-Age` to cache the preflight response to minimize network overhead.

## The Request Flow Mental Model

```text
CLIENT (Browser)                                    API SERVER
       │                                                 │
       │ ── 1. Preflight Request (OPTIONS) ────────────▶ │
       │    (Asks: Do you allow my Origin & Methods?)    │
       │                                                 │
       │ ◀── 2. Preflight Response ───────────────────── │
       │    (Headers: Allow-Origin, Max-Age, etc.)       │
       │                                                 │
[Browser evaluates policy. If valid, proceeds:]          │
       │                                                 │
       │ ── 3. Actual Request (e.g., POST with body) ──▶ │
       │                                                 │
       │ ◀── 4. Actual Response ──────────────────────── │
```
