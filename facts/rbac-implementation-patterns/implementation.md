# RBAC Implementation Code

## Express Middleware Approach

```typescript
import { Request, Response, NextFunction } from 'express';

// Extend Express Request to include user
declare global {
  namespace Express {
    interface Request {
      user?: User & { permissions: string[] };
    }
  }
}

interface User {
  id: string;
  email: string;
  roles: string[];
}

// Middleware 1: Load user and permissions
const loadUserPermissions = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  try {
    const user = await db.users.findOne({ id: decodeToken(token).userId });
    if (!user) return res.status(401).json({ error: 'User not found' });

    // Load user roles and permissions
    const roles = await db.userRoles.findAll({ where: { userId: user.id } });
    const roleIds = roles.map(r => r.roleId);

    const permissions = await db.rolePermissions.findAll({
      where: { roleId: { $in: roleIds } }
    });

    req.user = {
      ...user,
      permissions: permissions.map(p => p.permissionName)
    };

    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Middleware 2: Check specific permission
const requirePermission = (requiredPermission: string) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    if (!req.user.permissions.includes(requiredPermission)) {
      // Audit denied access attempt
      logAudit({
        userId: req.user.id,
        action: requiredPermission,
        result: 'denied',
        ip: req.ip
      });

      return res.status(403).json({
        error: 'Insufficient permissions',
        required: requiredPermission
      });
    }

    next();
  };
};

// Middleware 3: Role-based access
const requireRole = (requiredRoles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    const hasRole = requiredRoles.some(role => req.user!.roles.includes(role));

    if (!hasRole) {
      return res.status(403).json({
        error: 'Insufficient role',
        required: requiredRoles
      });
    }

    next();
  };
};

// Usage in routes
app.use(loadUserPermissions);

app.delete('/api/users/:id', requirePermission('users.delete'), async (req, res) => {
  const userId = req.params.id;

  // Audit log
  await logAudit({
    userId: req.user!.id,
    action: 'users.delete',
    resource: `User:${userId}`,
    result: 'allowed',
    ip: req.ip
  });

  await db.users.delete({ id: userId });
  res.json({ success: true });
});

app.post('/api/system/config', requireRole(['admin']), async (req, res) => {
  // Only admins can update system config
  const config = req.body;
  await db.systemConfig.update(config);
  res.json({ success: true });
});
```

## Database Schema Implementation

```typescript
import { DataTypes, Model, Sequelize } from 'sequelize';

const sequelize = new Sequelize(process.env.DATABASE_URL);

// User model
class User extends Model {
  declare id: string;
  declare email: string;
  declare name: string;
}

User.init(
  {
    id: { type: DataTypes.UUID, primaryKey: true },
    email: { type: DataTypes.STRING, unique: true },
    name: DataTypes.STRING
  },
  { sequelize, tableName: 'users' }
);

// Role model
class Role extends Model {
  declare id: string;
  declare name: string;
  declare description: string;
}

Role.init(
  {
    id: { type: DataTypes.UUID, primaryKey: true },
    name: { type: DataTypes.STRING, unique: true },
    description: DataTypes.TEXT
  },
  { sequelize, tableName: 'roles' }
);

// Permission model
class Permission extends Model {
  declare id: string;
  declare name: string;
  declare resource: string;
  declare action: string;
}

Permission.init(
  {
    id: { type: DataTypes.UUID, primaryKey: true },
    name: { type: DataTypes.STRING, unique: true },
    resource: DataTypes.STRING,
    action: DataTypes.STRING
  },
  { sequelize, tableName: 'permissions' }
);

// Associations
User.hasMany(Role, { through: 'user_roles', as: 'roles' });
Role.hasMany(User, { through: 'user_roles', as: 'users' });
Role.hasMany(Permission, { through: 'role_permissions', as: 'permissions' });
Permission.hasMany(Role, { through: 'role_permissions', as: 'roles' });

// Seed data
async function seedRoles() {
  const adminRole = await Role.create({
    id: 'role_admin',
    name: 'admin',
    description: 'Administrator with full access'
  });

  const editorRole = await Role.create({
    id: 'role_editor',
    name: 'editor',
    description: 'Editor with content management'
  });

  const viewerRole = await Role.create({
    id: 'role_viewer',
    name: 'viewer',
    description: 'Viewer with read-only access'
  });

  // Admin gets all permissions
  const allPermissions = await Permission.findAll();
  await adminRole.addPermissions(allPermissions);

  // Editor gets create, read, update
  const editPermissions = await Permission.findAll({
    where: { action: ['create', 'read', 'update'] }
  });
  await editorRole.addPermissions(editPermissions);

  // Viewer gets read only
  const viewPermissions = await Permission.findAll({
    where: { action: 'read' }
  });
  await viewerRole.addPermissions(viewPermissions);
}

async function seedPermissions() {
  const permissions = [
    // User management
    { name: 'users.read', resource: 'users', action: 'read' },
    { name: 'users.create', resource: 'users', action: 'create' },
    { name: 'users.update', resource: 'users', action: 'update' },
    { name: 'users.delete', resource: 'users', action: 'delete' },
    // Project management
    { name: 'projects.read', resource: 'projects', action: 'read' },
    { name: 'projects.create', resource: 'projects', action: 'create' },
    { name: 'projects.delete', resource: 'projects', action: 'delete' },
    // System
    { name: 'system.config', resource: 'system', action: 'config' }
  ];

  for (const perm of permissions) {
    await Permission.create({
      id: `perm_${perm.name}`,
      ...perm
    });
  }
}
```

