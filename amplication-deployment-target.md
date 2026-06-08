# Amplication Deployment Target 与环境配置层职责边界分析（代码核查版）

> **核查说明**：沿实际代码路径逐函数核对，与初始理解有多处重大差异，已在各节标注 ✅（真实存在）/ ❌（未实现/理解偏差）。

---

## 一、核心概念与术语定义

在 Amplication 架构中，代码中没有显式的 `DeploymentTarget` 命名，但存在一套等价的概念体系。以下映射关系均已沿代码验证：

| 概念层次 | 代码中对应实体 | 职责定位 | 验证状态 |
|---------|--------------|---------|---------|
| **Deployment Target（部署目标）** | `Resource.codeGeneratorName` + `Resource.codeGeneratorVersion` + `Resource.codeGeneratorStrategy` | 决定"用什么引擎生成代码"——即代码生成器的选型与版本 | ✅ 已验证 |
| **环境配置层（Configuration）** | `ServiceSettings` Block + `ResourceSettings` Block + `ProjectConfigurationSettings` Block | 决定"生成什么内容"——即功能开关、路径、认证方式等业务配置 | ✅ 已验证 |
| **运行环境绑定（Runtime Binding）** | `Environment` 表 + `Deployment` 表（均在 Prisma schema 中定义） | 预留"在哪里运行"的模型槽位 | ❌ **模型已定义但业务逻辑未实现** |

---

## 二、Deployment Target：目标选择层

### 2.1 数据模型

部署目标选择完全通过 `Resource` 实体上的三个字段承载（参见 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L219-L259)）：

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

通过映射表 `CODE_GENERATOR_ENUM_TO_NAME_AND_LICENSE`（[resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L126-L141)）转换为实际存储的字符串名称：

| EnumCodeGenerator | codeGeneratorName | 计费 License |
|------------------|-------------------|-------------|
| `NodeJs` | `"NodeJS"` | 无（免费） |
| `DotNet` | `"DotNET"` | `BillingFeature.CodeGeneratorDotNet` |
| `Blueprint` | `"Blueprint"` | 无（免费） |

### 2.3 版本策略

`EnumCodeGeneratorVersionStrategy` 支持三种模式：
- `Specific`：锁定具体版本号
- `LatestMinor`：锁定主版本号，自动升级次版本
- `LatestMajor`：始终使用最新主版本（**数据库默认值**）

### 2.4 目标选择流程（逐函数验证）

#### 2.4.1 入口 A：创建 Service/MessageBroker/Component 时选择目标

**调用链**：
`createService()` / `createMessageBroker()` / `createComponent()` → `createResource()`

在 [resource.service.ts:333-345](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L333-L345)：

```typescript
const { codeGenerator, ...rest } = args.data;

const resource = await this.prisma.resource.create({
  data: {
    ...rest,
    codeGeneratorName: await this.getAndValidateCodeGeneratorName(
      codeGenerator,
      user
    ),
    // ...
  },
});
```

关键点：
- `codeGeneratorName` 在此处写入（由 Enum → String 转换）
- `codeGeneratorVersion` 和 `codeGeneratorStrategy` **不在此处显式写入**，由 Prisma schema 的 `@default(LatestMajor)` 和 nullable 默认取 null

#### 2.4.2 `getAndValidateCodeGeneratorName()` 校验逻辑

[resource.service.ts:415-450](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L415-L450)：
1. 如果 workspace 被限制 `CodeGeneratorNodeJsOnly`，则非 NodeJs 直接抛错
2. 如果目标是 DotNet，检查 `BillingFeature.CodeGeneratorDotNet` 访问权限
3. 返回映射后的字符串名称

#### 2.4.3 `getDefaultCodeGenerator()` 推断逻辑

[resource.service.ts:455-479](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L455-L479)：
1. 若被限制 NodeJs-only → 返回 `NodeJs`
2. 若用户有 DotNet 权限 → **优先返回 DotNet**
3. 兜底返回 `NodeJs`

在 `createServiceWithDefaultSettings()` 中使用（[resource.service.ts:693-694](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L693-L694)）：

```typescript
const actualCodeGenerator =
  codeGenerator || (await this.getDefaultCodeGenerator(user));
```

#### 2.4.4 版本更新入口

