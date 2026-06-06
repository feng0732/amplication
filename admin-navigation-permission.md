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
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │  L1 登录认证层 (Authentication)                                      │    │
│  │  GqlAuthGuard extends AuthGuard("jwt") → super.canActivate()         │    │
│  └────────────────────────────────────┬─────────────────────────────────┘    │
│                                       │ JWT 有效?                            │
│                                       ▼ YES                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │  L2 上下文授权层 (Context Authorization)                             │    │
│  │  authorizeContext() → 读取 @AuthorizeContext 元数据                   │    │
│  │  无装饰器? → 直接放行 (最大例外分支)                                   │    │
│  └────────────────────────────────────┬─────────────────────────────────┘    │
│                                       │ 有装饰器?                            │
│                                       ▼ YES                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │  L3 工作区归属与权限校验层 (Ownership + Permission)                   │    │
│  │  3a. VALIDATION_FUNCTIONS[originType]() → 资源归属校验                 │    │
│  │  3b. validatePermissions() → 权限字符串匹配                           │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
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
  displayName?: string;   // 用于 Tab 名称 (L14)
  iconName?: string;      // 导航图标 (L18)
  tabRoutes?: RouteDef[]; // 子 Tab 路由 (L10)
  isAnalytics?: boolean;  // 是否埋点 (L17)
  permission?: boolean;   // 仅控制是否需要登录认证（非细粒度权限）(L16)
  // ...
}
```

> **事实核实**：`RouteDef.permission` 字段**不用于**按权限过滤 Tab 显示。它仅在 [routesUtil.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/routes/routesUtil.tsx#L69-L84) 中用于判断路由是否需要登录认证（未登录则重定向到 `/login`），属于**路由级的登录认证门控**，不是细粒度权限控制。

**Settings 页面导航生成** ([WorkspaceSettingsPage.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceSettingsPage.tsx#L19-L60))：

```tsx
const WorkspaceSettingsPage: React.FC<Props> = ({ match, tabRoutes, tabRoutesDef }) => {
  const { tabs } = useTabRoutes(tabRoutesDef);  // L24

  const tabItems: TabItem[] = useMemo(() => {
    return [
      { name: OVERVIEW, to: match.url, exact: true, iconName: "app-settings" },
      ...(tabs || []),  // L34: 从路由配置自动生成 Users/Teams/Roles/Properties/Tokens
    ];
  }, [tabs, match.url]);
  // ...
};
```

> **事实核实**：Settings 页面的子标签（Users、Teams、Roles、Properties、API Tokens）**不按权限过滤**。`useTabRoutes` Hook ([useTabRoutes.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Layout/useTabRoutes.ts#L9-L42)) 中 L19-L30 仅做路由配置到 TabItem 的纯映射（`tabRoutes?.map(...)`，L20），没有调用任何 `canPerformTask()` 或权限过滤逻辑。所有已登录用户都能看到全部 Settings 子标签，权限过滤仅发生在标签页内部的操作按钮级别。

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
  // ...
  const canPerformTask = (task: RolesPermissions) => {
    return isAdmin || allowedTasks[task] || false;  // L35-L37
  };

  return { allowedTasks, canPerformTask, isAdmin };
};
```

