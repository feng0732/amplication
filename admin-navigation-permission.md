# Admin 控制台导航与权限菜单协作机制分析

## 1. 整体架构概览

Amplication 的 Admin 控制台采用 **前后端分离 + 多层权限校验** 的协作模式：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              前端 (amplication-client)                        │
│                                                                              │
│  ┌──────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐  │
│  │ WorkspaceLayout  │───▶│   AppContextProvider │───▶│  permissions Hook   │  │
│  │ (根布局容器)      │    │  (全局上下文注入)     │    │  (usePermissions)   │  │
│  └──────────────────┘    └─────────────────────┘    └─────────────────────┘  │
│           │                        │                        │                │
│           ▼                        ▼                        ▼                │
│  ┌──────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐  │
│  │ WorkspaceHeader  │    │ currentWorkspace    │    │ canPerformTask()    │  │
│  │ WorkspaceNav.    │    │ currentProject      │    │ isAdmin             │  │
│  │ Side Navigation  │    │ currentResource     │    │ allowedTasks{}      │  │
│  └──────────────────┘    └─────────────────────┘    └─────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │ GraphQL (GET_PERMISSIONS)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           后端 (amplication-server)                           │
│                                                                              │
│  ┌──────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐  │
│  │  GqlAuthGuard    │───▶│ PermissionsService  │───▶│ VALIDATION_FUNCTIONS│  │
│  │ (守卫层)         │    │ (权限校验核心)       │    │ (来源参数验证)      │  │
│  └──────────────────┘    └─────────────────────┘    └─────────────────────┘  │
│           │                        │                        │                │
│           ▼                        ▼                        ▼                │
│  ┌──────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐  │
│  │ @AuthorizeContext│    │ validateAccess()    │    │ WorkspaceId 校验    │  │
│  │ (装饰器声明)      │    │ validatePermissions │    │ ResourceId 校验     │  │
│  └──────────────────┘    └─────────────────────┘    └─────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 前端导航组件与可见性控制

### 2.1 导航层级结构

Admin 控制台导航分为三个层级：

