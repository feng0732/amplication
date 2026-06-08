# Amplication Action Runner & Build Executor 状态回写深度分析

## 一、整体架构与状态模型

### 1.1 核心服务与职责

| 服务 | 模块 | 职责 |
|------|------|------|
| `amplication-server` | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts) | Build 创建、Action/Step 管理、最终状态入库、Git 推送编排、Build Stale 兜底 |
| `amplication-server` | [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts) | Step 生命周期（create/complete/log） |
| `amplication-server` | [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.controller.ts) | Kafka 事件消费者（成功/失败/日志/PR） |
| `amplication-build-manager` | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | DSG 执行、Package Manager 编排、状态聚合、失败去重 |
| `amplication-build-manager` | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | 子作业拆分、Redis 状态读写、Job ID 编解码 |
| `amplication-build-manager` | [build-logger.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-logger/build-logger.service.ts) | DSG 日志转发（带 domain 前缀） |
| `amplication-build-manager` | [redis.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/redis/redis.service.ts) | Job 状态缓存层 |

### 1.2 三层状态模型

系统采用 **三层状态** 分层管理：

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Layer 3: PostgreSQL (Build 表) — 构建最终状态                             │
│  status: Running | Completed | Failed | Invalid | Unknown | Canceled       │
│  gitStatus: NotConnected | Waiting | Completed | Failed | Canceled        │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  Layer 2: PostgreSQL (ActionStep 表) — 用户可见步骤进度              │    │
│  │  status: Waiting | Running | Failed | Success                       │    │
│  │  Step 名称:                                                          │    │
│  │    · ADD_TO_QUEUE                                                    │    │
│  │    · DOWNLOAD_PRIVATE_PLUGINS (可选)                                 │    │
│  │    · GENERATE_APPLICATION                                            │    │
│  │    · PUSH_TO_GIT_PROVIDER (可选)                                     │    │
│  │  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │  │  Layer 1: Redis — 细粒度子 Job 状态（仅 Build Manager 内部用）  │    │    │
│  │  │  status: in-progress | success | failure                     │    │    │
│  │  │  Key: buildId, Value: { "buildId-server": status,            │    │    │
│  │  │                            "buildId-admin-ui": status }      │    │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  └────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

各层状态枚举定义：

- Build 状态：[EnumBuildStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/dto/EnumBuildStatus.ts)
- Build Git 状态：[EnumBuildGitStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/dto/EnumBuildGitStatus.ts)
- Action Step 状态：[EnumActionStepStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/dto/EnumActionStepStatus.ts)
- Job 状态：[types.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/types.ts)

### 1.3 Build.status 与 Build.gitStatus 的职责边界核准

基于对所有 `updateBuildStatuses()` 调用点的代码审计，两者职责边界如下：

| 字段 | 职责定位 | 语义说明 |
|------|---------|---------|
| **Build.status** | **构建整体结果的最终状态** | 由代码生成结果 + Git 推送结果（如有）共同决定，是对外 API 返回的主状态 |
| **Build.gitStatus** | **Git 推送专属子状态** | 只反映 Git 环节本身，与代码生成解耦 |

具体取值的语义：

**Build.status**：
- `Running`：构建尚未结束（代码生成中 / Git 推送中 / 等待回调）
- `Completed`：全流程成功（代码生成 + Git 推送都成功；或代码生成成功但未配置 Git）
- `Failed`：任一关键环节失败（代码生成失败 / Git 推送失败 / 插件下载失败 / 超时 stale）
- `Invalid`、`Unknown`、`Canceled`：历史兼容或异常路径，正常流程不产出

**Build.gitStatus**：
- `Waiting`：Git 环节尚未开始（代码生成阶段 / Git 请求已发送等待回调）
- `NotConnected`：资源未配置 Git 仓库（无需 Git 推送）
- `Completed`：Git 推送成功
- `Failed`：Git 推送失败
- `Canceled`：Git 推送被取消（因上游环节失败，Git 环节从未触发）

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
      status: EnumBuildStatus.Running,        // Layer 3: Build.status = Running
      gitStatus: EnumBuildGitStatus.Waiting,   // Layer 3: Build.gitStatus = Waiting
      entityVersions: { connect: latestEntityVersions.map(v => ({ id: v.id })) },
      action: {
        create: {
          steps: {
            create: createInitialStepData(version, args.data.message), // Layer 2: 初始 Step
          },
        },
      },
    },
    include: { commit: true, resource: true },
  });
  // ...
  // 有私钥插件 → 先下载插件；无插件 → 直接进入 generate()
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
    status: EnumActionStepStatus.Success,   // 立即标记 Success（瞬时操作）
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
      
      // 轻量消息：只传 resourceId + buildId，大对象通过共享文件系统传递
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
     │  - extractBuildId(): 去除 domain 后缀
     │  - 若为子 Job 日志，message 加 "[server]"/"[admin-ui]" 前缀
     ▼
