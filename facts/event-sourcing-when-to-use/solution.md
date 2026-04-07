# Event Sourcing: The Complete Picture

## What Is Event Sourcing?

**Traditional approach:** Store state
```sql
UPDATE users SET balance = 500 WHERE id = 1;
-- History is lost. You only know current balance.
```

**Event sourcing:** Store all state changes as events
```
Event 1: UserCreated(id=1, email="user@example.com")
Event 2: DepositMade(user_id=1, amount=1000)
Event 3: WithdrawalMade(user_id=1, amount=500)
Event 4: WithdrawalMade(user_id=1, amount=0)  // failed
Event 5: DepositMade(user_id=1, amount=500)

Current state: Replay all events → balance = 1000
```

## The Architecture

```
┌───────────────────────────────┐
│   Command (Action)            │
│   WithdrawMoney(user=1, amt=100)
└────────────┬──────────────────┘
             │
             ▼
┌───────────────────────────────┐
│   Command Handler             │
│   - Validate business logic   │
│   - Generate event            │
└────────────┬──────────────────┘
             │
             ▼
┌───────────────────────────────┐
│   Event Store (Append-only)   │
│   MoneyWithdrawn(..event..)   │
└────────────┬──────────────────┘
             │
        ┌────┴─────────────────┐
        │                      │
        ▼                      ▼
┌──────────────┐    ┌─────────────────┐
│ Projections  │    │ Analytics/Audit │
│ (Read models)│    │ (Event stream)  │
│ User balance │    │ Full history    │
└──────────────┘    └─────────────────┘
```

## Core Concepts

### 1. Events (Immutable Facts)

```python
@dataclass
class Event:
    event_id: str
    event_type: str
    aggregate_id: str  # what entity it's about
    timestamp: datetime
    data: dict
    version: int  # event sequence number

# Examples
MoneyDeposited = Event(
    event_id="evt_123",
    event_type="MoneyDeposited",
    aggregate_id="user_1",
    timestamp=datetime.now(),
    data={"amount": 500, "source": "bank_transfer"},
    version=5
)
```

### 2. Aggregate Root (Entity)

```python
class BankAccount:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.balance = 0
        self.events = []
        self.version = 0

    def deposit(self, amount: float, source: str):
        """Command: deposit money"""
        if amount <= 0:
            raise ValueError("Amount must be positive")

        # Generate event
        event = MoneyDeposited(
            amount=amount,
            source=source,
            timestamp=datetime.now()
        )
        self.events.append(event)

        # Apply to state
        self.balance += amount
        self.version += 1

    def withdraw(self, amount: float):
        """Command: withdraw money"""
        if amount > self.balance:
            raise InsufficientFunds()

        event = MoneyWithdrawn(
            amount=amount,
            timestamp=datetime.now()
        )
        self.events.append(event)

        self.balance -= amount
        self.version += 1

    def apply_event(self, event):
        """Replay event to reconstruct state"""
        if isinstance(event, MoneyDeposited):
            self.balance += event.amount
        elif isinstance(event, MoneyWithdrawn):
            self.balance -= event.amount
```

### 3. Event Store (Append-Only Log)

```python
class EventStore:
    def __init__(self, db_connection):
        self.db = db_connection

    def append(self, aggregate_id: str, events: list):
        """Add events (only append, never update)"""
        for event in events:
            self.db.events.insert_one({
                'aggregate_id': aggregate_id,
                'event_type': event.type,
                'data': event.data,
                'timestamp': event.timestamp,
                'version': event.version
            })

    def get_events(self, aggregate_id: str, from_version: int = 0):
        """Retrieve all events for an aggregate"""
        return self.db.events.find({
            'aggregate_id': aggregate_id,
            'version': {'$gt': from_version}
        }).sort('version', 1)

    def rebuild_aggregate(self, aggregate_id: str):
        """Reconstruct current state from events"""
        account = BankAccount(aggregate_id)
        events = self.get_events(aggregate_id)

        for event in events:
            account.apply_event(event)

        return account
```

### 4. Projections (Read Models)

```python
class UserBalanceProjection:
    """Read model: User's current balance"""
    def __init__(self, db):
        self.db = db

    def handle_event(self, event):
        """Update read model when event occurs"""
        if event.type == "MoneyDeposited":
            self.db.user_balances.update_one(
                {'user_id': event.aggregate_id},
                {'$inc': {'balance': event.data['amount']}}
            )
        elif event.type == "MoneyWithdrawn":
            self.db.user_balances.update_one(
                {'user_id': event.aggregate_id},
                {'$inc': {'balance': -event.data['amount']}}
            )


class TransactionHistoryProjection:
    """Read model: User's transaction history"""
    def __init__(self, db):
        self.db = db

    def handle_event(self, event):
        """Create read-friendly transaction record"""
        if event.type in ("MoneyDeposited", "MoneyWithdrawn"):
            self.db.transactions.insert_one({
                'user_id': event.aggregate_id,
                'type': event.type,
                'amount': event.data['amount'],
                'timestamp': event.timestamp,
                'reference': event.data.get('source')
            })
```

## When to Use Event Sourcing