| 层级 | 组件 | 文件位置 | 说明 |
|------|------|----------|------|
| L1 顶部导航 | `WorkspaceHeader` | [WorkspaceHeader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceHeader.tsx#L42-L218) | Logo、WorkspaceNavigation、通知、用户信息 |
| L2 面包屑导航 | `WorkspaceNavigation` | [WorkspaceNavigation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceNavigation.tsx#L29-L98) | Workspace → Project → Resource → Breadcrumbs |
| L3 侧边导航 | `VerticalNavigation` / `InnerTabLink` | [ModuleNavigationList.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Modules/ModuleNavigationList.tsx#L31-L139) / 设置页左侧标签栏 | 模块列表、设置 Tab 导航 |

### 2.2 路由配置驱动导航显示

导航菜单项并非硬编码，而是由路由配置自动生成：

**路由定义** ([appRoutes.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/routes/appRoutes.tsx#L5-L20))：

```typescript
export interface RouteDef {
  path: string;
  displayName?: string;   // 用于 Tab 名称
  iconName?: string;      // 导航图标
  tabRoutes?: RouteDef[]; // 子 Tab 路由
  isAnalytics?: boolean;  // 是否埋点
  permission?: boolean;   // 是否需要权限
  // ...
}
```

**Settings 页面导航生成** ([WorkspaceSettingsPage.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceSettingsPage.tsx#L19-L60))：

```tsx
const WorkspaceSettingsPage: React.FC<Props> = ({ match, tabRoutes, tabRoutesDef }) => {
  const { tabs } = useTabRoutes(tabRoutesDef);

  const tabItems: TabItem[] = useMemo(() => {
    return [
      { name: OVERVIEW, to: match.url, exact: true, iconName: "app-settings" },
      ...(tabs || []),  // 从路由配置自动生成 Users/Teams/Roles/Properties/Tokens
    ];
  }, [tabs, match.url]);

  return (
    <PageContent
      sideContent={
        <>
          {tabItems.map((tab) => (
            <InnerTabLink to={tab.to} icon={tab.iconName} exact={tab.exact}>
              {tab.name}
            </InnerTabLink>
          ))}
        </>
      }
    >
      {/* ... */}
    </PageContent>
  );
};
```

### 2.3 前端权限可见性控制

前端通过 `usePermissions()` Hook 获取权限后，使用 `canPerformTask()` 控制菜单项和操作按钮的可见性：

**权限 Hook** ([usePermissions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/hooks/usePermissions.ts#L12-L44))：

```typescript
export interface IPermissions {
  allowedTasks: Record<RolesPermissions, boolean>;
  canPerformTask: (task: RolesPermissions) => boolean;
  isAdmin: boolean;
}

const usePermissions = (): IPermissions => {
  const [allowedTasks, setAllowedTasks] = useState<Record<RolesPermissions, boolean>>({});
  const [isAdmin, setIsAdmin] = useState<boolean>(false);

  useQuery<{ permissions: string[] }>(GET_PERMISSIONS, {
    onCompleted: (data) => {
      setIsAdmin(data?.permissions.includes("*") || false);
      setAllowedTasks(() => {
        return data.permissions.reduce(
          (acc, permission) => ({ ...acc, [permission]: true }),
          {} as Record<RolesPermissions, boolean>
        );
      });
    },
  });

  const canPerformTask = (task: RolesPermissions) => {
    return isAdmin || allowedTasks[task] || false;
  };

  return { allowedTasks, canPerformTask, isAdmin };
};
```

**权限注入到全局上下文** ([appContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/context/appContext.tsx#L12-L68) + [WorkspaceLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx#L111-L242))：

```tsx
// WorkspaceLayout.tsx
const permissions = usePermissions();

return (
  <AppContextProvider
    newVal={{
      // ... 其他上下文
      permissions,  // 注入到全局上下文
    }}
  >
    {/* 子组件树 */}
  </AppContextProvider>
);
```

**组件中使用权限控制可见性**：

| 场景 | 文件 | 权限控制代码 |
|------|------|-------------|
| 创建角色按钮 | [RoleList.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Roles/RoleList.tsx#L31-L97) | `const canCreate = permissions.canPerformTask("role.create");` → 条件渲染 `NewRole` |
| 创建团队按钮 | [TeamList.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Teams/TeamList.tsx#L27-L85) | `const canCreate = permissions.canPerformTask("team.create");` → 条件渲染 `NewTeam` |
| 删除成员按钮 | [MemberList.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/MemberList.tsx#L48-L87) | `const canRemoveUser = permissions.canPerformTask("workspace.member.remove");` → 传递给 `MemberListItem` |
| 管理员标记 | [RoleList.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Roles/RoleList.tsx#L113-L115) | `role.permissions.includes("*")` → 显示 "Admin" Chip |

---

## 3. 工作区上下文（Workspace Context）传递机制

### 3.1 工作区上下文从 URL 到组件的传递链路

```
URL: /{workspaceId}/{projectId}/{resourceId}/...
        │
        ▼
React Router 匹配 (appRoutes.tsx)
        │
        ▼
WorkspaceLayout 组件接收 match.params.workspace
        │
        ▼
useWorkspaceSelector() → GET_CURRENT_WORKSPACE 查询
        │
        ▼
handleSetCurrentWorkspace() → SET_CURRENT_WORKSPACE mutation
        │
        ▼
AppContextProvider 注入 currentWorkspace
        │
        ▼
所有子组件通过 useContext(AppContext) 访问
```

**工作区切换逻辑** ([useCurrentWorkspace.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/hooks/useCurrentWorkspace.ts#L12-L49))：

```typescript
const useCurrentWorkspace = (authenticated: boolean) => {
  const [getCurrentWorkspace, { loading, data }] = useLazyQuery<TData>(
    GET_CURRENT_WORKSPACE,
    { onError: (error) => { if (error.message === "Unauthorized") { unsetToken(); history.push("/login"); } } }
  );

  // 路径为 "/" 时，获取当前工作区并跳转
  useEffect(() => {
    if (location.pathname !== "/") return;
    authenticated && getCurrentWorkspace();
  }, [authenticated, location.pathname]);

  // 获取到工作区后，跳转到 /{workspaceId}
  useEffect(() => {
    if (!(data && data.currentWorkspace)) return;
    location.pathname === "/" && history.push({ pathname: `/${data.currentWorkspace.id}` });
  }, [data, history, location]);
};
```

**后端 currentWorkspace 查询** ([workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L76-L92))：

```typescript
@Query(() => Workspace, { nullable: true })
async currentWorkspace(@UserEntity() currentUser: User): Promise<Workspace | null> {
  await this.analytics.trackWithContext({ properties: {}, event: EnumEventType.WorkspaceSelected });
  await this.userService.setLastActivity(currentUser.id);
  const externalId = await this.userService.setNotificationRegistry(currentUser);
  return { ...currentUser.workspace, externalId };
}
```

> **关键点**：`currentWorkspace` 直接从 JWT Token 解码后的 `AuthUser` 对象中获取，无需额外查询数据库。这意味着工作区上下文是 **Token 绑定** 的，切换工作区需要重新生成 Token。

**切换工作区的 Mutation** ([auth.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts)) + [SetCurrentWorkspaceArgs.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/dto/SetCurrentWorkspaceArgs.ts)：

```typescript
// workspaceQueries.ts
export const SET_CURRENT_WORKSPACE = gql`
  mutation setCurrentWorkspace($workspaceId: String!) {
    setCurrentWorkspace(data: { id: $workspaceId }) {
      token
    }
  }
`;
```

### 3.2 AppContext 中与导航/权限相关的字段

[appContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/context/appContext.tsx#L12-L141) 中定义的关键字段：

| 字段 | 类型 | 导航用途 |
|------|------|---------|
| `currentUser` | `models.User` | 用户身份、头像显示 |
| `currentWorkspace` | `models.Workspace` | 工作区名称、导航前缀 URL、订阅信息 |
| `currentProject` | `models.Project` | 项目选择器、项目级导航 |
| `currentResource` | `models.Resource` | 资源导航选择器 |
| `projectsList` | `models.Project[]` | 项目下拉菜单 |
| `resources` | `models.Resource[]` | 资源导航列表 |
| `permissions` | `IPermissions` | 控制菜单项/按钮可见性 |

---

## 4. 后端授权边界与权限检查机制

### 4.1 权限类型定义

所有权限字符串统一定义在 [roles-permissions.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/libs/util/roles-types/src/lib/roles-permissions.types.ts#L1-L98)：

| 权限分组 | 权限项 | 说明 |
|---------|--------|------|
| Admin | `*` | 超级管理员，拥有所有权限 |
| Workspace | `workspace.member.invite`、`workspace.member.remove`、`workspace.settings.edit` | 工作区成员与设置 |
| Team | `team.create`、`team.delete`、`team.edit`、`team.member.add`、`team.member.remove` | 团队管理 |
| Role | `role.create`、`role.delete`、`role.edit` | 角色管理 |
| Project | `project.create`、`project.delete`、`project.settings.edit` | 项目管理 |
| Resource | `resource.create`、`resource.delete`、`resource.*.edit`、`resource.setPermissions` 等 | 资源管理 |
| Git | `git.org.create`、`git.org.delete`、`git.repo.settings.edit` 等 | Git 集成 |
| Blueprint | `blueprint.create`、`blueprint.delete`、`blueprint.edit` | 蓝图管理 |
| Property | `property.create`、`property.delete`、`property.edit` | 自定义属性 |
| PrivatePlugin | `privatePlugin.create`、`privatePlugin.edit` 等 | 私有插件 |
| ApiToken | `apiToken.create` | API Token |

### 4.2 GraphQL 守卫授权流程

所有 GraphQL 请求都经过 `GqlAuthGuard` 进行授权校验：

**守卫核心逻辑** ([gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L19-L84))：

```typescript
@Injectable()
export class GqlAuthGuard extends AuthGuard("jwt") {
  constructor(
    private readonly reflector: Reflector,
    private permissionsService: PermissionsService
  ) { super(); }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // Step 1: JWT 认证（Passport）
    if (!(await super.canActivate(context))) return false;

    const ctx = GqlExecutionContext.create(context);
    const req = this.getRequest(context);
    const currentUser = req.user;  // AuthUser: User + account + workspace + permissions[]
    const handler = context.getHandler();
    const requestArgs = ctx.getArgByIndex(1);

    // Step 2: 基于 @AuthorizeContext 装饰器的授权校验
    return await this.authorizeContext(handler, requestArgs, currentUser);
  }

  authorizeContext(handler: Function, requestArgs: any, user: AuthUser): Promise<boolean> {
    const parameters = this.getAuthorizeContextParameters(handler);
    if (!parameters) return Promise.resolve(true);  // 无装饰器则放行

    const { parameterType, parameterPath, requiredPermissions } = parameters;
    const parameterValue = get(requestArgs, parameterPath);  // 从参数中提取 originId

    return this.permissionsService.validateAccess(
      user, parameterType, parameterValue, requiredPermissions
    );
  }
}
```

### 4.3 @AuthorizeContext 装饰器

**装饰器定义** ([authorizeContext.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/decorators/authorizeContext.decorator.ts#L21-L37))：

```typescript
export const AuthorizeContext = (
  parameterType: AuthorizableOriginParameter,  // 来源参数类型（如 WorkspaceId）
  parameterPath: string,                       // 参数路径（如 "where.id"）
  permissions?: RolesPermissions[] | RolesPermissions  // 需要的权限
): CustomDecorator<string> => {
  // ... 通过 SetMetadata 设置元数据供 GqlAuthGuard 读取
};
```

**使用示例** ([workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L58-L186))：

```typescript
// 仅校验工作区归属
@Query(() => Workspace, { nullable: true })
@AuthorizeContext(AuthorizableOriginParameter.WorkspaceId, "where.id")
async workspace(@Args() args: FindOneArgs): Promise<Workspace | null> {
  return this.workspaceService.getWorkspace(args);
}

// 校验工作区归属 + 需要 workspace.settings.edit 权限
@Mutation(() => Workspace, { nullable: true })
@AuthorizeContext(
  AuthorizableOriginParameter.WorkspaceId,
  "where.id",
  "workspace.settings.edit"
)
async updateWorkspace(@Args() args: UpdateOneWorkspaceArgs): Promise<Workspace | null> {
  return this.workspaceService.updateWorkspace(args);
}

// 不需要来源参数，仅校验权限（全局工作区级）
@Mutation(() => Invitation, { nullable: true })
@AuthorizeContext(AuthorizableOriginParameter.None, "", "workspace.member.invite")
async inviteUser(@UserEntity() currentUser: User, @Args() args: InviteUserArgs): Promise<Invitation> {
  return this.workspaceService.inviteUser(currentUser, args);
}
```

### 4.4 AuthorizableOriginParameter：可授权的资源类型

[AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts#L7-L34) 定义了 **33 种** 可授权参数类型，每种类型都有对应的数据库验证函数：

| 参数类型 | 验证逻辑 | 返回字段 |
|---------|---------|---------|
| `None` | 直接放行 | - |
| `WorkspaceId` | `originId === workspace.id` | - |
| `ProjectId` | 检查项目是否属于当前工作区 | `requestedProjectId` |
| `ResourceId` | 检查资源（未删除/未归档）是否属于当前工作区的项目 | `requestedResourceId` |
| `TeamId` / `RoleId` / `BlueprintId` | 检查是否属于当前工作区 | - |
| `EntityId` / `BlockId` / `BuildId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| `CommitId` | 通过 project 关联到工作区 | `requestedProjectId` |
| `EntityFieldId` | 通过 entityVersion → entity → resource 关联 | `requestedResourceId` |
| `ApiTokenId` | 检查是否属于当前用户 | - |

**验证函数实现** ([validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L30-L508))：

```typescript
// 通用资源级验证函数
const checkByResourceParameters = (originId: string, workspaceId: string) => ({
  where: {
    id: originId,
    resource: {
      deletedAt: null,
      project: { workspace: { id: workspaceId } },
    },
  },
  select: { resourceId: true },
});

// 示例：ProjectId 验证
[AuthorizableOriginParameter.ProjectId]: async (prisma, originId, workspaceId) => {
  const matching = await prisma.project.count({
    where: {
      deletedAt: null,
      id: originId,
      workspace: { id: workspaceId },
    },
  });
  return {
    canAccessWorkspace: matching === 1,
    requestedProjectId: originId,
  };
};
```

### 4.5 PermissionsService 权限验证核心

**权限验证流程** ([permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L18-L210))：

```
validateAccess(user, originType, originId, requiredPermissions)
        │
        ▼
Step 1: VALIDATION_FUNCTIONS[originType]()  →  检查资源归属
        │  canAccessWorkspace = false ?  →  直接拒绝
        │  提取 requestedResourceId / requestedProjectId
        ▼
validatePermissions(user, requiredPermissions, resourceId, projectId)
        │
        ├─── 无 requiredPermissions ?  →  放行 ✅
        │
        ├─── user.permissions.includes("*") ?  →  管理员放行 ✅
        │
        ├─── 团队级权限匹配:
        │    matchPermissions(required, user.permissions)  →  匹配成功 ✅
        │
        └─── 资源/项目级权限匹配:
             getUserResourceOrProjectPermissions()
             → 查询 teamAssignment（团队分配到资源/项目配置）
             → 收集所有关联角色的 permissions
             → matchPermissions()  →  匹配成功 ✅
             → 否则拒绝 ❌
```

**核心代码**：

```typescript
async validatePermissions(
  user: AuthUser,
  requiredPermissions: RolesPermissions[] | undefined,
  requestedResourceId?: string,
  requestedProjectId?: string
): Promise<boolean> {
  if (!requiredPermissions || requiredPermissions.length === 0) return true;
  if (user.permissions.includes("*")) return true;  // Admin

  // 团队级（工作区级）权限
  if (this.matchPermissions(requiredPermissions, user.permissions)) return true;

  // 资源/项目级权限（通过团队分配）
  if (await this.validateTeamAssignmentPermissions(
    user, requiredPermissions, requestedResourceId, requestedProjectId
  )) return true;

  return false;
}

private matchPermissions(permissionsToMatch: string[], userPermissions: string[]): boolean {
  return userPermissions.includes("*") ||
    permissionsToMatch.some((r) => userPermissions.includes(r));
}
```

---

## 5. 导航与权限协作的完整数据流

### 5.1 用户登录后的初始化流程

```
1. 用户登录
   │
   ▼
2. JWT Token 生成，包含 workspace + permissions[]
   │  (AuthUser = User + Account + Workspace + RolesPermissions[])
   ▼
3. 前端重定向到 / → useCurrentWorkspace() 触发
   │
   ▼
4. GET_CURRENT_WORKSPACE 查询（后端从 Token 中取 workspace）
   │
   ▼
5. 前端跳转 /{workspaceId} → WorkspaceLayout 挂载
   │
   ├─── useWorkspaceSelector() → workspacesList、currentWorkspace
   ├─── useProjectSelector() → currentProject、projectsList
   ├─── useResources() → currentResource、resources
   ├─── usePermissions() → GET_PERMISSIONS 查询
   │       │
   │       ▼
   │     isAdmin、allowedTasks{}、canPerformTask()
   │
   ▼
6. AppContextProvider 注入所有上下文
   │
   ▼
7. WorkspaceHeader 渲染：
   ├─── Logo → /{workspaceId}
   ├─── WorkspaceNavigation: Workspace名 → Project选择器 → Resource选择器
   └─── 用户菜单 / 通知
   │
   ▼
8. 内层路由组件通过 useAppContext() 获取 permissions
   │
   ▼
9. 根据 permissions.canPerformTask() 控制菜单项/按钮可见性
```

### 5.2 访问受保护资源的授权流程

```
用户点击导航链接 /{workspaceId}/settings/roles/{roleId}
        │
        ▼
前端路由渲染 Role.tsx，发起 GraphQL 查询（如 role(where: { id: roleId })）
        │
        ▼
后端 GqlAuthGuard.canActivate()
        │
        ├─── Step 1: JWT 认证 → 解码出 AuthUser{workspace, permissions[]}
        │
        └─── Step 2: authorizeContext()
                │
                ├─── 读取 @AuthorizeContext(AuthorizableOriginParameter.RoleId, "where.id")
                │
                ├─── get(requestArgs, "where.id") → 提取 roleId
                │
                ▼
        PermissionsService.validateAccess(user, RoleId, roleId, undefined)
                │
                ├─── VALIDATION_FUNCTIONS[RoleId](prisma, roleId, workspace.id)
                │       │
                │       ▼
                │     prisma.role.count({ where: { id: roleId, workspace: { id: workspace.id }}})
                │       │
                │       ├─── count === 0 → 不属于该工作区 → 返回 false ❌
                │       └─── count === 1 → canAccessWorkspace = true ✅
                │
                └─── requiredPermissions 为空 → 直接放行 ✅
                │
                ▼
        Resolver 方法执行，返回数据
```

### 5.3 创建操作的权限检查（含 requiredPermissions）

```
用户点击 "New Role" 按钮 → 权限检查
        │
        ▼
前端: permissions.canPerformTask("role.create") === true ? 显示按钮 : 隐藏
        │
        ▼
用户填写表单提交 → createRole(data: {...}) mutation
        │
        ▼
后端 GqlAuthGuard:
        │
        ├─── @AuthorizeContext(AuthorizableOriginParameter.None, "", "role.create")
        │
        ├─── parameterType = None → VALIDATION_FUNCTIONS[None]() → canAccessWorkspace = true
        │
        ▼
        PermissionsService.validatePermissions(user, ["role.create"], undefined, undefined)
        │
        ├─── user.permissions.includes("*") → Admin ✅
        │
        └─── user.permissions.includes("role.create") → 有该权限 ✅
                │
                └─── 两者都无 → 返回 false ❌（403 Forbidden）
```

---

## 6. 关键设计洞察

### 6.1 前端可见性 ≠ 后端安全

- **前端**：`canPerformTask()` 仅控制 UI 可见性，是 UX 优化手段
- **后端**：`GqlAuthGuard` + `@AuthorizeContext` 是真正的安全边界，每个 GraphQL 操作都独立校验
- 即使前端绕过 UI 限制直接发起 GraphQL 请求，后端仍会拒绝未授权操作

### 6.2 工作区作为授权锚点

- 所有资源（Project、Resource、Entity、Role、Team 等）都必须归属于某个 Workspace
- 授权的第一步永远是 **验证目标资源是否属于当前用户的 active workspace**
- 这通过 `AuthorizableOriginParameter` + `VALIDATION_FUNCTIONS` 实现，杜绝跨工作区访问

### 6.3 三级权限模型

| 层级 | 存储位置 | 适用场景 |
|------|---------|---------|
| L1 工作区级 | JWT Token (`user.permissions`) | 全局操作：邀请成员、创建团队、管理角色 |
| L2 项目级 | `teamAssignment` (项目配置资源) | 项目设置、项目级权限分配 |
| L3 资源级 | `teamAssignment` (具体资源) | 单个服务/消息代理的编辑权限 |

### 6.4 权限声明式编程

后端使用装饰器 `@AuthorizeContext(OriginType, paramPath, permissions)` 声明式定义授权规则，与业务逻辑解耦：

- **优点**：Resolver 代码纯净、授权逻辑统一、易审计
- **不足**：需要确保每个需要保护的 Resolver 方法都正确添加了装饰器（目前有 34 个 resolver 文件使用）

---

## 7. 核心文件索引

| 模块 | 文件 | 职责 |
|------|------|------|
| **前端导航** | [WorkspaceLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx) | 根布局、上下文注入 |
| | [WorkspaceHeader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceHeader.tsx) | 顶部导航栏 |
| | [WorkspaceNavigation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceNavigation.tsx) | 面包屑导航 |
| | [WorkspaceSettingsPage.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceSettingsPage.tsx) | 设置页侧边导航 |
| | [appRoutes.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/routes/appRoutes.tsx) | 路由配置（驱动导航生成） |
| **前端权限** | [usePermissions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/hooks/usePermissions.ts) | 权限 Hook |
| | [appContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/context/appContext.tsx) | 全局上下文定义 |
| | [workspaceQueries.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/queries/workspaceQueries.ts) | GraphQL 查询 |
| **后端授权** | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) | GraphQL 守卫 |
| | [authorizeContext.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/decorators/authorizeContext.decorator.ts) | 授权装饰器 |
| | [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts) | 权限验证服务 |
| | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts) | 来源参数验证 |
| | [AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts) | 可授权参数枚举 |
| | [auth/types.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/types.ts) | AuthUser 类型 |
| **共享类型** | [roles-permissions.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/libs/util/roles-types/src/lib/roles-permissions.types.ts) | 权限字符串定义 |
