# Amplication Action Runner & Build Executor 状态回写深度分析

## 一、整体架构与状态模型

### 1.1 核心服务与职责

| 服务 | 模块 | 职责 |
|------|------|------|
| `amplication-server` | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts) | Build 创建、Action/Step 管理、最终状态入库、Git 推送编排 |
| `amplication-server` | [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts) | Step 生命周期、ActionLog 写入 |
| `amplication-server` | [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.controller.ts) | Kafka 事件消费者（成功/失败/日志/PR） |
| `amplication-build-manager` | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | DSG 执行、Package Manager 编排、状态聚合、失败去重 |
| `amplication-build-manager` | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | 子作业拆分、Redis 状态读写、Job ID 编解码 |
| `amplication-build-manager` | [build-logger.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-logger/build-logger.service.ts) | DSG 日志转发（带 domain 前缀） |
| `amplication-build-manager` | [redis.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/redis/redis.service.ts) | Job 状态缓存层 |

### 1.2 三层状态模型

系统采用 **三层状态** 分层管理：

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3: PostgreSQL (Build 表)                                      │
│  status: Running | Completed | Failed | Invalid | Unknown | Canceled │
│  gitStatus: NotConnected | Waiting | Completed | Failed | Canceled   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Layer 2: PostgreSQL (ActionStep 表)                         │    │
│  │  status: Waiting | Running | Failed | Success               │    │
│  │  Step 类型: ADD_TO_QUEUE | GENERATE_APPLICATION |            │    │
│  │            DOWNLOAD_PRIVATE_PLUGINS | PUSH_TO_GIT_PROVIDER   │    │
│  │  ┌──────────────────────────────────────────────────────┐    │    │
│  │  │  Layer 1: Redis (子 Job 状态)                           │    │    │
│  │  │  status: in-progress | success | failure               │    │    │
│  │  │  Key: buildId, Value: {jobBuildId → status}            │    │    │
│  │  └──────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

各层状态枚举定义：

- Build 状态：[EnumBuildStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/dto/EnumBuildStatus.ts)
- Build Git 状态：[EnumBuildGitStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/dto/EnumBuildGitStatus.ts)
- Action Step 状态：[EnumActionStepStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/dto/EnumActionStepStatus.ts)
- Job 状态：[types.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/types.ts)

---

## 二、排队机制与初始状态入库

### 2.1 Build 创建与初始状态写入

构建请求入口在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L349) 的 `create()` 方法：

```typescript
// build.service.ts#L268-L311
async create(args: CreateBuildArgs): Promise<Build> {
  // ...
  const build = await this.prisma.build.create({
    ...args,
    data: {
      ...args.data,
      version,
      createdAt: new Date(),
      status: EnumBuildStatus.Running,        // Layer 3: Build 状态 = Running
      gitStatus: EnumBuildGitStatus.Waiting,   // Layer 3: Git 状态 = Waiting
      entityVersions: { connect: latestEntityVersions.map(v => ({ id: v.id })) },
      action: {
        create: {
          steps: {
            create: createInitialStepData(version, args.data.message), // Layer 2: 创建初始 Step
          },
        },
      },
    },
    include: { commit: true, resource: true },
  });
  // ...
  // 有私钥插件 → 先下载插件；无插件 → 直接进入 generate() 阶段
  if (resourcePrivatePlugins.length > 0) {
    await this.downloadPrivatePlugins(logger, build, user, resourcePrivatePlugins);
  } else {
    await this.generate(logger, build, user);
  }
}
```

### 2.2 初始 Step 创建（ADD_TO_QUEUE）

`createInitialStepData()` 在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L179-L208) 中创建第一个 Step：

```typescript
// build.service.ts#L179-L208
export function createInitialStepData(
  version: string,
  message: string
): Prisma.ActionStepCreateWithoutActionInput {
  return {
    message: "Adding task to queue",
    name: "ADD_TO_QUEUE",
    status: EnumActionStepStatus.Success,   // 立即成功（入队动作本身瞬时完成）
    completedAt: new Date(),
    logs: {
      create: [
        { level: EnumActionLogLevel.Info, message: "Create build generation task", meta: {} },
        { level: EnumActionLogLevel.Info, message: `Build version: ${version}`, meta: {} },
        { level: EnumActionLogLevel.Info, message: `Build message: ${message}`, meta: {} },
      ],
    },
  };
}
```

### 2.3 代码生成 Step 创建与 Kafka 入队

`generate()` 方法创建 `GENERATE_APPLICATION` Step 并发送 Kafka 消息：

