# Team 权限与工作区隔离机制代码解析

> 仓库根目录：`41-amplication`
> 所有相对路径均相对于仓库根目录

---

## 1. 核心数据模型关系

权限体系基于以下核心实体构建，它们之间形成严格的层级归属关系：

```
Workspace (工作区 - 最高隔离边界)
  ├── User (用户 - 属于某个 Workspace)
  │     └── isOwner: boolean (工作区所有者标志)
  ├── Team (团队 - 属于某个 Workspace)
  │     ├── members: User[] (团队成员)
  │     └── roles: Role[] (团队关联的角色)
  ├── Role (角色 - 属于某个 Workspace)
  │     └── permissions: string[] (权限字符串数组)
  ├── GitOrganization (Git 组织 - 属于某个 Workspace)
  │     └── GitRepository[] (Git 仓库 - 通过 gitOrganizationId 间接关联 Workspace)
  └── Project (项目 - 属于某个 Workspace)
        └── Resource (资源 - 属于某个 Project)
              └── TeamAssignment[] (团队对资源的分配 + 附加角色)
                    ├── teamId
                    ├── resourceId
                    └── roles: Role[]
```

**注意 GitRepository 的特殊归属链**：GitRepository 模型本身**没有直接的 workspaceId 字段**，它只通过 `gitOrganizationId` 关联到 GitOrganization，再由 GitOrganization 关联 Workspace。这一点在后面的验证函数中有重要影响。

### 关键模型定义位置

| 模型 | 相对路径 |
|-----|---------|
| Workspace | `packages/amplication-server/src/models/Workspace.ts` |
| User | `packages/amplication-server/src/models/User.ts` |
| Team | `packages/amplication-server/src/models/Team.ts` |
| Role | `packages/amplication-server/src/models/Role.ts` |
| Project | `packages/amplication-server/src/models/Project.ts` |
| TeamAssignment | `packages/amplication-server/src/models/TeamAssignment.ts` |
| GitOrganization | `packages/amplication-server/src/models/GitOrganization.ts` |
| GitRepository | `packages/amplication-server/src/models/GitRepository.ts` |

---

## 2. 工作区初始化：默认团队与角色

创建新 Workspace 时，系统会自动创建三个默认团队及对应角色。

代码位置：`packages/amplication-server/src/core/workspace/workspace.service.ts` 中的 `createDefaultTeams` 方法（L166-L200）。

默认配置见 `packages/amplication-server/src/core/workspace/constants.ts`（L14-L71）：

| 团队名称 | 角色 Key | 权限 | 说明 |
|---------|---------|------|-----|
| Admins | `ADMINS` | `["*"]` | 工作区管理员，拥有全部权限 |
| Platform Engineers | `PLATFORM_ENGINEERS` | `project.create`, `privatePlugin.*`, `resource.createTemplate` 等 | 平台工程师，可创建和管理插件与模板 |
| Developers | `DEVELOPER` | `project.create`, `resource.*.edit`, `resource.delete`, `resource.create*` 等 | 开发者，可创建和构建资源与服务 |

Workspace 创建者会被自动加入 Admins 团队，同时 User.isOwner 设为 true。

---

## 3. 用户权限加载流程

用户登录或获取认证信息时，系统会计算其拥有的全部权限。

代码位置：`packages/amplication-server/src/core/user/user.service.ts` 中的 `getUserPermissions` 方法（L74-L105）。

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

权限结果会被放入 JWT Token 中（见 `packages/amplication-server/src/core/auth/auth.service.ts` 的 `prepareToken`，L517-L527），避免每次请求都查库。

---

## 4. 请求授权：两阶段校验

所有 GraphQL 请求经过 `packages/amplication-server/src/guards/gql-auth.guard.ts` 中的 `GqlAuthGuard` 守卫，通过 `@AuthorizeContext` 装饰器声明授权规则。

### 4.1 装饰器声明方式

