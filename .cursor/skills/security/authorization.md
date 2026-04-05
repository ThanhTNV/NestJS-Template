# Authorization Patterns

This document covers role-based and resource-based access control implementations.

## Role-Based Access Control (RBAC)

### Role Enum

```typescript
// src/auth/enums/role.enum.ts
export enum UserRole {
  ADMIN = 'admin',
  MODERATOR = 'moderator',
  USER = 'user',
  GUEST = 'guest',
}

// Permissions by role
export const rolePermissions = {
  [UserRole.ADMIN]: ['*'],          // All permissions
  [UserRole.MODERATOR]: ['read', 'update', 'delete_comments'],
  [UserRole.USER]: ['read', 'create', 'update_own'],
  [UserRole.GUEST]: ['read'],
};
```

### Role Hierarchy

```typescript
// src/auth/services/role.service.ts
@Injectable()
export class RoleService {
  private readonly hierarchy = {
    [UserRole.ADMIN]: [UserRole.ADMIN, UserRole.MODERATOR, UserRole.USER],
    [UserRole.MODERATOR]: [UserRole.MODERATOR, UserRole.USER],
    [UserRole.USER]: [UserRole.USER],
    [UserRole.GUEST]: [UserRole.GUEST],
  };

  canAccessAs(userRole: UserRole, requiredRole: UserRole): boolean {
    return this.hierarchy[userRole]?.includes(requiredRole) || false;
  }

  hasPermission(userRole: UserRole, permission: string): boolean {
    const permissions = rolePermissions[userRole] || [];
    return permissions.includes('*') || permissions.includes(permission);
  }
}
```

### Roles Guard

```typescript
// src/auth/guards/roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private roleService: RoleService,
  ) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<UserRole[]>(
      'roles',
      context.getHandler(),
    );

    if (!requiredRoles) return true; // No role required

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      throw new UnauthorizedException('User not authenticated');
    }

    const hasRole = requiredRoles.some(role =>
      this.roleService.canAccessAs(user.role, role)
    );

    if (!hasRole) {
      throw new ForbiddenException(
        `Required roles: ${requiredRoles.join(', ')}`
      );
    }

    return true;
  }
}
```

### Roles Decorator

```typescript
// src/auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { UserRole } from '../enums/role.enum';

export const Roles = (...roles: UserRole[]) => SetMetadata('roles', roles);
```

### Controller Usage

```typescript
@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  // Admin only
  @Get('users')
  @Roles(UserRole.ADMIN)
  getAllUsers() {
    return this.userService.findAll();
  }

  // Admin or moderator
  @Delete('users/:id')
  @Roles(UserRole.ADMIN, UserRole.MODERATOR)
  deleteUser(@Param('id') id: string) {
    return this.userService.delete(id);
  }

  // Any authenticated user
  @Get('profile')
  @UseGuards(JwtAuthGuard)
  getProfile(@Request() req) {
    return this.userService.findById(req.user.id);
  }
}
```

## Resource-Based Access Control (RBAC)

### Ownership Check

```typescript
// src/auth/guards/resource-owner.guard.ts
@Injectable()
export class ResourceOwnerGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private userService: UserService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      throw new UnauthorizedException();
    }

    // Get resource ID from route params
    const resourceId = request.params.id;

    // Check if user is admin (admins can access anything)
    if (user.roles.includes('admin')) {
      return true;
    }

    // Check ownership (user can only access their own resources)
    const resource = await this.userService.findById(resourceId);

    if (resource.userId !== user.id) {
      throw new ForbiddenException('Cannot access other users resources');
    }

    return true;
  }
}
```

### Usage

```typescript
@Controller('posts')
export class PostsController {
  @Get(':id')
  @UseGuards(JwtAuthGuard, ResourceOwnerGuard)
  getPost(@Param('id') id: string, @Request() req) {
    return this.postService.findById(id);
  }

  @Put(':id')
  @UseGuards(JwtAuthGuard, ResourceOwnerGuard)
  updatePost(@Param('id') id: string, @Body() dto: UpdatePostDto, @Request() req) {
    // Only owner or admin can update
    return this.postService.update(id, dto);
  }

  @Delete(':id')
  @UseGuards(JwtAuthGuard, ResourceOwnerGuard)
  deletePost(@Param('id') id: string, @Request() req) {
    // Only owner or admin can delete
    return this.postService.delete(id);
  }
}
```

## Attribute-Based Access Control (ABAC)

### Policy Service

```typescript
// src/auth/services/policy.service.ts
interface Policy {
  name: string;
  evaluate: (subject: any, resource: any, action: string) => boolean;
}

@Injectable()
export class PolicyService {
  private policies: Map<string, Policy> = new Map();

  constructor() {
    this.registerDefaultPolicies();
  }

  private registerDefaultPolicies(): void {
    // Admin can do anything
    this.policies.set('admin_policy', {
      name: 'admin_policy',
      evaluate: (subject, _, __) => subject.roles.includes('admin'),
    });

    // User can only access own resources
    this.policies.set('own_resource_policy', {
      name: 'own_resource_policy',
      evaluate: (subject, resource, _) => subject.id === resource.userId,
    });

    // Moderator can access public resources
    this.policies.set('public_resource_policy', {
      name: 'public_resource_policy',
      evaluate: (subject, resource, _) =>
        subject.roles.includes('moderator') && resource.isPublic,
    });

    // Time-based access
    this.policies.set('business_hours_policy', {
      name: 'business_hours_policy',
      evaluate: (_, __, action) => {
        if (action !== 'delete') return true;
        const hour = new Date().getHours();
        return hour >= 9 && hour < 17; // 9am-5pm only
      },
    });
  }

  evaluate(subject: any, resource: any, action: string, policyNames: string[]): boolean {
    return policyNames.some(name => {
      const policy = this.policies.get(name);
      return policy?.evaluate(subject, resource, action);
    });
  }
}
```

