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

## 八、涉及的核心文件清单

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
