# Activity Log 与审计线索串联流程分析

## 1. 整体架构概览

Amplication 的 Activity Log 与审计线索通过 **四层嵌套数据模型 + Kafka 异步事件总线** 实现，将用户操作、资源变更与异步任务执行过程完整串联。

```
UserAction (用户操作层)
    └── Action (动作容器层)
          └── ActionStep (执行步骤层)
                └── ActionLog (详细日志层)
```

### 核心模块位置

| 模块 | 文件路径 |
|------|---------|
| UserAction 模块 | [userAction](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction) |
| Action 模块 | [action](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action) |
| Schema 注册中心 | [schema-registry](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry) |
| Kafka 主题定义 | [index.ts#KAFKA_TOPICS](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/index.ts#L29-L69) |

---

## 2. 数据库模型关系

### 2.1 ER 图

```
UserAction ──┐
             ├──► User (userId)
             ├──► Resource (resourceId)
             └──► Action ──► ActionStep ──► ActionLog
Build ───────┘
Deployment ──┘
```

### 2.2 各层模型详解

#### UserAction - 用户操作层
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

#### Action - 动作容器层
定义于 [schema.prisma#Action](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L452-L459)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | CUID 主键 |
| `steps` | ActionStep[] | 包含的执行步骤 |

**关键关联**：`UserAction`、`Build`、`Deployment` 均通过 `actionId` 外键关联到 Action。

#### ActionStep - 执行步骤层
定义于 [schema.prisma#ActionStep](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L461-L473)

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | CUID 主键 |
| `name` | String | 步骤名称（如 `PROCESSING_PRISMA_SCHEMA`、`START_CONVERSATION`） |
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

## 3. 用户上下文关联机制

### 3.1 发起用户关联

每个 `UserAction` 都通过 `userId` 外键直接关联到 `User` 表，确保每一条活动记录都能追溯到具体的操作人。

核心创建逻辑位于 [userAction.service.ts#createUserActionByTypeWithInitialStep](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts#L20-L62)：

```typescript
const data: Prisma.UserActionCreateInput = {
  userActionType,
  metadata,
  user: {
    connect: { id: userId },  // 关联发起用户
  },
  action: {
    create: {
      steps: { create: initialStepData },
    },
  },
};

if (resourceId) {
  data.resource = {
    connect: { id: resourceId },  // 关联目标资源
  };
}
```

### 3.2 GraphQL 解析用户上下文

在 [userAction.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.resolver.ts#L51-L57) 中，通过 `@ResolveField` 动态解析关联的用户和资源信息：

```typescript
@ResolveField()
async user(@Parent() userAction: UserAction): Promise<User> {
  return this.userService.findUser(
    { where: { id: userAction.userId } },
    true
  );
}
```

### 3.3 用户审计事件（UserAction Kafka Topic）

除了数据库记录，还通过 Kafka `USER_ACTION_TOPIC` 发布用户级别的审计事件，用于通知服务、订阅管理等下游消费。

定义于 [schema-registry user-action/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-action/value.ts)

| 字段 | 说明 |
|------|------|
| `userId` | 用户账户 ID |
| `email` / `firstName` / `lastName` | 用户基本信息 |
| `externalId` | 外部系统 ID（如 Novu 通知系统） |
| `action` | 操作类型：`SIGNUP` / `LOGIN` / `UPDATE` / `DELETE` / `CURRENT_WORKSPACE` |
| `enableUser` | 是否启用通知 |

发布示例：[user.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/user/user.service.ts#L184-L199)
```typescript
this.kafkaProducerService
  .emitMessage(KAFKA_TOPICS.USER_ACTION_TOPIC, {
    key: {},
    value: {
      userId: user.account.id,
      email: user.account.email,
      action: UserAction.UserActionType.CURRENT_WORKSPACE,
      // ...
    },
  });
```

---

## 4. 资源变更记录方式

### 4.1 资源关联

`UserAction.resourceId` 外键直接关联到目标 `Resource`，实现变更记录与资源的绑定。

Resolver 中动态解析：[userAction.resolver.ts#L44-L49](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.resolver.ts#L44-L49)

### 4.2 业务元数据 metadata

`UserAction.metadata`（Json 类型）存储业务相关的具体变更内容：

| 场景 | metadata 内容示例 |
|------|-------------------|
| DBSchemaImport | `{ schema: "...", fileName: "schema.prisma" }` |
| GptConversation | `{ data: "<GPT response JSON>" }` |
| ProjectRedesign/BTM | 服务拆分建议结果 |

更新 metadata 的方法：[userAction.service.ts#updateUserActionMetadata](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts#L131-L143)

### 4.3 实际场景：DB Schema Import

完整流程见 [dbSchemaImport.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts)：

1. **启动时**：创建 UserAction，将 Prisma schema 文件内容存入 metadata
2. **消费 Kafka 消息**：从 UserAction 中取出 metadata 和 resourceId
3. **创建实体过程**：通过 `ActionContext` 持续输出日志
4. **完成时**：通过 `onEmitUserActionLog` 标记步骤成功/失败

---

## 5. 异步任务记录与 Kafka 事件流

### 5.1 核心 Kafka 主题

| 主题名称 | 常量位置 | 用途 |
|----------|---------|------|
| `USER_ACTION_TOPIC` | `"user-action.internal.1"` | 用户级审计事件（注册、登录、切换工作区等） |
| `USER_ACTION_LOG_TOPIC` | `"user-action.internal.action-log.1"` | 异步任务执行过程中的日志和状态更新 |
| `DB_SCHEMA_IMPORT_TOPIC` | `"user-action.internal.db-schema-import.request.1"` | 数据库 Schema 导入任务 |
| `AI_CONVERSATION_START_TOPIC` | `"ai.internal.conversation.start.1"` | AI 对话启动 |
| `AI_CONVERSATION_COMPLETED_TOPIC` | `"ai.internal.conversation.completed.1"` | AI 对话完成 |

完整主题列表见 [schema-registry/index.ts#KAFKA_TOPICS](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/index.ts#L29-L69)

### 5.2 ActionContext：异步日志发射机制

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

### 5.3 UserActionLog Kafka 消息结构

Value Schema 定义于 [user-action-log/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-action-log/value.ts)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `stepId` | String | 目标步骤 ID |
| `level` | String | 日志级别：`error`/`warning`/`info`/`debug` |
| `message` | String | 日志内容 |
| `status` | EnumActionStepStatus | 当前步骤状态 |
| `isCompleted` | Boolean | 是否标记该步骤完成 |

### 5.4 消费端：日志持久化

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

## 6. 典型场景完整流程

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

### 场景二：GPT AI 对话（架构重构/Break the Monolith）

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

### 场景三：代码构建 Build（同步 Action + Kafka）

不同于上述 UserAction 流程，Build 直接关联 Action（不经过 UserAction），但复用同一套 Step/Log 机制：

1. [build.service.ts#create()](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L352) 创建 Build，内联创建 Action + 初始 Step
2. 使用 `actionService.run()` 执行步骤（如 `downloadPrivatePlugins`），自动管理步骤状态和异常日志
3. 发送 Kafka 消息到代码生成相关 Topic（`CODE_GENERATION_REQUEST_TOPIC` 等）
4. 异步接收 DSG_LOG_TOPIC 等日志，通过 build controller 消费落库

---

## 7. 前端消费方式

### 7.1 GraphQL 查询

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

### 7.2 轮询 Hook

[useUserActionWatchStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-client/src/UserAction/useUserActionWatchStatus.ts)

- 轮询间隔：**2000ms**
- 停止条件：`status !== Running`
- 轮询停止时将 `userAction.id` 加入 AppContext 已完成列表

---

## 8. 状态汇总与评估

UserAction 的最终状态由其所有 ActionStep 的状态聚合计算得出，见 [userAction.service.ts#evalUserActionStatus](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts#L69-L96)：

| 步骤状态 | UserAction 最终状态 |
|---------|-------------------|
| 所有步骤 Success | Completed |
| 任一步骤 Failed | Failed |
| 否则（至少一个 Running/Waiting，无 Failed） | Running |
| 无步骤 | Invalid |

---

## 9. 关键设计模式总结

| 模式 | 实现位置 | 说明 |
|------|---------|------|
| **四阶层进记录** | UserAction → Action → ActionStep → ActionLog | 粗到细完整追溯链 |
| **Fire-and-Forget 日志** | `void onEmitUserActionLog(...)` | 不阻塞业务主流程 |
| **Kafka 事件驱动** | USER_ACTION_LOG_TOPIC 异步解耦 | 日志写入与业务处理分离 |
| **Schema Registry** | `@amplication/schema-registry` | 集中管理所有 Kafka 消息的 key/value schema |
| **GraphQL ResolveField** | UserActionResolver 中动态解析 user/resource/action/status | 按需加载关联数据 |
| **泛化 Action 复用** | Build/Deployment/UserAction 均关联 Action | 同一套 Step/Log 机制服务多种场景 |

---

## 10. 涉及的关键文件索引

| 用途 | 文件 |
|------|------|
| UserAction 服务 | [userAction.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts) |
| Action 服务（核心日志/步骤逻辑） | [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/action.service.ts) |
| Action Kafka 消费者 | [action.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/action/action.controller.ts) |
| UserAction GraphQL Resolver | [userAction.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/userAction.resolver.ts) |
| 类型与枚举定义 | [userAction/types.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/userAction/types.ts) |
| Kafka 主题与 Schema | [schema-registry/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/index.ts) |
| UserActionLog Kafka Schema | [user-action-log/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/libs/schema-registry/src/lib/user-action-log/value.ts) |
| 数据库 Prisma Schema | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-prisma-db/prisma/schema.prisma) |
| DB Schema Import 实现（样例） | [dbSchemaImport.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts) |
| GPT AI 对话实现（样例） | [gpt.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/gpt/gpt.service.ts) |
| Build 实现（样例） | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-server/src/core/build/build.service.ts) |
| 前端轮询 Hook | [useUserActionWatchStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/amplication-client/src/UserAction/useUserActionWatchStatus.ts) |
| 通知服务用户订阅 | [subscribeUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/45-amplication/packages/notification-service/src/notification-packages/subscribeUser.ts) |
