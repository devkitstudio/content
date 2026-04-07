## The Problem: GitFlow with Long Branches

```
Trunk (main):         ─────────────────────────────
                      ↓
Feature A (3 weeks):  ──branch─────────────────┬──merge
                              ↑ diverges, conflicts grow

Feature B (2 weeks):  ────────branch──────────┬──merge
                              ↑ diverges, conflicts grow

Result: Integration hell
- Merge conflicts when merging A
- Merge conflicts when merging B
- Merge conflicts when merging B after A
- Each merge takes 1-2 days
```

## Trunk-Based Development

```
Trunk (main):        ─P1─P2─P3─P4─P5─P6─P7─P8─P9

Branches (max 1-2 days):
Feature A:           ───┐
                        └─P3 (merged within 24 hours)

Feature B:               ───┐
                            └─P5 (merged within 24 hours)

Hotfix C:                   ──┐
                              └─P6 (merged immediately)

Benefits:
- Max code divergence = 24 hours work
- Tiny merges (often zero conflicts)
- Can integrate features continuously
- Can deploy at any time
```

## Trunk-Based Workflow

```bash
# Step 1: Create short-lived branch
git checkout -b feature/user-auth
# Work for 1-2 days max

# Step 2: Keep branch fresh (rebase daily)
git fetch origin
git rebase origin/main
# Prevents divergence

# Step 3: When ready, create small PR
# Include: feature description, testing notes, risks

# Step 4: Code review (fast, small PR)
# Reviewer can understand in 20 minutes

# Step 5: Merge and delete branch
git merge feature/user-auth
git push origin main
git branch -d feature/user-auth

# Step 6: Deploy (can be done immediately)
./deploy.sh
```

## Preventing Long Branches

**Rule 1: Maximum 1-2 days before merge**

```
Day 1: Started feature/search
  - 4 hours work
  - Synced with main (rebase)

Day 1.5: Working on feature/search
  - 8 hours work total
  - Ready to merge? YES → merge
  - Not ready? Rebase again

Day 2: If still not done
  - Split into smaller PRs
  - Ship what's done
  - Continue with follow-up PR
```

**Rule 2: Rebase daily**

```bash
# Every morning or before committing
git fetch origin
git rebase origin/main

# If conflicts, fix now (small changes)
# Don't let them accumulate for 2 weeks
```

**Rule 3: Feature flags for incomplete work**

```typescript
// Hide incomplete feature behind flag
export function getUserProfile(userId: string) {
  const newUI = featureFlags.get('new-profile-ui');

  if (newUI && isUserInBetaGroup(userId)) {
    return <NewProfileUI userId={userId} />;
  }

  return <LegacyProfileUI userId={userId} />;
}

// Even if incomplete, ship to main
// Gradually roll out to real users
// Roll back instantly if bugs
```

## Code Review in Trunk-Based

**Fast reviews because:**
- PR is small (100-300 lines, not 2000)
- Can review in 20 minutes
- Easy to spot issues

```
Bad (2000 line PR):
Reviewer: "This is huge, I'll look at it later"
Reviewer after 1 week: "Did you already merge this? Too late."

Good (100 line PR):
Reviewer: "Done, approved in 15 minutes"
```

## Feature Flags for Control

```typescript
// Enable/disable without code changes
const config = {
  features: {
    'new-search': { enabled: true, rollout: 0.5 },     // 50% of users
    'dark-mode': { enabled: false, rollout: 0 },       // Disabled
    'advanced-analytics': { enabled: true, rollout: 1 }, // 100%
  }
};

function SearchComponent() {
  const newSearchEnabled = isFeatureEnabled('new-search');

  if (newSearchEnabled) {
    return <NewSearch />;
  }
  return <OldSearch />;
}

// At runtime, change config → instant feature toggle
// No deploy needed, no branch merge needed
```

## Deployment Pipeline

```
Trunk-Based allows:
- Commit to main at 10am
- Pass tests at 10:05am
- Deploy to staging at 10:10am
- Deploy to prod at 10:30am (after QA approval)

Total: 30 minutes from code to production

GitFlow takes 3-5 days due to integration conflicts
```

## Handling Concurrent Work

```
Multiple teams on same main branch?

Team A: Building search       (2 days)
Team B: Building recommendations (2 days)
Team C: Building notifications (1 day)

With trunk-based:
Mon 10am: All merge to main one by one (small PRs)
Mon 4pm: All in production (all 3 features)

With GitFlow:
Mon 10am: All start long-lived branches
Wed 5pm: Team C merges (done, but blocked on A & B)
Thu 2pm: Team A merges (1 day conflicts with B)
Fri 10am: Team B merges (2 days conflicts with A)
Next week: All finally integrated

Cost: 5 days integration vs 0 days
```

## CI/CD Requirements

Trunk-based requires strong CI/CD:

```yaml
# GitHub Actions example
name: Trunk-Based CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: npm install
      - run: npm test              # Must be fast (<10 min)
      - run: npm run lint
      - run: npm run build
  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - run: ./deploy-staging.sh
      - run: ./smoke-tests.sh
      - run: ./deploy-prod.sh
```

**Everything must be automated and fast:**
- Tests run in < 10 minutes
- Deploy in < 5 minutes
- Can't have manual approval gates
