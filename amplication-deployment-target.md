# Amplication Deployment Target 与环境配置层职责边界分析

## 一、核心概念与术语定义

在 Amplication 架构中，虽然代码中没有显式的 `DeploymentTarget` 命名，但存在一套等价的概念体系，可映射为三层：

| 概念层次 | 代码中对应实体 | 职责定位 |
|---------|--------------|---------|
| **Deployment Target（部署目标）** | `codeGeneratorName` + `codeGeneratorVersion` + `codeGeneratorStrategy` | 决定"用什么引擎生成代码"——即代码生成器的选型与版本 |
| **环境配置层（Configuration）** | `ServiceSettings` + `ResourceSettings` + `ProjectConfigurationSettings` | 决定"生成什么内容"——即功能开关、路径、认证方式等业务配置 |
| **运行环境绑定（Runtime Binding）** | `Environment` 实体 + `Deployment` 实体 | 决定"在哪里运行"——即构建产物与具体沙箱/部署环境的关联 |

---

## 二、Deployment Target：目标选择层

### 2.1 数据模型

部署目标选择主要通过 `Resource` 实体上的三个字段承载（参见 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L219-L259)）：

```prisma
model Resource {
  codeGeneratorVersion  String?
  codeGeneratorStrategy CodeGeneratorVersionStrategy @default(LatestMajor)
  codeGeneratorName     String?          // "NodeJS" | "DotNET" | "Blueprint"
  // ...
}
```

### 2.2 代码生成器枚举

