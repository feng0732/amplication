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

## 四、运行环境绑定层 —— 分层完整度核查：权限校验 / 只读查询 / 业务写入

### 4.1 核查结论总览（按维度拆分）

| 维度 | Environment | Deployment | Build.containerStatus* |
|-----|------------|-----------|----------------------|
| **Prisma 数据模型** | ✅ 已定义 [schema.prisma:615-627](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L615-L627) | ✅ 已定义 [schema.prisma:629-644](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L629-L644) | ✅ 已定义 [schema.prisma:496-497](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L496-L497) |
| **权限校验函数** | ✅ 已定义 [validation-functions.ts:445-457](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L445-L457) | ✅ 已定义 [validation-functions.ts:267-286](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L286) | N/A（非独立实体） |
| **权限校验实际触发** | ❌ **无调用方**（无任何 `@AuthorizeContext(EnvironmentId, ...)`） | ❌ **无调用方**（无任何 `@AuthorizeContext(DeploymentId, ...)`） | N/A |
| **GraphQL DTO** | ✅ `Environment`、`CreateEnvironmentArgs`、`EnvironmentWhereInput` 等 [dto/](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/dto/) | ❌ 无任何 Deployment DTO | N/A |
| **GraphQL 查询暴露** | ✅ 通过 `Resource.environments` ResolveField 间接暴露 [resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135) | ❌ **完全未暴露**（无独立 Query/Resolver） | ❌ 未暴露 |
| **GraphQL Mutation 暴露** | ❌ 无 Environment Mutation（DTO 存在但无 Resolver） | ❌ 无 Deployment Mutation | N/A |
| **业务写入逻辑** | ❌ `createDefaultEnvironment()` 仅在 spec 测试中调用 [environment.service.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.spec.ts) | ❌ 整个代码库无 `prisma.deployment.create/update/delete` | ❌ 无任何读写代码 |
| **Service 层能力** | ✅ `EnvironmentService`：`createDefaultEnvironment` / `getDefaultEnvironment` / `findMany` [environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts) | ❌ 无 DeploymentService | N/A |

### 4.2 Environment 权限校验 —— 函数已定义但无触发点

#### 4.2.1 权限校验函数定义

[validation-functions.ts:445-457](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L445-L457)：

```typescript
[AuthorizableOriginParameter.EnvironmentId]: async (
  prisma, originId, workspaceId
) => {
  const matching = await prisma.environment.findFirst(
    checkByResourceParameters(originId, workspaceId)
  );
  return {
    canAccessWorkspace: matching !== null,
    requestedResourceId: matching?.resourceId,
  };
},
```

复用工具函数 `checkByResourceParameters()`（L5-22），校验逻辑为：
```
Environment.id == originId
  AND Environment.resource.deletedAt == null
  AND Environment.resource.project.workspace.id == workspaceId
```
校验通过后返回关联的 `resourceId`，用于后续细粒度资源权限校验。

#### 4.2.2 无实际调用方

全代码库 grep `AuthorizableOriginParameter.EnvironmentId` → 仅在定义处出现。

对比已启用的校验（如 `ResourceId`），启用方式是在 GraphQL resolver 上使用装饰器：
```typescript
@AuthorizeContext(AuthorizableOriginParameter.ResourceId, "where.id")
```

但 Environment 没有任何独立的 Query 或 Mutation resolver，也就没有地方挂载此装饰器。

**结论**：`EnvironmentId` 校验函数是**预埋的基础设施代码**，未来出现独立的 `environment(id: ID!)` Query 时才会被使用，当前版本完全不触发。

### 4.3 Deployment 权限校验 —— 函数已定义但无触发点

#### 4.3.1 权限校验函数定义

[validation-functions.ts:267-286](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L286)：

```typescript
[AuthorizableOriginParameter.DeploymentId]: async (
  prisma, originId, workspaceId
) => {
  const matching = await prisma.deployment.count({
    where: {
      id: originId,
      environment: {
        resource: {
          deletedAt: null,
          project: { workspaceId },
        },
      },
    },
  });
  return { canAccessWorkspace: matching === 1 };
},
```

与 EnvironmentId 不同的是：
- 使用 `count` 而非 `findFirst`（不返回 `resourceId`，因此无法继续做细粒度资源权限校验）
- 只验证 workspace 级访问权限

#### 4.3.2 关联权限：ActionId 校验中的 Deployment 分支

[validation-functions.ts:212-266](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L212-L266) 的 `ActionId` 校验通过 OR 条件同时覆盖了三种场景：

```typescript
OR: [
  { id: originId, deployments: { some: { build: { resource: {...} } } } }, // ← 通过 Deployment 关联
  { id: originId, builds:      { some: { resource: {...} } } },
  { id: originId, userAction:  { some: { resource: {...} } } },
]
```

这是唯一一处在权限校验中**实际读 Deployment 表**的代码（通过 Action.deployments 关联）。但由于 Deployment 表无业务写入，此分支实际上永不为真。

#### 4.3.3 无独立调用方

全代码库 grep `AuthorizableOriginParameter.DeploymentId` → 仅在定义处出现。无任何 resolver 使用 `@AuthorizeContext(DeploymentId, ...)`。

**结论**：`DeploymentId` 校验函数同样是预埋代码，无实际触发路径。

### 4.4 Environment 只读查询边界核查

#### 4.4.1 暴露方式：仅通过 Resource 嵌套字段

[resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135)：