`updateCodeGeneratorVersion()`（[resource.service.ts:361-413](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L361-L413)）：
- 独立的付费功能 `BillingFeature.CodeGeneratorVersion`
- 更新 Resource 表的 `codeGeneratorVersion` 和 `codeGeneratorStrategy` 字段
- 对 ServiceTemplate 类型，同步写入 `TemplateCodeEngineVersion` Block 留痕

#### 2.4.5 目标→镜像解析（构建阶段）

在 Build Manager 的 [build-runner.service.ts:424-440](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L424-L440) 的 `codeGeneratorNameToContainerImageName()`：
- 名称为空 → 使用旧镜像名 `data-service-generator`
- 名称非空 → 通过 `CodeGeneratorService` 查询 catalog 获得完整镜像名

在 [version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator-catalog/src/version/version.service.ts#L96-L164) 的 `getCodeGeneratorVersion()`：
- 按 `codeGeneratorStrategy` 从 ECR/Prisma 解析出实际镜像 tag

### 2.5 关键职责边界

Deployment Target 层**只做选择与解析**，不关心配置内容：
- ✅ 选择代码生成器（NodeJS/DotNET/Blueprint）并校验计费权限
- ✅ 存储版本策略与具体版本号
- ✅ 构建阶段将目标解析为实际容器镜像名 + tag
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

**调用时机**：创建 Service 时由 `createServiceDefaultObjects()` 触发（[resource.service.ts:647-651](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L647-L651)）。

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

[build.service.ts:1384-1521](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts#L1384-L1521)

这是**最核心的配置注入函数**，将所有维度配置汇聚为 `DSGResourceData`。关键代码路径：

```typescript
async getDSGResourceData(resource, buildId, buildVersion, user, rootGeneration = true) {
  // 1. 拉取业务数据（配置注入层核心工作）
  const orderedPlugins = (await pluginInstallationService.getOrderedPluginInstallations(resourceId))
    .filter(p => p.enabled);
  const modules = await moduleService.findMany(...);
  const relations = await resourceService.getRelations(resourceId);
  const resourceSettings = await resourceSettingsService.getResourceSettingsBlock(...);
  const serviceSettings = resourceType === Service
    ? await serviceSettingsService.getServiceSettingsValues(...)
    : undefined;
  const entities = rootGeneration ? await getOrderedEntities(buildId) : [];
  const roles = await getResourceRoles(resourceId);

  // 2. 递归拉取关联资源
  if (rootGeneration) otherResources = await Promise.all(
    relatedResources.map(r => getDSGResourceData(r, buildId, buildVersion, user, false))
  );

  // 3. 组装 DSGResourceData（此处是 Deployment Target 与 Configuration 的汇合点）
  return {
    resourceType, buildId,
    entities, roles, pluginInstallations, moduleContainers, moduleActions,
    moduleDtos, relations, resourceSettings, topics, serviceTopics,
    resourceInfo: {
      name, description, version, id, url, properties: resource.properties,
      settings: serviceSettings,                                    // ← 来自配置注入层
      codeGeneratorVersionOptions: {                                // ← 来自 Deployment Target 层
        codeGeneratorVersion: resource.codeGeneratorVersion,
        codeGeneratorStrategy: CodeGeneratorVersionStrategy[resource.codeGeneratorStrategy],
      },
      codeGeneratorName: resource.codeGeneratorName,                // ← 来自 Deployment Target 层
    },
    otherResources,
  };
}
```

**真实汇合点**：`resourceInfo` 对象同时承载了两个不同关注点——
- `settings`（Configuration）：业务配置
- `codeGeneratorVersionOptions` + `codeGeneratorName`（Deployment Target）：生成目标

聚合后的数据通过 `saveDsgResourceDataToSharedStorage()`（[build.service.ts:541-560](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts#L541-L560)）写入共享文件系统：
- 路径由 `DSG_RESOURCE_DATA_BASE_FOLDER`（默认 `/amplication-data/dsg-resource-data`）+ `buildId` + `DSG_RESOURCE_DATA_FILE`（默认 `resource-data.json`）组成

### 3.4 构建侧读取与拆分

[build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159) `runBuild()`：
1. 从共享存储读取 `DSGResourceData`
2. 用 `codeGeneratorName` → `codeGeneratorNameToContainerImageName()` 解析容器镜像名
3. 用 `codeGeneratorVersionOptions` → `VersionService.getCodeGeneratorVersion()` 解析镜像 tag
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
  context.appInfo = appInfo;            // 注入 settings + codeGeneratorVersionOptions
  context.roles = roles;
  context.entities = normalizedEntities;
  context.plugins = plugins;
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

**边界**：这些 env 变量是基础设施运维配置，与用户在 UI 上配置的 ServiceSettings/ResourceSettings 完全分离。

### 3.7 关键职责边界

配置注入层**只做数据聚合与透传**，不决定生成目标：
- ✅ 按资源维度聚合 ServiceSettings/ResourceSettings/ProjectConfigurationSettings
- ✅ 拉取实体、角色、模块、插件等业务数据
- ✅ 将目标信息（codeGeneratorName/Version/Strategy）原样放入 `resourceInfo.codeGeneratorVersionOptions`
- ✅ 按 generateServer/generateAdminUI 开关拆分 job
- ✅ 在 DSG 上下文内装配 settings → `serverDirectories`/`clientDirectories` 等派生值
- ❌ 不选择代码生成器（仅读取已存在的目标信息）
- ❌ **不管理运行环境**（见下一节）

---

## 四、运行环境绑定层 —— ❌ 模型已定义，业务逻辑未实现

### 4.1 核查结论总览

| 功能 | Prisma 模型 | 实际写入代码 | 实际查询代码 | 状态 |
|-----|-----------|------------|------------|-----|
| Environment 创建 | ✅ `Environment` 表 | ❌ **无**（`createDefaultEnvironment` 仅在 spec 测试中被调用） | ✅ `ResourceResolver.environments` 只读查询 | **模型存在，创建逻辑缺失** |
| Deployment 写入 | ✅ `Deployment` 表 | ❌ **完全无 `prisma.deployment.create` 调用** | ✅ 权限校验中 `prisma.deployment.count` | **纯预留模型** |
| 沙箱部署流程 | - | ❌ 代码中无任何 Deployment 创建流程 | - | **未实现** |
| 容器状态查询 | ✅ `Build.containerStatusQuery/UpdatedAt` 字段 | ❌ **无任何读写代码** | ❌ 无 | **字段预留未启用** |

### 4.2 Environment 实体核查

[schema.prisma:615-627](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L615-L627)

```prisma
model Environment {
  id          String       @id @default(cuid())
  resourceId  String
  name        String
  description String?
  address     String
  resource    Resource     @relation(...)
  deployments Deployment[]
}
```

[environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts#L1-L58) 暴露：
- `createDefaultEnvironment(resourceId)`：创建名为 `"Sandbox environment"` 的 Environment
- `getDefaultEnvironment(resourceId)`：按名称查找
- `findMany()`：通用查询

**实际调用情况核查**：
- `environmentService.findMany()` → 仅在 [resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135) 的 GraphQL `environments` ResolveField 中用于只读查询
- `createDefaultEnvironment()` → **仅在 `environment.service.spec.ts` 测试文件中被调用**，业务代码零调用
- `getDefaultEnvironment()` → 整个代码库无调用

**结论**：Environment 表在数据库中可能有历史数据或通过外部脚本初始化，但**当前代码版本不会在 Resource 创建时自动创建 Environment**。

### 4.3 Deployment 实体核查

[schema.prisma:629-644](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L629-L644)

```prisma
model Deployment {
  id              String               @id @default(cuid())
  userId          String
  buildId         String
  environmentId   String
  status          EnumDeploymentStatus
  message         String?
  actionId        String
  statusQuery     Json?
  statusUpdatedAt DateTime?
  build           Build      @relation(...)
  environment     Environment @relation(...)
}
```

**实际调用情况核查**：
- `prisma.deployment.create()` / `prisma.deployment.update()` / `prisma.deployment.delete()` → **整个代码库零结果**
- 唯一使用：[validation-functions.ts:267-275](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L275) 中权限校验 `prisma.deployment.count(...)`（仅 count 查询，确保用户有权访问某个 deploymentId）
- 邮件通知：[mail.service.ts:55-82](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/mail/mail.service.ts#L55-L82) 中有 `sendDeploymentNotification()`，但被 `IS_EMAIL_DEPLOYMENT_NOTIFICATION = false`（L18）永久屏蔽，且无任何调用方

**结论**：Deployment 是**纯预留模型**，对应功能尚未实现。

### 4.4 Build.containerStatusQuery 字段核查

[schema.prisma:496-497](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L496-L497)

```prisma
model Build {
  containerStatusQuery     Json?
  containerStatusUpdatedAt DateTime?
  // ...
}
```

**实际调用情况核查**：整个代码库 grep 无任何 `containerStatusQuery` 或 `containerStatusUpdatedAt` 的读写代码（除 schema 和 migration SQL 外）。

**结论**：字段已预留但未启用。

### 4.5 运行环境绑定层的真实职责

根据代码核查，当前版本中**运行环境绑定层的实际职责为空**。所有 Environment/Deployment 相关的：
- 数据模型：已定义（Prisma schema）
- GraphQL 查询字段：已暴露（`Resource.environments`）
- 权限校验：已接入（`AuthorizableOriginParameter.DeploymentId`）
- **业务写入逻辑：未实现**

---

## 五、端到端真实代码流：从目标选择到代码生成与 Git 推送

> **重要修正**：之前分析中的"部署到 Sandbox Environment → 写入 Deployment 记录"在代码中不存在。实际流程止于 **代码推送到 Git 仓库（创建 PR）**。

以"用户创建 Service → Commit → 构建 → 推送 Git"为例，完整的代码验证流程：

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. 用户创建 Service（Resource 创建阶段）                              │
│                                                                      │
│    入口：createService() / createServiceWithDefaultSettings()        │
│    [resource.service.ts:592-627](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L592-L627) │
│                                                                      │
│      ├─ [Deployment Target]                                          │
│      │    getDefaultCodeGenerator() → EnumCodeGenerator              │
│      │    createResource() 中的                                      │
│      │      getAndValidateCodeGeneratorName(enum, user)              │
│      │        → 校验计费权限（NodeJsOnly / CodeGeneratorDotNet）     │
│      │        → 映射为字符串 "NodeJS" / "DotNET" / "Blueprint"       │
│      │        → 写入 Resource.codeGeneratorName                      │
│      │    codeGeneratorVersion=null（使用数据库默认值）              │
│      │    codeGeneratorStrategy=LatestMajor（Prisma @default）       │
│      │                                                               │
│      ├─ [Configuration]                                              │
│      │    createServiceDefaultObjects() →                            │
│      │      1. 创建 "user" ResourceRole                              │
│      │      2. serviceSettingsService.createDefaultServiceSettings() │
│      │         → 创建 ServiceSettings Block（auth、paths、开关）     │
│      │    createServiceWithDefaultSettings() 中按 codeGenerator      │
│      │      选择默认 DB 插件：                                       │
│      │        NodeJs → db-postgres                                   │
│      │        DotNet → dotnet-db-sqlserver                           │
│      │    installPlugins() → 创建 PluginInstallation Block          │
│      │                                                               │
│      └─ [Runtime Binding]                                            │
│           ❌ EnvironmentService.createDefaultEnvironment() **未调用** │
│           ❌ 不创建 Environment / Deployment                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 2. 用户 Commit → 触发 Build                                          │
│    BuildService.create()                                             │
│    [build.service.ts:268-352](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L352) │
│                                                                      │
│      ├─ 创建 Build 记录（status=Running, gitStatus=Waiting）         │
│      ├─ 关联最新 EntityVersions                                      │
│      ├─ 创建 Action + 初始 Step "ADD_TO_QUEUE"                       │
│      │                                                               │
│      └─ 分支：                                                       │
│         ├─ 有私有插件 → downloadPrivatePlugins() → Kafka             │
│         │   DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC                   │
│         │   → 下载完成后回调 generate()                               │
│         └─ 无私有插件 → 直接 generate()                               │
│                                                                      │
│    generate() [build.service.ts:568-618]                              │
│      │                                                               │
│      └─ [Configuration Injection（Deployment Target 与配置汇合）]    │
│           getDSGResourceData()                                       │
│             → 聚合 entities/roles/plugins/modules/topics             │
│             → resourceInfo.settings = ServiceSettings（配置）        │
│             → resourceInfo.codeGeneratorVersionOptions               │
│                  = { codeGeneratorVersion, codeGeneratorStrategy }   │
│                  （从 Resource 表读出，属于 Deployment Target）      │
│             → resourceInfo.codeGeneratorName = "NodeJS"              │
│                  （从 Resource 表读出，属于 Deployment Target）      │
│           saveDsgResourceDataToSharedStorage()                       │
│             → 写入 /amplication-data/dsg-resource-data/{buildId}/    │
│                resource-data.json                                     │
│           Kafka: CODE_GENERATION_REQUEST_TOPIC                       │
│             → value = { resourceId, buildId }（轻量消息，不传数据）  │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 3. Build Manager 消费 Kafka 消息                                     │
│    build-runner.controller.ts → BuildRunnerService.runBuild()        │
│    [build-runner.service.ts:109-159](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159) │
│                                                                      │
│      ├─ [Deployment Target]                                          │
│      │    从共享存储读取 DSGResourceData                              │
│      │    codeGeneratorNameToContainerImageName("NodeJS")            │
│      │      → 查 catalog 得镜像名 "data-service-generator"           │
│      │    VersionService.getCodeGeneratorVersion()                   │
│      │      → 按 strategy（LatestMajor/Specific/LatestMinor）        │
│      │        解析出实际镜像 tag "v2.0.1"                            │
│      │                                                               │
│      └─ [Configuration]                                              │
│           splitBuildsIntoJobs()                                      │
│             → generateServer? 拆出 server job                        │
│             → generateAdminUI? 拆出 admin-ui job                     │
│           → 触发 DSG Runner（Argo 工作流/容器）                      │
│             携带：镜像名 + tag + DSGResourceData 共享路径            │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 4. DSG 容器执行代码生成                                               │
│    prepareContext()                                                  │
│    [prepare-context.ts:41-124](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator/src/prepare-context.ts#L41-L124) │
│                                                                      │
│      └─ [Configuration]                                              │
│           context.appInfo = resourceInfo（含 settings + target 信息）│
│           context.serverDirectories =                                │
│             dynamicServerPathCreator(serverPath)                     │
│           context.clientDirectories =                                │
│             dynamicClientPathCreator(adminUIPath)                    │
│           context.plugins = registerPlugins(pluginInstallations)     │
│           → 各代码生成器读取 context 并生成文件                      │
│                                                                      │
│    代码生成完成 → Kafka CODE_GENERATION_SUCCESS_TOPIC / FAILURE_TOPIC│
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 5. 代码生成成功 → 推送 Git（**流程终点**）                             │
│    build.controller.ts                                                │
│    [build.controller.ts:87-101](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.controller.ts#L87-L101) │
│    onCodeGenerationSuccess() →                                       │
│      │                                                               │
│      ├─ saveToGitProvider() [build.service.ts:1130-1342]             │
│      │    → 判断 useDemoRepo or 真实 Git 仓库                        │
│      │    → 组装 CreatePrRequest                                     │
│      │    → Kafka: CREATE_PR_REQUEST_TOPIC                           │
│      │    → Git Provider Service 创建 PR                             │
│      │    → PR 创建完成回调 onCreatePRSuccess()                      │
│      │       → Build.gitStatus = Completed/Failed                    │
│      │                                                               │
│      └─ onCodeGenerationSuccess()                                    │
│           → Kafka: USER_BUILD_TOPIC（通知用户）                      │
│           → Action GENERATE step = Success                           │
│           → Build.status = Completed                                 │
│                                                                      │
│    ❌ **不**：创建 Environment / 创建 Deployment / 部署到沙箱        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、真实的职责边界与已发现的设计缺口

### 6.1 各层实际职责

| 层次 | 实际已实现职责 | 未实现/预留 |
|-----|-------------|----------|
| **Deployment Target** | 选择 NodeJS/DotNET/Blueprint 并校验计费；存储版本策略；构建时解析为镜像名+tag | - |
| **Configuration** | 聚合 ServiceSettings/ResourceSettings/Plugins/Entities 等为 DSGResourceData；拆分 Server/AdminUI job；装配 DSG Context | - |
| **Runtime Binding** | Prisma schema 模型（Environment/Deployment）已定义；GraphQL 查询字段已暴露；权限校验已接入 | **Environment 自动创建、Deployment 记录写入、沙箱容器部署均未实现** |

### 6.2 已发现的设计缺口

#### 缺口 1：Environment 创建逻辑缺失

- **现状**：`EnvironmentService.createDefaultEnvironment()` 存在但无业务调用方
- **风险**：GraphQL `Resource.environments` 字段将永远返回空数组（除非数据库中存在历史/外部写入数据）
- **相关文件**：
  - [environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts)
  - [resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135)

#### 缺口 2：Deployment 完全未实现

- **现状**：模型、DTO、权限校验齐全，但 `prisma.deployment.create()` 零调用
- **风险**：`AuthorizableOriginParameter.DeploymentId` 权限校验路径实际上永不可达
- **相关文件**：
  - [schema.prisma:629-644](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L629-L644)
  - [validation-functions.ts:267-275](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L275)

#### 缺口 3：Build.containerStatusQuery 预留字段未启用

- **现状**：字段存在于 Build 表，无任何读写代码
- **风险**：无直接风险，属预留扩展点
- **相关文件**：
  - [schema.prisma:496-497](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L496-L497)

#### 缺口 4：Deployment Target 与 Configuration 在 resourceInfo 中的耦合

- **现状**：`resourceInfo` 同时承载 `settings`（配置）和 `codeGeneratorVersionOptions` + `codeGeneratorName`（目标），两个不同关注点封装在同一对象
- **影响**：未来增加部署目标维度（如云厂商、region）时 `resourceInfo` 会膨胀
- **相关文件**：
  - [build.service.ts:1501-1516](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts#L1501-L1516)

#### 缺口 5：DEFAULT_ENVIRONMENT_NAME 常量重复定义

- **现状**：在 `resource.service.ts:83` 和 `environment.service.ts:12` 各定义了一份
- **影响**：低风险，可维护性问题
- **相关文件**：
  - [resource.service.ts:83](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L83)
  - [environment.service.ts:12](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts#L12)

---

## 七、各层核心文件索引

| 层次 | 文件 | 关键函数/类 |
|-----|------|------------|
| **Deployment Target** | [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | `createResource`（L333-345 写入 codeGeneratorName）, `getAndValidateCodeGeneratorName`（L415-450）, `getDefaultCodeGenerator`（L455-479）, `updateCodeGeneratorVersion`（L361-413） |
| | [EnumCodeGenerator.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/dto/EnumCodeGenerator.ts) | `EnumCodeGenerator` 枚举 |
| | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | `codeGeneratorNameToContainerImageName`（L424-440）, `runBuild`（L109-159） |
| | [version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator-catalog/src/version/version.service.ts) | `getCodeGeneratorVersion` |
| **Configuration** | [serviceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/serviceSettings/serviceSettings.service.ts) | `getServiceSettingsValues`, `createDefaultServiceSettings`, `updateServiceSettings` |
| | [resourceSettings.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resourceSettings/resourceSettings.service.ts) | `updateResourceSettings`, `validateResourceSettingsProperties` |
| | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts) | `create`（L268-352）, `generate`（L568-618）, `getDSGResourceData`（L1384-1521）, `saveDsgResourceDataToSharedStorage`（L541-560）, `saveToGitProvider`（L1130-1342） |
| | [build.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.controller.ts) | `onCodeGenerationSuccess`（L87-101） |
| | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | `splitBuildsIntoJobs`（L35-91） |
| | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator/src/prepare-context.ts) | `prepareContext`（L41-124） |
| | [dsg-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/data-service-generator/src/dsg-context.ts) | `DsgContext` 类 |
| | [dsg-resource-data.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/libs/util/code-gen-types/src/dsg-resource-data.ts) | `DSGResourceData` 类型 |
| **Runtime Binding（预留）** | [environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts) | `createDefaultEnvironment`（L22-45，仅测试调用）, `getDefaultEnvironment`（无调用） |
| | [resource.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts) | `environments` ResolveField（L130-135，只读查询） |
| | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L615-L644) | `Environment`（L615-627）, `Deployment`（L629-644） 模型, `EnumDeploymentStatus` 枚举 |
| | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L275) | `DeploymentId` 权限校验（仅 count 查询） |
| **进程级 Env** | [server/env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/env.ts) | `Env` 常量 |
| | [build-manager/env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/env.ts) | `Env` 常量 |
