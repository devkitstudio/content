## Branching Strategy Comparison

### Trunk-Based Development

```
main:  ─P1─P2─P3─P4─P5─P6─P7─P8

Branch lifetime: < 2 days
Merge frequency: Multiple times per day
Conflicts: Rare (branches too short)
Deployment: Any commit can deploy
```

**Best for:**
- Fast-moving teams (>5 developers)
- Continuous deployment
- Small features

**Adoption:** Google, Netflix, Amazon, Meta

### GitFlow (Traditional)

```
main:       ──P1─────────────P2──
                ↑                 ↑
release:    ──R1──────────────R2──
            ↑                   ↑
develop:    ─D1─D2─D3─D4─D5─D6───

Feature branches: 1-3 weeks each
                  ┌─F1─┐      ┌─F2─┐
develop:  ──D1────┘────D2──────┘────D3
```

**Best for:**
- Scheduled releases (quarterly)
- Multiple parallel versions
- Regulated environments

**Adoption:** Enterprise software, versioned libraries

### GitHub Flow

```
main:   ─P1─P2─P3─P4─P5─P6─

Feature:    ──F1──┐
                  └─P2 (merge + deploy)

Feature:       ──F2──┐
                      └─P4 (merge + deploy)

Branch lifetime: 1 day to 1 week
Merge frequency: Once per day
Conflicts: Occasional
Deployment: After merge
```

**Best for:**
- Web applications
- Continuous deployment
- Teams 3-20 developers

**Adoption:** GitHub, GitHub-like teams

## Comparison Table

| Aspect | Trunk-Based | GitHub Flow | GitFlow |
|--------|------------|-------------|---------|
| Branch lifetime | < 1 day | 1-5 days | 2+ weeks |
| Merge conflicts | Very rare | Rare | Common |
| Deployment frequency | Multiple/day | 1-2x/day | 1x/quarter |
| Learning curve | Easy | Easy | Complex |
| Team size | 5+ | 3-20 | 5-50 |
| Version complexity | Single version | Single version | Multiple versions |
| CI/CD requirements | Strong | Strong | Moderate |
| Rollback capability | Easy | Easy | Hard |
| Feature completeness | Via flags | Via flags | Via branches |

## Merge Conflict Comparison

**Scenario: Two teams, both modify auth.ts**

### Trunk-Based (1 day apart)
```
Team A: Modifies auth.ts, merges at 10am
Team B: Rebase at 9:59am (before A merges)
  - Gets latest from A (10 lines changed)
  - No conflicts (small, focused changes)
  - Merge takes 2 seconds
```

### GitHub Flow (3 days apart)
```
Team A: Branch created Mon 10am, merges Wed 3pm
Team B: Branch created Mon 11am, merges Wed 2pm
  - B merged first, A must rebase
  - 3 days of divergence
  - Likely conflicts in auth.ts
  - 30 minutes to resolve
```

### GitFlow (2 weeks apart)
```
Team A: Feature branch for 2 weeks
Team B: Feature branch for 2 weeks
  - Both based on develop from 2 weeks ago
  - Merge A first: develops changes
  - Merge B: conflicts with A + 100 more changes
  - 4-8 hours of conflict resolution
```

## Migration Path

```
GitFlow → GitHub Flow → Trunk-Based

Phase 1: Improve CI/CD
- All builds must be fast
- All tests must run automatically
- Deploy must be one-click

Phase 2: Shorten branches
- GitFlow: feature → develop → release
- GitHub Flow: feature → main
- Reduce from 2+ weeks to 5 days

Phase 3: Enforce 24-hour rule
- Max branch age: 1 day
- Daily rebases mandatory
- Early PRs for feedback

Phase 4: Add feature flags
- Disable incomplete features at runtime
- Don't need branch as feature container

Phase 5: Multiple deployments per day
- Confident in automated tests
- Roll back is safe and fast
```

## Team Size Impact

```
1-2 developers:
- Any method works
- Trunk-based simplest

3-5 developers:
- GitHub Flow recommended
- Trunk-based if strong CI/CD

5-20 developers:
- GitHub Flow optimal
- Trunk-based if mature
- GitFlow if multiple releases

20+ developers:
- GitFlow with discipline
- Trunk-based with very strong practices
- GitHub Flow with teams split by component
```

## Risk Assessment

| Risk | Trunk-Based | GitHub Flow | GitFlow |
|------|------------|------------|---------|
| Broken main branch | Medium | Low | Very Low |
| Integration hell | Very Low | Very Low | Very High |
| Deployment failure | Low | Low | Medium |
| Rollback difficulty | Very Low | Low | High |
| Code quality | Depends on tests | Depends on tests | Depends on discipline |