在 Resolver 方法上使用 `@AuthorizeContext(参数类型, 参数路径, [所需权限])`，示例见 `packages/amplication-server/src/core/team/team.resolver.ts`（L68-L82）：

```typescript
@Mutation(() => Team)
@AuthorizeContext(AuthorizableOriginParameter.TeamId, "where.id", "team.delete")
async deleteTeam(@Args() args: FindOneArgs): Promise<Team | null> { ... }
```

- `AuthorizableOriginParameter.TeamId`：告诉系统参数是 Team ID
- `"where.id"`：从请求参数中取值的路径
- `"team.delete"`：需要的权限（可多个）

### 4.2 阶段一：工作区隔离验证（Workspace Boundary）

调用 `packages/amplication-server/src/core/permissions/permissions.service.ts` 的 `validateAccess` 方法（L18-L47），首先通过 `packages/amplication-server/src/core/permissions/validation-functions.ts` 中的 `VALIDATION_FUNCTIONS` 校验目标对象是否属于当前用户的 workspace。

`packages/amplication-server/src/enums/AuthorizableOriginParameter.ts` 中定义的**每一种资源类型**都有对应的验证函数。

#### 4.2.1 标准模式：追溯 workspaceId 进行比对

绝大多数验证函数遵循统一模式：**通过资源 ID 向上追溯到 workspace，验证 workspace.id === user.workspace.id**。

示例验证逻辑（以 ProjectId 为例，`validation-functions.ts` L78-L96）：

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

#### 4.2.2 重要例外：不按 workspace 追溯的验证函数

在 26 种 `AuthorizableOriginParameter` 中，有 **3 种** 不遵循"追溯 workspaceId"的标准模式：

| 参数类型 | 验证逻辑 | 相对路径（行号） | 说明 |
|---------|---------|----------------|-----|
| `None` | 直接返回 `{ canAccessWorkspace: true }` | `validation-functions.ts` L39-L41 | 空类型，用于不需要绑定具体资源的操作 |
| `GitRepositoryId` | 只检查 `id: originId` 是否存在，**完全不验证 workspace 归属** | `validation-functions.ts` L66-L77 | ⚠️ 安全例外：GitRepository 本身无 workspaceId 字段，但应通过 `gitOrganization → workspace` 间接验证，当前实现缺失 |
| `ApiTokenId` | 检查 `id: originId AND userId: user.id`，按**用户所有权**而非 workspace 归属验证 | `validation-functions.ts` L198-L211 | 基于用户所有权的安全模型：用户只能访问自己创建的 API Token，天然限制在同一 workspace（因为一个 User 只属于一个 Workspace） |

**GitRepositoryId 验证函数的实际代码**：

```typescript
// validation-functions.ts L66-L77
[AuthorizableOriginParameter.GitRepositoryId]: async (
  prisma: PrismaService,
  originId: string,
  workspaceId: string
) => {
  const matching = await prisma.gitRepository.count({
    where: {
      id: originId,
      // ⚠️ 注意：这里没有加 workspace 相关的过滤条件！
    },
  });
  return { canAccessWorkspace: matching === 1 };
};
```

对比 **GitOrganizationId** 验证函数（标准模式）：

```typescript
// validation-functions.ts L51-L65
[AuthorizableOriginParameter.GitOrganizationId]: async (
  prisma: PrismaService,
  originId: string,
  workspaceId: string
) => {
  const matching = await prisma.gitOrganization.count({
    where: {
      id: originId,
      workspace: { id: workspaceId },  // ✅ 有 workspace 归属检查
    },
  });
  return { canAccessWorkspace: matching === 1 };
};
```

GitRepository 的跨工作区访问实际上在 **Resolver 层通过权限要求** 间接保障：使用 `GitRepositoryId` 的操作（如 `deleteGitRepository`、`updateGitRepository`）都需要 `"git.repo.disconnect"` 或 `"git.repo.settings.edit"` 权限，这些权限本身是 workspace 级别的，用户在另一 workspace 中不具备。