### ✓ Perfect For

1. **Financial systems**
   - Compliance: "Prove every transaction that happened"
   - Audit trail: "Who did what and when"
   - Example: Banking, insurance, payments

2. **Regulatory compliance**
   - GDPR: Need to show data changes over time
   - PCI-DSS: Payment audit trail
   - HIPAA: Medical record changes

3. **Complex business logic**
   - State transitions matter (draft → approved → rejected)
   - Need to replay "what if" scenarios
   - Domain-driven design with rich models

4. **Debugging production issues**
   - "Why did user get charged twice?"
   - Replay events to reproduce exact scenario
   - Don't guess, know what happened

### ✗ Not Ideal For

1. **Simple CRUD apps**
   - Blog: Just store title/content, who cares about history?
   - To-do list: Don't need full audit trail
   - Adds complexity for no benefit

2. **High-throughput systems**
   - Event sourcing can bottleneck writes
   - Event store becomes I/O bound
   - 100K events/sec is hard

3. **Simple reads**
   - If you never ask "what changed?", don't use it
   - Event sourcing is overkill for "show current balance"

4. **When consistency doesn't matter**
   - Social media likes: Approximate is fine
   - Analytics: Don't need exact history
   - Caching: Losing history is acceptable

## Cost-Benefit Analysis

### Benefits

| Benefit | Value |
|---------|-------|
| Audit trail | HIGH for finance/compliance |
| Debugging | HIGH for production incidents |
| Temporal queries | MEDIUM ("balance on Jan 1?") |
| Event replay | MEDIUM for testing "what if" |
| Regulatory proof | HIGH for compliance domains |

### Costs

| Cost | Impact |
|------|--------|
| Complexity | HIGH - harder to reason about |
| Operational | MEDIUM - event store maintenance |
| Storage | MEDIUM - store all events forever |
| Performance | LOW-MEDIUM (with snapshots) |
| Team knowledge | HIGH - not everyone knows ES |

## Decision Tree

```
Do you need to know exact history of state changes?
├─ NO → Regular database (simpler!)
│
├─ YES
│  ├─ Why? For compliance/audit?
│  │  ├─ YES → Event sourcing is good
│  │  ├─ NO  → Just add audit_log table (simpler!)
│  │
│  ├─ Will you replay events to debug?
│  │  ├─ YES → Event sourcing helps
│  │  ├─ NO  → Don't bother
│  │
│  └─ Do temporal queries matter?
│     ("What was the state on 2024-01-01?")
│     ├─ YES → Event sourcing wins
│     └─ NO  → Simple audit log enough
```

## Comparison: Simple Audit Log vs Event Sourcing

| Feature | Audit Log | Event Sourcing |
|---------|-----------|-----------------|
| Track changes | Yes | Yes |
| Rebuild history | Manual queries | Automatic replay |
| Complexity | Low | High |
| Compliance | ✓ | ✓✓ |
| Debugging | Okay | Excellent |
| Performance | Fast | Slower (rebuilds) |
| Team effort | 1 week | 4 weeks |

## Alternative: Simple Audit Approach

**If you don't need full event sourcing:**

```python
# Just add audit columns
class User:
    id: str
    email: str
    balance: float
    # Add these:
    updated_at: datetime
    updated_by: str
    change_reason: str

# Log changes separately
class AuditLog:
    user_id: str
    field: str
    old_value: any
    new_value: any
    timestamp: datetime
    user_id: str
    reason: str
```

**Benefits:**
- ✓ Simple to implement
- ✓ Can answer "what changed?"
- ✓ 1 week vs 4 weeks
- ✗ Can't replay to old state
- ✗ More manual queries

## When Stripe, Apple Pay, AWS Use Event Sourcing

**Financial domain = Event sourcing mandatory**

Why?
- "User claims they didn't get charged, refund them"
  - Event sourcing: Replay events, prove what happened
  - Normal DB: "balance is negative, not our fault"
- "Prove this transaction happened for tax audit"
  - Event sourcing: Full immutable proof
  - Normal DB: "table was updated, can't prove when"
- "Regulatory asks for data from Jan 2020"
  - Event sourcing: Replay to 2020 state
  - Normal DB: Hope you kept backups

## The Right Approach

```
Most startups: Regular database + audit_log table
├─ Fast to build
├─ Good enough for 99% of use cases
└─ Zero learning curve

Financial/Compliance companies: Event sourcing
├─ Proven pattern in domain
├─ Worth the investment
└─ Solves real problems

What NOT to do:
├─ Event sourcing for simple blog
├─ Event sourcing because "it sounds cool"
├─ Event sourcing without understanding trade-offs
```

## Red Flags: Cargo Cult Event Sourcing

```
"We implemented event sourcing because..."
├─ "Netflix uses it" → WRONG (different scale)
├─ "It's the best practice" → WRONG (domain-dependent)
├─ "Future-proofing" → WRONG (YAGNI)
├─ "Microservices need it" → WRONG (decoupling tool)
│
✓ "We need compliance audit trail"
✓ "Users dispute charges, we need proof"
✓ "Debugging requires knowing exact history"
```
