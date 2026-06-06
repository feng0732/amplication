# Background Worker Retry 语义深度解析

本文档从代码实现角度，系统梳理 Amplication 后台任务处理中**锁机制**、**重试记录**和**任务恢复**三者如何协同运转。

---

## 1. 整体架构概览

Amplication 的后台构建任务采用 **Kafka 事件驱动 + 多服务协作** 的架构：

```
amplication-server ──Kafka──▶ amplication-build-manager ──HTTP──▶ DSG Runner (Argo)
       ▲                              │
       │                              │ Kafka
       └────── CODE_GENERATION_* ◀────┘
```

核心服务模块：

| 服务 | 职责 | 关键文件 |
|---|---|---|
| `amplication-server` | 接收构建请求、持久化 Build/Action 状态、消费构建结果事件 | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.service.ts) |
| `amplication-build-manager` | 消费构建请求、拆分子任务、聚合子任务状态、调用 DSG Runner | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) |

---

## 2. 锁机制（Lock）

代码库中存在**两种独立的锁体系**，它们解决的问题域不同，互不依赖。

### 2.1 数据库级资源锁（Block/Entity 锁）

**用途**：防止多个用户同时编辑同一个 Block 或 Entity，属于**用户态协作锁**。

**实现位置**：
- [block.service.ts#L665-L752](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/block/block.service.ts#L665-L752)
- [entity.service.ts#L1472-L1537](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L1472-L1537)

**核心数据字段**（Prisma 模型层面）：
- `lockedByUserId: string | null` — 持有锁的用户 ID
- `lockedAt: Date | null` — 加锁时间戳

**获取锁 `acquireLock()` 流程**：
1. 读取目标资源当前状态
2. 若 `lockedByUserId === 当前用户 ID` → 重入成功，直接返回
3. 若 `lockedByUserId` 非空且不是当前用户 → 抛出 `AmplicationError`
4. 否则通过 Prisma `update` 原子写入 `lockedByUser.connect` + `lockedAt = new Date()`

> ⚠️ **注意**：这是 **check-then-act** 模式，并非严格的分布式锁。在高并发下存在 TOCTOU（Time-of-check to time-of-use）竞态窗口。实际中因为是用户级编辑冲突，并发概率极低。

**释放锁 `releaseLock()`**：
```typescript
prisma.block.update({
  where: { id },
  data: { lockedByUser: { disconnect: true }, lockedAt: null }
});
```

**高阶函数 `useLocking()`**：封装"获取锁 → 执行业务 → 更新锁（有变更则保持、无变更则释放）"的标准流程，使用 `try/finally` 保证异常情况下也会调用 `updateLock()`。

---

### 2.2 Redis 任务状态存储（非严格意义上的锁）

**用途**：在 build-manager 中追踪一个 Build 被拆分成的多个子 Job（Server / AdminUI）的执行状态。

**实现位置**：
- [build-job-handler.service.ts#L100-L148](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L100-L148)
- [redis.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/redis/redis.service.ts)

**Redis Key-Value 结构**：
```typescript
// Key: buildId (原始构建ID，无后缀)
// Value: { [jobBuildId]: EnumJobStatus }
type RedisValue = Record<JobBuildId<BuildId>, EnumJobStatus>;
```

示例：
```json
// Redis Key = "build-abc123"
{
  "build-abc123-server":   "in-progress",
  "build-abc123-admin-ui": "success"
}
```

**状态枚举** [types.ts#L6-L10](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/types.ts#L6-L10)：
```typescript
enum EnumJobStatus {
  InProgress = "in-progress",
  Success    = "success",
  Failure    = "failure",
}
```

**写入流程 `setJobStatus()`**：
```typescript
async setJobStatus(jobBuildId, status) {
  const key = extractBuildId(jobBuildId);        // 去掉 -server/-admin-ui 后缀
  const currentVal = await redisService.get(key); // 读取当前所有子任务状态
  const newVal = { ...currentVal, [jobBuildId]: status };
  await redisService.set(key, newVal);            // 整体写回
}
```

> ⚠️ **注意**：这也是 **read-modify-write** 模式，没有使用 Redis 的 `WATCH`/`MULTI` 或 Lua 脚本保证原子性。在两个子任务同时回调的极端情况下可能出现状态覆盖。由于每个子任务（Server/AdminUI）只更新自己对应的字段 key，且状态只从 `InProgress` 向终态单向迁移，实际出问题的概率很低。

**状态聚合 `getBuildStatus()`**：按以下优先级返回整体 Build 状态：
1. 所有子任务 Success → `Success`
2. 任一子任务 Failure → `Failure`
3. 任一子任务 InProgress → `InProgress`

---

## 3. 重试记录（Retry Records）

Amplication **没有显式的重试记录表或死信队列（DLT）**。重试语义完全依赖 **Kafka Consumer Group 机制** + 应用层的幂等性保障。

### 3.1 Kafka Consumer Group 级别重试

**配置位置**：
- [createNestjsKafkaConfig.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/util/nestjs/kafka/src/createNestjsKafkaConfig.ts)
- [kafkaEnv.ts#L54-L76](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/util/kafka/src/lib/kafkaEnv.ts#L54-L76)

**关键超时参数**（默认值）：

| 参数 | 默认值 | 含义 |
|---|---|---|
| `sessionTimeout` | 30,000 ms | Consumer 会话超时，超过该时间未发送心跳则被 Group Coordinator 认为已死亡，触发 Rebalance |
| `heartbeatInterval` | 10,000 ms | 心跳发送间隔（通常为 sessionTimeout 的 1/3） |
| `rebalanceTimeout` | 60,000 ms | Rebalance 时每个 Consumer 最长处理时间 |
| `maxBytesPerPartition` | 10,485,760 (10MB) | 每个分区每次拉取最大字节数 |

**重试发生条件**（NestJS + KafkaJS 行为）：
1. **消息处理函数抛出异常** → NestJS Kafka 适配器不会提交该消息的 offset，下次 poll 时会重新投递该消息
2. **Consumer 崩溃**（进程退出、OOM 等）→ 超过 `sessionTimeout` 后，Group Coordinator 将该 Consumer 的分区分配给其他实例，这些分区上未 commit 的消息全部重新投递
3. **处理时间超过 `rebalanceTimeout`** → 在 Rebalance 期间无法完成处理的消息会被重新分配

> 💡 **关键点**：系统采用 **At-Least-Once** 投递语义。消费端必须保证幂等。

### 3.2 Kafka Pacemaker — 长任务心跳保活

**实现位置**：[pacemaker.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/util/nestjs/kafka/src/pacemaker/pacemaker.service.ts)

某些消息处理可能耗时很长（如代码生成 > 30s），如果在处理期间不发送心跳，Consumer 会被踢出 Group，导致消息被重新投递（造成重复执行）。

`KafkaPacemaker.wrapLongRunningMethod()` 解决此问题：

```typescript
static async wrapLongRunningMethod(kafkaContext, fn, timeout = 3000) {
  const heartbeat = kafkaContext.getHeartbeat();
  let isFnDone = false;

  const fnPromise = fn(); // 启动业务函数

  while (!isFnDone) {
    // 每 timeout 毫秒抢一次：要么业务完成，要么到点发心跳
    await Promise.race([fnPromise, sleep(timeout)]);
    try {
      await heartbeat();     // 向 Broker 发送心跳，延长 session
    } catch (e) { /* swallow */ }
  }
  return await fnPromise;
}
```

**工作原理**：在业务函数执行期间，后台每 3 秒调用一次 `heartbeat()`，让 Group Coordinator 知道这个 Consumer 还活着。

### 3.3 应用层错误处理与"重试"

代码中**没有显式的重试计数或退避策略**。应用层错误处理分为两条路径：

**路径 A：build-manager 消费 `CODE_GENERATION_REQUEST_TOPIC` 出错**
- [build-runner.service.ts#L109-L158](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L158)
- `runBuild()` 中任何异常被 catch 后，调用 `emitCodeGenerationFailure()` 发送失败事件到 Kafka
- 之后该消息被视为"已处理"，offset 会被正常 commit
- **不会触发 Kafka 级别的重试**，而是直接走业务失败流程

**路径 B：build-manager 调用 DSG Runner HTTP 接口失败**
- [build-runner.service.ts#L161-L192](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L161-L192)
- `runJob()` 中 `axios.post()` 抛出异常会直接向上抛出，被外层 `runBuild()` 的 catch 捕获
- 同样发送 `CODE_GENERATION_FAILURE` 事件

**路径 C：DSG 子任务成功/失败回调**
- 成功：`handleDsgJobCompleted()` → 更新 Redis 状态 → 如果所有子任务成功则触发包管理器或发送 `CODE_GENERATION_SUCCESS`
- 失败：`emitCodeGenerationFailureWhenJobStatusFailed()` → 更新 Redis 状态为 Failure → 发送 `CODE_GENERATION_FAILURE`

失败处理的去重保护：
```typescript
async emitCodeGenerationFailureWhenJobStatusFailed(jobBuildId) {
  let otherJobsHaveNotFailed = true;
  const currentBuildStatus = await getBuildStatus(buildId);
  otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;
  // ... 更新当前 job 状态为 Failure ...
  if (otherJobsHaveNotFailed) {
    await emitCodeGenerationFailure(buildId);  // 只发送一次失败事件
  }
}
```
通过在更新状态前检查 Build 是否已处于 Failure，避免重复发送失败事件。

---

## 4. 任务恢复（Task Recovery）

### 4.1 共享文件系统作为任务持久化介质

任务的输入输出数据全部保存在共享文件系统中，这是任务可恢复的基础。

**数据目录布局**（由 Env 变量控制）：

| Env 变量 | 默认路径/含义 |
|---|---|
| `DSG_RESOURCE_DATA_BASE_FOLDER` | `/amplication-data/dsg-resource-data/{buildId}/resource-data.json` — server 写入的构建元数据 |
| `DSG_JOBS_BASE_FOLDER` | `{base}/{jobBuildId}/resource-data.json` — build-manager 为每个子任务保存的输入数据 |
| `DSG_JOBS_CODE_FOLDER` | DSG Runner 写回的代码生成结果 |
| `BUILD_ARTIFACTS_BASE_FOLDER` | `{base}/{resourceId}/{buildId}/` — 最终归档的构建产物 |

**数据流转**：
1. `amplication-server` → 写入 `DSG_RESOURCE_DATA_BASE_FOLDER/{buildId}/resource-data.json`
2. `build-manager` 读取后，按子任务拆分写入 `DSG_JOBS_BASE_FOLDER/{jobBuildId}/`
3. DSG Runner 读取对应目录的数据，生成代码写入 `DSG_JOBS_CODE_FOLDER`
4. build-manager 将成功子任务的结果 `copyFromJobToArtifact()` 复制到最终产物目录

### 4.2 构建状态机恢复（Action Step 追踪）

**实现位置**：
- [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.service.ts)
- [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/action/action.service.ts)

每个 Build 在数据库中关联一个 `Action`，Action 包含多个 `ActionStep`：

```
Build (status=Running)
  └── Action
       ├── Step: ADD_TO_QUEUE            (Success)
       ├── Step: GENERATE_APPLICATION    (Running/Failed/Success)
       ├── Step: DOWNLOAD_PRIVATE_PLUGINS (可选)
       └── Step: PUSH_TO_GIT_PROVIDER    (可选)
```

每个 Step 包含 `ActionLog` 列表，记录详细的执行日志。

**状态计算 `calcBuildStatus()`** [build.service.ts#L1563-L1618](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.service.ts#L1563-L1618)：
1. 首先检查是否为 Stale Build（Running 超过 5 小时）→ 直接标记为 Failed
2. 如果 `build.status != Unknown`，直接返回数据库中的状态
3. 否则根据所有 Step 重新计算：
   - 全部 Success → Build = Completed
   - 任一 Failed → Build = Failed
   - 其他情况（兼容性兜底）→ Build = Failed

### 4.3 Stale Build 检测（僵尸任务清理）

**定义** [build.service.ts#L177](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.service.ts#L177)：
```typescript
const STALE_BUILD_HOURS = 5;
```

判断逻辑：
```typescript
isBuildStale(build) {
  if (build.status === EnumBuildStatus.Running
      && Date.now() - build.createdAt.getTime() > 5 * 3600 * 1000) {
    return true;
  }
  return false;
}
```

当 `calcBuildStatus()` 被调用时（通常是查询 Build 详情时），会自动检测并把超时的 Running 构建标记为 Failed。

> ⚠️ **注意**：这是一个**被动检测**机制，没有后台定时任务主动扫描。只有当有人查询该 Build 时才会触发状态修正。

### 4.4 Kafka 消息重放带来的隐式恢复

由于采用 Kafka Consumer Group 的 At-Least-Once 语义，以下场景会自动"恢复"任务：

1. **build-manager 崩溃重启**：未 commit offset 的消息会被重新投递给重启后的实例或组内其他实例。由于：
   - `setJobStatus()` 是幂等的（同状态重复写入不改变结果）
   - `runBuild()` 重新读取共享目录的数据并重新执行
   - DSG Runner 被设计为可重复调用
   
   因此重放通常不会造成数据不一致。

2. **server 端崩溃重启**：未处理完成的 `CODE_GENERATION_SUCCESS` / `CODE_GENERATION_FAILURE` 事件会被重新投递。对应的处理函数 `onCodeGenerationSuccess()` / `onCodeGenerationFailure()` 内部通过数据库 Step 状态做天然幂等（重复调用 `actionService.complete()` 时，即使 Step 已是终态也不会出错，`updateBuildStatuses()` 也是幂等的 UPDATE）。

---

## 5. 端到端流程示例：一次构建的完整生命周期

```
[1] amplication-server BuildService.create()
    │  ├── Prisma: 创建 Build(status=Running) + Action + Step(ADD_TO_QUEUE)
    │  └── 写入 DSG_RESOURCE_DATA_BASE_FOLDER/{buildId}/resource-data.json
    │  └── Kafka: emit CODE_GENERATION_REQUEST_TOPIC
    ▼
[2] amplication-build-manager 消费请求
    │  BuildRunnerService.runBuild()
    │  ├── 从共享目录读取 resource-data.json
    │  ├── BuildJobsHandlerService.splitBuildsIntoJobs()
    │  │    └── Redis: set buildId → { "buildId-server": InProgress, "buildId-admin-ui": InProgress }
    │  └── 并发调用 runJob() 每个子任务
    │       └── HTTP POST DSG_RUNNER_URL (触发 Argo Workflow)
    ▼
[3] DSG Runner 执行代码生成（异步）
    │  完成后通过 HTTP 回调 build-manager 的 /code-generation-success 或 /code-generation-failure
    ▼
[4] build-runner.controller 接收回调
    │  ├── 成功: handleDsgJobCompleted()
    │  │    ├── Redis: 当前 job → Success
    │  │    ├── copyFromJobToArtifact() 复制代码到产物目录
    │  │    ├── getBuildStatus() 聚合所有子任务
    │  │    └── 全部成功?
    │  │         ├── 是: Kafka emit CODE_GENERATION_SUCCESS_TOPIC
    │  │         └── 否(还有InProgress): 等待
    │  └── 失败: emitCodeGenerationFailureWhenJobStatusFailed()
    │       ├── Redis: 当前 job → Failure
    │       └── Kafka emit CODE_GENERATION_FAILURE_TOPIC (仅首次)
    ▼
[5] amplication-server 消费结果事件
    │  onCodeGenerationSuccess():
    │    ├── saveToGitProvider() → 发出 CREATE_PR_REQUEST
    │    └── actionService.complete(step, Success)
    │  onCodeGenerationFailure():
    │    ├── 写入错误日志到 ActionLog
    │    ├── actionService.complete(step, Failed)
    │    └── updateBuildStatuses(buildId, Failed, Canceled)
    ▼
[6] git-sync-manager 消费 CREATE_PR_REQUEST
    │  PullRequestService.createPullRequest()
    │  └── GitClientService: clone → apply diff → push → create PR
    ▼
[7] amplication-server 消费 CREATE_PR_SUCCESS / CREATE_PR_FAILURE
    └── 更新 Build.gitStatus + 对应 ActionStep 状态
```

---

## 6. 设计特点与潜在风险

### ✅ 优点
1. **解耦彻底**：各服务通过 Kafka 事件通信，无直接 RPC 依赖
2. **状态分级存储**：短期运行态放 Redis、长期持久态放 Prisma/DB、大文件放共享存储
3. **天然幂等**：所有状态更新都是单向终态迁移（Running → Success/Failure），重复执行副作用可接受
4. **长任务保护**：Pacemaker 心跳机制避免 Kafka 会话超时

### ⚠️ 潜在风险点
1. **Redis 状态写入非原子**：`setJobStatus()` 的 read-modify-write 模式在极端并发下可能丢失状态更新
2. **无死信队列**：反复失败的消息会无限次重试，没有"放弃"机制
3. **无显式退避**：Kafka 级别的重试没有指数退避，瞬时故障可能引发消息风暴
4. **Stale Build 被动检测**：无人查询的僵尸任务永远停留在 Running 状态
5. **数据库锁非严格**：Block/Entity 锁的 check-then-act 模式存在 TOCTOU 竞态窗口