在 [EnumCodeGenerator.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/dto/EnumCodeGenerator.ts#L1-L9) 中定义了三种目标类型：

```typescript
export enum EnumCodeGenerator {
  DotNet = "DotNet",
  NodeJs = "NodeJs",
  Blueprint = "Blueprint",
}
```

### 2.3 版本策略

版本策略枚举定义在 `EnumCodeGeneratorVersionStrategy`，支持三种模式：
- `Specific`：锁定具体版本号
- `LatestMinor`：锁定主版本号，自动升级次版本
- `LatestMajor`：始终使用最新主版本（默认）

### 2.4 目标选择流程

**入口**：[resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L415-L479)

1. **创建资源时选择目标** — `getAndValidateCodeGeneratorName()`
   - 校验用户订阅权限（如 `.NET` 为付费功能 `BillingFeature.CodeGeneratorDotNet`）
   - 检查 `BillingFeature.CodeGeneratorNodeJsOnly` 限制
   - 将枚举映射为字符串名称（`EnumCodeGenerator.NodeJs → "NodeJS"`）

2. **默认目标推断** — `getDefaultCodeGenerator()`
   - 若被限制为 Node.js-only，直接返回 `NodeJs`
   - 若用户有 `.NET` 权限，优先返回 `DotNet`
   - 兜底返回 `NodeJs`

3. **版本更新** — `updateCodeGeneratorVersion()`（L361-L413）
   - 独立的付费功能 `BillingFeature.CodeGeneratorVersion`
   - 更新 Resource 表的 `codeGeneratorVersion` 和 `codeGeneratorStrategy` 字段
   - 对 ServiceTemplate 类型，同步写入 `TemplateCodeEngineVersion` Block 留痕

4. **目标→镜像解析**（构建时）
   - [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L424-L440) 的 `codeGeneratorNameToContainerImageName()`
     - 名称为空 → 使用旧镜像名 `data-service-generator`
     - 名称非空 → 通过 `CodeGeneratorService` 查 catalog 获得完整镜像名
   - [version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator-catalog/src/version/version.service.ts#L96-L164) 的 `getCodeGeneratorVersion()`
     - 按 `codeGeneratorStrategy` 从 ECR/Prisma 解析出实际镜像 tag

### 2.5 关键职责边界

Deployment Target 层**只做选择与解析**，不关心配置内容：
- ✅ 选择代码生成器（NodeJS/DotNET/Blueprint）
- ✅ 选择版本策略与具体版本
- ✅ 校验目标对应的计费权限
- ✅ 将目标解析为实际容器镜像名 + tag
- ❌ 不注入业务配置（路径、认证、开关等）
- ❌ 不管理运行环境

---

## 三、配置注入层

配置注入是将分散存储的各类设置聚合为 DSG（Data Service Generator）可消费输入的过程。

### 3.1 配置的分层存储

所有配置以 **Block** 为载体持久化，通过 `EnumBlockType` 区分类型（[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L683-L699)）：

```
EnumBlockType
├── ServiceSettings              // Service 专属设置（认证、路径、开关）
├── ProjectConfigurationSettings // 项目级全局设置
├── ResourceSettings             // 通用资源设置（properties JSON）
├── PluginInstallation           // 插件安装及配置
├── PluginOrder                  // 插件执行顺序
├── CodeEngineVersion            // 模板代码引擎版本历史
├── Module / ModuleAction / ModuleDto  // 自定义模块
├── Topic / ServiceTopics        // 消息代理主题
├── PrivatePlugin / Package      // 私有插件与包
└── Relation                     // 资源间关系
```

### 3.2 核心配置服务

#### 3.2.1 ServiceSettings（Service 专用）

[serviceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/serviceSettings/serviceSettings.service.ts#L1-L197)

数据结构：
```typescript
{
  authProvider: EnumAuthProviderType.Jwt,
  authEntityName: "User",
  serverSettings: {
    serverPath: "apps/foo-server",
    generateServer: true,       // 硬编码为 true
    generateGraphQL: boolean,
    generateRestApi: boolean,
  },
  adminUISettings: {
    adminUIPath: "apps/foo-admin",
    generateAdminUI: boolean,
  }
}
```

**职责**：
- `getServiceSettingsValues()`：读取 Block，缺省时自动创建默认值
- `updateServiceSettings()`：增量更新，处理 `generateGraphQL=false` 时联动关闭 Admin UI
- `createDefaultServiceSettings()`：以 `DEFAULT_SERVICE_SETTINGS` 为底版 merge 用户输入

#### 3.2.2 ResourceSettings（通用资源）

[resourceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts#L1-L146)

数据结构：
```typescript
{
  properties: JsonValue  // 按 Blueprint 的 customProperties 校验
}
```

**职责**：
- `validateResourceSettingsProperties()`：按 Blueprint 定义校验 properties 合法性
- `updateResourceSettings()`：不存在则创建 `DEFAULT_RESOURCE_SETTINGS`

#### 3.2.3 ProjectConfigurationSettings（项目级）

- 在 `ResourceService.createProjectConfiguration()` 中随 Project 自动创建唯一实例
- 包含 `overrideCustomizableFilesInGit` 等全局策略

### 3.3 配置聚合点：BuildService.getDSGResourceData()

[build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts#L1384-L1521)

这是**最核心的配置注入函数**，将所有维度配置汇聚为 `DSGResourceData`：

```typescript
async getDSGResourceData(
  resource, buildId, buildVersion, user, rootGeneration = true
): Promise<CodeGenTypes.DSGResourceData> {
  return {
    resourceType: resource.resourceType,
    buildId,
    entities: await this.getOrderedEntities(buildId),
    roles: await this.getResourceRoles(resourceId),
    pluginInstallations: orderedPlugins,        // 已过滤 enabled=true
    moduleContainers, moduleActions, moduleDtos,
    relations, resourceSettings, topics, serviceTopics,
    resourceInfo: {
      name, description, version: buildVersion,
      id: resourceId, url,
      settings: serviceSettings,                // ServiceSettings
      codeGeneratorVersionOptions: {
        codeGeneratorVersion: resource.codeGeneratorVersion,
        codeGeneratorStrategy: CodeGeneratorVersionStrategy[resource.codeGeneratorStrategy],
      },
      codeGeneratorName: resource.codeGeneratorName,
      properties: resource.properties,
    },
    otherResources,                              // 递归关联资源
  };
}
```

**注入的关键步骤**：
1. 拉取资源自身的实体、角色、模块、主题等数据
2. 拉取 `ServiceSettings`（仅 Service 类型）
3. 拉取 `ResourceSettings`
4. 从 Resource 表读取 `codeGeneratorName`、`codeGeneratorVersion`、`codeGeneratorStrategy`（即 Deployment Target）
5. 组装 `resourceInfo.settings`（业务配置）与 `resourceInfo.codeGeneratorVersionOptions`（目标信息）
6. 递归拉取关联资源（`otherResources`），同样执行上述流程

聚合后的数据通过 `saveDsgResourceDataToSharedStorage()` 写入共享文件系统（路径由 `DSG_RESOURCE_DATA_BASE_FOLDER` + `DSG_RESOURCE_DATA_FILE` 环境变量控制）。

### 3.4 构建侧读取与拆分

[build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159) `runBuild()`：
1. 从共享存储读取 `DSGResourceData`
2. 用 `codeGeneratorName` 解析出容器镜像名
3. 用 `codeGeneratorVersionOptions` 解析出镜像 tag
4. 将 Server 和 AdminUI 拆分为两个 job（当 DSG 版本 >= `FEATURE_SPLIT_JOBS_MIN_DSG_VERSION` 时）

[build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L35-L91) `splitBuildsIntoJobs()`：
- `generateServer=true` → 创建 server job，将 `adminUISettings.generateAdminUI` 置 false
- `generateAdminUI=true` → 创建 admin-ui job，将 `serverSettings.generateServer` 置 false

### 3.5 DSG 侧上下文装配

[data-service-generator/src/prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator/src/prepare-context.ts#L41-L124)

```typescript
export async function prepareContext(dSGResourceData, internalLogger, pluginInstallationPath?) {
  const { pluginInstallations, entities, roles, resourceInfo: appInfo,
          otherResources, moduleActions, moduleContainers, moduleDtos } = dSGResourceData;

  const plugins = await registerPlugins(resourcePlugins, pluginInstallationPath);
  // ...

  context.appInfo = appInfo;            // 注入 settings + codeGeneratorVersionOptions
  context.roles = roles;
  context.entities = normalizedEntities;
  context.plugins = plugins;            // 插件实例化后注入
  context.serverDirectories = dynamicServerPathCreator(appInfo.settings.serverSettings.serverPath);
  context.clientDirectories = dynamicClientPathCreator(appInfo.settings.adminUISettings.adminUIPath);
  // ...
}
```

### 3.6 进程级环境变量

各微服务独立的 `env.ts` 定义了基础设施层配置，不属于"资源级配置"：

| 服务 | 文件 | 关键变量 |
|-----|------|---------|
| amplication-server | [env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/env.ts#L1-L68) | `HOST`, `CLIENT_HOST`, GitHub/Bitbucket/GitLab/Azure OAuth 凭证, `DSG_RESOURCE_DATA_BASE_FOLDER`, `BILLING_ENABLED` |
| amplication-build-manager | [env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/env.ts#L1-L33) | `DSG_RUNNER_URL`, Redis 连接串, `BUILD_ARTIFACTS_*`, `DSG_JOBS_*`, `DSG_CATALOG_SERVICE_URL` |
| data-service-generator | `.env` | DSG 自身运行时参数 |

**关键边界**：这些 env 变量是基础设施运维配置，与用户在 UI 上配置的 ServiceSettings/ResourceSettings 完全分离。

### 3.7 关键职责边界

配置注入层**只做数据聚合与透传**，不决定生成目标：
- ✅ 按资源维度聚合 ServiceSettings/ResourceSettings/ProjectConfigurationSettings
- ✅ 拉取实体、角色、模块、插件等业务数据
- ✅ 将目标信息（codeGeneratorName/Version/Strategy）原样放入 `resourceInfo.codeGeneratorVersionOptions`
- ✅ 按 generateServer/generateAdminUI 开关拆分 job
- ✅ 在 DSG 上下文内装配 settings → `serverDirectories`/`clientDirectories` 等派生值
- ❌ 不选择代码生成器（仅读取已存在的目标信息）
- ❌ 不管理运行环境地址

---

## 四、运行环境绑定层

### 4.1 Environment 实体

[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L615-L627)

```prisma
model Environment {
  id          String       @id @default(cuid())
  resourceId  String
  name        String                    // "Sandbox environment"
  description String?
  address     String                    // cuid()，用作沙箱唯一标识
  resource    Resource     @relation(...)
  deployments Deployment[]
}
```

[environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts#L1-L58)：
- `createDefaultEnvironment(resourceId)`：创建 Resource 时自动创建名为 `"Sandbox environment"` 的默认 Environment
- `getDefaultEnvironment(resourceId)`：按名称查找

**含义**：`Environment` 代表一个部署目标槽位（如 Sandbox、Staging、Production），目前实现仅为单个 Sandbox。`address` 字段是沙箱实例的唯一标识（cuid）。

### 4.2 Deployment 实体

[schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L629-L644)

```prisma
model Deployment {
  id              String               @id @default(cuid())
  userId          String
  buildId         String               // 关联到具体 Build
  environmentId   String               // 关联到具体 Environment
  status          EnumDeploymentStatus // Completed/Waiting/Failed/Removed
  message         String?
  actionId        String               // 部署过程的 Action/Step/Log
  statusQuery     Json?                // 查询运行状态的元数据
  statusUpdatedAt DateTime?
  build           Build      @relation(...)
  environment     Environment @relation(...)
}
```

### 4.3 绑定关系图

```
Project
  └── Resource (含 codeGeneratorName/Version/Strategy = Deployment Target)
        ├── Build (每次提交构建)
        │     └── Deployment (多对一 Build → Environment)
        │           └── Action → ActionStep → ActionLog (部署过程记录)
        └── Environment (Sandbox / Staging / Production 槽位)
              ├── address: 沙箱唯一标识
              └── deployments
```

### 4.4 关键职责边界

运行环境绑定层**只做部署记录与沙箱关联**：
- ✅ 创建/查询 Resource 关联的 Environment 槽位
- ✅ 记录每次 Build 在某个 Environment 上的 Deployment 状态
- ✅ 存储部署状态查询元数据（`statusQuery`）
- ❌ 不参与代码生成目标选择
- ❌ 不存储业务配置（仅引用 Build）

---

## 五、端到端数据流：从目标选择到运行绑定

以"用户创建 Service → 构建 → 部署"为例，各层职责接力：

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. 用户在 UI 创建 Service                                            │
│    ResourceService.createService()                                   │
│      │                                                               │
│      ├─ [Deployment Target]                                          │
│      │    getDefaultCodeGenerator() → EnumCodeGenerator.NodeJs       │
│      │    getAndValidateCodeGeneratorName() → "NodeJS"               │
│      │    写入 Resource.codeGeneratorName                            │
│      │    codeGeneratorStrategy = LatestMajor (默认)                 │
│      │                                                               │
│      ├─ [Configuration]                                              │
│      │    ServiceSettingsService.createDefaultServiceSettings()      │
│      │    → 创建 ServiceSettings Block (auth, paths, 开关)           │
│      │    installPlugins() → 默认 db-postgres 插件                   │
│      │                                                               │
│      └─ [Runtime Binding]                                            │
│           EnvironmentService.createDefaultEnvironment()              │
│           → 创建 "Sandbox environment" Environment                   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. 用户 Commit → 触发 Build                                          │
│    BuildService.create()                                             │
│      │                                                               │
│      └─ [Configuration Injection]                                    │
│           getDSGResourceData()                                       │
│           → 聚合 entities/roles/plugins/modules/topics               │
│           → resourceInfo.settings = ServiceSettings                  │
│           → resourceInfo.codeGeneratorVersionOptions =               │
│                { codeGeneratorVersion, codeGeneratorStrategy }       │
│           → resourceInfo.codeGeneratorName = "NodeJS"                │
│           → saveDsgResourceDataToSharedStorage() 写磁盘              │
│           → Kafka: CODE_GENERATION_REQUEST_TOPIC                     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Build Manager 消费 Kafka 消息                                     │
│    BuildRunnerService.runBuild()                                     │
│      │                                                               │
│      ├─ [Deployment Target]                                          │
│      │    codeGeneratorNameToContainerImageName("NodeJS")            │
│      │      → 查 catalog 得镜像名 "data-service-generator"           │
│      │    VersionService.getCodeGeneratorVersion()                   │
│      │      → 按 strategy 解析出 tag "v2.0.1"                        │
│      │                                                               │
│      └─ [Configuration]                                              │
│           splitBuildsIntoJobs()                                      │
│           → generateServer? 拆出 server job                          │
│           → generateAdminUI? 拆出 admin-ui job                       │
│           → DSG_RUNNER_URL 触发 Argo 工作流，携带镜像名+tag           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. DSG 容器执行代码生成                                               │
│    prepareContext()                                                  │
│      │                                                               │
│      └─ [Configuration]                                              │
│           context.appInfo = resourceInfo (含 settings)               │
│           context.serverDirectories =                                │
│             dynamicServerPathCreator(serverPath)                     │
│           context.clientDirectories =                                │
│             dynamicClientPathCreator(adminUIPath)                    │
│           context.plugins = registerPlugins(pluginInstallations)     │
│           → 执行各代码生成器生成文件                                  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. 代码生成完成 → 部署到 Sandbox                                      │
│    [Runtime Binding]                                                 │
│      创建 Deployment 记录                                            │
│        buildId → 当前 Build                                          │
│        environmentId → 默认 Sandbox Environment                      │
│        status = EnumDeploymentStatus.Waiting/Completed               │
│        statusQuery → 保存查询沙箱运行状态所需元数据                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、当前职责边界存在的模糊点与潜在风险

### 6.1 Deployment Target 信息散落在多处

**现状**：
- `Resource.codeGeneratorName` / `codeGeneratorVersion` / `codeGeneratorStrategy` 三个独立字段
- `TemplateCodeEngineVersion` Block 又存了一份版本历史（仅 ServiceTemplate）
- Build 表还冗余保存了 `codeGeneratorVersion`

**风险**：数据同步一致性需要额外维护（如 `BuildService.updateCodeGeneratorVersion()`）。

### 6.2 Environment 实体当前功能较薄

**现状**：`Environment.address` 仅存 cuid，没有明确的 provider/region/credentials 等字段；`Deployment.statusQuery` 的 JSON schema 未约束。

**风险**：多环境（Staging/Production）扩展时需要大规模数据模型变更。

### 6.3 Deployment Target 与 Configuration 在 resourceInfo 中的耦合

**现状**：`resourceInfo` 同时承载了 `settings`（配置）和 `codeGeneratorVersionOptions` + `codeGeneratorName`（目标），属于两个不同关注点但封装在同一对象内。

**风险**：未来增加更多 target 维度（如部署目标云厂商、region）会让 `resourceInfo` 膨胀。

### 6.4 进程环境变量 vs 资源配置边界清晰，但命名易混淆

**现状**：
- `DSG_RESOURCE_DATA_BASE_FOLDER`：进程级 env 变量，控制共享存储路径
- `ResourceSettings` / `ServiceSettings`：资源级 Block 数据

两者命名均含"settings"，需开发者理解上下文区分。

---

## 七、各层核心文件索引

| 层次 | 文件 | 关键函数/类 |
|-----|------|------------|
| **Deployment Target** | [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | `getAndValidateCodeGeneratorName`, `getDefaultCodeGenerator`, `updateCodeGeneratorVersion` |
| | [EnumCodeGenerator.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/dto/EnumCodeGenerator.ts) | `EnumCodeGenerator` 枚举 |
| | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | `codeGeneratorNameToContainerImageName`, `runBuild` |
| | [version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator-catalog/src/version/version.service.ts) | `getCodeGeneratorVersion` |
| **Configuration** | [serviceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/serviceSettings/serviceSettings.service.ts) | `getServiceSettingsValues`, `updateServiceSettings` |
| | [resourceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts) | `updateResourceSettings`, `validateResourceSettingsProperties` |
| | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts) | `getDSGResourceData`, `saveDsgResourceDataToSharedStorage` |
| | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | `splitBuildsIntoJobs` |
| | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator/src/prepare-context.ts) | `prepareContext` |
| | [dsg-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator/src/dsg-context.ts) | `DsgContext` 类 |
| | [dsg-resource-data.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/libs/util/code-gen-types/src/dsg-resource-data.ts) | `DSGResourceData` 类型 |
| **Runtime Binding** | [environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts) | `createDefaultEnvironment`, `getDefaultEnvironment` |
| | [Environment.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/dto/Environment.ts) | `Environment` DTO |
| | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L615-L644) | `Environment`, `Deployment` 模型, `EnumDeploymentStatus` |
| **进程级 Env** | [server/env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/env.ts) | `Env` 常量 |
| | [build-manager/env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/env.ts) | `Env` 常量 |
