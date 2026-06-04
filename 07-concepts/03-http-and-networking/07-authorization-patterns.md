# Authorization Patterns

## Overview

Authorization determines what an authenticated user is allowed to do. While authentication verifies identity ("who are you?"), authorization controls access ("what can you do?"). Modern web applications use various authorization patterns including Role-Based Access Control (RBAC), Attribute-Based Access Control (ABAC), permissions systems, and resource-based access control.

## Authorization Models Comparison

```
RBAC (Role-Based Access Control):
User ──> Role ──> Permissions
John ──> Admin ──> [create, read, update, delete]
Jane ──> Editor ──> [read, update]
Bob ──> Viewer ──> [read]

ABAC (Attribute-Based Access Control):
User Attributes + Resource Attributes + Environment ──> Decision
{
  user: { department: "HR", level: 5 },
  resource: { department: "HR", sensitivity: "high" },
  environment: { time: "9-5", location: "office" }
} ──> Allow/Deny

Resource-Based Access Control:
User ──> Resource Ownership ──> Permissions
John ──> owns Document #123 ──> [read, write, delete]
Jane ──> editor on Document #123 ──> [read, write]
Bob ──> viewer on Document #123 ──> [read]
```

## RBAC (Role-Based Access Control)

RBAC assigns permissions to roles, and users are assigned roles.

### RBAC Implementation (React)

```typescript
// types/rbac.types.ts
export enum Permission {
  CREATE_POST = 'post:create',
  READ_POST = 'post:read',
  UPDATE_POST = 'post:update',
  DELETE_POST = 'post:delete',
  
  CREATE_USER = 'user:create',
  READ_USER = 'user:read',
  UPDATE_USER = 'user:update',
  DELETE_USER = 'user:delete',
  
  MANAGE_ROLES = 'role:manage',
  VIEW_ANALYTICS = 'analytics:view'
}

export enum Role {
  ADMIN = 'admin',
  EDITOR = 'editor',
  AUTHOR = 'author',
  VIEWER = 'viewer'
}

export type RolePermissions = {
  [key in Role]: Permission[];
};

export const rolePermissions: RolePermissions = {
  [Role.ADMIN]: [
    Permission.CREATE_POST,
    Permission.READ_POST,
    Permission.UPDATE_POST,
    Permission.DELETE_POST,
    Permission.CREATE_USER,
    Permission.READ_USER,
    Permission.UPDATE_USER,
    Permission.DELETE_USER,
    Permission.MANAGE_ROLES,
    Permission.VIEW_ANALYTICS
  ],
  [Role.EDITOR]: [
    Permission.CREATE_POST,
    Permission.READ_POST,
    Permission.UPDATE_POST,
    Permission.DELETE_POST,
    Permission.READ_USER
  ],
  [Role.AUTHOR]: [
    Permission.CREATE_POST,
    Permission.READ_POST,
    Permission.UPDATE_POST,
    Permission.READ_USER
  ],
  [Role.VIEWER]: [
    Permission.READ_POST,
    Permission.READ_USER
  ]
};
```

```typescript
// services/authorization.service.ts
import { Role, Permission, rolePermissions } from '../types/rbac.types';

class AuthorizationService {
  private userRoles: Role[] = [];

  setUserRoles(roles: Role[]) {
    this.userRoles = roles;
  }

  getUserRoles(): Role[] {
    return this.userRoles;
  }

  hasRole(role: Role): boolean {
    return this.userRoles.includes(role);
  }

  hasAnyRole(roles: Role[]): boolean {
    return roles.some(role => this.hasRole(role));
  }

  hasAllRoles(roles: Role[]): boolean {
    return roles.every(role => this.hasRole(role));
  }

  hasPermission(permission: Permission): boolean {
    return this.userRoles.some(role => 
      rolePermissions[role]?.includes(permission)
    );
  }

  hasAnyPermission(permissions: Permission[]): boolean {
    return permissions.some(permission => this.hasPermission(permission));
  }

  hasAllPermissions(permissions: Permission[]): boolean {
    return permissions.every(permission => this.hasPermission(permission));
  }

  getUserPermissions(): Permission[] {
    const permissions = new Set<Permission>();
    
    this.userRoles.forEach(role => {
      rolePermissions[role]?.forEach(permission => {
        permissions.add(permission);
      });
    });

    return Array.from(permissions);
  }

  can(action: string, resource: string): boolean {
    const permission = `${resource}:${action}` as Permission;
    return this.hasPermission(permission);
  }
}

export const authorizationService = new AuthorizationService();
```

