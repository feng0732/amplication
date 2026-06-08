# Amplication Billing 与 Plan 配额限制代码走向梳理

## 整体架构概览

Amplication 使用 **Stigg** 作为计费与配额管理的后端服务。整个配额限制体系由三个核心环节组成：

```
套餐判定(Plan Determination) → 额度消耗(Quota Consumption) → 拦截反馈(Interception & Feedback)
```

三者形成闭环：套餐决定了用户有哪些额度、额度在使用时被消耗、当额度不足时进行拦截并给出反馈。

---

## 一、套餐判定（Plan Determination）

### 1.1 套餐定义

套餐类型在 [billing-plan.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/libs/util/billing-types/src/lib/billing-plan.types.ts) 中定义：

```typescript
export enum BillingPlan {
  Enterprise = "plan-amplication-enterprise",
  Essential = "plan-amplication-essential",
  Free = "plan-amplication-free",
  PreviewBreakTheMonolith = "plan-amplication-preview-break-monolith",
  Pro = "plan-amplication-pro",
  ProWithTrial = "plan-amplication-pro-with-trial",
  Team = "plan-amplication-team",
}
```

数据库侧的枚举在 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L770-L777) 中定义为 `EnumSubscriptionPlan`。

### 1.2 套餐映射关系

在 [billing.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L30-L39) 中，Stigg 的 planId 映射到内部的 `EnumSubscriptionPlan`：

```typescript
const SUBSCRIPTION_PLAN_MAP: Record<BillingPlan, EnumSubscriptionPlan> = {
  [BillingPlan.Enterprise]: EnumSubscriptionPlan.Enterprise,
  [BillingPlan.Essential]: EnumSubscriptionPlan.Essential,
  [BillingPlan.Free]: EnumSubscriptionPlan.Free,
  [BillingPlan.PreviewBreakTheMonolith]: EnumSubscriptionPlan.PreviewBreakTheMonolith,
  [BillingPlan.Pro]: EnumSubscriptionPlan.Pro,
  [BillingPlan.ProWithTrial]: EnumSubscriptionPlan.Pro,  // ProWithTrial 映射到 Pro
  [BillingPlan.Team]: EnumSubscriptionPlan.Team,
};
```

### 1.3 套餐获取流程

套餐判定的核心在 [subscription.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts) 中，采用 **DB 优先 + Stigg 兜底** 的策略：

```typescript
// subscription.service.ts#L344-L372
async resolveSubscription(workspaceId: string): Promise<Subscription> {
  // 1. 先从数据库获取当前激活的订阅
  const databaseSubscription = await this.getCurrentSubscription(workspaceId);
  if (databaseSubscription) {
    return databaseSubscription;
  }

  // 2. DB 没有则从 Stigg 获取
  const stiggSubscription = await this.billingService.getSubscription(workspaceId);
  if (stiggSubscription) {
    // 3. 同步回 DB
    const savedSubscription = await this.prisma.subscription.create({...});
    return savedSubscription;
  }
  return null;
}
```

