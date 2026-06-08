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
                    │   (仅 HTTP/Upload 请求；WebSocket 未传递)
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              amplication-server (NestJS)               │
│  ┌─────────────────────────────────────────────┐      │
│  │    AnalyticsSessionIdInterceptor              │      │
│  │    (提取 header → req.analyticsSessionId)       │      │
│  └──────────────────┬───────────────────────┘      │
│                     │                               │
│  ┌─────────────────────────────────────────────┐      │
│  │    parseValidUnixTimestampOrUndefined          │      │
│  │    (仅非负整数型 Unix 时间戳才通过校验)          │      │
│  └──────────────────┬───────────────────────┘      │
│                     │                               │
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
- **Hotjar**（第 30-44 行）**：用户行为录像/热力图（硬编码 hjid: 2379803）
- **HubSpot**（第 108-120 行）**：CRM 聊天小部件（硬编码 ID 25691669）
  - 注意：HubSpot 脚本及配置**无任何环境变量开关控制**，在生产构建中始终注入

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

**发送边界（仅 Segment）**：仅当环境变量 `REACT_APP_ANALYTICS_API_KEY` 存在时才执行 `analytics.load()` 并 dispatch `AppSessionStart`。
**注意（HubSpot）**：HubSpot 的脚本加载和数据上报**完全不受该变量控制**，始终执行。

### 2.2 会话 ID（analytics_session_id）——来源与写入分析

#### 2.2.1 关键发现：客户端不存在写入逻辑