```typescript
// hooks/useAuthorization.ts
import { useAuth } from './useAuth';
import { authorizationService } from '../services/authorization.service';
import { Role, Permission } from '../types/rbac.types';
import { useEffect } from 'react';

export const useAuthorization = () => {
  const { user } = useAuth();

  useEffect(() => {
    if (user?.roles) {
      authorizationService.setUserRoles(user.roles);
    }
  }, [user]);

  const hasRole = (role: Role): boolean => {
    return authorizationService.hasRole(role);
  };

  const hasAnyRole = (roles: Role[]): boolean => {
    return authorizationService.hasAnyRole(roles);
  };

  const hasPermission = (permission: Permission): boolean => {
    return authorizationService.hasPermission(permission);
  };

  const hasAnyPermission = (permissions: Permission[]): boolean => {
    return authorizationService.hasAnyPermission(permissions);
  };

  const can = (action: string, resource: string): boolean => {
    return authorizationService.can(action, resource);
  };

  return {
    hasRole,
    hasAnyRole,
    hasPermission,
    hasAnyPermission,
    can
  };
};
```

```typescript
// components/ProtectedRoute.tsx
import React from 'react';
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';
import { useAuthorization } from '../hooks/useAuthorization';
import { Role, Permission } from '../types/rbac.types';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requireAuth?: boolean;
  requiredRoles?: Role[];
  requiredPermissions?: Permission[];
  fallback?: React.ReactNode;
}

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({
  children,
  requireAuth = true,
  requiredRoles,
  requiredPermissions,
  fallback = <Navigate to="/unauthorized" />
}) => {
  const { isAuthenticated } = useAuth();
  const { hasAnyRole, hasAllPermissions } = useAuthorization();

  if (requireAuth && !isAuthenticated) {
    return <Navigate to="/login" />;
  }

  if (requiredRoles && !hasAnyRole(requiredRoles)) {
    return <>{fallback}</>;
  }

  if (requiredPermissions && !hasAllPermissions(requiredPermissions)) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
};
```

```typescript
// components/Can.tsx
import React from 'react';
import { useAuthorization } from '../hooks/useAuthorization';
import { Permission } from '../types/rbac.types';

interface CanProps {
  perform: Permission | Permission[];
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export const Can: React.FC<CanProps> = ({ perform, children, fallback = null }) => {
  const { hasPermission, hasAllPermissions } = useAuthorization();

  const isAuthorized = Array.isArray(perform)
    ? hasAllPermissions(perform)
    : hasPermission(perform);

  return isAuthorized ? <>{children}</> : <>{fallback}</>;
};

// Usage:
// <Can perform={Permission.CREATE_POST}>
//   <button>Create Post</button>
// </Can>
```

```typescript
// components/PostActions.tsx
import React from 'react';
import { Can } from './Can';
import { Permission } from '../types/rbac.types';

interface PostActionsProps {
  postId: string;
}

const PostActions: React.FC<PostActionsProps> = ({ postId }) => {
  return (
    <div className="post-actions">
      <Can perform={Permission.UPDATE_POST}>
        <button>Edit</button>
      </Can>
      
      <Can perform={Permission.DELETE_POST}>
        <button>Delete</button>
      </Can>
      
      <Can perform={[Permission.UPDATE_POST, Permission.VIEW_ANALYTICS]}>
        <button>View Stats</button>
      </Can>
    </div>
  );
};

export default PostActions;
```

### RBAC (Angular)