Kafka: DSG_LOG_TOPIC  (key = { buildId })
     │
     ▼
Server [build.controller.ts] onDsgLog()
     │  actionService.logByStepId() → ActionLog 表
     │  Error 级别额外触发 Segment 埋点
     ▼
PostgreSQL ActionLog + Segment Analytics
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
  buildId: string;  // 可能是 buildId 或 jobBuildId（buildId-server / buildId-admin-ui）
}
```

### 3.3 Build Manager：日志加工与转发

[build-logger.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-logger/build-logger.service.ts#L16-L38) 对日志进行加工：

```typescript
// build-logger.service.ts#L16-L38
async addCodeGenerationLog(logEntry: CodeGenerationLogRequestDto): Promise<void> {
  const buildId = this.buildJobsHandlerService.extractBuildId(logEntry.buildId);

  // 子 Job 日志自动加 domain 前缀："Compiling" → "[server] Compiling"
  if (buildId !== logEntry.buildId) {
    const domain = this.buildJobsHandlerService.extractDomain(logEntry.buildId);
    logEntry.message = `[${domain}] ${logEntry.message}`;
  }

  const logEvent: CodeGenerationLog.KafkaEvent = {
    key: { buildId },
    value: { ...logEntry, buildId },
  };
  await this.producerService.emitMessage(KAFKA_TOPICS.DSG_LOG_TOPIC, logEvent);
}
```

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

  // 3. Error 级别额外触发 Segment 埋点（CodeGenerationError）
  if (ACTION_LOG_LEVEL[logEntry.level] === EnumActionLogLevel.Error) {
    // ... analytics.trackManual()
  }
}
```