## CASL-Based Authorization (Recommended)

```typescript
import { AbilityBuilder, createMongoAbility } from '@casl/ability';

// Define what subjects can be
type AppAbility = ReturnType<typeof createAppAbility>;

function createAppAbility(user: User) {
  const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

  if (user.roles.includes('admin')) {
    // Admin can do everything
    can('manage', 'all');
  } else if (user.roles.includes('editor')) {
    // Editor can create, read, update content
    can(['read', 'create', 'update'], 'Post');
    can(['read', 'update'], 'Post', { authorId: user.id });
    can('read', 'User');
    cannot('delete', 'Post');
  } else if (user.roles.includes('viewer')) {
    // Viewer can only read
    can('read', 'all');
  }

  // Team scoping: User can only manage their team
  if (user.teamId) {
    can(['read', 'update', 'delete'], 'Project', { teamId: user.teamId });
  }

  // Personal resource: Users can manage their own profile
  can(['read', 'update'], 'User', { id: user.id });

  return build();
}

// Middleware to enforce ability
const authorize = (action: string, subject: string) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    const ability = createAppAbility(req.user);

    if (!ability.can(action, subject)) {
      logAudit({
        userId: req.user.id,
        action: `${action}:${subject}`,
        result: 'denied',
        ip: req.ip
      });

      return res.status(403).json({ error: 'Access denied' });
    }

    // Store ability in request for route handler
    (req as any).ability = ability;
    next();
  };
};

// Usage
app.post('/api/posts', authorize('create', 'Post'), async (req, res) => {
  const post = await db.posts.create({
    ...req.body,
    authorId: req.user!.id
  });

  res.json(post);
});

app.delete('/api/posts/:id', authorize('delete', 'Post'), async (req, res) => {
  const post = await db.posts.findOne({ id: req.params.id });

  const ability = (req as any).ability;
  if (!ability.can('delete', 'Post', post)) {
    return res.status(403).json({ error: 'Cannot delete this post' });
  }

  await db.posts.delete({ id: req.params.id });
  res.json({ success: true });
});
```

## Audit Logging

```typescript
interface AuditLog {
  id: string;
  userId: string;
  action: string;
  resource: string;
  result: 'allowed' | 'denied';
  details?: Record<string, any>;
  timestamp: Date;
  ipAddress: string;
  userAgent: string;
}

class AuditService {
  async log(
    userId: string,
    action: string,
    resource: string,
    result: 'allowed' | 'denied',
    req: Request,
    details?: Record<string, any>
  ) {
    await db.auditLogs.create({
      id: generateId(),
      userId,
      action,
      resource,
      result,
      details,
      timestamp: new Date(),
      ipAddress: req.ip,
      userAgent: req.get('user-agent')
    });

    // Alert on denied access to sensitive resources
    if (result === 'denied' && this.isSensitive(resource)) {
      await this.alertSecurityTeam({
        userId,
        action,
        resource,
        ip: req.ip
      });
    }
  }

  private isSensitive(resource: string): boolean {
    const sensitiveResources = [
      'users.delete',
      'system.config',
      'data.export',
      'payments'
    ];
    return sensitiveResources.some(sr => resource.startsWith(sr));
  }

  private async alertSecurityTeam(details: Record<string, any>) {
    await notify('security-team', {
      title: 'Suspicious access attempt',
      details
    });
  }

  // Query audit logs
  async getAuditTrail(
    userId: string,
    startDate: Date,
    endDate: Date
  ) {
    return db.auditLogs.findAll({
      where: {
        userId,
        timestamp: { $gte: startDate, $lte: endDate }
      },
      order: [['timestamp', 'DESC']]
    });
  }
}

// Usage in middleware
const auditLog = new AuditService();

const requirePermission = (requiredPermission: string) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    if (!req.user?.permissions.includes(requiredPermission)) {
      await auditLog.log(
        req.user?.id || 'unknown',
        requiredPermission,
        req.path,
        'denied',
        req
      );

      return res.status(403).json({ error: 'Access denied' });
    }

    await auditLog.log(
      req.user.id,
      requiredPermission,
      req.path,
      'allowed',
      req,
      { method: req.method }
    );

    next();
  };
};
```

## Role Assignment Endpoint

```typescript
app.post('/api/users/:userId/roles/:roleId',
  requirePermission('users.update'),
  async (req, res) => {
    const { userId, roleId } = req.params;

    // Check if user/role exist
    const user = await db.users.findOne({ id: userId });
    const role = await db.roles.findOne({ id: roleId });

    if (!user || !role) {
      return res.status(404).json({ error: 'User or role not found' });
    }

    // Assign role
    await db.userRoles.create({ userId, roleId });

    // Audit
    await auditLog.log(
      req.user!.id,
      'assign_role',
      `User:${userId}`,
      'allowed',
      req,
      { role: role.name }
    );

    res.json({ success: true });
  }
);
```