`getCurrentSubscription` 查询条件（[subscription.service.ts#L37-L79](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts#L37-L79)）：
- 状态必须是 `Active` 或 `Trailing`
- `cancellationEffectiveDate` 为空或大于当前时间

### 1.4 Stigg 客户端初始化

在 [billing.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L68-L112) 的构造函数中：

```typescript
this.billingEnabled = Boolean(
  configService.get<string>(Env.BILLING_ENABLED) === "true"
);

if (this.isBillingEnabled) {
  const stiggApiKey = configService.get(Env.BILLING_API_KEY);
  this.stiggClient = Stigg.initialize({ apiKey: stiggApiKey });
  // ...
}
```

若 `BILLING_ENABLED !== "true"` 或初始化失败，整个 billing 模块将被禁用，所有配额检查都会被跳过。

### 1.5 工作区创建时的套餐预置

在 [workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L136) 中，创建工作区后会调用：

```typescript
await this.billingService.provisionCustomer(workspace.id);
```

该方法在 Stigg 上为工作区创建客户并预置默认套餐（Enterprise + 多个 addon）。

---

## 二、额度消耗（Quota Consumption）

### 2.1 三种 Entitlement 类型

BillingService 提供三种额度查询方法，对应 Stigg 的三种 Entitlement：

| 方法 | 类型 | 用途 | 典型场景 |
|------|------|------|---------|
| `getBooleanEntitlement` | Boolean | 功能开关 | 是否允许使用某 Git Provider |
| `getMeteredEntitlement` | Metered | 可计量额度（有 currentUsage/usageLimit） | 项目数、服务数、团队成员数 |
| `getNumericEntitlement` | Numeric | 固定数值配置 | 每个服务的最大实体数 |

### 2.2 Feature 定义

所有计费特性在 [billing-feature.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/libs/util/billing-types/src/lib/billing-feature.types.ts) 中定义。枚举共 **28 行**，但由于 `AzureDevOps` 和 `AzureDevops`（大小写不同）共享同一个 feature id `"feature-azure-devops"`，实际为 **27 个独立特性**（详见 6.1 节核准说明）：

```typescript
export enum BillingFeature {
  AllowWorkspaceCreation = "feature-allow-workspace-creation",
  AwsCodeCommit = "feature-awscodecommit",
  AzureDevOps = "feature-azure-devops",
  AzureDevops = "feature-azure-devops",  // 与 AzureDevOps 同一值，大小写别名
  Bitbucket = "feature-bitbucket",
  BlockBuild = "feature-block-build",
  BranchPerResource = "feature-branch-per-resource",
  ChangeGitBaseBranch = "feature-change-git-base-branch",
  CodeGenerationBuilds = "feature-code-generation-builds",
  CodeGeneratorDotNet = "feature-code-generator-dotnet",
  CodeGeneratorNodeJsOnly = "feature-code-generator-node-js-only",
  CodeGeneratorVersion = "feature-code-generator-version",
  CodePushToGit = "feature-code-push-to-git",
  CustomActions = "feature-custom-actions",
  EntitiesPerService = "feature-entities-per-service",
  GitLab = "feature-gitlab",
  IgnoreValidationCodeGeneration = "feature-ignore-validation-code-generation",
  ImportDBSchema = "feature-import-db-schema",
  JovuRequests = "feature-jovu-requests",
  Notification = "feature-notifications",
  PrivatePlugins = "feature-private-plugins-module",
  Projects = "feature-projects",
  RedesignArchitecture = "feature-redesign-architecture",
  Services = "feature-services",
  ServicesAboveEntitiesPerServiceLimit = "feature-services-above-entities-per-service-limit",
  SmartGitSync = "feature-smart-git-sync",
  TeamMembers = "feature-team-members",
}
```

### 2.3 额度上报方法

BillingService 提供两种上报方式：

**reportUsage** - 增量上报（[billing.service.ts#L131-L149](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L131-L149)）：
```typescript
async reportUsage(workspaceId: string, feature: BillingFeature, value = 1)
// 使用 UsageUpdateBehavior.Delta
```

**setUsage** - 覆盖上报（[billing.service.ts#L158-L176](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L158-L176)）：
```typescript
async setUsage(workspaceId: string, feature: BillingFeature, value: number)
// 使用 UsageUpdateBehavior.Set
```

**resetUsage** - 批量重置（[billing.service.ts#L407-L432](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L407-L432)）：
在 commit 时调用，重新计算 Projects、Services、ServicesAboveEntitiesPerServiceLimit、TeamMembers 的使用量。

### 2.4 额度消耗全景图

| 操作 | 触发位置 | Feature | 上报方式 |
|------|---------|---------|---------|
| 创建工作区 | [workspace.service.ts#L158-L161](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L158-L161) | `TeamMembers` | +1 |
| 接受邀请加入工作区 | [workspace.service.ts#L386-L389](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L386-L389) | `TeamMembers` | +1 |
| 创建项目 | [project.service.ts#L112-L115](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L112-L115) | `Projects` | +1 |
| 删除项目 | [project.service.ts#L150-L154](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L150-L154) | `Projects` | -1 |
| 删除项目（归档服务） | [project.service.ts#L136-L140](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L136-L140) | `Services` | -N |
| 创建 Service | [resource.service.ts#L621-L624](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L621-L624) | `Services` | +1 |
| 删除 Service | [resource.service.ts#L1490-L1494](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1490-L1494) | `Services` | -1 |
| 提交代码（Commit） | [project.service.ts#L464-L467](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L464-L467) | `CodeGenerationBuilds` | +1 |
| Git Push 成功 | [build.service.ts#L894-L897](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/build/build.service.ts#L894-L897) | `CodePushToGit` | +1 |
| AI 助手请求 | [assistant.service.ts#L159-L162](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/assistant/assistant.service.ts#L159-L162) | `JovuRequests` | +1 |
| Commit 时重置 | [project.service.ts#L393-L394](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L393-L394) | Projects/Services/TeamMembers 等 | setUsage |

### 2.5 licensed 字段与套餐联动

数据库中 `Project` 和 `Resource` 表都有 `licensed` 字段（[schema.prisma#L58](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L58)、[schema.prisma#L244](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L244)），用于标记对象是否在套餐配额内。

在 [subscription.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts) 中：

- `updateProjectLicensed`（[L103-L174](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts#L103-L174)）：按创建时间排序，配额内的项目 `licensed=true`，超出的 `licensed=false`
- `updateServiceLicensed`（[L176-L259](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts#L176-L259)）：同上，针对 Service 资源

这两个方法在以下时机被调用：
- 订阅状态变更（subscription.created / subscription.updated / promotionalEntitlement 事件）
- 删除项目/服务后
- 工作区级批量更新（`bulkUpdateWorkspaceProjectsAndResourcesLicensed` 定时任务）

---

## 三、拦截反馈（Interception & Feedback）

### 3.1 错误类层级

```
AmplicationError
  └── BillingLimitationError   [errors/BillingLimitationError.ts]
        └── (经 GqlResolverExceptionsFilter 转换)
              └── GraphQLBillingError   [errors/graphql/graphql-billing-limitation-error.ts]
```

**BillingLimitationError**（[BillingLimitationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/errors/BillingLimitationError.ts)）：
```typescript
export class BillingLimitationError extends AmplicationError {
  constructor(
    message: string,
    public readonly billingFeature: BillingFeature,
    public readonly bypassAllowed = true  // 是否允许绕过（代码生成场景）
  ) {
    super(`LimitationError: ${message}`);
  }
}
```

### 3.2 异常过滤器

[GqlResolverExceptions.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts#L64-L70) 统一捕获 BillingLimitationError：

```typescript
} else if (exception instanceof BillingLimitationError) {
  clientError = new GraphQLBillingError(
    exception.message,
    exception.billingFeature,
    exception.bypassAllowed
  );
  this.logger.info(clientError.message, { exception });
}
```

**GraphQLBillingError**（[graphql-billing-limitation-error.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/errors/graphql/graphql-billing-limitation-error.ts)）携带扩展字段返回给客户端：
- `billingFeature`：触发限制的特性 ID
- `bypassAllowed`：是否允许用户选择跳过限制继续代码生成

### 3.3 拦截点全景

#### 3.3.1 Boolean 型功能拦截

| 特性 | 拦截位置 | 错误消息 |
|------|---------|---------|
| `AllowWorkspaceCreation` | [workspace.service.ts#L105-L114](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L105-L114) | "Your current plan does not allow creating workspaces" |
| `BlockBuild` | [project.service.ts#L365-L368](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L365-L368) | "Your current plan does not allow code generation." |
| `PrivatePlugins` | [privatePlugin.service.ts#L229-L238](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/privatePlugin/privatePlugin.service.ts#L229-L238) | "Feature Unavailable. Please upgrade your plan to use the Private Plugins Module." |
| `CodeGeneratorVersion` | [resource.service.ts#L375-L383](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L375-L383) | "Feature Unavailable. Please upgrade your plan to access this feature." |
| `CodeGeneratorDotNet` | [resource.service.ts#L437-L446](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L437-L446) | "Feature Unavailable. Please upgrade your plan to use the code generator for DotNet." |
| `CodeGeneratorNodeJsOnly` | [resource.service.ts#L421-L431](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L421-L431) | 限制只能使用 Node.js 代码生成器 |
| `ImportDBSchema` | [entity.service.ts#L468-L480](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L468-L480) | "Feature Unavailable. Your current user permissions doesn't include importing Prisma schemas" |
| `CustomActions` | [block.util.ts#L47-L65](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/block/block.util.ts#L47-L65) | "User has no access to custom actions features" |
| `AwsCodeCommit` | [git.provider.service.ts#L646-L656](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/git/git.provider.service.ts#L646-L656) | "In order to connect AWS CodeCommit service should upgrade its plan" |
| `Bitbucket`/`GitLab`/`AzureDevOps` | [billing.service.ts#L348-L374](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L348-L374) | "Your workspace uses ${provider} integration, while it is not part of your current plan." |
| `ChangeGitBaseBranch` | [billing.service.ts#L376-L392](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L376-L392) | "Your workspace uses the custom git base branch feature..." |
| `BranchPerResource` | [build.service.ts#L1293-L1312](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/build/build.service.ts#L1293-L1312) | 控制是否按资源创建独立分支 |

#### 3.3.2 Metered 型额度拦截

| 特性 | 拦截位置 | 检查条件 |
|------|---------|---------|
| `Projects` | [project.service.ts#L78-L93](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L78-L93) | `!hasAccess \|\| currentUsage >= usageLimit` |
| `Services` | [resource.service.ts#L223-L245](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L223-L245) | `existingProjectResources.length >= usageLimit` |
| `Services` | [billing.service.ts#L328-L336](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L328-L336) | `!hasAccess`（在 commit 时的 workspace 级校验） |
| `TeamMembers` | [workspace.service.ts#L217-L230](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L217-L230) | `currentUsage < usageLimit`（邀请成员前检查） |
| `TeamMembers` | [billing.service.ts#L338-L346](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L338-L346) | `!hasAccess`（在 commit 时的 workspace 级校验） |
| `JovuRequests` | [assistant.service.ts#L146-L157](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/assistant/assistant.service.ts#L146-L157) | `!hasAccess`（调用 AI 助手前检查） |

#### 3.3.3 Numeric 型配置拦截

| 特性 | 拦截位置 | 检查条件 |
|------|---------|---------|
| `EntitiesPerService` | [entity.service.ts#L281-L313](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L281-L313) | `value <= currentEntityCount`（创建实体前） |
| `EntitiesPerService` | [project.service.ts#L311-L344](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L311-L344) | `value < maxResourceEntitiesCount`（commit 时校验） |
| `EntitiesPerService` | [resource.service.ts#L1204-L1262](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1204-L1262) | 架构重设计时移动实体的数量校验 |

### 3.4 Commit 时的综合校验

提交代码是最关键的拦截点，在 [project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts#L391-L423) 中按顺序执行：

```
commit()
  │
  ├── 1. shouldBlockBuild()          → 检查 BlockBuild（Boolean）
  │
  ├── 2. calculateMeteredUsage()     → 从 DB 重新统计 Projects/Services/TeamMembers
  │
  ├── 3. billingService.resetUsage() → 将统计值覆盖上报给 Stigg
  │
  ├── 4. validateSubscriptionPlanLimitationsForWorkspace()
  │     ├── Services (Metered)
  │     ├── TeamMembers (Metered)
  │     ├── 企业级 Git Providers (Boolean: Bitbucket/GitLab/Azure/AWS)
  │     └── ChangeGitBaseBranch (Boolean)
  │
  └── 5. validateProjectLimitations()
        └── EntitiesPerService (Numeric)
```

`validateSubscriptionPlanLimitationsForWorkspace` 还支持 `bypassLimitations` 参数，当用户有 `IgnoreValidationCodeGeneration` entitlement 时可跳过校验（[billing.service.ts#L311-L405](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L311-L405)）。

### 3.5 分析埋点

当发生配额限制时，会发送分析事件 `SubscriptionLimitPassed`（[billing.service.ts#L394-L400](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L394-L400)）：

```typescript
if (error instanceof BillingLimitationError) {
  await this.analytics.trackWithContext({
    event: EnumEventType.SubscriptionLimitPassed,
    properties: {
      reason: error.message,
    },
  });
}
```

其他升级相关事件：
- `WorkspacePlanUpgradeRequest` / `WorkspacePlanDowngradeRequest`：[billing.service.ts#L257-L262](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts#L257-L262)
- `WorkspacePlanUpgradeCompleted`：[subscription.service.ts#L81-L101](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts#L81-L101)

---

## 四、三者关系总览

```
                        ┌───────────────────────────────────────────┐
                        │           Stigg (外部计费服务)              │
                        │  - 存储 Plan/Feature/Entitlement 定义      │
                        │  - 记录 Usage                              │
                        └──────────────┬────────────────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
    ┌─────────▼──────────┐  ┌──────────▼──────────┐  ┌──────────▼──────────┐
    │   套餐判定          │  │    额度消耗         │  │    拦截反馈         │
    │                    │  │                     │  │                     │
    │ SubscriptionService│  │  BillingService     │  │ BillingLimitationEr │
    │  .resolveSubscription │  │  .reportUsage()   │  │                     │
    │  .getCurrentSubscription │  │  .setUsage()    │  │ GqlResolverException│
    │                    │  │  .resetUsage()      │  │  sFilter             │
    │ BillingService     │  │                     │  │                     │
    │  .getSubscription()│  │ 触发点:             │  │ GraphQLBillingError │
    │  .provisionCustomer│  │  - 创建资源         │  │                     │
    │  .mapSubscription  │  │  - 删除资源         │  │ 分析埋点:            │
    │   Plan/Status      │  │  - Commit           │  │  SubscriptionLimitPa │
    │                    │  │  - Git Push         │  │  ssed                │
    │ licensed 字段联动  │  │  - AI 助手调用      │  │                     │
    │  updateProject/    │  │                     │  │                     │
    │  ServiceLicensed   │  │                     │  │                     │
    └────────────────────┘  └─────────────────────┘  └─────────────────────┘
              │                        │                        │
              └────────────────────────┼────────────────────────┘
                                       │
                                       ▼
                        ┌───────────────────────────────────────────┐
                        │              业务代码层                    │
                        │  workspace / project / resource / entity  │
                        │  build / assistant / git / privatePlugin  │
                        └───────────────────────────────────────────┘
```

**核心数据流**：

1. **工作区创建** → `provisionCustomer()` 在 Stigg 预置套餐 → 套餐生效
2. **用户操作**（创建项目/服务/实体等）→ 先 `getXxxEntitlement()` 检查额度 → 通过后 `reportUsage()` 上报消耗
3. **Commit 操作** → 触发全量校验（Workspace 级 + Project 级）→ `resetUsage()` 同步真实使用量
4. **额度不足** → 抛出 `BillingLimitationError` → Filter 转换为 `GraphQLBillingError` → 前端显示升级提示
5. **订阅变更**（Webhook 事件）→ `handleUpdateSubscriptionStatusEvent()` → 更新 DB + 重新计算 licensed 字段

---

## 五、关键代码文件索引

| 层级 | 文件 | 职责 |
|------|------|------|
| 类型定义 | [billing-plan.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/libs/util/billing-types/src/lib/billing-plan.types.ts) | 套餐枚举 |
| 类型定义 | [billing-feature.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/libs/util/billing-types/src/lib/billing-feature.types.ts) | 特性枚举 |
| 类型定义 | [billing-addons.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/libs/util/billing-types/src/lib/billing-addons.types.ts) | 附加项枚举 |
| 核心服务 | [billing.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/billing/billing.service.ts) | Stigg 客户端封装、额度查询/上报、综合校验 |
| 核心服务 | [subscription.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/subscription/subscription.service.ts) | 订阅状态管理、licensed 字段更新 |
| 错误处理 | [BillingLimitationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/errors/BillingLimitationError.ts) | 配额限制异常 |
| 错误处理 | [graphql-billing-limitation-error.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/errors/graphql/graphql-billing-limitation-error.ts) | GraphQL 错误包装 |
| 错误处理 | [GqlResolverExceptions.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts) | 全局异常过滤器 |
| 业务:工作区 | [workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts) | 工作区创建、成员邀请的配额控制 |
| 业务:项目 | [project.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/project/project.service.ts) | 项目增删、Commit 综合校验 |
| 业务:资源 | [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | Service 创建删除、代码生成器、架构重设计 |
| 业务:实体 | [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/entity/entity.service.ts) | 实体数量、DB Schema 导入 |
| 业务:构建 | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/build/build.service.ts) | Git Push、BranchPerResource |
| 业务:AI | [assistant.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/assistant/assistant.service.ts) | Jovu 请求配额 |
| 业务:私有插件 | [privatePlugin.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/privatePlugin/privatePlugin.service.ts) | 私有插件模块许可 |
| 业务:Git | [git.provider.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/git/git.provider.service.ts) | 企业级 Git Provider 许可 |
| 业务:自定义动作 | [block.util.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/block/block.util.ts) | Custom Actions 许可 |
| 数据模型 | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-prisma-db/prisma/schema.prisma) | Subscription/Project.licensed/Resource.licensed |

---

## 六、核准事实澄清

### 6.1 计费特性（BillingFeature）准确数量

在 [billing-feature.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/libs/util/billing-types/src/lib/billing-feature.types.ts#L1-L29) 中枚举定义了 **28 行**，但实际只有 **27 个独立的 feature id**。原因是第 4 行 `AzureDevOps` 和第 5 行 `AzureDevops`（大小写不同）指向同一个字符串值 `"feature-azure-devops"`：

```typescript
AzureDevOps = "feature-azure-devops",   // L4
AzureDevops = "feature-azure-devops",   // L5  —— 与上一行值重复
```

这是 TypeScript 枚举的合法写法，会编译为两个不同的属性名映射到同一值。完整的 27 个独立特性：

| # | 特性名 | Feature ID |
|---|--------|-----------|
| 1 | AllowWorkspaceCreation | feature-allow-workspace-creation |
| 2 | AwsCodeCommit | feature-awscodecommit |
| 3 | AzureDevOps / AzureDevOps | feature-azure-devops |
| 4 | Bitbucket | feature-bitbucket |
| 5 | BlockBuild | feature-block-build |
| 6 | BranchPerResource | feature-branch-per-resource |
| 7 | ChangeGitBaseBranch | feature-change-git-base-branch |
| 8 | CodeGenerationBuilds | feature-code-generation-builds |
| 9 | CodeGeneratorDotNet | feature-code-generator-dotnet |
| 10 | CodeGeneratorNodeJsOnly | feature-code-generator-node-js-only |
| 11 | CodeGeneratorVersion | feature-code-generator-version |
| 12 | CodePushToGit | feature-code-push-to-git |
| 13 | CustomActions | feature-custom-actions |
| 14 | EntitiesPerService | feature-entities-per-service |
| 15 | GitLab | feature-gitlab |
| 16 | IgnoreValidationCodeGeneration | feature-ignore-validation-code-generation |
| 17 | ImportDBSchema | feature-import-db-schema |
| 18 | JovuRequests | feature-jovu-requests |
| 19 | Notification | feature-notifications |
| 20 | PrivatePlugins | feature-private-plugins-module |
| 21 | Projects | feature-projects |
| 22 | RedesignArchitecture | feature-redesign-architecture |
| 23 | Services | feature-services |
| 24 | ServicesAboveEntitiesPerServiceLimit | feature-services-above-entities-per-service-limit |
| 25 | SmartGitSync | feature-smart-git-sync |
| 26 | TeamMembers | feature-team-members |

### 6.2 服务许可（licensed）校验逻辑

`licensed` 字段不仅用于后台记录，还贯穿**服务端拦截**和**前端 UI 禁用**两层校验。

#### 6.2.1 服务端：Entity 创建前的双重许可校验

在 [entity.service.ts#L267-L313](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L267-L313) 中，`checkServiceEntityLicense()` 串联了两层检查：

```typescript
async checkServiceEntityLicense(resource: Resource) {
  // 第一层：检查 project.licensed 和 resource.licensed
  await this.checkServiceLicense(resource);
  // 第二层：检查 EntitiesPerService (Numeric)
  const serviceEntityEntitlement = await this.billingService.getNumericEntitlement(...);
  if (!serviceEntityEntitlement.hasAccess
      || serviceEntityEntitlement.value <= currentEntityCount) {
    throw new BillingLimitationError(...);
  }
}
```

其中 `checkServiceLicense()` 的判定条件（[L267-L279](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L267-L279)）：

```typescript
if (
  !resource.project?.licensed ||          // 项目超出配额
  (!resource.licensed                      // 服务超出配额（仅对 Service 类型资源校验）
    && resource.resourceType === EnumResourceType.Service)
) {
  throw new BillingLimitationError(
    "Your workspace reached its service limitation.",
    BillingFeature.Services
  );
}
```

#### 6.2.2 服务端：架构重设计移动实体时的许可校验

在 [resource.service.ts#L1246-L1262](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1246-L1262) 中，将实体从一个服务移动到另一个服务之前，会同时检查**来源项目**和**目标服务**的 `licensed` 状态：

```typescript
if (!project.licensed || (currentResource && !currentResource.licensed)) {
  throw new AmplicationError(
    `Cannot move entities to service: ${serviceName} due to your plan's limitations (number of services)`
  );
}
```

注：这里抛出的是通用的 `AmplicationError` 而非 `BillingLimitationError`，因此前端不会走计费限制弹窗路径，只会显示普通错误提示。

#### 6.2.3 前端：LicenseIndicatorContainer UI 层禁用

[LicenseIndicatorContainer.tsx](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Components/LicenseIndicatorContainer.tsx) 是通用的"许可证感知"容器组件，通过 `licensedResourceType` 参数决定读取哪个层级的 licensed 字段：

```typescript
const currentProjectLicensed = currentProject?.licensed ?? true;
const currentServiceLicensed = currentResource?.licensed ?? true;

useEffect(() => {
  if (licensedResourceType === LicensedResourceType.Project && !currentProjectLicensed) {
    setDisabled(true);
    setTooltipText(PROJECT_LICENSE_TOOLTIP_TEXT);
    return;
  }
  if (licensedResourceType === LicensedResourceType.Service && !currentServiceLicensed) {
    setDisabled(true);
    setTooltipText(SERVICE_LICENSE_TOOLTIP_TEXT);
    return;
  }
  // 同时也可传入 blockByFeatureId（如 BillingFeature.BlockBuild）做额外 Boolean 校验
  setDisabled(isBlockedFeature);
}, [...]);
```

典型使用场景在 [CommitButton.tsx#L159-L164](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L159-L164)：

```tsx
<LicenseIndicatorContainer
  blockByFeatureId={BillingFeature.BlockBuild}
  licensedResourceType={LicensedResourceType.Project}
>
  {element /* Generate the code 按钮 */}
</LicenseIndicatorContainer>
```

这意味着"Generate the code"按钮会在以下**任一**条件满足时被禁用并显示 Tooltip：
1. 当前 Project 的 `licensed === false`
2. Stigg 返回 `BlockBuild` entitlement 的 `hasAccess === false`

### 6.3 架构重设计（RedesignArchitecture）权限

`RedesignArchitecture` 是一个 **Boolean Entitlement**，用于控制"Break the Monolith / Architecture Redesign"功能，对应 Stigg 中的 `feature-redesign-architecture`。权限校验在前后端均有实现。

#### 6.3.1 后端：双重权限检查

在 [resourceBtm.service.ts#L75-L100](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-server/src/core/resource/resourceBtm.service.ts#L75-L100) 的 `checkAccessToBreakTheMonolithWithGpt()` 中，做了**两层独立校验**：

```typescript
private async checkAccessToBreakTheMonolithWithGpt(user: User) {
  if (this.billingService.isBillingEnabled) {
    // 第一层：BillingFeature.RedesignArchitecture 权限
    const btmWithGpt = (
      await this.billingService.getBooleanEntitlement(
        user.workspace?.id,
        BillingFeature.RedesignArchitecture
      )
    ).hasAccess;

    if (!btmWithGpt) {
      throw new BillingLimitationError(
        "Available as part of the Enterprise plan only.",
        BillingFeature.RedesignArchitecture
      );
    // 第二层：工作区的 allowLLMFeatures 开关
    } else if (!userWorkspace.allowLLMFeatures) {
      throw new AmplicationError(
        "your workspace settings forbid LLM features use."
      );
    }
  }
}
```

调用链：
- `triggerBreakServiceIntoMicroservices()` → 先调用 `checkAccessToBreakTheMonolithWithGpt()` → 再发起 GPT 请求
- `finalizeBreakServiceIntoMicroservices()` → 直接处理 GPT 返回结果（不再校验，因为已在 trigger 时校验过）
- `startRedesign()` → 只做埋点，**不做权限校验**（纯前端状态切换）

#### 6.3.2 前端：三处独立控制入口

前端共三处按钮/页面都绑定了 RedesignArchitecture 权限：

| 位置 | 组件 | 控制方式 |
|------|------|---------|
| 架构控制台工具栏 | [RedesignResourceButton.tsx#L36-L88](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Components/RedesignResourceButton.tsx#L36-L88) | `FeatureIndicatorContainer` + `EntitlementType.Boolean`，无权限时按钮禁用 + 锁图标 |
| 资源页 / 项目页 / 实体列表页 | [BtmButton.tsx#L99-L101](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Resource/break-the-monolith/BtmButton.tsx#L99-L101) | 直接调用 `stigg.getBooleanEntitlement()`，同时检查 `allowLLMFeature`，分 4 种 Tooltip 文案（试用/无权限/LLM关闭/完全访问） |
| 架构控制台根页面 | [ArchitectureConsole.tsx#L51-L52](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Project/ArchitectureConsole/ArchitectureConsole.tsx#L51-L52) | `FeatureIndicatorContainer` 包裹整个页面内容 |

BtmButton 的四种 Tooltip 状态逻辑（[BtmButton.tsx#L105-L145](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Resource/break-the-monolith/BtmButton.tsx#L105-L145)）：

```
hasAccess + (Enterprise/Team + Trailing)  →  试用提示 + Upgrade 链接
!hasAccess                                 →  "Available as part of the Enterprise plan only."
hasAccess + !allowLLMFeature               →  "LLM features are forbidden in workspace settings"
hasAccess + (Enterprise/Team + Active)     →  完整功能描述，无升级链接
```

### 6.4 前端配额错误展示路径

前端配额错误展示分为**两条独立路径**："异步错误弹窗路径"和"同步控件禁用路径"。

#### 6.4.1 路径一：Commit 异步错误 → LimitationDialog 弹窗

这是用户最常接触的路径，用于处理 GraphQL mutation 返回的计费限制错误。完整调用链：

```
用户点击 Commit 按钮
  │
  ▼
CommitButton.tsx handleClick()
  │  调用 commitChanges(data)
  │
  ▼
useCommits.ts [useMutation COMMIT_CHANGES]
  │
  ├── onError 回调 (L164-L174)
  │     检查 graphQLErrors 中是否存在
  │     extensions.code === GraphQLErrorCode.BILLING_LIMITATION_ERROR
  │     → 若是，setOpenLimitationDialog(true)
  │
  └── commitChangesLimitationError (useMemo L219-L233)
        从 graphQLErrors 中解析：
          - message: 去掉前缀 "LimitationError: " 后的纯文本
          - billingFeature: extensions.billingFeature
        ↓
  ▼
CommitButton.tsx render (L165-L199)
  │
  └── isLimitationError ? <LimitationDialog /> : <Snackbar />
        │
        ├── message={commitChangesLimitationError.message}
        ├── allowBypassLimitation={bypassLimitations}
        ├── onConfirm → 跳转到 /purchase（升级）
        ├── onDismiss → 关闭 + bypassLimitationsRef.current = false
        └── onBypass  → bypassLimitationsRef.current = true
                         （下次 commit 时会带上 bypassLimitations=true）
```

关键实现细节：

1. **错误码定义**：在 [graphql-error-codes.ts](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/libs/util/graphql-error-codes/src/lib/graphql-error-codes.ts#L1-L5) 中：
   ```typescript
   export enum GraphQLErrorCode {
     BILLING_LIMITATION_ERROR = "BILLING_LIMITATION_ERROR",
     // ...
   }
   ```

2. **bypassLimitations 判定**（[useCommits.ts#L212-L217](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/VersionControl/hooks/useCommits.ts#L212-L217)）：
   ```typescript
   const bypassLimitations = useMemo(() => {
     return (
       currentWorkspace?.subscription?.subscriptionPlan
         !== EnumSubscriptionPlan.Pro
     );
   }, [currentWorkspace]);
   ```
   即：**非 Pro 套餐都允许跳过限制**继续生成代码（Pro 套餐必须先升级）。

3. **bypass 回传后端**：用户点击 LimitationDialog 的"Upgrade Later"后，`bypassLimitationsRef.current = true`，下次 commit 时（[CommitButton.tsx#L100](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L100)）：
   ```tsx
   commitChanges({
     bypassLimitations: bypassLimitationsRef.current ?? false,
     // ...
   });
   ```
   后端 `validateSubscriptionPlanLimitationsForWorkspace` 收到 `bypassLimitations=true` 且用户拥有 `IgnoreValidationCodeGeneration` entitlement 时跳过校验。

#### 6.4.2 路径二：同步控件禁用 → FeatureIndicator Tooltip

适用于所有"提前预判用户无权操作"的场景，不让用户点击后再报错。由两个容器组件承载：

**FeatureIndicatorContainer**（[FeatureIndicatorContainer.tsx](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Components/FeatureIndicatorContainer.tsx)）：通用特性感知容器，支持 Boolean 和 Metered 两种 Entitlement 类型。

```
children / render()
  │
  ▼
FeatureIndicatorContainer
  │
  ├── stigg.getBooleanEntitlement(featureId).hasAccess  (Boolean 类型)
  └── stigg.getMeteredEntitlement(featureId)            (Metered 类型)
        ├── usageLimit / currentUsage / hasAccess
        └── usageExceeded = usageLimit && currentUsage >= usageLimit
  │
  ├── disabled 状态推导
  │     Boolean: disabled = !hasBooleanAccess
  │     Metered: disabled = usageExceeded ?? !hasMeteredAccess
  │
  ├── icon 状态推导
  │     disabled → IconType.Lock（锁图标）
  │     Enterprise/Team/Team + Trailing → IconType.Diamond（钻石图标表示试用）
  │     其他 → null
  │
  └── Tooltip 文案（FeatureIndicator）
        ├── 已付费完全访问: fullEnterpriseText（无升级链接）
        ├── 试用中: DEFAULT_TEXT_START + "Upgrade Now" 链接
        └── 无权限: limitationText + DISABLED_DEFAULT_TEXT_END + 升级链接
```

典型使用方：
- [RedesignResourceButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Components/RedesignResourceButton.tsx)（Boolean: RedesignArchitecture）
- [AddNewProject.tsx](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Project/AddNewProject.tsx)（Metered: Projects）
- [Teams/NewTeam.tsx](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Teams/NewTeam.tsx)（Metered: TeamMembers）
- [Entity/EntityList.tsx](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Entity/EntityList.tsx)（Boolean: ImportDBSchema）

**FeatureIndicator**（[FeatureIndicator.tsx](file:///d:/fz/0601/solo-dogfeeding/code/102-amplication/packages/amplication-client/src/Components/FeatureIndicator.tsx)）：底层 Tooltip 包装组件，不直接接触 Stigg，仅负责展示：
- 基于 MUI `Tooltip`，样式为蓝底白字带青蓝色边框
- 支持两种 CTA：`Upgrade Now`（跳转到 `/${workspaceId}/purchase`）和 `Talk to Us`（外链）
- 点击链接会埋点 `UpgradeClick`，携带 `billingFeature` 和 `eventOriginLocation`

#### 6.4.3 两条路径的对比

| 维度 | 路径一：LimitationDialog | 路径二：FeatureIndicator Tooltip |
|------|------------------------|-------------------------------|
| 触发时机 | GraphQL mutation 响应后（异步） | 渲染时（同步） |
| 交互形式 | 模态对话框（Confirm/Dismiss/Bypass） | Hover 显示 Tooltip + 按钮置灰 |
| 是否支持 bypass | 是（非 Pro 套餐可跳过限制） | 否（纯展示） |
| 错误来源 | 后端 BillingLimitationError | 前端 Stigg SDK entitlement |
| 典型场景 | Commit 代码生成 | 所有新建按钮（项目/服务/团队/实体等） |
| 埋点事件 | UpgradeClick / UpgradeLaterClick / PassedLimitsNotificationClose | UpgradeClick |

