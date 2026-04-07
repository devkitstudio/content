## Feature Flag Lifecycle

```
1. Development → 2. Testing → 3. Canary → 4. Gradual Rollout → 5. Full Release → 6. Cleanup
```

**Bad approach:**
```typescript
if (process.env.FEATURE_NEW_CHECKOUT) {
  // new code
} else {
  // old code
}
```
Problem: Flag scattered everywhere, impossible to remove.

**Good approach:**
```typescript
const featureFlags = {
  NEW_CHECKOUT: {
    enabled: true,
    rollout: 100,
    regions: ['US', 'EU'],
    minVersion: '2.5.0',
    description: 'New checkout flow with optimized UX',
    deprecatedAt: '2026-06-01'
  }
};

if (flags.isEnabled('NEW_CHECKOUT')) {
  // new code
}
```

## Flag Configuration

```typescript
interface FeatureFlag {
  name: string;
  enabled: boolean;
  rollout: number;              // 0-100 percentage
  targetAudience?: {
    userIds?: string[];
    regions?: string[];
    userTiers?: string[];
    betaTesters?: boolean;
  };
  minVersion?: string;
  deprecated?: boolean;
  deprecatedAt?: Date;
  variants?: {
    control: number;            // %
    variant_a: number;
    variant_b: number;
  };
  metadata?: Record<string, any>;
}
```

## Centralized Flag Evaluation

```typescript
import Hashids from 'hashids';

class FeatureFlagService {
  private flags: Map<string, FeatureFlag> = new Map();

  constructor(private config: FeatureFlag[]) {
    config.forEach(flag => this.flags.set(flag.name, flag));
  }

  isEnabled(
    flagName: string,
    context?: {
      userId?: string;
      region?: string;
      userTier?: string;
      appVersion?: string;
    }
  ): boolean {
    const flag = this.flags.get(flagName);
    if (!flag) return false;
    if (flag.deprecated) return false;

    if (!flag.enabled) return false;

    // Version check
    if (context?.appVersion && flag.minVersion) {
      if (this.compareVersions(context.appVersion, flag.minVersion) < 0) {
        return false;
      }
    }

    // Audience targeting
    if (flag.targetAudience) {
      if (
        flag.targetAudience.userIds &&
        !flag.targetAudience.userIds.includes(context?.userId || '')
      ) {
        return false;
      }

      if (
        flag.targetAudience.regions &&
        !flag.targetAudience.regions.includes(context?.region || '')
      ) {
        return false;
      }
    }

    // Percentage rollout (deterministic)
    if (flag.rollout < 100) {
      return this.isInRollout(flagName, context?.userId || '', flag.rollout);
    }

    return true;
  }

  private isInRollout(flagName: string, userId: string, percentage: number): boolean {
    const hashids = new Hashids(flagName);
    const hashValue = parseInt(hashids.encode(userId), 16) % 100;
    return hashValue < percentage;
  }

  private compareVersions(v1: string, v2: string): number {
    const parts1 = v1.split('.').map(Number);
    const parts2 = v2.split('.').map(Number);

    for (let i = 0; i < Math.max(parts1.length, parts2.length); i++) {
      const p1 = parts1[i] || 0;
      const p2 = parts2[i] || 0;
      if (p1 !== p2) return p1 - p2;
    }
    return 0;
  }

  getVariant(flagName: string, userId: string): string {
    const flag = this.flags.get(flagName);
    if (!flag?.variants) return 'control';

    const hashids = new Hashids(flagName);
    const hashValue = parseInt(hashids.encode(userId), 16) % 100;

    let cumulative = 0;
    for (const [variant, percentage] of Object.entries(flag.variants)) {
      cumulative += percentage;
      if (hashValue < cumulative) return variant;
    }

    return 'control';
  }
}

export const flags = new FeatureFlagService([
  {
    name: 'NEW_CHECKOUT',
    enabled: true,
    rollout: 50,
    targetAudience: { regions: ['US'] },
    minVersion: '2.5.0'
  },
  {
    name: 'DARK_MODE',
    enabled: true,
    rollout: 100,
    variants: {
      control: 50,
      dark_mode_v1: 25,
      dark_mode_v2: 25
    }
  }
]);
```

## Dead Code Cleanup

Track when flags were added:

```typescript
const flagMetadata = {
  'NEW_CHECKOUT': {
    addedAt: new Date('2026-01-15'),
    deprecationDeadline: new Date('2026-06-15'),
    trackingIssue: 'TICKET-1234'
  }
};

// CI check: fail if deprecated flag still used after deadline
if (new Date() > flagMetadata[flag].deprecationDeadline) {
  throw new Error(
    `Flag ${flag} is deprecated. Remove it or update deadline in ${flagMetadata[flag].trackingIssue}`
  );
}
```
