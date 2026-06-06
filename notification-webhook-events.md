# Notification 与 Webhook 事件触发路径分析

## 整体架构概览

Amplication 的通知系统采用 **事件驱动架构**，通过 Kafka 消息队列实现服务间解耦：

```
业务服务 (amplication-server)
        │
        ▼ 发送 Kafka 消息
   Kafka 消息队列 (5 个 Topic)
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
| amplication-server | [packages/amplication-server](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server) | 业务服务，产生各类事件 |
| notification-service | [packages/notification-service](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service) | 消费 Kafka 消息并转发至 Novu |
| schema-registry | [libs/schema-registry](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/libs/schema-registry) | Kafka Topic 定义及消息 Schema |
| util/nestjs/kafka | [libs/util/nestjs/kafka](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/libs/util/nestjs/kafka) | Kafka Producer/Consumer 工具 |

---

## Kafka Topics 定义

所有 Topic 定义在 [schema-registry/src/index.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/libs/schema-registry/src/index.ts#L29-L69) 中的 `KAFKA_TOPICS` 枚举：

| Topic 常量 | Topic 实际值 | 触发场景 |
|------------|-------------|---------|
| `USER_ACTION_TOPIC` | `user-action.internal.1` | 用户注册/登录/切换工作区（订阅管理） |
| `USER_BUILD_TOPIC` | `user-build.internal.1` | 构建（代码生成）完成 |
| `USER_ANNOUNCEMENT_TOPIC` | `user-announcement.internal.1` | 功能公告发送 |
| `TECH_DEBT_CREATED_TOPIC` | `platform.internal.tech-debt.created.1` | 插件版本/模板版本/代码引擎版本过期告警 |

---

## 通知处理管道（notification-service）

### 消息消费入口

通知服务通过 NestJS Microservice 监听 Kafka 消息，入口在 [app.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service/src/app.controller.ts)：

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

所有监听器统一调用 [app.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service/src/app.service.ts#L27-L41) 中的 `notificationService()`，使用 **compose 中间件模式** 串行处理：

```typescript
compose(
  subscribeUser,       // 用户订阅管理（创建/删除 Novu subscriber）
  buildCompleted,      // 构建完成通知
  featureAnnouncement, // 功能公告通知
  techDebtAlert,       // 技术债务告警通知
  novuPackage          // 统一执行：遍历 notifications 数组调用 Novu API
)({ message, topic, novuService, amplicationLogger, notifications: [] })
```

每个中间件根据 `topic` 判断是否处理，符合条件则向 `notifications` 数组 push 一条通知任务，最后由 `novuPackage` 统一执行 Novu API 调用。

### Novu 集成

通知最终通过 [novuService.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service/src/util/novuService.ts) 发送，支持以下操作：

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

1. **消息发送**：[build.service.ts#L473-L492](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/build/build.service.ts#L473-L492)

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

2. **消息消费处理**：[buildCompleted.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service/src/notification-packages/buildCompleted.ts)

3. **消息 Schema**：[user-build/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/libs/schema-registry/src/lib/user-build/value.ts)

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

1. **版本发布入口**：[resourceVersion.service.ts#L96-L109](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L96-L109)

2. **插件版本告警触发**：[resourceVersion.service.ts#L114-L169](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L114-L169)

3. **创建告警并发送通知**：[outdatedVersionAlert.service.ts#L45-L124](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L45-L124)

4. **通知处理**：[techDebtAlert.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service/src/notification-packages/techDebtAlert.ts)

5. **消息 Schema**：[tech-debt/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/libs/schema-registry/src/lib/tech-debt/value.ts)

### 告警类型

在 [tech-debt/value.ts#L39-L43](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/libs/schema-registry/src/lib/tech-debt/value.ts#L39-L43) 中定义了三种技术债务告警类型：

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

**场景**：用户登录或切换工作区时，在 Novu 中创建/删除订阅者，为后续通知做准备。

### 完整路径图

```
用户登录 / 切换工作区
    │
    ▼ (Auth 流程中调用)
