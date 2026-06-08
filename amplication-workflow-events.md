# Amplication Workflow 事件系统协作机制

## 1. 整体架构概述

Amplication 采用基于 **Kafka 消息队列** 的事件驱动架构，实现各微服务之间的解耦协作。系统由以下核心层次组成：

```
┌─────────────────────────────────────────────────────────────────┐
│                        触发源 (Triggers)                         │
│  GraphQL Resolver / REST Controller / 定时任务 / 外部事件        │
└────────────────────────────┬────────────────────────────────────┘
                             │ 业务方法调用
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    业务服务层 (Business Services)                 │
│  BuildService / UserActionService / GptService 等                │
│  - 执行业务逻辑                                                   │
│  - 通过 KafkaProducerService.emitMessage() 发布事件              │
└────────────────────────────┬────────────────────────────────────┘
                             │ emitMessage(topic, event)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Schema Registry (事件契约层)                   │
│  - 定义所有事件的 Key / Value 数据结构 (class-validator)          │
│  - 统一维护 KAFKA_TOPICS 枚举 (所有主题名)                        │
│  - 每个事件类型独立目录 (key.ts / value.ts / index.ts)            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Kafka 传输层 (NestJS Microservices)            │
│  - KafkaProducerService: 序列化 + 发送消息                        │
│  - KafkaCustomTransport: 扩展 ServerKafka，支持正则匹配主题       │
│  - createNestjsKafkaConfig: 统一 Kafka 客户端/消费者配置          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                 订阅消费层 (Event Consumers)                      │
│  @EventPattern / @MessagePattern 装饰器标注的 Controller 方法     │
│  - amplication-server (BuildController 等)                        │
│  - amplication-build-manager (BuildRunnerController)              │
│  - notification-service (AppController)                           │
│  - gpt-gateway (KafkaController)                                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  执行层 (Execution Pipeline)                      │
│  - ActionService.run(): 步骤化执行 + 状态管理                     │
│  - NovuService: 通知触发                                          │
│  - 各业务 Service: 领域逻辑执行                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 事件契约层：Schema Registry

所有事件的统一规范定义在 `@amplication/schema-registry` 包中。

### 2.1 位置与结构

- **模块路径**: [libs/schema-registry](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/schema-registry)
- **入口文件**: [index.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/schema-registry/src/index.ts)

每个事件类型是一个独立目录，包含三个文件：

```
lib/
├── code-generation-request/
│   ├── index.ts      # 导出 KafkaEvent 接口（extends DecodedKafkaMessage）
│   ├── key.ts        # 消息 Key 的 DTO（class-validator 装饰器）
│   └── value.ts      # 消息 Value 的 DTO（class-validator 装饰器）
├── code-generation-success/
├── user-build/
├── create-pr-request/
└── ... (约 25+ 种事件类型)
```

### 2.2 事件接口定义示例

以 `CodeGenerationRequest` 为例，见 [code-generation-request/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/schema-registry/src/lib/code-generation-request/index.ts)：

```typescript
import { DecodedKafkaMessage } from "@amplication/util/kafka";
import { Key } from "./key";
import { Value } from "./value";

interface KafkaEvent extends DecodedKafkaMessage {
  key: Key;
  value: Value;
}

