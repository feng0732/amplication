# Service Catalog 项目组织逻辑解析

本文档对照代码，清晰说明 **服务列表**、**工作区视图**、**状态摘要** 三部分的来源、数据链路和组件复用关系。

---

## 0. 核心实体层级

在深入之前先理清领域模型的层级关系，这是理解所有视图的基础：

```
Workspace（工作区）
  └── Project（项目）
        └── Resource（资源 = Service / MessageBroker / PluginRepository / ...）
              ├── Build（构建记录）
              ├── Entity（数据实体）
              ├── Relation（资源间关系）
              ├── Owner（User 或 Team）
              └── Blueprint（蓝图）
```

---

## 1. 服务列表（Catalog / Service List）

### 1.1 页面入口与复用关系

服务列表在 UI 上有 **三个入口**，但底层全部复用同一套数据组件：

| 页面 | 路由场景 | 入口组件 | 复用内核 |
|---|---|---|---|
| 工作区 Catalog | `/:workspace/catalog` | [Catalog.tsx](packages/amplication-client/src/Catalog/Catalog.tsx#L12-L46) | `CatalogGrid` |
| 项目资源列表 | `/:workspace/:project` | [ResourceList.tsx](packages/amplication-client/src/Workspaces/ResourceList.tsx#L11-L31) | `CatalogGrid` + `fixedFilters` 限定 `projectId` |
| 工作区图视图 | `/:workspace/graph` | [WorkspaceGraph.tsx](packages/amplication-client/src/Workspaces/WorkspaceGraph.tsx#L9-L19) | `CatalogGraph` |

核心复用链：

```
┌────────────────────────────────────────────────────────────────┐
│  表格视图（共享 Context 实例）                                   │
│  CatalogGrid (工作区 Catalog / 项目 ResourceList)               │
│      └── useCatalogContext()   ← CatalogContextProvider        │
│            └── useCatalog()    (Context 内部的共享实例)          │
│                 └── SEARCH_CATALOG GraphQL (分页模式)           │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  图视图（独立实例，不经过 Context）                               │
│  CatalogGraph (WorkspaceGraph)                                 │
│      └── useCatalogGraph({ initialPageSize: 1000 })            │
│            └── useCatalog({ initialPageSize: 1000 })           │
│                 └── SEARCH_CATALOG GraphQL (一次拉 1000 条)     │
│                      └── resourcesToNodesAndEdges() → nodes/edges
└────────────────────────────────────────────────────────────────┘
```

**关键差异**：表格视图和图视图各自拥有独立的 `useCatalog` 实例，数据状态和筛选条件互不共享（详见 4.5 节）。

### 1.2 前端数据流

#### (1) CatalogContextProvider —— 仅服务于表格视图

Catalog 的筛选、分页、搜索、数据状态都挂在 [CatalogContext.tsx](packages/amplication-client/src/Catalog/CatalogContext.tsx#L45-L77) 下。

它在 [WorkspaceLayout.tsx](packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx#L245-L297) 第 245 行被包裹在整个工作区布局内部，但**只被表格视图消费**：

| 消费者 | 代码位置 | 用法 |
|---|---|---|
| CatalogGrid | `CatalogGrid.tsx` L59-L67 | `useCatalogContext()` 获取共享状态 |
| CatalogGraph（图视图） | `useCatalogGraph.tsx` L42-L45 | **不使用 Context**，直接调用 `useCatalog({ initialPageSize: 1000 })` |

Context 实例的行为：
- 同一个 Workspace 下，切换 `工作区 Catalog` ↔ `项目 ResourceList`，**共享同一个表格实例**，筛选和分页状态会保留
- 图视图有自己独立的 `useCatalog` 实例，完全不受 Context 影响（详见 4.5 节）

#### (2) useCatalog Hook —— 业务逻辑核心

[hooks/useCatalog.ts](packages/amplication-client/src/Catalog/hooks/useCatalog.ts#L27-L194) 做了以下关键事情：

1. **分页**：通过 `useQueryPagination` 管理 `pageNumber/pageSize/totalCount`
2. **默认过滤**：`DEFAULT_PROJECT_TYPE_FILTER` 排除三类资源：
   - `ProjectConfiguration`（项目配置，不直接展示给用户）
   - `PluginRepository`（插件仓库）
   - `ServiceTemplate`（服务模板）
3. **搜索**：`searchPhrase` → `name { contains, mode: Insensitive }`
4. **过滤拆分**：`setFilter` 函数将筛选条件 **拆成两条管线**：
   - **内置字段**（resourceType / ownership / projectIdFilter 等）→ 拼成 `ResourceWhereInput`
   - **自定义属性**（CustomProperty）→ 拼成 `JsonPathStringFilter`，走 `properties` JSON 字段的 path 查询
5. **关联查询**：GraphQL query 一次返回资源及其 blueprint / project / owner / version / builds / gitRepository 等关联字段

#### (3) SEARCH_CATALOG GraphQL Query

定义在 [catalogQueries.ts](packages/amplication-client/src/Catalog/queries/catalogQueries.ts#L3-L94)，字段清单：
```
catalog(where, orderBy, take, skip) {
  totalCount
  data {
    id, name, description, resourceType, codeGenerator, properties
    blueprint { id, name, color }
    project { id, name }
    owner { ... on User / ... on Team }
    version { ... }
    builds(take: 1, orderBy: createdAt Desc) { status, version, ... }
    gitRepository { ... }
    serviceTemplate { ... }
    relations { ... }
  }
}
```

#### (4) CatalogDataColumns —— 列定义

[CatalogDataColumns.tsx](packages/amplication-client/src/Catalog/CatalogDataColumns.tsx#L27-L251) 定义了表格列：
- 固定列：Type / Name / Project / Blueprint / Owner / Code Generator / Git Org / Git Repo / Description / Pending Changes / Last Build / Code Gen Version / Template / Template Version
- 动态列：`columnsWithProperties()` 把工作区定义的 **CustomProperties** 动态追加为列

### 1.3 后端数据链路

GraphQL 请求到达 [resource.resolver.ts](packages/amplication-server/src/core/resource/resource.resolver.ts#L85-L96) 的 `catalog` Query：

```typescript
@Query(() => PaginatedResourceQueryResult)
@InjectContextValue(InjectableOriginParameter.WorkspaceId, "where.project.workspace.id")
async catalog(@Args() args: FindManyResourceArgs): Promise<PaginatedResourceQueryResult> {
  return this.resourceService.searchResourcesWithCount(args);
}
```

关键注解 `@InjectContextValue` 自动把当前登录用户的 WorkspaceId 注入到 `where.project.workspace.id`，**从权限层保证不会查到其他工作区的数据**。

Service 层 [resource.service.ts](packages/amplication-server/src/core/resource/resource.service.ts#L1320-L1383) 分两步：

#### (1) prepareResourceFindManyArgsForQuery —— 参数归一化

```
输入: FindManyResourceArgs
  │
  ├── 处理 serviceTemplateId → 通过 ResourceTemplateVersion 反查 resourceIds
  ├── 处理 projectIdFilter → 合并到 projectId
  ├── 处理 properties (JsonPathStringFilter) → 转成 Prisma JSON path 查询
  └── 强制条件: deletedAt = null AND archived != true
  │
  ▼
输出: Prisma.ResourceFindManyArgs
```

#### (2) searchResourcesWithCount —— 并行查询

```typescript
const [count, resources] = await Promise.all([
  this.prisma.resource.count({ where }),   // 总数
  this.prisma.resource.findMany(preparedArgs),  // 分页数据
]);
return { totalCount: count, data: resources };
```

#### (3) ResolveField —— 关联字段懒加载

GraphQL 返回的关联字段（project / owner / blueprint / builds / gitRepository 等）并不是 Prisma 一次查出来的，而是通过 `@ResolveField` **按字段独立解析**：
- `project` → [resource.resolver.ts L250-L255](packages/amplication-server/src/core/resource/resource.resolver.ts#L250-L255)
- `owner` → [L345-L357](packages/amplication-server/src/core/resource/resource.resolver.ts#L345-L357)
- `builds` → [L119-L128](packages/amplication-server/src/core/resource/resource.resolver.ts#L119-L128)
- `blueprint` → [L290-L301](packages/amplication-server/src/core/resource/resource.resolver.ts#L290-L301)

---

## 2. 工作区视图（Workspace View）

### 2.1 工作区页面层级

工作区的顶层布局是 [WorkspaceLayout.tsx](packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx#L55-L303)，它在一个组件里 **聚合了所有 hooks**，然后通过 `AppContextProvider` + `CatalogContextProvider` 向下分发：

```
WorkspaceLayout
├── useWorkspaceSelector   → currentWorkspace, workspacesList, subscription...
├── useProjectSelector     → currentProject, projectsList...
├── useResources           → resources[], projectConfigurationResource...
├── usePendingChanges      → pendingChanges[], commitRunning...
├── useCommits             → lastCommit...
├── useBlueprintsMap       → blueprintsMap
├── useCustomPropertiesMap → customPropertiesMap
└── usePermissions
```

这些 hooks 全部在 WorkspaceLayout 中调用一次，结果注入 AppContext，子组件通过 `useAppContext()` 直接取用。

### 2.2 WorkspaceOverview —— 工作区概览页

[WorkspaceOverview.tsx](packages/amplication-client/src/Workspaces/WorkspaceOverview.tsx#L48-L133) 展示内容：

| 区块 | 数据来源 |
|---|---|
| 工作区名称 + 订阅 Plan Chip | `currentWorkspace`（来自 AppContext） |
| Members 数量 | `GET_WORKSPACE_MEMBERS` Query，过滤 `type === User` |
| Project 卡片列表 | `projectsList`（来自 AppContext）→ `<ProjectList>` |

### 2.3 ProjectList —— 项目卡片网格

[ProjectListItem.tsx](packages/amplication-client/src/Project/ProjectListItem.tsx#L28-L114) 的要点：

1. **服务数量统计**：从 `project.resources` 中过滤出 `Service` + `MessageBroker` 类型，显示在卡片右上角
2. **两个入口**：
   - Platform 按钮 → `/:workspace/platform/:project`
   - Catalog 按钮 → `/:workspace/:project`（即 ResourceList，复用 CatalogGrid）

### 2.4 projectsList 的数据来源

`projectsList` 由 [useProjectSelector.ts](packages/amplication-client/src/Workspaces/hooks/useProjectSelector.ts#L37-L52) 发起，GraphQL 是 [projectQueries.ts](packages/amplication-client/src/Workspaces/queries/projectQueries.ts#L3-L34) 中的 `GET_PROJECTS`：

```graphql
query findProjects {
  projects {
    id, name, description, licensed
    resources {              # ← 内嵌 resources，供卡片计算"服务数量"
      id, name, resourceType, licensed, gitRepository { ... }
    }
    createdAt
  }
}
```

注意：这里只拉取了 resources 的**极简字段**（不含 builds / owner 等），因为概览页只需要用来计数和判断 Git 连接状态。进入具体项目后，才会通过 `useResources` / `useCatalog` 拉取完整字段。

### 2.5 WorkspaceGraph —— 工作区关系图（独立 useCatalog 实例）

[WorkspaceGraph.tsx](packages/amplication-client/src/Workspaces/WorkspaceGraph.tsx#L9-L19) 本身是个空壳，直接渲染 `<CatalogGraph />`。

[CatalogGraph.tsx](packages/amplication-client/src/Catalog/CatalogGraph/CatalogGraph.tsx#L51-L246) 的数据链路——**注意：它完全绕过 CatalogContextProvider**：

```
CatalogGraph.tsx L73
  └── useCatalogGraph({ onMessage })          (CatalogGraph/hooks/useCatalogGraph.tsx)
        │
        ├── 独立实例化：useCatalog({ initialPageSize: 1000 })
        │     └── 不调用 useCatalogContext()，和表格视图无任何状态共享
        │     └── SEARCH_CATALOG GraphQL：一次拉 1000 条
        │           └── 用于前端一次性生成所有节点和边
        │
        ├── 自己管理状态：groupByFields / layoutOptions / nodes / edges
        │     └── 持久化到 localStorage：`catalogGraphLayout-${workspaceId}`
        │
        └── resourcesToNodesAndEdges()
              └── 按 groupByFields (project / blueprint / owner) 生成分组节点
```

**为什么图视图不共享 Context？**
- 图视图需要一次性加载大量资源（1000 条）来生成完整拓扑
- 表格视图是分页加载（默认 20 条/页）
- 两者的筛选条件也各自持久化：
  - 表格 → `fixedFiltersKey` 决定（`workspace-catalog` 或项目 ID）
  - 图视图 → 固定为 `"catalog-graph"`（CatalogGraph.tsx L178）
- 图视图还需要管理图特有的状态：布局参数、分组方式、节点位置等，不需要也不应该影响表格

---

## 3. 状态摘要（Status Summary）

状态摘要散布在多个层级：**工作区级**、**项目级**、**资源级**。以下梳理每个指标的来龙去脉。

### 3.1 资源级状态（Catalog 列表列）

在 Catalog 表格中，每行资源展示以下状态：

| 指标 | 组件文件 | 数据来源 |
|---|---|---|
| Last Build（最近构建） | [ResourceLastBuild.tsx](packages/amplication-client/src/Workspaces/ResourceLastBuild.tsx#L21-L63) | `resource.builds[0]`，来自 SEARCH_CATALOG 中 `builds(take:1, orderBy: createdAt Desc)` |
| Code Gen Version | `ResourceLastBuildVersion`（在 CatalogDataColumns 中引用） | `resource.builds[0].codeGeneratorVersion` |
| Pending Changes（待提交变更数） | [ResourcePendingChangesCount.tsx](packages/amplication-client/src/Workspaces/ResourcePendingChangesCount.tsx#L14-L31) | 从 AppContext 的 `pendingChanges[]` 按 `resource.id` 过滤计数 |
| Git Repo / Git Org | `ResourceGitRepo` / `ResourceGitOrg` | `resource.gitRepository` 字段 |
| Template + Template Version | `ServiceTemplateChip` / `VersionTag` | `resource.serviceTemplate` + `resource.serviceTemplateVersion` |

**ResourceLastBuild 的颜色逻辑**：通过 `BUILD_STATUS_TO_COLOR` 把构建状态（Success / Failed / Running 等）映射成 `UserAndTime` 组件的文字颜色，同时左侧加一个 `CommitBuildsStatusIcon` 图标。

### 3.2 Pending Changes 的独立数据链路

Pending Changes **不是 SEARCH_CATALOG 返回的**，而是单独一条链路：

1. WorkspaceLayout 中调用 `usePendingChanges(currentProject, resourceTypeGroup)`
2. GraphQL: [GET_PENDING_CHANGES_STATUS](packages/amplication-client/src/Workspaces/queries/projectQueries.ts#L45-L102)
3. 返回 `pendingChanges[]`，每条包含：
   - `origin`（Entity 或 Block）
   - `resource`（变更所属的资源）
   - `action`（Create / Update / Delete）
4. 注入 AppContext
5. `ResourcePendingChangesCount` 按 `resource.id` 在前端内存中过滤计数

设计意图：**Pending Changes 是会话级、暂态的数据**（未 commit 的变更），不适合跟 catalog 的持久化资源数据放在同一 resolver 里。

### 3.3 资源概览摘要卡片（Resource Overview）

当用户进入某个具体 Service 资源时，[ResourceOverview.tsx](packages/amplication-client/src/Resource/ResourceOverview/ResourceOverview.tsx#L49-L222) 在顶部面板右侧展示 4 个摘要数字：**Entities / APIs / Installed Plugins / Roles**。

这 4 个数字以及插件分类数据都由同一个 hook —— [useResourceSummary.tsx](packages/amplication-client/src/Resource/hooks/useResourceSummary.tsx#L40-L180) 统一聚合。

```
useResourceSummary(currentResource)
├── 来源 1: currentResource.entities      → summaryData.models (实体数)
├── 来源 2: useModuleAction()             → summaryData.apis   (API 数)
├── 来源 3: usePlugins()                  → summaryData.installedPlugins + usedCategories
├── 来源 4: GET_ROLES Query               → summaryData.roles  (角色数)
└── 来源 5: GET_CATEGORIES Query          → availableCategories
```

#### (1) Entities 数量

```typescript
// useResourceSummary.tsx L127
const models = currentResource?.entities?.length || 0;
```

直接取自 `currentResource.entities` 数组长度。`currentResource.entities` 由 [useResources.ts](packages/amplication-client/src/Workspaces/hooks/useResources.ts#L117-L129) 中的 `GET_RESOURCES` Query 拉取，GraphQL 字段为：
```graphql
entities { id, name }
```

#### (2) APIs 数量

```typescript
// useResourceSummary.tsx L128, L163-L172
useEffect(() => {
  findModuleActions({
    variables: { where: { resource: { id: currentResource.id } } },
    fetchPolicy: "cache-and-network",
  });
}, [currentResource, findModuleActions]);

// L128
const modules = findModuleActionsData?.moduleActions?.length || 0;
```

数据来源于 `useModuleAction()` hook（[useModuleAction.tsx](packages/amplication-client/src/ModuleActions/hooks/useModuleAction.tsx#L33-L100)）中的 `FIND_MODULE_ACTIONS` GraphQL 查询，本质是统计该资源下定义的 `ModuleAction`（API 端点动作）总数。注意：UI 上显示为 "APIs"，但代码变量名是 `modules`，映射关系在 [ResourceOverview.tsx L71-L76](packages/amplication-client/src/Resource/ResourceOverview/ResourceOverview.tsx#L71-L76) 中硬编码：
```typescript
{ icon: "api", title: "APIs", link: `${baseUrl}/modules`, value: summaryData.apis }
```

#### (3) Installed Plugins 数量 + 插件分类

这部分由两个数据源组合而成：

**① 已安装插件列表**：来自 [usePlugins.ts](packages/amplication-client/src/Plugins/hooks/usePlugins.ts#L68-L150) hook，底层调用 `GET_PLUGIN_INSTALLATIONS` Query，返回当前资源的 `PluginInstallation[]`。
```typescript
// useResourceSummary.tsx L129
const installedPlugins = pluginInstallations?.length || 0;
```

**② 插件分类（Categories）**：来自 [categoriesQueries.ts](packages/amplication-client/src/Resource/hooks/categoriesQueries.ts#L1-L13) 的 `GET_CATEGORIES` Query，**注意这个请求走的是独立的 Apollo client**：
```typescript
// useResourceSummary.tsx L61-L69
useQuery(GET_CATEGORIES, {
  context: { clientName: "pluginApiHttpLink" },  // ← 指向 amplication-plugin-api 服务
  variables: {},
  skip: !currentResource.id,
});
```

分类数据由 `amplication-plugin-api` 服务提供，而非主 amplication-server。

**③ 已用分类 vs 可用分类**（供 [PluginsTile.tsx](packages/amplication-client/src/Resource/PluginsTile.tsx#L43-L164) 展示）：

在 `useResourceSummary.tsx` L83-L124 中，代码按 rank 排序 categories，然后：
- **usedCategories**：遍历已安装插件的 `plugin.categories`，按分类名分组，得到 `{ categoryName: { category, installedPlugin[] } }`；**过滤掉无 rank 的分类**
- **availableCategories**：sortedCategories 中排除 usedCategories 里已有的，同时排除 rank 为 null 的

PluginsTile 组件将两类数据各取前 4 个显示：
- Installed Plugins 区：每个分类显示图标 + 名称 + 该分类下的插件 logo 组（[PluginLogoGroup](packages/amplication-client/src/Plugins/PluginLogoGroup.tsx)）
- Available Plugins 区：每个分类显示图标 + 名称 + 描述 + "Try out" 链接

分类图标和描述全部来自 `GET_CATEGORIES` 返回的 PluginCategory 对象。

#### (4) Roles 数量

```typescript
// useResourceSummary.tsx L73-L81, L130
const { data: rolesData } = useQuery<TData>(GET_ROLES, {
  variables: { id: currentResource.id, orderBy: { createdAt: Asc } },
  skip: !currentResource.id,
});

const roles = rolesData?.resourceRoles?.length || 0;
```

GraphQL 查询 `GET_ROLES` 定义在 [RoleList.tsx](packages/amplication-client/src/ResourceRoles/RoleList.tsx#L127-L143)，字段为 `resourceRoles(where: { resource: { id: $id } })`，统计该资源下定义的角色总数。

#### (5) 渲染：ResourceOverview 的用法摘要区

在 [ResourceOverview.tsx L63-L90](packages/amplication-client/src/Resource/ResourceOverview/ResourceOverview.tsx#L63-L90) 中：
```typescript
const resourceUsageData = [
  { icon: "database", title: "Entities",          link: `${baseUrl}/entities`,  value: summaryData.models },
  { icon: "api",      title: "APIs",              link: `${baseUrl}/modules`,   value: summaryData.apis },
  { icon: "plugin",   title: "Installed Plugins", link: `${baseUrl}/plugins/installed`, value: summaryData.installedPlugins },
  { icon: "roles_outline", title: "Roles",        link: `${baseUrl}/roles`,     value: summaryData.roles },
];
```
每项渲染为一个带图标 + 标题 + 数值的可点击链接，指向对应功能页。

### 3.4 项目级状态摘要

在 WorkspaceOverview 的 Project 卡片（[ProjectListItem.tsx](packages/amplication-client/src/Project/ProjectListItem.tsx#L29-L37)）中：

```typescript
const services = useMemo(
  () => project.resources.filter(
    x => x.resourceType === Service || x.resourceType === MessageBroker
  ),
  [project.resources]
);
```

显示为 `"3 Services"` 这样的 Tag。这个数字的数据源是 `GET_PROJECTS` Query 中 `project.resources` 内嵌的轻量列表。

### 3.5 工作区级状态摘要

WorkspaceOverview 头部面板展示：

| 指标 | 来源 |
|---|---|
| Workspace 名称 + 颜色徽章 | `currentWorkspace.name` + 订阅 Plan 决定颜色 ([WorkspaceSelector.tsx](packages/amplication-client/src/Workspaces/WorkspaceSelector.tsx) 中的 `getWorkspaceColor`) |
| 订阅 Plan Chip | `currentWorkspace.subscription.subscriptionPlan` → 映射到 `SUBSCRIPTION_TO_CHIP_STYLE` |
| 成员数量 | `GET_WORKSPACE_MEMBERS` → 过滤 `type === User` 计数 |
| 项目列表计数标签 | [ProjectList.tsx L24-L28](packages/amplication-client/src/Project/ProjectList.tsx#L24-L28) 中 `projects.length` → 显示为 `"3 Projects"` 这样的 Tag |

**`projectsList.length` 的实际用途**（不是作为"工作区项目总数"的状态摘要展示，而是以下两处）：

| 使用位置 | 代码 | 用途 |
|---|---|---|
| AddNewProject 按钮配额检查 | [WorkspaceOverview.tsx L73](packages/amplication-client/src/Workspaces/WorkspaceOverview.tsx#L73) → [AddNewProject.tsx L45-L51](packages/amplication-client/src/Project/AddNewProject.tsx#L45-L51) | 作为 `FeatureIndicatorContainer` 的 `actualUsage` 参数，与计费配额 `BillingFeature.Projects` 比对，达到上限时禁用 "Add New Project" 按钮并提示升级 |
| ProjectList 计数标签 | [ProjectList.tsx L24-L28](packages/amplication-client/src/Project/ProjectList.tsx#L24-L28) | 在项目网格上方显示 `"X Project(s)"` 的文本标签 |
| useProjectSelector 路由逻辑 | [useProjectSelector.tsx L103](packages/amplication-client/src/Workspaces/hooks/useProjectSelector.ts#L103), [L136-L137](packages/amplication-client/src/Workspaces/hooks/useProjectSelector.ts#L136-L137) | 判断项目列表是否为空，决定是否触发欢迎页/购买页跳转，以及判断当前选中 project 是否在列表中 |

### 3.6 Workspace Footer 的最近提交信息

Workspace Footer 显示的最近 Commit ID 和 Build ID **不是全局提交**，而是严格遵循当前项目 + Services 资源组的查询条件。

调用链和过滤条件：

```
WorkspaceLayout (L109)
  └── useCommits(currentProject?.id)              # 参数限定为当前项目 ID
        ├── GET_COMMITS Query (useCommits.ts L240-L251)
        │     ├── projectId: currentProjectId     # ← 条件 1：仅当前项目
        │     ├── resourceTypeGroup: Services     # ← 条件 2：仅 Services 资源组（不含 Platform）
        │     ├── orderBy: createdAt Desc
        │     └── take: 20
        └── GET_LAST_COMMIT Query (L95-L99)
              └── projectId: currentProjectId     # 轮询同样限项目
          │
          ▼
  └── <WorkspaceFooter lastCommit={commitUtils.lastCommit} />
        └── WorkspaceFooter.tsx L39-L46
              lastCommit.builds.find(
                b => b.resourceId === currentResource.id  # ← 条件 3：仅当前选中资源的 build
              )
```

关键代码位置：
- Hook 调用：`packages/amplication-client/src/Workspaces/WorkspaceLayout.tsx` L109、L286
- 查询条件：`packages/amplication-client/src/VersionControl/hooks/useCommits.ts` L240-L251（GET_COMMITS）、L95-L99（GET_LAST_COMMIT）
- 资源级二次过滤：`packages/amplication-client/src/Workspaces/WorkspaceFooter.tsx` L39-L46

三层过滤保证用户在 Footer 看到的 Commit/Build ID 与当前上下文一致：**当前项目 → Services 资源组 → 当前选中资源**。

---

## 4. 容易混淆的点梳理

### 4.1 `resources` vs `catalog` 两个 Query 的区别

后端 resolver 暴露了两个查询：

| Query | 返回结构 | 使用场景 |
|---|---|---|
| `resources(where)` | `Resource[]` | `useResources` hook，项目内上下文，需要完整字段但不分页 |
| `catalog(where, take, skip, orderBy)` | `{ totalCount, data: Resource[] }` | `useCatalog` hook，带分页 + 总数，用于表格和图 |

两者底层都调用同一个 `prepareResourceFindManyArgsForQuery`，过滤逻辑完全一致，区别只在 **是否返回 totalCount** 和 **分页参数**。

### 4.2 `project.resources`（轻量） vs Catalog 里的 Resource（完整）

- `GET_PROJECTS` 内嵌的 `project.resources` 只有 `id/name/resourceType/licensed/gitRepository` —— 够概览页用
- `SEARCH_CATALOG` 返回的 Resource 有 builds / owner / blueprint / version / properties 等全量字段 —— 列表页需要

不要在概览页强求完整字段，也不要在列表页依赖 project.resources 的内嵌数据。

### 4.3 fixedFiltersKey 的作用

`CatalogGrid` / `DataGridFilters` 有个参数 `fixedFiltersKey`：
- Workspace Catalog 传 `"workspace-catalog"`
- 项目内 ResourceList 传 `currentProject.id`
- CatalogGraph 传 `"catalog-graph"`

这个 key 用于把用户的筛选条件持久化到 localStorage，**不同上下文的筛选互不干扰**。

### 4.4 两个 Resource 过滤管线

`useCatalog.setFilter` 把筛选条件拆成两条：
- **内置字段**（resourceType / projectId / ownership）→ 直接进 Prisma where
- **CustomProperties**（动态定义）→ `JsonPathStringFilter` → `jsonPathStringFilterToPrismaFilter()` 转成 Prisma 的 JSON path 查询

这是因为自定义属性存在 `Resource.properties` JSON 列里，不能走普通的关系过滤。

### 4.5 双实例模型：表格视图 vs 图视图的数据所有权与筛选边界

`useCatalog` 是一个工厂 hook，**每次调用都创建独立的状态副本**。表格视图和图视图分别调用了两次，形成两个平行实例：

| 维度 | 表格视图实例（Context 共享） | 图视图实例（独立） |
|---|---|---|
| 实例化位置 | `CatalogContextProvider` 内：`CatalogContext.tsx` L58 | `useCatalogGraph` 内：`useCatalogGraph.tsx` L42-L45 |
| 消费者 | `CatalogGrid.tsx` L59-L67 通过 `useCatalogContext()` | `CatalogGraph.tsx` L73 通过 `useCatalogGraph()` |
| 页面场景 | 工作区 Catalog / 项目 ResourceList（同一实例，场景切换保留状态） | 工作区总图 WorkspaceGraph |
| pageSize | 默认 20（分页加载） | 1000（一次性加载） |
| 筛选持久化 key | `fixedFiltersKey`：`"workspace-catalog"` 或 `currentProject.id` | 固定 `"catalog-graph"`（CatalogGraph.tsx L178） |
| 搜索词 | 内存状态（Context 内部），不跨实例共享 | 内存状态（useCatalogGraph 内部），不跨实例共享 |
| 附加状态 | 分页 / 排序 | 分组方式 / 布局参数 / 节点位置（持久化到 `catalogGraphLayout-${workspaceId}`） |
| Apollo 缓存 | 共享同一 Apollo Client，但因为 pageSize/变量不同，缓存条目独立 | 共享同一 Apollo Client，缓存条目独立 |

**筛选边界结论：**
- 在工作区 Catalog 页输入搜索词 → 切到项目 ResourceList：搜索词会保留（同一个 Context 实例）
- 在 Catalog 页输入搜索词 → 切到 Graph 总图：搜索词**不会**保留（两个独立实例）
- 在 Graph 总图调整布局参数 / 分组方式 → 回到表格页：不产生任何影响
- 两者唯一共享的是同一个后端 resolver 和 Apollo Client 缓存层，但因为查询变量不同，缓存不会互相复用

---

## 5. 总结：数据流向总图

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           WorkspaceLayout                                  │
│  ┌───────────────┐  ┌──────────────────┐  ┌─────────────────┐              │
│  │useWorkspaceSel│  │useProjectSelector│  │usePendingChanges│              │
│  └───────┬───────┘  └────────┬─────────┘  └────────┬────────┘              │
│          │                   │                     │                        │
│  ┌───────▼───────────────────▼─────────────────────▼───────────┐            │
│  │                    AppContextProvider                       │            │
│  │  currentWorkspace / projectsList / pendingChanges / ...    │            │
│  └───────────────────────────┬─────────────────────────────────┘            │
│                              │                                              │
│    ┌─────────────────────────┴───────────────────────────┐                  │
│    │                                                       │                  │
│    ▼                                                       ▼                  │
│  ┌─────────────────────────────┐          ┌─────────────────────────────┐  │
│  │   CatalogContextProvider    │          │     useCatalogGraph          │  │
│  │  ┌───────────────────────┐  │          │  ┌───────────────────────┐  │  │
│  │  │  useCatalog()         │  │          │  │ useCatalog(1000/page)  │  │  │
│  │  │  (默认 20/page)       │  │          │  │ (独立实例，不经Context) │  │  │
│  │  └──────────┬────────────┘  │          │  └──────────┬────────────┘  │  │
│  └─────────────┼───────────────┘          └─────────────┼───────────────┘  │
│                │                                        │                  │
│                ▼                                        ▼                  │
│      ┌──────────────────┐                  ┌──────────────────────────┐   │
│      │   CatalogGrid    │                  │      CatalogGraph        │   │
│      │ (工作区/项目表格) │                  │  (WorkspaceGraph 总图)   │   │
│      └──────────────────┘                  │ nodes / edges / 分组/布局 │   │
│                                             └──────────────────────────┘   │
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │                   后端 (amplication-server)                         │    │
│  │  resource.resolver.catalog()                                        │    │
│  │    └─ resource.service.searchResourcesWithCount()                   │    │
│  │         └─ prepareResourceFindManyArgsForQuery()                    │    │
│  │              └─ prisma.resource.count + findMany                    │    │
│  │         └─ @ResolveField 按需解析 project/owner/builds...           │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────────┘
```

关键设计原则：
1. **表格内部共享，图表各自独立**：表格视图通过 CatalogContextProvider 共享一个 useCatalog 实例；图视图绕过 Context，自己独立实例化 useCatalog
2. **视图复用，参数隔离**：都复用 `useCatalog` 工厂 hook，但通过不同的 `pageSize`、`fixedFiltersKey`、localStorage key 实现状态完全隔离
3. **字段按需分级**：概览用内嵌轻量资源，列表走完整 catalog 查询
4. **状态解耦**：Pending Changes、Build 状态等暂态/派生数据走独立链路
5. **总图独立建模**：图视图需要一次性加载 1000 条资源做拓扑，不与分页表格共享数据是合理的设计