**权限注入到全局上下文** ([appContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/context/appContext.tsx#L12-L68) + [WorkspaceLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx#L111-L242))：

```tsx
// WorkspaceLayout.tsx
const permissions = usePermissions();
return (
  <AppContextProvider newVal={{ ..., permissions }}>  {/* 注入到全局上下文 */}
    {children}
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
  // L25-L28: 路径为 "/" 时，获取当前工作区并跳转
  // L33-L36: 获取到工作区后，跳转到 /{workspaceId}
};
```

**后端 currentWorkspace 查询** ([workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L76-L92))：

```typescript
@Query(() => Workspace, { nullable: true })
async currentWorkspace(@UserEntity() currentUser: User): Promise<Workspace | null> {
  // L78-L90: 直接从 JWT Token 的 user.workspace 返回，不查数据库
  await this.analytics.trackWithContext({ properties: {}, event: EnumEventType.WorkspaceSelected });
  await this.userService.setLastActivity(currentUser.id);
  const externalId = await this.userService.setNotificationRegistry(currentUser);
  return { ...currentUser.workspace, externalId };
}
```

> **关键点**：`currentWorkspace` 直接从 JWT Token 解码后的 `AuthUser` 对象中获取，无需额外查询数据库。这意味着工作区上下文是 **Token 绑定** 的，切换工作区需要重新生成 Token。

**切换工作区的 Mutation** ([auth.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L126-L140))：

```typescript
@Mutation(() => Auth)
@UseGuards(GqlAuthGuard)
async setCurrentWorkspace(
  @UserEntity() user: User,
  @Args() args: SetCurrentWorkspaceArgs
): Promise<Auth> {
  if (!user.account) throw new Error("User has no account");
  const token = await this.authService.setCurrentWorkspace(
    user.account.id, args.data.id  // L135-L137
  );
  return { token };
}
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

## 4. 后端三层授权边界与权限检查机制

### 4.1 三层授权架构总览

后端 GraphQL 请求的授权分为**三个独立且可绕过的层级**，每层都有自己的例外分支：

| 层级 | 名称 | 核心代码 | 职责 | 例外条件 |
|------|------|---------|------|---------|
| **L1** | 登录认证层 | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L31-L35) `super.canActivate()` | 验证 JWT Token 是否有效、用户是否登录 | Resolver 无 `@UseGuards(GqlAuthGuard)`（如 signup/login） |
| **L2** | 上下文授权层 | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L61-L83) `authorizeContext()` | 读取 `@AuthorizeContext` 装饰器元数据、提取参数 | Resolver 方法无 `@AuthorizeContext` 装饰器 → **直接放行**（最大例外） |
| **L3a** | 工作区归属校验 | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L39-L77) `VALIDATION_FUNCTIONS` | 校验目标资源是否属于当前用户的 workspace | `AuthorizableOriginParameter.None` → 不查数据库；`GitRepositoryId` → 不限定 workspace；`ApiTokenId` → 按用户维度 |
| **L3b** | 权限字符串校验 | [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L49-L87) `validatePermissions()` | 校验用户是否拥有所需权限字符串 | `requiredPermissions` 为空；`user.permissions` 含 `*`（Admin） |

### 4.2 L1 登录认证层 (Authentication)

**实现位置**：[gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L19-L35)

```typescript
@Injectable()
export class GqlAuthGuard extends AuthGuard("jwt") {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    // L31-L35: Step 1 - Passport JWT 认证
    if (!(await super.canActivate(context))) {
      return false;  // Token 无效或不存在 → 拒绝
    }
    // ... 通过后进入 L2 上下文授权层
  }
}
```

**L1 例外分支（完全绕过登录认证）**：

Resolver 方法/类上未添加 `@UseGuards(GqlAuthGuard)` 装饰器，则 L1-L3 全部跳过。仅 [auth.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L59-L80) 中 3 个认证端点属于此类：

| 方法 | 行号 | 说明 |
|------|------|------|
| `signupWithBusinessEmail()` | L59-L64 | 业务邮箱注册（匿名访问） |
| `signup()` | L66-L73 | 通用注册（匿名访问） |
| `login()` | L75-L80 | 登录（匿名访问） |

### 4.3 L2 上下文授权层 (Context Authorization)

**实现位置**：[gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L53-L83)

```typescript
/* eslint-disable-next-line @typescript-eslint/ban-types */
private getAuthorizeContextParameters(handler: Function) {
  return this.reflector.get<AuthorizeContextParameters>(  // L54-L58
    AUTHORIZE_CONTEXT, handler
  );
}

authorizeContext(handler, requestArgs, user): Promise<boolean> {
  const parameters = this.getAuthorizeContextParameters(handler);  // L67

  if (!parameters) {
    return Promise.resolve(true);  // L69-L71: ⚠️ 无装饰器 → 直接放行（最大例外分支）
  }

  const { parameterType, parameterPath, requiredPermissions } = parameters;  // L73
  const parameterValue = get(requestArgs, parameterPath);  // L75: 从参数中提取 originId

  return this.permissionsService.validateAccess(  // L77-L82: 进入 L3 归属与权限校验
    user, parameterType, parameterValue, requiredPermissions
  );
}
```

**L2 例外分支（通过登录认证但跳过上下文授权）**：

以下 Resolver 方法添加了 `@UseGuards(GqlAuthGuard)` 但**未添加** `@AuthorizeContext` 装饰器，导致 L3 校验被完全跳过。

**WorkspaceResolver** ([workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L46-L236)) 类级别有 `@UseGuards(GqlAuthGuard)`（L46），但以下方法无 `@AuthorizeContext`：

| 方法 | 行号 | 类型 | 说明 |
|------|------|------|------|
| `currentWorkspace()` | L76-L92 | Query | 获取当前工作区，直接从 JWT Token 返回 |
| `projects` (ResolveField) | L94-L99 | ResolveField | 工作区下的项目列表，依赖父级授权 |
| `createWorkspace()` | L115-L128 | Mutation | 创建新工作区，无归属校验 |
| `workspaceMembers()` | L188-L197 | Query | 获取工作区成员列表，用 `currentUser.workspace.id` 查询 |
| `workspaceUsers()` | L199-L206 | Query | 获取工作区用户列表，同上 |
| `subscription` (ResolveField) | L208-L211 | ResolveField | 工作区订阅信息，依赖父级授权 |
| `gitOrganizations` (ResolveField) | L213-L217 | ResolveField | 工作区 Git 组织，依赖父级授权 |
| `provisionSubscription()` | L220-L231 | Mutation | 配置订阅，无装饰器 |
| `redeemCoupon()` | L233-L240 | Mutation | 兑换优惠券，仅有 `@UseGuards(GqlAuthGuard)` 重复标注 |

**AuthResolver** ([auth.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L34-L153)) 无类级别守卫，以下方法有 `@UseGuards(GqlAuthGuard)` 但无 `@AuthorizeContext`：

| 方法 | 行号 | 说明 |
|------|------|------|
| `me()` | L34-L38 | 直接返回当前用户 |
| `permissions()` | L40-L44 | 直接返回 `user.permissions`（来自 JWT） |
| `resourcePermissions()` | L46-L57 | 仅有 GqlAuthGuard，无来源参数校验 |
| `userApiTokens()` | L97-L101 | 用 `user.id` 查询 |
| `changePassword()` | L103-L114 | 用 `user.account` |
| `setCurrentWorkspace()` | L126-L140 | 切换工作区，需重新生成 Token，内部自行校验 |
| `completeInvitation()` | L142-L153 | 完成邀请，内部自行校验 |

### 4.4 @AuthorizeContext 装饰器

**装饰器定义** ([authorizeContext.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/decorators/authorizeContext.decorator.ts#L21-L37))：

```typescript
export const AuthorizeContext = (
  parameterType: AuthorizableOriginParameter,  // L22: 来源参数类型
  parameterPath: string,                       // L23: 参数路径（如 "where.id"）
  permissions?: RolesPermissions[] | RolesPermissions  // L24: 需要的权限
): CustomDecorator<string> => {
  const requiredPermissions = permissions
    ? Array.isArray(permissions) ? permissions : [permissions]  // L26-L30
    : undefined;
  return SetMetadata<string, AuthorizeContextParameters>(AUTHORIZE_CONTEXT, {  // L32-L36
    parameterType, parameterPath, requiredPermissions,
  });
};
```

**使用示例** ([workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L58-L186))：

```typescript
// L68-L74: 仅校验工作区归属，无 requiredPermissions
@Query(() => Workspace, { nullable: true })
@AuthorizeContext(AuthorizableOriginParameter.WorkspaceId, "where.id")
async workspace(@Args() args: FindOneArgs): Promise<Workspace | null> { ... }

// L101-L113: 校验工作区归属 + 需要 workspace.settings.edit 权限
@Mutation(() => Workspace, { nullable: true })
@AuthorizeContext(
  AuthorizableOriginParameter.WorkspaceId, "where.id", "workspace.settings.edit"
)
async updateWorkspace(@Args() args: UpdateOneWorkspaceArgs): Promise<Workspace | null> { ... }

// L130-L144: 不需要来源参数，仅校验权限（AuthorizableOriginParameter.None）
@Mutation(() => Invitation, { nullable: true })
@AuthorizeContext(AuthorizableOriginParameter.None, "", "workspace.member.invite")
async inviteUser(@UserEntity() currentUser: User, @Args() args: InviteUserArgs): Promise<Invitation> { ... }
```

### 4.5 L3a 工作区归属校验层 (Workspace Ownership)

**枚举定义**：[AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts#L7-L34) 共 **26 种**（索引 0~25）：

| 索引 | 参数类型 | 验证逻辑（[validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts)） | 返回字段 |
|-----|---------|--------------------------------------------------------------------------------------------------|---------|
| 0 | `None` | [L39-L41](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L39-L41): 直接 `return { canAccessWorkspace: true }`，**不查数据库** | - |
| 1 | `WorkspaceId` | [L42-L50](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L42-L50): `originId === workspaceId` | - |
| 2 | `ResourceId` | 检查资源（未删除/未归档）属于当前工作区的项目 | `requestedResourceId` |
| 3 | `EntityId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 4 | `EntityFieldId` | 通过 entityVersion → entity → resource 关联 | `requestedResourceId` |
| 5 | `EntityPermissionFieldId` | 通过 field → entityVersion → entity → resource | `requestedResourceId` |
| 6 | `BlockId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 7 | `ResourceRoleId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 8 | `BuildId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 9 | `ResourceVersionId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 10 | `ActionId` | 通过 deployments/builds/userAction → build/resource → workspace | - |
| 11 | `EnvironmentId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 12 | `DeploymentId` | 通过 environment → resource 关联到工作区 | - |
| 13 | `CommitId` | 通过 project 关联到工作区 | `requestedProjectId` |
| 14 | `ApiTokenId` | [L198-L211](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L198-L211): 校验 `userId === user.id`，**不校验 workspace** | - |
| 15 | `GitOrganizationId` | [L51-L65](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L51-L65): `where: { id, workspace: { id: workspaceId }}` | - |
| 16 | `GitRepositoryId` | [L66-L77](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L66-L77): ⚠️ 仅 `where: { id: originId }`，**不限定 workspaceId** | - |
| 17 | `InvitationId` | 检查是否属于当前工作区 | - |
| 18 | `ProjectId` | [L78-L96](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L78-L96): `where: { deletedAt: null, id, workspace: { id: workspaceId }}` | `requestedProjectId` |
| 19 | `UserActionId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 20 | `OutdatedVersionAlertId` | 通过 resource 关联到工作区 | `requestedResourceId` |
| 21 | `TeamId` | [L97-L112](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L97-L112): `where: { deletedAt: null, id, workspace: { id: workspaceId }}` | - |
| 22 | `CustomPropertyId` | [L113-L128](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L113-L128): 同上 | - |
| 23 | `BlueprintId` | 检查蓝图（未删除）属于当前工作区 | - |
| 24 | `RoleId` | [L129-L144](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L129-L144): `where: { deletedAt: null, id, workspace: { id: workspaceId }}` | - |
| 25 | `UserId` | 检查用户是否属于当前工作区 | - |

> **事实核实**：`GitRepositoryId`（索引 16）的验证函数 ([L66-L77](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L66-L77)) 仅执行 `prisma.gitRepository.count({ where: { id: originId } })`，**没有限定 workspaceId**，这是一个潜在的跨工作区访问漏洞。

### 4.6 L3b 权限字符串校验层 (Permission Check) + L3 前置检查

**实现位置**：[permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L18-L87)

```typescript
async validateAccess(user, originType, originId, requiredPermissions): Promise<boolean> {
  const { workspace } = user;

  // L26-L28: L3a 前置检查 → originId 为空且类型不是 None → 直接拒绝
  if (!originId && originType !== AuthorizableOriginParameter.None) {
    return false;
  }

  // L30-L35: L3a 工作区归属校验
  const validationResponse = await VALIDATION_FUNCTIONS[originType](
    this.prisma, originId, workspace.id, user
  );

  // L37-L39: 归属校验失败 → 拒绝
  if (!validationResponse || !validationResponse.canAccessWorkspace) {
    return false;
  }

  // L41-L46: L3b 权限字符串校验
  return this.validatePermissions(
    user, requiredPermissions,
    validationResponse.requestedResourceId, validationResponse.requestedProjectId
  );
}

async validatePermissions(user, requiredPermissions, requestedResourceId?, requestedProjectId?): Promise<boolean> {
  // L55-L58: L3b 例外 1 → 无 requiredPermissions → 放行
  if (!requiredPermissions || requiredPermissions.length === 0) return true;

  // L60-L63: L3b 例外 2 → Admin 用户（*）→ 放行
  if (user.permissions.includes("*")) return true;

  // L65-L72: L3b 常规 1 → 团队级（工作区级）权限匹配
  if (this.matchPermissions(requiredPermissions, user.permissions)) return true;

  // L74-L84: L3b 常规 2 → 资源/项目级权限匹配（通过 teamAssignment）
  if (await this.validateTeamAssignmentPermissions(
    user, requiredPermissions, requestedResourceId, requestedProjectId
  )) return true;

  return false;  // 都不匹配 → 拒绝
}
```

### 4.7 三层授权例外分支汇总（按层级明确区分）

以下按 **登录认证 → 上下文授权 → 工作区归属与权限校验** 三层结构，逐一列出每层的例外分支：

---

#### 4.7.1 第一层：登录认证 (Authentication) 例外

本层职责：验证 JWT Token 是否有效、用户是否已登录。

| 例外类型 | 触发条件 | 代码位置 | 结果 | 安全影响 |
|---------|---------|---------|------|---------|
| 匿名端点 | Resolver 方法/类上未标注 `@UseGuards(GqlAuthGuard)` | [auth.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L59-L80) | L1/L2/L3 全部校验跳过 | 符合预期（signup/login 需匿名访问） |

具体匿名端点共 3 个：
- `signupWithBusinessEmail()` [L59-L64](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L59-L64)
- `signup()` [L66-L73](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L66-L73)
- `login()` [L75-L80](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L75-L80)

---

#### 4.7.2 第二层：上下文授权 (Context Authorization) 例外

本层职责：读取 `@AuthorizeContext` 装饰器元数据，提取参数后进入 L3 校验。

| 例外类型 | 触发条件 | 代码位置 | 结果 | 安全影响 |
|---------|---------|---------|------|---------|
| 无装饰器放行 | Resolver 方法通过了 L1（有 `@UseGuards(GqlAuthGuard)`），但**未标注** `@AuthorizeContext` 装饰器 | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L67-L71) | **L3 归属与权限校验完全跳过**，直接返回 `true` | ⚠️ 共 16 个方法直接放行（见下方清单） |