```typescript
@ResolveField(() => [Environment])
async environments(@Parent() resource: Resource): Promise<Environment[]> {
  return this.environmentService.findMany({
    where: { resource: { id: resource.id } },
  });
}
```

关键特征：
- **无独立 Query 端点**（无 `environment(id: ID!)` 或 `environments(where: ...)`）
- **无 `@AuthorizeContext` 装饰器**
- 只能在已解析出 Resource 对象的上下文中被访问

#### 4.4.2 实际权限保护机制

ResourceResolver 类级别有 [@UseGuards(GqlAuthGuard)](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L53)。

`GqlAuthGuard.canActivate()` 的工作流（[gql-auth.guard.ts:31-45](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L31-L45)）：

```
1. super.canActivate() → JWT 认证（所有 resolver 必须先过这关）
2. authorizeContext(handler, requestArgs, user)
   └─ getAuthorizeContextParameters(handler) → 读取 @AuthorizeContext 元数据
      ├─ 有元数据 → permissionsService.validateAccess(...)
      └─ 无元数据 → 直接返回 true（L69-71）
```

对 `environments` ResolveField 的影响：
1. **无独立 JWT 认证和权限校验**（ResolveField 不触发任何 Guard/Interceptor，详见 4.4.4 节 fieldResolverEnhancers 配置核准）
2. **没有独立的 EnvironmentId 级权限校验**（@ResolveField 无 @AuthorizeContext，且 Guard 根本不被调用）
3. **受父级 Resource 查询的间接保护**：要访问 `Resource.environments`，必须先通过顶级 Query 查询到该 Resource。而 `resource()` Query（L68-72）上有 `@AuthorizeContext(AuthorizableOriginParameter.ResourceId, "where.id")`，只有能通过 ResourceId 校验的用户才能拿到父级 Resource 对象，进而访问嵌套的 environments。ResolveField 层面完全依赖父级查询的安全性传递。

#### 4.4.3 查询权限边界总结

| 访问路径 | 是否受保护 | 保护方式 |
|---------|----------|---------|
| 直接按 ID 查询 Environment | ❌ 无法访问 | 无此 Query 端点 |
| `query { resource(id) { environments } }` | ✅ | 父级 Resource 的 `@AuthorizeContext(ResourceId, ...)`（顶级 Query 触发） |
| `@AuthorizeContext(EnvironmentId, ...)` | ❌ 永不触发 | 无挂载点，且 ResolveField 不触发 Guard |
| JWT 认证 | ✅（仅顶级方法） | 类级 `@UseGuards(GqlAuthGuard)` 对 @Query/@Mutation 生效，对 @ResolveField 不生效 |

#### 4.4.4 GraphQLModule 的 fieldResolverEnhancers 配置核准（重大发现）

