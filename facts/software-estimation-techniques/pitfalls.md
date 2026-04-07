## Anti-Patterns

### 1. Anchoring

```
Manager: "How long for SSO?"
Developer: "Umm... 3 weeks?"
Manager: "We need it in 1 week"
Developer: "Ok... maybe 1 week then"

Problem: Estimate changed based on external pressure, not technical reality
Better: Estimate based on scope, negotiate scope if timeline is tight
```

### 2. Optimism Bias

```
Estimate: "2 weeks"

Hidden assumptions (you didn't think about):
- I'll write perfect code (won't need debugging)
- No meetings/interruptions
- All dependencies are ready
- No unexpected edge cases
- Single attempt (no rewrites)

Reality factor: 2-3x actual
```

### 3. Ignoring Dependencies

```
Task: Deploy new API endpoint (estimated: 3 days)

Actual breakdown:
- Code: 1 day
- Waiting for DB team to create schema: 5 days (blocked!)
- Review: 2 days
- Testing: 1 day
- Deployment: 1 day
Total: 10 days (blocked most of the time)

Solution: Break down and identify dependencies first
```

### 4. Confusing Effort with Duration

```
Task: "Refactor authentication module"

Effort: 40 hours (5 days of actual work)
Duration: 3 weeks (due to:
  - Other projects: 20 hours
  - Code review: 2 days
  - Bug fixes discovered: 3 days
  - Deployment: 1 week
)

Always distinguish:
- How long will it take you? (effort)
- When will it be done? (duration with delays)
```

### 5. False Precision

```
WRONG: "This will take exactly 47 hours"
(implies certainty you don't have)

BETTER:
- T-shirt size: M (5-8 days)
- Range: 1-2 weeks (accounting for unknowns)
- Dependencies: Requires auth team approval
```

## Real-World Failure Patterns

### Pattern 1: Hidden Scope

```
"Add dark mode"

Estimated: 3 days (CSS changes)
Actual: 2 weeks

Forgotten:
- Update 50+ components individually
- Test all color contrasts (accessibility)
- Update color schemes in database
- Migrate user preferences
- Test on different browsers
- Update design system
```

### Pattern 2: Integration Surprises

```
"Integrate payment provider"

Estimated: 5 days
Actual: 4 weeks

Surprises:
- API is slower than documented (timing out)
- Webhook signatures don't match docs
- Support tickets required (API credentials issue)
- PCI compliance requirements
- 3D Secure flow not documented
- Settlement delays unknown
```

### Pattern 3: Technical Debt

```
"Add new feature"

Estimated: 1 week
Actual: 3 weeks

Hidden work:
- Legacy code doesn't support new patterns
- Refactoring required to add feature cleanly
- Old tests fail in new context
- Documentation is outdated
```

## Warning Signs

```
🚩 Estimate is a round number (2 weeks, 1 month, 3 days)
🚩 No breakdown (one number for whole feature)
🚩 No dependencies identified
🚩 Optimistic tone ("should be straightforward")
🚩 No "what if?" analysis
🚩 Estimate ignored unknowns ("assuming X works")
🚩 Created under time pressure
🚩 Single person estimated (no sanity check)
```

## Estimation Smells

| Smell | Problem | Fix |
|-------|---------|-----|
| "About 2 weeks" | Vague | Make it "8-13 days with risks" |
| "A month maybe?" | Unsure | Break down further |
| "Could be 5-50 days" | Huge range | Reduce uncertainty first |
| "Whatever you want" | Gave up | Pair with someone experienced |
| "Same as last time" | No thinking | Re-estimate (different context) |
