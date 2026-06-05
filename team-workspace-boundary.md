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

#### 4.2.3 深度分析：GitRepositoryId 跨工作区风险链路

下面沿着**仓库删除**和**仓库设置更新**两条调用链路，逐层分析 GitRepository 如何绑定到 workspace、哪里缺失了绑定检查。

---

##### （一）数据库层：GitRepository 与 Workspace 的绑定方式

Prisma Schema 定义（相对路径 `packages/amplication-prisma-db/prisma/schema.prisma`）：

```prisma
model GitOrganization {
  id            String          @id @default(cuid())
  // ...
  workspaceId   String                              // ✅ 直接字段
  workspace     Workspace       @relation(fields: [workspaceId], ...)
  gitRepositories GitRepository[]
}

model GitRepository {
  id                String          @id @default(cuid())
  // ...
  gitOrganizationId String                              // ⚠️ 无 workspaceId，仅有此间接关联字段
  gitOrganization   GitOrganization @relation(fields: [gitOrganizationId], ...)
  resources         Resource[]
}
```

**绑定链**：`GitRepository.id → GitRepository.gitOrganizationId → GitOrganization.workspaceId → Workspace.id`

GitRepository 本身没有 `workspaceId` 字段，必须通过 GitOrganization 间接追溯。

---

##### （二）链路一：deleteGitRepository（删除仓库）

完整调用链：**Resolver → GqlAuthGuard → PermissionsService → GitProviderService**

###### Step 1: Resolver 层（`packages/amplication-server/src/core/git/git.resolver.ts` L94-L104）

```typescript
@Mutation(() => Resource)
@AuthorizeContext(
  AuthorizableOriginParameter.GitRepositoryId,   // ← 参数类型
  "gitRepositoryId",                             // ← 参数取值路径
  "git.repo.disconnect"                          // ← 所需权限
)
async deleteGitRepository(
  @Args() args: DeleteGitRepositoryArgs          // ← { gitRepositoryId: string }
): Promise<boolean> {
  return this.gitService.deleteGitRepository(args);
}
```

装饰器声明：
- 待验证参数类型：`GitRepositoryId`
- 参数在请求中的位置：`args.gitRepositoryId`
- 需要权限：`"git.repo.disconnect"`（GitRepositoryPermissions 中定义，见 `libs/util/roles-types/src/lib/roles-permissions.types.ts` L33-L38）

###### Step 2: GqlAuthGuard 守卫 → PermissionsService.validateAccess

进入 [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/41-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts)，依次调用：
- `validateAccess()` → 阶段一（工作区隔离验证）
- `validatePermissions()` → 阶段二（权限匹配验证）

###### Step 3: 阶段一 — VALIDATION_FUNCTIONS[GitRepositoryId]（**⚠️ 缺失 workspace 检查**）

代码见 `packages/amplication-server/src/core/permissions/validation-functions.ts` L66-L77：

```typescript
[AuthorizableOriginParameter.GitRepositoryId]: async (
  prisma: PrismaService,
  originId: string,     // 即 gitRepositoryId
  workspaceId: string   // 当前用户的 workspaceId
) => {
  const matching = await prisma.gitRepository.count({
    where: {
      id: originId,
      // ⚠️ 关键缺失点：这里没有 join gitOrganization 校验 workspaceId
      // 正确写法应类似：
      // gitOrganization: { workspaceId: workspaceId }
    },
  });
  return { canAccessWorkspace: matching === 1 };
};
```

**结论（阶段一）**：只要传入任意一个存在的 GitRepository ID，无论属于哪个 workspace，`canAccessWorkspace` 都会返回 `true`。**此处完全缺失 workspace 归属验证。**

作为对比，同文件中 `GitOrganizationId` 的验证函数（L51-L65）是正确的：
```typescript
where: {
  id: originId,
  workspace: { id: workspaceId },  // ✅ 有 workspace 归属检查
}
```

###### Step 4: 阶段二 — validatePermissions