**关键配置核查**：[app.module.ts:33-65](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/app.module.ts#L33-L65)

```typescript
GraphQLModule.forRootAsync<ApolloDriverConfig>({
  driver: ApolloDriver,
  useFactory: async (configService: ConfigService) => {
    return {
      autoSchemaFile: ...,
      sortSchema: true,
      debug: ...,
      playground: ...,
      introspection: ...,
      context: (context) => { ... },
      subscriptions: { ... },
      // ❌ 完全没有 fieldResolverEnhancers 选项
    };
  },
  ...
});
```

**NestJS GraphQL v10 默认行为（官方文档确认）**：

> In the GraphQL context, Nest does not run **enhancers** (the generic name for interceptors, guards and filters) at the field level. They only run for the top level `@Query()`/`@Mutation()`/`@Subscription()` method. You can tell Nest to execute interceptors, guards or filters for methods annotated with `@ResolveField()` by setting the `fieldResolverEnhancers` option.

**结论**：当前 GraphQLModule **未配置 `fieldResolverEnhancers`**，意味着：

| 增强器类型 | 对 `@Query()`/`@Mutation()`/`@Subscription()` | 对 `@ResolveField()` |
|-----------|----------------------------------------------|---------------------|
| 类级 `@UseGuards(GqlAuthGuard)` | ✅ 生效 | ❌ **完全不触发** |
| 全局 `APP_INTERCEPTOR`（InjectContextInterceptor） | ✅ 生效 | ❌ **完全不触发** |
| 方法级 `@UseGuards(...)` | ✅ 生效 | ❌ **完全不触发** |
| 方法级 `@UseInterceptors(...)` | ✅ 生效 | ❌ **完全不触发** |

**重要修正之前的错误理解**：之前的分析认为"ResolveField 仍然会触发类级 GqlAuthGuard，只是 authorizeContext() 在无 @AuthorizeContext 元数据时直接返回 true"；实际情况是 **GqlAuthGuard 和 InjectContextInterceptor 对 ResolveField 根本不会被调用**，不存在"返回 true"这一步——执行流程根本不进入 Guard/Interceptor 代码。

核查佐证：
- 全代码库 grep `fieldResolverEnhancers` → **零结果**
- 全代码库所有 `@ResolveField` 方法上无任何 `@AuthorizeContext` 或 `@InjectContextValue` 装饰器（multiline grep 零命中）

---

#### 4.4.5 两种权限装饰器的机制差异（结合 fieldResolverEnhancers 现状）

在分析父级资源访问路径之前，必须先明确项目中两种装饰器的本质区别，以及它们在 ResolveField 上的实际生效边界：

| 机制 | `@AuthorizeContext` | `@InjectContextValue`（不带 permissions 参数） |
|-----|--------------------|----------------------------------------------|
| 生效位置 | `GqlAuthGuard.canActivate()`（Guard 阶段） | `InjectContextInterceptor.intercept()`（全局 Interceptor 阶段） |
| 注册方式 | resolver 方法装饰器 + 类级 `@UseGuards(GqlAuthGuard)` | 全局 `APP_INTERCEPTOR`（[app.module.ts:80-83](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/app.module.ts#L80-L83)） |
| 核心行为 | 从请求参数中读出值，用 `VALIDATION_FUNCTIONS[parameterType]` 校验该值是否属于用户 workspace | 把 `user.workspace.id` 或 `user.id` **强制写入**请求参数的指定路径，覆盖用户输入 |
| 权限校验 | ✅ 执行（workspace 归属 + 可选细粒度权限） | ❌ **不执行**。无 permissions 参数时仅做参数注入，`GqlAuthGuard.authorizeContext()` 在 [gql-auth.guard.ts:69-71](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L69-L71) 直接返回 true |
| 对顶级 Query/Mutation | ✅ 触发 | ✅ 触发 |
| **对 @ResolveField（当前配置）** | ❌ **完全不触发** | ❌ **完全不触发** |
| 无装饰器时（顶级方法） | authorizeContext() 返回 true（放行） | 不做任何注入（放行） |

两种装饰器均定义于：
- [authorizeContext.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/decorators/authorizeContext.decorator.ts)
- [injectContextValue.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/decorators/injectContextValue.decorator.ts)

#### 4.4.6 父级资源访问路径与 environments 字段的权限继承链

`Resource.environments` ResolveField（[resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135)）本身无任何装饰器，其保护完全依赖**父级 Resource 对象是通过哪条查询路径获取的**。以下逐一核对 3 条可达路径：

---

**路径 A：`query { resource(id) { environments } }` — 单资源直查**

```typescript
// [resource.resolver.ts:68-72]
@Query(() => Resource, { nullable: true })
@AuthorizeContext(AuthorizableOriginParameter.ResourceId, "where.id")
async resource(@Args() args: FindOneArgs): Promise<Resource | null> {
  return this.resourceService.resource(args);
}
```

保护链路：
1. ✅ JWT 认证（类级 GqlAuthGuard，顶级 Query 触发）
2. ✅ `@AuthorizeContext(ResourceId, "where.id")` — 用 `VALIDATION_FUNCTIONS[ResourceId]` 校验该 Resource 是否属于用户 workspace
3. ✅ Resource 对象通过校验后，才会被 ResolveField 消费
4. `environments` 是 ResolveField → **GqlAuthGuard 和 InjectContextInterceptor 完全不触发**（因无 fieldResolverEnhancers 配置）→ 直接执行 Service 层
5. `environmentService.findMany({ where: { resource: { id: resource.id } } })` — 用已校验的 resourceId 查询

**结论**：✅ 安全。environments 继承了父级 Resource 的完整校验。ResolveField 层面没有二次权限检查是架构设计的结果（依赖父级校验的传递性），而非漏洞。

---

**路径 B：`query { resources(where) { environments } }` — 资源列表查询**

```typescript
// [resource.resolver.ts:74-83]
@Query(() => [Resource], { nullable: false })
@InjectContextValue(
  InjectableOriginParameter.WorkspaceId,
  "where.project.workspace.id"
)
async resources(@Args() args: FindManyResourceArgs): Promise<Resource[]> {
  return this.resourceService.resources(args);
}
```

保护链路：
1. ✅ JWT 认证（类级 GqlAuthGuard，顶级 Query 触发）
2. ⚠️ `@InjectContextValue(WorkspaceId, "where.project.workspace.id")` — **仅注入不校验**：将 `user.workspace.id` 强制写入 args.where.project.workspace.id，覆盖用户可能的越权输入
3. ❌ 无 `@AuthorizeContext` → GqlAuthGuard 不做权限校验
4. `resourceService.resources(args)` → `prepareResourceFindManyArgsForQuery()` 内部（[resource.service.ts:1320-1364](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1320-L1364)）：
   - 仅处理 `serviceTemplateId`、`projectIdFilter`、`properties` JSON 过滤
   - **强制追加** `deletedAt: null` 和 `archived: { not: true }`
   - **不再追加 workspace 过滤**（完全依赖 InjectContextInterceptor 已注入的 `where.project.workspace.id`）
5. ResolveField `environments` 对每个返回的 Resource 逐个执行 → **GqlAuthGuard 和 InjectContextInterceptor 完全不触发**（因无 fieldResolverEnhancers 配置）→ 直接执行 Service 层

**结论**：✅ 安全，但保护机制不同。不是"逐条校验归属"，而是"强制在查询条件中注入 workspace.id 限定范围"。`prepareResourceFindManyArgsForQuery()` 中的注释（L1329）显式说明了这一依赖："workspace.id is expected to be injected in the resolver middleware."

**脆弱点**：如果内部调用方绕过 resolver 直接调用 `resourceService.resources()` 且忘记传入 `where.project.workspace.id`，将导致越权（当前代码库中 service 内部确实存在多处绕过调用，但均传入了特定的 projectId 等限定条件，未发现越权漏洞）。

---

**路径 C：`query { project(id) { resources { environments } } }` — Project 嵌套 → Resource 列表 → environments**

```typescript
// [project.resolver.ts:53-57]  — 父级 Project 查询
@Query(() => Project, { nullable: true })
@AuthorizeContext(AuthorizableOriginParameter.ProjectId, "where.id")
async project(@Args() args: FindOneArgs): Promise<Project | null> {
  return this.projectService.findUnique(args);
}

// [project.resolver.ts:103-108]  — Project.resources 嵌套字段
@ResolveField(() => [Resource])
async resources(@Parent() project: Project): Promise<Resource[]> {
  return this.resourceService.resources({
    where: { project: { id: project.id } },
  });
}
```

保护链路（关键差异点）：
1. ✅ JWT 认证（类级 GqlAuthGuard，顶级 Query 触发）
2. ✅ `@AuthorizeContext(ProjectId, "where.id")` — 父级 Project 通过 workspace 归属校验
3. ⚠️ `Project.resources` ResolveField：**GqlAuthGuard 和 InjectContextInterceptor 完全不触发**（因无 fieldResolverEnhancers 配置）
   - 不存在"authorizeContext() 返回 true"这一步——Guard 根本不被调用
   - 不存在"InjectContextInterceptor 不注入"这一步——Interceptor 根本不被调用
4. `resourceService.resources({ where: { project: { id: project.id } } })`：
   - 传入的 args 中**没有** `where.project.workspace.id`
   - `prepareResourceFindManyArgsForQuery()` 不会补加 workspace 过滤（只追加 deletedAt 和 archived）
   - **最终 Prisma 查询条件仅为** `WHERE project.id = <已校验的projectId> AND deletedAt IS NULL AND archived != true`
5. 每个 Resource 上的 `environments` ResolveField：同样，**GqlAuthGuard 和 InjectContextInterceptor 完全不触发** → 直接执行 Service 层

**结论**：✅ 在正常调用链下安全。保护依赖"父级 Project 已通过 ProjectId 归属校验"→ project.id 可信 → 用此 id 查询出的 Resource 列表自然属于同一 workspace。但与路径 B 不同，此路径**没有在 Service 层二次注入 workspace.id 作为防御纵深**。由于 ResolveField 根本不触发 Guard/Interceptor，任何对 ResolveField 级别的保护（如未来在 Project.resources 上加 @AuthorizeContext）都需要先启用 fieldResolverEnhancers 配置才能生效。

---

**路径 D（不存在）：`query { environment(id) { ... } }`**
- 无此 Query 端点，完全不可达。

---

三条可达路径的权限保护对比：

| 维度 | 路径 A：resource(id) | 路径 B：resources(where) | 路径 C：project(id).resources |
|-----|---------------------|------------------------|------------------------------|
| 装饰器类型 | @AuthorizeContext | @InjectContextValue | 父级 @AuthorizeContext + ResolveField 无装饰器 |
| 校验粒度 | 单条 Resource 逐条校验 | 批量（SQL WHERE 注入 workspace.id） | 仅父级 Project 单条校验 |
| 权限校验函数 | VALIDATION_FUNCTIONS[ResourceId] | 无（依赖参数注入） | VALIDATION_FUNCTIONS[ProjectId] |
| Service 层防御纵深 | findUnique 直接按 ID 查 | prepareResourceFindManyArgsForQuery 补 deletedAt/archived（不补 workspace） | prepareResourceFindManyArgsForQuery 补 deletedAt/archived（不补 workspace） |
| environments 保护强度 | ✅ 最强（直接继承 ResourceId 校验） | ✅ 强（继承 workspace 范围注入） | ⚠️ 中等（依赖父级 Project 校验的传递性，无二次校验） |

#### 4.4.7 Service 层内部绕过 resolver 调用 resources() 的情况

`resourceService.resources()` 在代码库中被大量内部调用（grep 命中 10+ 处）。这些内部调用均绕过 `@InjectContextValue` 装饰器（装饰器只对 Nest 路由入口生效）。核查代表性调用：

| 调用位置 | 传入 where 条件 | 是否安全 |
|---------|---------------|---------|
| [resource.service.ts:230](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L230) | `project.id + resourceType` | ✅ 限定 project |
| [resource.service.ts:749](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L749) | `project.id + name` | ✅ 限定 project |
| [resource.service.ts:1385](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1385) | （resolver 入口，走装饰器） | ✅ 见路径 B |
| [build.service.ts:1441](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts#L1441) | `project.id` | ✅ 限定 project |
| [project.resolver.ts:105](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/project/project.resolver.ts#L105) | `project.id`（来自已校验的 @Parent Project） | ✅ 见路径 C |

所有内部调用均传入了 `project.id` 或其他限定条件，未发现无条件全表扫描调用。

### 4.5 Environment/Deployment 业务写入缺失的核查

#### 4.5.1 Environment 写入核查

| 写入能力 | 是否存在 | 调用方 |
|---------|---------|-------|
| `EnvironmentService.createDefaultEnvironment()` | ✅ 实现 | ❌ 仅 `environment.service.spec.ts` |
| `PrismaService.environment.create()` | ✅（Prisma 自动生成） | ❌ 业务代码零调用 |
| Resource 创建流程 | - | ❌ `createService()` → `createResource()` 流程完全不涉及 Environment |
| `CreateEnvironmentArgs` DTO | ✅ 已定义 | ❌ 无任何 Mutation resolver 消费此 DTO |

Resource 创建完整链路（[resource.service.ts:592-627](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L592-L627)）：

```
createService(args, user)
  ├─ createResource(...)           // 仅创建 Resource 记录 + codeGeneratorName
  ├─ createServiceDefaultObjects()
  │    ├─ 创建 "user" ResourceRole
  │    ├─ 条件创建 User Entity
  │    └─ serviceSettingsService.createDefaultServiceSettings()
  └─ billingService.reportUsage()  // 上报用量
```

**无任何 Environment 创建步骤。**

#### 4.5.2 Deployment 写入核查

| 写入能力 | 是否存在 | 调用方 |
|---------|---------|-------|
| DeploymentService | ❌ 不存在 | - |
| `PrismaService.deployment.create/update/delete` | ✅（Prisma 自动生成） | ❌ 业务代码零调用 |
| Build 完成流程 | - | ❌ `onCodeGenerationSuccess()` 止于 Git PR 推送 + USER_BUILD_TOPIC |
| `sendDeploymentNotification()` 邮件 | ✅ 实现 | ❌ `IS_EMAIL_DEPLOYMENT_NOTIFICATION = false` 永久屏蔽 + 无调用方 |

Build 完成完整链路（[build.controller.ts:87-101](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.controller.ts#L87-L101)）：

```
onCodeGenerationSuccess(message)
  ├─ buildService.saveToGitProvider(buildId)    // 创建 Git PR
  └─ buildService.onCodeGenerationSuccess(buildId)
       ├─ kafka.emit(USER_BUILD_TOPIC, ...)     // 通知用户
       └─ actionService.complete(step, Success) // 标记 Action 步骤完成
```

**无任何 Deployment 创建步骤。**

#### 4.5.3 Build.containerStatusQuery 字段核查

[schema.prisma:496-497](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L496-L497) 定义了 `containerStatusQuery Json?` 和 `containerStatusUpdatedAt DateTime?`，全代码库 grep 无任何读写代码。

**结论**：预留字段，用于未来存储沙箱容器状态查询参数（如 Kubernetes deployment name、namespace 等），当前未启用。

### 4.6 权限校验、只读查询、业务写入三者的关系架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│  A. 权限基础设施层（全部已就绪，预留型）                              │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ AuthorizableOriginParameter 枚举                              │  │
│  │   ├─ EnvironmentId  ✅ 已定义                                  │  │
│  │   └─ DeploymentId   ✅ 已定义                                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ VALIDATION_FUNCTIONS（validation-functions.ts）              │  │
│  │   ├─ [EnvironmentId]  ✅ 已实现  ❌ 无任何 @AuthorizeContext 触发│  │
│  │   └─ [DeploymentId]   ✅ 已实现  ❌ 无任何 @AuthorizeContext 触发│  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ActionId 校验中包含 deployments 关联 OR 分支                   │  │
│  │   ✅ 已实现  ❌ Deployment 表无数据，实际永不匹配               │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 依赖（但未被消费）
                              │
┌──────────────────────────────────────────────────────────────────────┐
│  B. GraphQL 暴露层（部分已就绪）                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Environment DTO 集合                                          │  │
│  │   Environment / EnvironmentCreateInput / CreateEnvironmentArgs│  │
│  │   EnvironmentWhereInput / EnvironmentOrderByInput / ...       │  │
│  │   ✅ 全部已定义  ❌ 仅 ResolveField 消费，无独立 Query/Mutation │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ ResourceResolver.environments ResolveField                   │  │
│  │   ✅ 已暴露  ❌ 无 @AuthorizeContext 装饰器                    │  │
│  │   ⚠️  仅靠父级 Resource 查询的鉴权间接保护                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Deployment GraphQL 层                                         │  │
│  │   ❌ 完全不存在（无 DTO、无 Query、无 Mutation、无 Resolver）  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 依赖（但未被消费）
                              │
┌──────────────────────────────────────────────────────────────────────┐
│  C. 业务写入层（完全缺失）                                            │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ EnvironmentService.createDefaultEnvironment()                │  │
│  │   ✅ 已实现  ❌ Resource 创建流程零调用                        │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Deployment 写入                                              │  │
│  │   ❌ 无 Service、无 Prisma 调用、Build 完成流程零写入          │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Build.containerStatusQuery / containerStatusUpdatedAt        │  │
│  │   ✅ 字段已定义  ❌ 无任何读写代码                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.7 设计特征：典型的"自上而下、由外向内"预留架构

从完整度分布（A 层 100% → B 层 50% → C 层 0%）可以看出，这是典型的**自上而下预留开发模式**：

1. **第 1 步**：数据模型先行（Prisma schema 全部就位）
2. **第 2 步**：权限基础设施就位（AuthorizableOriginParameter + VALIDATION_FUNCTIONS 全部就位）
3. **第 3 步**：DTO 类型定义就位（Environment DTO 全部就绪）
4. **第 4 步**：Service 层骨架就位（EnvironmentService 基本方法就绪）
5. **第 5 步**：GraphQL 只读端点就位（Resource.environments ResolveField 暴露）
6. **第 6 步（未完成）**：独立 Query/Mutation 端点 + 业务写入逻辑

当前代码停留在**第 5 步刚完成、第 6 步未开始**的状态。运行环境绑定功能在权限系统、类型系统、查询路径上已经"预留了位置"，但真正的业务操作（创建 Environment、写入 Deployment、部署到沙箱、查询容器状态）尚未开发。

### 4.8 运行环境绑定层的真实职责

根据代码核查，当前版本中运行环境绑定层的**实际可用职责**极其有限：

| 可用能力 | 说明 |
|---------|------|
| 数据库中存储 Environment 行 | ✅ 模型存在，若由外部/历史数据写入则可读 |
| 通过 `Resource.environments` 嵌套查询返回 Environment | ✅ 受 Resource 级权限间接保护，返回空数组或已有数据 |
| 按 EnvironmentId/DeploymentId 做权限校验 | ❌ 无端点触发，函数存在但不可达 |
| 自动创建默认 Sandbox Environment | ❌ 业务流程零调用 |
| 记录 Build 部署结果 | ❌ Deployment 表零写入 |
| 查询沙箱容器运行状态 | ❌ containerStatusQuery 字段零读写 |
| 发送部署通知邮件 | ❌ 永久屏蔽 + 零调用 |

**一句话总结**：运行环境绑定层当前仅具备"读已有数据"的骨架能力，不具备任何"写数据/触发操作"的业务能力。

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

### 6.2 已发现的设计缺口与边界不一致点

#### 缺口 1：Environment 写入链路完整度缺失（Service 层就绪 → 业务流程未接入）

- **现状**：数据模型 ✅ / Service 层 `createDefaultEnvironment()` ✅ / DTO `CreateEnvironmentArgs` ✅，但 Resource 创建流程（`createService()` → `createResource()`）中完全不调用 Environment 创建，也无任何 GraphQL Mutation 消费 `CreateEnvironmentArgs`
- **风险**：GraphQL `Resource.environments` 字段将永远返回空数组（除非数据库中存在历史/外部写入数据）；Service 层方法永远不被调用成为死代码
- **相关文件**：
  - [environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts)
  - [resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135)
  - [resource.service.ts:592-627](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L592-L627)

#### 缺口 2：Deployment 完整度更低（模型与权限就绪 → DTO/Resolver/Service 全缺失）

- **现状**：与 Environment 相比，Deployment 缺失层级更多：Prisma 模型 ✅ / 权限校验函数 ✅，但 **DTO 层 ❌ / Service 层 ❌ / GraphQL 层 ❌ / 业务写入 ❌**。Deployment 甚至没有独立的 DTO 类
- **风险**：
  - `AuthorizableOriginParameter.DeploymentId` 权限校验路径永不可达（无 resolver 挂载 `@AuthorizeContext`）
  - `ActionId` 校验中的 `deployments` OR 分支（[validation-functions.ts:221-234](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L221-L234)）永不匹配，死分支
- **相关文件**：
  - [schema.prisma:629-644](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L629-L644)
  - [validation-functions.ts:267-286](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L286)

#### 缺口 3：权限校验函数与 GraphQL 端点不匹配（函数已定义 → 无触发点）

- **现状**：`EnvironmentId` 和 `DeploymentId` 在 [AuthorizableOriginParameter](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts#L19-L20) 枚举中已定义，对应的 `VALIDATION_FUNCTIONS` 校验函数也已实现，但全代码库无任何 resolver 使用 `@AuthorizeContext(AuthorizableOriginParameter.EnvironmentId, ...)` 或 `@AuthorizeContext(AuthorizableOriginParameter.DeploymentId, ...)`
- **风险**：校验函数成为死代码；未来新增端点时容易遗漏权限装饰器（因为缺乏现有示例）
- **相关文件**：
  - [validation-functions.ts:267-286](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L267-L286) （DeploymentId）
  - [validation-functions.ts:445-457](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L445-L457) （EnvironmentId）
  - [gql-auth.guard.ts:61-83](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts#L61-L83) （触发机制）

#### 缺口 4：Resource.environments 缺少独立的 @AuthorizeContext 装饰器，且即使添加也不会生效

- **现状**：`environments` ResolveField（[resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135)）没有 `@AuthorizeContext` 装饰器，权限完全依赖父级 Resource 查询的保护。与同 resolver 中的其他 ResolveField（如 `entities`、`builds`）保持一致
- **更严重的问题**：即便给 `environments` ResolveField 加上 `@AuthorizeContext(EnvironmentId, ...)`，**在当前配置下也不会生效**——因为 GraphQLModule 未配置 `fieldResolverEnhancers`，所有 ResolveField 都不会触发 Guard（详见缺口 12）。这意味着如果未来要给任何 ResolveField 加独立权限校验，必须先修改 GraphQLModule 配置
- **边界情况**：若未来新增独立的 `environment(id: ID!)` Query，必须同时补上 `@AuthorizeContext(AuthorizableOriginParameter.EnvironmentId, "where.id")`，否则将形成越权漏洞
- **相关文件**：
  - [resource.resolver.ts:130-135](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L130-L135)
  - [app.module.ts:33-65](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/app.module.ts#L33-L65) （fieldResolverEnhancers 未配置）

#### 缺口 5：ActionId 权限校验的 Deployment 关联分支为死代码

- **现状**：ActionId 校验的 OR 条件中第一个分支通过 `Action.deployments` 关联来验证归属（[validation-functions.ts:221-234](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L221-L234)）。但 Deployment 表零业务写入，`Action.deployments` 关联永远为空，此分支永不匹配
- **影响**：无直接安全风险（其他分支仍能校验），但增加了查询复杂度（每次 ActionId 校验都会 JOIN 一个空表），且容易误导后续开发者认为 Deployment 功能已上线
- **相关文件**：
  - [validation-functions.ts:212-266](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts#L212-L266)

#### 缺口 6：Build.containerStatusQuery 预留字段未启用

- **现状**：`containerStatusQuery Json?` 和 `containerStatusUpdatedAt DateTime?` 字段存在于 Build 表，全代码库无任何读写代码
- **风险**：无直接风险，属预留扩展点。字段命名暗示将用于存储沙箱容器的查询参数（如 Kubernetes deployment name、namespace）
- **相关文件**：
  - [schema.prisma:496-497](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L496-L497)

#### 缺口 7：Deployment Target 与 Configuration 在 resourceInfo 中的耦合

- **现状**：`resourceInfo` 同时承载 `settings`（配置）和 `codeGeneratorVersionOptions` + `codeGeneratorName`（目标），两个不同关注点封装在同一对象
- **影响**：未来增加部署目标维度（如云厂商、region）时 `resourceInfo` 会膨胀
- **相关文件**：
  - [build.service.ts:1501-1516](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/build/build.service.ts#L1501-L1516)

#### 缺口 8：三条父级 Resource 查询路径的权限保护机制不一致

- **现状**：获取 Resource 对象有 3 条路径，使用了 3 种不同的保护模式：
  - 路径 A `resource(id)`：用 `@AuthorizeContext(ResourceId, ...)` **逐条校验归属**
  - 路径 B `resources(where)`：用 `@InjectContextValue(WorkspaceId, ...)` **强制注入 workspace 范围**（不逐条校验）
  - 路径 C `project(id).resources`：父级用 `@AuthorizeContext(ProjectId, ...)` 校验后，ResolveField **完全无装饰器**，仅用 project.id 过滤
- **风险**：三种机制语义不同，后续开发者新增路径时容易选错或遗漏。路径 C 尤其脆弱，完全依赖父级校验的传递性
- **相关文件**：
  - [resource.resolver.ts:68-83](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts#L68-L83)
  - [project.resolver.ts:103-108](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/project/project.resolver.ts#L103-L108)

#### 缺口 9：Project.resources ResolveField 缺少防御纵深（Guard/Interceptor 不触发 + 无 workspace.id 二次注入）

- **现状**：`Project.resources` ResolveField（[project.resolver.ts:103-108](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/project/project.resolver.ts#L103-L108)）调用 `resourceService.resources({ where: { project: { id: project.id } } })`，存在两层缺失：
  1. **Guard/Interceptor 完全不触发**：因 GraphQLModule 未配置 `fieldResolverEnhancers`，类级 `@UseGuards(GqlAuthGuard)` 和全局 `APP_INTERCEPTOR` 对此 ResolveField 无效
  2. **Service 层不兜底**：`prepareResourceFindManyArgsForQuery()` 不补加 workspace 过滤（只补 deletedAt/archived），最终 SQL 仅靠 `project.id = <值>` 限定范围
- **风险**：在正常调用链下 project.id 来自已校验的父级 Project 对象，暂安全。但若未来出现可绕过父级查询直接调用此 ResolveField 且传入任意 project 对象的场景，将存在越权风险。与路径 B `resources(where)` Query 相比缺少两层防御纵深
- **相关文件**：
  - [project.resolver.ts:103-108](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/project/project.resolver.ts#L103-L108)
  - [app.module.ts:33-65](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/app.module.ts#L33-L65) （fieldResolverEnhancers 未配置）
  - [resource.service.ts:1320-1364](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1320-L1364) （`prepareResourceFindManyArgsForQuery` 不补 workspace 过滤）

#### 缺口 10：resourceService.resources() 完全依赖调用方传入 workspace.id，Service 层自身不兜底

- **现状**：`prepareResourceFindManyArgsForQuery()`（[resource.service.ts:1320-1364](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1320-L1364)）内部只处理 `serviceTemplateId`、`projectIdFilter`、`properties` 过滤，并强制追加 `deletedAt: null` 和 `archived: { not: true }`，但**不追加 workspace 限定**。注释 L1329 显式说明"workspace.id is expected to be injected in the resolver middleware"。Service 层内部 10+ 处调用均绕过装饰器，但传入了 project.id 等其他限定条件，暂未发现漏洞
- **风险**：后续新增内部调用若忘记传入任何限定条件，将导致越权查询。Service 层自身不做 workspace 兜底是一种架构选择，但属于隐性契约，容易被打破
- **相关文件**：
  - [resource.service.ts:1320-1364](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L1320-L1364)

#### 缺口 11：DEFAULT_ENVIRONMENT_NAME 常量重复定义

- **现状**：在 `resource.service.ts:83` 和 `environment.service.ts:12` 各定义了一份
- **影响**：低风险，可维护性问题
- **相关文件**：
  - [resource.service.ts:83](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts#L83)
  - [environment.service.ts:12](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts#L12)

#### 缺口 12（架构级）：GraphQLModule 未配置 fieldResolverEnhancers，所有 ResolveField 都不触发 Guard 和 Interceptor

- **现状**：[app.module.ts:33-65](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/app.module.ts#L33-L65) 的 GraphQLModule.forRootAsync 配置中**完全没有 `fieldResolverEnhancers` 选项**。根据 NestJS GraphQL v10 官方文档，默认情况下 enhancers（guards、interceptors、filters）**只在顶级 @Query/@Mutation/@Subscription 上运行，不在 @ResolveField 上运行**
- **实际影响**：
  1. 全代码库 20+ 个 Resolver 类上的类级 `@UseGuards(GqlAuthGuard)` **对 ResolveField 完全无效**
  2. 全局注册的 `APP_INTERCEPTOR`（InjectContextInterceptor、AnalyticsSessionIdInterceptor）**对 ResolveField 完全无效**
  3. 任何在 @ResolveField 方法上添加的 @AuthorizeContext、@InjectContextValue、@UseGuards、@UseInterceptors **当前都不会生效**
  4. 影响所有嵌套字段查询：`project(id) { resources }`、`resource(id) { environments }`、`resource(id) { builds }`、`resource(id) { entities }`、`workspace(id) { projects }` 等等，全依赖父级顶级查询的安全性传递
- **风险评估**：
  - 当前无直接越权漏洞（所有 ResolveField 都通过 @Parent() 使用已校验父级对象的 id 做限定查询）
  - 但这是一种"隐性安全契约"，后续开发者若新增 ResolveField 忘记用父级 id 限定，或假设"类级 Guard 会兜底"，将引入越权漏洞
  - 若未来需要在 ResolveField 层面做独立权限校验，必须**先启用 fieldResolverEnhancers**，否则加了装饰器也白加
- **启用选项**（若未来修复）：在 GraphQLModule.forRoot 配置中添加 `fieldResolverEnhancers: ['guards', 'interceptors', 'filters']`。官方文档警告："Enabling enhancers for field resolvers can cause performance issues as they will be executed for each field in your query"
- **相关文件**：
  - [app.module.ts:33-65](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/app.module.ts#L33-L65)
  - [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) （类级装饰器注册 20+ 处）
  - [inject-context.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/interceptors/inject-context.interceptor.ts) （全局 APP_INTERCEPTOR 注册）

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
| **Runtime Binding（预留）** | [environment.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/environment.service.ts) | `createDefaultEnvironment`（L22-45，仅测试调用）, `getDefaultEnvironment`（无调用）, `findMany` |
| | [environment/dto/](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/environment/dto/) | `Environment` DTO、`CreateEnvironmentArgs`、`EnvironmentWhereInput`、`EnvironmentCreateInput` 等（全部已定义但无独立 Mutation 消费） |
| | [resource.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.resolver.ts) | `resource(id)` Query（L68-72，@AuthorizeContext ResourceId）, `resources(where)` Query（L74-83，@InjectContextValue WorkspaceId）, `environments` ResolveField（L130-135，只读查询，无独立 @AuthorizeContext） |
| | [project.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/project/project.resolver.ts) | `project(id)` Query（L53-57，@AuthorizeContext ProjectId）, `resources` ResolveField（L103-108，无装饰器，直接用 project.id 查 Resource） |
| | [resource.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/resource/resource.service.ts) | `resources()`（L1385-1389）, `prepareResourceFindManyArgsForQuery()`（L1320-1364，补 deletedAt/archived，不补 workspace 过滤） |
| | [authorizeContext.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/decorators/authorizeContext.decorator.ts) | `@AuthorizeContext` 装饰器定义（设置 AUTHORIZE_CONTEXT metadata） |
| | [injectContextValue.decorator.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/decorators/injectContextValue.decorator.ts) | `@InjectContextValue` 装饰器定义（设置 INJECT_CONTEXT_VALUE metadata，可选 permissions） |
| | [inject-context.interceptor.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/interceptors/inject-context.interceptor.ts) | `InjectContextInterceptor`（全局 APP_INTERCEPTOR，强制注入 user.workspace.id / user.id） |
| | [InjectableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/enums/InjectableOriginParameter.ts) | `UserId`、`WorkspaceId`（仅两种可注入参数类型） |
| | [app.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/app.module.ts) | InjectContextInterceptor 注册为全局 APP_INTERCEPTOR（L80-83）; **GraphQLModule 未配置 fieldResolverEnhancers（L33-65）—— 所有 ResolveField 不触发 Guard 和 Interceptor** |
| | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-prisma-db/prisma/schema.prisma#L496-L644) | `Build.containerStatusQuery/UpdatedAt`（L496-497，预留未启用）, `Environment`（L615-627）, `Deployment`（L629-644）, `EnumDeploymentStatus` 枚举 |
| | [AuthorizableOriginParameter.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/enums/AuthorizableOriginParameter.ts) | `ResourceId`（L10）、`ProjectId`（L26）、`EnvironmentId`（L19）、`DeploymentId`（L20）枚举定义 |
| | [validation-functions.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/permissions/validation-functions.ts) | `ResourceId` 校验（L161-182）, `ProjectId` 校验（L78-96）, `DeploymentId` 校验（L267-286，仅 count，无调用方）, `ActionId` 校验含 deployments OR 分支（L212-266，死分支）, `EnvironmentId` 校验（L445-457，无调用方） |
| | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) | `canActivate`（L31-45）, `authorizeContext`（L61-83）—— 权限校验触发机制，无装饰器时直接放行（L69-71） |
| | [mail.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/core/mail/mail.service.ts) | `sendDeploymentNotification`（L55-82，被 `IS_EMAIL_DEPLOYMENT_NOTIFICATION=false` 永久屏蔽，零调用） |
| **进程级 Env** | [server/env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-server/src/env.ts) | `Env` 常量 |
| | [build-manager/env.ts](file:///d:/fz/0601/solo-dogfeeding/code/108-amplication/packages/amplication-build-manager/src/env.ts) | `Env` 常量 |
