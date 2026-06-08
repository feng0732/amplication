# Amplication Workflow 事件系统协作机制

## 1. 整体架构概述

Amplication 采用基于 **Kafka 消息队列** 的事件驱动架构，实现各微服务之间的解耦协作。系统由以下核心层次组成：

```
┌─────────────────────────────────────────────────────────────────┐
│                        触发源 (Triggers)                         │
│  GraphQL Resolver / REST Controller / 定时任务 / 外部事件        │
│  - WorkspaceResolver.currentWorkspace()                         │
│  - BuildResolver.createBuild()                                   │
│  - ResourceBtmResolver.triggerBreakServiceIntoMicroservices()    │
│  - UserController.notifyUseFeatureAnnouncement()                 │
│  - OutdatedVersionAlertService (内部告警触发)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │ 业务方法调用
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    业务服务层 (Business Services)                 │
│  BuildService / UserService / ResourceBtmService /               │
│  GptService / OutdatedVersionAlertService                        │
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
│  - amplication-server: BuildController / GptController /         │
│                        DBSchemaImportController /                 │
│                        UserActionController                      │
│  - amplication-build-manager: BuildRunnerController              │
│  - notification-service: AppController                           │
│  - gpt-gateway: KafkaController                                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  执行层 (Execution Pipeline)                      │
│  - ActionService.run(): 步骤化执行 + 状态管理                     │
│  - Notification Pipeline (compose 中间件)                         │
│  - NovuService: 通知触发                                          │
│  - 各业务 Service: 领域逻辑执行                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 事件契约层：Schema Registry

所有事件的统一规范定义在 `@amplication/schema-registry` 包中。

### 2.1 位置与结构

- **模块路径**: [libs/schema-registry](libs/schema-registry)
- **入口文件**: [index.ts](libs/schema-registry/src/index.ts)

每个事件类型是一个独立目录，包含三个文件：

```
lib/
├── code-generation-request/
│   ├── index.ts      # 导出 KafkaEvent 接口（extends DecodedKafkaMessage）
│   ├── key.ts        # 消息 Key 的 DTO（class-validator 装饰器）
│   └── value.ts      # 消息 Value 的 DTO（class-validator 装饰器）
├── code-generation-success/
├── user-build/
├── user-action/
├── gpt-conversation-start/
├── create-pr-request/
└── ... (约 25+ 种事件类型)
```

### 2.2 事件接口定义示例

以 `CodeGenerationRequest` 为例，见 [code-generation-request/index.ts](libs/schema-registry/src/lib/code-generation-request/index.ts)：

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

其中 `Value` DTO 定义了事件负载结构，见 [code-generation-request/value.ts](libs/schema-registry/src/lib/code-generation-request/value.ts)：

```typescript
export class Value {
  @IsString() buildId!: string;
  @IsString() resourceId!: string;
}
```

### 2.3 Kafka 主题枚举

所有 Kafka 主题统一在 [KAFKA_TOPICS](libs/schema-registry/src/index.ts#L29-L69) 枚举中维护：

| 主题分类 | 主题名变量 | 实际主题值 | 用途 |
|---------|-----------|-----------|------|
| Build 管理 | `CODE_GENERATION_REQUEST_TOPIC` | `build.internal.code-generation.request.1` | 触发代码生成 |
| | `CODE_GENERATION_SUCCESS_TOPIC` | `build.internal.code-generation.success.1` | 代码生成成功 |
| | `CODE_GENERATION_FAILURE_TOPIC` | `build.internal.code-generation.failure.1` | 代码生成失败 |
| | `CODE_GENERATION_NOTIFY_VERSION_TOPIC` | `build.internal.code-generation.notify-version.1` | 通知代码生成器版本 |
| | `BUILD_PLUGIN_NOTIFY_VERSION_TOPIC` | `build.internal.plugin.notify-version.1` | 通知插件版本 |
| | `DSG_LOG_TOPIC` | `build.internal.dsg-log.1` | 代码生成日志 |
| | `USER_BUILD_TOPIC` | `user-build.internal.1` | 构建完成用户通知 |
| 包管理器 | `PACKAGE_MANAGER_CREATE_REQUEST` | `package.manager.create-packages.request.0` | 发起包生成请求 |
| | `PACKAGE_MANAGER_CREATE_SUCCESS` | `package.manager.create-packages.success.0` | 包生成成功 |
| | `PACKAGE_MANAGER_CREATE_FAILURE` | `package.manager.create-packages.failure.0` | 包生成失败 |
| Git/PR | `CREATE_PR_REQUEST_TOPIC` | `git.internal.create-pr.request.2` | 发起 PR 创建 |
| | `CREATE_PR_SUCCESS_TOPIC` | `git.internal.create-pr.success.1` | PR 创建成功 |
| | `CREATE_PR_FAILURE_TOPIC` | `git.internal.create-pr.failure.1` | PR 创建失败 |
| | `CREATE_PR_LOG_TOPIC` | `git.internal.create-pr.log.1` | PR 创建日志 |
| | `KAFKA_REPOSITORY_PUSH_QUEUE` | `git.external.push.event.0` | 外部 Git push 事件 |
| | `GENERATE_PULL_REQUEST_TOPIC` | `git.internal.pull-request.request.1` | 发起拉取 PR |
| | `CREATE_PULL_REQUEST_COMPLETED_TOPIC` | `git.internal.pull-request.completed.1` | 拉取 PR 完成 |
| 插件 | `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` | `git.internal.download-private-plugins.request.0` | 下载私有插件请求 |
| | `DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC` | `git.internal.download-private-plugins.success.0` | 下载私有插件成功 |
| | `DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC` | `git.internal.download-private-plugins.failure.0` | 下载私有插件失败 |
| | `DOWNLOAD_PRIVATE_PLUGINS_LOG_TOPIC` | `git.internal.download-private-plugins.log.0` | 下载私有插件日志 |
| 用户/通知 | `USER_ACTION_TOPIC` | `user-action.internal.1` | 用户订阅注册 |
| | `USER_ANNOUNCEMENT_TOPIC` | `user-announcement.internal.1` | 功能公告广播 |
| | `TECH_DEBT_CREATED_TOPIC` | `platform.internal.tech-debt.created.1` | 技术债务告警 |
| AI | `AI_CONVERSATION_START_TOPIC` | `ai.internal.conversation.start.1` | GPT 对话开始 |
| | `AI_CONVERSATION_COMPLETED_TOPIC` | `ai.internal.conversation.completed.1` | GPT 对话完成 |
| 权限 | `CHECK_USER_ACCESS_TOPIC` | `authorization.internal.can-access-build.request.0` | 用户构建权限校验（请求/响应模式） |
| 其他 | `USER_ACTION_LOG_TOPIC` | `user-action.internal.action-log.1` | 用户操作步骤日志 |
| | `DB_SCHEMA_IMPORT_TOPIC` | `user-action.internal.db-schema-import.request.1` | DB Schema 导入请求 |
| | `SHARED_GRAPHQL_SUBSCRIPTION_PUBSUB_TOPIC` | `shared.internal.graphql-subscrition-pubsub.1` | GraphQL 订阅内部广播 |

---

## 3. Kafka 传输层

### 3.1 Kafka 模块

Kafka 基础设施封装在 `@amplication/util/nestjs/kafka` 中。

- **模块定义**: [Kafka.module.ts](libs/util/nestjs/kafka/src/Kafka.module.ts)
- **配置工厂**: [createNestjsKafkaConfig.ts](libs/util/nestjs/kafka/src/createNestjsKafkaConfig.ts)

模块导出两个核心服务：
- `KAFKA_SERIALIZER` (KafkaMessageJsonSerializer): 消息序列化
- `KafkaProducerService`: 事件发布服务

### 3.2 事件发布：KafkaProducerService

代码位置: [KafkaProducer.service.ts](libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts)

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

代码位置: [kafka.transport.ts](libs/util/nestjs/kafka/src/kafka.transport.ts)

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

每个需要消费 Kafka 事件的服务，都在其 `main.ts` 中连接 Kafka 微服务。

**使用 KafkaCustomTransport 的服务**（notification-service），见 [notification-service/src/main.ts](packages/notification-service/src/main.ts)：

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

**使用原生 ServerKafka 的服务**（amplication-server），见 [amplication-server/src/main.ts](packages/amplication-server/src/main.ts#L41)：

```typescript
app.connectMicroservice<MicroserviceOptions>(createNestjsKafkaConfig());
```

---

## 4. 触发入口分析（按事件主题逐一梳理）

### 4.1 构建触发：CODE_GENERATION_REQUEST_TOPIC

#### 4.1.1 最上层入口：BuildResolver.createBuild

GraphQL mutation 入口，调用 `BuildService.create()`。

#### 4.1.2 BuildService.create()

代码位置: [build.service.ts](packages/amplication-server/src/core/build/build.service.ts)

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
       └─ 4. generate() [见 4.1.3]
```