代码见 `packages/amplication-server/src/core/permissions/permissions.service.ts` L49-L87，按以下顺序检查：
1. requiredPermissions 为空？→ 通过
2. user.permissions 含 `"*"`？→ 通过（Owner / Admins 团队成员）
3. user.permissions（从 Team 级 Role 收集）是否包含 `"git.repo.disconnect"`？→ 通过
4. 资源级 TeamAssignment 权限？→ GitRepository 本身不是 Resource，不走此分支

**说明**：阶段二只校验**当前用户在自己 workspace 中是否拥有该权限字符串**，不校验目标 GitRepository 与当前 workspace 的关系。

###### Step 5: 服务层 — GitProviderService.deleteGitRepository（**⚠️ 再次缺失 workspace 检查**）

代码见 `packages/amplication-server/src/core/git/git.provider.service.ts` L433-L451：

```typescript
async deleteGitRepository(args: DeleteGitRepositoryArgs): Promise<boolean> {
  const gitRepository = await this.prisma.gitRepository.findUnique({
    where: {
      id: args.gitRepositoryId,
      // ⚠️ 缺失点：没有过滤 gitOrganization.workspaceId
    },
  });

  if (isEmpty(gitRepository)) {
    throw new AmplicationError(INVALID_GIT_REPOSITORY_ID);
  }

  await this.prisma.gitRepository.delete({
    where: {
      id: args.gitRepositoryId,
      // ⚠️ 缺失点：同样没有 workspace 归属过滤
    },
  });

  return true;
}
```

**结论（服务层）**：只要通过了前两阶段的守卫，服务层直接按 ID 删除，**完全没有再次验证目标仓库属于当前 workspace**。

---

##### （三）链路二：updateGitRepository（更新仓库设置）

完整调用链与删除相同，以下仅列出各层差异：

###### Step 1: Resolver 层（`packages/amplication-server/src/core/git/git.resolver.ts` L106-L116）

```typescript
@Mutation(() => GitRepository)
@AuthorizeContext(
  AuthorizableOriginParameter.GitRepositoryId,   // ← 同样的参数类型
  "where.id",                                    // ← 参数取值路径不同
  "git.repo.settings.edit"                       // ← 所需权限不同
)
async updateGitRepository(
  @Args() args: UpdateGitRepositoryArgs          // ← { where: { id }, data: {...} }
): Promise<GitRepository> {
  return this.gitService.updateGitRepository(args);
}
```

###### Step 2-4: Guard + PermissionsService

与 deleteGitRepository **完全相同**：
- 阶段一调用同一个 `VALIDATION_FUNCTIONS[GitRepositoryId]` → **缺失 workspace 检查**
- 阶段二检查权限 `"git.repo.settings.edit"` → 仅检查用户自己 workspace 中是否有该权限

###### Step 5: 服务层 — GitProviderService.updateGitRepository（**⚠️ 完全透传，无任何校验**）

代码见 `packages/amplication-server/src/core/git/git.provider.service.ts` L453-L457：

```typescript
async updateGitRepository(
  args: UpdateGitRepositoryArgs
): Promise<GitRepository> {
  return this.prisma.gitRepository.update(args);  // ⚠️ 直接透传给 Prisma，无任何校验
}
```

**结论**：服务层没有任何 workspace 归属校验，直接将 Prisma 的 `args`（含 `where` 和 `data`）透传执行 `update`。**风险比 delete 更高**，因为可以修改任意 workspace 的仓库设置。

---

##### （四）对比：哪些操作做对了 Workspace 绑定

在同一个 `GitProviderService` 中，以下操作**正确地**做了 workspace 一致性校验：

| 方法 | 相对路径 & 行号 | 校验方式 |
|-----|---------------|---------|
| `connectResourceToNewRemoteGitRepository` | `git.provider.service.ts` L283-L319 | 显式查 resource.project.workspaceId 与 gitOrganization.workspace.id 是否相等 |
| `validateGitOrganization` | `git.provider.service.ts` L333-L381 | 通过嵌套 `workspace.projects.resources.some` 查询，确保 gitOrganization 与 resource 属于同一 workspace |
| `VALIDATION_FUNCTIONS[GitOrganizationId]` | `validation-functions.ts` L51-L65 | 直接检查 `workspace.id === workspaceId` |

**但 deleteGitRepository 和 updateGitRepository 都没有复用上述校验逻辑。**

---

##### （五）风险场景总结

