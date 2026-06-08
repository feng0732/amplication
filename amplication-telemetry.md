# Amplication Telemetry & 使用分析上报链路

## 一、整体架构概览

Amplication 的遥测系统由 **客户端（Browser）** 与 **服务端（Node.js）** 双端协同采集，最终汇聚到两个外部分析平台：

```
┌─────────────────────────────────────────────────────────────┐
│                   amplication-client (Browser)               │
│  ┌──────────────┐   ┌──────────────────────┐      │
│  │ Segment SDK      │   │ HubSpot (_hsq)        │      │
│  │ (analytics.js)│   │                      │      │
│  └───────┬───────┘   └──────────┬───────────┘      │
│          │                    │                   │
│          └────────┬───────────┘                   │
│                   │                                │
│          dispatch() / identity() / page()                │
└───────────────────┬────────────────────────────────┘
                    │ HTTP Header: analytics-session-id
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              amplication-server (NestJS)               │
│  ┌─────────────────────────────────────────────┐      │
│  │       SegmentAnalyticsService                 │      │
│  │  (@segment/analytics-node)                    │      │
│  └──────────────────┬───────────────────────┘      │
│                     │                               │
│        trackWithContext() / trackManual() / identify()    │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
            ┌───────────────────┐
            │   Segment API    │
            │  (Segment.io)   │
            └───────────────────┘
```

**关键特性：**
- **客户端**：Segment SDK（`analytics.js`）+ HubSpot（`_hsq`）双写
- **服务端**：`@segment/analytics-node` SDK
- **会话串联**：通过 `analytics-session-id` 请求头将前后端事件关联
- **用户识别**：登录用户使用 `accountId` 作为 `userId`，未登录使用 `anonymousId`

---

## 二、客户端采集链路

### 2.1 SDK 初始化