```typescript
// services/authorization.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

export enum Role {
  ADMIN = 'admin',
  EDITOR = 'editor',
  VIEWER = 'viewer'
}

export enum Permission {
  CREATE_POST = 'post:create',
  READ_POST = 'post:read',
  UPDATE_POST = 'post:update',
  DELETE_POST = 'post:delete'
}

@Injectable({
  providedIn: 'root'
})
export class AuthorizationService {
  private rolesSubject = new BehaviorSubject<Role[]>([]);
  public roles$ = this.rolesSubject.asObservable();

  private rolePermissions: Map<Role, Permission[]> = new Map([
    [Role.ADMIN, [
      Permission.CREATE_POST,
      Permission.READ_POST,
      Permission.UPDATE_POST,
      Permission.DELETE_POST
    ]],
    [Role.EDITOR, [
      Permission.CREATE_POST,
      Permission.READ_POST,
      Permission.UPDATE_POST
    ]],
    [Role.VIEWER, [Permission.READ_POST]]
  ]);

  setUserRoles(roles: Role[]) {
    this.rolesSubject.next(roles);
  }

  hasRole(role: Role): boolean {
    return this.rolesSubject.value.includes(role);
  }

  hasAnyRole(roles: Role[]): boolean {
    return roles.some(role => this.hasRole(role));
  }

  hasPermission(permission: Permission): boolean {
    return this.rolesSubject.value.some(role =>
      this.rolePermissions.get(role)?.includes(permission)
    );
  }

  getUserPermissions(): Permission[] {
    const permissions = new Set<Permission>();
    this.rolesSubject.value.forEach(role => {
      this.rolePermissions.get(role)?.forEach(p => permissions.add(p));
    });
    return Array.from(permissions);
  }
}
```

```typescript
// directives/has-permission.directive.ts
import { Directive, Input, TemplateRef, ViewContainerRef, OnInit } from '@angular/core';
import { AuthorizationService, Permission } from '../services/authorization.service';

@Directive({
  selector: '[appHasPermission]'
})
export class HasPermissionDirective implements OnInit {
  @Input() appHasPermission: Permission | Permission[];

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private authService: AuthorizationService
  ) {}

  ngOnInit() {
    this.updateView();
    
    this.authService.roles$.subscribe(() => {
      this.updateView();
    });
  }

  private updateView() {
    const permissions = Array.isArray(this.appHasPermission)
      ? this.appHasPermission
      : [this.appHasPermission];

    const hasPermission = permissions.every(p => this.authService.hasPermission(p));

    if (hasPermission) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else {
      this.viewContainer.clear();
    }
  }
}

// Usage:
// <button *appHasPermission="Permission.CREATE_POST">Create</button>
```

```typescript
// guards/permission.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, Router, ActivatedRouteSnapshot } from '@angular/router';
import { AuthorizationService, Permission } from '../services/authorization.service';

@Injectable({
  providedIn: 'root'
})
export class PermissionGuard implements CanActivate {
  constructor(
    private authService: AuthorizationService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredPermissions = route.data['permissions'] as Permission[];
    
    if (!requiredPermissions || requiredPermissions.length === 0) {
      return true;
    }

    const hasPermission = requiredPermissions.every(p =>
      this.authService.hasPermission(p)
    );

    if (!hasPermission) {
      this.router.navigate(['/unauthorized']);
      return false;
    }

    return true;
  }
}

// Usage in routing:
// {
//   path: 'posts/create',
//   component: CreatePostComponent,
//   canActivate: [PermissionGuard],
//   data: { permissions: [Permission.CREATE_POST] }
// }
```

## ABAC (Attribute-Based Access Control)

ABAC makes authorization decisions based on attributes of the user, resource, and environment.

```typescript
// types/abac.types.ts
export interface UserAttributes {
  id: string;
  department: string;
  level: number;
  location: string;
  clearance: string;
}

export interface ResourceAttributes {
  id: string;
  type: string;
  owner: string;
  department: string;
  sensitivity: 'public' | 'internal' | 'confidential' | 'secret';
  tags: string[];
}

export interface EnvironmentAttributes {
  time: Date;
  location: string;
  ipAddress: string;
  deviceType: string;
}

export interface AccessContext {
  user: UserAttributes;
  resource: ResourceAttributes;
  environment: EnvironmentAttributes;
  action: string;
}

export interface Policy {
  id: string;
  name: string;
  description: string;
  evaluate: (context: AccessContext) => boolean;
}
```

