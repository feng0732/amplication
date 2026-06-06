# Notification 与 Webhook 事件触发路径分析

> **路径说明**：本文档中所有文件链接均为**仓库根目录下的相对路径**。将仓库克隆到任意机器后，以仓库根目录为基准即可定位到对应文件。例如 `packages/amplication-server/src/core/user/user.service.ts` 对应仓库根目录下的同名文件。

## 整体架构概览

Amplication 的通知系统采用 **事件驱动架构**，通过 Kafka 消息队列实现服务间解耦：

```
业务服务 (amplication-server)
        │
        ▼ 发送 Kafka 消息
   Kafka 消息队列 (notification-service 监听 5 个 Topic，其中 4 个有完整链路)
        │
        ▼ 消费消息
  notification-service (微服务)
        │
        ▼ 调用
       Novu (第三方通知服务)
        │
        ▼ 最终触达
   邮件 / In-App / 推送 等
```

### 核心模块

| 模块 | 路径 | 职责 |
|------|------|------|
| amplication-server | [packages/amplication-server](packages/amplication-server) | 业务服务，产生各类事件 |
| notification-service | [packages/notification-service](packages/notification-service) | 消费 Kafka 消息并转发至 Novu |
| schema-registry | [libs/schema-registry](libs/schema-registry) | Kafka Topic 定义及消息 Schema |
| util/nestjs/kafka | [libs/util/nestjs/kafka](libs/util/nestjs/kafka) | Kafka Producer/Consumer 工具 |

---

## Kafka Topics 定义

notification-service 在 [app.controller.ts](packages/notification-service/src/app.controller.ts) 中共监听 **5 个 Kafka Topic**，但其中只有 4 个在 schema-registry 中有常量定义且具备完整的发送方 → 处理包链路。

### Schema Registry 中定义的 4 个 Topic（完整链路）