具体无装饰器放行方法清单（16 个）：

**WorkspaceResolver**（类级有 `@UseGuards(GqlAuthGuard)` [L46](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L46)，9 个方法无装饰器）：
- `currentWorkspace()` [L76-L92](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L76-L92)
- `projects` (ResolveField) [L94-L99](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L94-L99)
- `createWorkspace()` [L115-L128](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L115-L128)
- `workspaceMembers()` [L188-L197](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L188-L197)
- `workspaceUsers()` [L199-L206](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L199-L206)
- `subscription` (ResolveField) [L208-L211](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L208-L211)
- `gitOrganizations` (ResolveField) [L213-L217](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L213-L217)
- `provisionSubscription()` [L220-L231](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L220-L231)
- `redeemCoupon()` [L233-L240](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L233-L240)

**AuthResolver**（类级无守卫，方法级有 `@UseGuards(GqlAuthGuard)` 但无 `@AuthorizeContext`，7 个方法）：
- `me()` [L34-L38](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L34-L38)
- `permissions()` [L40-L44](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L40-L44)
- `resourcePermissions()` [L46-L57](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L46-L57)
- `userApiTokens()` [L97-L101](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L97-L101)
- `changePassword()` [L103-L114](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L103-L114)
- `setCurrentWorkspace()` [L126-L140](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L126-L140)
- `completeInvitation()` [L142-L153](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts#L142-L153)

---

#### 4.7.3 第三层：工作区归属校验 (Workspace Ownership) 例外

本层职责：校验目标资源是否归属于当前用户的 active workspace。

| 例外类型 | 触发条件 | 代码位置 | 结果 | 安全影响 |
|---------|---------|---------|------|---------|
| 前置检查拒绝 | `originId` 为空字符串且 `originType !== AuthorizableOriginParameter.None` | [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L26-L28) | 直接返回 `false` 拒绝访问 | 保护性安全加固（防止参数缺失绕过校验） |
| None 类型绕过 | 装饰器使用 `AuthorizableOriginParameter.None`（如创建操作无目标资源） | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L39-L41) | **不查数据库**，直接返回 `{ canAccessWorkspace: true }` | 符合预期（创建类操作无需校验已有资源归属） |
| GitRepositoryId 归属缺失 | `originType === AuthorizableOriginParameter.GitRepositoryId` | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L66-L77) | 仅 `count({ where: { id } })`，**查询条件未限定 workspaceId** | ⚠️ 潜在跨工作区访问漏洞（任意 workspace 用户只要知道 ID 即可通过归属校验） |
| ApiTokenId 用户维度校验 | `originType === AuthorizableOriginParameter.ApiTokenId` | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L198-L211) | 按 `userId === user.id` 校验，**不校验 workspace 归属** | 符合预期（API Token 是用户私有资源，不绑定 workspace） |