```typescript
// services/abac.service.ts
import { AccessContext, Policy, UserAttributes, ResourceAttributes } from '../types/abac.types';

class ABACService {
  private policies: Policy[] = [];

  registerPolicy(policy: Policy) {
    this.policies.push(policy);
  }

  evaluateAccess(context: AccessContext): boolean {
    // All policies must pass (AND logic)
    return this.policies.every(policy => policy.evaluate(context));
  }

  evaluateAnyPolicy(context: AccessContext): boolean {
    // At least one policy must pass (OR logic)
    return this.policies.some(policy => policy.evaluate(context));
  }

  can(
    action: string,
    user: UserAttributes,
    resource: ResourceAttributes,
    environment: any = {}
  ): boolean {
    const context: AccessContext = {
      user,
      resource,
      environment: {
        time: new Date(),
        location: environment.location || 'unknown',
        ipAddress: environment.ipAddress || 'unknown',
        deviceType: environment.deviceType || 'unknown'
      },
      action
    };

    return this.evaluateAccess(context);
  }
}

export const abacService = new ABACService();

// Define policies
abacService.registerPolicy({
  id: 'same-department-access',
  name: 'Same Department Access',
  description: 'Users can only access resources in their department',
  evaluate: (context) => {
    return context.user.department === context.resource.department;
  }
});

abacService.registerPolicy({
  id: 'owner-full-access',
  name: 'Owner Full Access',
  description: 'Resource owners have full access',
  evaluate: (context) => {
    return context.user.id === context.resource.owner;
  }
});

abacService.registerPolicy({
  id: 'sensitivity-clearance',
  name: 'Sensitivity Clearance',
  description: 'User clearance must match resource sensitivity',
  evaluate: (context) => {
    const clearanceLevels = {
      'public': 0,
      'internal': 1,
      'confidential': 2,
      'secret': 3
    };

    const userLevel = clearanceLevels[context.user.clearance] || 0;
    const resourceLevel = clearanceLevels[context.resource.sensitivity] || 0;

    return userLevel >= resourceLevel;
  }
});

abacService.registerPolicy({
  id: 'business-hours-only',
  name: 'Business Hours Only',
  description: 'Access only during business hours',
  evaluate: (context) => {
    const hour = context.environment.time.getHours();
    return hour >= 9 && hour <= 17;
  }
});

abacService.registerPolicy({
  id: 'high-level-confidential-access',
  name: 'High Level Confidential Access',
  description: 'Level 5+ users can access confidential resources',
  evaluate: (context) => {
    if (context.resource.sensitivity === 'confidential') {
      return context.user.level >= 5;
    }
    return true;
  }
});
```

```typescript
// components/ABACProtectedComponent.tsx
import React from 'react';
import { abacService } from '../services/abac.service';
import { useAuth } from '../hooks/useAuth';

interface ABACProtectedProps {
  action: string;
  resource: any;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

const ABACProtected: React.FC<ABACProtectedProps> = ({
  action,
  resource,
  children,
  fallback = null
}) => {
  const { user } = useAuth();

  if (!user) {
    return <>{fallback}</>;
  }

  const userAttributes = {
    id: user.id,
    department: user.department,
    level: user.level,
    location: user.location,
    clearance: user.clearance
  };

  const canAccess = abacService.can(action, userAttributes, resource);

  return canAccess ? <>{children}</> : <>{fallback}</>;
};

export default ABACProtected;
```

## Resource-Based Access Control

```typescript
// types/resource-access.types.ts
export enum ResourcePermission {
  OWNER = 'owner',
  EDITOR = 'editor',
  VIEWER = 'viewer'
}

export interface ResourceAccess {
  resourceId: string;
  resourceType: string;
  userId: string;
  permission: ResourcePermission;
  grantedAt: Date;
  grantedBy: string;
}

export interface Resource {
  id: string;
  type: string;
  ownerId: string;
  access: ResourceAccess[];
}
```

