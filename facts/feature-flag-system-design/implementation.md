## Complete Feature Flag Implementation

```typescript
// flags.ts
import Hashids from 'hashids';

export interface FeatureFlag {
  name: string;
  enabled: boolean;
  rollout: number;
  targetAudience?: {
    userIds?: string[];
    regions?: string[];
    userTiers?: string[];
  };
  minVersion?: string;
  deprecated?: boolean;
  deprecatedAt?: Date;
  variants?: Record<string, number>;
}

export class FlagService {
  private flags: Map<string, FeatureFlag>;
  private hashids: Hashids;

  constructor(flagConfigs: FeatureFlag[]) {
    this.flags = new Map(flagConfigs.map(f => [f.name, f]));
    this.hashids = new Hashids(process.env.FLAG_SALT || 'flag');
  }

  isEnabled(
    name: string,
    userId?: string,
    context?: {
      region?: string;
      userTier?: string;
      appVersion?: string;
    }
  ): boolean {
    const flag = this.flags.get(name);
    if (!flag) throw new Error(`Flag ${name} not found`);
    if (flag.deprecated) return false;
    if (!flag.enabled) return false;

    // Version check
    if (context?.appVersion && flag.minVersion) {
      if (this.compareVersions(context.appVersion, flag.minVersion) < 0) {
        return false;
      }
    }

    // Audience check
    if (flag.targetAudience) {
      if (flag.targetAudience.regions && context?.region) {
        if (!flag.targetAudience.regions.includes(context.region)) {
          return false;
        }
      }

      if (flag.targetAudience.userTiers && context?.userTier) {
        if (!flag.targetAudience.userTiers.includes(context.userTier)) {
          return false;
        }
      }
    }

    // Rollout percentage
    if (flag.rollout < 100 && userId) {
      return this.isInRollout(name, userId, flag.rollout);
    }

    return true;
  }

  getVariant(name: string, userId: string): string {
    const flag = this.flags.get(name);
    if (!flag?.variants) return 'control';

    const hash = this.hashids.encode(userId + name);
    const value = parseInt(hash, 16) % 100;

    let cumulative = 0;
    for (const [variant, percentage] of Object.entries(flag.variants)) {
      cumulative += percentage;
      if (value < cumulative) return variant;
    }

    return 'control';
  }

  private isInRollout(name: string, userId: string, percentage: number): boolean {
    const hash = this.hashids.encode(userId + name);
    const value = parseInt(hash, 16) % 100;
    return value < percentage;
  }

  private compareVersions(v1: string, v2: string): number {
    const [major1, minor1, patch1] = v1.split('.').map(Number);
    const [major2, minor2, patch2] = v2.split('.').map(Number);

    if (major1 !== major2) return major1 - major2;
    if (minor1 !== minor2) return minor1 - minor2;
    return patch1 - patch2;
  }

  getAllFlags(): FeatureFlag[] {
    return Array.from(this.flags.values());
  }
}

// config.ts
export const flagConfig: FeatureFlag[] = [
  {
    name: 'NEW_CHECKOUT',
    enabled: true,
    rollout: 50,
    targetAudience: { regions: ['US', 'EU'] },
    minVersion: '2.5.0'
  },
  {
    name: 'DARK_MODE',
    enabled: true,
    rollout: 100,
    variants: {
      control: 50,
      dark_v1: 25,
      dark_v2: 25
    }
  },
  {
    name: 'OLD_FEATURE',
    enabled: false,
    rollout: 0,
    deprecated: true,
    deprecatedAt: new Date('2026-03-01')
  }
];

export const flags = new FlagService(flagConfig);

// Usage in Express
app.get('/api/checkout', (req, res) => {
  const isNewCheckout = flags.isEnabled('NEW_CHECKOUT', req.user.id, {
    region: req.user.region,
    appVersion: req.headers['x-app-version'] as string
  });

  if (isNewCheckout) {
    // Use new checkout implementation
    return res.json(newCheckoutFlow(req.user));
  }

  // Fallback to old checkout
  res.json(legacyCheckoutFlow(req.user));
});

// A/B testing
app.get('/api/dashboard', (req, res) => {
  const variant = flags.getVariant('DASHBOARD_REDESIGN', req.user.id);

  let dashboardData;
  switch (variant) {
    case 'variant_a':
      dashboardData = getDashboardV1(req.user);
      break;
    case 'variant_b':
      dashboardData = getDashboardV2(req.user);
      break;
    default:
      dashboardData = getDashboardLegacy(req.user);
  }

  res.json(dashboardData);
});

// Cleanup: CI/CD check
const validateFlags = () => {
  const now = new Date();
  const deprecatedFlags = flags.getAllFlags().filter(f => f.deprecated);

  for (const flag of deprecatedFlags) {
    if (flag.deprecatedAt && new Date(flag.deprecatedAt) < now) {
      console.warn(`Flag ${flag.name} is past deprecation deadline`);
      // CI can fail here in strict mode
    }
  }
};

validateFlags();
```

## React Integration

```typescript
// hooks/useFeatureFlag.ts
import { flags } from '../flags';

export function useFeatureFlag(name: string, userId: string) {
  const [isEnabled, setIsEnabled] = React.useState(false);

  React.useEffect(() => {
    setIsEnabled(flags.isEnabled(name, userId));
  }, [name, userId]);

  return isEnabled;
}

// Component usage
export function Checkout({ userId }: { userId: string }) {
  const isNewCheckout = useFeatureFlag('NEW_CHECKOUT', userId);

  if (isNewCheckout) {
    return <NewCheckout />;
  }

  return <LegacyCheckout />;
}
```
