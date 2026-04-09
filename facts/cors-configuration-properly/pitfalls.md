## The Traps of Cross-Origin Misconfiguration

### 1. The Regex Bypass Vulnerability

When configuring dynamic origins, developers often use weak regular expressions to allow all subdomains.

- **The Flawed Regex:** `new RegExp('https://.*\.mycompany\.com')`
- **The Exploit:** An attacker registers `https://malicious-mycompany.com` or `https://attacker.com?.mycompany.com`. The weak regex matches, granting the attacker full CORS access to your API.
- **The Fix:** Always anchor your regex boundaries strictly: `^https:\/\/(app|admin)\.mycompany\.com$` or use exact string matching in an array.

### 2. The "Reflect Origin" Anti-Pattern

A common (and dangerous) StackOverflow answer to "How to bypass CORS" is to write middleware that simply echoes back whatever `Origin` header the client sends.

```javascript
// FATAL ANTI-PATTERN
app.use((req, res, next) => {
  res.header("Access-Control-Allow-Origin", req.headers.origin); // ❌ Allows ANY site
  res.header("Access-Control-Allow-Credentials", "true");
  next();
});
```

This entirely defeats the purpose of CORS. Combined with `Credentials: true`, any malicious website can make authenticated requests to your API using the victim's session.

### 3. The `null` Origin Trust

Local HTML files (`file:///`) and redirects can send an origin of `null`.

- **The Trap:** Some developers explicitly add `"null"` to their allowlist to fix local development issues.
- **The Exploit:** Attackers can generate a malicious iframe with a sandboxed environment, forcing its origin to be `null`, thereby bypassing your CORS policy. Never allow `"null"` in production.
