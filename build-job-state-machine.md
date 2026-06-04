# 构建任务状态机详解

## 目录

1. [核心概念](#核心概念)
2. [状态定义](#状态定义)
3. [任务推进流程](#任务推进流程)
4. [状态切换机制](#状态切换机制)
5. [失败处理策略](#失败处理策略)
6. [回调协作模式](#回调协作模式)
7. [关键代码溯源](#关键代码溯源)

---

## 核心概念

### 构建任务拆分模型

Amplication 的构建系统采用**"一主多子"**的任务模型：

- **Build（主构建）**：用户触发的一次完整构建请求，具有唯一 `buildId`
- **Job（子任务）**：主构建被拆分为多个并行子任务，每个子任务有 `jobBuildId`
  - 格式：`${buildId}-${domain}`，如 `build-123-server`、`build-123-admin-ui`
  - Domain 类型：`Server`、`AdminUI`

### 状态存储设计

使用 Redis 作为状态存储，数据结构设计巧妙：

- **Key**: `buildId`（主构建ID）
- **Value**: 对象，键为 `jobBuildId`，值为任务状态

```typescript
type RedisValue = Record<JobBuildId<BuildId>, EnumJobStatus>;

// 示例
{
  "build-123-server": "in-progress",
  "build-123-admin-ui": "success"
}
```

---

## 状态定义

### EnumJobStatus 枚举

定义在 [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/types.ts#L6-L10)

| 状态 | 值 | 含义 |
|------|----|------|
| `InProgress` | `in-progress` | 任务进行中 |
| `Success` | `success` | 任务成功完成 |
| `Failure` | `failure` | 任务执行失败 |

### 状态优先级

状态聚合时遵循以下优先级：

```
Failure > InProgress > Success
```

即：**任一任务失败 → 整体失败；任一任务进行中 → 整体进行中；全部成功 → 整体成功**

---

## 任务推进流程

### 完整生命周期流程图

```
用户触发构建
    ↓
[Server] build.service.ts
    ↓ 发送 Kafka 事件
CODE_GENERATION_REQUEST_TOPIC
    ↓
[Build-Manager] build-runner.controller.ts
    ↓ @EventPattern 监听
runBuild(resourceId, buildId)
    ↓
┌─────────────────────────────────────┐
│ splitBuildsIntoJobs()               │
│ ├─ 检查是否支持拆分（版本判断）      │
│ ├─ 是：拆分为 Server + AdminUI      │
│ └─ 否：保持单一任务                 │
└─────────────────────────────────────┘
    ↓ 为每个 job 设置初始状态
setJobStatus(jobBuildId, InProgress)
    ↓
runJob() → 调用 DSG Runner (Argo)
    ↓
[异步执行 - DSG 容器]
    ↓ 完成后调用 Webhook
code-generation-success / code-generation-failure
    ↓
handleDsgJobCompleted() / emitCodeGenerationFailureWhenJobStatusFailed()
    ↓
更新状态 + 聚合判断
    ↓
全部完成？
    ├─ 是 → 触发 PackageManager（如有包）或发送成功事件
    └─ 否 → 等待其他任务
```

### 任务拆分逻辑

[splitBuildsIntoJobs()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L35-L91)

**拆分条件**（同时满足）：
1. 资源类型为 `Service`
2. 代码生成器版本不是 `latest-local`
3. 版本 >= `FEATURE_SPLIT_JOBS_MIN_DSG_VERSION`

**拆分结果**：
- 如果 `generateServer: true` → 创建 Server 子任务
- 如果 `generateAdminUI: true` → 创建 AdminUI 子任务

---

## 状态切换机制

### 状态设置：setJobStatus()

[setJobStatus()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L139-L148)

```typescript
async setJobStatus(jobBuildId: string, status: EnumJobStatus): Promise<void> {
  const key = this.extractBuildId(jobBuildId);  // 提取主 buildId
  const currentVal = await this.redisService.get<RedisValue>(key);
  const newVal = {
    ...currentVal,  // 保留其他任务状态
    [jobBuildId]: status,  // 更新当前任务状态
  };
  await this.redisService.set<RedisValue>(key, newVal);
}
```

**关键设计**：
- 通过 `extractBuildId()` 从 `jobBuildId` 中提取主 `buildId` 作为 Redis key
- 使用对象展开运算符 `...currentVal` 确保只更新当前任务，不影响其他任务
- 支持并发安全的状态更新（Redis 单线程模型保证原子性）

### 状态聚合：getBuildStatus()

[getBuildStatus()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L100-L122)

```typescript
async getBuildStatus(key: BuildId): Promise<EnumJobStatus> {
  const buildValue = await this.redisService.get<RedisValue>(key);
  const jobsStatus = Object.values(buildValue);

  // 1. 全部成功 → 成功
  if (jobsStatus.every(status => status === EnumJobStatus.Success))
    return EnumJobStatus.Success;

  // 2. 任一失败 → 失败
  if (jobsStatus.some(status => status === EnumJobStatus.Failure))
    return EnumJobStatus.Failure;

  // 3. 任一进行中 → 进行中
  if (jobsStatus.some(status => status === EnumJobStatus.InProgress))
    return EnumJobStatus.InProgress;
}
```

**状态判定顺序**：
1. **全部成功** → 整体成功
2. **任一失败** → 整体失败（快速失败）
3. **否则** → 进行中

---

## 失败处理策略

### 三层失败防护机制

#### 第一层：runBuild() 顶层异常捕获

[runBuild()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159)

```typescript
try {
  // ... 任务拆分和执行逻辑
} catch (error) {
  this.logger.error(error.message, error);
  await this.emitCodeGenerationFailure(buildId, error.message);
}
```

**作用**：捕获任务拆分和启动阶段的异常

#### 第二层：handleDsgJobCompleted() 任务完成异常

[handleDsgJobCompleted()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L208-L255)

```typescript
try {
  // 拷贝产物、更新状态、聚合判断
} catch (error) {
  this.logger.error(error.message, error);
  if (otherJobsHaveNotFailed) {
    await this.emitCodeGenerationFailure(buildId, error.message);
  }
}
```

**关键优化**：`otherJobsHaveNotFailed` 标志
- 任务开始前先检查当前状态
- 如果已有任务失败，不重复发送失败事件
- 避免失败风暴（多个任务同时失败导致多次通知）

#### 第三层：emitCodeGenerationFailureWhenJobStatusFailed() 专门失败处理

[emitCodeGenerationFailureWhenJobStatusFailed()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L257-L277)

```typescript
async emitCodeGenerationFailureWhenJobStatusFailed(jobBuildId: string) {
  let otherJobsHaveNotFailed = true;
  const buildId = this.extractBuildId(jobBuildId);

  try {
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Failure);
  } catch (error) {
    this.logger.error(error.message, error);
  } finally {
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId);
    }
  }
}
```

**设计要点**：
- `finally` 块确保无论状态更新是否成功，只要需要就发送失败通知
- 同样采用 `otherJobsHaveNotFailed` 防重复机制

### 失败事件发送：emitCodeGenerationFailure()

[emitCodeGenerationFailure()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L299-L311)

```typescript
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

---

## 回调协作模式

### 事件驱动架构概览

构建系统采用 Kafka 事件驱动，核心事件流如下：

```
┌────────────┐     ┌──────────────────┐     ┌──────────────┐
│ amplication│     │                  │     │ code-gen     │
│   server   │────▶│  Kafka Topics    │◀───▶│  runner      │
└────────────┘     │                  │     └──────────────┘
         ▲         └──────────────────┘            ▲
         │                ▲                        │
         │                │                        │
         ▼                ▼                        ▼
    ┌───────────────────────────────────────────────────┐
    │          amplication-build-manager                │
    │  (build-runner.controller + build-runner.service) │
    └───────────────────────────────────────────────────┘
```

### 核心事件监听与回调

| Kafka Topic | 监听者 | 回调函数 | 触发时机 |
|------------|--------|----------|----------|
| `CODE_GENERATION_REQUEST_TOPIC` | BuildRunnerController | `onCodeGenerationRequest()` → `runBuild()` | 用户触发构建 |
| `PACKAGE_MANAGER_CREATE_SUCCESS` | BuildRunnerController | `onPackageManagerCreateSuccess()` | 包生成成功 |
| `PACKAGE_MANAGER_CREATE_FAILURE` | BuildRunnerController | `onPackageManagerCreateFailure()` | 包生成失败 |

### Webhook 回调（HTTP POST）

DSG Runner 完成后通过 HTTP POST 回调通知：

| 端点 | 处理函数 | 说明 |
|------|----------|------|
| `/build-runner/code-generation-success` | `onCodeGenerationSuccess()` → `handleDsgJobCompleted()` | 单个子任务成功 |
| `/build-runner/code-generation-failure` | `onCodeGenerationFailure()` → `emitCodeGenerationFailureWhenJobStatusFailed()` | 单个子任务失败 |

### handleDsgJobCompleted() 核心协作逻辑

[handleDsgJobCompleted()](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L208-L255)

这是**最核心**的状态协作函数，完整流程：

```typescript
async handleDsgJobCompleted(resourceId: string, jobBuildId: string) {
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  let otherJobsHaveNotFailed = true;

  try {
    // 步骤1：检查当前状态（防重复失败）
    const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // 步骤2：拷贝产物到 artifacts 目录
    await this.copyFromJobToArtifact(resourceId, jobBuildId);

    // 步骤3：更新当前任务状态为 Success
    await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Success);

    // 步骤4：再次聚合判断整体状态
    const buildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);

    // 步骤5：根据聚合状态决定后续动作
    if (buildStatus === EnumJobStatus.InProgress) {
      return;  // 还有任务在进行中，什么都不做
    }

    if (buildStatus === EnumJobStatus.Success) {
      // 全部成功，检查是否需要生成包
      const dsgResourceData = await this.buildJobsHandlerService.extractDsgResourceData(jobBuildId);
      
      if (dsgResourceData.packages?.length > 0 && this.enablePackageManager) {
        await this.generatePackages(buildId, resourceId, dsgResourceData);
      } else {
        await this.codeGenerationAndPackagesCompleted(jobBuildId);
      }
    }
  } catch (error) {
    // 异常处理
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId, error.message);
    }
  }
}
```

**关键协作点**：

1. **"检查-设置-再检查"模式**
   - 先检查避免重复失败通知
   - 设置当前任务状态
   - 再次检查聚合状态决定下一步

2. **状态驱动的流程分支**
   - `InProgress` → 静默等待
   - `Success` → 触发后续流程（包管理或完成）
   - `Failure` → 已由其他任务处理

3. **产物拷贝在状态更新前**
   - 确保只有产物拷贝成功才标记任务成功
   - 避免"状态成功但产物丢失"的不一致

---

## 关键代码溯源

### 核心服务

| 文件 | 核心职责 | 关键函数 |
|------|----------|----------|
| [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | 构建执行调度、状态机驱动 | `runBuild()`, `handleDsgJobCompleted()`, `emitCodeGenerationFailureWhenJobStatusFailed()` |
| [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | 任务拆分、状态管理 | `splitBuildsIntoJobs()`, `setJobStatus()`, `getBuildStatus()` |
| [build-runner.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/25-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts) | 事件监听、Webhook 接收 | `onCodeGenerationRequest()`, 各 @EventPattern 和 @Post 方法 |

### 状态机设计亮点

1. **无锁设计**：利用 Redis 单线程特性避免复杂的分布式锁
2. **幂等设计**：`otherJobsHaveNotFailed` 防止重复发送失败事件
3. **最终一致**：状态更新和业务操作分离，确保最终一致性
4. **快速失败**：任一任务失败立即标记整体失败，不等待其他任务

### 常见问题排查

**Q: 为什么构建显示失败但某个子任务还是 InProgress？**
A: 状态聚合优先级决定：只要有一个任务失败，整体就是 Failure，其他任务可能还在运行。

**Q: 为什么同一个 buildId 会收到多次失败事件？**
A: 理论上不会，`otherJobsHaveNotFailed` 标志应该防止这种情况。如果出现，检查 Redis 连接或并发时序问题。

**Q: 任务成功了但产物找不到？**
A: 检查 `copyFromJobToArtifact()` 方法，它在状态更新前执行，如果拷贝失败会进入 catch 块。