#### 4.1.3 generate() — 代码生成事件触发

代码位置: [build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L568-L618)

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

#### 4.1.4 onCodeGenerationSuccess() — 构建成功后的多事件触发

代码位置: [build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L450-L495)

当 BuildController 消费到 CODE_GENERATION_SUCCESS_TOPIC 时调用，发布两个事件：
1. 调用 `saveToGitProvider()` → emit CREATE_PR_REQUEST_TOPIC
2. emit USER_BUILD_TOPIC（构建完成用户通知）

---

### 4.2 用户订阅注册：USER_ACTION_TOPIC

**⚠️ 实际发布源头不是 UserActionService，而是 UserService.setNotificationRegistry()**

#### 4.2.1 触发入口：WorkspaceResolver.currentWorkspace()

代码位置: [workspace.resolver.ts](packages/amplication-server/src/core/workspace/workspace.resolver.ts#L76-L92)

```typescript
@Query(() => Workspace, { nullable: true })
async currentWorkspace(@UserEntity() currentUser: User): Promise<Workspace | null> {
  await this.analytics.trackWithContext({ properties: {}, event: EnumEventType.WorkspaceSelected });
  await this.userService.setLastActivity(currentUser.id);
  const externalId = await this.userService.setNotificationRegistry(currentUser);  // ← 触发点
  return { ...currentUser.workspace, externalId };
}
```

每当用户切换工作区（查询 currentWorkspace）时，就会触发通知订阅注册。

#### 4.2.2 UserService.setNotificationRegistry()

代码位置: [user.service.ts](packages/amplication-server/src/core/user/user.service.ts#L175-L202)

```typescript
async setNotificationRegistry(user: User) {
  const externalId = encryptString(user.id);
  const canShowUserNotification = (await this.billingService.getBooleanEntitlement(...))?.hasAccess;

  this.kafkaProducerService
    .emitMessage(KAFKA_TOPICS.USER_ACTION_TOPIC, <UserAction.KafkaEvent>{
      key: {},
      value: {
        userId: user.account.id,
        externalId,
        firstName: user.account.firstName,
        lastName: user.account.lastName,
        email: user.account.email,
        action: UserAction.UserActionType.CURRENT_WORKSPACE,
        enableUser: canShowUserNotification,
      },
    })
    .catch((error) => this.logger.error(...));

  return externalId;
}
```

---

### 4.3 功能公告广播：USER_ANNOUNCEMENT_TOPIC

#### 4.3.1 触发入口：UserController.notifyUseFeatureAnnouncement()

代码位置: [user.controller.ts](packages/amplication-server/src/core/user/user.controller.ts)

这是一个 **REST API** 入口（非 GraphQL）：

```typescript
@Controller("announce-new-feature")
export class UserController {
  @Post(`notifyUseFeatureAnnouncement/:token`)
  async notifyUseFeatureAnnouncement(
    @Param("token") token: string,
    @Body() data: NotifyUseFeatureAnnouncementInput
  ): Promise<boolean> {
    // 校验 token 后调用
    return this.userService.notifyUserFeatureAnnouncement(
      userActiveDaysBack, notificationId
    );
  }
}
```

需要携带配置的 `FEATURE_ANNOUNCEMENT_NOTIFICATION_TOKEN` 才能触发。

#### 4.3.2 UserService.notifyUserFeatureAnnouncement()

代码位置: [user.service.ts](packages/amplication-server/src/core/user/user.service.ts#L204-L269)

查询最近 `userActiveDaysBack` 天内活跃的用户，对每个用户 emit 一条 USER_ANNOUNCEMENT_TOPIC 事件。

---

### 4.4 技术债务告警：TECH_DEBT_CREATED_TOPIC

#### 4.4.1 触发源：OutdatedVersionAlertService

代码位置: [outdatedVersionAlert.service.ts](packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L96-L119)

当检测到服务存在过时依赖时，对工作区内每个用户 emit 一条 TECH_DEBT_CREATED_TOPIC 事件。

---

### 4.5 AI 对话（拆分单体架构）：AI_CONVERSATION_START_TOPIC

**⚠️ 实际触发入口不是 GptResolver，而是 ResourceBtmResolver（GraphQL Mutation）**

#### 4.5.1 最上层入口：ResourceBtmResolver.triggerBreakServiceIntoMicroservices()

代码位置: [resourceBtm.resolver.ts](packages/amplication-server/src/core/resource/resourceBtm.resolver.ts#L20-L39)

```typescript
@Mutation(() => UserAction, {
  nullable: true,
  description: "Trigger the generation of recommendations for breaking a resource into microservices",
})
async triggerBreakServiceIntoMicroservices(
  @Args({ name: "resourceId", type: () => String }) resourceId: string,
  @UserEntity() user: User
): Promise<UserAction> {
  return this.resourceBtmService.triggerBreakServiceIntoMicroservices({
    resourceId,
    user,
  });
}
```

#### 4.5.2 ResourceBtmService.triggerBreakServiceIntoMicroservices()

代码位置: [resourceBtm.service.ts](packages/amplication-server/src/core/resource/resourceBtm.service.ts#L116-L155)

```typescript
async triggerBreakServiceIntoMicroservices({ resourceId, user }) {
  // ...权限校验、获取资源实体...
  const prompt = this.generatePromptForBreakTheMonolith(resource);
  const conversationParams = [{ name: "userInput", value: prompt }];

  const userAction = await this.gptService.startConversation(
    ConversationTypeKey.BreakTheMonolith,
    conversationParams,
    user.id,
    resourceId
  );
  return userAction;
}
```

#### 4.5.3 GptService.startConversation()

代码位置: [gpt.service.ts](packages/amplication-server/src/core/gpt/gpt.service.ts#L36-L65)

```typescript
async startConversation(conversationTypeKey, params, userId, resourceId?): Promise<UserAction> {
  // 1. 创建 UserAction + Action + 初始 Step (START_CONVERSATION: Waiting)
  const userAction = await this.userActionService.createUserActionByTypeWithInitialStep(
    EnumUserActionType.GptConversation,
    {},
    CONVERSATION_INITIAL_STEP,
    userId,
    resourceId
  );

  // 2. 发布 AI_CONVERSATION_START_TOPIC 到 Kafka
  const kafkaMessage: GptConversationStart.KafkaEvent = {
    key: { requestUniqueId: userAction.id },
    value: {
      requestUniqueId: userAction.id,
      messageTypeKey: conversationTypeKey,
      params,
    },
  };
  await this.kafkaProducerService.emitMessage(
    KAFKA_TOPICS.AI_CONVERSATION_START_TOPIC,
    kafkaMessage
  );
  return userAction;
}
```

**调用链总览**：
```
ResourceBtmResolver.triggerBreakServiceIntoMicroservices() [GraphQL Mutation]
           │
           ▼
ResourceBtmService.triggerBreakServiceIntoMicroservices()
           │
           ├─ 生成 prompt（包含资源实体信息）
           │
           ▼
GptService.startConversation()
           │
           ├─ 创建 UserAction 记录（步骤状态管理）
           │
           └─ emit AI_CONVERSATION_START_TOPIC → gpt-gateway 消费
```

---

### 4.6 AI 对话完成：AI_CONVERSATION_COMPLETED_TOPIC

由 gpt-gateway 服务在处理完 AI 对话后发布，amplication-server 的 GptController 消费并调用 `GptService.onConversationCompleted()` 更新 UserAction 步骤状态和元数据。

---

## 5. 事件消费与订阅机制

### 5.1 所有消费者方法总览（共 24 个）

项目中共有 **5 个服务、7 个 Controller、24 个消费者方法**。

#### 5.1.1 amplication-server（4 个 Controller，15 个消费者方法）

**BuildController**（12 个方法）— [build.controller.ts](packages/amplication-server/src/core/build/build.controller.ts)：

| 装饰器 | 订阅主题 | 处理方法 | plainToInstance 校验 |
|--------|---------|---------|---------------------|
| `@MessagePattern` | `CHECK_USER_ACCESS_TOPIC` | `checkUserAccess()` | ✅ 有 |
| `@EventPattern` | `CODE_GENERATION_NOTIFY_VERSION_TOPIC` | `onCodeGenerationNotifyVersion()` | ✅ 有 |
| `@EventPattern` | `BUILD_PLUGIN_NOTIFY_VERSION_TOPIC` | `onPluginNotifyVersion()` | ✅ 有 |
| `@EventPattern` | `CODE_GENERATION_SUCCESS_TOPIC` | `onCodeGenerationSuccess()` | ✅ 有 |
| `@EventPattern` | `CODE_GENERATION_FAILURE_TOPIC` | `onCodeGenerationFailure()` | ✅ 有 |
| `@EventPattern` | `CREATE_PR_SUCCESS_TOPIC` | `onPullRequestCreated()` | ✅ 有 |
| `@EventPattern` | `CREATE_PR_FAILURE_TOPIC` | `onPullRequestFailure()` | ✅ 有 |
| `@EventPattern` | `DSG_LOG_TOPIC` | `onDsgLog()` | ✅ 有 |
| `@EventPattern` | `CREATE_PR_LOG_TOPIC` | `onCreatePullRequestLog()` | ✅ 有 |
| `@EventPattern` | `DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC` | `onDownloadPrivatePluginsSuccess()` | ✅ 有 |
| `@EventPattern` | `DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC` | `onDownloadPrivatePluginsFailure()` | ✅ 有 |
| `@EventPattern` | `DOWNLOAD_PRIVATE_PLUGINS_LOG_TOPIC` | `onDownloadPrivatePluginsLog()` | ✅ 有 |

**DBSchemaImportController**（1 个方法）— [dbSchemaImport.controller.ts](packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.controller.ts)：

| 装饰器 | 订阅主题 | 处理方法 | plainToInstance 校验 |
|--------|---------|---------|---------------------|
| `@EventPattern` | `DB_SCHEMA_IMPORT_TOPIC` | `onDBSchemaImportRequest()` | ✅ 有 |

**GptController**（1 个方法）— [gpt.controller.ts](packages/amplication-server/src/core/gpt/gpt.controller.ts)：

| 装饰器 | 订阅主题 | 处理方法 | plainToInstance 校验 |
|--------|---------|---------|---------------------|
| `@EventPattern` | `AI_CONVERSATION_COMPLETED_TOPIC` | `onAiConversationCompleted()` | ❌ **无** |

GptController.onAiConversationCompleted() 直接使用 message 作为参数传递给 service，未做 DTO 转换。

**UserActionController**（1 个方法）— [action.controller.ts](packages/amplication-server/src/core/action/action.controller.ts)：

| 装饰器 | 订阅主题 | 处理方法 | plainToInstance 校验 |
|--------|---------|---------|---------------------|
| `@EventPattern` | `USER_ACTION_LOG_TOPIC` | `onUserActionLog()` | ✅ 有 |

#### 5.1.2 amplication-build-manager（1 个 Controller，3 个消费者方法）

**BuildRunnerController**（3 个消费者方法 + 3 个 HTTP POST 端点）— [build-runner.controller.ts](packages/amplication-build-manager/src/build-runner/build-runner.controller.ts)：

| 装饰器 | 订阅主题/路由 | 处理方法 | plainToInstance 校验 |
|--------|---------|---------|---------------------|
| `@EventPattern` | `PACKAGE_MANAGER_CREATE_SUCCESS` | `onPackageManagerCreateSuccess()` | ✅ 有 |
| `@EventPattern` | `PACKAGE_MANAGER_CREATE_FAILURE` | `onPackageManagerCreateFailure()` | ✅ 有 |
| `@EventPattern` | `CODE_GENERATION_REQUEST_TOPIC` | `onCodeGenerationRequest()` | ❌ **无** |
| `@Post` (HTTP) | `code-generation-success` | `onCodeGenerationSuccess()` | N/A (HTTP DTO) |
| `@Post` (HTTP) | `code-generation-failure` | `onCodeGenerationFailure()` | N/A (HTTP DTO) |
| `@Post` (HTTP) | `notify-plugin-version` | `onNotifyPluginVersion()` | N/A (HTTP DTO) |

`onCodeGenerationRequest()` 直接使用 `message.resourceId` 和 `message.buildId`，未做 DTO 转换。

**Package Manager 事件链补充说明：**

Package Manager 是构建流程中的一个可选项（通过环境变量 `ENABLE_PACKAGE_MANAGER` 开关控制），完整事件链如下：

1. **REQUEST 发布方**：`BuildRunnerService.handleDsgJobCompleted()` → `generatePackages()`（见 [build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L69-L88)）
   - 触发条件：所有 DSG 子任务 (jobs) 全部成功，且 `dsgResourceData.packages?.length > 0` 且 `enablePackageManager === true`
   - 发布事件：`PACKAGE_MANAGER_CREATE_REQUEST`，payload 含 `{ resourceId, buildId, dsgResourceData }`

2. **REQUEST 消费方**：外部 Package Manager 微服务（不在当前代码库中），负责执行 `npm/pnpm/yarn install` 等包安装操作

3. **SUCCESS 消费方**：`BuildRunnerController.onPackageManagerCreateSuccess()` → `BuildRunnerService.onPackageManagerCreateSuccess()` → `codeGenerationAndPackagesCompleted()` → 发布 `CODE_GENERATION_SUCCESS_TOPIC`（见 [build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L54-L58)）

4. **FAILURE 消费方**：`BuildRunnerController.onPackageManagerCreateFailure()` → `BuildRunnerService.onPackageManagerCreateFailure()` → `emitCodeGenerationFailure()` → 发布 `CODE_GENERATION_FAILURE_TOPIC`（见 [build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L60-L67)）

如果无需 Package Manager（无 packages 或开关关闭），则在 `handleDsgJobCompleted()` 中直接调用 `codeGenerationAndPackagesCompleted()` 发布 CODE_GENERATION_SUCCESS，跳过该分支。

#### 5.1.3 notification-service（1 个 Controller，5 个消费者方法）

**AppController**（5 个方法）— [app.controller.ts](packages/notification-service/src/app.controller.ts)：

所有 5 个方法**全部没有** plainToInstance，message 类型声明为 `{ [key: string]: any }`，直接透传给 AppService。

| 装饰器 | 订阅主题 | 处理方法 | plainToInstance 校验 |
|--------|---------|---------|---------------------|
| `@EventPattern` | `"user-action.internal.1"` | `subscribeNotification()` | ❌ **无** |
| `@EventPattern` | `"user-build.internal.1"` | `notifyBuild()` | ❌ **无** |
| `@EventPattern` | `"user-announcement.internal.1"` | `notifyFeatureAnnouncement()` | ❌ **无** |
| `@EventPattern` | `"user-preview-generation-completed.internal.1"` | `previewUserGenerationCompleted()` | ❌ **无** |
| `@EventPattern` | `"platform.internal.tech-debt.created.1"` | `notifyTechDebt()` | ❌ **无** |

注意：这里使用了**字符串字面量主题名**而非 `KAFKA_TOPICS` 枚举。

#### 5.1.4 gpt-gateway（1 个 Controller，1 个消费者方法）

**KafkaController**（1 个方法）— [kafka.controller.ts](packages/gpt-gateway/src/kafka/kafka.controller.ts)：

| 装饰器 | 订阅主题 | 处理方法 | plainToInstance 校验 |
|--------|---------|---------|---------------------|
| `@EventPattern` | `AI_CONVERSATION_START_TOPIC` | `onAiConversationStart_1()` | ✅ 有 |

### 5.2 plainToInstance DTO 校验情况总结

**24 个消费者方法中，17 个使用了 plainToInstance，7 个没有使用：**

| 未使用 plainToInstance 的方法 | 所在 Controller | 说明 |
|------------------------------|----------------|------|
| `onAiConversationCompleted()` | amplication-server/GptController | 直接透传 message |
| `onCodeGenerationRequest()` | amplication-build-manager/BuildRunnerController | 直接访问 message 属性 |
| `subscribeNotification()` | notification-service/AppController | 声明为 any |
| `notifyBuild()` | notification-service/AppController | 声明为 any |
| `notifyFeatureAnnouncement()` | notification-service/AppController | 声明为 any |
| `previewUserGenerationCompleted()` | notification-service/AppController | 声明为 any |
| `notifyTechDebt()` | notification-service/AppController | 声明为 any |

### 5.3 NestJS 装饰器订阅模式

使用 `@EventPattern()` (fire-and-forget，无返回值) 或 `@MessagePattern()` (request-response，需要返回值) 装饰器标注 Controller 方法。

唯一使用 `@MessagePattern` 的是 `BuildController.checkUserAccess()`，用于用户构建权限校验的请求-响应模式。

---

## 6. 订阅执行管道：Notification Pipeline

notification-service 使用**函数组合 (compose)** 模式构建通知处理管道。

### 6.1 管道定义：AppService.notificationService()

代码位置: [app.service.ts](packages/notification-service/src/app.service.ts)

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

#### subscribeUser 示例（USER_ACTION_TOPIC 处理）

代码位置: [subscribeUser.ts](packages/notification-service/src/notification-packages/subscribeUser.ts)

处理 `user-action.internal.1` 主题，根据 `action` 和 `enableUser` 字段决定调用 Novu 的 `createSubscriber()` 还是 `deleteSubscriber()`。

#### buildCompleted 示例

代码位置: [buildCompleted.ts](packages/notification-service/src/notification-packages/buildCompleted.ts)

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

代码位置: [novuService.ts](packages/notification-service/src/util/novuService.ts)

封装 Novu 通知平台 API：
- `createSubscriber()` / `deleteSubscriber()` - 订阅者管理
- `triggerNotificationToSubscriber()` - 触发单用户通知
- `broadCastEventToAll()` - 广播通知

---

## 7. 步骤化执行框架：ActionService

代码位置: [action.service.ts](packages/amplication-server/src/core/action/action.service.ts)

### 7.1 核心数据模型

```
Action (1) ────→ ActionStep (N) ────→ ActionLog (N)
  │                 │
  └─ build          └─ name (如 "GENERATE_APPLICATION")
  └─ userAction     └─ status: Running / Success / Failed / Waiting
                     └─ message
                     └─ completedAt
```

Action 通过 `actionId` 字段关联到 Build 或 UserAction。

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

### 7.3 UserAction 与 Action 的关系

UserAction 是更高层的用户操作记录（如 GptConversation、DBSchemaImport），其 `actionId` 字段关联到 Action。UserActionService.createUserActionByTypeWithInitialStep() 负责创建 UserAction + Action + 初始 Step。

---

## 8. 完整工作流示例：构建代码生成 → 包管理 → 推送 Git → 通知用户

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
│ 3. amplication-build-manager - BuildRunnerController                          │
│    @EventPattern(CODE_GENERATION_REQUEST_TOPIC)                              │
│    └─ buildRunnerService.runBuild()                                          │
│         ├─ emit CODE_GENERATION_NOTIFY_VERSION_TOPIC (版本通知)              │
│         ├─ splitBuildsIntoJobs() 将大构建拆分为多个 job                        │
│         └─ 对每个 job 调用 DSG Runner (HTTP POST 到 DSG_RUNNER_URL)           │
│            ├─ 过程中多次 emit DSG_LOG_TOPIC (日志流式回写)                    │
│            └─ DSG Runner 完成后回调 build-runner.controller                  │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │ HTTP POST (不是 Kafka)
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 4. amplication-build-manager - BuildRunnerController (HTTP 回调端点)          │
│    POST /build-runner/code-generation-success → handleDsgJobCompleted()      │
│    POST /build-runner/code-generation-failure → emitCodeGenerationFailure()  │
│                                                                              │
│ handleDsgJobCompleted() 处理逻辑:                                             │
│    ├─ buildJobsHandlerService 聚合所有 job 的状态 (Redis)                     │
│    ├─ 如有失败 job → emit CODE_GENERATION_FAILURE_TOPIC                      │
│    ├─ 如仍有 job 进行中 → 直接返回 (InProgress)                               │
│    └─ 所有 job 成功 → 判断是否需要 Package Manager                            │
│         │                                                                     │
│         ├─ 有 packages 且 enablePackageManager=true → 进入步骤 5             │
│         │                                                                     │
│         └─ 无 packages 或开关关闭 → emit CODE_GENERATION_SUCCESS_TOPIC       │
│                                 ──────────────────────────────────────→ 跳至步骤 6 │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │ 需要 Package Manager
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 5. Package Manager 事件链 (可选分支)                                           │
│                                                                              │
│  5a. BuildRunnerService.generatePackages()                                    │
│      └─ emit PACKAGE_MANAGER_CREATE_REQUEST                                  │
│         { resourceId, buildId, dsgResourceData }                              │
│                                   │ Kafka                                     │
│                                   ▼                                           │
│  5b. 外部 Package Manager 微服务 (不在当前代码库)                               │
│      执行 npm/pnpm/yarn install 等包安装操作                                  │
│         ├─ 成功 → emit PACKAGE_MANAGER_CREATE_SUCCESS                        │
│         └─ 失败 → emit PACKAGE_MANAGER_CREATE_FAILURE                        │
│                                   │ Kafka                                     │
│                                   ▼                                           │
│  5c. BuildRunnerController 消费响应事件                                       │
│      ├─ SUCCESS → codeGenerationAndPackagesCompleted()                       │
│      │         └─ emit CODE_GENERATION_SUCCESS_TOPIC                         │
│      └─ FAILURE → onPackageManagerCreateFailure()                            │
│                └─ emit CODE_GENERATION_FAILURE_TOPIC                         │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │ CODE_GENERATION_SUCCESS / FAILURE
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 6. amplication-server - BuildController                                       │
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
│ 7a. git-sync-manager (ee 目录)     │  │ 7b. notification-service             │
│     消费 CREATE_PR_REQUEST_TOPIC   │  │     消费 USER_BUILD_TOPIC            │
│     调用 GitHub/GitLab API 创建 PR  │  │     compose 管道:                    │
│     成功 → emit CREATE_PR_SUCCESS  │  │       subscribeUser → buildCompleted  │
│     失败 → emit CREATE_PR_FAILURE  │  │       → novuPackage                  │
└──────────────┬─────────────────────┘  │         → Novu.trigger("build-       │
               │ Kafka                  │           completed", {subscriberId})│
               ▼                        └──────────────────────────────────────┘
┌──────────────────────────────────────────────────────────────────────────────┐
│ 8. amplication-server - BuildController                                       │
│    @EventPattern(CREATE_PR_SUCCESS_TOPIC)                                    │
│    └─ buildService.onCreatePRSuccess()                                       │
│         ├─ Step(PUSH_TO_GIT_PROVIDER:Success)                                │
│         └─ Build.status = Completed                                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. AI 对话工作流：拆分单体架构（Break The Monolith）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 1. 用户在前端点击 "Break The Monolith"                                        │
│    GraphQL: mutation triggerBreakServiceIntoMicroservices(resourceId)        │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 2. amplication-server - ResourceBtmResolver                                   │
│    └─ ResourceBtmService.triggerBreakServiceIntoMicroservices()              │
│         ├─ 检查订阅权限（PreviewBreakTheMonolith 计费项）                     │
│         ├─ 获取资源所有实体信息                                                │
│         ├─ generatePromptForBreakTheMonolith() 生成 AI prompt                │
│         └─ 调用 GptService.startConversation()                               │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 3. amplication-server - GptService.startConversation()                       │
│    ├─ UserActionService.createUserActionByTypeWithInitialStep()              │
│    │   创建 UserAction(type=GptConversation) + Action + Step(START_          │
│    │   CONVERSATION:Waiting)                                                 │
│    └─ emit AI_CONVERSATION_START_TOPIC                                       │
│       { requestUniqueId, messageTypeKey="BREAK_THE_MONOLITH", params }      │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │ Kafka
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 4. gpt-gateway - KafkaController                                              │
│    @EventPattern(AI_CONVERSATION_START_TOPIC)                                │
│    ├─ plainToInstance(GptConversationStart.Value, value)                     │
│    └─ conversationType.startConversion(messageInput) → 调用 AI API           │
│       (完成后 emit AI_CONVERSATION_COMPLETED_TOPIC)                          │
└──────────────────────────────────┬───────────────────────────────────────────┘
                                   │ Kafka
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 5. amplication-server - GptController                                         │
│    @EventPattern(AI_CONVERSATION_COMPLETED_TOPIC)                            │
│    └─ GptService.onConversationCompleted()                                   │
│         ├─ 更新 Step(START_CONVERSATION:Success/Failed)                      │
│         └─ 将 AI 返回结果存入 UserAction.metadata                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 关键协作模式总结

### 10.1 两种消息模式

| 模式 | 装饰器 | 返回值 | 使用场景 |
|------|--------|--------|---------|
| 发布订阅 (Pub/Sub) | `@EventPattern` | `void` | 代码生成请求、日志、通知、AI 对话 |
| 请求响应 (Req/Res) | `@MessagePattern` | `{value: ...}` | CHECK_USER_ACCESS_TOPIC 权限校验 |

### 10.2 事件编排：Saga 模式

构建工作流和 AI 对话工作流均采用 **Saga 模式** 的事件驱动编排：
- 没有中央协调器，每个消费者处理完后发布下一个事件
- 步骤状态通过数据库 (ActionStep) 持久化，支持故障恢复
- 通过 `buildId` / `actionId` / `requestUniqueId` (即 userAction.id) 作为 Correlation ID 关联所有事件

### 10.3 大对象传输策略

DSGResourceData（可能包含实体、模块、插件等大量数据）**不通过 Kafka 传输**：
- Producer 侧写入共享文件系统：`/amplication-data/dsg-resource-data/{buildId}/resource-data.json`
- Kafka 消息仅携带 `buildId` 和 `resourceId`
- Consumer 侧从共享存储读取

### 10.4 消费者注册机制

NestJS Microservices 启动时扫描所有 Controller，收集 `@EventPattern` / `@MessagePattern` 装饰器标注的方法，注册到 `messageHandlers` Map 中。`KafkaCustomTransport.bindEvents()` 在连接 Kafka 时统一订阅所有主题。

### 10.5 主题引用方式不一致

- amplication-server、amplication-build-manager、gpt-gateway：使用 `KAFKA_TOPICS.XXX` 枚举
- notification-service AppController：使用**字符串字面量**（如 `"user-action.internal.1"`），但在 Notification Package 中使用 `KAFKA_TOPICS` 枚举做比较

---

## 11. 核心文件速查表

| 组件 | 文件路径 |
|------|---------|
| 事件主题枚举 | [schema-registry/src/index.ts](libs/schema-registry/src/index.ts#L29-L69) |
| Kafka 生产者服务 | [KafkaProducer.service.ts](libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts) |
| Kafka 自定义传输 | [kafka.transport.ts](libs/util/nestjs/kafka/src/kafka.transport.ts) |
| Kafka 配置工厂 | [createNestjsKafkaConfig.ts](libs/util/nestjs/kafka/src/createNestjsKafkaConfig.ts) |
| 构建服务 (BUILD 事件发布源) | [build.service.ts](packages/amplication-server/src/core/build/build.service.ts) |
| 构建控制器 (BUILD 事件消费中枢) | [build.controller.ts](packages/amplication-server/src/core/build/build.controller.ts) |
| 用户服务 (USER_ACTION / USER_ANNOUNCEMENT 发布源) | [user.service.ts](packages/amplication-server/src/core/user/user.service.ts) |
| 用户通知 REST 入口 | [user.controller.ts](packages/amplication-server/src/core/user/user.controller.ts) |
| 工作区 Resolver (USER_ACTION 触发入口) | [workspace.resolver.ts](packages/amplication-server/src/core/workspace/workspace.resolver.ts) |
| 技术债务告警服务 (TECH_DEBT 发布源) | [outdatedVersionAlert.service.ts](packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts) |
| 拆单体 Resolver (AI 对话触发入口) | [resourceBtm.resolver.ts](packages/amplication-server/src/core/resource/resourceBtm.resolver.ts) |
| 拆单体服务 (AI 对话中间层) | [resourceBtm.service.ts](packages/amplication-server/src/core/resource/resourceBtm.service.ts) |
| GPT 服务 (AI_CONVERSATION_START 发布源) | [gpt.service.ts](packages/amplication-server/src/core/gpt/gpt.service.ts) |
| GPT 控制器 (AI_CONVERSATION_COMPLETED 消费者) | [gpt.controller.ts](packages/amplication-server/src/core/gpt/gpt.controller.ts) |
| 构建执行器 (代码生成消费者+Package Manager) | [build-runner.controller.ts](packages/amplication-build-manager/src/build-runner/build-runner.controller.ts) |
| 构建执行服务 (runBuild/job拆分/Package Manager事件发布) | [build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts) |
| Package Manager 请求 DTO | [package-manager-create-request/value.ts](libs/schema-registry/src/lib/package-manager-create-request/value.ts) |
| Package Manager 成功 DTO | [package-manager-create-success/value.ts](libs/schema-registry/src/lib/package-manager-create-success/value.ts) |
| Package Manager 失败 DTO | [package-manager-create-failure/value.ts](libs/schema-registry/src/lib/package-manager-create-failure/value.ts) |
| 通知消费者 (5 个通知主题) | [app.controller.ts](packages/notification-service/src/app.controller.ts) |
| 通知处理管道 (compose 中间件) | [app.service.ts](packages/notification-service/src/app.service.ts) |
| 通知中间件：用户订阅 | [subscribeUser.ts](packages/notification-service/src/notification-packages/subscribeUser.ts) |
| 通知中间件：构建完成 | [buildCompleted.ts](packages/notification-service/src/notification-packages/buildCompleted.ts) |
| 步骤执行框架 | [action.service.ts](packages/amplication-server/src/core/action/action.service.ts) |
| 用户操作服务 | [userAction.service.ts](packages/amplication-server/src/core/userAction/userAction.service.ts) |
| AI 对话启动消费者 | [kafka.controller.ts](packages/gpt-gateway/src/kafka/kafka.controller.ts) |
| DB Schema 导入消费者 | [dbSchemaImport.controller.ts](packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.controller.ts) |
| 用户操作日志消费者 | [action.controller.ts](packages/amplication-server/src/core/action/action.controller.ts) |