定义位置：[schema-registry/src/index.ts](libs/schema-registry/src/index.ts#L29-L69) 中的 `KAFKA_TOPICS` 枚举：

| Topic 常量 | Topic 实际值 | 触发场景 |
|------------|-------------|---------|
| `USER_ACTION_TOPIC` | `user-action.internal.1` | 用户注册/登录/切换工作区（订阅管理） |
| `USER_BUILD_TOPIC` | `user-build.internal.1` | 构建（代码生成）完成 |
| `USER_ANNOUNCEMENT_TOPIC` | `user-announcement.internal.1` | 功能公告发送 |
| `TECH_DEBT_CREATED_TOPIC` | `platform.internal.tech-debt.created.1` | 插件版本/模板版本/代码引擎版本过期告警 |

### 未注册的第 5 个 Topic：`user-preview-generation-completed.internal.1`

该 Topic **仅出现在 notification-service 的 `@EventPattern` 硬编码字符串中**，状态如下：

| 检查项 | 结果 | 证据 |
|-------|------|------|
| 是否在 KAFKA_TOPICS 枚举中定义 | ❌ 无 | [schema-registry/src/index.ts](libs/schema-registry/src/index.ts#L29-L69) 无对应常量 |
| 是否有 schema（key/value 类型） | ❌ 无 | `libs/schema-registry/src/lib/` 下无 preview 相关子目录 |
| 是否有发送方代码 | ❌ 无 | 全仓库搜索无任何 `emitMessage`/`send` 相关引用 |
| 是否有对应 notification-packages 处理包 | ❌ 无 | `notification-packages/` 目录下只有 4 个处理包（见下） |
| compose 中间件是否能处理 | ❌ 不能 | 所有中间件的 topic 判断均不匹配 |

> **结论**：`user-preview-generation-completed.internal.1` 是一个**预留的空监听器**，目前不会产生任何通知。即使该 topic 有消息到达，也会经过 compose 管道但没有中间件匹配 topic，最终 `notifications` 数组为空，不执行任何 Novu 调用。

---

## 通知处理管道（notification-service）

### 消息消费入口

通知服务通过 NestJS Microservice 监听 Kafka 消息，入口在 [app.controller.ts](packages/notification-service/src/app.controller.ts)：

```typescript
@EventPattern("user-action.internal.1")
subscribeNotification(@Payload() message, @Ctx() context: KafkaContext) { ... }

@EventPattern("user-build.internal.1")
notifyBuild(@Payload() message, @Ctx() context: KafkaContext) { ... }

@EventPattern("user-announcement.internal.1")
notifyFeatureAnnouncement(@Payload() message, @Ctx() context: KafkaContext) { ... }

@EventPattern("user-preview-generation-completed.internal.1")
previewUserGenerationCompleted(@Payload() message, @Ctx() context: KafkaContext) { ... }

@EventPattern("platform.internal.tech-debt.created.1")
notifyTechDebt(@Payload() message, @Ctx() context: KafkaContext) { ... }
```

### 处理管道（Pipeline）

所有监听器统一调用 [app.service.ts](packages/notification-service/src/app.service.ts#L27-L41) 中的 `notificationService()`，使用 **compose 中间件模式** 串行处理：

```typescript
compose(
  subscribeUser,       // topic: user-action.internal.1          → 用户订阅管理（创建/删除 Novu subscriber）
  buildCompleted,      // topic: user-build.internal.1            → 构建完成通知
  featureAnnouncement, // topic: user-announcement.internal.1     → 功能公告通知
  techDebtAlert,       // topic: platform.internal.tech-debt.created.1 → 技术债务告警通知
  novuPackage          // 统一执行：遍历 notifications 数组调用 Novu API
)({ message, topic, novuService, amplicationLogger, notifications: [] })
```

每个中间件根据 `topic` 判断是否处理，符合条件则向 `notifications` 数组 push 一条通知任务，最后由 `novuPackage` 统一执行 Novu API 调用。

> **注意**：compose 链中只有 **4 个业务通知中间件**，对应 4 个已注册的 Kafka Topic。`user-preview-generation-completed.internal.1` 没有对应的中间件，消息到达后 `notifications` 数组保持为空，不产生任何 Novu 调用。

### notification-packages 目录清单

实际文件位于 [packages/notification-service/src/notification-packages/](packages/notification-service/src/notification-packages/)：

| 处理包文件 | 对应 Topic |
|-----------|-----------|
| `subscribeUser.ts` | `user-action.internal.1` |
| `buildCompleted.ts` | `user-build.internal.1` |
| `featureAnnouncement.ts` | `user-announcement.internal.1` |
| `techDebtAlert.ts` | `platform.internal.tech-debt.created.1` |

目录中不存在 `previewUserGenerationCompleted.ts` 或类似文件，进一步印证该 topic 尚无处理逻辑。

### Novu 集成

通知最终通过 [novuService.ts](packages/notification-service/src/util/novuService.ts) 发送，支持以下操作：

- `createSubscriber()` / `updateSubscriber()` / `deleteSubscriber()` — 管理订阅者
- `triggerNotificationToSubscriber()` — 向指定用户触发通知（eventName 对应 Novu 中的工作流模板）
- `broadCastEventToAll()` — 广播给所有用户
- `addSubscribersToTopic()` / `removeSubscribersFromTopic()` — 话题订阅管理

---

## 一、任务（Build）事件路径

**场景**：用户触发构建，代码生成成功后向用户发送"构建完成"通知。

### 完整路径图

```
用户触发构建
    │
    ▼
BuildService.create()           [build.service.ts]
    │
    ▼
发送 CODE_GENERATION_REQUEST_TOPIC  →  data-service-generator 执行代码生成
    │
    ▼ (代码生成成功后 DSG 回传)
BuildController.onCodeGenerationSuccess()
    │
    ▼
BuildService.onCodeGenerationSuccess()   [build.service.ts#L450-L495]
    │
    ▼
发送 USER_BUILD_TOPIC Kafka 消息
    │
    ▼
notification-service AppController.notifyBuild()
    │
    ▼
buildCompleted() 中间件处理        [buildCompleted.ts]
    │
    ▼
novuService.triggerNotificationToSubscriber(eventName: "build-completed")
```

### 关键代码位置

1. **消息发送**：[build.service.ts#L473-L492](packages/amplication-server/src/core/build/build.service.ts#L473-L492)

   ```typescript
   this.kafkaProducerService.emitMessage(KAFKA_TOPICS.USER_BUILD_TOPIC, <UserBuild.KafkaEvent>{
     key: {},
     value: {
       buildId, commitId, commitMessage, resourceId, resourceName,
       workspaceId, projectId, projectName,
       createdAt: Date.now(),
       externalId: encryptString(commitWithAccount.commit.user.id),
       envBaseUrl: this.configService.get<string>(Env.CLIENT_HOST),
     },
   })
   ```

2. **消息消费处理**：[buildCompleted.ts](packages/notification-service/src/notification-packages/buildCompleted.ts)

3. **消息 Schema**：[user-build/value.ts](libs/schema-registry/src/lib/user-build/value.ts)

### 消息体字段说明

| 字段 | 说明 |
|------|------|
| `externalId` | 加密后的用户 ID，作为 Novu subscriberId |
| `buildId` / `commitId` | 构建和提交标识 |
| `resourceId` / `projectId` / `workspaceId` | 资源层级信息 |
| `shortBuildId` | 处理时截取 buildId 末尾 8 位用于展示 |
| `createdAt` | 时间戳，处理时格式化为可读日期 |

---

## 二、插件（Plugin）事件路径

**场景**：Plugin Repository（插件仓库）发布新版本后，向所有安装了该插件的服务用户发送"插件版本过期"告警。

### 完整路径图

```
发布 Plugin Repository 新版本
    │
    ▼
ResourceVersionService.create()    [resourceVersion.service.ts#L40-L112]
    │
    ▼ resourceType === PluginRepository
checkForAlertsForNewPrivatePluginVersions()   [resourceVersion.service.ts#L114-L169]
    │
    ▼ 对比版本差异，找出有新版本的 PrivatePlugin
OutdatedVersionAlertService.triggerAlertsForNewPluginVersion()
                                                [outdatedVersionAlert.service.ts#L256-L319]
    │
    ▼ 遍历所有安装了该 pluginId 的 pluginInstallation
OutdatedVersionAlertService.create()           [outdatedVersionAlert.service.ts#L45-L73]
    │
    ▼
raiseNotifications()                           [outdatedVersionAlert.service.ts#L75-L124]
    │
    ▼ 遍历 workspaceUsers，为每个用户发送
发送 TECH_DEBT_CREATED_TOPIC Kafka 消息（每人一条）
    │
    ▼
notification-service AppController.notifyTechDebt()
    │
    ▼
techDebtAlert() 中间件处理                     [techDebtAlert.ts]
    │
    ▼
novuService.triggerNotificationToSubscriber(eventName: "technical-debt-alert")
```

### 关键代码位置

1. **版本发布入口**：[resourceVersion.service.ts#L96-L109](packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L96-L109)

2. **插件版本告警触发**：[resourceVersion.service.ts#L114-L169](packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L114-L169)

3. **创建告警并发送通知**：[outdatedVersionAlert.service.ts#L45-L124](packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L45-L124)

4. **通知处理**：[techDebtAlert.ts](packages/notification-service/src/notification-packages/techDebtAlert.ts)

5. **消息 Schema**：[tech-debt/value.ts](libs/schema-registry/src/lib/tech-debt/value.ts)

### 告警类型

在 [tech-debt/value.ts#L39-L43](libs/schema-registry/src/lib/tech-debt/value.ts#L39-L43) 中定义了三种技术债务告警类型：

| 类型 | 触发场景 |
|------|---------|
| `PluginVersion` | 插件有新版本可用 |
| `TemplateVersion` | ServiceTemplate 有新版本可用 |
| `CodeEngineVersion` | 代码引擎版本更新（目前未见显式触发） |

### 模板版本过期（同类路径）

ServiceTemplate 新版本发布同样通过 TECH_DEBT_CREATED_TOPIC 发送通知，路径类似：

```
ResourceVersionService.create()
    │ resourceType === ServiceTemplate
    ▼
triggerAlertsForTemplateVersion()            [outdatedVersionAlert.service.ts#L179-L246]
    │
    ▼
OutdatedVersionAlertService.create() → raiseNotifications()
    │
    ▼
发送 TECH_DEBT_CREATED_TOPIC (alertType: "TemplateVersion")
```

---

## 三、成员（User/Member）事件路径

**场景**：前端请求 `currentWorkspace` GraphQL Query 时，同步检查通知权限并在 Novu 中创建/删除订阅者，为后续通知做准备。

### 完整路径图

```
前端打开工作区页面
    │
    ▼ 调用 GraphQL Query
WorkspaceResolver.currentWorkspace(currentUser)
                                    [workspace.resolver.ts#L79-L92]
    │  执行顺序:
    │  ① analytics.trackWithContext(WorkspaceSelected)
    │  ② setLastActivity(userId)
    │  ③ setNotificationRegistry(user) ← 调用通知注册
    │
    ▼
UserService.setNotificationRegistry(user)     [user.service.ts#L175-L202]
    │
    ├─ 生成 externalId = encryptString(user.id)
    ├─ 调用 BillingService.getBooleanEntitlement(
    │      user.workspace.id, BillingFeature.Notification)
    │      → canShowUserNotification = hasAccess
    │
    └─ 发送 USER_ACTION_TOPIC Kafka 消息:
          action = UserActionType.CURRENT_WORKSPACE
          enableUser = canShowUserNotification
    │
    ▼
notification-service AppController.subscribeNotification()
    │
    ▼
subscribeUser() 中间件处理                    [subscribeUser.ts]
    │
    ├─ enableUser = true  → novuService.createSubscriber()
    └─ enableUser = false → novuService.deleteSubscriber()
```

### 关键代码位置

1. **GraphQL 触发入口（唯一调用点）**：[workspace.resolver.ts#L79-L92](packages/amplication-server/src/core/workspace/workspace.resolver.ts#L79-L92)

   `setNotificationRegistry` 在全仓库中**只被这一处调用**，不存在于 Auth 登录流程或其他位置。

   ```typescript
   @Query(() => Workspace, { nullable: true })
   async currentWorkspace(@UserEntity() currentUser: User): Promise<Workspace | null> {
     await this.analytics.trackWithContext({ event: EnumEventType.WorkspaceSelected });
     await this.userService.setLastActivity(currentUser.id);
     const externalId = await this.userService.setNotificationRegistry(currentUser);  // ← 唯一调用
     return { ...currentUser.workspace, externalId };
   }
   ```

2. **消息发送（setNotificationRegistry 实现）**：[user.service.ts#L175-L202](packages/amplication-server/src/core/user/user.service.ts#L175-L202)

   ```typescript
   async setNotificationRegistry(user: User) {
     const externalId = encryptString(user.id);
     const booleanEntityUserNotification =
       await this.billingService.getBooleanEntitlement(
         user.workspace.id,
         BillingFeature.Notification          // feature-notifications
       );
     const canShowUserNotification = booleanEntityUserNotification?.hasAccess;

     this.kafkaProducerService.emitMessage(KAFKA_TOPICS.USER_ACTION_TOPIC, <UserAction.KafkaEvent>{
       key: {},
       value: {
         userId: user.account.id,
         externalId,
         firstName: user.account.firstName,
         lastName: user.account.lastName,
         email: user.account.email,
         action: UserAction.UserActionType.CURRENT_WORKSPACE,  // ← 固定值
         enableUser: canShowUserNotification,                  // ← 由权限决定
       },
     });
     return externalId;
   }
   ```

3. **订阅处理**：[subscribeUser.ts](packages/notification-service/src/notification-packages/subscribeUser.ts)

4. **消息 Schema / Action 类型**：[user-action/value.ts](libs/schema-registry/src/lib/user-action/value.ts)

### UserActionType 枚举与实际使用的对应关系

```typescript
enum UserActionType {
  SIGNUP = "signup",
  LOGIN = "login",
  UPDATE = "update",
  DELETE = "delete",
  CURRENT_WORKSPACE = "currentWorkspace", // ← setNotificationRegistry 固定使用此值
}
```

> **关键点**：
> - `CURRENT_WORKSPACE` 不是"用户切换工作区"的语义，而是对应 `currentWorkspace` 这个 GraphQL Query 的命名。每次调用该 Query（如前端打开工作区页面、刷新页面等）都会触发一次通知注册同步。
> - subscribeUser 中间件**并不区分** action 类型，仅根据 `enableUser` 布尔值判断创建或删除 Novu 订阅者。

### Stigg Webhook → 订阅更新 → 通知权限变化 的完整链路

这是最容易被忽略的一条链路：**Stigg 支付系统的订阅状态变更通过 Webhook 写入数据库，进而影响用户的 BillingFeature.Notification 权限，最终决定 Novu 订阅者的创建/删除**。

#### 完整路径图

```
Stigg 支付系统（订阅创建/升级/降级/取消/过期/促销权益变更）
    │ HTTP POST，header 带 stigg-webhooks-secret
    ▼
SubscriptionController.updateStatus()     [subscription.controller.ts#L20-L32]
    │  POST /subscriptions/updateStatus
    │  校验 header 中的 stigg-webhooks-secret
    ▼
SubscriptionService.handleUpdateSubscriptionStatusEvent()
                                            [subscription.service.ts#L261-L332]
    │
    ├═══════════════════════════════════════════════════════════════╗
    │ 分支 A: subscription.*（4 种事件）                               ║
    │   "subscription.created" / "subscription.updated"              ║
    │   "subscription.expired" / "subscription.canceled"             ║
    │                                                                 ║
    │   switch 内执行:                                                 ║
    │   → prisma.subscription.upsert()  ← 所有 4 种事件都执行        ║
    │       create: id, workspaceId, status, subscriptionPlan        ║
    │       update: 仅 status                                          ║
    │                                                                 ║
    │   switch 后追加执行（仅部分事件）:                                ║
    │   → subscription.created: trackUpgradeCompletedEvent()          ║
    │   → subscription.created/.updated:                              ║
    │         updateProjectLicensed(workspaceId)                      ║
    │         updateServiceLicensed(workspaceId)                      ║
    │   → subscription.expired/.canceled: ❌ 不更新 licensed 字段    ║
    ├═══════════════════════════════════════════════════════════════╣
    │ 分支 B: promotionalEntitlement.*（4 种事件）                     ║
    │   "promotionalEntitlement.granted" / ".updated"                ║
    │   "promotionalEntitlement.revoked" / ".expired"                ║
    │                                                                 ║
    │   switch 内执行:                                                 ║
    │   → ❌ 不写 subscription 表                                     ║
    │   → ✅ 所有 4 种事件都执行:                                     ║
    │         updateProjectLicensed(workspaceId)                      ║
    │         updateServiceLicensed(workspaceId)                      ║
    ┚═══════════════════════════════════════════════════════════════╝
    │
    │ ⚠  重要: 以上所有分支都不会立即发送任何 Kafka 通知消息
    │    Stigg Webhook 只写 DB，不直接触发 Novu 同步
    │
    ▼ （用户下次刷新前端页面或访问工作区）
WorkspaceResolver.currentWorkspace()      [workspace.resolver.ts#L79-L92]
    │  GraphQL Query: currentWorkspace
    ▼
UserService.setNotificationRegistry(user)  [user.service.ts#L175-L202]
    │
    ▼
BillingService.getBooleanEntitlement(workspaceId, BillingFeature.Notification)
                                            [billing.service.ts#L212-L227]
    │  从 Stigg SDK 实时拉取布尔权限（不是读 DB 的 subscription 表）
    │  BillingFeature.Notification = "feature-notifications"
    ▼
  hasAccess = true / false → canShowUserNotification
    │
    ▼
发送 USER_ACTION_TOPIC Kafka 消息
    value.enableUser = canShowUserNotification
    │
    ▼
notification-service AppController.subscribeNotification()
    │
    ▼
subscribeUser() 中间件                    [subscribeUser.ts]
    │
    ├─ enableUser = true  → novuService.createSubscriber() / updateSubscriber()
    └─ enableUser = false → novuService.deleteSubscriber()
```

#### 关键代码位置

1. **Stigg Webhook 入口**：[subscription.controller.ts#L20-L32](packages/amplication-server/src/core/subscription/subscription.controller.ts#L20-L32)

   ```typescript
   @Post("updateStatus")
   async updateStatus(
     @Headers("stigg-webhooks-secret") stiggWebhooksSecret,
     @Body() updateStatusDto: UpdateStatusDto
   ): Promise<void> {
     if (stiggWebhooksSecret !== this.stiggWebhooksSecret) {
       throw new Error("Invalid stigg-webhooks-secret");
     }
     await this.subscriptionService.handleUpdateSubscriptionStatusEvent(updateStatusDto);
   }
   ```

2. **订阅状态事件处理（handleUpdateSubscriptionStatusEvent）**：[subscription.service.ts#L261-L332](packages/amplication-server/src/core/subscription/subscription.service.ts#L261-L332)

   两个分支的 DB 影响对比：

   | 事件类型 | 分支 | subscription 表 upsert | 写 subscriptionPlan | 写 subscription.status | 更新 Project/Service licensed | 其他副作用 |
   |---------|------|----------------------|-------------------|----------------------|---------------------------|-----------|
   | `subscription.created` | A | ✅ | ✅（首次创建） | ✅ | ✅ | trackUpgradeCompletedEvent |
   | `subscription.updated` | A | ✅（update） | ❌（仅首次创建时写） | ✅ | ✅ | - |
   | `subscription.expired` | A | ✅（update） | ❌ | ✅ | ❌ | - |
   | `subscription.canceled` | A | ✅（update） | ❌ | ✅ | ❌ | - |
   | `promotionalEntitlement.granted` | B | ❌（不写表） | - | - | ✅ | - |
   | `promotionalEntitlement.updated` | B | ❌（不写表） | - | - | ✅ | - |
   | `promotionalEntitlement.revoked` | B | ❌（不写表） | - | - | ✅ | - |
   | `promotionalEntitlement.expired` | B | ❌（不写表） | - | - | ✅ | - |

   > **代码逻辑要点**：
   > - 分支 A（subscription.*）的 upsert 和分支 B（promotionalEntitlement.*）的 licensed 更新都在 **switch case 内**完成
   > - subscription.created/.updated 的 **licensed 更新**在 **switch 之后**的 `if` 条件中执行（L320-L331）
   > - subscription.expired/.canceled **故意不触发** licensed 更新
   > - promotionalEntitlement.* **完全不涉及** subscription 表写入

3. **updateProjectLicensed / updateServiceLicensed 具体作用**：
   - [subscription.service.ts#L103-L174](packages/amplication-server/src/core/subscription/subscription.service.ts#L103-L174)
   - 逻辑：根据 Stigg 返回的 metered entitlement 限额（BillingFeature.Projects / Services），按创建时间升序对 workspace 下的 project/service 排序，限额内的设 `licensed: true`，超出限额的设 `licensed: false`
   - **与 Notification 权限无关**：licensed 字段控制的是项目/服务是否"有许可证"，而 Notification 权限来自 BillingFeature.Notification 的布尔 entitlement，通过 Stigg SDK 实时查询

4. **Notification Billing Feature 定义**：[billing-feature.types.ts#L21](libs/util/billing-types/src/lib/billing-feature.types.ts#L21)
   ```typescript
   Notification = "feature-notifications"
   ```

5. **用户侧触发点（权限拉取 + Kafka 发送）**：
   - GraphQL 入口（唯一调用点）：[workspace.resolver.ts#L79-L92](packages/amplication-server/src/core/workspace/workspace.resolver.ts#L79-L92)
   - 权限检查 + Kafka 发送：[user.service.ts#L175-L202](packages/amplication-server/src/core/user/user.service.ts#L175-L202)

#### 设计要点与注意事项

- **延迟同步（非实时）**：Stigg Webhook 只负责写入数据库，**不会立即向 Kafka 发消息**。必须等到用户下次请求 `currentWorkspace`（通常是前端刷新或打开工作区页面）时，才会通过 `setNotificationRegistry()` 重新拉取 Stigg 权限并同步 Novu 订阅者状态。

- **通知权限判断不读本地 subscription 表**：`setNotificationRegistry` 中调用 `BillingService.getBooleanEntitlement(workspaceId, BillingFeature.Notification)` 是通过 **Stigg SDK 实时查询**（不是读 DB 的 subscription 表）。DB 的 subscription 表仅用于其他业务逻辑（如页面展示订阅状态），Notification 的 enableUser 完全由 Stigg 实时返回的 hasAccess 决定。

- **subscription.expired / canceled 不刷新 licensed 的意图**：设计上认为订阅过期/取消后，无需立即回收已授权项目/服务的 licensed 标记，仅在订阅 created/updated（付费升级/降级）或促销权益变化时刷新限额。

- **promotionalEntitlement 不写 subscription 表的意图**：促销权益（试用、赠送额度等）是临时的、叠加在主订阅之上的权益，不需要持久化到 subscription 表，只需要反映在 project/service 的 licensed 限额上即可。

- **licensed 字段与 Notification 无关**：`updateProjectLicensed()` 和 `updateServiceLicensed()` 更新的是项目和服务的 `licensed` 布尔字段，用于控制代码生成等功能是否可用，与通知功能的开关（BillingFeature.Notification）是两条独立的权限链路。

### 关于"成员邀请"的说明

成员邀请（邀请邮件发送）**不走 Kafka 通知管道**，而是直接调用邮件服务：

- 触发位置：[workspace.service.ts#L232-L308](packages/amplication-server/src/core/workspace/workspace.service.ts#L232-L308) `inviteUser()`
- 直接调用：`this.mailService.sendInvitation(...)`

邀请接受（`completeInvitation`）也不会产生通知，仅更新数据模型和上报 Analytics。

---

## 四、功能公告（Feature Announcement）路径

**场景**：管理员向指定活跃用户批量发送新功能公告。

### 完整路径图

```
管理员调用接口
    │
    ▼
UserService.notifyUserFeatureAnnouncement()   [user.service.ts#L204-L269]
    │
    ▼
查询最近 N 天活跃的用户（含 workspace/project/service 信息）
    │
    ▼ 遍历用户
发送 USER_ANNOUNCEMENT_TOPIC Kafka 消息（每人一条）
    │
    ▼
notification-service AppController.notifyFeatureAnnouncement()
    │
    ▼
featureAnnouncement() 中间件处理              [featureAnnouncement.ts]
    │
    ▼
novuService.triggerNotificationToSubscriber(eventName: notificationTemplateIdentifier)
```

### 关键代码位置

1. **消息发送**：[user.service.ts#L204-L269](packages/amplication-server/src/core/user/user.service.ts#L204-L269)

2. **通知处理**：[featureAnnouncement.ts](packages/notification-service/src/notification-packages/featureAnnouncement.ts)

### 设计要点

- `notificationTemplateIdentifier` 由调用方传入，直接映射为 Novu 中的工作流模板标识
- 消息中附带 `envBaseUrl`、`workspaceId`、`projectId`、`serviceId` 用于通知模板渲染跳转链接

---

## 数据流向总结表

| 事件类型 | 触发源头 | 发送方 Service | Kafka Topic | Notification 中间件 | Novu 调用方法 | Novu eventName |
|---------|---------|---------------|-------------|---------------------|--------------|----------------|
| 用户订阅（含订阅权限变化） | 前端调用 `currentWorkspace` GraphQL Query（Stigg Webhook 会更新 DB，但需等用户下次访问工作区才会同步 Novu） | UserService | `user-action.internal.1` | subscribeUser | createSubscriber / deleteSubscriber | - |
| 构建完成 | DSG 代码生成成功回调 | BuildService | `user-build.internal.1` | buildCompleted | triggerNotificationToSubscriber | `build-completed` |
| 插件过期告警 | Plugin Repository 发布新版本 → ResourceVersion.create | OutdatedVersionAlertService | `platform.internal.tech-debt.created.1` | techDebtAlert | triggerNotificationToSubscriber | `technical-debt-alert` |
| 模板过期告警 | ServiceTemplate 发布新版本 → ResourceVersion.create | OutdatedVersionAlertService | `platform.internal.tech-debt.created.1` | techDebtAlert | triggerNotificationToSubscriber | `technical-debt-alert` |
| 功能公告 | 管理员手动调用接口 | UserService | `user-announcement.internal.1` | featureAnnouncement | triggerNotificationToSubscriber | 动态（模板标识符） |
| Stigg 订阅 Webhook（不直接发通知） | Stigg 支付系统推送 | SubscriptionService | **不直接发 Kafka** | 分支 A（subscription.*）：写 subscription 表；分支 B（promotionalEntitlement.*）：只更新 project/service.licensed；均不直接触发 Novu 同步 | - | - |
| Preview 生成完成（预留，未实现） | 无 | 无发送方 | `user-preview-generation-completed.internal.1` | 无对应中间件 | - | - |

---

## 设计模式与特点

### 1. 中间件管道（Middleware Pipeline）

`compose()` 函数实现了洋葱模型的简化版，每个 notification-package 模块职责单一：判定 topic → 构造 Notification 对象 → 推入队列。最后由 `novuPackage` 统一批量执行。

优点：
- 新增通知类型只需新增一个 notification-package 文件并加入 compose 链
- 各模块之间完全解耦，topic 判定逻辑内聚

### 2. 外部 ID 加密

所有用户标识在跨服务传递时通过 `encryptString(userId)` 加密（见于 [build.service.ts#L486](packages/amplication-server/src/core/build/build.service.ts#L486)、[user.service.ts#L176](packages/amplication-server/src/core/user/user.service.ts#L176)、[outdatedVersionAlert.service.ts#L110](packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L110)），避免明文用户 ID 在消息系统中传播。

### 3. 面向工作区用户广播

插件/模板告警采用"按用户逐条发送"模式：在 `raiseNotifications()` 中先查询 workspace 的所有用户，然后为每人发送一条 Kafka 消息。相比 topic 广播模式，这种方式更灵活（可基于用户维度做过滤），但用户量大时消息数量线性增长。

### 4. 事件溯源（Action + ActionStep + ActionLog）

构建任务执行过程伴随完整的 Action → ActionStep → ActionLog 三层记录模型（见 [action.service.ts](packages/amplication-server/src/core/action/action.service.ts)），但此模型仅用于任务状态追踪，**不直接触发通知**。通知仅在关键业务节点（代码生成成功、插件新版本发布等）显式触发。

### 5. Stigg Webhook 延迟同步（Pull-on-Read）

订阅/权限变化的同步采用 **Webhook 写 DB + 用户请求时拉取权限** 的两段式设计，而非 Webhook 直接推送通知：

1. **写路径（Webhook → DB）**：Stigg Webhook 触发时，`handleUpdateSubscriptionStatusEvent()` 按事件类型分流：
   - `subscription.created/updated/expired/canceled` 写入或更新 `subscription` 表，其中 created/updated 才会刷新 project/resource 的 `licensed` 字段
   - `promotionalEntitlement.*` 不写 `subscription` 表，只刷新 project/resource 的 `licensed` 字段
   
2. **读路径（用户请求 → 实时权限 → Novu 同步）**：用户每次请求 `currentWorkspace` 时：
   - 调 `BillingService.getBooleanEntitlement(workspaceId, BillingFeature.Notification)` 从 Stigg SDK 实时拉取布尔权限
   - 用权限结果填充 `enableUser` 字段，发送 `USER_ACTION_TOPIC` Kafka 消息
   - notification-service 消费后在 Novu 中 createSubscriber/deleteSubscriber

**这种设计的考量**：
- Webhook 侧无需感知 Novu，职责单一（仅持久化订阅状态）
- 权限判断走 Stigg SDK 实时查询，避免 DB 缓存与 Stigg 实际状态不一致
- 以用户访问为自然触发点，不必遍历所有工作区用户批量同步，节省资源
- **代价**：订阅变更后用户如果一直不访问系统，Novu 侧的订阅者状态不会被刷新

### 6. 预留空监听器（Skeleton Listener）

`user-preview-generation-completed.internal.1` 是一个典型的预留监听器：在 [app.controller.ts](packages/notification-service/src/app.controller.ts#L44-L52) 中声明了 `@EventPattern`，但未配置以下任何一项：

| 组件 | 是否存在 | 说明 |
|------|---------|------|
| KAFKA_TOPICS 枚举常量 | ❌ | schema-registry 中未定义 |
| Kafka message schema（key/value） | ❌ | 无 TypeScript 类型约束 |
| 发送方（producer） | ❌ | 全仓库无 `emitMessage` 调用 |
| notification-packages 处理包 | ❌ | compose 链中无对应中间件 |

这种设计的意图通常是**提前为未来功能预留 Topic 接入点**，避免后续上线时需要修改 notification-service 的部署配置（Kafka consumer 订阅列表）。但目前该 Topic 不会产生任何实际效果。
