# What to Do When a Secret Leaks

## Immediate Actions (Minutes)

1. **Rotate the credential immediately**
   - Deactivate the exposed key/token
   - Generate new credential
   - Update all running systems

2. **Alert the team**
   - Notify security, ops, and relevant engineers
   - Document timeline of exposure

3. **Search for usage**
   - Check git history for first commit
   - Search all branches and tags
   - Review CI/CD logs for logs containing secret

## Short-term (Hours)

4. **Remove from history**
   ```bash
   # Using git-filter-repo (preferred)
   git filter-repo --invert-paths --path "config.js"
   git push origin --force-with-lease --all

   # Or using BFG Repo-Cleaner
   bfg --delete-files config.js
   ```

5. **Check access logs**
   - Monitor API/service for unauthorized access
   - Review timestamps after secret was committed
   - Check from unusual IPs/geographies

6. **Notify stakeholders**
   - Inform affected users if data was accessed
   - Document incident report
   - Update security incident log

## Long-term (Days)

7. **Implement detection**
   - Add git-secrets hooks to prevent recurrence
   - Configure GitHub secret scanning
   - Enable CI/CD scanning

8. **Analysis and learning**
   - Root cause: Why was secret hardcoded?
   - Process improvement: Better defaults?
   - Training: Educate team on secret management

## Code Examples for Remediation

### Remove Secret from Git History

```bash
#!/bin/bash
# Script to remove sensitive file from all commits

git filter-repo \
  --invert-paths \
  --path '.env' \
  --path 'config/secrets.js' \
  --force

# Force push to all remotes
git push origin --force-with-lease --all
git push origin --force-with-lease --tags
```

### Add to .gitignore (Prevent Future)

```bash
# .gitignore
.env
.env.local
.env.*.local
config/secrets.js
*.pem
*.key
*.cert
npm-debug.log
.aws/credentials
```

### Verify No Secrets Remain

```bash
# Using git-secrets
git secrets --install
git secrets --add 'AKIA[0-9A-Z]{16}'  # AWS key pattern
git secrets --add 'password\s*=\s*.+'  # Password pattern
git secrets --scan --all

# Using TruffleHog
truffleHog filesystem . --json
truffleHog git https://github.com/user/repo.git
```

### Recover Rotated Secret

```typescript
// Before: Hardcoded secret
const API_KEY = 'sk-1234567890abcdef';

// After: From environment
const API_KEY = process.env.EXTERNAL_API_KEY;
if (!API_KEY) {
  throw new Error('EXTERNAL_API_KEY environment variable not set');
}

// From secret management
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const getSecret = async (name: string) => {
  const client = new SecretsManager();
  const response = await client.getSecretValue({ SecretId: name });
  return JSON.parse(response.SecretString!);
};

const credentials = await getSecret('prod/external-api-key');
```

### Enable GitHub Secret Scanning

GitHub automatically scans for patterns, but you can add custom patterns:

```yaml
# In repository settings > Security > Secret scanning
# Add custom pattern for proprietary credentials
patterns:
  - name: Internal API Key
    pattern: internal[_-]api[_-]key[_=][\w\-]{32,}
  - name: Database URL
    pattern: postgresql://[^\s:]+:[^\s@]+@[^\s/]+/\w+
```