```typescript
// services/resource-authorization.service.ts
import { Resource, ResourcePermission, ResourceAccess } from '../types/resource-access.types';

class ResourceAuthorizationService {
  private resources = new Map<string, Resource>();

  registerResource(resource: Resource) {
    this.resources.set(resource.id, resource);
  }

  grantAccess(
    resourceId: string,
    userId: string,
    permission: ResourcePermission,
    grantedBy: string
  ) {
    const resource = this.resources.get(resourceId);
    if (!resource) {
      throw new Error('Resource not found');
    }

    const access: ResourceAccess = {
      resourceId,
      resourceType: resource.type,
      userId,
      permission,
      grantedAt: new Date(),
      grantedBy
    };

    resource.access.push(access);
  }

  revokeAccess(resourceId: string, userId: string) {
    const resource = this.resources.get(resourceId);
    if (!resource) return;

    resource.access = resource.access.filter(a => a.userId !== userId);
  }

  canAccess(
    resourceId: string,
    userId: string,
    requiredPermission?: ResourcePermission
  ): boolean {
    const resource = this.resources.get(resourceId);
    if (!resource) return false;

    // Owner has all permissions
    if (resource.ownerId === userId) {
      return true;
    }

    const userAccess = resource.access.find(a => a.userId === userId);
    if (!userAccess) return false;

    if (!requiredPermission) {
      return true; // Any access is sufficient
    }

    // Permission hierarchy: owner > editor > viewer
    const permissionLevels = {
      [ResourcePermission.OWNER]: 3,
      [ResourcePermission.EDITOR]: 2,
      [ResourcePermission.VIEWER]: 1
    };

    return permissionLevels[userAccess.permission] >= permissionLevels[requiredPermission];
  }

  getUserPermission(resourceId: string, userId: string): ResourcePermission | null {
    const resource = this.resources.get(resourceId);
    if (!resource) return null;

    if (resource.ownerId === userId) {
      return ResourcePermission.OWNER;
    }

    const userAccess = resource.access.find(a => a.userId === userId);
    return userAccess?.permission || null;
  }

  getResourceCollaborators(resourceId: string): ResourceAccess[] {
    const resource = this.resources.get(resourceId);
    return resource?.access || [];
  }
}

export const resourceAuthService = new ResourceAuthorizationService();
```

```typescript
// components/ShareResourceModal.tsx
import React, { useState } from 'react';
import { resourceAuthService } from '../services/resource-authorization.service';
import { ResourcePermission } from '../types/resource-access.types';
import { useAuth } from '../hooks/useAuth';

interface ShareResourceModalProps {
  resourceId: string;
  onClose: () => void;
}

const ShareResourceModal: React.FC<ShareResourceModalProps> = ({
  resourceId,
  onClose
}) => {
  const { user } = useAuth();
  const [email, setEmail] = useState('');
  const [permission, setPermission] = useState<ResourcePermission>(
    ResourcePermission.VIEWER
  );

  const collaborators = resourceAuthService.getResourceCollaborators(resourceId);

  const handleShare = async () => {
    // Lookup user by email (API call)
    const targetUser = await fetchUserByEmail(email);
    
    if (targetUser && user) {
      resourceAuthService.grantAccess(
        resourceId,
        targetUser.id,
        permission,
        user.id
      );
      setEmail('');
    }
  };

  const handleRevoke = (userId: string) => {
    resourceAuthService.revokeAccess(resourceId, userId);
  };

  return (
    <div className="modal">
      <h2>Share Resource</h2>
      
      <div className="share-form">
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          placeholder="Enter email"
        />
        
        <select
          value={permission}
          onChange={(e) => setPermission(e.target.value as ResourcePermission)}
        >
          <option value={ResourcePermission.VIEWER}>Viewer</option>
          <option value={ResourcePermission.EDITOR}>Editor</option>
          <option value={ResourcePermission.OWNER}>Owner</option>
        </select>
        
        <button onClick={handleShare}>Share</button>
      </div>

      <div className="collaborators-list">
        <h3>Current Collaborators</h3>
        {collaborators.map((collab) => (
          <div key={collab.userId} className="collaborator">
            <span>{collab.userId}</span>
            <span>{collab.permission}</span>
            <button onClick={() => handleRevoke(collab.userId)}>
              Remove
            </button>
          </div>
        ))}
      </div>

      <button onClick={onClose}>Close</button>
    </div>
  );
};

export default ShareResourceModal;
```

## Combining Authorization Patterns

