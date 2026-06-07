# Background Worker Retry 语义深度解析

本文档从代码实现角度，系统梳理 Amplication 后台任务处理中**锁机制**、**重试记录**和**任务恢复**三者如何协同运转。

---

## 1. 整体架构概览

Amplication 的后台构建任务采用 **Kafka 事件驱动 + 多服务协作** 的架构：

```
amplication-server ──Kafka──▶ amplication-build-manager ──HTTP──▶ DSG Runner (Argo)
       ▲                              │                          │
       │                              │ Kafka                    │ HTTP回调
       │                              ▼                          ▼
       │                     amplication-build-manager      (POST /code-generation-*)
       │                     接收DSG执行结果
       │
       │                       ┌──────────────────────┐
       └──── CODE_GENERATION_* │  git-sync-manager (EE) │ CREATE_PR_* ──┘
                               │  (Pacemaker心跳保活)   │
                               └──────────────────────┘
```

核心服务模块：

| 服务 | 职责 | 关键文件 |
|---|---|---|
| `amplication-server` | 接收构建请求、持久化 Build/Action 状态、消费构建结果事件 | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.service.ts) |
| `amplication-build-manager` | 消费构建请求、拆分子任务、聚合子任务状态、调用 DSG Runner、接收 DSG 回调 | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) |
| `git-sync-manager` (EE) | 消费 PR 创建请求、执行 git 操作（长任务，使用 Pacemaker） | [pull-request.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts) |

---

## 2. 锁机制（Lock）

### 2.1 两套完全独立的系统

代码库中存在**两套完全独立、无任何关联**的"锁"体系：

| 体系 | 用途 | 使用方 | 与后台任务的关系 |
|---|---|---|---|
| **用户编辑锁** (Block/Entity) | 防止多个前端用户同时编辑同一代码资源 | **仅** GraphQL resolvers（前台用户接口） | 后台任务从不获取 |
| **Redis 任务状态存储** | 追踪 Build 子任务的执行进度 | amplication-build-manager | 后台任务状态追踪（非严格意义的锁） |

> ⚠️ **重要澄清**：用户编辑锁与后台任务之间**不存在任何交互**。后台构建流程不会尝试获取 Block/Entity 锁，也不会被用户锁阻塞。

---

### 2.2 用户编辑锁（Block/Entity 锁）

**用途**：防止多个前端用户同时编辑同一个 Block 或 Entity，属于**纯用户态协作锁**。

