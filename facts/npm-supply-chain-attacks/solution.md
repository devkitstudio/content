# Protecting Against Supply Chain Attacks

## Multi-Layer Defense

### 1. Dependency Locking

Use lock files to ensure reproducibility:
- **package-lock.json** (npm): Locks exact versions and checksums
- **yarn.lock** (Yarn): Prevents dependency drift
- Commit lock files to version control
- Prevents unexpected updates to compromised versions

### 2. Dependency Auditing

Continuous scanning for known vulnerabilities:
- `npm audit`: Built-in vulnerability scanner
- **Snyk**: Real-time monitoring and remediation
- **Socket.dev**: Detects suspicious behavior patterns
- Run in CI/CD on every pull request

### 3. Allowlist/Denylist Strategy

Maintain control over dependencies:
- **Allowlist**: Explicitly approve which packages can be used
- **Denylist**: Block known-bad packages immediately
- Override policies for critical packages
- Regular review of approved packages

### 4. Package Integrity Verification

Verify package authenticity:
- Check npm package signatures (npm provenance)
- Verify publisher identity (npm 2FA requirement)
- Compare package hashes in lock file
- Use npm package scope verification

### 5. Update Strategy

Thoughtful dependency updates:
- Don't auto-update major versions
- Test all updates in staging first
- Update high-risk packages more frequently
- Pin critical dependencies to specific versions

### 6. Dependency Minimization

Reduce attack surface:
- Remove unused dependencies
- Prefer lighter alternatives
- Evaluate necessity before adding packages
- Regular cleanup of deprecated packages

### 7. Network Monitoring

Detect anomalous behavior:
- Monitor outbound connections from Node process
- Alert on unexpected domains contacted
- Check for unusual data exfiltration
- Runtime instrumentation/APM

### 8. Build Reproducibility

Enable verification of builds:
- Reproducible builds with exact versions
- Docker image pinning with digests
- No dynamic downloads during build
- Build artifacts signed and verified

## Real Incidents Overview

| Incident | Impact | Detection |
|----------|--------|-----------|
| **event-stream** (2018) | Compromised to steal Bitcoin wallets | Community review of code changes |
| **faker.js** (2022) | Maintainer sabotage, broken builds | CI/CD failures, version pinning |
| **ua-parser-js** (2021) | Injected malware into millions of apps | npm audit, Snyk alerts |
| **npm-registry-fetch** (2020) | Dependency confusion attack | Package allowlists |