### 4.3 阶段二：权限匹配验证

通过 `packages/amplication-server/src/core/permissions/permissions.service.ts` 的 `validatePermissions` 方法（L49-L87）：

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

代码位置：`packages/amplication-server/src/core/permissions/permissions.service.ts` 中的 `getUserResourceOrProjectPermissions` 方法（L106-L199）。

查询逻辑：
1. 如果传入 projectId，找到该项目的 `ProjectConfiguration` 资源 ID
2. 如果传入 resourceId，找到其所属项目的 `ProjectConfiguration` 资源 ID + 资源自身 ID
3. 查询 `teamAssignment` 表中，用户所在 Team 对这些 resourceId 的分配，收集附加 Role 的 permissions

这意味着：**Project 级别的 TeamAssignment 会对项目内所有 Resource 生效**（通过 ProjectConfiguration 资源作为桥梁）。

---

## 5. 跨工作区隔离的多层保障

跨工作区隔离是多维度共同保障的，不是单一检查点：

### 5.1 数据模型层：各实体 workspaceId 关联方式

| 实体 | workspaceId 关联方式 | 备注 |
|-----|---------------------|-----|
| User | 直接字段 | |
| Team | 直接字段 | |
| Role | 直接字段 | |
| Project | 直接字段 | |
| Resource | 通过 Project.workspaceId | 间接关联 |
| Invitation | 直接字段 | |
| GitOrganization | 直接字段 | |
| **GitRepository** | **无直接字段，通过 GitOrganization 间接关联** | ⚠️ 特殊：无 workspaceId，只有 gitOrganizationId |
| Blueprint | 直接字段 | |
| CustomProperty | 直接字段 | |
| ApiToken | 通过 User.workspaceId 间接关联 | 有 userId 字段，User 属于某一 Workspace |

### 5.2 关联操作层：跨工作区关联被拒绝

在业务操作中，显式检查不允许跨工作区关联：

**添加成员到 Team 时**（`packages/amplication-server/src/core/team/team.service.ts` 的 `addMembersToTeam`，L122-L182）：
```typescript
OR: [
  { deletedAt: { not: null } },
  { workspaceId: { not: team.workspaceId } },  // 拒绝不同 workspace 的用户
]
```

**添加角色到 Team 时**（`packages/amplication-server/src/core/team/team.service.ts` 的 `validateRoles`，L332-L361）：
```typescript
OR: [
  { deletedAt: { not: null } },
  { workspaceId: { not: workspaceId } },  // 拒绝不同 workspace 的角色
]
```

**分配 Team 到资源时**（`packages/amplication-server/src/core/team/team.service.ts` 的 `validateTeams`，L363-L407）：
```typescript
OR: [
  { deletedAt: { not: null } },
  { workspaceId: { not: workspaceId } },  // 拒绝不同 workspace 的团队
]
```

**删除 workspace 用户时**（`packages/amplication-server/src/core/workspace/workspace.service.ts` 的 `deleteUser`，L565-L587）：
```typescript
if (currentUser.workspace.id !== user.workspace.id) {
  throw new ConflictException("The requested user is not in the current user's workspace");
}
```

**关联 Resource 与 Git 仓库时**（`packages/amplication-server/src/core/git/git.provider.service.ts` 的 `connectResourceToNewRemoteGitRepository`，L283-L320）：
```typescript
// 显式验证 resource 和 gitOrganization 属于同一 workspace
if (resource.project.workspaceId !== gitOrganization.workspace.id) {
  throw new AmplicationError(
    "The resource does not belong to the same workspace as the git organization"
  );
}
```

### 5.3 请求守卫层：VALIDATION_FUNCTIONS 全覆盖 + 两个例外