UserService.setNotificationRegistry(user)     [user.service.ts#L175-L202]
    │
    ▼
检查 BillingFeature.Notification 权限 → canShowUserNotification
    │
    ▼
发送 USER_ACTION_TOPIC Kafka 消息
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

1. **消息发送**：[user.service.ts#L175-L202](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/user/user.service.ts#L175-L202)

   ```typescript
   this.kafkaProducerService.emitMessage(KAFKA_TOPICS.USER_ACTION_TOPIC, <UserAction.KafkaEvent>{
     value: {
       userId, externalId: encryptString(user.id),
       firstName, lastName, email,
       action: UserActionType.CURRENT_WORKSPACE,
       enableUser: canShowUserNotification,
     },
   })
   ```

2. **订阅处理**：[subscribeUser.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service/src/notification-packages/subscribeUser.ts)

3. **消息 Schema / Action 类型**：[user-action/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/libs/schema-registry/src/lib/user-action/value.ts)

### UserActionType 枚举

```typescript
enum UserActionType {
  SIGNUP = "signup",
  LOGIN = "login",
  UPDATE = "update",
  DELETE = "delete",
  CURRENT_WORKSPACE = "currentWorkspace", // 实际使用的类型
}
```

> **注意**：当前代码中 `setNotificationRegistry` 固定使用 `CURRENT_WORKSPACE` action 类型，而 subscribeUser 中间件并未区分 action 类型，仅根据 `enableUser` 布尔值判断创建或删除订阅者。

### 关于"成员邀请"的说明

成员邀请（邀请邮件发送）**不走 Kafka 通知管道**，而是直接调用邮件服务：

- 触发位置：[workspace.service.ts#L232-L308](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L232-L308) `inviteUser()`
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

1. **消息发送**：[user.service.ts#L204-L269](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/user/user.service.ts#L204-L269)

2. **通知处理**：[featureAnnouncement.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/notification-service/src/notification-packages/featureAnnouncement.ts)

### 设计要点

- `notificationTemplateIdentifier` 由调用方传入，直接映射为 Novu 中的工作流模板标识
- 消息中附带 `envBaseUrl`、`workspaceId`、`projectId`、`serviceId` 用于通知模板渲染跳转链接

---

## 数据流向总结表

| 事件类型 | 发送方 Service | Kafka Topic | Notification 中间件 | Novu 调用方法 | Novu eventName |
|---------|---------------|-------------|---------------------|--------------|----------------|
| 用户订阅 | UserService | `user-action.internal.1` | subscribeUser | createSubscriber / deleteSubscriber | - |
| 构建完成 | BuildService | `user-build.internal.1` | buildCompleted | triggerNotificationToSubscriber | `build-completed` |
| 插件过期告警 | OutdatedVersionAlertService | `platform.internal.tech-debt.created.1` | techDebtAlert | triggerNotificationToSubscriber | `technical-debt-alert` |
| 模板过期告警 | OutdatedVersionAlertService | `platform.internal.tech-debt.created.1` | techDebtAlert | triggerNotificationToSubscriber | `technical-debt-alert` |
| 功能公告 | UserService | `user-announcement.internal.1` | featureAnnouncement | triggerNotificationToSubscriber | 动态（模板标识符） |

---

## 设计模式与特点

### 1. 中间件管道（Middleware Pipeline）

`compose()` 函数实现了洋葱模型的简化版，每个 notification-package 模块职责单一：判定 topic → 构造 Notification 对象 → 推入队列。最后由 `novuPackage` 统一批量执行。

优点：
- 新增通知类型只需新增一个 notification-package 文件并加入 compose 链
- 各模块之间完全解耦，topic 判定逻辑内聚

### 2. 外部 ID 加密

所有用户标识在跨服务传递时通过 `encryptString(userId)` 加密（见于 [build.service.ts#L486](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/build/build.service.ts#L486)、[user.service.ts#L176](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/user/user.service.ts#L176)、[outdatedVersionAlert.service.ts#L110](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L110)），避免明文用户 ID 在消息系统中传播。

### 3. 面向工作区用户广播

插件/模板告警采用"按用户逐条发送"模式：在 `raiseNotifications()` 中先查询 workspace 的所有用户，然后为每人发送一条 Kafka 消息。相比 topic 广播模式，这种方式更灵活（可基于用户维度做过滤），但用户量大时消息数量线性增长。

### 4. 事件溯源（Action + ActionStep + ActionLog）

构建任务执行过程伴随完整的 Action → ActionStep → ActionLog 三层记录模型（见 [action.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/52-amplication/packages/amplication-server/src/core/action/action.service.ts)），但此模型仅用于任务状态追踪，**不直接触发通知**。通知仅在关键业务节点（代码生成成功、插件新版本发布等）显式触发。