```typescript
// services/unified-authorization.service.ts
import { authorizationService } from './authorization.service';
import { abacService } from './abac.service';
import { resourceAuthService } from './resource-authorization.service';
import { Permission } from '../types/rbac.types';
import { ResourcePermission } from '../types/resource-access.types';

class UnifiedAuthorizationService {
  canPerformAction(
    action: string,
    resourceId?: string,
    resourceType?: string
  ): boolean {
    // 1. Check RBAC permissions
    const permission = `${resourceType}:${action}` as Permission;
    if (!authorizationService.hasPermission(permission)) {
      return false;
    }

    // 2. Check resource-specific access if resourceId provided
    if (resourceId) {
      const requiredResourcePermission = this.mapActionToResourcePermission(action);
      if (!resourceAuthService.canAccess(resourceId, getCurrentUserId(), requiredResourcePermission)) {
        return false;
      }
    }

    // 3. Check ABAC policies
    // This would require building the full context
    // const canAccess = abacService.can(action, userAttrs, resourceAttrs);
    // if (!canAccess) return false;

    return true;
  }

  private mapActionToResourcePermission(action: string): ResourcePermission {
    const mapping: Record<string, ResourcePermission> = {
      'read': ResourcePermission.VIEWER,
      'update': ResourcePermission.EDITOR,
      'delete': ResourcePermission.OWNER
    };

    return mapping[action] || ResourcePermission.VIEWER;
  }
}

function getCurrentUserId(): string {
  // Get from auth context
  return 'current-user-id';
}

export const unifiedAuthService = new UnifiedAuthorizationService();
```

## Common Mistakes

### Mistake 1: Client-Side Only Authorization

```typescript
// BAD: Relying only on client-side checks
const canDelete = hasPermission(Permission.DELETE_POST);
if (canDelete) {
  await deletePost(postId); // Server doesn't verify!
}

// GOOD: Always verify on server
const canDelete = hasPermission(Permission.DELETE_POST);
if (canDelete) {
  try {
    await deletePost(postId); // Server MUST verify permissions
  } catch (error) {
    // Handle unauthorized error
  }
}
```

### Mistake 2: Hardcoding Roles in Components

```typescript
// BAD: Hardcoded role checks
if (user.role === 'admin') {
  return <AdminPanel />;
}

// GOOD: Use permission-based checks
if (hasPermission(Permission.MANAGE_ROLES)) {
  return <AdminPanel />;
}
```

### Mistake 3: Not Implementing Hierarchical Roles

```typescript
// BAD: Flat role structure
hasRole('admin') // Doesn't inherit editor permissions

// GOOD: Hierarchical permissions
// Admin inherits all editor permissions
rolePermissions[Role.ADMIN] = [
  ...rolePermissions[Role.EDITOR],
  ...additionalAdminPermissions
];
```

## Best Practices

1. Always verify authorization on the server side
2. Use permission-based checks, not role-based checks in UI
3. Implement principle of least privilege
4. Cache authorization decisions appropriately
5. Log all authorization failures for security auditing
6. Use hierarchical role structures
7. Separate authentication from authorization
8. Implement fine-grained resource-level permissions
9. Use ABAC for complex authorization scenarios
10. Regularly audit and review permissions

## Key Takeaways

1. Authorization determines what authenticated users can do
2. RBAC assigns permissions to roles, users get roles
3. ABAC makes decisions based on attributes and policies
4. Resource-based access controls per-resource permissions
5. Always verify authorization on server side
6. Use permission-based checks in UI, not role checks
7. Implement principle of least privilege
8. Combine multiple authorization patterns as needed

## Interview Questions

1. What is the difference between authentication and authorization?
2. Explain RBAC vs ABAC
3. How do you implement hierarchical roles?
4. What is the principle of least privilege?
5. How do you handle resource-level permissions?
6. What are the security risks of client-side only authorization?
7. How do you design a flexible permission system?
8. What is attribute-based access control?
9. How do you implement row-level security?
10. What are best practices for authorization in microservices?

## Resources

- NIST RBAC Standard
- XACML (eXtensible Access Control Markup Language)
- AWS IAM Policy Documentation
- OWASP Authorization Cheat Sheet
- Casbin (Authorization Library)
- OAuth 2.0 Scopes and Claims