---

#### 4.7.4 第三层附属：权限字符串校验 (Permission Check) 例外

本层职责：校验用户是否拥有所需的权限字符串（在归属校验通过后执行）。

| 例外类型 | 触发条件 | 代码位置 | 结果 | 安全影响 |
|---------|---------|---------|------|---------|
| 无权限要求 | 装饰器未传入 `permissions` 参数，导致 `requiredPermissions` 为空数组或 `undefined` | [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L55-L58) | 权限校验自动通过，返回 `true` | 符合预期（仅需校验资源归属、无需额外权限的只读操作） |
| Admin 全权限 | 用户 JWT Token 的 `permissions[]` 中包含 `"*"` | [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L60-L63) | 权限校验自动通过，返回 `true` | 符合预期（超级管理员拥有所有权限） |

---

#### 4.7.5 三层例外一览表（便于快速复核）

| 层级 | 例外类型 | 数量 | 安全等级 |
|------|---------|------|---------|
| **L1 登录认证** | 匿名端点 | 3 个 | 绿（预期） |
| **L2 上下文授权** | 无 `@AuthorizeContext` 装饰器放行 | 16 个 | 橙（需审计） |
| **L3a 工作区归属** | 前置检查拒绝 | 逻辑分支 | 绿（保护） |
| **L3a 工作区归属** | `None` 类型绕过（创建操作） | 设计 | 绿（预期） |
| **L3a 工作区归属** | `GitRepositoryId` 不限定 workspace | 1 类 | 红（漏洞） |
| **L3a 工作区归属** | `ApiTokenId` 按用户维度校验 | 1 类 | 绿（预期） |
| **L3b 权限校验** | `requiredPermissions` 为空 | 设计 | 绿（预期） |
| **L3b 权限校验** | Admin `"*"` 全权限 | 设计 | 绿（预期） |

