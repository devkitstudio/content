# When NOT to Use Event Sourcing

## The Complexity Tax

Event sourcing adds complexity everywhere:

```python
# Simple update (normal DB)
user.balance = 500
db.save(user)
# ✓ 2 lines, everyone understands

# Event sourced (ES)
class Account:
    def deposit(self, amount):
        if amount <= 0:
            raise ValueError()
        event = MoneyDeposited(amount)
        self.apply_event(event)
        self.events.append(event)

    def apply_event(self, event):
        if isinstance(event, MoneyDeposited):
            self.balance += event.amount

    def get_balance(self):
        # Rebuild from all events every time!
        account = Account()
        for event in self.event_store.get_events(self.id):
            account.apply_event(event)
        return account.balance

# ✗ 40 lines, needs careful thought, easy to break
```

## Real Problems

### 1. Event Schema Evolution

```python
# Version 1
event = {
    "type": "MoneyDeposited",
    "amount": 100,
    "timestamp": "2024-01-01"
}

# Months later, you realize you need source
event_v2 = {
    "type": "MoneyDeposited",
    "amount": 100,
    "source": "bank_transfer",  # New field!
    "timestamp": "2024-01-01"
}

# Problem: Old events don't have "source"
# Now replay breaks: KeyError: 'source'

# Solution: Complex upcasting/migration code
def upcast_v1_to_v2(event):
    if 'source' not in event:
        event['source'] = 'unknown'
    return event

# This gets out of hand fast
```

### 2. Performance Cliffs

```python
# First month: 100 events
account = Account()
for event in event_store.get_events(user_id):
    account.apply_event(event)  # 100 iterations

# Year 2: 100,000 events
for event in event_store.get_events(user_id):
    account.apply_event(event)  # 100,000 iterations!

# Suddenly slow. Need snapshots.
# More complexity.
```

**Solution: Snapshots (but adds even more complexity)**
```python
# "Every 1000 events, save state snapshot"
snapshot = {
    "balance": 50000,
    "at_version": 5000
}

# Replay only from snapshot
for event in event_store.get_events(user_id, from_version=5001):
    account.apply_event(event)  # Only 100 new events
```

### 3. Eventually Consistent Reads

```python
# User deposits $100
user.deposit(100)

# Immediately ask: "What's my balance?"
GET /api/balance
↓
# Oops, projection not updated yet
# Returns $0 instead of $100
# User sees "Balance: $0" and panics
```

**You need to handle eventual consistency:**
- Show "Balance: $100 (pending)"
- Poll until projection catches up
- More complex UX

### 4. Deleting User Data (GDPR)

```
GDPR says: "User asks to delete their data"

Event sourcing says:
"LOL, we have an append-only log of everything they did"

Solution: Encrypt sensitive data, delete keys
But:
- Complex to implement correctly
- Vulnerable to key recovery attacks
- Not genuine deletion
```

**Most compliance lawyers hate this.**

### 5. Debugging Event Storms

```python
# Something goes wrong and events flood in
MoneyDeposited x100000
UserUpdated x50000
AccountLocked x1000

# "Why did this happen?"
# You have to trace through 150K events
# And understand which event caused the cascade

# Normal DB: One row changed, clear cause
# ES: Hidden in event storm
```

## Real Horror Stories

### Case 1: Banking Startup

```
Year 1: Built beautiful event sourcing
Year 2: Hit event schema migration bug
Result: Couldn't replay events, corrupted state
Cost: $2M data recovery, 2 months down
```

### Case 2: SaaS Company

```
Customer: "I was charged twice in 2022"
Engineer: "Let me replay to 2022..."
System: *takes 30 minutes to rebuild 100K events*
Customer: "Why is this slow?"
Reality: ES scalability wasn't thought through
```

### Case 3: Fintech

```
Implemented event sourcing
Realized GDPR compliance requires deletion
Had to build second system for deletion
Now maintaining two architectures
Cost: 4x more complex
```

## The Simple Alternative

```python
# Instead of event sourcing:
class Account:
    id: str
    balance: float
    updated_at: datetime
    version: int  # Optimistic locking

# Audit table (simple)
class AuditLog:
    account_id: str
    old_balance: float
    new_balance: float
    reason: str
    timestamp: datetime
    user_id: str

# Queries:
SELECT * FROM audit_log WHERE account_id = 1
# Shows all balance changes

SELECT balance FROM accounts WHERE id = 1
# Shows current balance (fast!)

# Still compliant, 10x simpler
```

## When This Approach Fails

```
If you need:
├─ Temporal queries ("balance on Jan 1?")
│  └─ Use: Bitemporal modeling (simpler than ES)
├─ Replaying exact state at any point
│  └─ Use: Point-in-time backups instead
├─ Event-driven analytics
│  └─ Use: Change Data Capture (simpler than ES)
└─ Debugging "what if" scenarios
   └─ Use: Logging + analysis tools
```

## The Truth

**"Event sourcing is the pain of microservices without the benefits."** - Many experienced engineers

### It's good for:
- Financial institutions (10+ year track record)
- Regulated industries (compliance burden justifies cost)
- Domain-driven design (rich domain models)

### It's bad for:
- 95% of business logic
- Startups (wrong problem to solve)
- Teams without event sourcing expertise
- Simple applications

## Red Flags Your Team Isn't Ready

```
"Let's use event sourcing because..."
├─ "Netflix does it" → You're not Netflix
├─ "It's modern" → Complexity ≠ Modern
├─ "Future-proof" → YAGNI
├─ "Might need it" → Build when needed
├─ "Looks cool" → Career-limiting risk
│
If your team:
├─ Doesn't know ES well → Will build it wrong
├─ Never debugged ES issues → Will suffer
├─ Has >5 person team → Over-complication tax
├─ Ship fast is goal → Slows you down 3x
```

## The Honest Truth

**Most teams that adopt event sourcing regret it.**

Reasons:
1. Complexity slows shipping
2. Event schema changes are painful
3. Debugging is harder, not easier
4. Operational overhead is high
5. Few people understand it well
6. Team churn means lost knowledge
7. New engineer onboarding: 6 weeks to understand

**The solution:**
- Use for financial/compliance systems only
- Use simple audit tables for everything else
- Adopt ES if you hit the problem, not "just in case"
