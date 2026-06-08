# Amplication Action Runner & Build Executor 调度实现深度分析

## 一、整体架构概览

Amplication 的构建调度系统采用 **分布式事件驱动架构**，核心组件分布在两个服务中：

| 服务 | 核心模块 | 主要职责 |
|------|---------|---------|
| `amplication-server` | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts) | 构建请求发起、状态持久化、Action Step 管理 |
| `amplication-server` | [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts) | Action/Step 生命周期管理、日志记录 |
| `amplication-build-manager` | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | 构建执行、作业拆分、DSG 调用 |
| `amplication-build-manager` | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | 子作业拆分、Job 状态聚合、Redis 状态管理 |
| `amplication-build-manager` | [redis.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/redis/redis.service.ts) | Job 状态缓存层 |

调度通过 **Kafka** 作为消息总线，涉及的核心 Topic 如下：

```
CODE_GENERATION_REQUEST_TOPIC        → 构建请求入队
CODE_GENERATION_SUCCESS_TOPIC        → 代码生成成功
CODE_GENERATION_FAILURE_TOPIC        → 代码生成失败
CODE_GENERATION_NOTIFY_VERSION_TOPIC → 代码生成器版本通知
DSG_LOG_TOPIC                        → 构建日志流
PACKAGE_MANAGER_CREATE_REQUEST       → 包生成请求
PACKAGE_MANAGER_CREATE_SUCCESS       → 包生成成功
PACKAGE_MANAGER_CREATE_FAILURE       → 包生成失败
```

---

## 二、排队机制（Queueing）

### 2.1 排队入口：Server 端发起构建

排队的起点在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/build.service.ts#L568-L618) 的 `generate()` 方法。

```typescript
// build.service.ts#L568-L618
private async generate(logger: ILogger, build: Build, user: User): Promise<string> {
  return this.actionService.run(
    build.actionId,
    GENERATE_STEP_NAME,      // "GENERATE_APPLICATION"
    GENERATE_STEP_MESSAGE,   // "Generating Application"
    async (step) => {
      // 1. 组装 DSGResourceData（资源描述数据）
      const dsgResourceData = await this.getDSGResourceData(resource, buildId, buildVersion, user);
      
      // 2. 将 DSGResourceData 写入共享存储（文件系统）
      await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);
      
      // 3. 发送轻量级 Kafka 消息（只传 resourceId 和 buildId）
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
    true  // leaveStepOpenAfterSuccessfulExecution = true，保持 Step 为 Running 状态
  );
}
```

**排队设计特点**：
- **消息体最小化**：Kafka 消息只传 `resourceId` 和 `buildId`，避免大消息
- **数据共享**：重量级的 `DSGResourceData` 通过共享文件系统传递
- **异步解耦**：入队后立即返回，Step 保持 `Running` 状态等待回调

### 2.2 Action Step 初始状态

