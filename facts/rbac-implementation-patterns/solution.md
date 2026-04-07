# Role-Based Access Control (RBAC)

## Core Concepts

### Users, Roles, and Permissions

```
User: John (Engineering Manager)
  ↓
Role: engineering_manager
  ↓
Permissions:
  - users.read
  - users.create
  - projects.read
  - projects.edit
```

### Permission Model

**Principle**: Fine-grained permissions > coarse roles

```
User has Role(s)
Role has Permission(s)
Permission grants access to Resource.Action

Deny by default:
- No access unless explicitly granted
- Safer than allow-by-default
```

## RBAC Hierarchy

### Level 1: Simple Roles
```
- admin: Full access
- user: Basic access
- guest: Read-only access
```

### Level 2: Department-Based Roles
```
- admin: All permissions
- engineering.manager: Manage team, view code
- engineering.developer: View code, create PRs
- marketing.manager: Manage campaigns
- hr.recruiter: Manage job postings
```

### Level 3: Fine-Grained Permissions
```
users.read          - View user list
users.create        - Create new user
users.delete        - Delete user
projects.read       - View projects
projects.edit       - Edit project details
data.export         - Export sensitive data
system.config       - Change system settings
```

## Database Schema

```sql
-- Users table
CREATE TABLE users (
  id INT PRIMARY KEY,
  email VARCHAR(255) UNIQUE,
  name VARCHAR(255),
  created_at TIMESTAMP
);

-- Roles table
CREATE TABLE roles (
  id INT PRIMARY KEY,
  name VARCHAR(50) UNIQUE,
  description TEXT,
  created_at TIMESTAMP
);

-- Permissions table
CREATE TABLE permissions (
  id INT PRIMARY KEY,
  name VARCHAR(100) UNIQUE,
  resource VARCHAR(50),
  action VARCHAR(50),
  description TEXT
);

-- Role-Permission mapping (Many-to-Many)
CREATE TABLE role_permissions (
  role_id INT,
  permission_id INT,
  PRIMARY KEY (role_id, permission_id)
);

-- User-Role mapping (Many-to-Many)
CREATE TABLE user_roles (
  user_id INT,
  role_id INT,
  assigned_at TIMESTAMP,
  PRIMARY KEY (user_id, role_id)
);

-- Audit log
CREATE TABLE audit_logs (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT,
  action VARCHAR(255),
  resource VARCHAR(255),
  result VARCHAR(10), -- 'allowed' or 'denied'
  timestamp TIMESTAMP,
  ip_address VARCHAR(45)
);
```

## Permission Naming Conventions

```
{resource}.{action}

Examples:
- users.read
- users.create
- users.update
- users.delete
- projects.create
- projects.delete
- reports.export
- system.config
- team.manage
```

## Common RBAC Patterns

### Pattern 1: User can perform action on own resource
```
User A edits Project A (owner)
User B cannot edit Project A

Condition: user.id == resource.owner_id
```

### Pattern 2: Role-based access to resource type
```
Role: admin
Permissions: *.* (all resources, all actions)

Role: viewer
Permissions: *.read (all resources, read-only)
```

### Pattern 3: Hierarchical roles
```
admin > manager > member > viewer

A manager can do everything a member can, plus more
```

### Pattern 4: Team-scoped permissions
```
User belongs to Team A
User can edit Team A projects only
User cannot access Team B projects
```

### Pattern 5: Organizational hierarchy
```
CEO
  ├─ VP Engineering
  │   ├─ Engineering Manager
  │   │   └─ Engineers (5)
  │   └─ QA Lead
  │       └─ QA Engineers (3)
  └─ VP Marketing
      └─ Marketing Manager
          └─ Marketing Specialists (4)

Each level has different permissions
```

## Enforcement Points

### 1. API/Route Level
```
Check permission before route handler executes
Block requests without permission
```

### 2. Business Logic Level
```
Validate permission even in internal functions
Prevent accidental bypasses
```

### 3. Database Level
```
Query filters ensure users only see data they own
```

### 4. UI Level
```
Hide buttons/pages user can't access
Prevents confusion, not security
```

## Best Practices

- Start with broad roles, narrow down
- Use permission inheritance where sensible
- Audit all permission changes
- Implement time-limited elevated permissions
- Require approval for sensitive actions
- Separate read and write permissions
- Use deny lists for security-critical actions