**入口文件**：[index.html](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/public/index.html#L45-L104)

页面加载时预先注入 Segment Snippet，创建 `window.analytics` 队列，所有 `track/identify/page` 方法先进入队列，SDK 加载后自动 flush。

同时嵌入：
- **Hotjar**（第 30-44 行）**：用户行为录像/热力图
- **HubSpot**（第 108-120 行）**：CRM 聊天小部件

**初始化调用**：[App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/App.tsx#L68-L70)

```tsx
useEffect(() => {
  initAnalytics();
}, []);
```

`initAnalytics()` 实现：[analytics.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts#L51-L66)

```typescript
export function init() {
  if (REACT_APP_ANALYTICS_API_KEY) {
    const analytics = window.analytics;
    analytics.load && analytics.load(REACT_APP_ANALYTICS_API_KEY);
    dispatch({
      eventName: AnalyticsEventNames.AppSessionStart,
    });
  }
}
```

**发送边界**：仅当环境变量 `REACT_APP_ANALYTICS_API_KEY` 存在时才执行。

### 2.2 会话 ID（analytics_session_id）

客户端在 [analytics.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts#L16-L17) 定义：

```typescript
const ANALYTICS_SESSION_ID_KEY = "analytics_session_id";
export const ANALYTICS_SESSION_ID_HEADER_KEY = "analytics-session-id";
```

通过 [graphqlClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/graphqlClient.ts#L45-L56) 将 Session ID 通过 Apollo Link 注入所有 GraphQL 请求头：

```typescript
const authLink = setContext((_, { headers }) => {
  const token = getToken();
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",
      [ANALYTICS_SESSION_ID_HEADER_KEY]: getSessionId(),
    },
  };
});
```

### 2.3 事件上报 API

所有客户端 API 定义在 [analytics.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts)：

| API | 用途 | 发送目标 |
|-----|------|---------|
| `dispatch(event)` | 自定义行为事件 | Segment + HubSpot 双写 |
| `identity(userId, props)` | 用户识别 | Segment + HubSpot 双写 |
| `page(name, props)` | 页面浏览 | Segment + HubSpot 双写 |
| `track` (react-tracking HOC) | 组件级声明式跟踪 | 通过 dispatch |
| `useTracking` | Hook 式跟踪 | 通过 dispatch |

**dispatch 双写逻辑**（[analytics.ts#L33-L49](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts#L33-L49)：

```typescript
export function dispatch(event: Partial<Event>) {
  const { eventName, ...rest } = event;
  const versionObj = version ? { version } : {};
  // HubSpot
  _hsq.push([
    "trackCustomBehavioralEvent",
    { name: eventName, properties: { ...versionObj, ...rest } },
  ]);
  // Segment (仅当配置了 API Key 时)
  if (REACT_APP_ANALYTICS_API_KEY) {
    const analytics = window.analytics;
    analytics.track(eventName || MISSING_EVENT_NAME, {
      ...versionObj,
      ...rest,
    });
  }
}
```

### 2.4 用户识别触发点

在 [UserBadge.tsx](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/Components/UserBadge.tsx#L12-L17) 中，当用户数据加载完成后调用 `identity()`：

```typescript
useEffect(() => {
  if (data) {
    identity(data.account.id, {
      createdAt: data.account.createdAt,
      email: data.account.email,
    });
  }
}, [data]);
```

### 2.5 页面追踪

[usePageTracking.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/usePageTracking.ts) Hook 用于路由级别页面浏览上报：

```typescript
useEffect(() => {
  analytics.page(path.replaceAll("/", "-"), {
    path,
    url,
    params: match.params,
  });
}, []);
```

### 2.6 客户端事件类型枚举

完整事件名定义在 [analytics-events.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics-events.types.ts)，共 170+ 事件，涵盖：

| 分类 | 示例事件 |
|------|---------|
| 会话/认证 | `startAppSession, signInWithGitHub, EmailLogin |
| 工作区 | createWorkspace, inviteUser, selectWorkspace |
| 项目/服务 | createProject, createResourceFromScratch |
| 实体/字段 | importPrismaSchemaClick |
| 构建/提交 | commitClicked, openGithubCodeView |
| Git 同步 | startAuthResourceWithGitHub, createGitRepository |
| 计费升级 | PricingPageCTAClick, UpgradeClick |
| 向导流程 | ServiceWizard\*, ViewServiceWizard\* |
| AI 助手 | AskJovuClick, CreateWithJovuClick |

---

## 三、服务端采集链路

### 3.1 模块注册

**模块定义**：[segmentAnalytics.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.module.ts)

在 [app.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/app.module.ts#L71-L73) 中全局注册：

```typescript
SegmentAnalyticsModule.registerAsync({
  useClass: SegmentAnalyticsOptionsService,
}),
```

### 3.2 配置读取

[segmentAnalyticsOptionsService.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalyticsOptionsService.ts) 从环境变量 `SEGMENT_WRITE_KEY_SECRET` 读取 Segment Write Key。

**发送边界**：[segmentAnalytics.service.ts#L23-L28](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts#L23-L28)：只有当 `segmentWriteKey` 存在且非空时，才初始化 `Analytics` 实例。所有上报方法开头都有 `if (!this.analytics) return;` 守卫。

### 3.3 Session ID 拦截器

[analytics-session-id.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/interceptors/analytics-session-id.interceptor.ts) 作为全局 APP_INTERCEPTOR（在 app.module.ts 第 84-87 行注册），从请求头提取 `analytics-session-id` 并挂载到 `req.analyticsSessionId`：

```typescript
const analyticsSessionId = req.headers[ANALYTICS_SESSION_ID_HEADER_KEY];
req.analyticsSessionId = analyticsSessionId;
```

### 3.4 SegmentAnalyticsService 核心 API

[segmentAnalytics.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts) 提供三个核心方法：

#### 3.4.1 `identify(data: IdentifyData)`

用户身份识别。使用 `accountId` 作为 `userId`，`analyticsSessionId` 作为 `anonymousId`，`data` 作为 traits。

#### 3.4.2 `trackWithContext(data: EventTrackData)`

**适用场景**：已登录用户、处于 HTTP 请求上下文中的调用。自动从 `RequestContext.currentContext.req` 获取 `user`（包含 `accountId` 和 `workspaceId`，内部调用 `trackManual`。

**注意**：不适用于 Kafka 事件处理器等非 HTTP 请求场景。

#### 3.4.3 `trackManual({ user, data })`

**适用场景**：未登录用户或非 HTTP 上下文（如 Kafka 回调）。手动传入 `user.accountId` 和 `user.workspaceId`。

自动事件自动 enrich 逻辑（[segmentAnalytics.service.ts#L62-L91](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts#L62-L91)）：

```typescript
private async getEventProperties(workspaceId, properties) {
  // 若只传 resourceId，自动反查 projectId
  if (!projectId && resourceId) { ...查数据库...
  }
  return {
    workspaceId,
    $groups: { groupWorkspace: workspaceId },
    projectId,
    resourceId,
  };
}
```

最终发送到 Segment 的 Track 数据结构：

```typescript
{
  event: data.event,           // 事件名（EnumEventType）
  userId: user?.accountId,       // 登录用户 ID
  anonymousId: analyticsSessionId,  // 未登录会话 ID
  properties: {
    workspaceId,
    $groups: { groupWorkspace: workspaceId },
    projectId,
    resourceId,
    ...data.properties,  // 业务自定义属性
    source: "amplication-server",
  },
  context: {
    ...data.context,
    amplication: { analyticsSessionId }
  }
}
```

### 3.5 服务端事件类型枚举

完整事件定义在 [segmentAnalyticsEventType.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalyticsEventType.types.ts)，共 60+ 事件：

| 分类 | 事件 |
|------|------|
| 注册/认证 | Signup, StartEmailSignup, CompleteEmailSignup |
| 工作区/订阅 | WorkspacePlanUpgradeRequest/Completed, WorkspacePlanDowngradeRequest, InvitationAcceptance, RedeemCoupon |
| 提交/构建 | commit, createResourceVersion |
| 资源创建 | createEntity, updateEntity, createEntityField, updateEntityField, EntityFieldFromImportPrismaSchemaCreate |
| 插件 | installPlugin, updatePlugin |
| 资源类型 | createService, createMessageBroker, createPluginRepository, createServiceTemplate, createComponent, createResourceFromTemplate |
| 模块/DTO/动作 | CreateModule, InteractModule, CreateUserAction, InteractUserAction, CreateUserDTO, InteractUserDTO, InteractAmplicationAction |
| 蓝图/团队/角色/自定义属性 | BlueprintCreate/Update/Delete, TeamCreate/Update/Delete..., RoleCreate/Update/Delete, CustomPropertyCreate/Update/Delete |
| 错误监控 | gitSyncError, codeGenerationError |
| 其他 | WorkspaceSelected, ServiceWizard_ServiceGenerated, GitHubAuthResourceComplete, CodeGeneratorVersionUpdate, DemoRepoCreate, StartJovuThread, 架构重构相关事件 |

### 3.6 主要调用场景分布

| 模块 | 文件 | 主要事件 | 方法 |
|------|------|---------|------|
| 账户 | [account.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/account/account.service.ts) | Signup | identify + trackManual |
| 认证 | [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/auth/auth.service.ts) | StartEmailSignup, CompleteEmailSignup | trackManual |
| 工作区 | [workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts) | InvitationAcceptance, RedeemCoupon | trackWithContext |
| 工作区 Resolver | [workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts) | selectWorkspace | trackWithContext |
| 项目 | [project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/project/project.service.ts) | commit, createResourceVersion, CreateDemoRepo | trackWithContext / trackManual |
| 资源 | [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | 各资源类型创建事件 | trackWithContext |
| 实体 | [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/entity/entity.service.ts) | createEntity/updateEntity/createEntityField 等 | trackWithContext / trackManual |
| 构建 | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/build/build.service.ts) | gitSyncError, codeGenerationError | trackManual（Kafka 回调场景） |
| Git | [git.provider.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/git/git.provider.service.ts) | GitHub 仓库相关事件 | trackWithContext |
| 计费 | [billing.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/billing/billing.service.ts) | WorkspacePlanUpgrade* 等 | trackWithContext |
| 订阅 | [subscription.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts) | WorkspacePlanUpgradeCompleted | trackManual |
| 团队 | [team.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/team/team.service.ts) | Team* 系列事件 | trackWithContext |
| 角色 | [role.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/role/role.service.ts) | RoleCreate/Update/Delete | trackWithContext |
| 蓝图 | [blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts) | Blueprint* 系列 | trackWithContext |
| 自定义属性 | [customProperty.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/customProperty/customProperty.service.ts) | CustomProperty* | trackWithContext |
| 插件安装 | [pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) | installPlugin, updatePlugin | trackWithContext |
| 模块/DTO/动作 | [module.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/module/module.service.ts), [moduleDto.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/moduleDto/moduleDto.service.ts), [moduleAction.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/moduleAction/moduleAction.service.ts) | CreateModule, InteractModule 等 | trackWithContext |
| AI 助手 | [assistant.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/assistant/assistant.service.ts) | StartJovuThread | trackWithContext |
| 服务模板 | [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | createResourceFromTemplate | trackWithContext |
| BTM（单体拆分） | [resourceBtm.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/core/resource/resourceBtm.service.ts) | architectureRedesign\* | trackWithContext |

---

## 四、发送边界与开关控制

### 4.1 客户端开关

| 层级 | 控制变量 | 位置 | 说明 |
|------|---------|------|------|
| Segment 加载 | `REACT_APP_ANALYTICS_API_KEY | analytics.ts | 无 Key 时，Segment 不初始化，dispatch 仅 HubSpot |
| HubSpot | 始终加载 | index.html | HubSpot `_hsq` 在 index.html 中始终注入 |

### 4.2 服务端开关

| 层级 | 控制变量 | 位置 | 说明 |
|------|---------|------|------|
| 整体 | `SEGMENT_WRITE_KEY_SECRET` | segmentAnalyticsOptionsService.ts | 无 Key 时，analytics 实例为 undefined，所有方法空操作 |
| 计费 | `BILLING_ENABLED` | billing.service.ts | 计费模块开关，不影响 analytics |

### 4.3 错误静默失败

- **try-catch 包裹所有 analytics 调用，失败仅打日志，不影响业务流程：

```typescript
// segmentAnalytics.service.ts
try {
  this.analytics.track(trackData);
} catch (error) {
  this.logger.error(this.analyticsErrorMessage, error, { data });
}
```

### 4.4 异步 fire-and-forget

所有 `trackManual` 在业务调用（如 auth.service.ts#L133-L152）使用 `void this.analytics.trackManual(...).catch(...) 模式，不 await，不阻塞主流程。

---

## 五、跨端事件关联机制

```
浏览器                                     amplication-server
  │                                            │
  │ 1. 生成 session ID (localStorage)            │
  │                                            │
  │ 2. 所有 GraphQL 请求                     │
  │    header: analytics-session-id             │
  │──────────────────────────────────────────────►│
  │                                            │ 3. AnalyticsSessionIdInterceptor
  │                                            │    提取到 req.analyticsSessionId
  │                                            │
  │                                            │ 4. 调用 trackWithContext / trackManual
  │                                            │    - userId = accountId（登录）
  │                                            │    - anonymousId = analyticsSessionId
  │                                            │
  │                                            │ 5. 发送到 Segment
  │                                            │    context.amplication.analyticsSessionId
  ▼                                            ▼
              Segment 后台根据 userId + anonymousId 进行用户合并
```

关键关联字段：
- 登录后，使用 `userId` 覆盖关联事件
- 未登录，使用 `anonymousId = analyticsSessionId 追踪
- Signup 后，Segment 根据 anonymousId 与 userId 自动合并用户画像

---

## 六、类型定义索引

| 文件 | 内容 |
|------|------|
| [segmentAnalytics.interfaces.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.interfaces.ts) | SegmentAnalyticsOptions, SegmentAnalyticsOptionsFactory |
| [segmentAnalytics.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.types.ts) | IdentifyData, EventTrackData, ContextEventProperties |
| [segmentAnalyticsEventType.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalyticsEventType.types.ts) | EnumEventType（服务端事件枚举） |
| [analytics-events.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics-events.types.ts) | AnalyticsEventNames（客户端事件枚举） |

---

## 七、核心文件速查

| 层级 | 文件 | 职责 |
|------|------|------|
| 客户端 SDK 初始化 | [analytics.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts) | dispatch/identity/page/init 封装 |
| 客户端页面追踪 | [usePageTracking.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/usePageTracking.ts) | 路由级 page 事件 |
| 客户端请求注入 | [graphqlClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/graphqlClient.ts) | session ID header 注入 |
| 客户端入口 | [App.tsx](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/App.tsx) | initAnalytics() 调用 |
| 服务端核心服务 | [segmentAnalytics.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts) | trackWithContext/trackManual/identify |
| 服务端模块 | [segmentAnalytics.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.module.ts) | 全局模块注册 |
| 服务端配置 | [segmentAnalyticsOptionsService.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalyticsOptionsService.ts) | Write Key 读取 |
| 服务端拦截器 | [analytics-session-id.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/interceptors/analytics-session-id.interceptor.ts) | session ID 提取 |
| 服务端应用入口 | [app.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/app.module.ts) | 模块与拦截器注册 |