### 4.8 权限类型定义

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
   │  L2 无装饰器 → 直接放行
   ▼
5. 前端跳转 /{workspaceId} → WorkspaceLayout 挂载
   │
   ├─── useWorkspaceSelector() → workspacesList、currentWorkspace
   ├─── useProjectSelector() → currentProject、projectsList
   ├─── useResources() → currentResource、resources
   ├─── usePermissions() → GET_PERMISSIONS 查询
   │       │  L2 无装饰器 → 直接放行
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
9. 根据 permissions.canPerformTask() 控制操作按钮可见性
   │  ⚠️ Settings Tab 本身不做权限过滤
```

### 5.2 访问受保护资源的完整三层授权流程

```
用户点击导航链接 /{workspaceId}/settings/roles/{roleId}
        │
        ▼
前端路由渲染 Role.tsx，发起 GraphQL 查询（role(where: { id: roleId })）
        │
        ▼
后端 GqlAuthGuard.canActivate()
        │
        ├─── L1 登录认证 ([L31-L35](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L31-L35))
        │    super.canActivate() → JWT 有效?
        │    └─── NO → 拒绝 ❌
        │    └─── YES → 进入 L2
        │
        └─── L2 上下文授权 ([L61-L83](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L61-L83))
                │
                ├─── 读取 @AuthorizeContext(AuthorizableOriginParameter.RoleId, "where.id")
                │    └─── 无装饰器? → 放行 ✅（见 4.3 节 16+ 个例外方法）
                │
                ├─── get(requestArgs, "where.id") → 提取 roleId
                │
                ▼
        L3a 工作区归属校验 ([validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L129-L144))
                │
                ├─── 前置检查: originId 非空 → 通过
                ├─── VALIDATION_FUNCTIONS[RoleId](prisma, roleId, workspace.id)
                │    prisma.role.count({ where: { id: roleId, deletedAt: null, workspace: { id: workspace.id }}})
                │    ├─── count === 0 → 不属于该工作区 → 拒绝 ❌
                │    └─── count === 1 → canAccessWorkspace = true ✅
                │
                ▼
        L3b 权限字符串校验 ([permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L49-L87))
                │
                ├─── requiredPermissions 为空 → 放行 ✅
                │
                ▼
        Resolver 方法执行，返回数据