假设：
- **Workspace A**：ID = `ws_A`，用户 `User_A` 是 Owner（permissions = `["*"]`），Team = Admins
- **Workspace B**：ID = `ws_B`，其中有一个 GitRepository，ID = `repo_B`

攻击路径：

```
User_A 已登录 ws_A，JWT 中含 workspace=ws_A, permissions=["*"]
          │
          ▼
User_A 构造 GraphQL Mutation，传入 gitRepositoryId=repo_B
          │
          ▼
阶段一：VALIDATION_FUNCTIONS[GitRepositoryId](repo_B, ws_A)
       prisma.gitRepository.count({ where: { id: repo_B } })
       → 返回 1（repo_B 确实存在）
       → canAccessWorkspace = true  ✅ 绕过
          │
          ▼
阶段二：validatePermissions(["git.repo.disconnect"], ["*"])
       → user.permissions 含 "*"
       → 返回 true                    ✅ 绕过
          │
          ▼
服务层：prisma.gitRepository.delete({ where: { id: repo_B } })
       → repo_B 被删除               ❌ 跨 workspace 攻击成功
```

updateGitRepository 的攻击路径完全相同，危害是可以修改其他 workspace 的仓库设置（baseBranchName、name、groupName 等字段）。

---

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

### 5.3 请求守卫层：VALIDATION_FUNCTIONS 全覆盖 + 三个例外

如 4.2 节所述，`AuthorizableOriginParameter` 枚举中定义的 26 种资源类型全部对应了验证函数，其中：
- **23 种**：按标准模式追溯 workspace 归属
- **1 种（None）**：直接放行（用于无需绑定资源的操作）
- **1 种（GitRepositoryId）**：只验证 ID 存在性，**不验证 workspace 归属** ⚠️ — 服务层 `deleteGitRepository` 和 `updateGitRepository` 也缺失校验，存在跨工作区风险（详见 4.2.3 节深度分析）
- **1 种（ApiTokenId）**：按用户所有权验证（userId 匹配）— 由于一个 User 只属于一个 Workspace，天然保证了 workspace 隔离

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
| Prisma 数据库 Schema（含 GitRepository/GitOrganization 表结构） | `packages/amplication-prisma-db/prisma/schema.prisma` |
| 用户权限计算 | `packages/amplication-server/src/core/user/user.service.ts` |
| 授权守卫 | `packages/amplication-server/src/guards/gql-auth.guard.ts` |
| 权限服务（含两阶段校验） | `packages/amplication-server/src/core/permissions/permissions.service.ts` |
| 资源归属验证函数（含 GitRepositoryId 等例外情况） | `packages/amplication-server/src/core/permissions/validation-functions.ts` |
| 授权装饰器 | `packages/amplication-server/src/decorators/authorizeContext.decorator.ts` |
| 团队服务 | `packages/amplication-server/src/core/team/team.service.ts` |
| 团队 Resolver（装饰器使用示例） | `packages/amplication-server/src/core/team/team.resolver.ts` |
| 工作区服务 | `packages/amplication-server/src/core/workspace/workspace.service.ts` |
| 默认团队/角色配置 | `packages/amplication-server/src/core/workspace/constants.ts` |
| 可授权参数枚举 | `packages/amplication-server/src/enums/AuthorizableOriginParameter.ts` |
| 认证服务（含 Token 生成、工作区切换） | `packages/amplication-server/src/core/auth/auth.service.ts` |
| Git 服务（含 deleteGitRepository、updateGitRepository、connectResourceToNewRemoteGitRepository） | `packages/amplication-server/src/core/git/git.provider.service.ts` |
| Git Resolver（deleteGitRepository、updateGitRepository 装饰器声明） | `packages/amplication-server/src/core/git/git.resolver.ts` |
| Git 相关 DTO（DeleteGitRepositoryArgs、UpdateGitRepositoryArgs） | `packages/amplication-server/src/core/git/dto/` |
| 项目服务（含查询层 workspace 校验） | `packages/amplication-server/src/core/project/project.service.ts` |
| 权限字符串类型定义（含 git.repo.* 权限） | `libs/util/roles-types/src/lib/roles-permissions.types.ts` |
