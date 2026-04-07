## Junior vs Mid vs Senior

### 1. Technical Depth

**Junior:**
- Solves assigned tasks
- Follows patterns shown by others
- Writes code that works

**Mid-level:**
- Solves problems independently
- Understands why patterns matter
- Writes code that's maintainable

**Senior:**
- Understands systems (across services, teams)
- Chooses patterns based on trade-offs
- Writes code that scales and teaches others

### 2. Problem-Solving

**Junior:**
```
Problem: "Database queries are slow"
Approach: Ask for help / Google the error
Solution: Follow suggestion from senior
```

**Mid-level:**
```
Problem: "Database queries are slow"
Approach: Profile, find N+1 queries, optimize indexes
Solution: Implement caching + query optimization
```

**Senior:**
```
Problem: "Database queries are slow"
Approach: Understand access patterns → Design at system level
Solution:
- Is read-heavy? Add cache layer (Redis)
- Is write-heavy? Denormalize or shard
- Is mix? Add read replicas
- Root cause: Schema wasn't designed for scale
  → Prevents future problems
```

### 3. Ownership

**Junior:** "Here's my PR, ready for review"

**Mid-level:** "I analyzed the problem, considered 3 solutions, here's why this one is best"

**Senior:** "I analyzed the problem across our system. This affects 3 teams. I talked to them. Here's why this is best and how we'll migrate."

### 4. Communication

**Junior:**
- Asks questions when stuck
- Writes straightforward code comments

**Mid-level:**
- Explains trade-offs in code reviews
- Documents decisions in code

**Senior:**
- Sees the big picture
- Explains to non-technical stakeholders
- Anticipates objections
- Writes design documents
- Communicates risk clearly

Example:

```
JUNIOR: "This will take 2 weeks"

MID-LEVEL: "This will take 2 weeks. It requires refactoring
the auth module which is used by 5 services."

SENIOR: "This will take 2 weeks.
Scope: Refactor auth module (used by 5 services)
Risk: Cache invalidation issues (low probability, high impact)
Mitigation: Gradual rollout with feature flags
Dependencies: Need approval from auth and payments teams
Benefits: Reduces bugs in session handling, enables new features
Alternatives: We could patch the issue (1 day, but temporary)"
```

### 5. System Thinking

**Junior:**
- Solves immediate problem
- "How do I make this test pass?"

**Mid-level:**
- Considers impact on related components
- "How does this affect the database schema?"

**Senior:**
- Understands system holistically
- "How does this affect our infrastructure, team structure, hiring, scaling?"

```
Example: Adding feature to payment system

JUNIOR: Adds a new API endpoint
Result: API works but causes database locks during peak traffic

MID-LEVEL: Adds endpoint + optimizes queries + adds caching
Result: Works fine but now has 3 cache invalidation issues

SENIOR: Designs async processing with event queue
Result: Scales to 100x traffic, other teams can integrate
Prevents future performance issues across org
```

### 6. Mentorship

**Junior:** Focuses on own learning

**Mid-level:** Helps teammates on specific tasks

**Senior:**
- Unblocks junior developers
- Reviews code thoughtfully (teaches, not gatekeeps)
- Shares knowledge (talks, docs, design sessions)
- Helps career growth of others

### 7. Making Hard Decisions

**Junior:** "I don't know, what do you think?"

**Mid-level:** "Option A is faster, option B is cleaner. I'll go with B."

**Senior:**
- Gathers data
- Considers long-term impact
- Explains trade-offs clearly
- Makes decision that's right for the org, not just the feature
- Owns the consequences

## Technical Skills Checklist

### Architecture
- [ ] Understand microservices patterns and trade-offs
- [ ] Design for scale (10x, 100x growth)
- [ ] Know when to use different databases (SQL, NoSQL, graphs)
- [ ] Understand distributed systems (CAP, eventual consistency)
- [ ] Design async systems (message queues, event-driven)

### Performance
- [ ] Profile code to find bottlenecks
- [ ] Optimize queries (execution plans, indexes)
- [ ] Design caching strategies (cache invalidation, TTLs)
- [ ] Understand network performance (HTTP/2, TCP, DNS)

### Reliability
- [ ] Design error handling (graceful degradation)
- [ ] Implement monitoring and alerting
- [ ] Design for failure (circuit breakers, retries)
- [ ] Handle edge cases and race conditions
- [ ] Disaster recovery and data consistency

### Code Quality
- [ ] Write maintainable, readable code
- [ ] Design good abstractions
- [ ] Balance YAGNI vs future-proofing
- [ ] Security best practices (injection, auth, encryption)
- [ ] Test coverage and strategy

## Soft Skills

### Influence Without Authority
- Convince team to adopt new patterns
- Lead technical decisions across teams
- Gain buy-in for major refactors

### Strategic Thinking
- Understand business goals
- Align technical decisions with business
- Plan for 1 year, 5 year growth

### Communication
- Explain complex ideas simply
- Present to executives
- Write clear technical proposals
- Handle disagreement professionally

### Mentorship
- Unblock junior developers
- Teach through code review
- Create learning opportunities

## Path to Senior

**Year 1 (Junior):** Build T-shaped skills (broad basics, one specialty)

**Year 2-3 (Mid-level):** Deepen specialty, broaden adjacent skills

**Year 4-5 (Senior):** System thinking, mentorship, influence

**Timeline:** 5-7 years to senior for most people

**What accelerates it:**
- Working on diverse projects (not same system for 5 years)
- Mentors who push you
- Taking on ownership (not just assigned tasks)
- Learning from failures (blameless post-mortems)

**What slows it:**
- Only working on small features
- Not seeking feedback
- Avoiding difficult problems
- Technical tunnel vision (ignoring business)