ActionLog 写入底层在 [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts#L166-L183)：

```typescript
// action.service.ts#L166-L183
async logByStepId(stepId: string, level: EnumActionLogLevel, message: string, meta: JsonValue = {}): Promise<void> {
  await this.prisma.actionLog.create({
    data: { level, message, meta, step: { connect: { id: stepId } } },
    select: SELECT_ID,
  });
}
```

---

## 四、Package Manager 回调流程

### 4.1 触发条件

当所有子 Job 代码生成成功，且 `dsgResourceData.packages.length > 0` 且 `ENABLE_PACKAGE_MANAGER=true` 时，进入 Package Manager 阶段：

```typescript
// build-runner.service.ts#L366-L376
if (buildStatus === EnumJobStatus.Success) {
  const dsgResourceData = await this.buildJobsHandlerService.extractDsgResourceData(jobBuildId);
  
  if (dsgResourceData.packages?.length > 0 && this.enablePackageManager) {
    await this.generatePackages(buildId, resourceId, dsgResourceData);  // 有包 → 调 PM
  } else {
    await this.codeGenerationAndPackagesCompleted(jobBuildId);          // 无包 → 直接完成
  }
}
```

### 4.2 发送包生成请求

```typescript
// build-runner.service.ts#L69-L88
async generatePackages(buildId: string, resourceId: string, dsgResourceData: DSGResourceData) {
  this.buildLoggerService.addCodeGenerationLog({
    buildId,
    message: `Sending ${dsgResourceData.packages?.length} package(s) for generation`,
    level: LogLevel.Info,
  });

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

```typescript
// build-runner.controller.ts#L44-L54
@EventPattern(KAFKA_TOPICS.PACKAGE_MANAGER_CREATE_SUCCESS)
async onPackageManagerCreateSuccess(@Payload() message: PackageManagerCreateSuccess.Value): Promise<void> {
  const args = plainToInstance(PackageManagerCreateSuccess.Value, message);
  await this.buildRunnerService.onPackageManagerCreateSuccess(args);
}
```

处理逻辑：PM 成功 = 代码生成 + 包生成都完成，发送 `CODE_GENERATION_SUCCESS_TOPIC`：

```typescript
// build-runner.service.ts#L54-L58
async onPackageManagerCreateSuccess(response: PackageManagerCreateSuccess.Value) {
  await this.codeGenerationAndPackagesCompleted(response.buildId);
}
```

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
  await this.buildRunnerService.onPackageManagerCreateFailure(args);
}
```

处理逻辑：PM 失败 = 整个构建失败：

```typescript
// build-runner.service.ts#L60-L67
async onPackageManagerCreateFailure(response: PackageManagerCreateFailure.Value) {
  return this.emitCodeGenerationFailure(response.buildId, response.errorMessage);
}
```

> **注意**：Package Manager 阶段**没有**多 Job 去重保护，因为此时代码生成已全部完成，PM 是单一阶段。

---

## 五、Git 推送 Step 状态流转（代码核准版）

### 5.1 调用链入口

Server 收到 `CODE_GENERATION_SUCCESS_TOPIC` 后，在 [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.controller.ts#L87-L101) 顺序执行：

```typescript
// build.controller.ts#L87-L101
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC)
async onCodeGenerationSuccess(@Payload() message: CodeGenerationSuccess.Value): Promise<void> {
  const args = plainToInstance(CodeGenerationSuccess.Value, message);
  try {
    await this.buildService.saveToGitProvider(args.buildId);     // 先 Git 推送
    await this.buildService.onCodeGenerationSuccess(args.buildId); // 再完成 GENERATE Step
  } catch (error) {
    // ⚠️ 注意：外层 try-catch 捕获的是 saveToGitProvider 本身抛出的异常
    // 但 saveToGitProvider 内部的 CREATE_PR_REQUEST emit 异常被内部吞掉，不会走到这里
    this.logger.error("Failed to Complete Code Generation Step ", error, {...});
  }
}
```

### 5.2 Step 创建：PUSH_TO_GIT_PROVIDER

`saveToGitProvider()` 在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1130-L1342) 中创建 Step：

**分支 A：资源未配置 Git 仓库**

```typescript
// build.service.ts#L1227-L1234
if (!resourceRepository) {
  // 无 Git → 直接完成 Build，不创建 PUSH_TO_GIT Step
  await this.updateBuildStatuses(
    build.id,
    EnumBuildStatus.Completed,
    EnumBuildGitStatus.NotConnected
  );
  return;
}
```

**分支 B：有 Git 配置 → 创建 PUSH_TO_GIT Step**

```typescript
// build.service.ts#L1285-L1341
return this.actionService.run(
  build.actionId,
  PUSH_TO_GIT_STEP_NAME,                    // "PUSH_TO_GIT_PROVIDER"
  PUSH_TO_GIT_STEP_MESSAGE(gitProvider),    // "Push changes to GitHub"
  async (step) => {
    try {
      await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_START_LOG);
      // ... 组装 createPullRequestMessage

      const createPullRequestEvent: CreatePrRequest.KafkaEvent = {
        key: { resourceRepositoryId: kafkaEventKey, resourceId: ... },
        value: createPullRequestMessage,
      };
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CREATE_PR_REQUEST_TOPIC,
        createPullRequestEvent
      );
    } catch (error) {
      // ⚠️ 关键代码行为核准：内部 catch，只打日志，不向外抛出异常
      logger.error("Failed to emit Create Pull Request Message.", error);
    }
  },
  true  // leaveStepOpenAfterSuccessfulExecution = true
);
```

### 5.3 代码核准：CREATE_PR_REQUEST 发送失败后的完整状态链路

**核心结论**：CREATE_PR_REQUEST 发送失败后，**两个 Step 的状态不同**：
- `GENERATE_APPLICATION` → **正常完成（Success）**
- `PUSH_TO_GIT_PROVIDER` → **永久挂起（Running）**

原因是：`saveToGitProvider()` 内部吞掉异常后**正常返回**，`build.controller.ts` 继续执行后续的 `onCodeGenerationSuccess()`。

#### 5.3.1 PUSH_TO_GIT_PROVIDER Step 挂起推演

推演 `actionService.run()` 的执行路径 [action.service.ts#L275-L295](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts#L275-L295)：

```typescript
// action.service.ts#L275-L295
async run<T>(
  actionId: string, stepName: string, message: string,
  stepFunction: (step: ActionStep) => Promise<T>,
  leaveStepOpenAfterSuccessfulExecution = false
): Promise<T> {
  const step = await this.createStep(actionId, stepName, message);  // ① Step.status = Running
  try {
    const result = await stepFunction(step);   // ② 执行用户回调
    if (!leaveStepOpenAfterSuccessfulExecution) {
      await this.complete(step, EnumActionStepStatus.Success);  // ③ leaveStepOpen=true → 不执行
    }
    return result;   // ④ stepFunction 正常返回 → run() 正常返回，不抛异常
  } catch (error) {
    // ⑤ 用户回调内部已吞掉异常，不会走到这里
    await this.log(step, EnumActionLogLevel.Error, error.message);
    await this.complete(step, EnumActionStepStatus.Failed);
    throw error;
  }
}
```

**PUSH_TO_GIT_PROVIDER Step 挂起链路**：

| 步骤 | 实际行为 | 结果 |
|------|---------|------|
| ① | `createStep()` 创建 PUSH_TO_GIT Step | `Step.status = Running` |
| ② | `stepFunction()` 执行，内部 `emitMessage()` 抛异常，被内部 try-catch 捕获，只打日志，**不向外 throw** | stepFunction **正常返回 undefined**（无异常） |
| ③ | `leaveStepOpen = true`，跳过 `complete(Success)` | Step 未被标记 Success |
| ④ | `run()` 正常返回（无异常）→ `saveToGitProvider()` **正常返回** | Step 未被标记 Failed |
| — | **PUSH_TO_GIT Step 最终** | `Running`（永久挂起） |

#### 5.3.2 GENERATE_APPLICATION Step 仍会正常完成

关键代码在 [build.controller.ts#L87-L101](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.controller.ts#L87-L101)：

```typescript
// build.controller.ts#L87-L101
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC)
async onCodeGenerationSuccess(@Payload() message: CodeGenerationSuccess.Value): Promise<void> {
  const args = plainToInstance(CodeGenerationSuccess.Value, message);
  try {
    await this.buildService.saveToGitProvider(args.buildId);      // ① 先 Git 推送（内部吞异常后正常返回）
    await this.buildService.onCodeGenerationSuccess(args.buildId);  // ② 再完成 GENERATE Step（① 正常返回，所以② 一定会执行）
  } catch (error) {
    // saveToGitProvider 内部异常已被吞，不会向外抛 → 外层 catch 不会触发
    this.logger.error("Failed to Complete Code Generation Step ", error, {...});
  }
}
```

**GENERATE_APPLICATION Step 完成链路**：

| 步骤 | 实际行为 | 结果 |
|------|---------|------|
| ① | `saveToGitProvider()` 执行，内部 CREATE_PR_REQUEST 发送失败被吞 → 正常返回 | 无异常抛出 |
| ② | `onCodeGenerationSuccess()` 被调用 → `actionService.complete(step, Success)` | `GENERATE_APPLICATION` Step = **Success** |
| ③ | 同时发送 `USER_BUILD_TOPIC` 通知用户侧 | 用户侧收到构建完成通知 |
| — | **GENERATE_APPLICATION Step 最终** | `Success`（正常完成） |

#### 5.3.3 CREATE_PR_REQUEST 发送失败后的最终状态全景

| 对象 | 最终状态 | 归类 |
|------|---------|------|
| ActionStep[GENERATE_APPLICATION] | **Success** | ✅ 正常完成 |
| ActionStep[PUSH_TO_GIT_PROVIDER] | **Running** | ⚠️ 挂起（永久） |
| Build.status | **Running** | 未决（因 PUSH_TO_GIT 仍 Running） |
| Build.gitStatus | **Waiting** | 未决（Git 请求未成功发出） |
| 用户侧通知 | 已发送 USER_BUILD_TOPIC | 可能误导用户以为构建成功 |

#### 5.3.4 补救机制：Stale 兜底

只有当 `calcBuildStatus()` 被调用，且 `isBuildStale()` 判断创建时间超过 5 小时时，才会兜底标记为 Failed：

```typescript
// build.service.ts#L1547-L1561
isBuildStale(build: Build): boolean {
  if (build.status === EnumBuildStatus.Running) {
    const stalePeriod = STALE_BUILD_HOURS * 60 * 60 * 1000;  // 5 小时
    if (Date.now() - build.createdAt.getTime() > stalePeriod) {
      return true;
    }
  }
  return false;
}
```

Stale 兜底后状态：
- `Build.status = Failed`
- `Build.gitStatus = Failed`
- **但两个 ActionStep 状态不变**：GENERATE_APPLICATION 仍为 Success，PUSH_TO_GIT_PROVIDER 仍为 Running

### 5.4 Git 推送成功回调：状态入库

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
    // 1. 资源同步状态通知（Success）
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
      EnumBuildStatus.Completed,       // Build.status = Completed
      EnumBuildGitStatus.Completed      // Build.gitStatus = Completed
    );

    // 7. 计费上报：CodePushToGit 次数
    await this.billingService.reportUsage(workspace.id, BillingFeature.CodePushToGit);

  } catch (error) {
    // ⚠️ Git 业务操作成功，但本地处理（LOC统计/计费）异常 → 整体标记失败
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
- `ActionStep[PUSH_TO_GIT_PROVIDER].status = Success`
- `ActionStep[GENERATE_APPLICATION].status = Success`（由 onCodeGenerationSuccess() 标记）

### 5.5 Git 推送失败回调：状态入库

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
  const build = await this.prisma.build.findUnique({ where: { id: response.buildId }, include: {...} });
  const steps = await this.actionService.getSteps(build.actionId);
  const step = steps.find((step) => step.name === PUSH_TO_GIT_STEP_NAME);

  // 1. 资源同步状态通知（Error）
  await this.resourceService.reportSyncMessage(build.resourceId, `Error: ${response.errorMessage}`);

  // 2. 写入失败日志
  await this.actionService.logInfo(step, PUSH_TO_GIT_STEP_FAILED_LOG(response.gitProvider));
  await this.actionService.log(step, EnumActionLogLevel.Error, response.errorMessage);

  // 3. Layer 2: 标记 PUSH_TO_GIT Step = Failed
  await this.actionService.complete(step, EnumActionStepStatus.Failed);

  // 4. Layer 3: 双失败
  await this.updateBuildStatuses(
    build.id,
    EnumBuildStatus.Failed,
    EnumBuildGitStatus.Failed
  );

  // 5. Segment 埋点：GitSyncError
  await this.analytics.trackManual({...});
}
```

失败回调后最终状态：
- `Build.status = Failed`
- `Build.gitStatus = Failed`
- `ActionStep[PUSH_TO_GIT_PROVIDER].status = Failed`

### 5.6 代码生成 Step 完成（与 Git 并行）

`onCodeGenerationSuccess()` 在 `saveToGitProvider()` 返回后执行，仅完成 GENERATE_APPLICATION Step：

```typescript
// build.service.ts#L450-L495
async onCodeGenerationSuccess(buildId: string): Promise<void> {
  const step = await this.getBuildStep(buildId, GENERATE_STEP_NAME);
  
  // 发送 USER_BUILD_TOPIC 通知用户侧
  this.kafkaProducerService.emitMessage(KAFKA_TOPICS.USER_BUILD_TOPIC, <UserBuild.KafkaEvent>{...});

  // Layer 2: 标记 GENERATE_APPLICATION Step = Success
  await this.actionService.complete(step, EnumActionStepStatus.Success);
  
  // ⚠️ 关键核准：此处**不修改 Build.status**！Build 最终状态由 Git 回调（或 NotConnected 分支）决定
}
```

---

## 六、最终构建状态入库与状态矩阵（代码核准版）

### 6.1 updateBuildStatuses() 统一入口

所有 Build 状态修改都通过 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L497-L509)：

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

### 6.2 状态矩阵：全部 11 个调用点核准

基于代码中所有 `updateBuildStatuses()` 调用点的完整审计：

| # | 场景 | Build.status | Build.gitStatus | GENERATE Step | PUSH_TO_GIT Step | 代码来源 |
|---|------|-------------|-----------------|---------------|------------------|---------|
| 1 | Build 创建初始化 | `Running` | `Waiting` | `Running` | 未创建 | [L291](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L291) |
| 2 | 代码生成失败 | `Failed` | `Canceled` | `Failed` | 未创建 | [L529](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L529) |
| 3 | Git 推送成功回调 | `Completed` | `Completed` | `Success` | `Success` | [L885](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L885) |
| 4 | Git 推送成功但本地处理异常 | `Failed` | `Failed` | `Success` | `Failed` | [L905](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L905) |
| 5 | Git 推送失败回调 | `Failed` | `Failed` | `Success` | `Failed` | [L948](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L948) |
| 6 | 插件下载失败 | `Failed` | `Canceled` | 未创建 | 未创建 | [L1076](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1076) |
| 7 | 代码生成成功但未配置 Git | `Completed` | `NotConnected` | `Success` | 未创建 | [L1228](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1228) |
| **8** | **CREATE_PR_REQUEST Kafka 发送失败（内部吞异常）** | **保持 `Running`** | **保持 `Waiting`** | **`Success`（正常完成）** | **`Running`（永久挂起）** | 见 §5.3 |
| 9 | Build stale（Running 超过 5h）兜底 | `Failed` | `Failed` | 未修改 | 未修改 | [L1573](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1573) |
| 10 | calcBuildStatus() 兜底：所有 Step Success | `Completed` | `Completed` | 未直接修改 | 未直接修改 | [L1595](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1595) |
| 11 | calcBuildStatus() 兜底：任一 Step Failed | `Failed` | `Failed` | 未直接修改 | 未直接修改 | [L1603](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1603) |

> **场景 #8 注解**：CREATE_PR_REQUEST 发送失败是唯一出现"GENERATE Step 已完成但 PUSH_TO_GIT Step 仍挂起"的状态不一致场景。原因是 `saveToGitProvider()` 内部吞掉异常后正常返回，`onCodeGenerationSuccess()` 继续执行并完成 GENERATE Step，而 PUSH_TO_GIT Step 因 `leaveStepOpen=true` 且异常未向外抛出而永久停留在 Running。

### 6.3 按阶段归类的状态流转

```
Build 创建
  ├── status=Running, gitStatus=Waiting (#1)
  │   GENERATE Step=Running
  │
  ├─[有私钥插件]→ 下载插件
  │     ├── 成功 → generate()
  │     └── 失败 → status=Failed, gitStatus=Canceled (#6)
  │
  └─[无私钥插件]→ generate()
        │
        ├── DSG 执行失败 → status=Failed, gitStatus=Canceled (#2)
        │                  GENERATE Step=Failed
        │
        └── DSG 执行成功（收到 CODE_GENERATION_SUCCESS_TOPIC）
              │
              ├── build.controller.ts 按顺序执行:
              │     ① saveToGitProvider()
              │     ② onCodeGenerationSuccess()
              │
              ├─[未配置 Git]→ status=Completed, gitStatus=NotConnected (#7)
              │                PUSH_TO_GIT Step 不创建
              │                GENERATE Step=Success（由 ② 标记）
              │
              └─[已配置 Git]→ 创建 PUSH_TO_GIT Step=Running
                    │
                    ├── CREATE_PR_REQUEST Kafka 发送失败（内部吞异常）
                    │     │ saveToGitProvider() 正常返回
                    │     │ onCodeGenerationSuccess() 继续执行
                    │     ├─ GENERATE Step=Success ✅（正常完成，由 ② 标记）
                    │     ├─ PUSH_TO_GIT Step=Running ⚠️（永久挂起）
                    │     ├─ status=Running, gitStatus=Waiting (#8)
                    │     └─ 5h 后 stale 兜底: status=Failed, gitStatus=Failed (#9)
                    │           （注意：两个 Step 状态不变）
                    │
                    ├── Git 推送成功回调 (CREATE_PR_SUCCESS_TOPIC)
                    │     ├── 本地处理成功
                    │     │     GENERATE Step=Success（先由 ② 标记）
                    │     │     PUSH_TO_GIT Step=Success
                    │     │     status=Completed, gitStatus=Completed (#3)
                    │     └── 本地处理异常
                    │           GENERATE Step=Success（先由 ② 标记）
                    │           PUSH_TO_GIT Step=Failed
                    │           status=Failed, gitStatus=Failed (#4)
                    │
                    └── Git 推送失败回调 (CREATE_PR_FAILURE_TOPIC)
                          GENERATE Step=Success（先由 ② 标记）
                          PUSH_TO_GIT Step=Failed
                          status=Failed, gitStatus=Failed (#5)
```

> **关键分类**：
> - **正常完成类**：场景 #1（初始）、#2、#3、#6、#7 — 两个 Step 状态一致，Build 状态明确
> - **挂起类（状态不一致）**：场景 #8 — GENERATE Step=Success 但 PUSH_TO_GIT Step=Running，Build.status 保持 Running
> - **兜底类**：场景 #9、#10、#11 — Stale 或 calcBuildStatus 触发，修正 Build 状态但不修正 Step 状态

### 6.4 Step 状态汇总（完成 vs 挂起归类）

| Step Name | 创建时机 | 正常完成时机 | 可能状态 | 挂起风险 |
|-----------|---------|-------------|---------|---------|
| ADD_TO_QUEUE | Build.create() | 创建即完成（瞬时操作） | Success（唯一） | ❌ 无挂起风险 |
| DOWNLOAD_PRIVATE_PLUGINS | 有私钥插件时 | 插件下载成功/失败 Kafka 回调 | Success / Failed | ❌ 无挂起风险（异常会向外抛） |
| **GENERATE_APPLICATION** | generate() | CODE_GENERATION_SUCCESS / FAILURE Kafka 回调 → `onCodeGenerationSuccess()` / `onCodeGenerationFailure()` | **Success / Failed** | ✅ **不会挂起**（CREATE_PR_REQUEST 发送失败后仍会由 `build.controller.ts` 正常完成） |
| **PUSH_TO_GIT_PROVIDER** | saveToGitProvider()（有 Git 配置时） | CREATE_PR_SUCCESS / FAILURE Kafka 回调 → `onCreatePRSuccess()` / `onCreatePRFailure()` | Success / Failed / **Running** | ⚠️ **唯一会挂起的 Step**（CREATE_PR_REQUEST 发送失败时异常被内部吞，配合 `leaveStepOpen=true` 永久停留在 Running） |

> **关键区别解释**：
> - GENERATE_APPLICATION Step 的完成由 `build.controller.ts` 中 `onCodeGenerationSuccess()` 消息处理函数的第 ② 步触发，与 `saveToGitProvider()` 的内部异常无关（只要 saveToGitProvider 正常返回，第 ② 步就一定会执行）
> - PUSH_TO_GIT_PROVIDER Step 的完成完全依赖未来的 CREATE_PR_SUCCESS / FAILURE Kafka 回调。如果 CREATE_PR_REQUEST 根本没发出去（被内部吞异常），就永远不会有回调，Step 永远 Running。

---

## 七、失败去重机制深度解析

### 7.1 去重问题背景

一个 Build 可能被拆分为 2 个子 Job（Server + AdminUI），两个 Job 并发执行。如果两个 Job 都失败，会触发两次失败回调。没有去重保护则：
- `CODE_GENERATION_FAILURE_TOPIC` 被发送两次
- Server 端 `onCodeGenerationFailure()` 执行两次
- ActionLog 重复写入错误日志
- 下游监听器（通知、计费等）触发两次

### 7.2 去重核心机制：otherJobsHaveNotFailed 标志

去重逻辑在两个关键位置实现，模式一致：

#### 位置 A：DSG 执行失败回调

```typescript
// build-runner.service.ts#L257-L277
async emitCodeGenerationFailureWhenJobStatusFailed(jobBuildId: string) {
  let otherJobsHaveNotFailed = true;
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  try {
    // ① 在标记当前 Job 之前，先读 Redis 聚合状态
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // ② 再标记当前 Job = Failure
    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Failure);
  } catch (error) {
    this.logger.error(error.message, error);
  } finally {
    // ③ 只有第一个失败者才发送事件
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
    // ① 先读聚合状态
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    await this.copyFromJobToArtifact(resourceId, jobBuildId);  // 可能抛异常

    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Success);
    // ...
  } catch (error) {
    this.logger.error(error.message, error);
    if (otherJobsHaveNotFailed) {  // ② 只有第一个失败的才发事件
      await this.emitCodeGenerationFailure(buildId, error.message);
    }
  }
}
```

### 7.3 去重状态机时序

以 Server Job 先失败、AdminUI Job 后失败为例：

```
T0: Redis = { "build-123-server": "in-progress", "build-123-admin-ui": "in-progress" }
    getBuildStatus() → InProgress

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

### 7.4 去重边界情况

| 场景 | 去重效果 | 原因 |
|------|---------|------|
| Server 失败 → AdminUI 失败 | ✅ 仅 1 次事件 | Server 先标记，AdminUI 读取时已是 Failure |
| AdminUI 失败 → Server 失败 | ✅ 仅 1 次事件 | 顺序无关，谁先读谁先发 |
| 两个 Job 同时失败（极端并发） | ⚠️ 理论可能双发 | Redis get+set 非原子，两者都可能先读到 InProgress |
| runBuild() 同步异常（作业提交前） | N/A | 单线程，尚未拆分 Job |
| Package Manager 失败 | N/A | 单阶段，无需去重 |
| Git 推送失败 | N/A | 单阶段，无需去重 |
| CREATE_PR_REQUEST 发送失败 | N/A | 单阶段，且异常被吞（Step 挂起） |

> **并发风险**：Redis 的 `get` + `set` 非原子操作。极高并发下两个 Job 可能都在对方 `setJobStatus()` 前完成 `getBuildStatus()`，导致都认为自己是第一个失败者。由于 DSG 作业执行时间为分钟级，这种竞态概率极低。如需严格保证，可改用 Redis Lua 脚本或 `SETNX` 分布式锁。

### 7.5 去重保护范围汇总

```
阶段                          是否有去重保护
───────────────────────────────────────────────
runBuild() 同步异常              ❌（无需，单线程）
DSG 执行失败回调                  ✅（otherJobsHaveNotFailed）
产物复制阶段异常                  ✅（otherJobsHaveNotFailed）
Package Manager 失败              ❌（单阶段，无需）
Git 推送回调失败                  ❌（单阶段，无需）
CREATE_PR_REQUEST 发送失败        ❌（异常被吞，Step 挂起）
onCodeGenerationFailure() 服务端   ❌（依赖上游去重）
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
     │              │              │<──consume PM_CREATE_SUCCESS─────────────────│（或无包跳过）
     │              │              │ emit CODE_GENERATION_SUCCESS ─────────────>│
     │              │<─consume─────│              │             │               │
     │              │ saveToGitProvider()         │             │               │
     │              │  ├─ 无 Git → status=Completed, gitStatus=NotConnected    │
     │              │  └─ 有 Git → 创建 PUSH_TO_GIT Step=Running                │
     │              │     emit CREATE_PR_REQUEST ──────────────────────────────>│
     │              │        ├─ 发送失败（内部吞异常）→ Step 保持 Running（挂起）│
     │              │        └─ 发送成功 → 等待回调             │               │
     │              │ onCodeGenerationSuccess()   │             │               │
     │              │ Step: GENERATE_APPLICATION=Success       │               │
     │              │ emit USER_BUILD_TOPIC ────>│             │               │
     │              │              │              │             │ 创建 PR/Push   │
     │              │              │              │             │               │
     │              │<─consume CREATE_PR_SUCCESS ───────────────────────────────│
     │              │  ├─ 本地成功 → status=Completed, gitStatus=Completed      │
     │              │  └─ 本地异常 → status=Failed, gitStatus=Failed            │
     │              │ 或                          │             │               │
     │              │<─consume CREATE_PR_FAILURE ───────────────────────────────│
     │              │     → status=Failed, gitStatus=Failed                    │
     │  Notify Done │              │              │             │               │
     │<─────────────│              │              │             │               │
```

---

## 九、关键设计要点（代码核准版）

### 9.1 三层状态隔离设计
- **Layer 1 Redis**：细粒度子 Job 状态，毫秒级聚合判断，无需查库
- **Layer 2 ActionStep**：面向用户的步骤进度，每个 Step 独立生命周期和日志
- **Layer 3 Build**：双轴最终状态，对外 API 展示

### 9.2 异步 Step 的 leaveStepOpen 模式
`actionService.run()` 的 `leaveStepOpenAfterSuccessfulExecution=true` 参数使 Step 在同步回调返回后仍保持 Running，等待未来的 Kafka 事件最终完成。这是长时间异步任务的标准处理模式。

### 9.3 Step 挂起风险与状态不一致（代码实际行为）

**CREATE_PR_REQUEST 发送失败会导致「两个 Step 状态不一致」的半完成状态**，这是整个系统中唯一出现状态分裂的场景：

| 对象 | 最终状态 | 原因 |
|------|---------|------|
| `GENERATE_APPLICATION` Step | **Success（正常完成）** | `saveToGitProvider()` 正常返回后，`build.controller.ts` 继续执行 `onCodeGenerationSuccess()`，该函数标记 GENERATE Step 为 Success |
| `PUSH_TO_GIT_PROVIDER` Step | **Running（永久挂起）** | 异常被 `saveToGitProvider()` 内部 try-catch 吞噬 → stepFunction 正常返回 → `leaveStepOpen=true` 跳过 complete(Success) → 异常未向外抛跳过 complete(Failed) → Step 永久 Running |
| `Build.status` | **Running** | PUSH_TO_GIT Step 仍为 Running，Build 状态未决 |
| `Build.gitStatus` | **Waiting** | Git 请求未成功发出，仍处于等待状态 |

**关键代码链**：
1. [build.controller.ts#L87-L101](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.controller.ts#L87-L101)：顺序执行 `saveToGitProvider()` → `onCodeGenerationSuccess()`，前者不抛异常则后者必然执行
2. [build.service.ts#L474-L477](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L474-L477)：`emitMessage()` 异常被内部 catch，只打日志不向外 throw
3. [action.service.ts#L275-L295](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts#L275-L295)：`leaveStepOpen=true` + 无异常 = Step 保持 Running

**唯一补救**：5 小时后 [isBuildStale()](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L1547-L1561) 兜底，将 `Build.status` 和 `Build.gitStatus` 标记为 Failed，但**两个 ActionStep 状态不变**（GENERATE 仍为 Success，PUSH_TO_GIT 仍为 Running）。这是一个潜在的设计缺陷。

### 9.4 Build.status 与 gitStatus 的双轴设计
- `Build.status` 是**整体构建结果**，由代码生成 + Git 推送联合决定
- `Build.gitStatus` 是**Git 环节专属子状态**，与代码生成解耦
- 代码生成失败 → `gitStatus=Canceled`（Git 未执行）
- 代码生成成功但未配 Git → `gitStatus=NotConnected`（无需执行）
- Git 推送成功 → `status=Completed, gitStatus=Completed`
- Git 推送失败 → `status=Failed, gitStatus=Failed`（即使代码生成成功，整体也标记失败）

### 9.5 日志的 domain 前缀区分
Build Manager 的 `addCodeGenerationLog()` 对拆分作业日志自动添加 `[server]`/`[admin-ui]` 前缀，用户在 UI 上可清晰区分日志来源。

### 9.6 失败去重的"先读后写"模式
通过在修改状态前先读取聚合状态，用局部变量 `otherJobsHaveNotFailed` 判断是否发送事件，避免多 Job 并发失败时的重复通知。虽非严格原子（Redis get+set 非事务），但 DSG 分钟级执行时间下实际足够可靠。

### 9.7 Stale 兜底机制
`calcBuildStatus()` 提供兜底：超过 5 小时仍 Running 的 Build 会被强制标记 Failed，避免异常情况下（如 #8 挂起场景）永远处于 Running 状态。
