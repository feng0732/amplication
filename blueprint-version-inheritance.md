# Blueprint 版本与继承取值关系详解

本文档详细说明 Amplication 中 Blueprint（蓝图）、模板版本、资源局部覆盖和升级提示之间的取值关系与数据流。

---

## 一、核心实体分层模型

系统通过 **6 层结构** 来管理版本与继承关系：

| 层级 | 实体名称 | 存储方式 | 关键字段 | 说明 |
|------|---------|---------|---------|------|
| 1 | Blueprint（蓝图） | 独立表 | `resourceType`, `codeGeneratorName` | 资源类型的顶层定义，规定资源可用的代码生成器 |
| 2 | Resource（资源） | 独立表 | `blueprintId`, `codeGeneratorVersion`, `codeGeneratorStrategy`, `codeGeneratorName` | 实际的项目资源（Service/Component/ServiceTemplate 等），自身持有版本字段 |
| 3 | TemplateCodeEngineVersion（模板引擎版本） | Block（`CodeEngineVersion` 类型） | `codeGeneratorVersion`, `codeGeneratorStrategy` | ServiceTemplate 的代码引擎版本历史记录 |
| 4 | ResourceVersion（模板版本快照） | 独立表 | `version`, `blockVersions[]`, `entityVersions[]` | ServiceTemplate 的发布版本（类似 Git Tag） |
| 5 | ResourceTemplateVersion（资源-模板关联） | Block（`ResourceTemplateVersion` 类型） | `serviceTemplateId`, `version` | 记录某个资源基于哪个模板的哪个版本创建 |
| 6 | OutdatedVersionAlert（过期告警） | 独立表 | `type`, `outdatedVersion`, `latestVersion`, `status` | 版本落后时的升级提示 |

> **注意**：Block 是一种通用存储结构，通过 `blockType` 区分不同用途，`settings` 字段存储具体 JSON 数据。

---

## 二、代码生成器版本的取值优先级

代码生成器版本决定了构建（Build）时使用哪个版本的 DSG（Data Service Generator）。

### 2.1 版本策略枚举

在 [EnumCodeGeneratorVersionStrategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/dto/EnumCodeGeneratorVersionStrategy.ts#L3-L7) 中定义了三种策略：

```typescript
export enum CodeGeneratorVersionStrategy {
  LatestMajor = "LatestMajor",   // 始终使用最新的主版本（默认）
  LatestMinor = "LatestMinor",   // 使用指定主版本下的最新次版本
  Specific    = "Specific",      // 锁定到某个具体版本号
}
```

### 2.2 版本解析流程

在 [version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/data-service-generator-catalog/src/version/version.service.ts#L96-L164) 的 `getCodeGeneratorVersion()` 方法中实现解析：

| 策略 | 所需参数 | 解析逻辑 |
|------|---------|---------|
| `Specific` | `codeGeneratorVersion`（如 "v2.1.0"） | 精确匹配该版本号 |
| `LatestMinor` | `codeGeneratorVersion`（如 "v2.0.0"） | 提取主版本号（2），在所有 active 版本中找该主版本的最新次版本 |
| `LatestMajor` | 无需 `codeGeneratorVersion` | 在所有 active 版本中找最新版本 |

### 2.3 局部覆盖优先级（从高到低）

构建时在 [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/build/build.service.ts#L1509-L1514) 中读取：

```
Resource.codeGeneratorVersion + Resource.codeGeneratorStrategy
           ↓（若资源自身未设置，则使用默认）
     LatestMajor 策略 + 空版本号
```

**关键点**：
- 资源自身的 `codeGeneratorVersion` 和 `codeGeneratorStrategy` 是**最终生效值**。
- Blueprint 只在**创建资源时**校验代码生成器类型是否合法（见 [blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts#L164-L219) 的 `updateBlueprintEngine()`），不参与运行时版本解析。
- 从模板创建资源时，模板的代码生成器名称会拷贝到新资源（见下文）。

### 2.4 更新资源版本

在 [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L361-L413) 的 `updateCodeGeneratorVersion()` 中：

```typescript
// 直接写入 Resource 表的两个字段
codeGeneratorVersion: args.data.codeGeneratorVersionOptions.codeGeneratorVersion,
codeGeneratorStrategy: args.data.codeGeneratorVersionOptions.codeGeneratorStrategy,

// 如果资源是 ServiceTemplate，同步记录到 TemplateCodeEngineVersion Block
if (resource.resourceType === EnumResourceType.ServiceTemplate) {
  await this.templateCodeEngineVersionService.update(...);
}
```

---

## 三、从模板创建资源时的版本继承

### 3.1 创建流程

在 [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L242-L371) 的 `createResourceFromTemplate()` 中：

```
步骤1：校验模板存在 → 获取模板最新版本 ResourceVersion
步骤2：调用 internalCreateServiceFromTemplate() / internalCreateComponentFromTemplate()
步骤3：设置 ResourceTemplateVersion Block（记录关联关系）
步骤4：复制插件安装
```

### 3.2 代码生成器名称的继承

在 `internalCreateServiceFromTemplate()` 中（[serviceTemplate.service.ts#L373-L426](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L373-L426)）：

```typescript
codeGenerator: template.codeGeneratorName
  ? CODE_GENERATOR_NAME_TO_ENUM[template.codeGeneratorName]
  : EnumCodeGenerator.NodeJs,
```

模板的 `codeGeneratorName` 被转换为枚举后传入 `createService()`，最终写入新资源的 `codeGeneratorName` 字段。

> **注意**：此时**不继承**模板的 `codeGeneratorVersion` 和 `codeGeneratorStrategy`，新资源使用默认值（由 `createService` 内部逻辑决定，通常为空 / LatestMajor）。

### 3.3 模板关联记录

创建完成后，写入 [ResourceTemplateVersion Block](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts#L51-L88)：

```typescript
{
  serviceTemplateId: template.id,  // 关联的模板 ID
  version: templateVersion.version // 使用的模板版本号（如 "1.0.0"）
}
```

该 Block 是后续升级提示和模板升级的依据。

---

## 四、模板升级（Upgrade）的取值逻辑

### 4.1 触发入口

在 [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L498-L599) 的 `upgradeServiceToLatestTemplateVersion()` 中。

### 4.2 升级流程

```
步骤1：读取资源当前的 ResourceTemplateVersion Block（serviceTemplateId + version）
步骤2：获取模板的最新 ResourceVersion
步骤3：如果版本相同，抛出"已是最新"错误
步骤4：比较两个版本的差异（createdBlocks / updatedBlocks / deletedBlocks）
步骤5：逐块合并变更
步骤6：更新 ResourceTemplateVersion Block 的 version 字段
步骤7：将相关的 OutdatedVersionAlert 标记为 Resolved
```

### 4.3 三类 Block 的合并处理

| Block 类型 | 处理函数 | 逻辑 |
|-----------|---------|------|
| `PluginInstallation` | `handleMergeCreatedBlock` / `handleMergeUpdatedBlock` | 调用 `pluginInstallationService.mergeVersionIntoLatest()` 合并插件配置 |
| `CodeEngineVersion` | 同上两个函数 | 调用 `resourceService.updateCodeGeneratorVersion()` **覆盖**资源的 codeGeneratorVersion 和 codeGeneratorStrategy |
| 其他（deletedBlocks） | `handleMergeDeletedBlock` | 目前为空实现（no-op） |

**关键点**：模板升级时，`CodeEngineVersion` Block 的变更会**直接覆盖**资源自身的版本设置——这就是模板版本覆盖局部设置的入口。

---

## 五、升级提示（OutdatedVersionAlert）的触发机制

### 5.1 告警类型

在 [EnumOutdatedVersionAlertType.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/outdatedVersionAlert/dto/EnumOutdatedVersionAlertType.ts#L3-L7) 中定义：

```typescript
TemplateVersion    = "TemplateVersion",    // 模板版本落后
PluginVersion      = "PluginVersion",      // 插件版本落后
CodeEngineVersion  = "CodeEngineVersion",  // 代码引擎版本落后
```

### 5.2 模板版本告警的触发

在 [resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts#L40-L112) 的 `create()` 方法中：

当创建的 ResourceVersion 属于 **ServiceTemplate** 类型时，自动调用：

```typescript
await this.outdatedVersionAlertService.triggerAlertsForTemplateVersion(
  resourceId,           // 模板的资源 ID
  previousVersion?.version,  // 模板的上一个版本号
  args.data.version     // 刚发布的新版本号
);
```

### 5.3 告警生成逻辑

在 [outdatedVersionAlert.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L179-L246) 的 `triggerAlertsForTemplateVersion()` 中：

```
步骤1：校验资源是 ServiceTemplate 类型
步骤2：通过 serviceTemplateId 查询所有使用该模板的资源
       （ResourceTemplateVersion Block 中 serviceTemplateId 匹配）
步骤3：对每个资源：
         a. 读取其 ResourceTemplateVersion Block 中的当前 version
         b. 先将该资源同类型（TemplateVersion）的所有 New 状态告警 → Canceled
         c. 创建新告警：
              {
                resourceId: 服务资源ID,
                type: TemplateVersion,
                outdatedVersion: 资源当前使用的模板版本号,
                latestVersion: 模板刚发布的新版本号,
                status: New
              }
         d. 通过 Kafka 发送 TECH_DEBT_CREATED_TOPIC 消息通知工作区用户
```

### 5.4 告警状态流转

在 [EnumOutdatedVersionAlertStatus.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/outdatedVersionAlert/dto/EnumOutdatedVersionAlertStatus.ts#L3-L8) 中定义：

| 状态 | 含义 | 触发场景 |
|------|------|---------|
| `New` | 新建待处理 | 模板发布新版本时自动创建 |
| `Resolved` | 已解决 | 用户执行模板升级成功后自动标记（`resolvesServiceTemplateUpdated()`） |
| `Ignored` | 已忽略 | 用户手动忽略 |
| `Canceled` | 已取消 | 模板再次发布新版本时，旧的 New 告警自动被取消 |

---

## 六、完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Blueprint (蓝图)                              │
│  - resourceType: Service / Component / MessageBroker                │
│  - codeGeneratorName: @amplication/data-service-generator / ...     │
│            │ （创建时校验类型合法性，不参与运行时解析）               │
│            ▼                                                         │
┌─────────────────────────────────────────────────────────────────────┐
│                ServiceTemplate (模板资源)                            │
│  ┌──────────────────────────┐  ┌────────────────────────────────┐   │
│  │ Resource (表字段)        │  │ TemplateCodeEngineVersion      │   │
│  │  - codeGeneratorName     │  │  (Block: CodeEngineVersion)    │   │
│  │  - codeGeneratorVersion  │  │  - codeGeneratorVersion        │   │
│  │  - codeGeneratorStrategy │  │  - codeGeneratorStrategy       │   │
│  └────────────┬─────────────┘  └────────────────────────────────┘   │
│               │ 发布新版本时快照                                      │
│               ▼                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ ResourceVersion (v1.0.0 → v1.1.0)                            │   │
│  │  - blockVersions[] (包含 TemplateCodeEngineVersion 快照)     │   │
│  │  - entityVersions[]                                          │   │
│  └────────────┬─────────────────────────────────────────────────┘   │
└───────────────┼─────────────────────────────────────────────────────┘
                │
       模板发布新版本时触发
                │
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│              OutdatedVersionAlert (升级提示)                         │
│  对每个使用该模板的资源创建：                                         │
│  {                                                                   │
│    type: TemplateVersion,                                            │
│    outdatedVersion: "1.0.0",  ← 资源当前使用的版本                   │
│    latestVersion: "1.1.0",    ← 模板最新版本                        │
│    status: New                                                       │
│  }                                                                   │
└─────────────────────────────────────────────────────────────────────┘
                │
       用户点击"升级模板"时触发
                │
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 Resource (实际服务/组件)                              │
│  ┌──────────────────────────────────┐  ┌─────────────────────────┐   │
│  │ Resource (表字段 - 局部覆盖)     │  │ ResourceTemplateVersion │   │
│  │  ✅ codeGeneratorVersion   ←─────┼──┤  (Block) - 被升级覆盖    │   │
│  │  ✅ codeGeneratorStrategy ←──┐   │  │  - serviceTemplateId    │   │
│  │     codeGeneratorName        │   │  │  - version: "1.1.0" ✓   │   │
│  └──────────────────────────────┼───┘  └─────────────────────────┘   │
│                                  │                                   │
│                           构建时读取                                   │
│                                  ▼                                   │
│                    VersionService.getCodeGeneratorVersion()          │
│                    根据 strategy + version 解析出实际使用的版本       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 七、关键问题解答

### Q1: 资源自己修改了代码生成器版本，模板升级会覆盖吗？
**会**。模板升级时 `CodeEngineVersion` Block 的合并逻辑会调用 `updateCodeGeneratorVersion()` 直接覆盖资源的两个字段。（见 [serviceTemplate.service.ts#L616-L634](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L616-L634) 和 [L663-L684](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L663-L684)）

### Q2: Blueprint 的 codeGeneratorName 和 Resource 的 codeGeneratorName 是什么关系？
Blueprint 的 `codeGeneratorName` 仅作为**校验约束**存在——更新 Blueprint 引擎时会校验资源类型和代码生成器的组合是否合法。资源创建后，其自身的 `codeGeneratorName` 字段就是实际生效值，不再依赖 Blueprint。

### Q3: TemplateCodeEngineVersion 和 Resource.codeGeneratorVersion 的区别？
| 维度 | TemplateCodeEngineVersion (Block) | Resource.codeGeneratorVersion (表字段) |
|------|----------------------------------|---------------------------------------|
| 所属对象 | ServiceTemplate | 所有 Resource |
| 用途 | 模板的版本历史记录（会被快照进 ResourceVersion） | 构建时实际使用的版本 |
| 更新时机 | ServiceTemplate 更新代码生成器版本时自动同步 | 用户手动设置，或模板升级时被覆盖 |

### Q4: 为什么 deletedBlocks 的合并是空实现？
代码中明确标注了 `@todo - allow the user to decide if they want to delete the plugin or keep it`，说明模板升级时对删除操作暂时采取保守策略（不自动删除用户资源中的插件）。

### Q5: 同一资源多次发布模板新版本，旧告警如何处理？
每次创建新的 TemplateVersion 告警前，会先把该资源同类型所有 `New` 状态的告警批量更新为 `Canceled`，确保用户只看到最新的升级提示。

---

## 九、CodeEngineVersion 合并的异步执行顺序深度分析

### 9.1 代码位置

模板升级的核心逻辑在 [serviceTemplate.service.ts#L498-L599](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L498-L599) 的 `upgradeServiceToLatestTemplateVersion()` 方法中。

### 9.2 期望的执行流程（设计意图）

```typescript
// 步骤 A: 收集所有异步操作
const createdPromises = changes.createdBlocks.map(...);   // 新增的 Blocks
const deletedPromises = changes.deletedBlocks.map(...);   // 删除的 Blocks
const updatedPromises = changes.updatedBlocks.map(...);   // 更新的 Blocks ← 含 CodeEngineVersion

// 步骤 B: 等待所有合并操作完成
await Promise.all([...createdPromises, ...deletedPromises, ...updatedPromises]);

// 步骤 C: 更新 ResourceTemplateVersion Block（标记资源已升级到新版本）
await this.resourceTemplateVersionService.updateResourceTemplateVersion(...);

// 步骤 D: 将升级提示标记为 Resolved
await this.outdatedVersionAlertService.resolvesServiceTemplateUpdated(...);
```

**设计意图是串行顺序：A → B（并行合并） → C → D**

---

### 9.3 实际代码中的 Bug

#### Bug 1：`updatedPromises` 使用了 `forEach` 而非 `map`

在 [serviceTemplate.service.ts#L575-L577](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L575-L577)：

```typescript
// ❌ 错误写法：forEach 返回 undefined
const updatedPromises = changes.updatedBlocks.forEach(async (diff) => {
  return this.handleMergeUpdatedBlock(resourceId, diff, user, mergeOptions);
});
```

`Array.prototype.forEach()` 的返回值是 **`undefined`**。

这意味着：
- `updatedPromises` 的值是 `undefined`，而不是 Promise 数组
- `changes.updatedBlocks` 中的所有异步操作（**包括 CodeEngineVersion 的更新**）被启动后，**没有任何机制等待它们完成**
- 这些异步操作会以"失控"的方式在后台继续执行

#### Bug 2：`Promise.all` 接收的是嵌套数组而非扁平数组

在 [serviceTemplate.service.ts#L579](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L579)：

```typescript
// ❌ 错误写法：传入的是 [Promise[], Promise[], undefined]
await Promise.all([createdPromises, deletedPromises, updatedPromises]);
```

这里有两层问题：
1. `createdPromises` 和 `deletedPromises` 本身是 `Promise[]`（数组），被当作元素传入，而不是被展开
2. `updatedPromises` 是 `undefined`

`Promise.all` 的行为是：传入的可迭代对象中的每个值如果不是 Promise，会被 `Promise.resolve()` 包装后立即 resolve。

所以 `Promise.all([createdPromises, deletedPromises, undefined])` 实际上：
- `Promise.resolve(createdPromises)` → 立即 resolve（因为数组不是 Promise）
- `Promise.resolve(deletedPromises)` → 立即 resolve
- `Promise.resolve(undefined)` → 立即 resolve

**结果：`Promise.all` 几乎是立即返回的，没有任何一个实际的合并操作被等待。**

---

### 9.4 实际执行时序图（有 Bug 的情况）

```
用户调用 upgradeServiceToLatestTemplateVersion()
    │
    ├─ 步骤1: 读取资源、校验版本 ──────────────────────── ✅ 正确 await
    │
    ├─ 步骤2: compareResourceVersions ─────────────────── ✅ 正确 await
    │
    ├─ 步骤3: createdBlocks.map(async ...) ──── 启动 N 个 Promise（不等待）
    │       ├─ handleMergeCreatedBlock(PluginInstallation)
    │       └─ handleMergeCreatedBlock(CodeEngineVersion)
    │
    ├─ 步骤4: deletedBlocks.map(async ...) ──── 启动 M 个 Promise（不等待）
    │       └─ handleMergeDeletedBlock(全部 no-op)
    │
    ├─ 步骤5: updatedBlocks.forEach(async ...) ─ 启动 K 个 Promise（不等待，返回 undefined）
    │       ├─ handleMergeUpdatedBlock(PluginInstallation)
    │       └─ handleMergeUpdatedBlock(CodeEngineVersion)
    │              └─ resourceService.updateCodeGeneratorVersion()
    │                     ├─ prisma.resource.update({ codeGeneratorVersion, codeGeneratorStrategy })
    │                     └─ [如果是模板] templateCodeEngineVersionService.update()
    │
    ├─ 步骤6: await Promise.all([数组, 数组, undefined]) ──────── ❌ 立即返回，什么都没等
    │
    ├─ 步骤7: updateResourceTemplateVersion() ──────── ✅ await，此时 ResourceTemplateVersion 已更新为新版本
    │       └─ blockService.update<ResourceTemplateVersion>({ version: latestVersion.version })
    │
    ├─ 步骤8: resolvesServiceTemplateUpdated() ──────── ✅ await，OutdatedVersionAlert → Resolved
    │       └─ prisma.outdatedVersionAlert.updateMany({ status: Resolved })
    │
    └─ return resource ──────────────────────────────── 函数返回给用户
                      
          ╔══════════════════════════════════════════════════════════╗
          ║  此时后台还在运行：                                        ║
          ║  - PluginInstallation 的创建/更新                          ║
          ║  - CodeEngineVersion 导致的 Resource 表字段更新            ║
          ║  （这些操作可能成功，也可能失败，用户完全感知不到）          ║
          ╚══════════════════════════════════════════════════════════╝
```

---

### 9.5 CodeEngineVersion 覆盖局部字段的所有执行时机

`Resource.codeGeneratorVersion` 和 `Resource.codeGeneratorStrategy` 两个字段（即"局部覆盖字段"）被覆盖的入口共有 **3 个**：

#### 入口 A：用户通过 GraphQL API 手动更新（同步安全路径）

在 [resource.resolver.ts#L236-L241](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L236-L241)：

```typescript
async updateCodeGeneratorVersion(args, user) {
  return this.resourceService.updateCodeGeneratorVersion(args, user);
}
```

执行链路：
```
GraphQL Mutation → ResourceResolver.updateCodeGeneratorVersion()
    ↓ await
ResourceService.updateCodeGeneratorVersion() [resource.service.ts#L361-L413]
    ├─ await billing 权限校验
    ├─ await analytics.trackWithContext()
    ├─ await prisma.resource.update(
    │      { codeGeneratorVersion, codeGeneratorStrategy }
    │    )   ← ✅ 直接覆盖 Resource 表字段，有 await
    └─ [如果是 ServiceTemplate]
          await templateCodeEngineVersionService.update()
               ← ✅ 同步更新 CodeEngineVersion Block
```

**该路径完全同步，没有问题。**

---

#### 入口 B：模板升级时 CodeEngineVersion 出现在 `createdBlocks`

当模板**首次**设置代码生成器版本时（之前不存在 TemplateCodeEngineVersion Block），版本比较会将其归入 `createdBlocks`。

在 [serviceTemplate.service.ts#L601-L642](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L601-L642) 的 `handleMergeCreatedBlock()`：

```typescript
if (blockType === EnumBlockType.CodeEngineVersion) {
  return this.resourceService.updateCodeGeneratorVersion(
    { ... }, user
  );  // ← 覆盖 Resource 字段
}
```

**该调用被启动于 `changes.createdBlocks.map(async ...)`，但由于 Bug 2（Promise.all 嵌套数组），不被等待。**

---

#### 入口 C：模板升级时 CodeEngineVersion 出现在 `updatedBlocks`

当模板修改了已有的代码生成器版本（versionNumber 变化），版本比较会将其归入 `updatedBlocks`。

在 [serviceTemplate.service.ts#L645-L692](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts#L645-L692) 的 `handleMergeUpdatedBlock()`：

```typescript
if (blockType === EnumBlockType.CodeEngineVersion) {
  return this.resourceService.updateCodeGeneratorVersion(
    { ... }, user
  );  // ← 覆盖 Resource 字段
}
```

**该调用被启动于 `changes.updatedBlocks.forEach(async ...)`，由于 Bug 1（forEach 返回 undefined），完全不被等待。**

---

#### ❌ 注意：BuildService.updateCodeGeneratorVersion 不是此入口

[build.service.ts#L362-L387](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/build/build.service.ts#L362-L387) 中的 `updateCodeGeneratorVersion()` 更新的是 **Build 表** 的 `codeGeneratorVersion` 字段，记录某次构建实际使用的 DSG 版本，**不影响 Resource 表的局部覆盖字段**。两者是独立的。

---

### 9.6 CodeEngineVersion 覆盖的具体时序与不一致窗口

以入口 C（updatedBlocks 中出现 CodeEngineVersion）为例：

```
handleMergeUpdatedBlock()
    └─ resourceService.updateCodeGeneratorVersion()
           ├─ billing 权限校验 ................................ (1)
           ├─ analytics.trackWithContext() .................... (2)
           ├─ prisma.resource.update() → 覆盖 codeGeneratorVersion
           │    和 codeGeneratorStrategy 字段 .................. (3)
           └─ [如果是 ServiceTemplate]
                templateCodeEngineVersionService.update() ..... (4)
```

由于 Bug 1 + Bug 2，步骤 (3) 对 `Resource` 表字段的覆盖**发生在函数返回之后**。

这造成一个**不一致窗口**：

| 时间点 | ResourceTemplateVersion Block | Resource.codeGeneratorVersion | OutdatedVersionAlert | 风险 |
|--------|-------------------------------|-------------------------------|----------------------|------|
| 升级前 | v1.0.0 | 用户自定义值（如 v2.0.0 Specific） | New | — |
| 函数刚返回时 | v1.1.0 ✅ 已更新 | 用户自定义值 ❌ 尚未覆盖 | Resolved ✅ 已解决 | 此时触发构建，使用的仍是旧版本 |
| 若干毫秒后 | v1.1.0 | v1.1.0 对应的模板值 ✅ 被覆盖 | Resolved | — |
| （异常情况）步骤(3)抛错 | v1.1.0 | 用户自定义值 ❌ 永远不覆盖 | Resolved | 用户认为升级成功，实际永远停留在旧版本 |

---

## 十、升级提示创建入口的 await 链路与 Kafka 异步边界

### 10.1 告警触发的完整调用链

升级提示（OutdatedVersionAlert）的创建入口在 ServiceTemplate 发布新版本时被触发，完整的 await 链路如下：

```
GraphQL: createResourceVersion
    ↓ await
ResourceVersionService.create() [resourceVersion.service.ts#L40-L112]
    ├─ await validateVersion()
    ├─ await entityService.getLatestVersions()
    ├─ await blockService.getLatestVersions()
    ├─ await getLatest()
    ├─ await prisma.resourceVersion.create()  ← ✅ ResourceVersion 入库
    │
    └─ [如果是 ServiceTemplate]
         await outdatedVersionAlertService.triggerAlertsForTemplateVersion(...)
         ← ✅ 有 await，见 L97
             │
             ▼
triggerAlertsForTemplateVersion() [outdatedVersionAlert.service.ts#L179-L246]
    ├─ await prisma.resource.findUnique()  ← 校验是 ServiceTemplate
    ├─ await blockService.findManyByBlockType()  ← 查询所有使用该模板的资源
    │
    └─ for (const service of services)  ← ⚠️ 串行遍历，不是并行
         ├─ await resourceService.getServiceTemplateSettings()  ✅
         └─ await outdatedVersionAlertService.create(...)       ✅
                 │
                 ▼
         OutdatedVersionAlertService.create() [outdatedVersionAlert.service.ts#L45-L73]
             ├─ await prisma.outdatedVersionAlert.updateMany()  ✅ 旧告警 → Canceled
             ├─ await prisma.outdatedVersionAlert.create()      ✅ 新告警入库
             └─ await this.raiseNotifications(alertId, ...)    ✅ 有 await，见 L70
                     │
                     ▼
             raiseNotifications() [outdatedVersionAlert.service.ts#L75-L124]
                 ├─ await prisma.outdatedVersionAlert.findFirst()   ✅
                 ├─ await workspaceService.findWorkspaceUsers()     ✅
                 └─ for (const user of workspaceUsers)
                       this.kafkaProducerService
                           .emitMessage(...)   ← ❌ 没有 await
                           .catch(logger.error)
```

---

### 10.2 KafkaProducerService.emitMessage() 的异步边界

在 [KafkaProducer.service.ts#L22-L38](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/libs/util/nestjs/kafka/src/producer/KafkaProducer.service.ts#L22-L38) 中：

```typescript
async emitMessage(
  topic: string,
  message: DecodedKafkaMessage,
  schemaIds?: SchemaIds
): Promise<void> {
  const kafkaMessage = await this.serializer.serialize(message, schemaIds);
  return await new Promise((resolve, reject) => {
    this.kafkaClient.emit(topic, kafkaMessage).subscribe({
      error: (err) => reject(err),
      next: () => resolve(),    // ← 等待 Kafka broker 确认 ACK
    });
  });
}
```

**关键点**：
1. `emitMessage()` 的返回类型是 `Promise<void>`，它**确实是一个真正的 Promise**
2. 内部通过 `new Promise` 包装了 RxJS 的 `subscribe`，会等待 Kafka broker 返回 ACK 后才 resolve
3. 如果 Kafka 发送失败，Promise 会 reject

---

### 10.3 raiseNotifications() 的异步边界

在 [outdatedVersionAlert.service.ts#L75-L124](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L75-L124)：

```typescript
async raiseNotifications(alertId: string, alertInitiator: string) {
  const alert = await this.prisma.outdatedVersionAlert.findFirst(...);  // ✅ await
  const { resource } = alert;
  const { project } = resource;

  const workspaceUsers = await this.workspaceService.findWorkspaceUsers(...);  // ✅ await

  for (const user of workspaceUsers) {
    this.kafkaProducerService
      .emitMessage(...)          // ❌ 没有 await！
      .catch((error) => this.logger.error(...));
  }
}
```

**关键点**：
1. 函数被 `async` 标记，前两步数据库查询都正确使用了 `await`
2. 但在 `for` 循环中，`emitMessage()` **没有 await**，只附加了 `.catch()` 吞掉错误
3. 由于没有 await，循环会瞬间完成（启动了 N 个 fire-and-forget 的 Promise）
4. `raiseNotifications()` 随后隐式返回 `Promise.resolve(undefined)`
5. 外层 `create()` 中的 `await raiseNotifications()` 只等待了**数据库查询部分**，**不等待 Kafka 发送**

---

### 10.4 精确的异步边界图

```
OutdatedVersionAlertService.create()
    │
    ├─ await updateMany(Canceled)   ──── 数据库写入完成
    ├─ await create(New)            ──── 数据库写入完成
    │
    ├─ await raiseNotifications()
    │       ├─ await findFirst(alert)    ──┐
    │       └─ await findWorkspaceUsers()  ── 数据库部分完成
    │       │
    │       └─ for (user) {
    │            emitMessage()  ← fire-and-forget，不等待
    │            emitMessage()  ← fire-and-forget，不等待
    │            ...
    │          }
    │       │
    │       └─ raiseNotifications 返回（Kafka 仍在后台发送）
    │
    └─ create() 返回 alert 对象
           │
           └─ triggerAlertsForTemplateVersion 继续处理下一个服务（串行）
                  │
                  └─ 所有服务处理完后
                         └─ ResourceVersionService.create() 返回

          ╔══════════════════════════════════════════════╗
          ║  Kafka 消息此时仍在后台发送中：                ║
          ║  - serialize 消息                             ║
          ║  - kafkaClient.emit + subscribe 等待 ACK     ║
          ║  - 失败时只打日志，不影响主流程                ║
          ╚══════════════════════════════════════════════╝
```

---

### 10.5 triggerAlertsForTemplateVersion 的串行遍历

在 [outdatedVersionAlert.service.ts#L222-L244](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L222-L244)：

```typescript
for (const service of services) {
  const currentTemplateVersion = await this.resourceService.getServiceTemplateSettings(...);
  await this.create({...}, template.name);
}
```

使用 `for...of` + `await` 逐个服务**串行**创建告警。如果有 N 个服务使用该模板，将产生 N 次数据库读取 + N 次告警创建（每次告警创建内部还有多次 DB 查询和 Kafka 发送的启动），全部串行执行，时间复杂度为 O(N)。

---

## 十一、局部覆盖字段与升级提示状态的执行顺序分析

### 11.1 模板升级中的四类数据变更

模板升级过程中共发生四类数据写入操作，它们之间存在时序依赖：

| # | 操作 | 写入位置 | 被谁触发 |
|---|------|---------|---------|
| ① | CodeEngineVersion 合并 → 覆盖局部字段 | `Resource` 表 `codeGeneratorVersion` / `codeGeneratorStrategy` | `handleMergeCreatedBlock` 或 `handleMergeUpdatedBlock` |
| ② | PluginInstallation 合并 | `PluginInstallation` Block | `handleMergeCreatedBlock` 或 `handleMergeUpdatedBlock` |
| ③ | 更新 ResourceTemplateVersion Block | `Block` 表（blockType=ResourceTemplateVersion） | `resourceTemplateVersionService.updateResourceTemplateVersion()` |
| ④ | 告警状态 New → Resolved | `OutdatedVersionAlert` 表 | `outdatedVersionAlertService.resolvesServiceTemplateUpdated()` |

### 11.2 正确的依赖关系应该是

```
① CodeEngineVersion 覆盖 Resource 字段 ──┐
② PluginInstallation 合并 ───────────────┼── 必须全部完成
                                          │
                                          ▼
                         ③ ResourceTemplateVersion 更新
                                          │
                                          ▼
                         ④ OutdatedVersionAlert → Resolved
```

**理由**：
- 只有 ①② 全部成功完成，才说明"升级"真正生效，此时才能把 ③ 中的版本号推进
- 只有 ③ 版本号推进了，才能把 ④ 告警标记为已解决

如果 ①② 中有任何一项失败，应该：
1. 整体回滚或标记失败
2. 不推进 ③ 中的版本号
3. 不修改 ④ 中的告警状态

---

### 11.3 当前代码实际的执行顺序

由于 Bug 1 和 Bug 2，**实际执行顺序变成了并行且不可预测**：

```
┌─────────────────────────────────────────────────────────────┐
│ 函数主流程（串行）                                            │
│    ③ ResourceTemplateVersion 更新                            │
│    ④ OutdatedVersionAlert → Resolved                        │
│    return                                                    │
└─────────────────────────────────────────────────────────────┘
          ▲
          │ 完全独立，不等待下面的操作
          │
┌─────────┴───────────────────────────────────────────────────┐
│ 后台并发（fire-and-forget，无错误处理）                        │
│    ① CodeEngineVersion 覆盖 Resource 字段                    │
│    ② PluginInstallation 合并                                 │
└─────────────────────────────────────────────────────────────┘
```

**竞态场景举例**：

| 场景 | 结果 |
|------|------|
| ① 在 ③ 之前完成 | 看起来正常，但仍然是偶然的（依赖数据库延迟） |
| ① 在 ③ 之后、④ 之前完成 | ResourceTemplateVersion 先升级，CodeEngineVersion 后写入，告警此时仍为 New，随后被 ④ 置为 Resolved |
| ① 在 ④ 之后完成 | 用户看到告警已解决、版本已升级，但 Resource 表字段在某个时刻才被覆盖。如果构建在此刻发生，使用的是旧版本 |
| ① 抛出异常（如数据库超时） | 异常被吞掉，用户看到升级成功，但 CodeEngineVersion 实际上从未更新 |
| ① 和 ③ 同时写同一张 Resource 表 | 不涉及同一条记录（③ 写 Block 表，① 写 Resource 表），但存在数据一致性语义冲突 |

---

### 11.4 OutdatedVersionAlert.create() 的内部时序（已校准）

当模板发布新版本时（[outdatedVersionAlert.service.ts#L45-L73](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts#L45-L73)），告警创建的内部顺序如下：

```typescript
async create(args, alertInitiator) {
  // 步骤 1: 先将同资源同类型的 New 告警 → Canceled
  await this.prisma.outdatedVersionAlert.updateMany({
    where: { resourceId, blockId, type, status: New },
    data: { status: Canceled }
  });

  // 步骤 2: 创建新的 New 告警
  const alert = await this.prisma.outdatedVersionAlert.create({
    data: { ...args.data, status: New }
  });

  // 步骤 3: 发送通知（⚠️ 只 await 数据库查询部分，不 await Kafka 发送）
  await this.raiseNotifications(alert.id, alertInitiator);

  return alert;
}
```

**关键校准**：之前的分析称 `raiseNotifications()` "没有 await"是不准确的。实际上：
- ✅ `create()` 中 `await this.raiseNotifications()` —— 有 await
- ✅ `raiseNotifications()` 内部 `findFirst()` 和 `findWorkspaceUsers()` —— 有 await
- ❌ `raiseNotifications()` 内部循环中 `kafkaProducerService.emitMessage()` —— **没有 await**，只有 `.catch()`

即：数据库操作都被正确等待，只有 Kafka 消息发送是 fire-and-forget。

---

## 十二、Bug 修复建议

### 修复 1：将 `forEach` 改为 `map`

```typescript
// ❌ 旧代码
const updatedPromises = changes.updatedBlocks.forEach(async (diff) => {
  return this.handleMergeUpdatedBlock(resourceId, diff, user, mergeOptions);
});

// ✅ 修复后
const updatedPromises = changes.updatedBlocks.map(async (diff) => {
  return this.handleMergeUpdatedBlock(resourceId, diff, user, mergeOptions);
});
```

### 修复 2：将嵌套数组展开传入 `Promise.all`

```typescript
// ❌ 旧代码
await Promise.all([createdPromises, deletedPromises, updatedPromises]);

// ✅ 修复后
await Promise.all([...createdPromises, ...deletedPromises, ...updatedPromises]);
```

### 修复 3（可选增强）：raiseNotifications 中 Kafka 发送改为并行等待

```typescript
// ❌ 旧代码：fire-and-forget
for (const user of workspaceUsers) {
  this.kafkaProducerService
    .emitMessage(...)
    .catch((error) => this.logger.error(...));
}

// ✅ 修复后：并行发送，全部完成后返回
const emitPromises = workspaceUsers.map((user) =>
  this.kafkaProducerService
    .emitMessage(...)
    .catch((error) => this.logger.error(...))
);
await Promise.all(emitPromises);
```

### 修复 4（可选增强）：使用事务保证一致性

将 ①②③④ 纳入同一个数据库事务中，确保要么全部成功，要么全部回滚。当前所有操作都是独立的 `prisma.update()` / `prisma.updateMany()` 调用，没有事务包裹。

---

## 十三、涉及的核心文件清单

| 文件 | 职责 |
|------|------|
| [blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts) | Blueprint 管理与引擎校验 |
| [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | 资源创建、代码生成器版本更新 |
| [serviceTemplate.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | 从模板创建资源、模板升级、Block 合并 |
| [resourceVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resourceVersion/resourceVersion.service.ts) | 模板版本快照、版本差异比较、触发升级告警 |
| [resourceTemplateVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/resourceTemplateVersion/resourceTemplateVersion.service.ts) | 资源与模板版本的关联记录（Block） |
| [templateCodeEngineVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/templateCodeEngineVersion/templateCodeEngineVersion.service.ts) | 模板代码引擎版本历史（Block） |
| [outdatedVersionAlert.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/outdatedVersionAlert/outdatedVersionAlert.service.ts) | 过期版本告警的创建、状态流转、通知发送 |
| [version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/data-service-generator-catalog/src/version/version.service.ts) | DSG 版本解析（Specific / LatestMinor / LatestMajor） |
| [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/47-amplication/packages/amplication-server/src/core/build/build.service.ts) | 构建时读取资源的版本配置传入 DSG |