### Policy Guard

```typescript
// src/auth/guards/policy.guard.ts
@Injectable()
export class PolicyGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private policyService: PolicyService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPolicies = this.reflector.get<string[]>(
      'policies',
      context.getHandler(),
    );

    if (!requiredPolicies) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const resource = request.resource; // Set by middleware

    const allowed = this.policyService.evaluate(
      user,
      resource,
      request.method,
      requiredPolicies
    );

    if (!allowed) {
      throw new ForbiddenException('Access denied by policy');
    }

    return true;
  }
}
```

### Usage

```typescript
@Require Policy Decorator

import { SetMetadata } from '@nestjs/common';

export const RequirePolicy = (...policies: string[]) =>
  SetMetadata('policies', policies);

@Controller('posts')
export class PostsController {
  @Delete(':id')
  @UseGuards(JwtAuthGuard, PolicyGuard)
  @RequirePolicy('admin_policy', 'own_resource_policy', 'business_hours_policy')
  deletePost(@Param('id') id: string) {
    return this.postService.delete(id);
  }
}
```

## Scope-Based Access

### Scopes

```typescript
// src/auth/enums/scope.enum.ts
export enum Scope {
  READ = 'read',
  WRITE = 'write',
  DELETE = 'delete',
  ADMIN = 'admin',
}

export const scopeHierarchy = {
  [Scope.ADMIN]: [Scope.ADMIN, Scope.DELETE, Scope.WRITE, Scope.READ],
  [Scope.DELETE]: [Scope.DELETE, Scope.WRITE, Scope.READ],
  [Scope.WRITE]: [Scope.WRITE, Scope.READ],
  [Scope.READ]: [Scope.READ],
};
```

### Scope Guard

```typescript
// src/auth/guards/scope.guard.ts
@Injectable()
export class ScopeGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredScope = this.reflector.get<Scope>(
      'requiredScope',
      context.getHandler(),
    );

    if (!requiredScope) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    const userScopes = user.scopes || [];
    const hasScope = userScopes.some(scope =>
      scopeHierarchy[scope]?.includes(requiredScope)
    );

    if (!hasScope) {
      throw new ForbiddenException(`Required scope: ${requiredScope}`);
    }

    return true;
  }
}
```

## Dynamic Permissions

### Permission Resolver

```typescript
// src/auth/services/permission-resolver.ts
@Injectable()
export class PermissionResolver {
  constructor(
    private userService: UserService,
    private roleService: RoleService,
  ) {}

  async resolvePermissions(userId: string): Promise<Set<string>> {
    const user = await this.userService.findById(userId);
    const permissions = new Set<string>();

    // Add role-based permissions
    const rolePerms = rolePermissions[user.role] || [];
    rolePerms.forEach(p => permissions.add(p));

    // Add user-specific permissions
    user.grantedPermissions?.forEach(p => permissions.add(p));

    // Add team-based permissions (if user is in a team)
    if (user.teamId) {
      const teamPerms = await this.getTeamPermissions(user.teamId);
      teamPerms.forEach(p => permissions.add(p));
    }

    return permissions;
  }

  private async getTeamPermissions(teamId: string): Promise<string[]> {
    // Implement based on your team model
    return [];
  }

  canAccess(permissions: Set<string>, requiredPermission: string): boolean {
    return permissions.has('*') || permissions.has(requiredPermission);
  }
}
```

## Access Control List (ACL)

### ACL Model

```typescript
// src/auth/entities/acl-entry.entity.ts
@Entity('acl_entries')
export class AclEntry {
  @PrimaryGeneratedColumn()
  id: number;

  @Column() // resource_type:resource_id:owner_id
  resource: string;

  @Column()
  principal: string; // user_id or role

  @Column()
  permission: string; // read, write, delete

  @CreateDateColumn()
  createdAt: Date;
}
```

### ACL Service

```typescript
// src/auth/services/acl.service.ts
@Injectable()
export class AclService {
  constructor(private aclRepository: Repository<AclEntry>) {}

  async grant(
    resource: string,
    principal: string,
    permission: string,
  ): Promise<void> {
    await this.aclRepository.save({
      resource,
      principal,
      permission,
    });
  }

  async revoke(
    resource: string,
    principal: string,
    permission: string,
  ): Promise<void> {
    await this.aclRepository.delete({
      resource,
      principal,
      permission,
    });
  }

  async canAccess(
    resource: string,
    userId: string,
    permission: string,
  ): Promise<boolean> {
    const entry = await this.aclRepository.findOne({
      where: {
        resource,
        principal: userId,
        permission,
      },
    });

    return !!entry;
  }
}
```