客户端在 [analytics.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts#L16-L17) 定义：

```typescript
const ANALYTICS_SESSION_ID_KEY = "analytics_session_id";
export const ANALYTICS_SESSION_ID_HEADER_KEY = "analytics-session-id";
```

读取逻辑：[analytics.ts#L92-L94](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts#L92-L94)

```typescript
export function getSessionId(): string | null {
  return localStorage.getItem(ANALYTICS_SESSION_ID_KEY);
}
```

**重大修正**：经全仓库搜索，**不存在任何 `localStorage.setItem("analytics_session_id", ...)` 的写入代码**。该值完全依赖外部系统（浏览器端）预先写入，可能的来源包括：
1. 部署环境中由外部脚本（如反向代理、A/B 测试框架、其他埋点 SDK）写入
2. 登录/注册流程中由服务器端通过 Set-Cookie 或响应体间接写入（当前代码中也未发现）
3. Segment SDK 自身的 `ajs_anonymous_id`（默认 key 不同，需额外映射）

若 localStorage 中不存在该 key，`getSessionId()` 将始终返回 `null`，后续所有链路的 `anonymousId` 均为 `undefined`。

#### 2.2.2 请求头传递链路（HTTP vs WebSocket）

**HTTP 请求（Query / Mutation / Upload）**：通过 [graphqlClient.ts#L45-L56](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/graphqlClient.ts#L45-L56) 的 Apollo `authLink` 将 Session ID 注入所有 GraphQL 请求头：

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

该 `authLink` 被 concat 到三条链路上传：
- `authLink.concat(uploadLink)` —— 带文件上传的请求
- `authLink.concat(httpLink)` —— 普通 Query/Mutation
- `authLink.concat(wsLink)` —— Subscription（⚠️ 见下）

**WebSocket（Subscription）—— 未传递**：[graphqlClient.ts#L24-L31](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/graphqlClient.ts#L24-L31)

```typescript
const wsLink = new GraphQLWsLink(
  createClient({
    url: REACT_APP_DATA_SOURCE.replace("http", "ws").replace("https", "wss"),
    connectionParams: () => ({
      authorization: `Bearer ${getToken()}`,
      // ⚠️ 此处未包含 analytics-session-id
    }),
  })
);
```

虽然 `authLink.concat(wsLink)` 在链路中串联了 authLink，但 **GraphQL over WebSocket 不使用 HTTP headers，而是使用 `connectionParams` 在握手阶段传递上下文。** 服务端 [app.module.ts#L44-L57](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/app.module.ts#L44-L57) 的 context 构造中，WS 请求将 `connectionParams` 合并入 headers，但由于客户端 connectionParams 中未包含 session id，所有 Subscription 触发的服务端事件（如 build 状态推送回调等）**无法关联到客户端会话**。

---

### 2.3 事件上报 API 与 Segment/HubSpot 开关差异

所有客户端 API 定义在 [analytics.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts)：

| API | 用途 | Segment 写入条件 | HubSpot 写入条件 |
|-----|------|-----------------|-----------------|
| `dispatch(event)` | 自定义行为事件 | `REACT_APP_ANALYTICS_API_KEY` 存在 | **无条件**（始终 `_hsq.push`） |
| `identity(userId, props)` | 用户识别 | `REACT_APP_ANALYTICS_API_KEY` 存在 | **无条件**（始终 `_hsq.push`） |
| `page(name, props)` | 页面浏览 | `REACT_APP_ANALYTICS_API_KEY` 存在 | **无条件**（始终 `_hsq.push`） |
| `init()` | SDK 加载 + `AppSessionStart` | `REACT_APP_ANALYTICS_API_KEY` 存在 | HubSpot 脚本在 HTML 中硬编码加载，不受此控制 |
| `track` (react-tracking HOC) | 组件级声明式跟踪 | 通过 dispatch，同上 | 通过 dispatch，同上 |
| `useTracking` | Hook 式跟踪 | 通过 dispatch，同上 | 通过 dispatch，同上 |

**dispatch 双写逻辑**（[analytics.ts#L33-L49](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics.ts#L33-L49)）：

```typescript
export function dispatch(event: Partial<Event>) {
  const { eventName, ...rest } = event;
  const versionObj = version ? { version } : {};
  // HubSpot：无任何开关，始终写入
  _hsq.push([
    "trackCustomBehavioralEvent",
    { name: eventName, properties: { ...versionObj, ...rest } },
  ]);
  // Segment：仅当配置了 API Key 时才写入
  if (REACT_APP_ANALYTICS_API_KEY) {
    const analytics = window.analytics;
    analytics.track(eventName || MISSING_EVENT_NAME, {
      ...versionObj,
      ...rest,
    });
  }
}
```

**开关差异总结表：**

| 维度 | Segment | HubSpot |
|------|---------|---------|
| 脚本加载 | index.html 中 Snippet 始终注入，但 `analytics.load(KEY)` 仅在有 KEY 时调用 | index.html 中 `<script src="//js-eu1.hs-scripts.com/25691669.js">` 始终注入，无条件 |
| track/dispatch | `if (REACT_APP_ANALYTICS_API_KEY)` 守卫 | 无守卫，始终 `_hsq.push` |
| identity | `if (REACT_APP_ANALYTICS_API_KEY)` 守卫 | 无守卫，始终 `_hsq.push` |
| page | `if (REACT_APP_ANALYTICS_API_KEY)` 守卫 | 无守卫，始终 `_hsq.push(["trackPageView"])` |
| SDK 初始化 | 有 KEY 时 `analytics.load(KEY)` 才真正下载 analytics.js | HTML 解析到 `<script>` 标签时立即下载 |
| 私有化配置 | `REACT_APP_ANALYTICS_DOMAIN` 控制 CDN 域名（index.html 模板变量） | 硬编码 `js-eu1.hs-scripts.com/25691669.js` |

**实际影响**：在未配置 `REACT_APP_ANALYTICS_API_KEY` 的开发/自托管环境中，Segment 完全静默，但 HubSpot 仍会持续收集并上报所有 `trackCustomBehavioralEvent`、`identify` 和 `trackPageView` 事件。

---

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

[usePageTracking.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/usePageTracking.ts) Hook 用于路由级别页面浏览上报。

### 2.6 客户端事件类型枚举

完整事件名定义在 [analytics-events.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-client/src/util/analytics-events.types.ts)，共 170+ 事件。

---

## 三、服务端采集链路

### 3.1 模块注册

**模块定义**：[segmentAnalytics.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.module.ts)

在 [app.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/app.module.ts#L71-L73) 中全局注册。

### 3.2 配置读取

[segmentAnalyticsOptionsService.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalyticsOptionsService.ts) 从环境变量 `SEGMENT_WRITE_KEY_SECRET` 读取 Segment Write Key。

**发送边界**：[segmentAnalytics.service.ts#L23-L28](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts#L23-L28)：只有当 `segmentWriteKey` 存在且非空时，才初始化 `Analytics` 实例。所有上报方法开头都有 `if (!this.analytics) return;` 守卫。

### 3.3 Session ID 拦截器与 Unix 时间戳校验

#### 3.3.1 拦截器提取

[analytics-session-id.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/interceptors/analytics-session-id.interceptor.ts) 作为全局 APP_INTERCEPTOR（在 app.module.ts 第 84-87 行注册），从请求头提取 `analytics-session-id` 并挂载到 `req.analyticsSessionId`：

```typescript
const analyticsSessionId = req.headers[ANALYTICS_SESSION_ID_HEADER_KEY];
req.analyticsSessionId = analyticsSessionId;
```

#### 3.3.2 非负整数 Unix 时间戳过滤（parseValidUnixTimestampOrUndefined）

**核心校验函数**：[segmentAnalytics.service.ts#L30-L41](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts#L30-L41)

```typescript
private parseValidUnixTimestampOrUndefined(
  value: string
): string | undefined {
  const timestamp = parseInt(value, 10);
  // Check if the value is an integer and within a valid range for Unix timestamps
  if (!isNaN(timestamp) && Number.isInteger(timestamp) && timestamp >= 0) {
    return value;
  } else {
    return undefined;
  }
}
```

**校验规则详解：**

| 输入值 (value) | parseInt 结果 | isNaN? | Number.isInteger? | >= 0? | 返回值 |
|----------------|--------------|--------|-------------------|-------|-------|
| `"1717800000"` (合法 Unix 时间戳) | `1717800000` | ❌ | ✅ | ✅ | `"1717800000"` |
| `"0"` | `0` | ❌ | ✅ | ✅ | `"0"` |
| `"123"` | `123` | ❌ | ✅ | ✅ | `"123"` |
| `"abc"` (非数字字符串) | `NaN` | ✅ | ❌ | - | `undefined` |
| `"uuid-xxx-123"` | `NaN` | ✅ | ❌ | - | `undefined` |
| `"-1"` (负数) | `-1` | ❌ | ✅ | ❌ | `undefined` |
| `"123.45"` (浮点数字符串) | `123` | ❌ | ✅ | ✅ | `"123.45"` ⚠️（见下方注释） |
| `undefined` (header 不存在) | `NaN` | ✅ | ❌ | - | `undefined` |
| `null` | `NaN` | ✅ | ❌ | - | `undefined` |
| `" "` (空字符串/空格) | `NaN` | ✅ | ❌ | - | `undefined` |

⚠️ **注意**：对于 `"123.45"`，`parseInt("123.45", 10)` 返回 `123`，因此校验通过，函数返回原始字符串 `"123.45"`（而非截断后的 `"123"`）。该值会被原样传给 Segment 作为 `anonymousId`。

**实际含义**：
- 该函数的命名和注释暗示 `analytics_session_id` 预期是一个 **Unix 时间戳**（毫秒或秒级），而非 UUID 或随机字符串。
- 若客户端 localStorage 中写入的是 UUID、随机字符串等格式（更常见的 session id 形式），**将被全部过滤为 `undefined`**，导致服务端无法关联 anonymousId。
- 该函数在 `identify()` 和 `trackManual()` 两个入口被调用，过滤结果直接决定了是否发送 `anonymousId`。

---

### 3.4 SegmentAnalyticsService 核心 API 与 anonymousId 使用条件

[segmentAnalytics.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts) 提供三个核心方法。

#### 3.4.1 `identify(data: IdentifyData)`

[segmentAnalytics.service.ts#L43-L60](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts#L43-L60)

```typescript
public async identify(data: IdentifyData): Promise<void> {
  if (!this.analytics) return;
  const req = RequestContext?.currentContext?.req;
  const analyticsSessionId = this.parseValidUnixTimestampOrUndefined(
    req?.analyticsSessionId
  );
  try {
    this.analytics.identify({
      userId: data.accountId,
      anonymousId: analyticsSessionId,  // 可能为 undefined
      traits: data,
    });
  } catch (error) { ... }
}
```

**anonymousId 使用条件**：
- `identify` 始终同时发送 `userId` 和 `anonymousId`（只要校验通过）
- 用途：在用户注册/登录后，将之前匿名会话（anonymousId）与正式账号（userId）在 Segment 后台合并
- 若 `analyticsSessionId` 校验未通过（返回 undefined），Segment SDK 将忽略 `anonymousId` 字段，仅按 `userId` 识别

#### 3.4.2 `trackWithContext(data: EventTrackData)`

[segmentAnalytics.service.ts#L101-L121](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts#L101-L121)

**适用场景**：已登录用户、处于 HTTP 请求上下文中的调用。自动从 `RequestContext.currentContext.req` 获取 `user`（包含 `accountId` 和 `workspaceId`），内部调用 `trackManual`。

**注意**：不适用于 Kafka 事件处理器等非 HTTP 请求场景。

#### 3.4.3 `trackManual({ user, data })`

[segmentAnalytics.service.ts#L129-L173](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalytics.service.ts#L129-L173)

**anonymousId 使用条件（关键）**：

```typescript
const analyticsSessionId = this.parseValidUnixTimestampOrUndefined(
  req?.analyticsSessionId
);
// ...
const trackData: TrackParams = {
  event: data.event,
  userId: user?.accountId,
  // If the user is not logged in, use an anonymous ID from the client to track the event and merge the user on signup
  anonymousId: analyticsSessionId,
  properties: {
    ...eventProperties,
    ...data.properties,
    source: "amplication-server",
  },
  context: {
    ...data.context,
    amplication: {
      analyticsSessionId: analyticsSessionId,  // 可能为 undefined → {}
    },
  },
};
```

**anonymousId 使用条件矩阵：**

| 场景 | user.accountId | analyticsSessionId (校验后) | Segment 最终收到 |
|------|---------------|-----------------------------|-----------------|
| 已登录 + 合法 session id | 存在 (string) | 存在 (string，非负整数) | `userId` + `anonymousId` 同时发送 |
| 已登录 + 非法/缺失 session id | 存在 (string) | `undefined` | 仅发送 `userId`，`anonymousId` 被忽略 |
| 未登录 + 合法 session id | `undefined` | 存在 (string，非负整数) | 仅发送 `anonymousId` |
| 未登录 + 非法/缺失 session id | `undefined` | `undefined` | 两者均缺失，Segment 可能拒绝或使用 SDK 自身 anonymousId |

**关于注释与实际代码的差异**：注释写着 *"If the user is not logged in, use an anonymous ID"*，但实际上**无论用户是否登录**，只要 session id 校验通过，`anonymousId` 都会被发送。这种做法（同时传 userId + anonymousId）在 Segment 的最佳实践中被称为 **Identity Merge**，用于在用户登录后将之前的匿名行为与账号关联。

最终发送到 Segment 的 Track 数据结构：

```typescript
{
  event: data.event,
  userId: user?.accountId,              // 可能为 undefined（未登录）
  anonymousId: analyticsSessionId,      // 可能为 undefined（校验未通过）
  properties: {
    workspaceId,
    $groups: { groupWorkspace: workspaceId },
    projectId,
    resourceId,
    ...data.properties,
    source: "amplication-server",
  },
  context: {
    ...data.context,
    amplication: {
      analyticsSessionId: analyticsSessionId,  // 仅当校验通过时有值
    },
  }
}
```

---

### 3.5 服务端事件类型枚举

完整事件定义在 [segmentAnalyticsEventType.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/109-amplication/packages/amplication-server/src/services/segmentAnalytics/segmentAnalyticsEventType.types.ts)，共 60+ 事件。

### 3.6 主要调用场景分布

（同原版文档，略）

---

## 四、发送边界与开关控制

### 4.1 客户端开关（修正后）

| 层级 | 控制变量 | 位置 | Segment | HubSpot | 说明 |
|------|---------|------|---------|---------|------|
| SDK 脚本加载 | `REACT_APP_ANALYTICS_API_KEY` | analytics.ts `init()` | 有 Key 时才 `analytics.load()` 下载 SDK | 不受控制，index.html 中硬编码 `<script>` 始终加载 | HubSpot 脚本 ID `25691669` 写死在 HTML 中 |
| track (dispatch) | `REACT_APP_ANALYTICS_API_KEY` | analytics.ts | `if` 守卫 | **无条件**，始终 `_hsq.push` | |
| identity | `REACT_APP_ANALYTICS_API_KEY` | analytics.ts | `if` 守卫 | **无条件**，始终 `_hsq.push` | |
| page | `REACT_APP_ANALYTICS_API_KEY` | analytics.ts | `if` 守卫 | **无条件**，始终 `_hsq.push(["trackPageView"])` | |
| analytics_session_id header | localStorage | graphqlClient.ts | 通过 session id 关联 anonymousId | HubSpot 自身管理 cookie，不依赖该 header | 仅 HTTP 请求有；WebSocket 缺失 |

### 4.2 服务端开关

| 层级 | 控制变量 | 位置 | 说明 |
|------|---------|------|------|
| 整体 | `SEGMENT_WRITE_KEY_SECRET` | segmentAnalyticsOptionsService.ts | 无 Key 时，analytics 实例为 undefined，所有方法空操作 |
| anonymousId 有效性 | `parseValidUnixTimestampOrUndefined` 逻辑内联 | segmentAnalytics.service.ts | session id 非非负整数格式时，等同于未发送 anonymousId |

### 4.3 错误静默失败

- 所有 analytics 调用均使用 try-catch 包裹，失败仅打日志，不影响业务流程。

### 4.4 异步 fire-and-forget

业务调用（如 auth.service.ts）使用 `void this.analytics.trackManual(...).catch(...)` 模式，不 await，不阻塞主流程。

---

## 五、跨端事件关联机制（修正后完整链路）

```
浏览器                                     amplication-server
  │                                            │
  │ 0. [外部写入] localStorage["analytics_session_id"]
  │    （注意：当前仓库中不存在写入代码）        │
  │                                            │
  │ 1. getSessionId()                           │
  │    localStorage.getItem("analytics_session_id") │
  │    → 返回 string 或 null                     │
  │                                            │
  │ 2. HTTP 请求 (Query/Mutation/Upload)        │   WebSocket (Subscription)
  │    authLink 设置 header                       │   connectionParams 未传 session id
  │    "analytics-session-id": <value或null>      │   ⚠️ 无法关联
  │──────────────────────────────────────────────►│────────────────────────┐
  │                                            │                        │
  │                                            │ 3. AnalyticsSessionIdInterceptor
  │                                            │    req.headers["analytics-session-id"]
  │                                            │    → req.analyticsSessionId
  │                                            │
  │                                            │ 4. parseValidUnixTimestampOrUndefined()
  │                                            │    仅当值为"非负整数字符串"时通过
  │                                            │    非法值 → undefined
  │                                            │
  │                                            │ 5. trackWithContext / trackManual / identify
  │                                            │    - userId = accountId（登录时）
  │                                            │    - anonymousId = 校验通过的 session id
  │                                            │      (可能为 undefined)
  │                                            │
  │                                            │ 6. 发送到 Segment
  │                                            │    context.amplication.analyticsSessionId
  ▼                                            ▼
           Segment 后台根据 userId + anonymousId 进行用户合并
                      （若 anonymousId 被过滤掉则无法合并）
```

**修正后的关键关联结论：**

1. **前提条件**：localStorage 中必须存在键 `analytics_session_id` 且值为**非负整数格式的字符串**（如 Unix 时间戳），否则关联完全失效。
2. **登录用户**：`userId` + `anonymousId` 同时发送时，Segment 可做 Identity Merge，将登录前后行为合并。
3. **未登录用户**：仅 `anonymousId` 可用，作为唯一追踪标识。
4. **WebSocket 请求**：Subscription 场景下不传递 session id，该类请求触发的服务端事件完全无法关联客户端会话。
5. **HubSpot 侧**：HubSpot 通过自身的 Cookie（`hubspotutk`）管理会话标识，不依赖 `analytics-session-id` header，因此不受以上链路问题影响。

---

## 六、类型定义索引（同原版）

## 七、核心文件速查（同原版）

---

## 八、重要发现汇总

| # | 发现 | 位置 | 影响 |
|---|------|------|------|
| 1 | **analytics_session_id 无客户端写入代码** | 全仓库搜索无 `setItem("analytics_session_id")` | 若部署环境未通过其他途径写入 localStorage，服务端 anonymousId 始终为 undefined |
| 2 | **session id 必须是非负整数字符串** | `parseValidUnixTimestampOrUndefined` | UUID、随机字符串等常见 session id 格式将被过滤，关联失效 |
| 3 | **WebSocket (Subscription) 不传 session id** | `graphqlClient.ts` 的 `connectionParams` | 所有 Subscription 场景的服务端事件无法关联客户端会话 |
| 4 | **HubSpot 完全不受 Segment 开关控制** | `analytics.ts` 中 dispatch/identity/page 无 HubSpot 守卫 + index.html 硬编码脚本 | 开发/自托管环境未配置 Segment Key 时，HubSpot 仍持续上报所有事件 |
| 5 | **anonymousId 同时发给登录用户** | `trackManual` 实际代码与注释差异 | 这是 Segment Identity Merge 的正确做法（注释描述不完整），登录用户也能关联之前的匿名行为 |
