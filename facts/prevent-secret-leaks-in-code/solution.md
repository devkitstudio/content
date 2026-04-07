# Preventing and Detecting Secret Leaks

## Defense Strategy

### 1. Pre-Commit Hooks

Scan code before commits reach repository:
- **git-secrets**: Pattern-based detection
- **TruffleHog**: Entropy analysis for secret-like strings
- **husky**: Easy pre-commit hook setup
- Blocks commits containing hardcoded credentials

### 2. Local Development

Set up secure environment handling:
- Use `.env` files (never committed)
- `.gitignore` patterns for sensitive files
- IDE plugins to warn on hardcoded secrets
- Environment variable injection at runtime

### 3. GitHub Secret Scanning

Built-in detection on GitHub:
- Scans pushes for known secret patterns
- Detects common formats: AWS keys, API tokens, etc.
- Alerts repository maintainers
- Custom patterns for proprietary credentials

### 4. CI/CD Pipeline Scanning

Multiple scanning layers:
- **SAST tools**: SonarQube, Semgrep detect secrets
- **Dependency checks**: Snyk, npm audit
- **Container scanning**: Trivy, Anchore for Docker images
- **Infrastructure as Code**: Checkov for Terraform/CloudFormation

### 5. Rotation Policy

Assume secrets may leak:
- Rotate credentials every 90 days minimum
- Immediate rotation if leak detected
- Multi-factor authentication on sensitive services
- API key versioning and gradual deprecation

### 6. Secret Management

Centralized secret storage:
- **AWS Secrets Manager**, **HashiCorp Vault**
- **1Password, LastPass** for team credentials
- Audit logs on all secret access
- Automatic rotation capabilities

## Detection Techniques

### Entropy-Based Detection
Low-entropy strings unlikely to be secrets; high-entropy strings (random characters) flagged

### Pattern Matching
Regex patterns for:
- AWS access key format: `AKIA[0-9A-Z]{16}`
- Private key headers: `-----BEGIN.*PRIVATE KEY-----`
- API token patterns

### Hash Comparison
Compare against known-leaked credentials databases (Have I Been Pwned, etc.)

## Incident Response

If secret is leaked:
1. Immediately rotate the credential
2. Check access logs for unauthorized use
3. Notify security team and stakeholders
4. Remove from git history (filter-branch, BFG)
5. Implement additional monitoring