```typescript
// build.service.ts#L568-L618
private async generate(logger: ILogger, build: Build, user: User): Promise<string> {
  return this.actionService.run(
    build.actionId,
    GENERATE_STEP_NAME,              // "GENERATE_APPLICATION"
    GENERATE_STEP_MESSAGE,           // "Generating Application"
    async (step) => {
      const dsgResourceData = await this.getDSGResourceData(resource, buildId, buildVersion, user);
      await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);
      
      // 发送轻量消息：只传 resourceId + buildId
      const codeGenerationEvent: CodeGenerationRequest.KafkaEvent = {
        key: null,
        value: { resourceId, buildId },
      };
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
        codeGenerationEvent
      );
      return null;
    },
    true   // leaveStepOpenAfterSuccessfulExecution = true，保持 Running 等待回调
  );
}
```

`actionService.run()` 在 [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts#L275-L295) 创建 Step：

```typescript
// action.service.ts#L71-L90
async createStep(actionId: string, stepName: string, message: string): Promise<ActionStep> {
  return this.prisma.actionStep.create({
    data: {
      status: EnumActionStepStatus.Running,   // Layer 2: Step = Running
      message,
      name: stepName,
      action: { connect: { id: actionId } },
    },
  });
}
```

此时数据库状态：
- `Build.status = Running`
- `Build.gitStatus = Waiting`
- `ActionStep[ADD_TO_QUEUE].status = Success`
- `ActionStep[GENERATE_APPLICATION].status = Running`

---

## 三、日志事件完整流转

### 3.1 日志数据流全景

```
DSG Runner (容器内)
     │  POST /build-logger/create-log  (HTTP)
     ▼
Build Manager [build-logger.controller.ts]
     │  addCodeGenerationLog()
     │  - 提取 buildId（去 domain 后缀）
     │  - Job 有 domain 时加 [server]/[admin-ui] 前缀
     ▼
Kafka: DSG_LOG_TOPIC
     │
     ▼
Server [build.controller.ts] onDsgLog()
     │  actionService.logByStepId()
     ▼
PostgreSQL: ActionLog 表
     │  (同时 Error 级别会触发 Segment 埋点)
     ▼
Segment Analytics
```

### 3.2 DSG Runner → Build Manager：日志接收

Build Manager 通过 HTTP 接收 DSG Runner 的日志：

```typescript
// build-logger.controller.ts#L9-L14
@Controller("build-logger")
export class BuildLoggerController {
  @Post("create-log")
  async onCodeGenerationLog(@Body() logEntry: CodeGenerationLogRequestDto): Promise<void> {
    await this.buildLoggerService.addCodeGenerationLog(logEntry);
  }
}
```

DTO 定义在 [OnCodeGenerationLogRequest.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-logger/dto/OnCodeGenerationLogRequest.ts)：

```typescript
export interface CodeGenerationLogRequestDto extends LogEntry {
  buildId: string;  // 可能是 buildId 或 buildId-server / buildId-admin-ui
}
```

### 3.3 Build Manager：日志加工与转发

[build-logger.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-logger/build-logger.service.ts#L16-L38) 对日志进行加工：

```typescript
// build-logger.service.ts#L16-L38
async addCodeGenerationLog(logEntry: CodeGenerationLogRequestDto): Promise<void> {
  const buildId = this.buildJobsHandlerService.extractBuildId(logEntry.buildId);

  // 如果是子 Job 日志，给 message 加 domain 前缀区分来源
  if (buildId !== logEntry.buildId) {
    const domain = this.buildJobsHandlerService.extractDomain(logEntry.buildId);
    logEntry.message = `[${domain}] ${logEntry.message}`;
  }

  // 转发到 Kafka，key = { buildId }
  const logEvent: CodeGenerationLog.KafkaEvent = {
    key: { buildId },
    value: { ...logEntry, buildId },
  };
  await this.producerService.emitMessage(KAFKA_TOPICS.DSG_LOG_TOPIC, logEvent);
}
```

日志加工规则：
- `extractBuildId("build-123-server")` → `"build-123"`
- `extractDomain("build-123-server")` → `"server"`
- 消息处理：`"Compiling TypeScript"` → `"[server] Compiling TypeScript"`

### 3.4 Server：日志入库 + 错误埋点

Server 端 Kafka 消费在 [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.controller.ts#L141-L145)：

```typescript
// build.controller.ts#L141-L145
@EventPattern(KAFKA_TOPICS.DSG_LOG_TOPIC)
async onDsgLog(@Payload() message: CodeGenerationLog.Value): Promise<void> {
  const logEntry = plainToInstance(CodeGenerationLog.Value, message);
  await this.buildService.onDsgLog(logEntry);
}
```

[build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L970-L1010) 实现入库 + 埋点：

```typescript
// build.service.ts#L970-L1010
public async onDsgLog(logEntry: CodeGenerationLog.Value): Promise<void> {
  // 1. 找到 GENERATE_APPLICATION Step
  const step = await this.getBuildStep(logEntry.buildId, GENERATE_STEP_NAME);
  
  // 2. 写入 ActionLog 表
  await this.actionService.logByStepId(
    step.id,
    ACTION_LOG_LEVEL[logEntry.level],
    logEntry.message
  );

  // 3. Error 级别额外触发 Segment 错误埋点
  if (ACTION_LOG_LEVEL[logEntry.level] === EnumActionLogLevel.Error) {
    const build = await this.prisma.build.findUnique({
      where: { id: logEntry.buildId },
      include: { createdBy: { include: { account: true } }, resource: { include: { project: true } } },
    });
    await this.analytics.trackManual({
      user: { accountId: build.createdBy.account.id, workspaceId: build.resource.project.workspaceId },
      data: {
        properties: {
          resourceId: build.resource.id,
          projectId: build.resource.project.id,
          message: logEntry.message,
        },
        event: EnumEventType.CodeGenerationError,
      },
    });
  }
}
```

ActionLog 写入底层在 [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts#L166-L183)：

```typescript
// action.service.ts#L166-L183
async logByStepId(stepId: string, level: EnumActionLogLevel, message: string, meta: JsonValue = {}): Promise<void> {
  await this.prisma.actionLog.create({
    data: {
      level,
      message,
      meta,
      step: { connect: { id: stepId } },
    },
    select: SELECT_ID,
  });
}
```

---

## 四、Package Manager 回调流程

### 4.1 触发条件

当代码生成全部子 Job 成功，且 `dsgResourceData.packages.length > 0` 且 `ENABLE_PACKAGE_MANAGER=true` 时，进入 Package Manager 阶段：

```typescript
// build-runner.service.ts#L366-L376
if (buildStatus === EnumJobStatus.Success) {
  const dsgResourceData = await this.buildJobsHandlerService.extractDsgResourceData(jobBuildId);
  
  if (dsgResourceData.packages?.length > 0 && this.enablePackageManager) {
    // 有包需要生成 → 进入 Package Manager 流程
    await this.generatePackages(buildId, resourceId, dsgResourceData);
  } else {
    // 无包 → 直接完成
    await this.codeGenerationAndPackagesCompleted(jobBuildId);
  }
}
```

### 4.2 发送包生成请求

```typescript
// build-runner.service.ts#L69-L88
async generatePackages(buildId: string, resourceId: string, dsgResourceData: DSGResourceData) {
  // 写日志：发送 N 个包待生成
  this.buildLoggerService.addCodeGenerationLog({
    buildId,
    message: `Sending ${dsgResourceData.packages?.length} package(s) for generation`,
    level: LogLevel.Info,
  });

  // 发送 Kafka 消息
  const requestPackagesEvent: PackageManagerCreateRequest.KafkaEvent = {
    key: null,
    value: { resourceId, buildId, dsgResourceData },
  };
  await this.producerService.emitMessage(
    KAFKA_TOPICS.PACKAGE_MANAGER_CREATE_REQUEST,
    requestPackagesEvent
  );
}
```

### 4.3 Package Manager 成功回调

Kafka 消费在 [build-runner.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L44-L54)：

```typescript
// build-runner.controller.ts#L44-L54
@EventPattern(KAFKA_TOPICS.PACKAGE_MANAGER_CREATE_SUCCESS)
async onPackageManagerCreateSuccess(@Payload() message: PackageManagerCreateSuccess.Value): Promise<void> {
  this.logger.info("Code package manager create success response received", { build: message.buildId });
  const args = plainToInstance(PackageManagerCreateSuccess.Value, message);
  await this.buildRunnerService.onPackageManagerCreateSuccess(args);
}
```

处理逻辑：包生成成功 = 整个代码生成 + 包生成都完成，直接发送 `CODE_GENERATION_SUCCESS_TOPIC`：

```typescript
// build-runner.service.ts#L54-L58
async onPackageManagerCreateSuccess(response: PackageManagerCreateSuccess.Value) {
  await this.codeGenerationAndPackagesCompleted(response.buildId);
}
```

`codeGenerationAndPackagesCompleted()` 发送最终成功事件：

```typescript
// build-runner.service.ts#L93-L107
async codeGenerationAndPackagesCompleted(buildIdOrJobBuildId: string) {
  const buildId = this.buildJobsHandlerService.extractBuildId(buildIdOrJobBuildId);
  const successEvent: CodeGenerationSuccess.KafkaEvent = {
    key: null,
    value: { buildId },
  };
  await this.producerService.emitMessage(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC, successEvent);
}
```

### 4.4 Package Manager 失败回调

```typescript
// build-runner.controller.ts#L56-L66
@EventPattern(KAFKA_TOPICS.PACKAGE_MANAGER_CREATE_FAILURE)
async onPackageManagerCreateFailure(@Payload() message: PackageManagerCreateFailure.Value): Promise<void> {
  const args = plainToInstance(PackageManagerCreateFailure.Value, message);
  this.logger.info("Code package manager create failure response received", { error: args.errorMessage });
  await this.buildRunnerService.onPackageManagerCreateFailure(args);
}
```

处理逻辑：包生成失败 = 整个构建失败，直接发送 `CODE_GENERATION_FAILURE_TOPIC`：

```typescript
// build-runner.service.ts#L60-L67
async onPackageManagerCreateFailure(response: PackageManagerCreateFailure.Value) {
  return this.emitCodeGenerationFailure(response.buildId, response.errorMessage);
}
```

> **注意**：Package Manager 阶段的失败**没有**多 Job 去重保护，因为此时代码生成已经全部完成，Package Manager 是单一阶段。

---

## 五、Git 推送 Step 完整状态流转

### 5.1 Step 创建：PUSH_TO_GIT_PROVIDER

当 Server 收到 `CODE_GENERATION_SUCCESS_TOPIC` 后，首先调用 `saveToGitProvider()`：

```typescript
// build.controller.ts#L87-L101
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC)
async onCodeGenerationSuccess(@Payload() message: CodeGenerationSuccess.Value): Promise<void> {
  const args = plainToInstance(CodeGenerationSuccess.Value, message);
  try {
    await this.buildService.saveToGitProvider(args.buildId);   // Step 1: Git 推送
    await this.buildService.onCodeGenerationSuccess(args.buildId); // Step 2: 完成代码生成 Step
  } catch (error) {
    this.logger.error("Failed to Complete Code Generation Step ", error, {...});
  }
}
```

`saveToGitProvider()` 在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1130-L1342) 中创建 `PUSH_TO_GIT_PROVIDER` Step：

```typescript
// build.service.ts#L1285-L1341
return this.actionService.run(
  build.actionId,
  PUSH_TO_GIT_STEP_NAME,                    // "PUSH_TO_GIT_PROVIDER"
  PUSH_TO_GIT_STEP_MESSAGE(gitProvider),    // "Push changes to GitHub"
  async (step) => {
    try {
      await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_START_LOG); // "Pull request creation job added to queue..."
      
      const createPullRequestMessage: CreatePrRequest.Value = {
        ...gitSettings,
        resourceId: resource.id,
        resourceName: kebabCase(resource.name),
        newBuildId: build.id,
        oldBuildId: oldBuild?.id,
        gitResourceMeta: {
          adminUIPath: serviceSettings?.adminUISettings?.adminUIPath,
          serverPath: serviceSettings?.serverSettings?.serverPath,
        },
        isBranchPerResource: branchPerResourceEntitlement?.hasAccess ?? false,
        overrideCustomizableFilesInGit: projectConfigurationSettings.overrideCustomizableFilesInGit ?? false,
      };

      const createPullRequestEvent: CreatePrRequest.KafkaEvent = {
        key: {
          resourceRepositoryId: kafkaEventKey,
          resourceId: createPullRequestMessage.isBranchPerResource ? resource.id : null,
        },
        value: createPullRequestMessage,
      };
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CREATE_PR_REQUEST_TOPIC,
        createPullRequestEvent
      );
    } catch (error) {
      logger.error("Failed to emit Create Pull Request Message.", error);
    }
  },
  true  // leaveStepOpenAfterSuccessfulExecution = true，异步等待 PR 结果回调
);
```

**关键分支**：如果资源没有配置 Git 仓库，则直接标记 Build 完成（无需 Git 推送）：

```typescript
// build.service.ts#L1227-L1234
if (!resourceRepository) {
  await this.updateBuildStatuses(
    build.id,
    EnumBuildStatus.Completed,
    EnumBuildGitStatus.NotConnected
  );
  return;  // 不创建 PUSH_TO_GIT Step，直接返回
}
```

此时状态：
- `ActionStep[GENERATE_APPLICATION]` 尚未完成（等待 onCodeGenerationSuccess()）
- `ActionStep[PUSH_TO_GIT_PROVIDER].status = Running`（如果配置了 Git）
- `Build.status = Running`
- `Build.gitStatus = Waiting`

### 5.2 Git 推送成功回调：状态入库

Kafka 消费在 [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.controller.ts#L117-L127)：

```typescript
// build.controller.ts#L117-L127
@EventPattern(KAFKA_TOPICS.CREATE_PR_SUCCESS_TOPIC)
async onPullRequestCreated(@Payload() message: CreatePrSuccess.Value): Promise<void> {
  try {
    const args = plainToInstance(CreatePrSuccess.Value, message);
    await this.buildService.onCreatePRSuccess(args);
  } catch (error) {
    this.logger.error(error.message, error);
  }
}
```

`onCreatePRSuccess()` 在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L851-L915) 中做完整状态入库：

```typescript
// build.service.ts#L851-L915
public async onCreatePRSuccess(response: CreatePrSuccess.Value): Promise<void> {
  const build = await this.findOne({ where: { id: response.buildId } });
  const steps = await this.actionService.getSteps(build.actionId);
  const step = steps.find((step) => step.name === PUSH_TO_GIT_STEP_NAME);

  try {
    // 1. 资源同步状态通知
    await this.resourceService.reportSyncMessage(build.resourceId, "Sync Completed Successfully");

    // 2. 更新代码行统计（新增/删除行数、变更文件数）
    const changes = await this.commitService.getChangesByResource(build.commitId, build.resourceId);
    if (changes.length > 0) {
      await this.updateBuildLOC(response.buildId, this.formatDiffStat(response.diffStat));
    }

    // 3. 写入 PR URL 和变更统计到 ActionLog
    await this.actionService.logInfo(step, response.url, {
      githubUrl: response.url,
      diffStat: this.formatDiffStat(response.diffStat),
    });

    // 4. 写入完成日志
    await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_FINISH_LOG(response.gitProvider));

    // 5. Layer 2: 标记 PUSH_TO_GIT Step = Success
    await this.actionService.complete(step, EnumActionStepStatus.Success);

    // 6. Layer 3: 更新 Build 最终状态
    await this.updateBuildStatuses(
      build.id,
      EnumBuildStatus.Completed,       // Build 完成
      EnumBuildGitStatus.Completed      // Git 推送完成
    );

    // 7. 计费上报：Git 推送次数
    const workspace = await this.resourceService.getResourceWorkspace(build.resourceId);
    await this.billingService.reportUsage(workspace.id, BillingFeature.CodePushToGit);

  } catch (error) {
    // 异常分支：即使 PR 业务成功，本地处理也可能失败
    await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_FAILED_LOG(response.gitProvider));
    await this.actionService.logInfo(step, error);
    await this.actionService.complete(step, EnumActionStepStatus.Failed);
    await this.updateBuildStatuses(build.id, EnumBuildStatus.Failed, EnumBuildGitStatus.Failed);
    await this.resourceService.reportSyncMessage(build.resourceId, `Error: ${error}`);
  }
}
```

成功回调后最终状态：
- `Build.status = Completed`
- `Build.gitStatus = Completed`
- `Build.linesOfCodeAdded/Deleted/filesChanged` 已更新
- `ActionStep[PUSH_TO_GIT_PROVIDER].status = Success`
- `ActionStep[GENERATE_APPLICATION]` 已由 `onCodeGenerationSuccess()` 标记 Success

### 5.3 Git 推送失败回调：状态入库

```typescript
// build.controller.ts#L129-L139
@EventPattern(KAFKA_TOPICS.CREATE_PR_FAILURE_TOPIC)
async onPullRequestFailure(@Payload() message: CreatePrFailure.Value): Promise<void> {
  try {
    const args = plainToInstance(CreatePrFailure.Value, message);
    await this.buildService.onCreatePRFailure(args);
  } catch (error) {
    this.logger.error(error.message, error);
  }
}
```

`onCreatePRFailure()` 在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L917-L968)：

```typescript
// build.service.ts#L917-L968
public async onCreatePRFailure(response: CreatePrFailure.Value): Promise<void> {
  const build = await this.prisma.build.findUnique({
    where: { id: response.buildId },
    include: { createdBy: { include: { account: true } }, resource: { include: { project: true } } },
  });

  const steps = await this.actionService.getSteps(build.actionId);
  const step = steps.find((step) => step.name === PUSH_TO_GIT_STEP_NAME);

  // 1. 资源同步状态通知（错误）
  await this.resourceService.reportSyncMessage(build.resourceId, `Error: ${response.errorMessage}`);

  // 2. 写入失败日志
  await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_FAILED_LOG(response.gitProvider));
  await this.actionService.log(step, EnumActionLogLevel.Error, response.errorMessage);

  // 3. Layer 2: 标记 PUSH_TO_GIT Step = Failed
  await this.actionService.complete(step, EnumActionStepStatus.Failed);

  // 4. Layer 3: 标记 Build + Git 双失败
  await this.updateBuildStatuses(
    build.id,
    EnumBuildStatus.Failed,
    EnumBuildGitStatus.Failed
  );

  // 5. Segment 错误埋点
  await this.analytics.trackManual({
    user: { accountId: build.createdBy.account.id, workspaceId: build.resource.project.workspaceId },
    data: {
      properties: { resourceId: build.resource.id, projectId: build.resource.project.id, message: response.errorMessage },
      event: EnumEventType.GitSyncError,
    },
  });
}
```

失败回调后最终状态：
- `Build.status = Failed`
- `Build.gitStatus = Failed`
- `ActionStep[PUSH_TO_GIT_PROVIDER].status = Failed`

### 5.4 代码生成 Step 完成（与 Git 并行）

在 `saveToGitProvider()` 创建 Git Step 的同时，`onCodeGenerationSuccess()` 完成代码生成 Step：

```typescript
// build.service.ts#L450-L495
async onCodeGenerationSuccess(buildId: string): Promise<void> {
  const step = await this.getBuildStep(buildId, GENERATE_STEP_NAME);
  
  // 发送 USER_BUILD_TOPIC 通知用户侧
  this.kafkaProducerService.emitMessage(KAFKA_TOPICS.USER_BUILD_TOPIC, <UserBuild.KafkaEvent>{
    key: {},
    value: {
      commitId: commitWithAccount.commit.id,
      resourceId: commitWithAccount.resourceId,
      buildId: buildId,
      workspaceId: commitWithAccount.commit.project.workspaceId,
      // ...
    },
  }).catch(error => this.logger.error(`Failed to queue user build ${buildId}`, error));

  // Layer 2: 标记 GENERATE_APPLICATION Step = Success
  await this.actionService.complete(step, EnumActionStepStatus.Success);
}
```

> **注意**：此处仅完成 `GENERATE_APPLICATION` Step，**不修改 Build.status**。Build 的最终状态（Completed/Failed）由 Git 推送回调（或无 Git 场景下的 NotConnected 分支）决定。

---

## 六、最终构建状态入库总览

### 6.1 updateBuildStatuses() 统一入口

所有 Build 状态修改都通过 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L497-L509) 的统一方法：

```typescript
// build.service.ts#L497-L509
async updateBuildStatuses(
  buildId: string,
  status: EnumBuildStatus | undefined,
  gitStatus?: EnumBuildGitStatus | undefined
): Promise<void> {
  await this.prisma.build.update({
    where: { id: buildId },
    data: { status, gitStatus },
  });
}
```

### 6.2 状态矩阵：所有场景的最终状态

| 场景 | Build.status | Build.gitStatus | 触发点 |
|------|-------------|-----------------|--------|
| Build 创建 | Running | Waiting | create() |
| 代码生成失败 | Failed | Canceled | onCodeGenerationFailure() |
| 代码生成成功 + 无 Git 仓库 | Completed | NotConnected | saveToGitProvider() 内分支 |
| Git 推送成功 | Completed | Completed | onCreatePRSuccess() |
| Git 推送失败 | Failed | Failed | onCreatePRFailure() / onCreatePRSuccess() catch |
| 私钥插件下载失败 | Failed | Canceled | onDownloadPrivatePluginFailure() |
| Git 推送过程中 saveToGitProvider() 本地异常 | 见 catch 分支 | 见 catch 分支 | onCreatePRSuccess() catch |

### 6.3 Step 状态汇总

| Step Name | 创建时机 | 完成时机 | 状态 |
|-----------|---------|---------|------|
| ADD_TO_QUEUE | Build.create() | 创建即完成 | Success |
| DOWNLOAD_PRIVATE_PLUGINS | 有私钥插件时 | 下载成功/失败 Kafka 回调 | Success / Failed |
| GENERATE_APPLICATION | generate() | CODE_GENERATION_SUCCESS/FAILURE Kafka 回调 | Success / Failed |
| PUSH_TO_GIT_PROVIDER | 代码生成成功且有 Git 配置时 | CREATE_PR_SUCCESS/FAILURE Kafka 回调 | Success / Failed |

---

## 七、失败去重机制深度解析

### 7.1 去重问题背景

一个 Build 可能被拆分为 2 个子 Job（Server + AdminUI），两个 Job 并发执行。如果两个 Job 都失败，会触发两次失败回调。如果没有去重保护：
- `CODE_GENERATION_FAILURE_TOPIC` 会被发送两次
- Server 端 `onCodeGenerationFailure()` 会执行两次
- ActionLog 会重复写入错误日志
- 下游监听器（通知、计费等）会被触发两次

### 7.2 去重核心机制：otherJobsHaveNotFailed 标志

去重逻辑在两个关键位置实现，模式一致：

#### 位置 A：DSG 执行失败回调

```typescript
// build-runner.service.ts#L257-L277
async emitCodeGenerationFailureWhenJobStatusFailed(jobBuildId: string) {
  let otherJobsHaveNotFailed = true;          // 默认假设其他 Job 还没失败
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  try {
    // 关键：在标记当前 Job 为 Failure 之前，先读取 Redis 中的聚合状态
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    // 只要已经有任一 Job = Failure，聚合状态就是 Failure，说明已有人先失败了
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // 再标记当前 Job 为 Failure
    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Failure);
  } catch (error) {
    this.logger.error(error.message, error);
  } finally {
    // 只有"我是第一个失败者"才发送失败事件
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId);
    }
  }
}
```

#### 位置 B：DSG 成功回调中的异常分支（产物复制失败等）

```typescript
// build-runner.service.ts#L344-L383
async handleDsgJobCompleted(resourceId: string, jobBuildId: string) {
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  let otherJobsHaveNotFailed = true;

  try {
    // 同样：先读，后写，判断是否已有人失败
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    await this.copyFromJobToArtifact(resourceId, jobBuildId);  // 这里可能抛异常

    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Success);
    // ...
  } catch (error) {
    this.logger.error(error.message, error);
    if (otherJobsHaveNotFailed) {  // 只有第一个失败的才发事件
      await this.emitCodeGenerationFailure(buildId, error.message);
    }
  }
}
```

### 7.3 去重状态机时序

以 Server Job 先失败、AdminUI Job 后失败为例：

```
时间轴 ──────────────────────────────────────────────────────────────>

T0: Redis = { "build-123-server": "in-progress", "build-123-admin-ui": "in-progress" }
    getBuildStatus() = InProgress

T1: [Server Job] 失败回调
    ① getBuildStatus() → InProgress → otherJobsHaveNotFailed = true
    ② setJobStatus("build-123-server", Failure)
    Redis = { "build-123-server": "failure", "build-123-admin-ui": "in-progress" }
    ③ otherJobsHaveNotFailed = true → emit CODE_GENERATION_FAILURE_TOPIC ✓

T2: [AdminUI Job] 失败回调
    ① getBuildStatus() → Failure → otherJobsHaveNotFailed = false
    ② setJobStatus("build-123-admin-ui", Failure)
    Redis = { "build-123-server": "failure", "build-123-admin-ui": "failure" }
    ③ otherJobsHaveNotFailed = false → 跳过 emit ✗ （去重成功）
```

### 7.4 去重边界情况分析

| 场景 | 去重效果 | 原因 |
|------|---------|------|
| Server 失败 → AdminUI 失败 | ✅ 仅 1 次事件 | Server 先标记，AdminUI 读取时已是 Failure |
| AdminUI 失败 → Server 失败 | ✅ 仅 1 次事件 | 同上，顺序无关 |
| 两个 Job 同时失败（极端并发） | ⚠️ 理论可能双发 | 两者都先读到 InProgress，然后都认为自己是第一个 |
| runBuild() 同步异常（作业提交前） | ✅ 仅 1 次事件 | 此时尚未拆分 Job，无并发问题 |
| Package Manager 失败 | N/A（无去重） | 单阶段，不存在多 Job 并发 |
| Git 推送失败 | N/A（无去重） | 单阶段，Server 端直接处理 |

> **并发风险**：Redis 的 `get` + `set` 不是原子操作。在极高并发下，两个 Job 可能都在对方 `setJobStatus()` 之前完成 `getBuildStatus()`，导致都认为自己是第一个失败者。生产环境中由于 DSG 作业执行时间较长（分钟级），这种竞态概率极低。如需严格保证，可改用 Redis Lua 脚本或 `SETNX` 实现分布式锁。

### 7.5 去重保护范围汇总

```
阶段                      是否有去重保护
──────────────────────────────────────────
runBuild() 同步异常            ❌（无需，单线程）
DSG 执行失败回调                ✅（otherJobsHaveNotFailed）
产物复制阶段异常                ✅（otherJobsHaveNotFailed）
Package Manager 失败            ❌（单阶段，无需）
Git 推送失败                    ❌（单阶段，无需）
onCodeGenerationFailure() 服务端  ❌（依赖上游去重）
```

---

## 八、完整状态回写时序图

```
┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌───────┐  ┌────────────┐  ┌────────────┐
│   User   │  │  Server  │  │ Build Manager│  │ Kafka │  │ DSG Runner │  │   Git Svc  │
└────┬─────┘  └────┬─────┘  └──────┬───────┘  └───┬───┘  └─────┬──────┘  └─────┬──────┘
     │ Trigger Build│              │              │             │               │
     │─────────────>│              │              │             │               │
     │              │ DB: Build.create()           │             │               │
     │              │ status=Running, gitStatus=Waiting           │               │
     │              │ Step: ADD_TO_QUEUE=Success  │             │               │
     │              │ Step: GENERATE_APPLICATION=Running        │               │
     │              │ save DSGResourceData to FS  │             │               │
     │              │ emit CODE_GENERATION_REQUEST─────────────>│               │
     │              │              │<──consume─────│             │               │
     │              │              │ read DSGResourceData from FS               │
     │              │              │ split Jobs: Server + AdminUI               │
     │              │              │ Redis: set both=InProgress │               │
     │              │              │ POST DSG_RUNNER_URL ×2 ───────────────────>│
     │              │              │              │             │ 执行代码生成    │
     │              │              │<─────────────────────────────POST /build-logger/create-log
     │              │              │ addCodeGenerationLog()     │               │
     │              │              │ emit DSG_LOG_TOPIC────────>│               │
     │              │<─consume─────│              │             │               │
     │              │ DB: ActionLog.insert        │             │               │
     │              │              │<───────callback success/failure─────────────│
     │              │              │ copy artifacts             │               │
     │              │              │ Redis: set Job=Success     │               │
     │              │              │ aggregate: InProgress? → wait              │
     │              │              │              │             │               │
     │              │              │<───────callback success/failure─────────────│
     │              │              │ copy artifacts             │               │
     │              │              │ Redis: set Job=Success     │               │
     │              │              │ aggregate: all Success?    │               │
     │              │              │ has packages? → emit PACKAGE_MANAGER_CREATE_REQUEST──>│
     │              │              │              │             │               │
     │              │              │<──consume PACKAGE_MANAGER_CREATE_SUCCESS───│（或无包跳过）
     │              │              │ emit CODE_GENERATION_SUCCESS ─────────────>│
     │              │<─consume─────│              │             │               │
     │              │ saveToGitProvider()         │             │               │
     │              │ Step: PUSH_TO_GIT=Running (if has Git)   │               │
     │              │ emit CREATE_PR_REQUEST ─────────────────────────────────>│
     │              │ Step: GENERATE_APPLICATION=Success       │               │
     │              │ emit USER_BUILD_TOPIC ────>│             │               │
     │              │              │              │             │ 创建 PR/Push   │
     │              │<─consume CREATE_PR_SUCCESS ───────────────────────────────│
     │              │ DB: update LOC stats        │             │               │
     │              │ Step: PUSH_TO_GIT=Success   │             │               │
     │              │ DB: Build.status=Completed  │             │               │
     │              │ DB: Build.gitStatus=Completed            │               │
     │              │ report billing: CodePushToGit            │               │
     │  Notify Done │              │              │             │               │
     │<─────────────│              │              │             │               │
```

---

## 九、关键设计要点

### 9.1 三层状态隔离设计
- **Layer 1 Redis**：细粒度子 Job 状态，支持快速聚合判断，无需查库
- **Layer 2 ActionStep**：面向用户的步骤进度，每个 Step 有独立生命周期和日志
- **Layer 3 Build**：最终持久化状态，对外 API 展示和后续流程依赖

### 9.2 异步 Step 的 leaveStepOpen 模式
`actionService.run()` 的第 5 个参数 `leaveStepOpenAfterSuccessfulExecution=true` 使 Step 在同步回调完成后仍保持 Running，等待未来的 Kafka 事件最终完成。这是长时间异步任务的标准处理模式。

### 9.3 日志的 domain 前缀区分
Build Manager 的 `addCodeGenerationLog()` 对拆分作业的日志自动添加 `[server]`/`[admin-ui]` 前缀，用户在 UI 上可清晰区分日志来源。

### 9.4 失败去重的"先读后写"模式
通过在修改状态前先读取聚合状态，用局部变量 `otherJobsHaveNotFailed` 作为是否发送事件的依据，避免了多 Job 并发失败时的重复通知。虽非严格原子（依赖 Redis 非事务操作），但在 DSG 分钟级执行时间下实际足够可靠。

### 9.5 Build 最终状态的"双轴"设计
`Build.status`（代码生成结果）和 `Build.gitStatus`（Git 推送结果）独立维护，支持部分成功场景：例如代码生成成功但 Git 推送失败，用户可以下载代码产物但无法同步到 Git。