export { Key, Value, KafkaEvent };
```

其中 `Value` DTO 定义了事件负载结构，见 [code-generation-request/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/schema-registry/src/lib/code-generation-request/value.ts)：

```typescript
export class Value {
  @IsString() buildId!: string;
  @IsString() resourceId!: string;
}
```

### 2.3 Kafka 主题枚举

所有 Kafka 主题统一在 [KAFKA_TOPICS](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/schema-registry/src/index.ts#L29-L69) 枚举中维护：

| 主题分类 | 主题名示例 | 用途 |
|---------|-----------|------|
| Build 管理 | `CODE_GENERATION_REQUEST_TOPIC` | 触发代码生成 |
| | `CODE_GENERATION_SUCCESS_TOPIC` | 代码生成成功 |
| | `CODE_GENERATION_FAILURE_TOPIC` | 代码生成失败 |
| | `USER_BUILD_TOPIC` | 构建完成用户通知 |
| | `DSG_LOG_TOPIC` | 代码生成日志 |
| Git/PR | `CREATE_PR_REQUEST_TOPIC` | 发起 PR 创建 |
| | `CREATE_PR_SUCCESS_TOPIC` | PR 创建成功 |
| | `CREATE_PR_FAILURE_TOPIC` | PR 创建失败 |
| | `KAFKA_REPOSITORY_PUSH_QUEUE` | 外部 Git push 事件 |
| 插件 | `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` | 下载私有插件请求 |
| 用户/通知 | `USER_ACTION_TOPIC` | 用户操作事件 |
| | `USER_ANNOUNCEMENT_TOPIC` | 功能公告 |
| | `TECH_DEBT_CREATED_TOPIC` | 技术债务告警 |
| AI | `AI_CONVERSATION_START_TOPIC` | GPT 对话开始 |
| | `AI_CONVERSATION_COMPLETED_TOPIC` | GPT 对话完成 |
| 权限 | `CHECK_USER_ACCESS_TOPIC` | 用户构建权限校验（请求/响应模式） |

---

## 3. Kafka 传输层

### 3.1 Kafka 模块

Kafka 基础设施封装在 `@amplication/util/nestjs/kafka` 中。

- **模块定义**: [Kafka.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/util/nestjs/kafka/src/Kafka.module.ts)
- **配置工厂**: [createNestjsKafkaConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/util/nestjs/kafka/src/createNestjsKafkaConfig.ts)

模块导出两个核心服务：
- `KAFKA_SERIALIZER` (KafkaMessageJsonSerializer): 消息序列化
- `KafkaProducerService`: 事件发布服务

### 3.2 事件发布：KafkaProducerService

代码位置: [KafkaProducer.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts)

核心方法 `emitMessage()`：

```typescript
@Injectable()
export class KafkaProducerService {
  constructor(
    @Inject(KAFKA_CLIENT) private readonly kafkaClient: ClientKafka,
    @Inject(KAFKA_SERIALIZER) private readonly serializer: IKafkaMessageSerializer
  ) {}

  async emitMessage(
    topic: string,
    message: DecodedKafkaMessage,
    schemaIds?: SchemaIds
  ): Promise<void> {
    const kafkaMessage = await this.serializer.serialize(message, schemaIds);
    return await new Promise((resolve, reject) => {
      this.kafkaClient.emit(topic, kafkaMessage).subscribe({
        error: (err: Error) => reject(err),
        next: () => resolve(),
      });
    });
  }
}
```

**设计要点**：
- 使用 NestJS `ClientKafka` 的 `emit()` 方法（fire-and-forget 模式）
- 先通过序列化器将消息转为 Kafka 格式
- 将 RxJS Observable 转为 Promise 以便 async/await 调用

### 3.3 自定义 Kafka 传输层：KafkaCustomTransport

代码位置: [kafka.transport.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/util/nestjs/kafka/src/kafka.transport.ts)

扩展了 NestJS 原生 `ServerKafka`，增加了**正则表达式主题匹配**能力：

```typescript
export class KafkaCustomTransport extends ServerKafka {
  override async bindEvents(consumer: Consumer): Promise<void> {
    // 将形如 "/pattern/" 的 handler pattern 转为 RegExp 对象
    const registeredPatterns = [...this.messageHandlers.entries()].map(
      ([pattern, handler]) =>
        pattern.startsWith("/") && pattern.endsWith("/")
          ? new RegExp(pattern.slice(1, pattern.length - 2), handler.extras?.flags)
          : pattern
    );
    // 订阅所有匹配的主题
    await Promise.all(registeredPatterns.map(subscribeToPattern));
  }

  public override getHandlerByPattern(pattern: string) {
    return super.getHandlerByPattern(pattern) ?? this.getHandlerByRegExp(pattern);
  }
}
```

### 3.4 微服务启动方式

每个需要消费 Kafka 事件的服务，都在其 `main.ts` 中连接 Kafka 微服务：

以 [notification-service/src/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/notification-service/src/main.ts) 为例：

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });

  app.connectMicroservice<MicroserviceOptions>({
    strategy: new KafkaCustomTransport(createNestjsKafkaConfig().options),
  });

  await app.startAllMicroservices();
  await app.listen(port);
}
```

