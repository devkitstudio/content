## Why Estimates Fail

```
Reasons for 10x overrun:
1. Scope creep (40% of delays)
2. Technical unknowns (30%)
3. Dependencies blocking (15%)
4. Optimism bias (10%)
5. Interruptions (5%)
```

**Key insight:** Estimates assume you know what you're building. You usually don't.

## The Cone of Uncertainty

```
Initial estimate: ±50% accuracy
↓ (as you learn)
↓ (sprint planning: ±25%)
↓ (design complete: ±10%)
↓ (coding in progress: ±5%)
Final reality: ±0%
```

**Don't estimate upfront.** Estimate as you learn.

## Estimation Techniques

### 1. T-Shirt Sizing (Agile)

```
Feature list:
- User login: XS (1-2 days)
- Password reset: S (3-5 days)
- OAuth integration: M (1 week)
- Admin dashboard: L (2-3 weeks)
- Analytics engine: XL (4+ weeks)

Benefits:
- No false precision (2 weeks vs 9 days)
- Relative, not absolute
- Accounts for unknowns

Conversion to days (empirical):
XS = 2 days, S = 4 days, M = 8 days, L = 16 days, XL = 32 days
(Adjust based on your team's velocity)
```

### 2. Planning Poker

Team estimates together (prevents anchoring):

```
Feature: Implement payment processing

Each person estimates:
Developer A: 8 points
Developer B: 13 points
Developer C: 5 points

Discuss the outliers:
- C thinks it's 5 because they use an existing SDK
- B thinks 13 because they want to add fraud detection
- A settles at 8

Final: 8 points (highest confidence)
```

**Why it works:** Collective knowledge > individual estimate

### 3. Three-Point Estimation

```
Feature: Real-time notifications

Optimistic (best case): 3 days
  - Uses existing WebSocket library
  - Simple schema

Realistic (expected): 8 days
  - Some edge cases (reconnection, offline queuing)
  - Testing

Pessimistic (if wrong): 20 days
  - Browser compatibility issues
  - Database migration needed
  - Real-time sync with existing data

Calculation:
Expected = (Optimistic + 4×Realistic + Pessimistic) / 6
         = (3 + 4×8 + 20) / 6
         = (3 + 32 + 20) / 6
         = 55 / 6
         ≈ 9 days

Add buffer: 9 × 1.25 = 11 days (confidence increases)
```

### 4. Historical Velocity

```
Past 5 sprints:
Sprint 1: 45 story points completed
Sprint 2: 38 story points
Sprint 3: 52 story points
Sprint 4: 41 story points
Sprint 5: 48 story points

Average velocity: (45 + 38 + 52 + 41 + 48) / 5 = 44.8 ≈ 45 points/sprint

Next sprint roadmap:
- Feature A: 13 points
- Feature B: 8 points
- Feature C: 21 points
- Feature D: 5 points
Total: 47 points

Prediction: Can likely complete all in 1 sprint (45-50 capacity)

Benefits:
- Data-driven
- Accounts for interruptions, meetings, etc.
```

### 5. Fibonacci Estimation (Agile)

```
Use Fibonacci sequence: 1, 2, 3, 5, 8, 13, 21, 34, 55...

Why Fibonacci?
- Forces you to pick (can't say "7")
- Larger numbers have more uncertainty
- Reflects growing complexity

Example breakdown:
- Add validation: 3 points
- Add error handling: 5 points
- Add logging: 2 points
- Add testing: 8 points
Total: 18 points (round to 21)
```

## Practical Framework

```python
def estimate_feature(description):
    """
    4-step estimation process
    """
    # Step 1: Break it down
    tasks = break_down(description)

    # Step 2: T-shirt size each task
    sizes = {
        'xs': 1,   # < 1 day
        's': 3,    # 1-2 days
        'm': 5,    # 2-3 days
        'l': 13,   # 1 week
        'xl': 21   # 2+ weeks
    }

    total = sum(sizes[estimate_task(t)] for t in tasks)

    # Step 3: Check estimates in planning poker
    team_estimates = [get_estimate_from(person) for person in team]

    # Step 4: Account for unknowns
    unknown_factor = 1.25  # 25% buffer for unknowns
    final_estimate = total * unknown_factor

    return final_estimate

# Example
tasks = [
    'Database schema',      # S (3)
    'API endpoint',         # M (5)
    'Error handling',       # S (3)
    'Tests',                # M (5)
    'Deploy & monitor',     # S (3)
]
# Total: 19 points ≈ 13 + 8 = 2 weeks realistic
```

## When Estimates Go Wrong

```
Estimated: 2 weeks
Actual: 8 weeks

Post-mortem reveals:
✗ Didn't account for:
  - Legacy database migration (1 week)
  - Third-party API integration (2 weeks)
  - Security review (1 week)
  - Testing edge cases (1 week)
  - Performance optimization (1 week)

Total hidden work: 6 weeks

Lesson: Ask "What could go wrong?" before estimating
```

## Better Estimation Practices

1. **Break work down** until tasks are < 3 days
2. **Revisit after 25% complete** (adjust if needed)
3. **Track velocity** (what did you actually complete?)
4. **Use historical data** (past sprints as baseline)
5. **Add buffer** (25-50% depending on uncertainty)
6. **Communicate assumptions** ("assumes no DB changes")
7. **Update as you learn** (estimates are not commitments)