`actionService.run()` 在 [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/action.service.ts#L275-L295) 中创建 Step：

```typescript
// action.service.ts#L275-L295
async run<T>(
  actionId: string,
  stepName: string,
  message: string,
  stepFunction: (step: ActionStep) => Promise<T>,
  leaveStepOpenAfterSuccessfulExecution = false
): Promise<T> {
  const step = await this.createStep(actionId, stepName, message); // status = Running
  try {
    const result = await stepFunction(step);  // 执行排队逻辑
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

Step 状态枚举定义在 [EnumActionStepStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/action/dto/EnumActionStepStatus.ts)：

```typescript
export enum EnumActionStepStatus {
  Waiting = "Waiting",
  Running = "Running",
  Failed = "Failed",
  Success = "Success",
}
```

### 2.3 Build Manager 消费端

Kafka 消费者在 [build-runner.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L68-L78)：

```typescript
// build-runner.controller.ts#L68-L78
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC)
async onCodeGenerationRequest(
  @Payload() message: CodeGenerationRequest.Value
): Promise<void> {
  this.logger.info("Code generation request received", {
    buildId: message.buildId,
    resourceId: message.resourceId,
  });
  await this.buildRunnerService.runBuild(message.resourceId, message.buildId);
}
```

---

## 三、执行调度机制（Execution Scheduling）

### 3.1 构建执行主流程

执行入口是 [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159) 的 `runBuild()` 方法：

```typescript
// build-runner.service.ts#L109-L159
async runBuild(resourceId: string, buildId: string) {
  // 1. 从共享存储读取 DSGResourceData
  const dsgResourceData = await this.readDsgResourceDataFromSharedStorage(buildId);
  
  try {
    // 2. 解析代码生成器版本
    codeGeneratorVersion = await this.codeGeneratorService.getCodeGeneratorVersion({...});
    
    // 3. 发送版本通知事件
    await this.emitCodeGenerationNotifyVersion(buildId, codeGeneratorVersion);
    
    // 4. 拆分子作业（Server / AdminUI）
    const jobs = await this.buildJobsHandlerService.splitBuildsIntoJobs(
      dsgResourceData, buildId, codeGeneratorVersion
    );
    
    // 5. 逐个执行子作业
    for (const [jobBuildId, data] of jobs) {
      await this.runJob(resourceId, jobBuildId, data, codeGeneratorVersion, codeGeneratorFullName, buildId);
    }
  } catch (error) {
    this.logger.error(error.message, error);
    await this.emitCodeGenerationFailure(buildId, error.message);
  }
}
```

### 3.2 作业拆分策略

作业拆分逻辑在 [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L35-L91) 的 `splitBuildsIntoJobs()`：

```typescript
// build-job-handler.service.ts#L35-L91
async splitBuildsIntoJobs(
  dsgResourceData: DSGResourceData,
  buildId: BuildId,
  codeGeneratorVersion: string
): Promise<ResourceTuple[]> {
  // 拆分条件：Service 类型 + DSG 版本 >= minDsgVersionToSplitBuild
  const shouldSplitBuild =
    dsgResourceData.resourceType === EnumResourceType.Service &&
    codeGeneratorVersion !== "latest-local" &&
    this.codeGeneratorService.compareVersions(
      codeGeneratorVersion, this.minDsgVersionToSplitBuild
    ) >= 0;

  const jobs: ResourceTuple[] = [];
  if (shouldSplitBuild) {
    // 拆分为 Server 作业
    if (generateServer) {
      const serverDSGResourceData = cloneDeep(dsgResourceData);
      serverDSGResourceData.resourceInfo.settings.adminUISettings.generateAdminUI = false;
      const jobBuildId = this.generateJobBuildId(buildId, EnumDomainName.Server); // buildId-server
      await this.setJobStatus(jobBuildId, EnumJobStatus.InProgress);
      jobs.push([jobBuildId, serverDSGResourceData]);
    }
    // 拆分为 AdminUI 作业
    if (generateAdminUI) {
      const adminUiDSGResourceData = cloneDeep(dsgResourceData);
      adminUiDSGResourceData.resourceInfo.settings.serverSettings.generateServer = false;
      const jobBuildId = this.generateJobBuildId(buildId, EnumDomainName.AdminUI); // buildId-admin-ui
      await this.setJobStatus(jobBuildId, EnumJobStatus.InProgress);
      jobs.push([jobBuildId, adminUiDSGResourceData]);
    }
  } else {
    // 不拆分，整个 build 作为单一作业
    await this.setJobStatus(buildId, EnumJobStatus.InProgress);
    jobs.push([buildId, dsgResourceData]);
  }
  return jobs;
}
```

Job 状态枚举在 [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/types.ts)：

```typescript
export enum EnumJobStatus {
  InProgress = "in-progress",
  Success = "success",
  Failure = "failure",
}
```

### 3.3 单作业执行：触发 DSG Runner

子作业通过 `runJob()` 触发 Argo Events/DSG Runner：

```typescript
// build-runner.service.ts#L161-L192
async runJob(
  resourceId: string,
  jobBuildId: string,
  data: DSGResourceData,
  codeGeneratorVersion: string,
  codeGeneratorFullName: string,
  plainBuildId: string
) {
  try {
    // 1. 保存作业资源数据到作业目录
    await this.saveDsgResourceData(jobBuildId, data, codeGeneratorVersion);
    // 2. 复制相关 DSG 资源
    await this.saveRelevantDsgAssets(resourceId, jobBuildId, plainBuildId);

    // 3. 调用 DSG Runner (Argo Events)
    const url = this.configService.get(Env.DSG_RUNNER_URL);
    const postBody: CodeGenerationRequest = {
      resourceId,
      buildId: jobBuildId,
      codeGeneratorVersion,
      codeGeneratorName: codeGeneratorFullName,
    };
    await axios.post(url, postBody);  // 触发容器化的代码生成器
  } catch (error) {
    throw new Error(error.message, { cause: {...} });
  }
}
```

DSG Runner 执行完成后，会通过 HTTP 回调 `POST /build-runner/code-generation-success` 或 `/code-generation-failure`。

---

## 四、状态回写机制（Status Write-back）

### 4.1 Redis 状态缓存层

子作业状态存储在 Redis 中，数据结构为：

```
Redis Key: buildId (e.g. "build-123")
Redis Value: {
  "build-123-server":    "in-progress" | "success" | "failure",
  "build-123-admin-ui":  "in-progress" | "success" | "failure"
}
```

Redis 读写封装在 [redis.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-build-manager/src/redis/redis.service.ts)。

**设置 Job 状态**：

```typescript
// build-job-handler.service.ts#L139-L148
async setJobStatus(jobBuildId: string, status: EnumJobStatus): Promise<void> {
  const key = this.extractBuildId(jobBuildId);  // 去掉 -server/-admin-ui 后缀
  const currentVal = await this.redisService.get<RedisValue>(key);
  const newVal = {
    ...currentVal,
    [jobBuildId]: status,
  };
  await this.redisService.set<RedisValue>(key, newVal);
}
```

### 4.2 Job 状态聚合

**单 Job 状态查询**：

```typescript
// build-job-handler.service.ts#L124-L129
async getJobStatus(jobBuildId: JobBuildId<BuildId>): Promise<EnumJobStatus> {
  const key = this.extractBuildId(jobBuildId);
  const value = await this.redisService.get<RedisValue>(key);
  return value[jobBuildId];
}
```

**Build 整体状态聚合**（所有子 Job 的聚合状态）：

```typescript
// build-job-handler.service.ts#L101-L122
async getBuildStatus(key: BuildId): Promise<EnumJobStatus> {
  const buildValue = await this.redisService.get<RedisValue>(key);
  const jobsStatus = Object.values(buildValue);

  // 全部成功 → Success
  const allSucceeded = jobsStatus.every(s => s === EnumJobStatus.Success);
  if (allSucceeded) return EnumJobStatus.Success;

  // 任一失败 → Failure
  const atLeaseOneFailed = jobsStatus.some(s => s === EnumJobStatus.Failure);
  if (atLeaseOneFailed) return EnumJobStatus.Failure;

  // 任一进行中 → InProgress
  const atLeastOneInProgress = jobsStatus.some(s => s === EnumJobStatus.InProgress);
  if (atLeastOneInProgress) return EnumJobStatus.InProgress;
}
```

### 4.3 成功状态回写流程

DSG Runner 成功完成后回调 → Build Manager 处理 → Server 持久化

**Step 1: DSG Runner 回调 Build Manager HTTP**

```typescript
// build-runner.controller.ts#L23-L28
@Post("code-generation-success")
async onCodeGenerationSuccess(@Payload() dto: CodeGenerationSuccessDto): Promise<void> {
  await this.buildRunnerService.onCodeGenerationSuccess(dto);
}
```

**Step 2: Build Manager 处理成功回调**

```typescript
// build-runner.service.ts#L208-L255
async handleDsgJobCompleted(resourceId: string, jobBuildId: string) {
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  let otherJobsHaveNotFailed = true;

  try {
    // 先检查当前整体状态（判断是否已有其他 Job 失败）
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // 将 Job 生成的代码从作业目录复制到构建产物目录
    await this.copyFromJobToArtifact(resourceId, jobBuildId);

    // 更新当前 Job 状态为 Success
    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Success);

    // 重新获取聚合状态
    const buildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);

    if (buildStatus === EnumJobStatus.InProgress) {
      return;  // 还有其他 Job 在跑，等待
    }

    if (buildStatus === EnumJobStatus.Success) {
      // 所有 Job 都成功了
      const dsgResourceData = await this.buildJobsHandlerService.extractDsgResourceData(jobBuildId);
      
      // 如有包需要生成，先调 Package Manager
      if (dsgResourceData.packages?.length > 0 && this.enablePackageManager) {
        await this.generatePackages(buildId, resourceId, dsgResourceData);
      } else {
        // 无包生成，直接发送 CODE_GENERATION_SUCCESS_TOPIC
        await this.codeGenerationAndPackagesCompleted(jobBuildId);
      }
    }
  } catch (error) {
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId, error.message);
    }
  }
}
```

**Step 3: Server 端消费成功事件并持久化**

```typescript
// build.controller.ts#L87-L101
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC)
async onCodeGenerationSuccess(@Payload() message: CodeGenerationSuccess.Value): Promise<void> {
  const args = plainToInstance(CodeGenerationSuccess.Value, message);
  try {
    await this.buildService.saveToGitProvider(args.buildId); // 推送到 Git
    await this.buildService.onCodeGenerationSuccess(args.buildId);
  } catch (error) {
    this.logger.error("Failed to Complete Code Generation Step ", error, {...});
  }
}
```

```typescript
// build.service.ts#L450-L495
async onCodeGenerationSuccess(buildId: string): Promise<void> {
  const step = await this.getBuildStep(buildId, GENERATE_STEP_NAME);
  
  // 发送 USER_BUILD_TOPIC 通知用户
  this.kafkaProducerService.emitMessage(KAFKA_TOPICS.USER_BUILD_TOPIC, {...});

  // 标记 GENERATE_APPLICATION Step 为 Success
  await this.actionService.complete(step, EnumActionStepStatus.Success);
}
```

### 4.4 代码生成器版本回写

```typescript
// build.controller.ts#L52-L68
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_NOTIFY_VERSION_TOPIC)
async onCodeGenerationNotifyVersion(@Payload() message: CodeGenerationNotifyVersion.Value): Promise<void> {
  const args = plainToInstance(CodeGenerationNotifyVersion.Value, message);
  await this.buildService.updateCodeGeneratorVersion(args.buildId, args.codeGeneratorVersion);
}
```

---

## 五、失败处理机制（Failure Handling）

### 5.1 失败场景分类

| 失败阶段 | 触发点 | 处理方式 |
|---------|--------|---------|
| 构建准备阶段 | `runBuild()` 异常（读取数据、版本解析） | 直接发送失败事件 |
| 作业提交阶段 | `runJob()` 异常（DSG Runner 调用失败） | 抛给上层，由 runBuild catch 后发失败事件 |
| DSG 执行阶段 | DSG Runner 回调失败 HTTP | `onCodeGenerationFailure()` 处理 |
| 产物复制阶段 | `copyFromJobToArtifact()` 异常 | catch 后发失败事件 |
| Package Manager 阶段 | 包生成失败 | `onPackageManagerCreateFailure()` 处理 |
| Git 推送阶段 | `saveToGitProvider()` 异常 | Server 端本地 catch 记录日志 |

### 5.2 构建准备/提交阶段失败

```typescript
// build-runner.service.ts#L109-L159
async runBuild(resourceId: string, buildId: string) {
  try {
    // ... 版本解析、作业拆分、作业执行
  } catch (error) {
    this.logger.error(error.message, error);
    await this.emitCodeGenerationFailure(buildId, error.message);
  }
}
```

### 5.3 DSG 执行失败回调

DSG Runner 失败后回调 `POST /build-runner/code-generation-failure`：

```typescript
// build-runner.service.ts#L194-L196
onCodeGenerationFailure(response: CodeGenerationFailureDto) {
  return this.emitCodeGenerationFailureWhenJobStatusFailed(response.buildId);
}
```

核心失败处理逻辑 `emitCodeGenerationFailureWhenJobStatusFailed()`：

```typescript
// build-runner.service.ts#L257-L277
async emitCodeGenerationFailureWhenJobStatusFailed(jobBuildId: string) {
  let otherJobsHaveNotFailed = true;
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  try {
    // 先判断是否已有其他 Job 失败（防止重复发失败事件）
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // 标记当前 Job 为 Failure
    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Failure);
  } catch (error) {
    this.logger.error(error.message, error);
  } finally {
    // 只要这是第一个失败的 Job，就发送整体失败事件
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId);
    }
  }
}
```

**关键设计：去重保护**
- 通过 `otherJobsHaveNotFailed` 标志确保只发送一次失败事件
- 即使后续还有其他 Job 失败，也不会重复触发失败通知

### 5.4 产物复制阶段失败

```typescript
// build-runner.service.ts#L248-L254
} catch (error) {
  this.logger.error(error.message, error);
  // 只有当其他 Job 还没失败时才发送失败事件（避免重复）
  if (otherJobsHaveNotFailed) {
    await this.emitCodeGenerationFailure(buildId, error.message);
  }
}
```

### 5.5 发送失败事件

```typescript
// build-runner.service.ts#L299-L311
async emitCodeGenerationFailure(buildId: string, errorMessage?: string) {
  const failureEvent: CodeGenerationFailure.KafkaEvent = {
    key: null,
    value: { buildId, errorMessage },
  };
  await this.producerService.emitMessage(
    KAFKA_TOPICS.CODE_GENERATION_FAILURE_TOPIC,
    failureEvent
  );
}
```

### 5.6 Server 端失败持久化

```typescript
// build.controller.ts#L103-L115
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_FAILURE_TOPIC)
async onCodeGenerationFailure(@Payload() message: CodeGenerationFailure.Value): Promise<void> {
  const args = plainToInstance(CodeGenerationFailure.Value, message);
  await this.buildService.onCodeGenerationFailure(args);
}
```

```typescript
// build.service.ts#L511-L534
public async onCodeGenerationFailure(response: CodeGenerationFailure.Value): Promise<void> {
  const { buildId } = response;

  // 1. 写错误日志到 ActionLog
  await this.onDsgLog({
    buildId: buildId,
    level: "error",
    message: response.errorMessage || "Code generation failed",
  });

  // 2. 查找 GENERATE_APPLICATION Step
  const step = await this.getBuildStep(buildId, GENERATE_STEP_NAME);

  // 3. 标记 Step 为 Failed
  await this.actionService.complete(step, EnumActionStepStatus.Failed);
  
  // 4. 更新 Build 状态为 Failed，Git 状态为 Canceled
  await this.updateBuildStatuses(buildId, EnumBuildStatus.Failed, EnumBuildGitStatus.Canceled);
}
```

### 5.7 Package Manager 失败处理

```typescript
// build-runner.service.ts#L60-L67
async onPackageManagerCreateFailure(response: PackageManagerCreateFailure.Value) {
  return this.emitCodeGenerationFailure(response.buildId, response.errorMessage);
}
```

### 5.8 构建状态枚举

最终 Build 实体状态在 [EnumBuildStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/103-amplication/packages/amplication-server/src/core/build/dto/EnumBuildStatus.ts)：

```typescript
export enum EnumBuildStatus {
  Running = "Running",
  Completed = "Completed",
  Failed = "Failed",
  Invalid = "Invalid",
  Unknown = "Unknown",
  Canceled = "Canceled",
}
```

---

## 六、完整调用时序图

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐     ┌───────┐     ┌──────────────┐
│     User     │     │    Server    │     │  Build Manager   │     │ Kafka │     │  DSG Runner  │
└──────┬───────┘     └──────┬───────┘     └────────┬─────────┘     └───┬───┘     └──────┬───────┘
       │ Trigger Build      │                       │                   │                │
       │───────────────────>│                       │                   │                │
       │                    │ 1. createStep(Running)│                   │                │
       │                    │ 2. save DSGResourceData to FS             │                │
       │                    │ 3. emit CODE_GENERATION_REQUEST ─────────>│                │
       │                    │                       │                   │                │
       │                    │                       │<── consume ───────│                │
       │                    │                       │ 4. read DSGResourceData from FS     │
       │                    │                       │ 5. split into Jobs (Server/AdminUI)  │
       │                    │                       │ 6. set JobStatus=InProgress (Redis)  │
       │                    │                       │ 7. POST to DSG_RUNNER_URL ───────────────────>│
       │                    │                       │                   │                │
       │                    │                       │                   │                │ 8. Generate Code
       │                    │                       │                   │                │
       │                    │                       │<──── callback success/failure ───────│
       │                    │                       │                   │                │
       │                    │                       │ 9. copy artifacts │                │
       │                    │                       │ 10. set JobStatus=Success (Redis)    │
       │                    │                       │ 11. aggregate status                │
       │                    │                       │    (check all jobs done?)            │
       │                    │                       │                   │                │
       │                    │                       │ 12a. if has packages → emit PACKAGE_MANAGER_CREATE_REQUEST
       │                    │                       │                   │                │
       │                    │                       │ 12b. else → emit CODE_GENERATION_SUCCESS ──>│
       │                    │<── consume ────────────────────────────────│                │
       │                    │ 13. saveToGitProvider()                   │                │
       │                    │ 14. complete(step, Success)               │                │
       │                    │ 15. emit USER_BUILD_TOPIC ───────────────>│                │
       │   Notify Success   │                       │                   │                │
       │<───────────────────│                       │                   │                │
```

---

## 七、关键设计要点总结

### 7.1 分布式状态管理

- **两层状态**：Redis 存子 Job 状态（快速聚合），PostgreSQL 存 Action Step 和 Build 最终状态（持久化）
- **Job ID 编码**：`{buildId}-{domain}`，domain 为 `server` 或 `admin-ui`
- **状态聚合算法**：全成功→Success，任一失败→Failure，否则→InProgress

### 7.2 失败去重机制

通过 `otherJobsHaveNotFailed` 标志和 Redis 状态预检查，确保多子作业场景下只发送一次失败事件。

### 7.3 异步编排模式

采用 **Saga 模式** 的变体：
- 每个子作业独立执行
- 通过 Kafka 事件驱动状态流转
- 无集中式协调器，各服务通过事件协作

### 7.4 消息轻量化设计

- Kafka 消息只传 ID，大对象通过共享文件系统传递
- 避免了 Kafka 大消息问题，同时提高了吞吐量

### 7.5 Step 生命周期设计

- `actionService.run()` 的 `leaveStepOpenAfterSuccessfulExecution` 参数支持长时间运行的异步 Step
- Step 不依赖同步调用结果，而是通过 Kafka 事件最终完成