`amplication-server` 则使用默认配置（使用原生 `ServerKafka`），见 [amplication-server/src/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/main.ts#L41)：

```typescript
app.connectMicroservice<MicroserviceOptions>(createNestjsKafkaConfig());
```

---

## 4. 触发入口分析

### 4.1 构建触发：BuildService.create()

代码位置: [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L352)

这是最核心的工作流触发入口。完整调用链如下：

```
GraphQL Mutation (createBuild)
       │
       ▼
BuildResolver → BuildService.create()
       │
       ├─ 1. 在 DB 创建 Build 记录（status=Running）
       │     并创建关联的 Action + 初始 Step (ADD_TO_QUEUE)
       │
       ├─ 2. 判断资源类型是否为 Service / Component
       │
       ├─ 3. 检查是否有私有插件
       │     │
       │     ├─ 有 → downloadPrivatePlugins()
       │     │       └─ emit DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC
       │     │
       │     └─ 无 → generate()
       │
       └─ 4. generate() [见 4.1.1]
```

#### 4.1.1 generate() — 代码生成事件触发

代码位置: [build.service.ts#L568-L618](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/build/build.service.ts#L568-L618)

```typescript
private async generate(logger, build, user): Promise<string> {
  return this.actionService.run(
    build.actionId,
    GENERATE_STEP_NAME,          // "GENERATE_APPLICATION"
    GENERATE_STEP_MESSAGE,
    async (step) => {
      // 1. 收集资源数据 (entities, roles, plugins, modules...)
      const dsgResourceData = await this.getDSGResourceData(resource, buildId, buildVersion, user);

      // 2. 保存到共享存储 (本地文件系统)
      await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);

      // 3. 发布轻量级代码生成请求事件到 Kafka
      const codeGenerationEvent: CodeGenerationRequest.KafkaEvent = {
        key: null,
        value: { resourceId, buildId },
      };
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
        codeGenerationEvent
      );
    },
    true  // leaveStepOpenAfterSuccessfulExecution: 步骤保持 Running 状态
  );
}
```

**关键设计**：
- 使用 **ActionService.run()** 进行步骤化执行
- DSG 资源数据（可能很大）**不通过 Kafka 传输**，而是保存到共享文件系统
- Kafka 消息只携带 `buildId` 和 `resourceId` 两个轻量标识

#### 4.1.2 onCodeGenerationSuccess() — 构建成功后的通知触发

代码位置: [build.service.ts#L450-L495](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/build/build.service.ts#L450-L495)

```typescript
async onCodeGenerationSuccess(buildId: string): Promise<void> {
  // ... 获取 build 关联数据 ...

  // 发布用户构建通知事件
  this.kafkaProducerService.emitMessage(KAFKA_TOPICS.USER_BUILD_TOPIC, <UserBuild.KafkaEvent>{
    key: {},
    value: {
      commitId, commitMessage, resourceId, resourceName,
      workspaceId, projectId, buildId, projectName,
      createdAt: Date.now(),
      externalId: encryptString(userId),
      envBaseUrl: this.configService.get<string>(Env.CLIENT_HOST),
    },
  });

  await this.actionService.complete(step, EnumActionStepStatus.Success);
}
```

### 4.2 用户操作触发：UserActionService

代码位置: [userAction.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/userAction/userAction.service.ts)

```typescript
async createUserActionByTypeWithInitialStep(
  userActionType, metadata, initialStepData, userId, resourceId?
): Promise<UserAction> {
  return await this.prisma.userAction.create({
    data: {
      userActionType, metadata,
      user: { connect: { id: userId } },
      action: { create: { steps: { create: initialStepData } } },
      ...(resourceId ? { resource: { connect: { id: resourceId } } } : {}),
    },
    include: { action: { include: { steps: true } } },
  });
}
```

### 4.3 AI 对话触发：GptService

触发入口在 amplication-server 的 GptResolver，发布 `AI_CONVERSATION_START_TOPIC` 事件，由 `gpt-gateway` 服务消费。

---

## 5. 事件消费与订阅机制

### 5.1 NestJS 装饰器订阅模式

使用 `@EventPattern()` (fire-and-forget) 或 `@MessagePattern()` (request-response) 装饰器标注 Controller 方法。

#### 5.1.1 amplication-server：BuildController

代码位置: [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/build/build.controller.ts)

该 Controller 是构建工作流的**事件中枢**，消费所有与构建相关的 Kafka 事件：

| 装饰器 | 订阅主题 | 处理方法 | 说明 |
|--------|---------|---------|------|
| `@MessagePattern` | `CHECK_USER_ACCESS_TOPIC` | `checkUserAccess()` | 请求-响应模式，校验用户权限 |
| `@EventPattern` | `CODE_GENERATION_SUCCESS_TOPIC` | `onCodeGenerationSuccess()` | 代码生成成功 → 触发 PR 创建 + 用户通知 |
| `@EventPattern` | `CODE_GENERATION_FAILURE_TOPIC` | `onCodeGenerationFailure()` | 代码生成失败 → 标记步骤失败 |
| `@EventPattern` | `CREATE_PR_SUCCESS_TOPIC` | `onPullRequestCreated()` | PR 创建成功 → 标记构建完成 |
| `@EventPattern` | `CREATE_PR_FAILURE_TOPIC` | `onPullRequestFailure()` | PR 创建失败 → 标记构建失败 |
| `@EventPattern` | `DSG_LOG_TOPIC` | `onDsgLog()` | 代码生成日志 → 写入 ActionLog |
| `@EventPattern` | `DOWNLOAD_PRIVATE_PLUGINS_*` | 对应方法 | 私有插件下载生命周期事件 |

#### 5.1.2 amplication-build-manager：BuildRunnerController

代码位置: [build-runner.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts)

消费代码生成请求，实际调用 DSG (Data Service Generator) 执行代码生成：

```typescript
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC)
async onCodeGenerationRequest(
  @Payload() message: CodeGenerationRequest.Value
): Promise<void> {
  await this.buildRunnerService.runBuild(message.resourceId, message.buildId);
}
```

#### 5.1.3 notification-service：AppController

代码位置: [notification-service/src/app.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/notification-service/src/app.controller.ts)

消费用户通知相关的多个主题：

| 订阅主题 | 处理方法 |
|---------|---------|
| `user-action.internal.1` | `subscribeNotification()` |
| `user-build.internal.1` | `notifyBuild()` |
| `user-announcement.internal.1` | `notifyFeatureAnnouncement()` |
| `platform.internal.tech-debt.created.1` | `notifyTechDebt()` |

所有方法统一调用 `appService.notificationService(message, topic)`。

#### 5.1.4 gpt-gateway：KafkaController

代码位置: [kafka.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/gpt-gateway/src/kafka/kafka.controller.ts)

```typescript
@EventPattern(KAFKA_TOPICS.AI_CONVERSATION_START_TOPIC)
async onAiConversationStart_1(@Payload() value, @Ctx() context: KafkaContext) {
  this.conversationType.startConversion(messageInput);
}
```

### 5.2 消息校验：plainToInstance

每个消费者方法都使用 `class-transformer` 的 `plainToInstance()` 将 Kafka 原始消息转为 Schema Registry 定义的强类型 DTO，配合 `class-validator` 实现运行时校验：

```typescript
const args = plainToInstance(CodeGenerationSuccess.Value, message);
```

---

## 6. 订阅执行管道：Notification Pipeline

notification-service 使用**函数组合 (compose)** 模式构建通知处理管道。

### 6.1 管道定义：AppService.notificationService()

代码位置: [app.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/notification-service/src/app.service.ts)

```typescript
type NotificationPackageFunc = (ctx: NotificationContext) => Promise<typeof ctx> | Promise<void>;

const compose = (...fns: NotificationPackageFunc[]) =>
  (x: NotificationContext) => fns.reduce(async (y, f) => f(await y), Promise.resolve(x));

@Injectable()
export class AppService {
  async notificationService(message, topic) {
    return compose(
      subscribeUser,          // 1. 用户订阅管理（创建/删除 Novu subscriber）
      buildCompleted,         // 2. 构建完成通知
      featureAnnouncement,    // 3. 功能公告广播
      techDebtAlert,          // 4. 技术债务告警
      novuPackage             // 5. 最终执行：批量调用 Novu API 发送通知
    )({
      message, topic,
      novuService: this.novuService,
      amplicationLogger: this.logger,
      notifications: [],
    });
  }
}
```

### 6.2 中间件（Notification Package）示例

每个中间件检查 topic 是否匹配，匹配则向 `ctx.notifications` 数组添加通知任务，否则透传 context。

#### buildCompleted 示例

代码位置: [buildCompleted.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/notification-service/src/notification-packages/buildCompleted.ts)

```typescript
export const buildCompleted = async (notificationCtx: NotificationContext) => {
  if (!notificationCtx.message || notificationCtx.topic !== KAFKA_TOPICS.USER_BUILD_TOPIC)
    return notificationCtx;

  const { externalId, ...restParams } = notificationCtx.message;
  const shortBuildId = restParams?.buildId.slice(-8);

  notificationCtx.notifications.push({
    notificationMethod: notificationCtx.novuService.triggerNotificationToSubscriber,
    subscriberId: externalId,
    eventName: "build-completed",
    payload: {
      payload: { ...restParams, shortBuildId, createdAt: format(...) },
    },
  });

  return notificationCtx;
};
```

### 6.3 执行层：NovuService

代码位置: [novuService.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/notification-service/src/util/novuService.ts)

封装 Novu 通知平台 API：
- `createSubscriber()` / `deleteSubscriber()` - 订阅者管理
- `triggerNotificationToSubscriber()` - 触发单用户通知
- `broadCastEventToAll()` - 广播通知

---

## 7. 步骤化执行框架：ActionService

代码位置: [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/action/action.service.ts)

### 7.1 核心数据模型

```
Action (1) ────→ ActionStep (N) ────→ ActionLog (N)
  │                 │
  └─ build          └─ name (如 "GENERATE_APPLICATION")
  └─ userAction     └─ status: Running / Success / Failed
                     └─ message
                     └─ completedAt
```

### 7.2 run() 方法：步骤模板方法

```typescript
async run<T>(
  actionId: string,
  stepName: string,
  message: string,
  stepFunction: (step: ActionStep) => Promise<T>,
  leaveStepOpenAfterSuccessfulExecution = false
): Promise<T> {
  const step = await this.createStep(actionId, stepName, message);  // status = Running
  try {
    const result = await stepFunction(step);
    if (!leaveStepOpenAfterSuccessfulExecution) {
      await this.complete(step, EnumActionStepStatus.Success);
    }
    return result;
  } catch (error) {
    await this.log(step, EnumActionLogLevel.Error, error.message);
    await this.complete(step, EnumActionStepStatus.Failed);
    throw error;
  }
}
```

**设计意图**：
- 将异步操作的生命周期（创建步骤 → 执行 → 成功/失败）统一封装
- 对 Kafka 事件驱动的长流程，使用 `leaveStepOpenAfterSuccessfulExecution = true` 让步骤保持 Running，待后续消费事件回调时再标记完成

### 7.3 emitLog 通过 Kafka

`createActionContext()` 返回一个闭包上下文，用于在长流程中通过 Kafka 发送步骤日志和状态更新：

```typescript
createActionContext(userActionId, step, topicName): ActionContext {
  return {
    onEmitUserActionLog: async (message, level, status, isStepCompleted) => {
      const kafkaMessage = this.createKafkaMessageForUserActionLog(...)(...);
      return this.kafkaProducerService.emitMessage(topicName, kafkaMessage);
    }
  };
}
```

---

## 8. 完整工作流示例：构建代码生成 → 推送 Git → 通知用户

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 1. 用户在前端点击 "Commit & Build"                                            │
│    GraphQL: mutation createBuild(...)                                        │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 2. amplication-server - BuildService.create()                                │
│    ├─ 创建 Build + Action + Step(ADD_TO_QUEUE:Success)                       │
│    └─ 调用 generate()                                                        │
│         ├─ Step(GENERATE_APPLICATION:Running)                                │
│         ├─ 收集 DSGResourceData 写入共享存储                                  │
│         └─ emit CODE_GENERATION_REQUEST_TOPIC {buildId, resourceId}          │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │ Kafka
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 3. amplication-build-manager - BuildRunnerController                         │
│    @EventPattern(CODE_GENERATION_REQUEST_TOPIC)                              │
│    └─ buildRunnerService.runBuild() → 调用 DSG 生成代码                       │
│         ├─ 过程中多次 emit DSG_LOG_TOPIC (日志流式回写)                       │
│         ├─ 成功 → emit CODE_GENERATION_SUCCESS_TOPIC {buildId}               │
│         └─ 失败 → emit CODE_GENERATION_FAILURE_TOPIC {buildId, errorMessage} │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │ Kafka
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 4. amplication-server - BuildController                                      │
│    @EventPattern(CODE_GENERATION_SUCCESS_TOPIC)                              │
│    ├─ buildService.saveToGitProvider()                                       │
│    │   └─ Step(PUSH_TO_GIT_PROVIDER:Running)                                 │
│    │      └─ emit CREATE_PR_REQUEST_TOPIC {gitSettings, buildId...}          │
│    └─ buildService.onCodeGenerationSuccess()                                 │
│         ├─ emit USER_BUILD_TOPIC {buildId, externalId, resource...}         │
│         └─ Step(GENERATE_APPLICATION:Success)                                │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │
                    ┌──────────────┴───────────────┐
                    │ Kafka                        │ Kafka
                    ▼                              ▼
┌────────────────────────────────────┐  ┌──────────────────────────────────────┐
│ 5a. git-sync-manager               │  │ 5b. notification-service             │
│     消费 CREATE_PR_REQUEST_TOPIC   │  │     消费 USER_BUILD_TOPIC            │
│     调用 GitHub/GitLab API 创建 PR  │  │     compose 管道:                    │
│     成功 → emit CREATE_PR_SUCCESS  │  │       subscribeUser → buildCompleted  │
│     失败 → emit CREATE_PR_FAILURE  │  │       → novuPackage                  │
└──────────────┬─────────────────────┘  │         → Novu.trigger("build-       │
               │ Kafka                  │           completed", {subscriberId})│
               ▼                        └──────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────────┐
│ 6. amplication-server - BuildController                                      │
│    @EventPattern(CREATE_PR_SUCCESS_TOPIC)                                    │
│    └─ buildService.onCreatePRSuccess()                                       │
│         ├─ Step(PUSH_TO_GIT_PROVIDER:Success)                                │
│         └─ Build.status = Completed                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 关键协作模式总结

### 9.1 两种消息模式

| 模式 | 装饰器 | 返回值 | 使用场景 |
|------|--------|--------|---------|
| 发布订阅 (Pub/Sub) | `@EventPattern` | `void` | 代码生成请求、日志、通知 |
| 请求响应 (Req/Res) | `@MessagePattern` | `{value: ...}` | CHECK_USER_ACCESS_TOPIC 权限校验 |

### 9.2 事件编排：Saga 模式

构建工作流采用 **Saga 模式** 的事件驱动编排：
- 没有中央协调器，每个消费者处理完后发布下一个事件
- 步骤状态通过数据库 (ActionStep) 持久化，支持故障恢复
- 通过 `buildId` / `actionId` 作为 Correlation ID 关联所有事件

### 9.3 大对象传输策略

DSGResourceData（可能包含实体、模块、插件等大量数据）**不通过 Kafka 传输**：
- Producer 侧写入共享文件系统：`/amplication-data/dsg-resource-data/{buildId}/resource-data.json`
- Kafka 消息仅携带 `buildId`
- Consumer 侧从共享存储读取

### 9.4 消费者注册机制

NestJS Microservices 启动时扫描所有 Controller，收集 `@EventPattern` / `@MessagePattern` 装饰器标注的方法，注册到 `messageHandlers` Map 中。`KafkaCustomTransport.bindEvents()` 在连接 Kafka 时统一订阅所有主题。

---

## 10. 核心文件速查表

| 组件 | 文件路径 |
|------|---------|
| 事件主题枚举 | [schema-registry/src/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/schema-registry/src/index.ts#L29-L69) |
| Kafka 生产者 | [KafkaProducer.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts) |
| Kafka 自定义传输 | [kafka.transport.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/util/nestjs/kafka/src/kafka.transport.ts) |
| Kafka 配置工厂 | [createNestjsKafkaConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/libs/util/nestjs/kafka/src/createNestjsKafkaConfig.ts) |
| 构建服务 (事件触发源) | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/build/build.service.ts) |
| 构建控制器 (事件消费中枢) | [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/build/build.controller.ts) |
| 构建执行器 (DSG消费) | [build-runner.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts) |
| 通知管道 | [app.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/notification-service/src/app.service.ts) |
| 通知消费者 | [app.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/notification-service/src/app.controller.ts) |
| 步骤执行框架 | [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/amplication-server/src/core/action/action.service.ts) |
| AI 对话消费者 | [kafka.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/106-amplication/packages/gpt-gateway/src/kafka/kafka.controller.ts) |
