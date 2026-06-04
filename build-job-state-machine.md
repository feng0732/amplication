# 构建任务状态机详解

## 目录

1. [核心概念](#核心概念)
2. [状态定义](#状态定义)
3. [任务推进流程](#任务推进流程)
4. [状态切换机制](#状态切换机制)
5. [Redis 并发状态丢失分析](#redis-并发状态丢失分析)
6. [失败处理与去重机制](#失败处理与去重机制)
7. [重复事件的下游幂等边界](#重复事件的下游幂等边界)
8. [回调先后顺序分析](#回调先后顺序分析)
9. [代码引用复核](#代码引用复核)
10. [关键代码溯源](#关键代码溯源)

---

## 核心概念

### 构建任务拆分模型

Amplication 的构建系统采用**"一主多子"**的任务模型：

- **Build（主构建）**：用户触发的一次完整构建请求，具有唯一 `buildId`
- **Job（子任务）**：主构建被拆分为多个并行子任务，每个子任务有 `jobBuildId`
  - 格式：`${buildId}-${domain}`，如 `build-123-server`、`build-123-admin-ui`
  - Domain 类型：`Server`、`AdminUI`

### 状态存储设计

使用 Redis 作为状态存储：

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

定义在 `packages/amplication-build-manager/src/types.ts` L6-L10

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

`build-job-handler.service.ts` L35-L91 `splitBuildsIntoJobs()`

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

`build-job-handler.service.ts` L139-L148

```typescript
async setJobStatus(jobBuildId: string, status: EnumJobStatus): Promise<void> {
  const key = this.extractBuildId(jobBuildId);  // 提取主 buildId
  const currentVal = await this.redisService.get<RedisValue>(key);
  const newVal = {
    ...currentVal,  // 展开旧值
    [jobBuildId]: status,  // 覆盖当前任务字段
  };
  await this.redisService.set<RedisValue>(key, newVal);
}
```

**关键设计**：
- 通过 `extractBuildId()` 从 `jobBuildId` 中提取主 `buildId` 作为 Redis key
- 所有子任务的状态存储在同一个 Redis key 下
- 使用对象展开运算符 `...currentVal` 拷贝旧值到新对象

### 状态聚合：getBuildStatus()

`build-job-handler.service.ts` L100-L122

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

## Redis 并发状态丢失分析

### Redis 服务实现

`redis.service.ts` L35-L45 — 普通的 GET/SET，无原子性保证

```typescript
async get<T>(key: string): Promise<T | null> {
  const value = await this.redisClient.get(key);
  return value ? this.deserializeValue<T>(value) : null;
}

async set<T>(key: string, value: T, ttl?: number): Promise<string | null> {
  const serializedValue = this.serializeValue(value);
  return ttl
    ? this.redisClient.set(key, serializedValue, "EX", ttl)
    : this.redisClient.set(key, serializedValue);
}
```

### setJobStatus 的非原子问题

`setJobStatus()` 是 GET → 修改内存对象 → SET 三步操作，**GET 和 SET 之间没有锁**。当两个子任务的回调并发执行时，可能出现一个回调的 SET 覆盖另一个回调的 SET。

### 并发丢失的精确推演

**初始状态**：Redis 中 `{ server: in-progress, admin-ui: in-progress }`

Server 回调和 AdminUI 回调几乎同时进入 `handleDsgJobCompleted`，两者都执行到 `setJobStatus`：

```
T1: Server 回调 → setJobStatus("build-123-server", Success)
    GET key → { server: in-progress, admin-ui: in-progress }
    在内存中构造 newVal = { server: success, admin-ui: in-progress }

T2: AdminUI 回调 → setJobStatus("build-123-admin-ui", Success)
    GET key → { server: in-progress, admin-ui: in-progress }  ← 读到旧值
    在内存中构造 newVal = { server: in-progress, admin-ui: success }

T3: Server 回调 → SET key { server: success, admin-ui: in-progress }

T4: AdminUI 回调 → SET key { server: in-progress, admin-ui: success }  ← 覆盖！
```

**最终 Redis 状态**：`{ server: in-progress, admin-ui: success }`

**Server 的 Success 状态丢失了。** AdminUI 的 GET 在 T2 时刻读到的还是旧值，`...currentVal` 展开的是旧值中的 `server: in-progress`，SET 时把 Server 已经更新的 `success` 覆盖回了 `in-progress`。

### 丢失后的连锁反应

```
T5: Server 回调 → getBuildStatus() → InProgress（因为 server 字段被回滚到 in-progress）
    Server 回调 return（等待其他任务）

T6: AdminUI 回调 → getBuildStatus() → InProgress（同上）
    AdminUI 回调 return（等待其他任务）
```

**两个回调都认为还有任务在进行中，都返回等待。但没有任何后续回调会再来了** —— 两个子任务实际已经完成，不会再有 Webhook 触发状态更新。

**最终结果：构建永久挂起，不会发送成功事件，也不会发送失败事件。**

### 为什么不是"字段隔离安全"

之前有分析认为"不同任务更新不同字段，所以不会互相覆盖"。这个判断是**错误的**。

`...currentVal` 展开的是 GET 读取时那一瞬间的**完整快照**。如果 GET 读取时另一个任务的 SET 还没执行，快照里所有字段都是旧值。SET 写回时，这个旧快照会覆盖掉另一个任务已经写入的新值。

**核心问题**：`...currentVal` 不是"合并当前 Redis 最新值"，而是"合并 GET 读到的那份旧值"。

### 什么条件下不会丢失

当两个回调的 `setJobStatus` **完全串行**执行时（一个的 SET 在另一个的 GET 之前完成），不会丢失：

```
T1: Server → GET { server: in-progress, admin-ui: in-progress }
T2: Server → SET { server: success, admin-ui: in-progress }
T3: AdminUI → GET { server: success, admin-ui: in-progress }  ← 读到 Server 的更新
T4: AdminUI → SET { server: success, admin-ui: success }      ← 正确
```

关键差异：T3 的 GET 发生在 T2 的 SET 之后，读到了最新值。

### 实际发生概率

**时间窗口**：两个回调的 `setJobStatus` 中 GET 和 SET 之间的间隔（网络延迟 + JSON 序列化/反序列化），通常在亚毫秒到毫秒级别。只要两个回调的 `setJobStatus` 在这个窗口内交错执行，就会触发丢失。

**缓解因素**：
- `handleDsgJobCompleted` 在 `setJobStatus` 之前有 `copyFromJobToArtifact`（文件拷贝），这增加了两个回调之间的时间差
- 如果 Server 和 AdminUI 的 DSG Runner 执行时间差异大，回调到达时间差通常远大于 GET-SET 窗口

**但仍存在风险**：当两个 DSG 任务执行时间接近时，两个回调可能几乎同时到达，进入 `setJobStatus` 的时间窗口就可能重叠。

---

## 失败处理与去重机制

### 三层失败防护机制

#### 第一层：runBuild() 顶层异常捕获

`build-runner.service.ts` L109-L159

```typescript
try {
  // ... 任务拆分和执行逻辑
} catch (error) {
  this.logger.error(error.message, error);
  await this.emitCodeGenerationFailure(buildId, error.message);
}
```

**作用范围**：代码生成器版本获取失败、`splitBuildsIntoJobs()` 失败、`runJob()` 调用 DSG Runner 失败。

#### 第二层：handleDsgJobCompleted() 任务完成异常

`build-runner.service.ts` L208-L255

```typescript
let otherJobsHaveNotFailed = true;
try {
  const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
  otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;
  // ... 拷贝产物、更新状态 ...
} catch (error) {
  if (otherJobsHaveNotFailed) {
    await this.emitCodeGenerationFailure(buildId, error.message);
  }
}
```

**otherJobsHaveNotFailed 快照机制**：

1. **检查时机**：业务操作之前先读取当前状态
2. **快照保存**：将检查结果保存到局部变量
3. **异常时使用快照**：catch 块中使用快照决定是否发送事件

**为什么用局部变量快照而不是 catch 中再次 getBuildStatus？**
- 避免 catch 中 Redis 调用失败导致二次异常
- 防止快照检查和使用之间状态变化导致不一致
- 减少一次 Redis 调用

#### 第三层：emitCodeGenerationFailureWhenJobStatusFailed() 专门失败处理

`build-runner.service.ts` L257-L277

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

**finally 块设计**：即使 `setJobStatus()` 失败（Redis 故障），只要快照显示尚未失败，仍然发送失败事件。这是**偏向可用性**的设计：宁可多通知一次，也不漏掉。

### 失败去重的竞态窗口

`otherJobsHaveNotFailed` 基于快照，不是原子操作，存在竞态窗口：

```
T1: 任务A失败回调 → getBuildStatus → InProgress
    otherJobsHaveNotFailed = true
T2: 任务B失败回调 → getBuildStatus → InProgress
    otherJobsHaveNotFailed = true
T3: 任务A → setJobStatus(Failure) + 发送失败事件 #1
T4: 任务B → setJobStatus(Failure) + 发送失败事件 #2（otherJobsHaveNotFailed 仍为 true）
```

**结果**：同一 buildId 收到两次失败事件。

---

## 重复事件的下游幂等边界

### 事件流向总览

```
build-manager                              amplication-server
─────────────                              ──────────────────
CODE_GENERATION_SUCCESS_TOPIC ──────────▶  build.controller.ts
                                           ├─ saveToGitProvider(buildId)
                                           └─ onCodeGenerationSuccess(buildId)

CODE_GENERATION_FAILURE_TOPIC ──────────▶  build.controller.ts
                                           └─ onCodeGenerationFailure(args)
```

### 重复成功事件的下游处理

`build.controller.ts` L87-L101:

```typescript
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC)
async onCodeGenerationSuccess(@Payload() message: CodeGenerationSuccess.Value): Promise<void> {
  await this.buildService.saveToGitProvider(args.buildId);
  await this.buildService.onCodeGenerationSuccess(args.buildId);
}
```

逐项分析幂等性：

| 操作 | 代码位置 | 是否幂等 | 重复执行的后果 |
|------|----------|----------|----------------|
| `saveToGitProvider()` | `build.service.ts` L1130 | ❌ 不幂等 | 发送 `CREATE_PR_REQUEST` Kafka 事件，会创建重复 PR |
| `onCodeGenerationSuccess()` → `USER_BUILD_TOPIC` | L473-L492 | ❌ 不幂等 | 发送用户构建通知事件，可能触发重复分析/通知 |
| `onCodeGenerationSuccess()` → `actionService.complete(step, Success)` | L494 | ✅ 幂等 | Prisma update 操作，覆盖写入 status + completedAt，多次执行结果一致 |

**关键风险**：`saveToGitProvider()` 不幂等。重复调用会发送两次 `CREATE_PR_REQUEST`，导致 Git 提供商创建两个 Pull Request。

### 重复失败事件的下游处理

`build.controller.ts` L103-L115:

```typescript
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_FAILURE_TOPIC)
async onCodeGenerationFailure(@Payload() message: CodeGenerationFailure.Value): Promise<void> {
  await this.buildService.onCodeGenerationFailure(args);
}
```

`build.service.ts` L511-L534:

```typescript
public async onCodeGenerationFailure(response: CodeGenerationFailure.Value): Promise<void> {
  await this.onDsgLog({ buildId, level: "error", message: response.errorMessage || "..." });
  const step = await this.getBuildStep(buildId, GENERATE_STEP_NAME);
  await this.actionService.complete(step, EnumActionStepStatus.Failed);
  await this.updateBuildStatuses(buildId, EnumBuildStatus.Failed, EnumBuildGitStatus.Canceled);
}
```

逐项分析幂等性：

| 操作 | 代码位置 | 是否幂等 | 重复执行的后果 |
|------|----------|----------|----------------|
| `onDsgLog()` | 写入日志 | ❌ 不幂等 | 创建重复的错误日志记录 |
| `actionService.complete(step, Failed)` | Prisma update | ✅ 幂等 | 覆盖写入 status + completedAt |
| `updateBuildStatuses(Failed, Canceled)` | Prisma update | ✅ 幂等 | 覆盖写入 build 的 status + gitStatus |

**关键风险**：`onDsgLog()` 不幂等。重复调用会在 ActionStep 的日志中追加重复错误条目。

### 幂等边界总结

| 事件类型 | 幂等操作 | 非幂等操作 | 严重程度 |
|----------|----------|------------|----------|
| 成功事件重复 | `actionService.complete()` | `saveToGitProvider()` → 重复 PR | 🔴 严重 |
| 成功事件重复 | `actionService.complete()` | `USER_BUILD_TOPIC` → 重复通知 | 🟡 中等 |
| 失败事件重复 | `complete()` + `updateBuildStatuses()` | `onDsgLog()` → 重复日志 | 🟡 轻微 |

### 构建挂起的幂等问题

当 Redis 并发状态丢失导致构建挂起时（server 状态被回滚为 in-progress），**不发送任何事件**。下游 `amplication-server` 收不到成功也收不到失败事件，构建永远停留在 Running 状态。

这是比重复事件更严重的问题 —— 重复事件至少有明确的终态，挂起则是无限等待。

---

## 回调先后顺序分析

### 回调入口总览

`build-runner.controller.ts`

| 回调类型 | 触发源 | 处理函数 |
|----------|--------|----------|
| Kafka `@EventPattern` | amplication-server | `onCodeGenerationRequest()` → `runBuild()` |
| Kafka `@EventPattern` | package-manager | `onPackageManagerCreateSuccess()` |
| Kafka `@EventPattern` | package-manager | `onPackageManagerCreateFailure()` |
| HTTP `@Post` | DSG Runner | `onCodeGenerationSuccess()` |
| HTTP `@Post` | DSG Runner | `onCodeGenerationFailure()` |
| HTTP `@Post` | ? | `onNotifyPluginVersion()` |

### 回调并发处理

NestJS 默认并发处理 HTTP 请求。状态机通过 Redis 作为单一真值来源来保证正确性，但如前分析，GET-SET 非原子性在并发时可能导致状态丢失。

### 回调顺序对结果的影响

| 回调顺序 | 结果 | 是否正确 |
|----------|------|----------|
| Server 成功 → AdminUI 成功（串行） | 只发一次成功事件 | ✅ |
| AdminUI 成功 → Server 成功（串行） | 只发一次成功事件 | ✅ |
| Server 失败 → AdminUI 成功 | 只发一次失败事件 | ✅ |
| AdminUI 成功 → Server 失败 | 只发一次失败事件 | ✅ |
| 两者几乎同时成功（setJobStatus 交错） | Redis 状态丢失 → 构建挂起 | ❌ |
| 两者几乎同时失败 | 重复失败事件 + 非幂等日志写入 | ⚠️ |

---

## 代码引用复核

### 问题1：`otherJobsHaveNotFailed` 快照使用是否正确？

`build-runner.service.ts` L210

```typescript
let otherJobsHaveNotFailed = true;  // 定义在 try 外面

try {
  const currentBuildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);
  otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;
  // ... 其他操作 ...
} catch (error) {
  if (otherJobsHaveNotFailed) {  // 使用 try 块中设置的快照值
    await this.emitCodeGenerationFailure(buildId, error.message);
  }
}
```

**复核结论**：✅ 逻辑正确。快照在操作开始时获取，反映"操作开始时是否已有任务失败"。但它不是原子检查-设置，无法防止并发窗口内的重复发送。

### 问题2：状态更新顺序是否正确？

`build-runner.service.ts` L218-L224

```typescript
await this.copyFromJobToArtifact(resourceId, jobBuildId);  // 先拷贝产物
await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Success);  // 再更新状态
```

**复核结论**：✅ 逻辑正确。产物拷贝是业务成功的前提，拷贝失败不会标记 Success。但 `setJobStatus` 本身的 GET-SET 非原子性问题仍然存在。

### 问题3：状态聚合后是否需要再次检查？

`build-runner.service.ts` L227-L234

```typescript
await this.buildJobsHandlerService.setJobStatus(jobBuildId, EnumJobStatus.Success);

const buildStatus = await this.buildJobsHandlerService.getBuildStatus(buildId);

if (buildStatus === EnumJobStatus.InProgress) {
  return;
}
```

**复核结论**：✅ 逻辑正确。`setJobStatus` 可能因并发覆盖导致状态与预期不符，重新 `getBuildStatus` 能读到 Redis 的实际值。但当并发覆盖导致状态丢失时（如 server 被回滚为 in-progress），此处读到的 InProgress 会让回调错误返回。

### 问题4：`finally` 块中发送失败事件是否合理？

`build-runner.service.ts` L271-L276

```typescript
} finally {
  if (otherJobsHaveNotFailed) {
    await this.emitCodeGenerationFailure(buildId);
  }
}
```

**复核结论**：✅ 偏向可用性的合理设计。代价是可能出现"发送了失败事件但 Redis 状态没更新"的不一致。

### 问题5：测试用例是否覆盖了并发场景？

`build-runner.service.spec.ts`

- ✅ "已有任务失败时不重复发送失败事件"（L559-L583）
- ✅ "全部成功时发送成功事件"（L585-L621）
- ✅ "部分成功时不发送事件"（L623-L650）
- ✅ "有包时发送包管理事件"（L652-L705）
- ❌ **没有测试两个回调并发到达时 setJobStatus 的交错执行**
- ❌ **没有测试并发覆盖导致状态丢失后构建挂起的场景**

---

## 关键代码溯源

### 核心服务

| 文件 | 核心职责 | 关键函数 |
|------|----------|----------|
| `build-runner.service.ts` | 构建执行调度、状态机驱动 | `runBuild()`, `handleDsgJobCompleted()`, `emitCodeGenerationFailureWhenJobStatusFailed()` |
| `build-job-handler.service.ts` | 任务拆分、状态管理 | `splitBuildsIntoJobs()`, `setJobStatus()`, `getBuildStatus()` |
| `build-runner.controller.ts` | 事件监听、Webhook 接收 | `onCodeGenerationRequest()`, 各 `@EventPattern` 和 `@Post` 方法 |
| `redis.service.ts` | Redis 基础操作 | `get()`, `set()` |
| `build.controller.ts` | 下游事件消费 | `onCodeGenerationSuccess()`, `onCodeGenerationFailure()` |
| `build.service.ts` (server) | 下游业务处理 | `saveToGitProvider()`, `onCodeGenerationSuccess()`, `onCodeGenerationFailure()` |
| `build-runner.service.spec.ts` | 测试验证 | 各场景测试用例 |

### 潜在改进点

1. **Redis 原子性增强**：用 Lua 脚本替代 GET-SET，消除并发丢失窗口
   ```lua
   -- 原子更新单个字段
   local key = KEYS[1]
   local field = ARGV[1]
   local value = ARGV[2]
   redis.call('HSET', key, field, value)
   return redis.call('HGETALL', key)
   ```
   或者更简单的方案：将 `RedisValue` 从平面对象改为 Redis Hash（HSET/HGET），HSET 天然是字段级原子更新。

2. **下游幂等性增强**：
   - `saveToGitProvider()` 调用前检查 build 状态，如果已经是 Completed 则跳过
   - `onDsgLog()` 增加去重键（如 buildId + timestamp）
   - Kafka 事件中增加 `jobBuildId` 字段，下游按 buildId 去重

3. **并发测试补充**：增加两个回调并发到达的集成测试

### 常见问题排查

**Q: 为什么构建显示失败但某个子任务还是 InProgress？**
A: 状态聚合优先级决定：只要有一个任务失败，整体就是 Failure，其他任务可能还在运行。

**Q: 为什么同一个 buildId 会收到多次失败事件？**
A: `otherJobsHaveNotFailed` 是基于快照的，不是原子检查-设置。两个任务在极短时间内同时失败时，两者都在对方的 setJobStatus 执行前读到了 InProgress，都判断为"尚未失败"。

**Q: Redis GET+SET 模式会不会丢失状态更新？**
A: **会**。当两个不同子任务的回调并发执行 `setJobStatus`，且两者的 GET 在对方的 SET 之前执行时，后执行的 SET 会用旧快照覆盖前者的更新。这不是理论推演——`...currentVal` 展开的是 GET 读到的旧值，不是 Redis 当前最新值。丢失后会导致构建永久挂起。

**Q: 任务成功了但产物找不到？**
A: 检查 `copyFromJobToArtifact()` 方法，它在状态更新前执行，如果拷贝失败会进入 catch 块，不会标记 Success。

**Q: 重复成功事件会有什么后果？**
A: `saveToGitProvider()` 不幂等，会创建重复 PR。`onCodeGenerationSuccess()` 中的 `USER_BUILD_TOPIC` 事件会重复发送。只有 `actionService.complete()` 是幂等的。
