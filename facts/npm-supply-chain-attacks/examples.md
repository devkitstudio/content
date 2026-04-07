# Real Incidents and Lessons Learned

## Case Study: event-stream (2018)

**What happened:**
- Popular NPM package with 2+ million downloads
- Attacker gained access to maintainer account
- Added malicious code that stole Bitcoin wallets
- Targeted crypto apps using the package

**Lesson:** Even popular packages can be compromised. Code review and runtime monitoring are critical.

**Detection code:**
```javascript
// What event-stream was doing:
// Monitoring for .bip39 files (Bitcoin seed phrases)
// Exfiltrating wallet data to external server

// Prevention: Network monitoring
const http = require('http');
const originalRequest = http.request;

http.request = function(options, callback) {
  // Alert on suspicious external calls
  if (suspiciousDomain(options.hostname)) {
    console.error('ALERT: Suspicious outbound connection', options);
    process.exit(1);
  }
  return originalRequest.apply(this, arguments);
};
```

## Case Study: faker.js (2022)

**What happened:**
- Maintainer frustrated with unpaid labor
- Sabotaged package by publishing empty versions
- Broke builds for millions of applications
- Demonstrated maintainer burnout risk

**Lesson:** Version pinning and staging environment testing are essential.

**Prevention code:**
```json
{
  "dependencies": {
    "faker": "6.6.6"
  },
  "resolutions": {
    "faker": "6.6.6"
  },
  "engines": {
    "npm": ">=8.0.0"
  }
}
```

```javascript
// Yarn resolution strategy
const packageJson = {
  resolutions: {
    'faker': '6.6.6',  // Pin exact version
    'lodash': '4.17.21'
  }
};

// Always verify in staging first
const validateDependencies = () => {
  const packageJson = require('./package.json');
  const lockfile = require('./yarn.lock');

  Object.entries(packageJson.dependencies).forEach(([pkg, version]) => {
    const locked = lockfile.packages[pkg];
    if (!locked) throw new Error(`${pkg} not locked`);
    console.log(`${pkg}: ${version} -> ${locked.version}`);
  });
};
```

## Case Study: ua-parser-js (2021)

**What happened:**
- Dependency confusion attack
- Attacker published malicious version on npm
- Version numbering suggested it was newer
- Installed on 100+ million machines per week

**Lesson:** Validate package sources and use private registries.

**Prevention code:**
```bash
# Strict npm configuration
npm config set strict-ssl true
npm config set registry https://registry.npmjs.org/
npm config set scope:@company:registry https://private-registry.company.com/

# .npmrc
registry=https://registry.npmjs.org/
@company:registry=https://private-registry.company.com/
always-auth=true
```

```typescript
// Verify package authenticity
import crypto from 'crypto';

interface PackageVerification {
  name: string;
  version: string;
  integrity: string;
  publisher: string;
}

class SupplyChainValidator {
  private allowedPublishers = new Set([
    'legitimate-author',
    'company-team'
  ]);

  async verifyPackage(metadata: PackageVerification): Promise<boolean> {
    // 1. Check publisher
    if (!this.allowedPublishers.has(metadata.publisher)) {
      console.error(`Untrusted publisher: ${metadata.publisher}`);
      return false;
    }

    // 2. Verify integrity hash
    const expected = this.getKnownIntegrity(
      metadata.name,
      metadata.version
    );
    if (metadata.integrity !== expected) {
      console.error('Integrity mismatch');
      return false;
    }

    // 3. Check for 2FA on publisher account
    const publisherInfo = await fetch(
      `https://registry.npmjs.org/-/user/org.couchdb.user:${metadata.publisher}`
    ).then(r => r.json());

    return publisherInfo.has2fa === true;
  }
}
```

## Snyk Continuous Monitoring

```yaml
# .snyk configuration
version: v1.25.0
cli:
  monitor:
    # Fail on high severity vulnerabilities
    severity-threshold: high
  test:
    # Strict testing policy
    fail-on: upgradable
protect:
  - path: 'package.json'
    rules:
      - id: 'npm-package:event-stream'
        action: 'disallow'
      - id: 'npm-package:ua-parser-js:<=0.7.28'
        action: 'disallow'
```

## Socket.dev Runtime Detection

```typescript
// Detect suspicious patterns in packages
interface SuspiciousSignals {
  newDependencies: boolean;      // Added new external deps
  networkCalls: boolean;         // Makes external HTTP calls
  fileSystemAccess: boolean;     // Reads sensitive files
  processSpawning: boolean;      // Spawns child processes
  obfuscatedCode: boolean;       // Uses code obfuscation
  dynamicRequire: boolean;       // Uses dynamic requires
  cryptoUsage: boolean;          // Uses crypto libs unexpectedly
}

// Socket.dev can detect all these patterns
const riskLevel = (signals: SuspiciousSignals): 'safe' | 'caution' | 'critical' => {
  const riskCount = Object.values(signals).filter(Boolean).length;
  if (riskCount >= 4) return 'critical';
  if (riskCount >= 2) return 'caution';
  return 'safe';
};
```

## Dependency Allowlist Example

```typescript
// config/dependency-allowlist.json
{
  "allowed": {
    "express": ["4.18.2"],
    "lodash": ["4.17.21"],
    "axios": ["1.x"],
    "uuid": ["9.x"]
  },
  "blocked": {
    "event-stream": "all",
    "ua-parser-js": ["<=0.7.28"],
    "crypto-js": "all"  // Use node's crypto instead
  },
  "requireAudit": [
    "aws-sdk",
    "firebase-admin",
    "stripe"
  ]
}
```

```javascript
// Validate during npm install
const fs = require('fs');
const path = require('path');

function validateDependencies() {
  const allowlist = JSON.parse(
    fs.readFileSync('./config/dependency-allowlist.json', 'utf8')
  );

  const packageJson = JSON.parse(
    fs.readFileSync('./package.json', 'utf8')
  );

  const deps = {
    ...packageJson.dependencies,
    ...packageJson.devDependencies
  };

  for (const [pkg, version] of Object.entries(deps)) {
    // Check if package is blocked
    if (allowlist.blocked[pkg]) {
      throw new Error(`Package "${pkg}" is blocked: ${allowlist.blocked[pkg]}`);
    }

    // Check if version is allowed
    const allowed = allowlist.allowed[pkg];
    if (allowed && !allowed.includes(version)) {
      throw new Error(
        `Version ${version} of "${pkg}" not in allowlist: ${allowed.join(', ')}`
      );
    }
  }

  console.log('All dependencies validated');
}

// Run during npm postinstall
if (require.main === module) {
  validateDependencies();
}
```
