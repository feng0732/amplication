# Team 权限与工作区隔离机制代码解析

## 1. 核心数据模型关系

权限体系基于以下五个核心实体构建，它们之间形成严格的层级归属关系：

```
Workspace (工作区 - 最高隔离边界)
  ├── User (用户 - 属于某个 Workspace)
  │     └── isOwner: boolean (工作区所有者标志)
  ├── Team (团队 - 属于某个 Workspace)
  │     ├── members: User[] (团队成员)
  │     └── roles: Role[] (团队关联的角色)
  ├── Role (角色 - 属于某个 Workspace)
  │     └── permissions: string[] (权限字符串数组)
  └── Project (项目 - 属于某个 Workspace)
        └── Resource (资源 - 属于某个 Project)
              └── TeamAssignment[] (团队对资源的分配 + 附加角色)
                    ├── teamId
                    ├── resourceId
                    └── roles: Role[]
```

### 关键模型定义位置

- [Workspace.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/Workspace.ts) - 工作区模型
- [User.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/User.ts) - 用户模型（含 `isOwner` 标志）
- [Team.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/Team.ts) - 团队模型
- [Role.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/Role.ts) - 角色模型（含 `permissions` 字符串数组）
- [Project.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/Project.ts) - 项目模型
- [TeamAssignment.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/TeamAssignment.ts) - 团队-资源分配关联模型

---

## 2. 工作区初始化：默认团队与角色

创建新 Workspace 时，系统会自动创建三个默认团队及对应角色。

代码位置：[workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L166-L200) 中的 `createDefaultTeams` 方法。