**调用入口（仅 GraphQL 层）**：
- [entity.resolver.ts#L149](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/entity/entity.resolver.ts#L149) — `acquireLock` mutation
- [block.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/block/block.resolver.ts) — 返回查询中附带 `lockedByUser` 信息

**服务层实现**：
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

**后台构建中的 `lockedByUserId` 字段**：在 [build.service.ts#L212-L219](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.service.ts#L212-L219) 中，`lockedByUserId` 出现在 `DSG_RESOURCE_DATA_PROPERTIES_TO_REMOVE` 列表里——它只是**作为需要清理的敏感字段**，在构建数据发送给 DSG 前被剔除，不参与任何锁逻辑。

---

### 2.3 Redis 任务状态存储（非严格意义上的锁）

**用途**：在 build-manager 中追踪一个 Build 被拆分成的多个子 Job（Server / AdminUI）的执行状态。这是**状态存储**而非互斥锁——它不会阻塞任何操作，只是记录进度。

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
1. **消息处理函数抛出异常且未被捕获** → NestJS Kafka 适配器不会提交该消息的 offset，下次 poll 时会重新投递该消息
2. **Consumer 崩溃**（进程退出、OOM 等）→ 超过 `sessionTimeout` 后，Group Coordinator 将该 Consumer 的分区分配给其他实例，这些分区上未 commit 的消息全部重新投递
3. **处理时间超过 `rebalanceTimeout`** → 在 Rebalance 期间无法完成处理的消息会被重新分配

> 💡 **关键点**：系统采用 **At-Least-Once** 投递语义。消费端必须保证幂等。

---

### 3.2 Kafka 重投 vs 业务失败事件：精确边界

这是整个重试语义中最关键的分界点。每个 Kafka 消费者的异常处理方式决定了消息是被 Kafka 重投，还是被转化为业务失败事件。

**判定规则**：
- ✅ **异常被 try/catch 吞没** → handler 正常返回 → NestJS commit offset → **无 Kafka 重投**，走业务失败流程
- ❌ **异常抛出到 handler 之外** → NestJS 不 commit offset → **Kafka 重投**

下面是各消费者的实际行为：

#### 3.2.1 amplication-build-manager 消费者

**消费者 1：`CODE_GENERATION_REQUEST_TOPIC`**
[build-runner.controller.ts#L68-L78](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L68-L78)

```typescript
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC)
async onCodeGenerationRequest(@Payload() message) {
  // 控制器层无 try/catch
  await this.buildRunnerService.runBuild(...);
}
```

但 `runBuild()` 服务层内部**完全捕获**了所有异常 [build-runner.service.ts#L109-L158](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L158)：
```typescript
try {
  // ... 拆分子任务、调用 DSG Runner ...
} catch (error) {
  this.logger.error(error.message, error);
  await this.emitCodeGenerationFailure(buildId, error.message); // 发送业务失败事件
}
```

**结论**：✅ **无 Kafka 重投**，异常被服务层捕获，转化为 `CODE_GENERATION_FAILURE` Kafka 事件。

**消费者 2：`PACKAGE_MANAGER_CREATE_SUCCESS`**
[build-runner.controller.ts#L44-L54](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L44-L54)

```typescript
@EventPattern(KAFKA_TOPICS.PACKAGE_MANAGER_CREATE_SUCCESS)
async onPackageManagerCreateSuccess(@Payload() message) {
  // 无 try/catch
  await this.buildRunnerService.onPackageManagerCreateSuccess(args);
}
```

`onPackageManagerCreateSuccess()` → `codeGenerationAndPackagesCompleted()` → `producerService.emitMessage()` 均无 try/catch。

**结论**：❌ **可能触发 Kafka 重投**。若 Kafka producer 发送失败（如 broker 不可用），异常会冒泡到 handler，导致 offset 不 commit，消息被 Kafka 重投。

**消费者 3：`PACKAGE_MANAGER_CREATE_FAILURE`**
[build-runner.controller.ts#L56-L66](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L56-L66) — 同样无 try/catch。

**结论**：❌ **可能触发 Kafka 重投**。

另外值得注意：**build-manager 的所有消费者均未注入 `@Ctx() KafkaContext`**，因此它们无法使用 Pacemaker，也无法手动控制 offset 提交行为。

---

#### 3.2.2 git-sync-manager 消费者（EE，使用 Pacemaker）——逐行精确边界分析

git-sync-manager 有两个消费者：`CREATE_PR_REQUEST_TOPIC`（PR 创建）和 `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC`（私有插件下载）。它们的 try/catch 边界比表面看起来要复杂得多——**并非所有异常都能被捕获**，存在多个会触发 Kafka 重投的逃逸路径。

---

##### 3.2.2.1 `CREATE_PR_REQUEST_TOPIC` 逐行异常边界分析

代码位置：[pull-request.controller.ts#L54-L162](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts#L54-L162)

整体结构分为三段：**try 块之外（L59-L86）→ try 块之内（L87-L157）→ try 块之后（L159-L161）**

```
handler generatePullRequest()
├── [L59-L86] try 块之外 — 异常直接触发 Kafka 重投 ⚠️
│    ├── L59:  Date.now() — 安全
│    ├── L60:  plainToInstance(CreatePrRequest.Value, message) — 安全
│    ├── L61:  validateOrReject(validArgs) — ❌ 校验失败直接抛
│    ├── L63-65: context.getMessage() offset/topic/partition — 安全
│    ├── L66-69: plainToInstance(CreatePrRequest.Key, key.toString())
│    │              — ❌ key 为 null/undefined 时 toString() 抛
│    ├── L70-73: logger.child() — 安全
│    ├── L75-79: this.log() → emitMessage(CREATE_PR_LOG_TOPIC)
│    │              — ❌ Kafka broker 不可用时抛
│    └── L80-85: logger.info() — 安全（本地日志）
│
├── [L87-L157] try 块之内
│    ├── [L88-L92] KafkaPacemaker.wrapLongRunningMethod(
│    │                context, () => createPullRequest(validArgs)
│    │              )
│    │              — ✅ Pacemaker 覆盖（仅此处），异常进入 catch
│    ├── [L94-L99] logger.info('Finish process, committing') — 安全
│    ├── [L101-L115] emitMessage(CREATE_PR_SUCCESS_TOPIC)
│    │                  — ❌ 抛异常进入 catch 块
│    │
│    └── [L116-L156] catch (error)
│         ├── L117:  instanceof NoChangesOnPullRequest?
│         ├──   YES:
│         │    ├── L118-L122: this.log() → emitMessage(CREATE_PR_LOG_TOPIC)
│         │    │                  — ❌ 抛异常逃离 catch → Kafka 重投
│         │    └── L123-L133: emitMessage(CREATE_PR_SUCCESS_TOPIC)
│         │                       — ❌ 抛异常逃离 catch → Kafka 重投
│         ├──   NO:
│         │    ├── L137-L140: logger.error() — 安全
│         │    └── L142-L156: emitMessage(CREATE_PR_FAILURE_TOPIC)
│         │                       — ❌ 抛异常逃离 catch → Kafka 重投
│         └──   两个分支中任何 emit 失败，异常都会逃离 catch 块
│
└── [L159-L161] try 块之后
     └── logger.info('Pull request item processed') — 安全
```

**逃逸路径汇总（触发 Kafka 重投）**：

| 路径 | 位置 | 触发条件 | 异常类型 |
|---|---|---|---|
| ① | L61 `validateOrReject` | 消息格式不符合 class-validator 规则 | `ValidationError` 数组 |
| ② | L68 `key.toString()` | Kafka 消息 key 为 null/undefined | `TypeError: Cannot read properties of null` |
| ③ | L75 `this.log()` → `emitMessage(CREATE_PR_LOG_TOPIC)` | Kafka broker 不可用、序列化失败 | KafkaJS producer error / serializer error |
| ④ | L118 catch 内 `this.log()` (NoChanges 分支) | Kafka broker 不可用 | 同上 |
| ⑤ | L123 catch 内 `emitMessage(CREATE_PR_SUCCESS)` (NoChanges 分支) | Kafka broker 不可用 | 同上 |
| ⑥ | L153 catch 内 `emitMessage(CREATE_PR_FAILURE)` | Kafka broker 不可用 | 同上 |

> 💡 **关键洞察**：路径 ① 和 ② 是**永久性错误**（坏消息永远无法通过校验），如果没有死信队列，会导致 Kafka **无限重投同一条坏消息**。路径 ③~⑥ 是**瞬时性错误**（Kafka broker 恢复后可自愈）。

---

##### 3.2.2.2 `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` 逐行异常边界分析

代码位置：[private-plugin.controller.ts#L32-L84](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts#L32-L84)

```
handler downloadPrivatePlugins()
├── [L37-L46] try 块之外 — 异常直接触发 Kafka 重投 ⚠️
│    ├── L37-40: plainToInstance(Key, key.toString())
│    │              — ❌ key 为 null/undefined 时 toString() 抛
│    ├── L42-45: plainToInstance(Value, message) — 安全
│    └── L46:    validateOrReject(validArgs) — ❌ 校验失败直接抛
│
├── [L48-L83] try 块之内
│    ├── [L49-L53] KafkaPacemaker.wrapLongRunningMethod(
│    │                context, () => downloadPrivatePlugins(validArgs)
│    │              )
│    │              — ✅ Pacemaker 覆盖（仅此处），异常进入 catch
│    ├── [L55-L67] emitMessage(DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC)
│    │              — ❌ 抛异常进入 catch 块
│    │
│    └── [L68-L83] catch (error)
│         ├── L69:    logger.error() — 安全
│         └── L70-L82: emitMessage(DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC)
│                        — ❌ 抛异常逃离 catch → Kafka 重投
│
└── (无 try 后代码)
```

**逃逸路径汇总（触发 Kafka 重投）**：

| 路径 | 位置 | 触发条件 | 异常类型 |
|---|---|---|---|
| ① | L39 `key.toString()` | Kafka 消息 key 为 null/undefined | `TypeError: Cannot read properties of null` |
| ② | L46 `validateOrReject` | 消息格式不符合 class-validator 规则 | `ValidationError` 数组 |
| ③ | L79 catch 内 `emitMessage(DOWNLOAD_PRIVATE_PLUGINS_FAILURE)` | Kafka broker 不可用 | KafkaJS producer error |

---

##### 3.2.2.3 `emitMessage` 的异常来源

所有逃逸路径中的 `emitMessage` 都经过 [KafkaProducer.service.ts#L22-L38](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts#L22-L38)，其中有两个抛异常点：

```typescript
async emitMessage(topic, message, schemaIds?) {
  const kafkaMessage = await this.serializer.serialize(message, schemaIds);
  //                                                  ^^^^ 抛点 1: 序列化失败
  return await new Promise((resolve, reject) => {
    this.kafkaClient.emit(topic, kafkaMessage).subscribe({
      error: (err) => reject(err),  // 抛点 2: Kafka produce 失败
      next: () => resolve(),
    });
  });
}
```

序列化器实现 [KafkaMessageJsonSerializer.ts#L58-L80](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/util/kafka/src/lib/serializer/json/KafkaMessageJsonSerializer.ts#L58-L80) 中，`serialize()` 使用 `JSON.stringify()` + `Buffer.from()`，对于正常对象不会抛异常（反序列化才会对前导零字节做二进制检测并抛异常）。因此实际生产中，`emitMessage` 抛出异常的主要来源是 **Kafka broker 不可用**。

---

##### 3.2.2.4 结论

git-sync-manager 的两个消费者 **并非完全无 Kafka 重投风险**。精确结论：

- ✅ **核心业务异常（git 操作失败、插件下载失败）**：被 try/catch 捕获，转化为业务失败事件，**无 Kafka 重投**
- ❌ **参数校验失败、消息 key 为空**：在 try 块之外，**直接触发 Kafka 重投**（永久性错误，无限循环）
- ❌ **catch 块内发送失败/日志事件失败**：异常逃离 catch，**触发 Kafka 重投**（瞬时错误，Kafka 恢复后自愈）

---

##### 3.2.2.5 事件 key 解析路径深度分析

git-sync-manager 中 Kafka 消息 key 的解析存在**两层问题**：key 不经过 `validateOrReject` 校验、且 `plainToInstance` 接收到的是未 JSON 解析的原始字符串。

**完整链路（以 CREATE_PR_REQUEST 为例）**：

```
Producer (amplication-server build.service.ts#L1318-L1331)
  │ 构造: { key: { resourceRepositoryId: kafkaEventKey, resourceId: branchPer ? res.id : null }, value: {...} }
  │        ↑ kafkaEventKey 始终为非空 string (demoRepoName 或 resourceRepository.id)
  │        ↑ resourceId 可能为 null (非 branchPerResource 模式下显式设为 null)
  ▼
KafkaProducerService.emitMessage()
  │ serialiseField(keyObj): typeof keyObj === 'object' → JSON.stringify(keyObj) → Buffer
  │ 例: {"resourceRepositoryId":"repo-123","resourceId":null} → Buffer
  ▼
Kafka Broker (消息持久化)
  ▼
Consumer (git-sync-manager pull-request.controller.ts#L66-L69)
  │ context.getMessage().key  ← 从 KafkaJS 获取的原始 Buffer
  │ .toString()                ← Buffer → JSON 字符串: '{"resourceRepositoryId":"repo-123","resourceId":null}'
  │ plainToInstance(CreatePrRequest.Key, jsonString)  ← ⚠️ 传入的是 STRING，不是 OBJECT
  ▼
结果: eventKey.resourceRepositoryId = undefined, eventKey.resourceId = undefined
```

---

**问题 1：key 是否经过 `validateOrReject` 校验？—— 否**

两个消费者中，**只有 value 被校验，key 从不校验**：

| 消费者 | value 校验? | key 校验? |
|---|---|---|
| CREATE_PR_REQUEST | ✅ L61: `await validateOrReject(validArgs)` | ❌ 无 |
| DOWNLOAD_PRIVATE_PLUGINS_REQUEST | ✅ L46: `await validateOrReject(validArgs)` | ❌ 无 |

Key 的 schema 类虽然定义了 `@IsString()` 装饰器：
- [create-pr-request/key.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/schema-registry/src/lib/create-pr-request/key.ts): `resourceRepositoryId!: string`, `resourceId!: string | null`
- [download-private-plugins-request/key.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/schema-registry/src/lib/download-private-plugins-request/key.ts): `resourceId!: string | null`

但由于 key 从未被 `validateOrReject()` 调用，这些装饰器完全不生效——**即使 key 内容完全不符合 schema（如字段缺失、类型错误），也不会触发任何异常**。

---

**问题 2：`plainToInstance` 对非空但不符合 schema 的 key 会如何处理？—— 返回全 undefined 的实例**

`class-transformer` 的 `plainToInstance(cls, plain)` 行为：
- 若 `plain` 是**普通对象** `{foo: "bar"}` → 创建 `cls` 实例并拷贝属性
- 若 `plain` 是**字符串**（如 JSON 字符串 `'{"foo":"bar"}'`）→ 创建 `cls` 空实例，**不做任何属性拷贝**
- 若 `plain` 是**畸形对象**（如 key 字段拼写错误 `{resourceRepoId: "x"}`）→ 创建实例，存在的字段拷贝，不存在的字段为 undefined

在当前代码中，`context.getMessage().key.toString()` 返回的是 **JSON 字符串**，不是解析后的对象。因此 **无论 key 内容是否符合 schema，`eventKey` 的所有字段都是 `undefined`**。

所有 key 内容异常场景的结果一致：

| key Buffer 内容 | `.toString()` 结果 | `plainToInstance` 结果 | 是否抛异常 |
|---|---|---|---|
| 合法 JSON 对象 `{"resourceRepositoryId":"repo-1","resourceId":null}` | JSON string | `{resourceRepositoryId: undefined, resourceId: undefined}` | ❌ 不抛 |
| 非法 JSON（如 `"not-json"` 或部分损坏） | 原始字符串 | 全字段 undefined | ❌ 不抛 |
| key 为对象但字段缺失 `{resourceRepositoryId:"repo-1"}` | JSON string | 全字段 undefined | ❌ 不抛 |
| key 为 `null` | `TypeError: Cannot read properties of null` | — | ✅ 抛（try 外 → Kafka 重投）|

---

**问题 3：缺失的 `resourceRepositoryId` / `resourceId` 对后续事件的影响**

解析出来的 `eventKey`（字段全为 undefined）仅用于**构造下游 Kafka 事件的 key**，不参与任何业务逻辑判断：

**CREATE_PR_REQUEST 后续 key 使用**：

| 位置 | 代码 | 实际效果 |
|---|---|---|
| [L102-L104](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts#L102-L104) 成功事件 | `key: { resourceRepositoryId: eventKey.resourceRepositoryId }` | `{ resourceRepositoryId: undefined }` |
| [L126](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts#L126) NoChanges 分支 | `key: eventKey` | `{ resourceRepositoryId: undefined, resourceId: undefined }` |
| [L142-L145](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts#L142-L145) 失败事件 | `key: { resourceRepositoryId: eventKey.resourceRepositoryId }` | `{ resourceRepositoryId: undefined }` |

这些 key 经 `JSON.stringify` 序列化后，`undefined` 字段会被省略，最终所有下游事件的 key 都变成 `{}`。

**DOWNLOAD_PRIVATE_PLUGINS_REQUEST 后续 key 使用**：

| 位置 | 代码 | 实际效果 |
|---|---|---|
| [L56-L58](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts#L56-L58) 成功事件 | `key: { resourceId: eventKey.resourceId }` | `{ resourceId: undefined }` → `{}` |
| [L71-L73](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts#L71-L73) 失败事件 | `key: { resourceId: eventKey.resourceId }` | `{ resourceId: undefined }` → `{}` |

---

**影响评估**：

1. **Kafka 分区丢失**：key 全部变成 `{}`，CREATE_PR_SUCCESS/FAILURE、DOWNLOAD_PRIVATE_PLUGINS_SUCCESS/FAILURE 事件失去按 `resourceRepositoryId` / `resourceId` 分区的能力，所有消息 hash 到同一分区。

2. **业务逻辑不受影响**：下游消费者（amplication-server）**只读取 `@Payload()` value，从不读取 key**（见 [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.controller.ts) 所有 handler 均无 `@Ctx()` 或 `context.getMessage()` 调用）。

3. **`resourceId: null` 的语义变化**：Producer 端对于非 branch-per-resource 构建显式将 `resourceId` 设为 `null`（表示"整个项目级别，避免并行 PR 冲突"，见 [build.service.ts#L1321-L1328](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.service.ts#L1321-L1328) 的注释）。由于 key 解析 bug，这个 `null` 变成了 `undefined`，最终在序列化时被省略为 `{}`——语义上等价（都不包含 resourceId），但丢失了显式 null 的意图。

---

##### 3.2.2.6 业务偏移 vs Kafka 重投：本质区别

git-sync-manager 中存在两种完全不同的"失败处理"模式，二者在 offset 管理、重试机制、控制方上有本质区别：

| 维度 | Kafka 重投（Kafka-level retry） | 业务偏移（Business offset） |
|---|---|---|
| **触发条件** | 异常**逃离 handler 函数**（try 外抛异常、catch 内二次逃逸） | handler **正常返回**（offset 已 commit），但业务操作本身失败 |
| **offset 行为** | ❌ **不 commit** → 消息滞留原 offset | ✅ **commit** → Kafka 认为消息已处理 |
| **消息是否重投** | ✅ Kafka 自动重新投递同一消息（可能无限循环） | ❌ Kafka 不再关心此消息 |
| **失败通知方式** | 无（消息重复消费） | 业务事件：`CREATE_PR_FAILURE`、`DOWNLOAD_PRIVATE_PLUGINS_FAILURE` 等 Kafka 事件 |
| **控制方** | Kafka Consumer Group 协议（sessionTimeout、rebalance、poll 循环） | 应用代码（try/catch、事件发送） |
| **重试语义** | 完全相同的消息从头执行 | 无自动重试，需应用层设计补偿逻辑 |
| **幂等性要求** | 高（重复执行 git clone/push 可能创建重复 PR） | 低（仅发送 Kafka 事件） |

**代码中的具体实例对照**：

**Kafka 重投示例（offset 不 commit）**：
```typescript
// pull-request.controller.ts L61 - try 块外
await validateOrReject(validArgs);  // ← 抛 ValidationError → 逃离 handler
// → NestJS 不 commit offset → Kafka 重投同一条消息
```

**业务偏移示例（offset commit，走业务失败）**：
```typescript
// pull-request.controller.ts L87-L156 - try/catch 内
try {
  await KafkaPacemaker.wrapLongRunningMethod(
    context, () => this.pullRequestService.createPullRequest(validArgs)
    // ← git 操作失败抛异常
  );
} catch (error) {
  // ← 异常被捕获
  await this.producerService.emitMessage(CREATE_PR_FAILURE_TOPIC, failureEvent);
  // ← 发送业务失败事件（假设这次 emit 成功）
}
// → handler 正常返回 → NestJS commit offset → Kafka 不再重投
// → 下游 amplication-server 消费 CREATE_PR_FAILURE 事件处理业务失败
```

> 💡 **关键洞察**：业务偏移是一种"承认失败、记录失败、继续前进"的策略——offset 向前推进，失败状态通过事件系统传播给下游；Kafka 重投则是"卡在原地、反复尝试"——offset 不动，希望下一次处理能成功。前者适合**确定性失败**（git 仓库不存在、权限不足等），后者适合**瞬时性失败**（网络闪断、Kafka broker 临时不可用等）。当前代码中瞬时失败也会在 catch 内 `emitMessage` 失败时意外升级为 Kafka 重投。

---

#### 3.2.3 amplication-server 消费者

[build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-server/src/core/build/build.controller.ts) 中所有 `@EventPattern` 消费者：

| 消费者 | 有 try/catch? | Kafka 重投? |
|---|---|---|
| `CODE_GENERATION_NOTIFY_VERSION_TOPIC` | ✅ | 无 |
| `BUILD_PLUGIN_NOTIFY_VERSION_TOPIC` | ✅ | 无 |
| `CODE_GENERATION_SUCCESS_TOPIC` | ✅ | 无 |
| `CODE_GENERATION_FAILURE_TOPIC` | ✅ | 无 |
| `CREATE_PR_SUCCESS_TOPIC` | ✅ | 无 |
| `CREATE_PR_FAILURE_TOPIC` | ✅ | 无 |
| `DSG_LOG_TOPIC` | ❌ | 可能 |
| `CREATE_PR_LOG_TOPIC` | ✅ | 无 |
| `DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC` | ✅ | 无 |
| `DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC` | ✅ | 无 |
| `DOWNLOAD_PRIVATE_PLUGINS_LOG_TOPIC` | ✅ | 无 |

**结论**：除 `DSG_LOG_TOPIC` 外，其余均有 try/catch → **无 Kafka 重投**。

---

### 3.3 Kafka Pacemaker — 长任务心跳保活（仅 git-sync-manager 使用）

**实现位置**：[pacemaker.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/libs/util/nestjs/kafka/src/pacemaker/pacemaker.service.ts)

**使用范围**：仅 `ee/packages/git-sync-manager` 的两个消费者使用。`amplication-build-manager` 和 `amplication-server` 的所有 Kafka 消费者**均未使用** Pacemaker（甚至没有注入 `@Ctx() KafkaContext`）。

---

#### 3.3.1 Pacemaker 精确覆盖范围

Pacemaker 仅覆盖两个消费者 handler 中**一行代码**——核心业务服务方法调用：

| 消费者 | Pacemaker 覆盖的代码 | 覆盖行数 |
|---|---|---|
| `CREATE_PR_REQUEST_TOPIC` | `() => this.pullRequestService.createPullRequest(validArgs)` | [pull-request.controller.ts#L89-L92](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/pull-request/pull-request.controller.ts#L89-L92) |
| `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` | `() => this.privatePluginService.downloadPrivatePlugins(validArgs)` | [private-plugin.controller.ts#L49-L53](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts#L49-L53) |

**未被 Pacemaker 覆盖的代码**（在 handler 中但不在 `wrapLongRunningMethod` 内）：

| 代码段 | 所在位置 | 耗时风险 |
|---|---|---|
| 参数校验 `validateOrReject` | try 外 L61 / L46 | 毫秒级，可忽略 |
| key 解析 `plainToInstance(...Key, key.toString())` | try 外 L66-L69 / L37-L40 | 毫秒级，可忽略 |
| 前置日志 `this.log()` → `emitMessage(CREATE_PR_LOG_TOPIC)` | try 外 L75-L79（仅 PR 创建） | 毫秒级（Kafka 同步发送），broker 故障时可能阻塞 |
| 成功事件发送 `emitMessage(CREATE_PR_SUCCESS)` / `emitMessage(DOWNLOAD_PRIVATE_PLUGINS_SUCCESS)` | try 内 L101-L115 / L55-L67 | 毫秒级（Kafka 同步发送） |
| 失败事件发送 `emitMessage(CREATE_PR_FAILURE)` 等 | catch 内 L123-L156 / L70-L82 | 毫秒级（Kafka 同步发送） |
| 本地日志 `logger.info()` / `logger.error()` | 多处 | 微秒级 |

> 💡 **设计合理性分析**：虽然严格来说前置日志 `this.log()` 也可能因 Kafka 故障阻塞超过 30s，但 `this.log()` 在 try 块之外——如果它抛异常会直接触发 Kafka 重投，Pacemaker 也没必要保护它。真正耗时的是 git clone、diff、push 等 I/O 密集操作，这些全部在 `createPullRequest()` 和 `downloadPrivatePlugins()` 内部，Pacemaker 的覆盖范围是**正确且足够的**。

---

#### 3.3.2 Pacemaker 内部实现与异常语义

```typescript
static async wrapLongRunningMethod<T>(
  kafkaContext: KafkaContext,
  fn: () => Promise<T>,
  timeout = 3000
) {
  const heartbeat = kafkaContext.getHeartbeat();
  const sleep = promisify(setTimeout);
  let isFnDone = false;

  const wrappedFn = async () => {
    const result = await fn();    // ① 启动业务函数
    isFnDone = true;
    return result;
  };

  const fnPromise = wrappedFn();

  while (!isFnDone) {
    await Promise.race([fnPromise, sleep(timeout)]);  // ② 每 3s 抢一次
    try {
      await heartbeat();       // ③ 发送心跳
    } catch (error) {
      // swallow — 心跳失败不影响业务函数
    }
  }

  return await fnPromise;  // ④ 业务函数的异常直接由此 re-throw
}
```

**关键行为**：

1. **心跳失败被吞没**（③处 try/catch）：如果心跳发送失败（例如短暂网络闪断），Pacemaker 不会中断业务函数，继续循环。这是正确的——偶尔的心跳失败不足以让 Consumer 被踢出 Group（需要连续超过 `sessionTimeout`=30s 未收到心跳）。

2. **业务异常原样传递**（④处 `return await fnPromise`）：如果 `fn()` 抛异常，Pacemaker 会原封不动地 re-throw。异常不会在 Pacemaker 内部被吞掉，会正常进入外层 try/catch 的 catch 分支。

3. **与外层 try/catch 的配合**：Pacemaker 位于 try 块内部，所以：
   - 业务成功 → 正常返回 → 继续执行成功事件发送
   - 业务失败 → 异常 re-throw → 被外层 catch 捕获 → 发送业务失败事件

---

#### 3.3.3 使用场景

git 操作（clone、diff、push、create PR）和私有插件下载通常耗时数分钟，远超默认的 `sessionTimeout`（30s）。如果在处理期间不发送心跳，Consumer 会被踢出 Group，导致消息被重新投递（造成重复执行）。

Pacemaker 的工作原理是：在业务函数执行期间，后台每 3 秒调用一次 `heartbeat()`，让 Group Coordinator 知道这个 Consumer 还活着。

---

### 3.4 应用层错误处理与业务失败事件

代码中**没有显式的重试计数或退避策略**。应用层通过 Kafka 业务事件表达成功/失败：

**DSG 子任务成功/失败回调（HTTP，非 Kafka）**：
build-manager 通过 HTTP POST 接口接收 DSG Runner 的回调 [build-runner.controller.ts#L23-L42](file:///d:/fz/0601/solo-dogfeeding/code/55-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L23-L42)：
- 成功：`handleDsgJobCompleted()` → 更新 Redis 状态 → 如果所有子任务成功则触发包管理器或发送 `CODE_GENERATION_SUCCESS` Kafka 事件
- 失败：`emitCodeGenerationFailureWhenJobStatusFailed()` → 更新 Redis 状态为 Failure → 发送 `CODE_GENERATION_FAILURE` Kafka 事件

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

---

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

---

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

---

### 4.4 Kafka 消息重放带来的隐式恢复

由于采用 Kafka Consumer Group 的 At-Least-Once 语义，以下场景会自动"恢复"任务：

1. **build-manager 崩溃重启**：
   - `onCodeGenerationRequest`：虽然 `runBuild()` 内部 catch 异常不会触发重投，但如果进程在 handler 执行过程中崩溃，offset 尚未 commit，消息会被 Kafka 重投。
   - `onPackageManagerCreateSuccess/Failure`：这两个 handler 无 try/catch，任何异常都会触发重投。
   - 幂等保障：`setJobStatus()` 是幂等的（同状态重复写入不改变结果）；`runBuild()` 重新读取共享目录的数据并重新执行；DSG Runner 被设计为可重复调用。

2. **git-sync-manager 崩溃或异常逃逸**：
   - **进程崩溃**：如果在 `createPullRequest()` / `downloadPrivatePlugins()` 执行过程中崩溃（Pacemaker 心跳也随之停止），超过 `sessionTimeout` 后 Group Coordinator 会触发 Rebalance，消息被重新分配给其他实例。
   - **try 块外异常逃逸**：`validateOrReject` 失败（消息格式错误）或 `key.toString()` 失败（消息 key 为 null）直接触发 Kafka 重投。这是**永久性错误**，会无限循环。
   - **catch 块内异常二次逃逸**：catch 中发送 `CREATE_PR_FAILURE` / `CREATE_PR_SUCCESS` 等 Kafka 事件时，如果 broker 不可用，异常逃离 catch 触发 Kafka 重投。Kafka 恢复后消息被重新消费，git 操作重新执行——可能创建重复 PR。
   - 由于 git 操作（创建 PR）本身不是幂等的，上述场景中重复执行可能导致创建重复 PR。Pacemaker 仅保证**进程存活且执行核心业务函数期间**不会因会话超时而重投。

3. **amplication-server 崩溃重启**：
   - 大多数消费者有 try/catch（不会触发重投），但如果进程在 handler 执行过程中崩溃，offset 未 commit，消息会被 Kafka 重投。
   - 幂等保障：处理函数 `onCodeGenerationSuccess()` / `onCodeGenerationFailure()` 内部通过数据库 Step 状态做天然幂等（重复调用 `actionService.complete()` 时，即使 Step 已是终态也不会出错，`updateBuildStatuses()` 也是幂等的 UPDATE）。

---

## 5. 端到端流程示例：一次构建的完整生命周期

```
[1] amplication-server BuildService.create()
    │  ├── Prisma: 创建 Build(status=Running) + Action + Step(ADD_TO_QUEUE)
    │  └── 写入 DSG_RESOURCE_DATA_BASE_FOLDER/{buildId}/resource-data.json
    │       (lockedByUserId 等敏感字段在此前被剔除)
    │  └── Kafka: emit CODE_GENERATION_REQUEST_TOPIC
    ▼
[2] amplication-build-manager 消费请求 (onCodeGenerationRequest)
    │  BuildRunnerService.runBuild()
    │  ├── 从共享目录读取 resource-data.json
    │  ├── BuildJobsHandlerService.splitBuildsIntoJobs()
    │  │    └── Redis: set buildId → { "buildId-server": InProgress, "buildId-admin-ui": InProgress }
    │  └── 并发调用 runJob() 每个子任务
    │       └── HTTP POST DSG_RUNNER_URL (触发 Argo Workflow)
    │  注意：此过程中所有异常都被 runBuild() catch，转为 CODE_GENERATION_FAILURE 事件
    │  注意：此处未使用 Pacemaker（未注入 @Ctx() KafkaContext）
    ▼
[3] DSG Runner 执行代码生成（异步）
    │  完成后通过 HTTP 回调 build-manager 的 POST /code-generation-success 或 /code-generation-failure
    ▼
[4] build-runner.controller 接收 HTTP 回调
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
[5] amplication-server 消费结果事件 (所有消费者均有 try/catch，无 Kafka 重投)
    │  onCodeGenerationSuccess():
    │    ├── saveToGitProvider() → Kafka emit CREATE_PR_REQUEST
    │    └── actionService.complete(step, Success)
    │  onCodeGenerationFailure():
    │    ├── 写入错误日志到 ActionLog
    │    ├── actionService.complete(step, Failed)
    │    └── updateBuildStatuses(buildId, Failed, Canceled)
    ▼
[6] git-sync-manager 消费 CREATE_PR_REQUEST (Pacemaker 仅覆盖 createPullRequest)
    │  PullRequestController.generatePullRequest()
    │  ├── [try外] 参数校验、key 解析、前置日志 — 异常直接 Kafka 重投
    │  ├── [try内] KafkaPacemaker.wrapLongRunningMethod(context, () => createPullRequest())
    │  │    └── 每 3s 心跳保活，业务异常 re-throw 进入 catch
    │  ├── [try内] 成功: emit CREATE_PR_SUCCESS_TOPIC — 失败进入 catch
    │  └── [catch] emit CREATE_PR_FAILURE_TOPIC / CREATE_PR_SUCCESS_TOPIC(无变更)
    │       注意: catch 内 emit 失败 → 异常逃离 catch → Kafka 重投
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
4. **长任务保护**：Pacemaker 心跳机制在 git-sync-manager 中有效避免长任务因 Kafka 会话超时被重复投递

### ⚠️ 潜在风险点
1. **Redis 状态写入非原子**：`setJobStatus()` 的 read-modify-write 模式在极端并发下可能丢失状态更新
2. **无死信队列（DLT）——永久性错误会无限循环**：
   - `PACKAGE_MANAGER_CREATE_SUCCESS/FAILURE`（build-manager）和 `DSG_LOG_TOPIC`（server）的消费者无 try/catch，反复失败会无限次 Kafka 重投
   - git-sync-manager 的 `validateOrReject` 失败（消息格式错误）和 `key.toString()` 失败（消息 key 为 null）位于 try 块外，**坏消息会被 Kafka 无限重投，永远无法自愈**
3. **git-sync-manager key 解析 Bug——key 字段全为 undefined**：
   - `plainToInstance()` 直接接收 `key.toString()` 返回的 JSON 字符串，没有先 `JSON.parse()`，导致 `resourceRepositoryId` / `resourceId` 全部为 `undefined`
   - Key 从不经过 `validateOrReject` 校验，schema 上的 `@IsString()` 装饰器完全不生效
   - 后果：下游所有 CREATE_PR_SUCCESS/FAILURE、DOWNLOAD_PRIVATE_PLUGINS_SUCCESS/FAILURE 事件的 key 退化为 `{}`，失去按资源分区的能力；但由于下游消费者只读取 value，业务逻辑不受影响
4. **git-sync-manager catch 块内事件发送失败——异常二次逃逸**：
   - `CREATE_PR_REQUEST_TOPIC` 有 3 条逃逸路径（L118 日志发送、L123 成功事件发送、L153 失败事件发送）
   - `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` 有 1 条逃逸路径（L79 失败事件发送）
   - 这些路径在 catch 块内，若 Kafka broker 瞬时不可用，异常会逃离 catch，触发 Kafka 重投。恢复后该消息会被重新执行，git 操作可能重复（如创建重复 PR）
5. **无显式退避**：Kafka 级别的重试没有指数退避，瞬时故障可能引发消息风暴
6. **Stale Build 被动检测**：无人查询的僵尸任务永远停留在 Running 状态
7. **数据库锁非严格**：Block/Entity 锁的 check-then-act 模式存在 TOCTOU 竞态窗口（但仅用于用户前台编辑，影响面有限）
8. **build-manager 未使用 Pacemaker**：如果某个极端场景下 `runBuild()` 执行超过 30 秒（例如 Redis 慢查询），Consumer 可能因会话超时被踢出 Group 导致消息重投
9. **两套"锁"无关联但文档易混淆**：用户编辑锁与后台任务状态存储是完全独立的系统，但都被称为"锁"容易造成理解偏差