如 4.2 节所述，`AuthorizableOriginParameter` 枚举中定义的 26 种资源类型全部对应了验证函数，其中：
- **23 种**：按标准模式追溯 workspace 归属
- **1 种（None）**：直接放行（用于无需绑定资源的操作）
- **1 种（GitRepositoryId）**：只验证 ID 存在性，不验证 workspace 归属（间接靠权限要求保障）
- **1 种（ApiTokenId）**：按用户所有权验证（userId 匹配）

### 5.4 数据查询层：WorkspaceId 注入

使用 `@InjectContextValue` 装饰器在查询前自动注入当前用户的 workspaceId 到查询条件中（见 `packages/amplication-server/src/core/team/team.resolver.ts` L40-L47），防止越权查询其他 workspace 的数据。

```typescript
@Query(() => [Team])
@InjectContextValue(InjectableOriginParameter.WorkspaceId, "where.workspace.id")
async teams(@Args() args: TeamFindManyArgs): Promise<Team[]> { ... }
```

此外，`ProjectService` 的 `commit` 和 `getPendingChanges` 等方法中，查询资源时也强制加入 workspace 用户归属检查（`packages/amplication-server/src/core/project/project.service.ts` L180-L201）：

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

注意 `packages/amplication-server/src/models/User.ts` 通过 `account` 关联 Account，一个 Account 可以在**多个 Workspace** 中各有一个 User 记录。

- Account 代表登录账号（邮箱、密码、GitHub 身份等）
- User 代表该账号在**某个特定 Workspace** 中的身份（含 isOwner、teams、workspace 等）

切换工作区的逻辑在 `packages/amplication-server/src/core/auth/auth.service.ts` 的 `setCurrentWorkspace`（L366-L388）：通过 accountId + workspaceId 找到对应的 User，重新生成 JWT Token（包含新 workspace 的 permissions）。

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
       │     ┌─────────────────────────────────────────────┐
       │     │ 标准路径：追溯 workspaceId === user.workspaceId │
       │     │ 例外1 (None): 直接放行                         │
       │     │ 例外2 (GitRepositoryId): 只查 ID 存在          │
       │     │ 例外3 (ApiTokenId): 检查 userId 匹配           │
       │     └─────────────────────────────────────────────┘
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

## 8. 关键文件索引（相对路径）

| 功能 | 相对路径 |
|-----|---------|
| 核心模型目录 | `packages/amplication-server/src/models/` |
| 用户权限计算 | `packages/amplication-server/src/core/user/user.service.ts` |
| 授权守卫 | `packages/amplication-server/src/guards/gql-auth.guard.ts` |
| 权限服务（含两阶段校验） | `packages/amplication-server/src/core/permissions/permissions.service.ts` |
| 资源归属验证函数（含例外情况） | `packages/amplication-server/src/core/permissions/validation-functions.ts` |
| 授权装饰器 | `packages/amplication-server/src/decorators/authorizeContext.decorator.ts` |
| 团队服务 | `packages/amplication-server/src/core/team/team.service.ts` |
| 团队 Resolver（装饰器使用示例） | `packages/amplication-server/src/core/team/team.resolver.ts` |
| 工作区服务 | `packages/amplication-server/src/core/workspace/workspace.service.ts` |
| 默认团队/角色配置 | `packages/amplication-server/src/core/workspace/constants.ts` |
| 可授权参数枚举 | `packages/amplication-server/src/enums/AuthorizableOriginParameter.ts` |
| 认证服务（含 Token 生成、工作区切换） | `packages/amplication-server/src/core/auth/auth.service.ts` |
| Git 服务（含 workspace 一致性检查） | `packages/amplication-server/src/core/git/git.provider.service.ts` |
| Git Resolver（装饰器使用示例） | `packages/amplication-server/src/core/git/git.resolver.ts` |
| 项目服务（含查询层 workspace 校验） | `packages/amplication-server/src/core/project/project.service.ts` |