默认配置见 [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/workspace/constants.ts#L14-L71)：

| 团队名称 | 角色 Key | 权限 | 说明 |
|---------|---------|------|-----|
| Admins | `ADMINS` | `["*"]` | 工作区管理员，拥有全部权限 |
| Platform Engineers | `PLATFORM_ENGINEERS` | `project.create`, `privatePlugin.*`, `resource.createTemplate` 等 | 平台工程师，可创建和管理插件与模板 |
| Developers | `DEVELOPER` | `project.create`, `resource.*.edit`, `resource.delete`, `resource.create*` 等 | 开发者，可创建和构建资源与服务 |

Workspace 创建者会被自动加入 Admins 团队，同时 User.isOwner 设为 true。

---

## 3. 用户权限加载流程

用户登录或获取认证信息时，系统会计算其拥有的全部权限。

代码位置：[user.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/user/user.service.ts#L74-L105) 中的 `getUserPermissions` 方法。

```
getUserPermissions(userId)
    │
    ├─► 若 user.isOwner === true  ──► 直接返回 ["*"]
    │
    └─► 查询数据库：找出用户所属 Team 关联的所有 Role
          (条件：Team.deletedAt=null, Role.deletedAt=null, Team 包含该成员)
                │
                └─► 汇总所有 Role.permissions，去重后返回
```

权限结果会被放入 JWT Token 中（见 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L517-L527) 的 `prepareToken`），避免每次请求都查库。

---

## 4. 请求授权：两阶段校验

所有 GraphQL 请求经过 [GqlAuthGuard](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) 守卫，通过 `@AuthorizeContext` 装饰器声明授权规则。

### 4.1 装饰器声明方式

在 Resolver 方法上使用 `@AuthorizeContext(参数类型, 参数路径, [所需权限])`，示例见 [team.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/team/team.resolver.ts#L68-L82)：

```typescript
@Mutation(() => Team)
@AuthorizeContext(AuthorizableOriginParameter.TeamId, "where.id", "team.delete")
async deleteTeam(@Args() args: FindOneArgs): Promise<Team | null> { ... }
```

- `AuthorizableOriginParameter.TeamId`：告诉系统参数是 Team ID
- `"where.id"`：从请求参数中取值的路径
- `"team.delete"`：需要的权限（可多个）

### 4.2 阶段一：工作区隔离验证（Workspace Boundary）

调用 [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L18-L47) 的 `validateAccess`，首先通过 [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts) 中的 `VALIDATION_FUNCTIONS` 校验目标对象是否属于当前用户的 workspace。

`AuthorizableOriginParameter` 枚举中定义的**每一种资源类型**都有对应的验证函数（见 [AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts)），核心逻辑都是：**通过资源 ID 向上追溯到 workspace，验证 workspace.id === user.workspace.id**。

示例验证逻辑（以 ProjectId 为例，[validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L78-L96)）：

```typescript
[AuthorizableOriginParameter.ProjectId]: async (prisma, originId, workspaceId) => {
  const matching = await prisma.project.count({
    where: {
      deletedAt: null,
      id: originId,
      workspace: { id: workspaceId },  // 关键：归属检查
    },
  });
  return {
    canAccessWorkspace: matching === 1,
    requestedProjectId: originId,
  };
}
```

若 `canAccessWorkspace === false`，直接拒绝访问 —— 这是**跨工作区隔离的第一道防线**。

### 4.3 阶段二：权限匹配验证

通过 [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L49-L87) 的 `validatePermissions` 方法：

```
validatePermissions(user, requiredPermissions, resourceId, projectId)
    │
    ├─► requiredPermissions 为空？ ──► 返回 true（只要在工作区内即可）
    │
    ├─► user.permissions 含 "*"？ ──► 返回 true（Owner 或 Admins）
    │
    ├─► Team 级权限：user.permissions 中是否已包含所需权限？
    │     (即用户通过所属 Team 直接关联的 Role 获得的权限)
    │     └─► 是 ──► 返回 true
    │
    └─► Resource/Project 级权限（TeamAssignment）：
          通过 getUserResourceOrProjectPermissions 查询
          TeamAssignment 中该用户所在 Team 对目标 Resource 或 Project
          被分配的额外 Role 权限
                │
                └─► 匹配到 ──► 返回 true
                └─► 未匹配 ──► 返回 false
```

### 4.4 资源级权限（TeamAssignment）

TeamAssignment 是实现**同一工作区内不同项目/资源精细化授权**的关键。一个 Team 虽然有全局 Role 权限，但也可以在特定 Resource 上被赋予额外 Role。

代码位置：[permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L106-L199) 中的 `getUserResourceOrProjectPermissions`。

查询逻辑：
1. 如果传入 projectId，找到该项目的 `ProjectConfiguration` 资源 ID
2. 如果传入 resourceId，找到其所属项目的 `ProjectConfiguration` 资源 ID + 资源自身 ID
3. 查询 `teamAssignment` 表中，用户所在 Team 对这些 resourceId 的分配，收集附加 Role 的 permissions

这意味着：**Project 级别的 TeamAssignment 会对项目内所有 Resource 生效**（通过 ProjectConfiguration 资源作为桥梁）。

---

## 5. 跨工作区隔离的多层保障

跨工作区隔离是多维度共同保障的，不是单一检查点：

### 5.1 数据模型层：每个实体都带 workspaceId

所有核心实体都直接或间接关联 workspaceId：

| 实体 | workspaceId 关联方式 |
|-----|---------------------|
| User | 直接字段 |
| Team | 直接字段 |
| Role | 直接字段 |
| Project | 直接字段 |
| Resource | 通过 Project.workspaceId |
| Invitation | 直接字段 |
| GitOrganization | 直接字段 |
| Blueprint | 直接字段 |
| CustomProperty | 直接字段 |

### 5.2 关联操作层：跨工作区关联被拒绝

在业务操作中，显式检查不允许跨工作区关联：

**添加成员到 Team 时**（[team.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/team/team.service.ts#L122-L182) 的 `addMembersToTeam`）：
```typescript
OR: [
  { deletedAt: { not: null } },
  { workspaceId: { not: team.workspaceId } },  // 拒绝不同 workspace 的用户
]
```

**添加角色到 Team 时**（[team.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/team/team.service.ts#L332-L361) 的 `validateRoles`）：
```typescript
OR: [
  { deletedAt: { not: null } },
  { workspaceId: { not: workspaceId } },  // 拒绝不同 workspace 的角色
]
```

**分配 Team 到资源时**（[team.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/team/team.service.ts#L363-L407) 的 `validateTeams`）：
```typescript
OR: [
  { deletedAt: { not: null } },
  { workspaceId: { not: workspaceId } },  // 拒绝不同 workspace 的团队
]
```

**删除 workspace 用户时**（[workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L565-L587) 的 `deleteUser`）：
```typescript
if (currentUser.workspace.id !== user.workspace.id) {
  throw new ConflictException("The requested user is not in the current user's workspace");
}
```

### 5.3 请求守卫层：VALIDATION_FUNCTIONS 全覆盖

如 4.2 节所述，`AuthorizableOriginParameter` 枚举中的 20+ 种资源类型全部对应了验证函数，确保访问任何对象前都追溯 workspace 归属。

### 5.4 数据查询层：WorkspaceId 注入

使用 `@InjectContextValue` 装饰器在查询前自动注入当前用户的 workspaceId 到查询条件中（见 [team.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/team/team.resolver.ts#L40-L47)），防止越权查询其他 workspace 的数据。

```typescript
@Query(() => [Team])
@InjectContextValue(InjectableOriginParameter.WorkspaceId, "where.workspace.id")
async teams(@Args() args: TeamFindManyArgs): Promise<Team[]> { ... }
```

此外，`ProjectService` 的 `commit` 和 `getPendingChanges` 等方法中，查询资源时也强制加入 workspace 用户归属检查（[project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/project/project.service.ts#L180-L201)）：

```typescript
project: {
  workspace: {
    users: {
      some: { id: user.id },  // 确保用户属于该 workspace
    },
  },
}
```

---

## 6. Account 与 User 的关系：多工作区切换

注意 [User.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/User.ts) 通过 `account` 关联 Account，一个 Account 可以在**多个 Workspace** 中各有一个 User 记录。

- Account 代表登录账号（邮箱、密码、GitHub 身份等）
- User 代表该账号在**某个特定 Workspace** 中的身份（含 isOwner、teams、workspace 等）

切换工作区的逻辑在 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L366-L388) 的 `setCurrentWorkspace`：通过 accountId + workspaceId 找到对应的 User，重新生成 JWT Token（包含新 workspace 的 permissions）。

---

## 7. 整体流程图

```
用户发起 GraphQL 请求
       │
       ▼
  GqlAuthGuard.canActivate()
       │
       ├─► JWT 认证 ──► 失败则拒绝
       │
       ▼
  读取 @AuthorizeContext 元数据
  (参数类型 / 参数路径 / 所需权限)
       │
       ▼
  PermissionsService.validateAccess()
       │
       ├─►【阶段一：工作区隔离】
       │     VALIDATION_FUNCTIONS[参数类型]
       │     验证目标对象.workspaceId === user.workspaceId
       │           │
       │           ├─► 不匹配 ──► 拒绝 ❌
       │           └─► 匹配 ──► 继续
       │
       ▼
  PermissionsService.validatePermissions()
       │
       ├─► 无权限要求？ ──► 允许 ✅
       ├─► user 有 "*"？ ──► 允许 ✅ (Owner/Admins)
       ├─► Team 级权限匹配？ ──► 允许 ✅
       └─► TeamAssignment 资源级权限匹配？
                  ├─► 是 ──► 允许 ✅
                  └─► 否 ──► 拒绝 ❌
```

---

## 8. 关键文件索引

| 功能 | 文件路径 |
|-----|---------|
| 核心模型 | [models/](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/models/) |
| 用户权限计算 | [user.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/user/user.service.ts) |
| 授权守卫 | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) |
| 权限服务 | [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts) |
| 资源归属验证函数 | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts) |
| 授权装饰器 | [authorizeContext.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/decorators/authorizeContext.decorator.ts) |
| 团队服务 | [team.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/team/team.service.ts) |
| 工作区服务 | [workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts) |
| 默认团队/角色配置 | [workspace/constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/workspace/constants.ts) |
| 可授权参数枚举 | [AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts) |
| 认证服务 | [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/core/auth/auth.service.ts) |