```

### 5.3 创建操作的权限检查流程

```
用户点击 "New Role" 按钮 → L3a: permissions.canPerformTask("role.create")
        │  前端按钮可见性检查（UX 优化，非安全边界）
        ▼
用户填写表单提交 → createRole(data: {...}) mutation
        │
        ▼
L1 登录认证 → 通过 ✅
        │
        ▼
L2 上下文授权 → 读取装饰器: @AuthorizeContext(AuthorizableOriginParameter.None, "", "role.create")
        │
        ▼
L3a 工作区归属校验
        │
        ├─── originType === None → [L39-L41](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L39-L41)
        │    return { canAccessWorkspace: true }  ← 不查数据库，直接放行
        │
        ▼
L3b 权限字符串校验 ([L49-L87](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts#L49-L87))
        │
        ├─── user.permissions.includes("*") → Admin ✅
        │
        └─── user.permissions.includes("role.create") → 有该权限 ✅
                │
                └─── 两者都无 → 拒绝 ❌（403 Forbidden）
```

---

## 6. 关键设计洞察

### 6.1 前端可见性 ≠ 后端安全，且两端都存在盲区

- **前端**：`canPerformTask()` 仅控制 UI 按钮可见性，是 UX 优化手段。Settings 的 Tab 导航、RolesPage、TeamsPage 等页面级容器组件**均不做权限过滤**，所有已登录用户均可访问
- **后端**：三层授权架构中，**L2（上下文授权层）是最大的安全盲区**——共 16 个 Resolver 方法（WorkspaceResolver 9 个 + AuthResolver 7 个）因未加 `@AuthorizeContext` 装饰器而完全跳过 L3 校验（详见 4.3 节和 4.7 节详细清单）
- 即使前端绕过 UI 限制直接发起 GraphQL 请求，对于**添加了装饰器**的方法，后端 L3 归属与权限校验仍会拒绝未授权操作

### 6.2 工作区作为授权锚点（按三层分类的例外清单）

- **原则**：所有资源（Project、Resource、Entity、Role、Team 等）都必须归属于某个 Workspace，授权的第一步是验证目标资源是否属于当前用户的 active workspace
- **实现机制**：L3a 通过 `AuthorizableOriginParameter`（26 种）+ `VALIDATION_FUNCTIONS` 实现
- **完整例外清单**（按三层分类，详见 4.7 节详细说明）：

**第一层：登录认证 (Authentication) 例外**
| 例外类型 | 数量 | 安全影响 |
|---------|------|---------|
| 匿名端点（无 `@UseGuards(GqlAuthGuard)`） | 3 个 | 绿：符合预期（signup/login） |

**第二层：上下文授权 (Context Authorization) 例外**
| 例外类型 | 数量 | 安全影响 |
|---------|------|---------|
| 无 `@AuthorizeContext` 装饰器方法 | 16 个（WorkspaceResolver 9 + AuthResolver 7） | 橙：需审计，完全跳过 L3 校验 |

**第三层：工作区归属与权限校验例外**
| 子层 | 例外类型 | 数量 | 安全影响 |
|-----|---------|------|---------|
| L3a 归属 | 前置检查拒绝（空 originId） | 逻辑分支 | 绿：保护性加固 |
| L3a 归属 | `AuthorizableOriginParameter.None` | 设计 | 绿：符合预期（创建操作） |
| L3a 归属 | `GitRepositoryId` 不限定 workspace | 1 类 | 红：潜在跨工作区访问漏洞 |
| L3a 归属 | `ApiTokenId` 按用户维度校验 | 1 类 | 绿：符合预期（用户私有 Token） |
| L3b 权限 | `requiredPermissions` 为空 | 设计 | 绿：符合预期（仅归属校验） |
| L3b 权限 | Admin `"*"` 全权限 | 设计 | 绿：符合预期（超级管理员） |

### 6.3 三级权限模型

| 层级 | 存储位置 | 适用场景 |
|------|---------|---------|
| L1 工作区级 | JWT Token (`user.permissions`) | 全局操作：邀请成员、创建团队、管理角色 |
| L2 项目级 | `teamAssignment` (项目配置资源) | 项目设置、项目级权限分配 |
| L3 资源级 | `teamAssignment` (具体资源) | 单个服务/消息代理的编辑权限 |

### 6.4 权限声明式编程

后端使用装饰器 `@AuthorizeContext(OriginType, paramPath, permissions)` 声明式定义授权规则，与业务逻辑解耦：

- **优点**：Resolver 代码纯净、授权逻辑统一、易审计
- **不足**：需要确保每个需要保护的 Resolver 方法都正确添加了装饰器——目前共有 16 个方法（WorkspaceResolver 9 个 + AuthResolver 7 个）未添加装饰器而直接放行，这是授权的主要盲区
- **风险点**：新增 Resolver 方法时遗漏装饰器会导致该方法完全不受授权保护，且无任何编译时或运行时告警

---

## 7. 核心文件索引（含精确行号便于外部复核）

| 模块 | 文件 | 职责 | 关键行号 |
|------|------|------|---------|
| **前端导航** | [WorkspaceLayout.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx) | 根布局、上下文注入 | L111-L242 |
| | [WorkspaceHeader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceHeader.tsx) | 顶部导航栏 | L42-L218 |
| | [WorkspaceNavigation.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceNavigation.tsx) | 面包屑导航 | L29-L98 |
| | [WorkspaceSettingsPage.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/WorkspaceSettingsPage.tsx) | 设置页侧边导航 | L19-L60 |
| | [useTabRoutes.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Layout/useTabRoutes.ts) | 路由→TabItem 纯映射（无权限过滤） | L9-L42 |
| | [appRoutes.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/routes/appRoutes.tsx) | 路由配置（permission 仅控制登录） | L5-L20 |
| | [routesUtil.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/routes/routesUtil.tsx) | `RouteDef.permission` 登录认证门控逻辑 | L69-L84 |
| **前端权限** | [usePermissions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/Workspaces/hooks/usePermissions.ts) | 权限 Hook（canPerformTask） | L12-L44 |
| | [appContext.tsx](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-client/src/context/appContext.tsx) | 全局上下文定义 | L12-L141 |
| **L1 登录认证** | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) | L1 Passport JWT 认证 | L31-L35 |
| **L2 上下文授权** | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) | L2 authorizeContext（含无装饰器放行分支） | L53-L83 |
| | [authorizeContext.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/decorators/authorizeContext.decorator.ts) | 授权装饰器定义 | L21-L37 |
| **L3a 归属校验** | [AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts) | 26 种可授权参数枚举（None=0 到 UserId=25） | L7-L34 |
| | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts) | None 直接放行（L39-L41）、GitRepositoryId 漏洞（L66-L77）、ApiTokenId 用户维度（L198-L211） | L30-L508 |
| **L3b 权限校验** | [permissions.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/permissions/permissions.service.ts) | validateAccess 前置检查（L26-L28）、validatePermissions Admin/空权限放行（L55-L63） | L18-L210 |
| **例外 Resolver** | [workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts) | 9 个无 @AuthorizeContext 的方法（类级 L1 守卫 L46） | L46-L241 |
| | [auth.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/auth.resolver.ts) | 3 个匿名端点（L59-L80）、7 个无 @AuthorizeContext 方法 | L34-L153 |
| | [auth/types.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/packages/amplication-server/src/core/auth/types.ts) | AuthUser 类型（含 workspace + permissions[]） | - |
| **共享类型** | [roles-permissions.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/48-amplication/libs/util/roles-types/src/lib/roles-permissions.types.ts) | 权限字符串定义 | L1-L98 |
