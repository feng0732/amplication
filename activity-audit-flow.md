# Activity Log 与审计线索串联流程分析

## 1. 整体架构概览

Amplication 的 Activity Log 与审计线索通过 **四层嵌套数据模型 + 两套独立 Kafka 事件流** 实现，将用户操作、资源变更与异步任务执行过程完整串联。

```
实际生效链路（代码有写入）
───────────────────────────────────────────────────────────────
数据模型层：
  UserAction (用户业务操作层) ──┐
                                 ├──► Action (动作容器层)
  Build (代码构建层)     ────────┤         │
                                           ├──► ActionStep ──► ActionLog
事件流层（Kafka）：
  USER_ACTION_TOPIC       → 用户账户级通知事件（注册/登录/切换工作区）
  USER_ACTION_LOG_TOPIC   → UserAction 业务操作的异步日志落库
  USER_BUILD_TOPIC        → 构建成功用户通知事件
  DSG_LOG_TOPIC / ...     → Build 相关的异步日志落库

───────────────────────────────────────────────────────────────
仅 Prisma Schema 预留（代码无任何写入）
───────────────────────────────────────────────────────────────
  Deployment (部署层) ── schema 定义 actionId 外键，但从未被填充
```

### 核心模块位置

| 模块 | 文件路径 |
|------|---------|
| UserAction 模块 | [userAction](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction) |
| Action 模块 | [action](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action) |
| Schema 注册中心 | [schema-registry](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry) |
| Kafka 主题定义 | [index.ts#KAFKA_TOPICS](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/index.ts#L29-L69) |
| 通知服务 | [notification-service](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src) |

---

## 2. 数据库模型关系

### 2.1 ER 图

```
实际写入数据库的关联：
  UserAction ──┐
               ├──► User (userId)
               ├──► Resource (resourceId)
               └──► Action ──► ActionStep ──► ActionLog
  Build ───────┘

仅 Schema 层定义、无任何代码写入：
  Deployment ── actionId (FK) ──► Action  (外键存在，但从未被填充)
```

### 2.2 各层模型详解

#### UserAction - 用户业务操作层
定义于 [schema.prisma#UserAction](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L564-L576)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | CUID 主键 |
| `userActionType` | EnumUserActionType | 操作类型：`DBSchemaImport` / `GptConversation` / `ProjectRedesign` |
| `metadata` | Json | 业务相关的自定义元数据（如导入的 schema 文件名、GPT 对话结果） |
| `userId` | String | 关联 User（谁发起的操作） |
| `resourceId` | String | 关联 Resource（在哪个资源上执行） |
| `actionId` | String | 关联 Action（用于追踪执行过程） |
| `createdAt` | DateTime | 创建时间 |
| `updatedAt` | DateTime | 更新时间 |

**操作类型枚举** 定义于 [types.ts#EnumUserActionType](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/types.ts#L5-L9)

> **注意**：`Build` 不经过 UserAction 表，直接通过 `actionId` 外键关联 Action，且有完整的 Step/Log 执行链路。`Deployment` 虽然在 Prisma schema 中定义了 `actionId` 外键，但**无任何实际执行或写日志代码**（详见第 8.3 节）。

#### Action - 动作容器层
定义于 [schema.prisma#Action](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L452-L459)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | CUID 主键 |
| `steps` | ActionStep[] | 包含的执行步骤 |

**实际使用 Action 的模型（代码有写入）**：
- `UserAction`：通过 `actionId` 关联，完整使用 Action → ActionStep → ActionLog 四层链路
- `Build`：通过 `actionId` 关联，独立创建 Step 并通过 Kafka 持续写入 Log

**仅 Schema 层预留（无任何代码写入）**：
- `Deployment`：Prisma schema 定义了 `actionId` 外键，但全局搜索无任何 `prisma.deployment.create/update` 调用，该字段从未被填充

#### ActionStep - 执行步骤层
定义于 [schema.prisma#ActionStep](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L461-L473)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | CUID 主键 |
| `name` | String | 步骤名称（如 `PROCESSING_PRISMA_SCHEMA`、`START_CONVERSATION`、`APPLYING_PROJECT_REDESIGN_CHANGES`） |
| `message` | String | 步骤描述信息 |
| `status` | ActionStepStatus | `Waiting` / `Running` / `Success` / `Failed` |
| `completedAt` | DateTime? | 完成时间 |
| `actionId` | String | 所属 Action |

步骤状态枚举定义于 [EnumActionStepStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/dto/EnumActionStepStatus.ts)

#### ActionLog - 详细日志层
定义于 [schema.prisma#ActionLog](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L475-L485)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | CUID 主键 |
| `message` | String | 日志消息 |
| `level` | EnumLogLevel | `Error` / `Warning` / `Info` / `Debug` |
| `meta` | Json | 附加元数据 |
| `stepId` | String | 所属 ActionStep |

日志级别枚举定义于 [EnumActionLogLevel.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/dto/EnumActionLogLevel.ts)

---

## 3. Activity Log 的实际范围：两套体系的区分

代码中存在两套名称相似但用途完全不同的体系，必须严格区分：

### 3.1 体系 A：USER_ACTION_TOPIC — 用户账户级通知事件（Kafka 专用，不入 UserAction 表）

**Topic 名**：`"user-action.internal.1"`

| 属性 | 说明 |
|------|------|
| **用途** | 仅用于通知服务的用户订阅管理（Novu 创建/删除订阅者） |
| **是否落库** | ❌ 不写入 UserAction 表 |
| **生产者** | [user.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/user/user.service.ts#L184-L199) |
| **消费者** | [notification-service/app.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/app.controller.ts#L14-L22) → [subscribeUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/notification-packages/subscribeUser.ts) |
| **Value Schema** | [user-action/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-action/value.ts) |

**消息字段**：

| 字段 | 说明 |
|------|------|
| `userId` | 用户账户 ID |
| `email` / `firstName` / `lastName` | 用户基本信息 |
| `externalId` | 外部系统 ID（Novu 通知系统的 subscriberId，加密后的 userId） |
| `action` | `SIGNUP` / `LOGIN` / `UPDATE` / `DELETE` / `CURRENT_WORKSPACE` |
| `enableUser` | 是否启用通知（true→创建订阅者，false→删除订阅者） |

**触发时机**：用户切换当前工作区时（`CURRENT_WORKSPACE`），用于同步 Novu 订阅状态。

**消费逻辑** [subscribeUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/notification-packages/subscribeUser.ts#L6-L19)：
```typescript
if (notificationCtx.topic !== KAFKA_TOPICS.USER_ACTION_TOPIC) return;
notificationCtx.notifications.push({
  notificationMethod: enableUser
    ? novuService.createSubscriber    // enableUser=true → 创建订阅者
    : novuService.deleteSubscriber,   // enableUser=false → 删除订阅者
  subscriberId: externalId,
  payload: restAccount,
});
```

### 3.2 体系 B：UserAction 数据库记录 — 用户业务操作审计（入 UserAction 表）

**持久化**：`UserAction` 表 + `Action` / `ActionStep` / `ActionLog` 四层关联

| 属性 | 说明 |
|------|------|
| **用途** | 追踪用户发起的业务操作（DB Schema 导入、AI 对话、架构重构 apply） |
| **是否落库** | ✅ 完整写入 UserAction 表 |
| **用户关联** | `userId` 外键 |
| **资源关联** | `resourceId` 外键 |
| **执行追踪** | `actionId` → Action → ActionStep[] → ActionLog[] |
| **Kafka 日志通道** | `USER_ACTION_LOG_TOPIC`（`"user-action.internal.action-log.1"`） |

**三种 UserAction 类型**：

| 类型 | 场景 | 同步/异步 |
|------|------|----------|
| `DBSchemaImport` | 导入 Prisma Schema 创建实体 | 异步（Kafka 中转） |
| `GptConversation` | AI 架构重构建议（BTM） | 异步（AI Gateway 回调） |
| `ProjectRedesign` | 架构重构 apply（实际落地变更） | 同步（直接在请求线程执行） |

### 3.3 两者对比总结

| 维度 | USER_ACTION_TOPIC（Kafka） | UserAction 数据库表 |
|------|---------------------------|-------------------|
| **关注点** | 用户账户通知订阅状态 | 用户业务操作执行过程 |
| **数据流向** | Kafka → Novu 通知系统 | 数据库 → 前端轮询展示 |
| **关联资源** | 不关联任何 Resource | 必须关联 resourceId |
| **ActionStep/Log** | 无 | 完整四层嵌套 |
| **操作类型** | SIGNUP/LOGIN/CURRENT_WORKSPACE | DBSchemaImport/GptConversation/ProjectRedesign |
| **前端可见** | 否 | 是（userAction query + 轮询 Hook） |

---

## 4. 用户上下文关联机制

### 4.1 UserAction 记录中的用户关联

每个 `UserAction` 都通过 `userId` 外键直接关联到 `User` 表，确保每一条活动记录都能追溯到具体的操作人。

核心创建逻辑位于 [userAction.service.ts#createUserActionByTypeWithInitialStep](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts#L20-L62)：

```typescript
const data: Prisma.UserActionCreateInput = {
  userActionType,
  metadata,
  user: { connect: { id: userId } },  // 关联发起用户
  action: {
    create: { steps: { create: initialStepData } },
  },
};
if (resourceId) {
  data.resource = { connect: { id: resourceId } };  // 关联目标资源
}
```

### 4.2 GraphQL 解析用户上下文

在 [userAction.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.resolver.ts#L51-L57) 中，通过 `@ResolveField` 动态解析关联的用户和资源信息：

```typescript
@ResolveField()
async user(@Parent() userAction: UserAction): Promise<User> {
  return this.userService.findUser({ where: { id: userAction.userId } }, true);
}

@ResolveField()
async resource(@Parent() userAction: UserAction): Promise<Resource> {
  return this.resourceService.resource({ where: { id: userAction.resourceId } });
}
```

### 4.3 Build 记录中的用户关联

Build 不经过 UserAction 表，直接通过自己的 `userId` 外键关联用户（以及 `createdBy` relation），同时通过 `USER_BUILD_TOPIC` 发送用户通知事件（不经过 Action 日志系统）。

USER_BUILD_TOPIC Value Schema 见 [user-build/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-build/value.ts)，构建成功后由 [buildCompleted.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/notification-packages/buildCompleted.ts) 消费，触发 Novu `build-completed` 通知。

---

## 5. 资源变更记录方式

### 5.1 UserAction 中的资源关联

`UserAction.resourceId` 外键直接关联到目标 `Resource`，实现变更记录与资源的绑定。

Resolver 中动态解析：[userAction.resolver.ts#L44-L49](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.resolver.ts#L44-L49)

### 5.2 业务元数据 metadata

`UserAction.metadata`（Json 类型）存储业务相关的具体变更内容：

| 场景 | metadata 内容示例 |
|------|-------------------|
| DBSchemaImport | `{ schema: "...", fileName: "schema.prisma" }` |
| GptConversation | `{ data: "<GPT response JSON>" }` |
| ProjectRedesign/BTM | `undefined`（不存 metadata，变更直接落库） |

更新 metadata 的方法：[userAction.service.ts#updateUserActionMetadata](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts#L131-L143)

### 5.3 实际场景：DB Schema Import

完整流程见 [dbSchemaImport.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts)：

1. **启动时**：创建 UserAction，将 Prisma schema 文件内容存入 metadata
2. **消费 Kafka 消息**：从 UserAction 中取出 metadata 和 resourceId
3. **创建实体过程**：通过 `ActionContext` 持续输出日志
4. **完成时**：通过 `onEmitUserActionLog` 标记步骤成功/失败

---

## 6. 场景补充：资源重构 Apply 阶段（ProjectRedesign）

这是**唯一同步执行**的 UserAction 场景，在 HTTP 请求线程内直接完成所有数据库变更并持续输出日志。

### 6.1 完整流程

入口方法：[resource.service.ts#redesignProject](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L743-L1147)

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. 前端触发 mutation redesignProject                                 │
│    参数：movedEntities[], newServices[], projectId                   │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. 创建 UserAction + 初始 Step（同步落库）                            │
│    type: ProjectRedesign                                              │
│    userId: 当前用户 id                                                │
│    resourceId: 原始服务 id（movedEntities[0].originalResourceId）     │
│    metadata: undefined（不使用）                                      │
│    Step 名称: APPLYING_PROJECT_REDESIGN_CHANGES (Running)            │
│    初始日志: "Starting to apply project redesign changes"             │
│    初始 Step 数据见 constants.ts                                      │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. 创建 ActionContext（绑定 userAction.id + step.id）                │
│    后续所有日志通过 USER_ACTION_LOG_TOPIC 异步落库                    │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. 数据验证阶段（同步执行）                                            │
│    ├─ onEmitUserActionLog("Starting data validation", Info)          │
│    ├─ validateNewResourcesData() — 新服务名称重复/数量限制检查        │
│    ├─ validateMovedEntitiesData() — 移动实体数据验证                  │
│    └─ onEmitUserActionLog("Data validation ended Successfully")      │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. 创建新服务（同步执行，循环）                                        │
│    对每个 newService：                                                │
│    ├─ 创建 Resource (Type=Service)                                    │
│    ├─ 安装默认 DB 插件                                                │
│    └─ onEmitUserActionLog("Successfully created service X")          │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 6. 按目标资源分组移动实体                                             │
│    对每个 targetResource：                                            │
│    ├─ onEmitUserActionLog("starting to move entities to resource X") │
│    └─ 为每个实体准备 createBulkData（实体 + 所有字段）                 │
│       每个字段处理都发射日志：                                         │
│       "Preparing data to move field X to entity Y"                   │
│       "moving relation field X ... related field in same resource"   │
│       "moving relation field X ... related entity in different res." │
│    └─ entityService.createBulkEntitiesAndFields(createBulkData, ctx) │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 7. 从原服务删除已移动实体                                             │
│    对每个 movedEntity：                                               │
│    ├─ onEmitUserActionLog("deleting entity X from source service")   │
│    └─ entityService.deleteOneEntity()                                │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 8. 完成标记（两条日志）                                                │
│    成功路径：                                                          │
│    ├─ onEmitUserActionLog("Successfully moved all entities...")      │
│    └─ onEmitUserActionLog(..., Info, Success, isCompleted=true)      │
│                                                                       │
│    失败路径（catch 块）：                                              │
│    ├─ onEmitUserActionLog(error.message, Error)                      │
│    └─ onEmitUserActionLog("Failed to move entities...",              │
│                           Error, Failed, isCompleted=true)           │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 9. 同步返回 userAction（前端开始轮询 status）                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 初始 Step 数据

定义于 [resource/constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/resource/constants.ts#L14-L26)

```typescript
export const REDESIGN_PROJECT_INITIAL_STEP_DATA = {
  name: APPLYING_PROJECT_REDESIGN_CHANGES,
  message: "Applying project redesign changes",
  status: EnumActionStepStatus.Running,
  logs: {
    create: [
      { message: "Starting to apply project redesign changes", level: Info, meta: {} },
      { message: `Moving ${movedEntities.length} entities to ${newServices.length} new services`, level: Info },
    ],
  },
};
```

### 6.3 三层关联总结（ProjectRedesign 场景）

| 层级 | 如何关联 | 内容 |
|------|---------|------|
| **UserAction.userId** | `createUserActionByTypeWithInitialStep(user.id)` | 操作发起人 |
| **UserAction.resourceId** | 传入 `movedEntities[0].originalResourceId` | 原始被重构的服务 |
| **ActionStep** | 通过 `actionContext = createActionContext(userAction.id, step, ...)` 绑定 | `APPLYING_PROJECT_REDESIGN_CHANGES` |
| **ActionLog** | `actionContext.onEmitUserActionLog(message, level, status?, isCompleted?)` → Kafka → 落库 | 数据验证、创建服务、移动实体/字段、删除实体等过程日志 |

---

## 7. 异步任务记录与 Kafka 事件流

### 7.1 完整 Kafka 主题分类

| 主题名称 | 常量名 | 用途 | 归属体系 |
|----------|--------|------|---------|
| `"user-action.internal.1"` | `USER_ACTION_TOPIC` | 用户账户通知（Novu 订阅管理） | **A. 用户通知事件**（不入 UserAction 表） |
| `"user-build.internal.1"` | `USER_BUILD_TOPIC` | 构建成功用户通知 | **A. 用户通知事件**（不入 UserAction 表） |
| `"user-action.internal.action-log.1"` | `USER_ACTION_LOG_TOPIC` | UserAction 业务操作日志落库 | **B. UserAction 日志流** |
| `"user-action.internal.db-schema-import.request.1"` | `DB_SCHEMA_IMPORT_TOPIC` | DB Schema 导入任务调度 | **B. UserAction 业务流** |
| `"ai.internal.conversation.start.1"` | `AI_CONVERSATION_START_TOPIC` | AI 对话启动 | **B. UserAction 业务流** |
| `"ai.internal.conversation.completed.1"` | `AI_CONVERSATION_COMPLETED_TOPIC` | AI 对话完成回调 | **B. UserAction 业务流** |
| `"build.internal.dsg-log.1"` | `DSG_LOG_TOPIC` | 代码生成日志落库 | **C. Build 日志流**（仅 Action 层） |
| `"build.internal.code-generation.success.1"` | `CODE_GENERATION_SUCCESS_TOPIC` | 代码生成成功 | **C. Build 业务流** |
| `"build.internal.code-generation.failure.1"` | `CODE_GENERATION_FAILURE_TOPIC` | 代码生成失败 | **C. Build 业务流** |
| `"git.internal.create-pr.log.1"` | `CREATE_PR_LOG_TOPIC` | PR 创建日志 | **C. Build 日志流** |
| `"git.internal.download-private-plugins.log.0"` | `DOWNLOAD_PRIVATE_PLUGINS_LOG_TOPIC` | 私有插件下载日志 | **C. Build 日志流** |

### 7.2 ActionContext：UserAction 的异步日志发射机制

`ActionContext` 是异步任务与审计日志系统之间的桥梁，定义于 [userAction/types.ts#ActionContext](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/types.ts#L33-L40)：

```typescript
export type ActionContext = {
  onEmitUserActionLog: (
    message: string,
    level: EnumActionLogLevel,
    status?: EnumActionStepStatus,
    isStepCompleted?: boolean
  ) => Promise<void>;
};
```

创建方式见 [action.service.ts#createActionContext](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/action.service.ts#L193-L243)，其核心机制：

1. 将日志消息封装为 Kafka 消息（包含 `userActionId`、`stepId`）
2. 通过 `kafkaProducerService.emitMessage` 发送到 `USER_ACTION_LOG_TOPIC`
3. **Fire-and-forget**：调用方使用 `void` 关键字，无需等待异步完成

### 7.3 UserActionLog Kafka 消息结构

Value Schema 定义于 [user-action-log/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-action-log/value.ts)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `stepId` | String | 目标步骤 ID |
| `level` | String | 日志级别：`error`/`warning`/`info`/`debug` |
| `message` | String | 日志内容 |
| `status` | EnumActionStepStatus | 当前步骤状态 |
| `isCompleted` | Boolean | 是否标记该步骤完成 |

### 7.4 消费端：UserAction 日志持久化

Kafka 消费者位于 [action.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/action.controller.ts)：

```typescript
@EventPattern(KAFKA_TOPICS.USER_ACTION_LOG_TOPIC)
async onUserActionLog(@Payload() message: UserActionLog.Value): Promise<void> {
  const logEntry = plainToInstance(UserActionLog.Value, message);
  await this.actionService.onUserActionLog(logEntry);
}
```

处理逻辑见 [action.service.ts#onUserActionLog](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/action.service.ts#L152-L164)：

1. 调用 `logByStepId` 将日志写入 `ActionLog` 表
2. 如果 `isCompleted === true`，调用 `updateActionStepStatus` 更新 `ActionStep` 的 `status` 和 `completedAt`

---

## 8. 仅复用 Action 层的异步任务（不经过 UserAction）

### 8.1 哪些任务不创建 UserAction

| 任务 | 关联方式 | 实际状态 | 日志 Kafka 主题 | 消费落库入口 |
|------|---------|---------|----------------|-------------|
| **Build（代码构建）** | `Build.actionId` 直接关联 Action | ✅ 完整实现，实际写 Step/Log | `DSG_LOG_TOPIC`、`CREATE_PR_LOG_TOPIC`、`DOWNLOAD_PRIVATE_PLUGINS_LOG_TOPIC` | [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.controller.ts) |
| **Deployment（部署）** | `Deployment.actionId` 直接关联 Action（仅 schema 定义） | ❌ 无实际执行代码，无任何 Step/Log 写入 | 无 | 无 |

### 8.2 Build 的日志链路详解

Build 创建时内联 Action：[build.service.ts#create](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.service.ts#L299-L305)

```typescript
action: {
  create: {
    steps: { create: createInitialStepData(version, args.data.message) },
  },
}
```

初始 Step：`ADD_TO_QUEUE`（Success，立即完成）

后续步骤由 `actionService.run()` 同步创建执行，或由 Kafka 异步消息驱动更新：

#### 同步步骤（actionService.run）
- `DOWNLOAD_PRIVATE_PLUGINS_STEP_NAME`：下载私有插件
- 内部通过 `step => { ... }` 执行，异常自动捕获并写入 ActionLog

#### 异步步骤（Kafka 驱动）
- `GENERATE_STEP_NAME`（`GENERATE_APPLICATION`）：代码生成
  - 日志：`DSG_LOG_TOPIC` → [build.controller.ts#onDsgLog](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.controller.ts#L141-L145)
  - 成功：`CODE_GENERATION_SUCCESS_TOPIC` → [build.service.ts#onCodeGenerationSuccess](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.service.ts#L450-L490) → `actionService.complete(step, Success)`
  - 失败：`CODE_GENERATION_FAILURE_TOPIC` → [build.service.ts#onCodeGenerationFailure](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.service.ts#L511-L534) → 写 Error 日志 + `actionService.complete(step, Failed)`

- `PUSH_TO_GIT_PROVIDER`：推送 Git
  - 日志：`CREATE_PR_LOG_TOPIC` → `build.service.onCreatePullRequestLog()`
  - 成功/失败：`CREATE_PR_SUCCESS_TOPIC` / `CREATE_PR_FAILURE_TOPIC`

Build 日志落库方法见 [build.service.ts#onDsgLog](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.service.ts#L970-L1010)：

```typescript
public async onDsgLog(logEntry: CodeGenerationLog.Value): Promise<void> {
  const step = await this.getBuildStep(logEntry.buildId, GENERATE_STEP_NAME);
  await this.actionService.logByStepId(step.id, ACTION_LOG_LEVEL[logEntry.level], logEntry.message);
  // Error 级别额外上报 Segment Analytics
}
```

### 8.3 Deployment — 仅 Prisma schema 定义，无实际执行代码

**结论**：Deployment 在当前代码库中是一个**预留结构**，仅存在 Prisma schema 定义，没有任何实际的创建、执行或写 Action/ActionStep/ActionLog 的业务代码。

#### 8.3.1 Prisma schema 定义

定义于 [schema.prisma#Deployment](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L629-L644)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | CUID 主键 |
| `createdAt` | DateTime | 创建时间 |
| `userId` | String | 关联创建用户（createdBy） |
| `buildId` | String | 关联 Build |
| `environmentId` | String | 关联 Environment |
| `status` | EnumDeploymentStatus | 部署状态 |
| `message` | String? | 状态信息 |
| `actionId` | String | **关联 Action（预留字段，从未被写入）** |
| `statusQuery` | Json? | 状态查询结果 |
| `statusUpdatedAt` | DateTime? | 状态更新时间 |

虽然定义了 `actionId → Action` 外键关联，但该字段从未被任何业务代码填充。

#### 8.3.2 代码中对 Deployment 的实际使用（逐条核实）

全局搜索 `deployment`（不区分大小写）仅命中 **5 个 TS 文件**，逐一核实：

| 文件 | 使用方式 | 是否写 Step/Log |
|------|---------|----------------|
| [AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts#L18-L20) | 枚举常量 `DeploymentId`，用于权限校验的 origin 参数类型 | ❌ |
| [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L282) | 权限校验函数，仅执行 `prisma.deployment.count()` 查询 Deployment 是否属于指定 workspace | ❌ 仅查询，无写入 |
| [mail.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/mail/mail.service.ts#L55-L82) | `sendDeploymentNotification()` 方法，用于发送部署成功/失败邮件 | ❌ 不涉及数据库/Action；且被 `IS_EMAIL_DEPLOYMENT_NOTIFICATION = false` 永久禁用 |
| [SendDeploymentArgs.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/mail/dto/SendDeploymentArgs.ts) | 邮件通知 DTO（to/success/url） | ❌ |
| [papermark.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/dbSchemaImport/predefinedSchemes/papermark/papermark.ts#L158) | 预定义示例 Schema 中的字段名 `deployment` | ❌ 仅字符串数据 |

关键证据：全局搜索 `prisma.deployment.(create|update|delete)` **无任何匹配**，说明 Deployment 表从未被创建或修改。

#### 8.3.3 Deployment vs Build 对比

| 维度 | Build（实际实现） | Deployment（预留结构） |
|------|------------------|----------------------|
| 独立 Service 模块 | ✅ 有完整 `core/build/` 目录 | ❌ 无任何 `core/deployment/` 目录 |
| GraphQL Resolver/Controller | ✅ BuildResolver + BuildController（Kafka） | ❌ 无 |
| Prisma create 调用 | ✅ build.service.ts 中大量 `prisma.build.create(...)` | ❌ 全局搜索无 `prisma.deployment.create(...)` |
| ActionStep 写入 | ✅ 创建时内联 step，后续 Kafka 持续更新 | ❌ 无任何代码写入 |
| ActionLog 写入 | ✅ `actionService.logByStepId()` / `actionService.run()` | ❌ 无任何代码写入 |
| Kafka 主题 | ✅ 7+ 个独立主题（DSG_LOG、CODE_GENERATION 等） | ❌ 无 |
| 前端可见 | ✅ 有 GraphQL query + 状态追踪 | ❌ 无 |

### 8.4 与 UserAction 体系的对比

| 维度 | UserAction 体系 | Build（仅 Action） |
|------|----------------|-------------------|
| **创建入口** | `userActionService.createUserActionByTypeWithInitialStep()` | Build 创建时内联 `action: { create: {...} }` |
| **用户关联** | UserAction.userId | Build.userId（自有字段） |
| **资源关联** | UserAction.resourceId | Build.resourceId（自有字段） |
| **日志发射** | `ActionContext.onEmitUserActionLog()` → Kafka | `actionService.logByStepId(stepId, ...)` 直接写库 |
| **日志主题** | `USER_ACTION_LOG_TOPIC` | `DSG_LOG_TOPIC` / `CREATE_PR_LOG_TOPIC` 等 |
| **日志消费** | `action.controller.ts#onUserActionLog` | `build.controller.ts#onDsgLog` 等 |
| **步骤完成** | Kafka 消息带 `isCompleted=true` 触发 | 直接调用 `actionService.complete(step, status)` |
| **前端查询** | `userAction()` query + 轮询 Hook | Build 自有状态 + `action { steps { logs } }` 嵌套 |
| **用户通知** | 无独立 Topic | `USER_BUILD_TOPIC`（独立 Kafka 事件） |

---

## 9. 典型场景完整流程

### 场景一：DB Schema Import（数据库 Schema 导入）

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 用户触发 import (GraphQL mutation)                           │
│    dbSchemaImportService.startProcessingDBSchema()              │
│    └─► 创建 UserAction + Action + ActionStep                    │
│    └─► metadata 存入 schema 文件内容                            │
│    └─► 关联 userId, resourceId                                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 发送 Kafka 消息到 DB_SCHEMA_IMPORT_TOPIC                     │
│    value: { actionId, file: schema内容 }                        │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 消费者处理 (dbSchemaImportService.createEntitiesFromPrismaSchema)│
│    └─► 根据 actionId 查询 UserAction                            │
│    └─► 取出 userId, resourceId, metadata                        │
│    └─► 创建 ActionContext                                       │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 实体创建过程中持续发射日志                                    │
│    void onEmitUserActionLog("Entity X created", Info)           │
│    └─► 发送 Kafka 消息到 USER_ACTION_LOG_TOPIC                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. ActionController 消费日志                                     │
│    └─► 每条日志写入 ActionLog 表                                │
│    └─► 最后一条标记 isCompleted=true, 更新 ActionStep 状态       │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 前端轮询 (useUserActionWatchStatus)                          │
│    └─► 每 2s 轮询 userAction query                              │
│    └─► status 变为 Completed/Failed 时停止轮询                 │
└─────────────────────────────────────────────────────────────────┘
```

### 场景二：GPT AI 对话（架构重构建议 / Break the Monolith）

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. resourceBtmService.triggerBreakServiceIntoMicroservices()    │
│    └─► gptService.startConversation()                           │
│        └─► 创建 UserAction (type=GptConversation)               │
│        └─► 创建 ActionStep: START_CONVERSATION (Waiting)        │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 发送 Kafka 消息到 AI_CONVERSATION_START_TOPIC                │
│    key: { requestUniqueId: userAction.id }                      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. AI Gateway 异步处理后回调                                     │
│    gptService.onConversationCompleted()                         │
│    └─► updateUserActionStep() 设置步骤 Success/Failed           │
│    └─► updateUserActionMetadata() 存入 AI 返回结果              │
└─────────────────────────────────────────────────────────────────┘
```

### 场景三：资源重构 Apply（ProjectRedesign，同步执行）

详见第 6 节完整流程图。

### 场景四：代码构建 Build（仅 Action，不经过 UserAction）

详见第 8.2 节。

---

## 10. 前端消费方式

### 10.1 UserAction GraphQL 查询

前端查询定义于 [UserAction/queries.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-client/src/UserAction/queries.ts)：

```graphql
query userAction($userActionId: String!) {
  userAction(where: { id: $userActionId }) {
    id
    createdAt
    status            # 动态解析：Running/Completed/Failed
    action {
      steps {
        id
        name
        status
        message
        completedAt
        logs {         # 嵌套获取所有日志
          message
          level
          meta
        }
      }
    }
  }
}
```

### 10.2 轮询 Hook

[useUserActionWatchStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-client/src/UserAction/useUserActionWatchStatus.ts)

- 轮询间隔：**2000ms**
- 停止条件：`status !== Running`
- 轮询停止时将 `userAction.id` 加入 AppContext 已完成列表

---

## 11. 状态汇总与评估

UserAction 的最终状态由其所有 ActionStep 的状态聚合计算得出，见 [userAction.service.ts#evalUserActionStatus](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts#L69-L96)：

| 步骤状态 | UserAction 最终状态 |
|---------|-------------------|
| 所有步骤 Success | Completed |
| 任一步骤 Failed | Failed |
| 否则（至少一个 Running/Waiting，无 Failed） | Running |
| 无步骤 | Invalid |

---

## 12. 关键设计模式总结

| 模式 | 实现位置 | 说明 |
|------|---------|------|
| **四阶层进记录** | UserAction → Action → ActionStep → ActionLog | 粗到细完整追溯链 |
| **Action 层复用** | Build/UserAction 实际使用；Deployment 仅 schema 定义 | 同一套 Step/Log 机制服务多种场景（Deployment 预留但未启用） |
| **Fire-and-Forget 日志** | `void onEmitUserActionLog(...)` | 不阻塞业务主流程 |
| **双通道 Kafka** | 通知事件（USER_ACTION_TOPIC/USER_BUILD_TOPIC）vs 日志落库（USER_ACTION_LOG_TOPIC/DSG_LOG_TOPIC） | 职责分离 |
| **Schema Registry** | `@amplication/schema-registry` | 集中管理所有 Kafka 消息的 key/value schema |
| **GraphQL ResolveField** | UserActionResolver 中动态解析 user/resource/action/status | 按需加载关联数据 |
| **同步+异步混合** | ProjectRedesign 同步执行 + DB Schema Import 异步调度 | 按业务特性灵活选择 |

---

## 13. 涉及的关键文件索引

| 用途 | 文件 |
|------|------|
| UserAction 服务 | [userAction.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts) |
| Action 服务（核心日志/步骤逻辑） | [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/action.service.ts) |
| Action Kafka 消费者（UserAction 日志落库） | [action.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/action.controller.ts) |
| Build Kafka 消费者（Build 日志落库） | [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.controller.ts) |
| UserAction GraphQL Resolver | [userAction.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.resolver.ts) |
| 类型与枚举定义 | [userAction/types.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/types.ts) |
| 资源重构 Apply 实现 | [resource.service.ts#redesignProject](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L743-L1147) |
| 资源重构常量（初始 Step 数据） | [resource/constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/resource/constants.ts) |
| Kafka 主题与 Schema | [schema-registry/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/index.ts) |
| UserAction Kafka 通知 Schema（USER_ACTION_TOPIC） | [user-action/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-action/value.ts) |
| UserActionLog Kafka Schema（USER_ACTION_LOG_TOPIC） | [user-action-log/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-action-log/value.ts) |
| Build 用户通知 Schema（USER_BUILD_TOPIC） | [user-build/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-build/value.ts) |
| 数据库 Prisma Schema | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma) |
| DB Schema Import 实现（样例） | [dbSchemaImport.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts) |
| GPT AI 对话实现（样例） | [gpt.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/gpt/gpt.service.ts) |
| Build 实现（样例） | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.service.ts) |
| Deployment Prisma Schema（仅定义，无实际代码） | [schema.prisma#Deployment](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L629-L644) |
| Deployment 权限校验（仅查询，无写入） | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L282) |
| Deployment 邮件通知（已永久禁用） | [mail.service.ts#sendDeploymentNotification](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/mail/mail.service.ts#L55-L82) |
| 用户服务（USER_ACTION_TOPIC 发布） | [user.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/user/user.service.ts) |
| 通知服务总入口 | [notification-service/app.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/app.controller.ts) |
| 通知服务用户订阅处理 | [subscribeUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/notification-packages/subscribeUser.ts) |
| 通知服务构建完成通知 | [buildCompleted.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/notification-packages/buildCompleted.ts) |
| 前端轮询 Hook | [useUserActionWatchStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-client/src/UserAction/useUserActionWatchStatus.ts) |
