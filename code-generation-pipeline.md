# Amplication 代码生成链路分析

## 概述

Amplication 的代码生成链路是一个从**建模数据**到**最终产物**的多阶段流水线，涉及输入组织、模板拼装、产物生成和失败回退四个核心环节。本文档详细分析各阶段的衔接机制、关键数据结构和核心算法。

---

## 一、输入组织：服务端如何汇总建模数据成 DSGResourceData

### 1.1 整体数据流转图

```
用户建模 (Server/UI)
       ↓
GraphQL API → 数据库持久化 (Entity, Field, Role, Plugin 等表)
       ↓
用户点击"构建"
       ↓
[amplication-server] GraphQL Mutation: createBuild()
       ↓
BuildService.create(args: CreateBuildArgs)  [build.service.ts#L268-L352]
       ├─ 提取 resourceId 和 userId
       ├─ 取 commitId 后 8 位作为 version
       ├─ 获取最新实体版本
       ├─ 写入数据库 build 表 (status=Running)
       ├─ 关联 action 和 step
       ├─ 检查资源类型 (仅 Service/Component)
       │
       └─ 检查是否有私有插件
              ├─ 有私有插件 → downloadPrivatePlugins()  [异步流程]
              │        ├─ 发送 Kafka: DOWNLOAD_PRIVATE_PLUGINS_REQUEST
              │        └─ 等待 Kafka 回调: DOWNLOAD_PRIVATE_PLUGINS_SUCCESS
              │                ↓
              │        onDownloadPrivatePluginSuccess()
              │                ↓
              │        generate(logger, build, user)
              │
              └─ 无私有插件 → 直接调用 generate(logger, build, user)
                                ↓
generate()  [build.service.ts#L568-L618]
       ├─ 包装在 actionService.run() 中（step 管理）
       ├─ getDSGResourceData()  ←──────┐
       │    ├─ getOrderedEntities()    │
       │    ├─ getResourceRoles()     │ 14 个数据
       │    ├─ getOrderedPluginInstallations() │ 源
       │    ├─ moduleActionService.findMany()  │
       │    ├─ moduleDtoService.findMany()     │
       │    ├─ moduleService.findMany()        │
       │    ├─ resourceService.getRelations()   │
       │    ├─ resourceSettingsService.getResourceSettingsBlock()
       │    ├─ serviceSettingsService.getServiceSettingsValues()
       │    ├─ topicService.findMany()
       │    ├─ serviceTopicsService.findMany()
       │    └─ getRelatedResourcesRecursive() → 递归调用 getDSGResourceData()
       ├─ 移除敏感字段: omitDeep(dsgResourceData, DSG_RESOURCE_DATA_PROPERTIES_TO_REMOVE)
       ├─ saveDsgResourceDataToSharedStorage() → /dsg-resource-data/{buildId}/resource-data.json
       └─ 发送 Kafka: CODE_GENERATION_REQUEST_TOPIC (resourceId + buildId)
                                ↓
[amplication-build-manager]
       ↓
BuildRunnerController.onCodeGenerationRequest()  ← Kafka 事件触发
       ↓
BuildRunnerService.runBuild(resourceId, buildId)
       ├─ readDsgResourceDataFromSharedStorage()  读取 DSGResourceData
       ├─ codeGeneratorNameToContainerImageName()  转换代码生成器名称
       ├─ codeGeneratorService.getCodeGeneratorVersion()  ← 调用 Catalog Service
       │   └─ POST /api/versions/code-generator-version  版本解析
       ├─ emitCodeGenerationNotifyVersion()  通知版本选择结果
       │
       ├─ buildJobsHandlerService.splitBuildsIntoJobs()  ← 检查 3 个条件
       │   ├─ 条件1: resourceType === Service ?
       │   ├─ 条件2: codeGeneratorVersion !== "latest-local" ?
       │   └─ 条件3: version >= FEATURE_SPLIT_JOBS_MIN_DSG_VERSION ?
       │
       └─ 对每个 Job: runJob()
            ├─ saveDsgResourceData()  写入 jobs/{jobBuildId}/resource-data.json
            ├─ saveRelevantDsgAssets()  ←──────┐  复制 dsg-assets
            │   └─ {DSG_ASSETS_FOLDER}/{resourceId}-{plainBuildId}/
            │      → {DSG_JOBS_BASE_FOLDER}/{jobBuildId}/dsg-assets/
            │
            └─ POST {DSG_RUNNER_URL}  ← 触发 Argo Workflow
                 ├─ resourceId
                 ├─ buildId: jobBuildId (带后缀)
                 ├─ codeGeneratorVersion (镜像 tag)
                 └─ codeGeneratorName (镜像名称)
                      ↓
DSG Runner (Argo Workflow) → 启动 Docker 容器
       ↓
data-service-generator/src/main.ts → generateCode()
```

### 1.2 DSGResourceData 的构建过程

**核心构建函数** - [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L1384-L1521)

```typescript
async getDSGResourceData(
  resource: Resource,
  buildId: string,
  buildVersion: string,
  user: User,
  rootGeneration = true
): Promise<CodeGenTypes.DSGResourceData>
```

**14 个数据源的汇总过程**：

| 数据字段 | 数据源获取方式 | 说明 |
|---------|--------------|------|
| **entities** | `getOrderedEntities(buildId)` | 按创建时间排序的实体列表，包含字段 |
| **roles** | `getResourceRoles(resourceId)` | 资源的角色定义 |
| **pluginInstallations** | `pluginInstallationService.getOrderedPluginInstallations()` | 按顺序的已启用插件 |
| **moduleContainers** | `moduleService.findMany({ where: { resource: { id: resourceId } } })` | 模块容器（自定义模块） |
| **moduleActions** | `moduleActionService.findMany({ where: { resource: { id: resourceId } } })` | 模块动作（自定义 API） |
| **moduleDtos** | `moduleDtoService.findMany({ where: { resource: { id: resourceId } } })` | 自定义 DTO 定义 |
| **relations** | `resourceService.getRelations(resourceId)` | 实体间关系定义 |
| **resourceSettings** | `resourceSettingsService.getResourceSettingsBlock()` | 资源级别设置 |
| **serviceSettings** | `serviceSettingsService.getServiceSettingsValues()` | 服务级设置（仅 Service 类型） |
| **topics** | `topicService.findMany({ where: { resource: { id: resourceId } } })` | 消息主题 |
| **serviceTopics** | `serviceTopicsService.findMany({ where: { resource: { id: resourceId } } })` | 服务消息主题映射 |
| **resourceType** | `resource.resourceType` | 资源类型（Service/MessageBroker 等） |
| **resourceInfo** | 从 resource 对象组装 | 包含名称、描述、版本、URL、设置等 |
| **otherResources** | 递归调用 `getDSGResourceData()` | 关联资源的数据（如微服务架构） |

**关键特性**：

1. **实体排序保证** - [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L1349-L1371)
   ```typescript
   private async getOrderedEntities(buildId: string): Promise<CodeGenTypes.Entity[]> {
     const entities = await this.entityService.getEntitiesByVersions({...});
     return orderBy(entities.map((entity) => {
       return {
         ...entity,
         fields: orderBy(entity.fields, (field) => field.name),
       };
     }), (entity) => entity.createdAt);
   }
   ```
   - 实体按 `createdAt` 升序排列
   - 字段按 `name` 字母顺序排列
   - **目的**：确保代码生成的确定性，避免不必要的变更

2. **递归关联资源**
   - `rootGeneration = true` 时，递归获取所有关联资源的 DSGResourceData
   - 支持微服务架构中的跨服务引用
   - `rootGeneration = false` 时，不包含实体数据（避免循环）

3. **敏感数据过滤**
   ```typescript
   return omitDeep(dsgResourceData, DSG_RESOURCE_DATA_PROPERTIES_TO_REMOVE);
   ```
   - 构建前移除敏感字段
   - 确保不泄露内部数据

### 1.3 共享存储与消息解耦

**保存到共享存储** - [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L541-L560)

```typescript
async saveDsgResourceDataToSharedStorage(
  buildId: string,
  dsgResourceData: CodeGenTypes.DSGResourceData
): Promise<void> {
  const savePath = join(
    this.configService.get(Env.DSG_RESOURCE_DATA_BASE_FOLDER) ||
      "/amplication-data/dsg-resource-data",
    buildId,
    this.configService.get(Env.DSG_RESOURCE_DATA_FILE) || "resource-data.json"
  );
  await fs.writeFile(savePath, JSON.stringify(dsgResourceData));
}
```

**Kafka 消息轻量化**：
- 消息体只包含 `resourceId` 和 `buildId`
- 不传递完整的 DSGResourceData（可能几 MB 大小）
- 通过文件系统共享大数据
- 构建管理器通过 `readDsgResourceDataFromSharedStorage()` 读取

---

### 1.4 服务端构建创建的完整调用链

**GraphQL 入口 → createBuild() → generate()**

**1. GraphQL Mutation 入口**

用户点击"构建"按钮后，通过 GraphQL Mutation 调用 `createBuild()`：
```graphql
mutation CreateBuild($data: BuildCreateInput!) {
  createBuild(data: $data) {
    id
    status
    # ...
  }
}
```

**2. BuildService.create() 主流程** - [build.service.ts#L268-L352](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L352)

```typescript
async create(args: CreateBuildArgs): Promise<Build> {
  // 2.1 提取基本信息
  const resourceId = args.data.resource.connect.id;
  const user = await this.userService.findUser({...});
  const version = commitId.slice(commitId.length - 8);  // 取 commit 后 8 位

  // 2.2 获取最新实体版本
  const latestEntityVersions = await this.entityService.getLatestVersions({
    where: { resourceId }
  });

  // 2.3 写入数据库 build 表
  const build = await this.prisma.build.create({
    data: {
      ...args.data,
      version,
      createdAt: new Date(),
      status: EnumBuildStatus.Running,      // 状态设为 Running
      gitStatus: EnumBuildGitStatus.Waiting,
      entityVersions: {
        connect: latestEntityVersions.map(v => ({ id: v.id }))
      },
      action: {
        create: {
          steps: {
            create: createInitialStepData(version, args.data.message)
          }
        }
      }
    },
    include: { commit: true, resource: true }
  });

  // 2.4 检查资源类型（仅 Service 和 Component 支持代码生成）
  const resource = await this.resourceService.resource({
    where: { id: resourceId }
  });
  if (resource.resourceType !== EnumResourceType.Service &&
      resource.resourceType !== EnumResourceType.Component) {
    logger.info("Code generation is supported only for services and blueprints");
    return;
  }

  // 2.5 检查是否有私有插件
  const resourcePrivatePlugins =
    await this.pluginInstallationService.getInstalledPrivatePluginsForBuild(
      resourceId
    );

  // 2.6 分支处理
  if (resourcePrivatePlugins.length > 0) {
    // 有私有插件：异步下载流程
    logger.info(`${resourcePrivatePlugins.length} private plugins found.`);
    await this.downloadPrivatePlugins(
      logger, build, user, resourcePrivatePlugins
    );
  } else {
    // 无私有插件：直接进入代码生成
    logger.info(JOB_STARTED_LOG);
    await this.generate(logger, build, user);
  }

  return build;
}
```

**3. 私有插件下载的异步流程**

当存在私有插件时，流程变为异步：

```
downloadPrivatePlugins()
       ↓
发送 Kafka: DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC
       ├─ buildId
       ├─ resourceId
       └─ repositoryPlugins[]（按仓库分组的插件列表）
       ↓
[plugin-manager] 处理下载
       ↓
下载成功 → Kafka: DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC
下载失败 → Kafka: DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC
       ↓
[amplication-server]
BuildController.onDownloadPrivatePluginsSuccess()
       ↓
BuildService.onDownloadPrivatePluginSuccess()
       ↓
generate(logger, build, user)  ← 进入代码生成流程
```

**4. generate() 代码生成入口** - [build.service.ts#L568-L618](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L568-L618)

```typescript
private async generate(
  logger: ILogger,
  build: Build,
  user: User
): Promise<string> {
  // 包装在 actionService.run() 中管理 step 状态
  return this.actionService.run(
    build.actionId,
    GENERATE_STEP_NAME,
    GENERATE_STEP_MESSAGE,
    async (step) => {
      const { resourceId, id: buildId, version: buildVersion } = build;

      logger.info("Preparing build generation message");

      // 4.1 获取资源信息
      const resource = await this.resourceService.resource({
        where: { id: resourceId },
      });

      // 4.2 构建 DSGResourceData
      // ⚠️ 注意：omitDeep 敏感字段过滤是在 getDSGResourceData 内部完成的
      const dsgResourceData = await this.getDSGResourceData(
        resource,
        buildId,
        buildVersion,
        user
      );

      // 4.3 直接保存到共享存储（dsgResourceData 已经过滤了敏感字段）
      logger.info("Saving DSG resource data to shared storage");
      await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);

      logger.info("Writing lightweight build generation message to queue");

      // 4.4 发送 Kafka 触发 DSG（只传 ID，不传完整数据）
      const codeGenerationEvent: CodeGenerationRequest.KafkaEvent = {
        key: null,
        value: {
          resourceId,
          buildId,
        },
      };

      // ⚠️ 注意：成员名是 kafkaProducerService，不是 producerService
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
        codeGenerationEvent
      );

      logger.info("Build generation message sent");

      return null;  // ⚠️ 注意：返回 null，不是 "done"
    },
    true  // actionService.run 的第二个参数
  );
}
```

**⚠️ 三个需要校正的事实错误**：

| 错误点 | 之前错误描述 | 真实实现 |
|-------|-------------|---------|
| **敏感字段过滤位置** | 在 generate() 内部调用 omitDeep | 在 `getDSGResourceData()` 内部最后一行返回时调用 |
| **Kafka producer 成员名** | `this.producerService.emitMessage()` | `this.kafkaProducerService.emitMessage()` |
| **返回值** | `return "done"` | `return null` |

**敏感字段过滤的真实位置** - [build.service.ts#L1520](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L1518-L1521)

```typescript
// getDSGResourceData() 方法的最后
return omitDeep(dsgResourceData, DSG_RESOURCE_DATA_PROPERTIES_TO_REMOVE);
```

**调用链总结**：

```
GraphQL createBuild()
       ↓
BuildService.create()
       ├─ 创建 build 记录 (status=Running)
       ├─ 检查资源类型
       └─ 检查私有插件
              ├─ 有 → downloadPrivatePlugins() → Kafka 请求 → 等待回调 → generate()
              └─ 无 → generate()
                            ↓
                     getDSGResourceData()  ← 14 个数据源
                            ↓
                     saveDsgResourceDataToSharedStorage()
                            ↓
                     发送 Kafka: CODE_GENERATION_REQUEST_TOPIC
                            ↓
[amplication-build-manager] 接收并处理
```

### 1.5 核心输入数据结构：DSGResourceData

定义于 [dsg-resource-data.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/util/code-gen-types/src/dsg-resource-data.ts#L17-L38)

```typescript
export class DSGResourceData {
  resourceType!: keyof typeof EnumResourceType;  // Service / MessageBroker
  resourceInfo?: AppInfo;                        // 应用基本信息
  buildId!: string;
  entities?: Entity[];                           // 实体模型
  roles?: Role[];                                // 角色定义
  pluginInstallations!: PluginInstallation[];    // 插件列表
  packages?: Package[];                          // 自定义包
  moduleContainers?: ModuleContainer[];          // 模块容器
  moduleActions?: ModuleAction[];                // 模块动作
  moduleDtos?: ModuleDto[];                      // 自定义 DTO
  serviceTopics?: ServiceTopics[];               // 消息主题
  otherResources?: DSGResourceData[];            // 关联资源
}
```

### 1.6 构建入口的真实调用链

**纠正：构建入口不是直接调用，而是通过 Kafka 事件驱动**

[BuildRunnerController.onCodeGenerationRequest()](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L68-L78)

```typescript
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC)
async onCodeGenerationRequest(
  @Payload() message: CodeGenerationRequest.Value
): Promise<void> {
  await this.buildRunnerService.runBuild(message.resourceId, message.buildId);
}
```

**调用链**：
```
amplication-server → Kafka (CODE_GENERATION_REQUEST_TOPIC)
       ↓
amplication-build-manager (Kafka Consumer)
       ↓
BuildRunnerController.onCodeGenerationRequest()
       ↓
BuildRunnerService.runBuild(resourceId, buildId)
       ├─ 读取 DSGResourceData
       ├─ 解析代码生成器版本
       ├─ 拆分任务
       └─ 触发每个 Job 的 Argo Workflow
```

---

### 1.7 Code Generator 版本选择机制

**通过 Catalog Service 动态解析版本** - [code-generator-catalog.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/code-generator/code-generator-catalog.service.ts#L36-L59)

```typescript
async getCodeGeneratorVersion({
  codeGeneratorFullName,
  codeGeneratorVersion,
  codeGeneratorStrategy,
}: {
  codeGeneratorFullName: string;
  codeGeneratorVersion?: string;
  codeGeneratorStrategy?: CodeGeneratorVersionStrategy;
}): Promise<string | undefined>
```

**版本选择流程** - [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L114-L131)

```typescript
codeGeneratorVersion =
  await this.codeGeneratorService.getCodeGeneratorVersion({
    codeGeneratorFullName,
    codeGeneratorVersion:
      dsgResourceData.resourceInfo.codeGeneratorVersionOptions
        .codeGeneratorVersion,
    codeGeneratorStrategy:
      dsgResourceData.resourceInfo.codeGeneratorVersionOptions
        .codeGeneratorStrategy,
  });
```

**版本策略**：
| 策略 | 说明 | 示例 |
|-----|------|------|
| **Specific** | 使用用户指定的精确版本 | `v1.2.0` → `v1.2.0` |
| **LatestMinor** | 使用指定 minor 版本的最新 patch | `v1.2.0` + LatestMinor → `v1.2.5` |
| **LatestMajor** | 使用指定 major 版本的最新 minor | `v1.2.0` + LatestMajor → `v1.5.3` |
| **latest-local** | 特殊本地开发版本，跳过版本解析 |

---

### 1.8 Split 生效条件的精确逻辑

**三个条件必须同时满足才会拆分** - [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L35-L91)

```typescript
const shouldSplitBuild =
  // 条件1: 资源类型必须是 Service
  dsgResourceData.resourceType === EnumResourceType.Service &&
  // 条件2: 不是本地开发版本
  codeGeneratorVersion !== "latest-local" &&
  // 条件3: 版本 >= 环境变量配置的最低版本
  this.codeGeneratorService.compareVersions(
    codeGeneratorVersion,
    this.minDsgVersionToSplitBuild  // 来自 FEATURE_SPLIT_JOBS_MIN_DSG_VERSION
  ) >= 0;
```

**不满足条件时**：
- 不拆分，只创建一个 Job
- Job ID = 原始 Build ID（不带 -server/-admin-ui 后缀）
- 同时生成 Server 和 Admin UI

**满足条件时**：
- 根据 `generateServer` 和 `generateAdminUI` 设置决定创建哪些 Job
- Server Job: 设置 `generateAdminUI = false`，Job ID = `{buildId}-server`
- AdminUI Job: 设置 `generateServer = false`，Job ID = `{buildId}-admin-ui`

---

### 1.9 dsg-assets 的传递机制

**从资源目录复制到 Job 目录** - [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L349-L370)

```typescript
async saveRelevantDsgAssets(
  resourceId: string,
  buildId: string,      // jobBuildId (带后缀)
  plainBuildId: string  // 原始 buildId
) {
  // 源路径: 按 resourceId + plainBuildId 组织
  const dsgAssetsPathForBuild = join(
    this.configService.get(Env.DSG_ASSETS_FOLDER),
    `${resourceId}-${plainBuildId}`
  );

  if (!(await exists(dsgAssetsPathForBuild))) {
    return;  // 没有 assets 则跳过
  }

  // 目标路径: 每个 Job 目录下的 dsg-assets
  const jobPathForDsgAssets = join(
    this.configService.get(Env.DSG_JOBS_BASE_FOLDER),
    buildId,
    "dsg-assets"
  );

  await copy(dsgAssetsPathForBuild, jobPathForDsgAssets);
}
```

**dsg-assets 目录结构**：
```
{DSG_ASSETS_FOLDER}/
└── {resourceId}-{plainBuildId}/     ← 源目录 (按资源+构建组织)
    ├── plugin-a/
    ├── plugin-b/
    └── ...

{DSG_JOBS_BASE_FOLDER}/
├── {buildId}-server/
│   ├── resource-data.json
│   └── dsg-assets/                  ← 复制到这里 (Server Job)
│       ├── plugin-a/
│       └── ...
└── {buildId}-admin-ui/
    ├── resource-data.json
    └── dsg-assets/                  ← 复制到这里 (AdminUI Job)
        ├── plugin-a/
        └── ...
```

**关键点**：
- dsg-assets 是**可选**的，不存在时静默跳过
- 每个 Job 都获得完整的 assets 副本
- 用于传递插件二进制文件、静态资源等

---

### 1.10 构建任务拆分

[build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L35-L91)

```typescript
async splitBuildsIntoJobs(
  dsgResourceData: DSGResourceData,
  buildId: BuildId,
  codeGeneratorVersion: string
): Promise<ResourceTuple[]>
```

- **拆分条件**：Service 类型 + 非 latest-local + DSG 版本 >= 最低支持版本
- **拆分策略**：
  - Server 任务：`generateAdminUI = false`，只生成后端
  - AdminUI 任务：`generateServer = false`，只生成前端
- **Job ID 格式**：`{buildId}-{domain}`，例如 `abc123-server`

**2. 数据持久化与读取**

- 写入路径：`{DSG_JOBS_BASE_FOLDER}/{jobBuildId}/resource-data.json`
- 读取路径：由 `BUILD_SPEC_PATH` 环境变量指定
- 传递方式：通过文件系统共享，而非网络传输

**3. 上下文准备（prepareContext）** - [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/prepare-context.ts#L41-L124)

```typescript
export async function prepareContext(
  dSGResourceData: DSGResourceData,
  internalLogger: ILogger,
  pluginInstallationPath?: string
): Promise<void>
```

核心处理步骤：

| 处理步骤 | 函数 | 说明 |
|---------|------|------|
| 插件注册 | `registerPlugins()` | 动态加载插件，按事件分类 |
| 实体复数名 | `prepareEntityPluralName()` | 使用 pluralize 库生成 |
| 关联字段解析 | `resolveLookupFields()` | 解析 Lookup 字段的双向关联 |
| 服务主题准备 | `prepareServiceTopics()` | 解析消息队列主题名称 |
| 模块动作准备 | `prepareEntityActions()` | 合并默认动作与自定义动作 |
| DTO 引用解析 | `prepareModuleActionsAndDtos()` | 解析 DTO 间的引用关系 |
| 路径生成 | `dynamicServerPathCreator()` | 生成服务端/客户端目录结构 |

---

## 二、模板拼装机制：模板选择、数据绑定、渲染逻辑

### 2.1 模板系统架构

Amplication 采用 **AST（抽象语法树）级别**的模板引擎，而非传统的字符串模板。核心优势：
- 类型安全的模板操作
- 支持复杂的代码结构变换
- 自动处理导入语句合并
- 与 Prettier 无缝集成

### 2.2 模板文件格式

模板文件以 `.template.ts` 为后缀，使用**大写标识符**作为占位符。

示例：[service.base.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/server/resource/service/service.base.template.ts#L1-L39)

```typescript
import { PrismaService } from "../../prisma/prisma.service";
import { Prisma, ENTITY as PRISMA_ENTITY } from "@prisma/client";

declare const CREATE_ARGS_MAPPING: Prisma.CREATE_ARGS;

export class SERVICE_BASE {
  constructor(protected readonly prisma: PrismaService) {}

  async CREATE_ENTITY_FUNCTION(
    args: Prisma.CREATE_ARGS
  ): Promise<PRISMA_ENTITY> {
    return this.prisma.DELEGATE.create(CREATE_ARGS_MAPPING);
  }
}
```

### 2.3 核心渲染流程

**1. 模板解析** - [code-gen-utils/src/lib/parse/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/util/code-gen-utils/src/lib/parse/main.ts#L22-L41)

```typescript
export function parse(source: string, options?: ParseOptions): namedTypes.File
```

- 使用 Recast + Babel TypeScript 解析器
- 将模板文件解析为完整的 AST 树
- 支持 TypeScript 和 JSX 语法

**2. 标识符插值（interpolate）** - [ast.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/utils/ast.ts#L133-L207)

这是模板渲染的核心算法：

```typescript
export function interpolate(
  ast: ASTNode,
  mapping: { [key: string]: ASTNode | undefined }
): void
```

**工作原理**：
- 遍历整个 AST 树的所有 Identifier 节点
- 如果标识符名称在 mapping 中存在，则用对应的 AST 节点替换
- 支持多种节点类型的智能替换：

| 节点类型 | 处理方式 |
|---------|---------|
| 普通标识符 | 直接替换为 mapping 中的节点 |
| 模板字面量 | 如果所有表达式都映射为字符串字面量，自动合并为普通字符串 |
| JSX 元素 | 支持 JSX 表达式容器内的标识符替换 |
| 类装饰器 | 修复 Recast 遍历 Bug，确保装饰器被正确访问 |
| 类属性装饰器 | 同上 |
| 调用表达式类型参数 | 修复 Recast 遍历 Bug |

**3. 模板映射创建** - [create-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/server/resource/service/create-service.ts#L380-L417)

```typescript
function createTemplateMapping(
  entityType: string,
  serviceId: namedTypes.Identifier,
  serviceBaseId: namedTypes.Identifier,
  delegateId: namedTypes.Identifier,
  entityActions: entityActions
): { [key: string]: any } {
  return {
    SERVICE: serviceId,
    SERVICE_BASE: serviceBaseId,
    ENTITY: builders.identifier(entityType),
    PRISMA_ENTITY: builders.identifier(`Prisma${entityType}`),
    DELEGATE: delegateId,
    CREATE_ENTITY_FUNCTION: builders.identifier(
      entityActions.entityDefaultActions.Create.name
    ),
    // ... 更多映射
  };
}
```

**4. 代码生成（print）**

```typescript
import { print } from "recast";
const code = print(ast).code;
```

- 将修改后的 AST 转换回源代码字符串
- 保留原始代码格式
- 后续通过 Prettier 进行格式化

### 2.4 插件扩展机制

**插件包装器（pluginWrapper）** - [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L59-L117)

```typescript
const pluginWrapper: PluginWrapper = async (
  func,
  event,
  args
): Promise<ModuleMap>
```

**执行流程**：

```
调用 pluginWrapper(func, EventNames.CreateServer, args)
          ↓
检查 context.plugins[event] 是否存在
          ↓
┌───────────────────────────────────┐
│ beforePlugins 管道执行           │
│ 每个插件可修改 eventParams        │
│ 支持 skipDefaultBehavior 标志     │
└───────────────────────────────────┘
          ↓
如果 !skipDefaultBehavior → 执行原始 func
          ↓
┌───────────────────────────────────┐
│ afterPlugins 管道执行            │
│ 每个插件可修改返回的 ModuleMap    │
└───────────────────────────────────┘
          ↓
将最终模块 upsert 到 context.modules
          ↓
返回最终 ModuleMap
```

**插件注册** - [register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/register-plugin.ts#L129-L160)

```typescript
const registerPlugins = async (
  pluginList: PluginInstallation[],
  pluginInstallationPath?: string
): Promise<PluginMap>
```

- 支持 npm 包和本地私有插件
- 插件通过 `register()` 方法返回事件监听映射
- 事件分为 `before` 和 `after` 两个钩子
- 多个插件按顺序形成执行管道

### 2.5 动态代码生成技术

除了模板替换，还支持以下高级代码生成技术：

**1. AST 节点操作** - [ast.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/utils/ast.ts)

| 函数 | 用途 |
|-----|------|
| `addClassMethod()` | 向类添加方法 |
| `removeClassMethodByName()` | 根据名称删除类方法 |
| `addImports()` | 添加并合并导入语句 |
| `getClassDeclarationById()` | 通过标识符查找类声明 |
| `addAutoGenerationComment()` | 添加自动生成注释 |

**2. 混入（Mixin）模式**

在 `createServiceBaseModule()` 中，通过读取 `to-one.template.ts` 和 `to-many.template.ts`，提取其中的方法和导入，混入到主类中。

---

## 三、产物生成：文件写入、目录组织

### 3.1 模块容器：ModuleMap / FileMap

定义于 [file-map.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/util/code-gen-types/src/files/file-map.ts#L12-L119)

```typescript
export class FileMap<T> implements IFileMap<T> {
  private map: Map<string, IFile<T>> = new Map();
  
  async merge(anotherMap: FileMap<T>): Promise<FileMap<T>>
  async set(file: IFile<T>)
  get(path: string): IFile<T> | null
  replaceFilesPath(fn: (path: string) => string): void
  async replaceFilesCode(fn: (path: string, code: T) => T): Promise<void>
  getAll(): IterableIterator<IFile<T>>
}
```

**文件接口**：

```typescript
interface IFile<T> {
  path: string;      // 相对路径
  code: T;           // 文件内容（字符串或 AST）
}
```

### 3.2 Server 端生成流程

[create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/server/create-server.ts#L34-L168)

```typescript
async function createServerInternal(
  eventParams: CreateServerParams
): Promise<ModuleMap> {
  // 1. 静态文件复制
  const staticModules = await readStaticModules(STATIC_DIRECTORY, baseDir);
  
  // 2. 各模块并行生成
  const [
    customDtos, gitIgnore, packageJsonModule, dtoModules,
    resourcesModules, customModulesModules, authModules,
    swagger, seedModule, messageBrokerModules, secretsManagerModule,
    appModule, typesRelatedFiles, mainFile, prismaSchemaModule,
    dotEnvModule, connectMicroservicesModule, dockerComposeFile,
    dockerComposeDevFile
  ] = await Promise.all([
    createCustomDtos(),
    createGitIgnore(),
    createServerPackageJson(),
    createDTOModules(context.DTOs, dtoNameToPath),
    createResourcesModules(entities, dtoNameToPath),
    // ... 更多生成器
  ]);

  // 3. 代码格式化
  await resourcesModules.replaceModulesCode((path, code) => 
    formatCode(path, code)
  );

  // 4. 合并所有模块
  const moduleMap = new ModuleMap(context.logger);
  await moduleMap.mergeMany([...]);
  
  return moduleMap;
}
```

### 3.3 静态文件处理

[read-static-modules.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/utils/read-static-modules.ts#L33-L66)

```typescript
export async function readStaticModulesInner({
  source,
  basePath,
}: LoadStaticFilesParams): Promise<ModuleMap>
```

- 使用 `fast-glob` 递归扫描目录
- 忽略 `.js` 和 `.js.map` 文件
- 过滤 `._*` 和 `.DS_Store` 等系统文件
- 自动处理二进制文件编码

### 3.4 文件写入流程

[generate-code.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/generate-code.ts#L18-L46)

```typescript
const writeModules = async (
  modules: ModuleMap,
  destination: string
): Promise<void> => {
  internalLogger.info("Creating base directory");
  await mkdir(destination, { recursive: true });

  for await (const module of modules.modules()) {
    const filePath = join(destination, module.path);
    await mkdir(dirname(filePath), { recursive: true });
    try {
      const encoding = getFileEncoding(filePath);
      await writeFile(filePath, module.code, {
        encoding: encoding,
        flag: "wx",  // 关键：只在文件不存在时写入
      });
    } catch (error) {
      if (error.code === "EEXIST") {
        internalLogger.warn(`File ${filePath} already exists`);
      } else {
        internalLogger.error(`Failed to write file ${filePath}`, { ...error });
        throw error;
      }
    }
  }
};
```

**关键特性**：

- **`flag: "wx"`**：确保文件不会被覆盖，已存在则抛出 `EEXIST` 错误
- **递归创建目录**：使用 `mkdir({ recursive: true })`
- **编码自动检测**：`getFileEncoding()` 根据文件扩展名选择编码
- **幂等性处理**：文件已存在时记录警告但继续执行

### 3.5 产物目录结构

```
{BUILD_OUTPUT_PATH}/
├── server/                          # 后端代码
│   ├── src/
│   │   ├── auth/                    # 认证模块
│   │   ├── {entity}/                # 每个实体一个目录
│   │   │   ├── base/                # 可被覆盖的基类
│   │   │   │   ├── {entity}.service.base.ts
│   │   │   │   ├── {entity}.controller.base.ts
│   │   │   │   └── {entity}.resolver.base.ts
│   │   │   ├── {entity}.module.ts
│   │   │   ├── {entity}.service.ts  # 继承基类，可自定义
│   │   │   └── dto/                 # DTO 定义
│   │   ├── prisma/                  # Prisma schema
│   │   ├── swagger/                 # Swagger UI
│   │   └── main.ts                  # 应用入口
│   ├── prisma/
│   │   └── schema.prisma            # 数据库 schema
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── package.json
└── admin-ui/                        # 前端代码（React Admin）
    ├── src/
    │   ├── {entity}/                # 每个实体的 CRUD 页面
    │   ├── auth-provider/           # 认证提供者
    │   ├── data-provider/           # GraphQL 数据提供者
    │   └── App.tsx
    └── package.json
```

---

## 四、Admin UI 生成流程

### 4.1 Admin UI 的参与方式

Admin UI 与 Server 是**并行独立生成**的两个任务：
- 由 `splitBuildsIntoJobs()` 拆分为两个独立的 Job
- 各自在独立的 DSG 容器中运行
- 共享同一份 DSGResourceData 输入
- 通过构建管理器在最后合并产物

### 4.2 Admin UI 生成主流程

[create-admin.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/admin/create-admin.ts#L32-L137)

```typescript
export function createAdminModules(): Promise<ModuleMap> {
  return pluginWrapper(
    createAdminModulesInternal,
    EventNames.CreateAdminUI,
    {}
  );
}

async function createAdminModulesInternal(): Promise<ModuleMap> {
  const context = DsgContext.getInstance;
  const { entities, roles, clientDirectories, logger } = context;
```

**生成步骤**：

| 步骤 | 模块 | 说明 |
|-----|------|------|
| 1 | `readStaticModules()` | 复制静态模板文件 |
| 2 | `createGitIgnore()` | 生成 .gitignore |
| 3 | `createAdminUIPackageJson()` | 生成 package.json |
| 4 | `createPublicFiles()` | 生成公共资源文件 |
| 5 | `createAdminDTONameToPath()` | DTO 到路径的映射 |
| 6 | `createDTOModules()` | 生成 DTO 模块 |
| 7 | `createEnumRolesModule()` | 生成角色枚举 |
| 8 | `createRolesModule()` | 生成角色模块 |
| 9 | `createEntityTitleComponents()` | 生成实体标题组件（先执行，被依赖） |
| 10 | `createEntitiesComponents()` | 生成实体 CRUD 组件 |
| 11 | `createAppModule()` | 生成 App.tsx 主模块 |
| 12 | `createDotEnvModule()` | 生成 .env 文件 |
| 13 | `formatCode()` | 格式化所有 TypeScript 文件 |
| 14 | `createTypesRelatedFiles()` | 生成类型相关文件 |

### 4.3 实体组件生成机制

[create-entities-components.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/admin/entity/create-entities-components.ts#L8-L27)

```typescript
export async function createEntitiesComponents(
  entities: Entity[],
  entityToDirectory: Record<string, string>,
  entityToTitleComponent: Record<string, EntityComponent>,
  entityNameToEntity: Record<string, Entity>
): Promise<Record<string, EntityComponents>>
```

**每个实体生成的组件**：

```typescript
interface EntityComponents {
  list: EntityComponent;      // 列表页 (List)
  show: EntityComponent;      // 详情页 (Show)
  edit: EntityComponent;      // 编辑页 (Edit)
  create: EntityComponent;    // 创建页 (Create)
}
```

**组件依赖链**：
```
createEntityTitleComponents()  → 生成 {Entity}Title.tsx
         ↓
createEntitiesComponents()
         ├─ createList()    → {Entity}List.tsx
         ├─ createShow()    → {Entity}Show.tsx
         ├─ createEdit()    → {Entity}Edit.tsx
         └─ createCreate()  → {Entity}Create.tsx
         ↓
createEntityComponentsModules() → 转为 ModuleMap
         ↓
createAppModule() → App.tsx 中注册所有资源路由
```

### 4.4 Admin UI 与 Server 的数据共享

**共享上下文**：
- 同一份 `entities` 数据（从 DSGResourceData 解析）
- 同一份 `roles` 定义
- 同一份 `DTOs` 结构

**路径配置**：
- `clientDirectories.baseDirectory` → `{output}/admin-ui`
- `serverDirectories.baseDirectory` → `{output}/server`

**API 路径映射**：
```typescript
const entityToResource = Object.fromEntries(
  entities.map((entity) => [
    entity.name,
    `${API_PATHNAME}/${paramCase(plural(entity.name))}`,
  ])
);
```
- 例如：`User` → `/api/users`
- 与 Server 端 REST API 路径保持一致

---

## 五、构建管理器：合并产物与失败处理

### 5.1 Job 完成状态处理

[build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L208-L255)

```typescript
async handleDsgJobCompleted(resourceId: string, jobBuildId: string) {
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  let otherJobsHaveNotFailed = true;

  try {
    // 1. 检查其他任务是否已失败
    const currentBuildStatus =
      await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // 2. 复制产物到 artifact 目录
    await this.copyFromJobToArtifact(resourceId, jobBuildId);

    // 3. 更新当前 Job 状态为 Success
    await this.buildJobsHandlerService.setJobStatus(
      jobBuildId,
      EnumJobStatus.Success
    );

    // 4. 重新检查整体状态
    const buildStatus = await this.buildJobsHandlerService.getBuildStatus(
      buildId
    );

    if (buildStatus === EnumJobStatus.InProgress) {
      return;  // 还有其他任务在运行
    }

    if (buildStatus === EnumJobStatus.Success) {
      // 5. 全部成功：生成包或发送完成通知
      const dsgResourceData =
        await this.buildJobsHandlerService.extractDsgResourceData(jobBuildId);

      if (dsgResourceData.packages?.length > 0 && this.enablePackageManager) {
        await this.generatePackages(buildId, resourceId, dsgResourceData);
      } else {
        await this.codeGenerationAndPackagesCompleted(jobBuildId);
      }
    }
  } catch (error) {
    this.logger.error(error.message, error);
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId, error.message);
    }
  }
}
```

### 5.2 包生成与完成回调

**包生成触发时机** - [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L236-L246)

```typescript
if (buildStatus === EnumJobStatus.Success) {
  const dsgResourceData =
    await this.buildJobsHandlerService.extractDsgResourceData(jobBuildId);

  // package manager is called only after all the jobs are completed
  if (dsgResourceData.packages?.length > 0 && this.enablePackageManager) {
    await this.generatePackages(buildId, resourceId, dsgResourceData);
  } else {
    this.logger.info("No packages to generate - complete build");
    await this.codeGenerationAndPackagesCompleted(jobBuildId);
  }
}
```

**关键点**：
- 包生成**只在所有 Job 都成功后**才触发
- 只有当 `dsgResourceData.packages.length > 0` 且 `enablePackageManager` 为 true 时才执行
- 否则直接调用完成回调

---

**包生成完成回调流程** - [build-runner.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L44-L66)

```typescript
// 包生成成功回调
@EventPattern(KAFKA_TOPICS.PACKAGE_MANAGER_CREATE_SUCCESS)
async onPackageManagerCreateSuccess(
  @Payload() message: PackageManagerCreateSuccess.Value
): Promise<void> {
  await this.buildRunnerService.onPackageManagerCreateSuccess(args);
}

// 包生成失败回调
@EventPattern(KAFKA_TOPICS.PACKAGE_MANAGER_CREATE_FAILURE)
async onPackageManagerCreateFailure(
  @Payload() message: PackageManagerCreateFailure.Value
): Promise<void> {
  await this.buildRunnerService.onPackageManagerCreateFailure(args);
}
```

**完成回调处理** - [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L54-L67)

```typescript
async onPackageManagerCreateSuccess(
  response: PackageManagerCreateSuccess.Value
) {
  await this.codeGenerationAndPackagesCompleted(response.buildId);
}

async onPackageManagerCreateFailure(
  response: PackageManagerCreateFailure.Value
) {
  await this.emitCodeGenerationFailure(
    response.buildId,
    response.errorMessage
  );
}
```

**codeGenerationAndPackagesCompleted 完成函数** - [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L90-L107)

```typescript
/// this function accepts either the buildId or the jobBuildId
/// This method is called when the code generation and the packages generation are completed
/// It emits a kafka event with the buildId and the code generator version
async codeGenerationAndPackagesCompleted(buildIdOrJobBuildId: string) {
  const buildId =
    this.buildJobsHandlerService.extractBuildId(buildIdOrJobBuildId);

  const successEvent: CodeGenerationSuccess.KafkaEvent = {
    key: null,
    value: { buildId },  // ✅ 实际只发送 buildId
  };

  this.logger.info("emit code generation success event", successEvent);
  await this.producerService.emitMessage(
    KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC,
    successEvent
  );
}
```

**⚠️ 重要校正**：完成通知 **实际只发送 buildId**，不包含 resourceId 和 codeGeneratorVersion。

**Schema 定义** - [code-generation-success/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/schema-registry/src/lib/code-generation-success/value.ts#L1-L6)

```typescript
export class Value {
  @IsString()
  buildId!: string;
}
```

| 字段 | 是否存在 | 说明 |
|-----|---------|------|
| buildId | ✅ 是 | 原始 Build ID（不带 -server/-admin-ui 后缀） |
| resourceId | ❌ 否 | 不发送 |
| codeGeneratorVersion | ❌ 否 | 不发送 |

**对比 CodeGenerationRequest** - [code-generation-request/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/schema-registry/src/lib/code-generation-request/value.ts#L1-L8)

```typescript
export class Value {
  @IsString()
  buildId!: string;
  @IsString()
  resourceId!: string;  // ✅ 请求时有 resourceId
}
```

**完整的成功路径**：
```
所有 DSG Job 成功
       ↓
检查是否需要生成包
       ├─ 不需要 → codeGenerationAndPackagesCompleted() → Kafka Success
       └─ 需要 → generatePackages() → Package Manager
                          ↓
                Package Manager 处理完成
                          ↓
                Kafka: PACKAGE_MANAGER_CREATE_SUCCESS
                          ↓
                codeGenerationAndPackagesCompleted() → Kafka Success
```

**完整的失败路径**：
```
任一 DSG Job 失败
       ↓
handleDsgJobCompleted catch 块
       ↓
emitCodeGenerationFailure() → Kafka Failure

或

包生成失败
       ↓
Kafka: PACKAGE_MANAGER_CREATE_FAILURE
       ↓
onPackageManagerCreateFailure()
       ↓
emitCodeGenerationFailure() → Kafka Failure
```

---

### 5.3 产物合并机制

[copyFromJobToArtifact()](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L372-L396)

```typescript
async copyFromJobToArtifact(
  resourceId: string,
  jobBuildId: string
): Promise<void> {
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);

  const jobPath = join(
    this.configService.get(Env.DSG_JOBS_BASE_FOLDER),
    jobBuildId,
    this.configService.get(Env.DSG_JOBS_CODE_FOLDER)  // 如 "code"
  );

  const artifactPath = join(
    this.configService.get(Env.BUILD_ARTIFACTS_BASE_FOLDER),
    resourceId,
    buildId
  );

  await copy(jobPath, artifactPath);
}
```

**目录布局**：
```
/dsg-jobs/                          # Job 工作目录
├── {buildId}-server/              # Server 任务目录
│   ├── resource-data.json
│   └── code/                      # Server 生成产物
│       └── server/
│           ├── src/
│           └── package.json
└── {buildId}-admin-ui/            # Admin UI 任务目录
    ├── resource-data.json
    └── code/                      # Admin UI 生成产物
        └── admin-ui/
            ├── src/
            └── package.json

/build-artifacts/                   # 最终产物目录
└── {resourceId}/
    └── {buildId}/                  # 合并后的产物
        ├── server/                 ← 从 {buildId}-server/code/server 复制
        └── admin-ui/               ← 从 {buildId}-admin-ui/code/admin-ui 复制
```

**合并特性**：
- **增量合并**：每个任务完成后立即复制，不等待其他任务
- **目录隔离**：server/ 和 admin-ui/ 目录天然隔离，不会冲突
- **幂等操作**：使用 fs-extra 的 `copy()`，默认覆盖已有文件

### 5.3 失败处理的精确逻辑

**失败状态更新** - [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L257-L277)

```typescript
async emitCodeGenerationFailureWhenJobStatusFailed(jobBuildId: string) {
  let otherJobsHaveNotFailed = true;

  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  try {
    // 检查其他任务是否已失败
    const currentBuildStatus =
      await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // 标记当前任务失败
    await this.buildJobsHandlerService.setJobStatus(
      jobBuildId,
      EnumJobStatus.Failure
    );
  } catch (error) {
    this.logger.error(error.message, error);
  } finally {
    // 只有第一个失败的任务触发通知
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId);
    }
  }
}
```

**失败通知去重机制**：

| 场景 | 处理方式 |
|-----|---------|
| Server 先失败 | 发送失败通知，Admin UI 即使后续失败也不重复通知 |
| Admin UI 先失败 | 发送失败通知，Server 即使后续失败也不重复通知 |
| 两者同时失败 | 只有第一个到达的发送通知 |

**Redis 状态聚合** - [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L101-L122)

```typescript
async getBuildStatus(key: BuildId): Promise<EnumJobStatus> {
  const buildValue = await this.redisService.get<RedisValue>(key);
  const jobsStatus = Object.values(buildValue);

  if (jobsStatus.every(s => s === EnumJobStatus.Success)) 
    return EnumJobStatus.Success;        // 全成功
  
  if (jobsStatus.some(s => s === EnumJobStatus.Failure)) 
    return EnumJobStatus.Failure;        // 任一失败 → 整体失败
  
  if (jobsStatus.some(s => s === EnumJobStatus.InProgress)) 
    return EnumJobStatus.InProgress;     // 仍在运行
}
```

### 5.4 产物保留与清理

**当前策略**：
- ✅ Job 目录保留：`/dsg-jobs/{jobBuildId}/`
- ✅ Artifact 目录保留：`/build-artifacts/{resourceId}/{buildId}/`
- ❌ 无自动清理机制（依赖外部清理策略）
- ❌ 失败时已复制的产物不会被删除

**潜在问题**：
1. **部分产物问题**：如果 Server 成功但 Admin UI 失败，artifact 目录会有不完整的 server 代码
2. **存储空间**：大量构建会占用大量磁盘空间
3. **调试困难**：Job 目录保留有助于调试，但也可能泄露敏感信息

---

## 六、失败回退机制：异常处理、事务性操作

### 6.1 异常处理层级结构

Amplication 的异常处理采用**多层防御**策略：

```
┌─────────────────────────────────────────────────────────────┐
│  Level 1: DSG 容器入口                                       │
│  [main.ts] generateCode()                                    │
│  - 捕获所有异常                                              │
│  - 调用 buildManagerNotifier.failure()                      │
│  - 进程退出码 1                                             │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Level 2: 数据服务创建                                       │
│  [create-data-service.ts] createDataService()               │
│  - 捕获上下文准备、DTO 创建、模块生成阶段异常                │
│  - 记录详细错误日志                                          │
│  - 重新抛出异常                                              │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Level 3: 插件执行                                           │
│  [plugin-wrapper.ts] pluginWrapper()                        │
│  - 捕获插件 before/after 钩子异常                            │
│  - 包装错误信息，注明是哪个事件失败                          │
│  - 支持 abortGeneration() 主动中止                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│  Level 4: 构建管理器                                         │
│  [build-runner.service.ts] handleDsgJobCompleted()          │
│  - 检查其他任务是否已失败                                    │
│  - 只在首次失败时发送失败通知                                │
│  - 更新 Redis 任务状态                                       │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 失败通知机制

**BuildManagerNotifier** - [notify-build-manager.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/util/dsg-utils/src/build-manager-notifier/notify-build-manager.ts#L16-L71)

```typescript
export class BuildManagerNotifier {
  async success(): Promise<void>   // POST /build-runner/code-generation-success
  async failure(): Promise<void>   // POST /build-runner/code-generation-failure
  async notifyPluginVersion(args): Promise<void>
}
```

**调用时机**：
- **success()**：所有模块生成并写入成功后
- **failure()**：`generateCode()` 的 catch 块中
- **notifyPluginVersion()**：每个插件安装成功后

### 6.3 多任务状态聚合

[build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L101-L122)

```typescript
async getBuildStatus(key: BuildId): Promise<EnumJobStatus> {
  const buildValue = await this.redisService.get<RedisValue>(key);
  const jobsStatus = Object.values(buildValue);

  if (jobsStatus.every(s => s === EnumJobStatus.Success)) 
    return EnumJobStatus.Success;
  
  if (jobsStatus.some(s => s === EnumJobStatus.Failure)) 
    return EnumJobStatus.Failure;
  
  if (jobsStatus.some(s => s === EnumJobStatus.InProgress)) 
    return EnumJobStatus.InProgress;
}
```

**状态规则**：
- **全成功** → Success
- **任一失败** → Failure（快速失败）
- **其他情况** → InProgress

### 6.4 失败回退的局限性

**重要：当前设计不支持事务性回滚**

| 特性 | 支持状态 | 说明 |
|-----|---------|------|
| 原子性文件写入 | ❌ 部分支持 | 使用 `flag: "wx"` 防止覆盖，但已写入文件不会自动删除 |
| 失败时清理已生成文件 | ❌ 不支持 | 没有 rollback 机制，部分生成的文件会残留在磁盘 |
| 多任务一致性 | ⚠️ 部分支持 | 一个任务失败不影响其他任务继续执行，但整体标记为失败 |
| 幂等重建 | ✅ 支持 | 重新触发构建会覆盖已有 job 目录，重新生成所有文件 |

**文件写入的竞态条件处理**：

在 `writeModules()` 中：
- 使用 `flag: "wx"`（write exclusive）确保不会覆盖已有文件
- 捕获 `EEXIST` 错误，记录警告后继续
- 这种设计假设输出目录是干净的（每次构建使用新目录）

### 6.5 日志与可观测性

**BuildLogger** - [build-logger.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/util/dsg-utils/src/build-logger/build-logger.ts#L6-L64)

```typescript
export class BuildLogger implements IBuildLogger {
  async info(message, params?, userFriendlyMessage?)
  async warn(message, params?, userFriendlyMessage?)
  async error(message, params?, userFriendlyMessage?, error?)
}
```

**双日志机制**：
1. **应用日志**：通过 `applicationLogger` 输出到控制台/日志系统
2. **构建日志**：通过 HTTP POST 发送到 `build-logger/create-log`，用户可在 UI 查看

**日志格式**：
- `message`：系统内部日志消息（英文，技术细节）
- `userFriendlyMessage`：面向用户的友好消息（可本地化）
- `params`：结构化元数据，用于调试

---

## 七、完整流水线时序图

```
用户点击"构建"
    │
    ▼
[amplication-server]
    │  创建 Build 记录
    │  发送 Kafka 消息: CODE_GENERATION_REQUEST_TOPIC
    ▼
[amplication-build-manager]
    │
    ├─► BuildRunnerController.onCodeGenerationRequest()
    │
    ├─► BuildRunnerService.runBuild()
    │    ├─ 读取 DSGResourceData
    │    ├─ splitBuildsIntoJobs() → [serverJob, adminUIJob]
    │    └─ 对每个 job:
    │        ├─ saveDsgResourceData()
    │        ├─ setJobStatus(InProgress)
    │        └─ POST 到 Argo 事件 → 启动 DSG 容器
    │
[DSG Container - data-service-generator]
    │
    ├─► main.ts: generateCode()
    │    ├─ 读取 input.json
    │    │
    │    ├─► createDataService()
    │    │    ├─ 动态安装插件
    │    │    ├─► prepareContext()
    │    │    │   ├─ registerPlugins()
    │    │    │   ├─ resolveLookupFields()
    │    │    │   └─ prepareEntityActions()
    │    │    ├─ createDTOs()
    │    │    ├─► createServer()  [pluginWrapper]
    │    │    │   ├─ before 插件钩子
    │    │    │   ├─ 读取静态文件
    │    │    │   ├─ 并行生成各模块
    │    │    │   │   ├─ createResourcesModules()
    │    │    │   │   ├─ createDTOModules()
    │    │    │   │   └─ ...
    │    │    │   ├─ 代码格式化
    │    │    │   ├─ 合并 ModuleMap
    │    │    │   └─ after 插件钩子
    │    │    └─► createAdminModules()  [同上]
    │    │
    │    ├─► writeModules()
    │    │    ├─ 创建输出目录
    │    │    └─ 遍历 ModuleMap 写入文件
    │    │
    │    └─ buildManagerNotifier.success() / failure()
    │
[amplication-build-manager]
    │
    ├─► onCodeGenerationSuccess() / onCodeGenerationFailure()
    │    ├─ setJobStatus(Success/Failure)
    │    ├─ getBuildStatus() 检查整体状态
    │    ├─ copyFromJobToArtifact()  复制到产物目录
    │    └─ 如全部成功:
    │        ├─ generatePackages() (如有)
    │        └─ 发送 Kafka: CODE_GENERATION_SUCCESS_TOPIC
    │
    ▼
用户下载/查看构建结果
```

---

## 八、关键代码路径索引

| 功能 | 文件 | 关键函数 |
|-----|------|---------|
| **DSGResourceData 构建** | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L1384-L1521) | `getDSGResourceData()` |
| 实体排序保证 | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L1349-L1371) | `getOrderedEntities()` |
| 共享存储保存 | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-server/src/core/build/build.service.ts#L541-L560) | `saveDsgResourceDataToSharedStorage()` |
| **构建 Kafka 入口** | [build-runner.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.controller.ts#L68-L78) | `onCodeGenerationRequest()` |
| **构建主流程** | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159) | `runBuild()` |
| **版本选择** | [code-generator-catalog.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/code-generator/code-generator-catalog.service.ts#L36-L59) | `getCodeGeneratorVersion()` |
| **任务拆分条件** | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L35-L91) | `splitBuildsIntoJobs()` |
| **dsg-assets 传递** | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L349-L370) | `saveRelevantDsgAssets()` |
| Job 执行 | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L161-L192) | `runJob()` |
| Job 完成处理 | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L208-L255) | `handleDsgJobCompleted()` |
| **包生成触发** | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L69-L88) | `generatePackages()` |
| **包生成成功回调** | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L54-L58) | `onPackageManagerCreateSuccess()` |
| **包生成失败回调** | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L60-L67) | `onPackageManagerCreateFailure()` |
| **完成通知** | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L90-L105) | `codeGenerationAndPackagesCompleted()` |
| 产物合并 | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L372-L396) | `copyFromJobToArtifact()` |
| 失败状态更新 | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L257-L277) | `emitCodeGenerationFailureWhenJobStatusFailed()` |
| DSG 主入口 | [main.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/main.ts#L1-L9) | `generateCode()` |
| 代码生成核心 | [generate-code.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/generate-code.ts#L48-L94) | `generateCodeByResourceData()` |
| 数据服务创建 | [create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/create-data-service.ts#L15-L105) | `createDataService()` |
| 上下文准备 | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/prepare-context.ts#L41-L124) | `prepareContext()` |
| 模板渲染 | [ast.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/utils/ast.ts#L133-L207) | `interpolate()` |
| 插件包装 | [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L59-L117) | `pluginWrapper()` |
| 服务端生成 | [create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/server/create-server.ts#L34-L168) | `createServer()` |
| **Admin UI 生成** | [create-admin.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/admin/create-admin.ts#L32-L137) | `createAdminModules()` |
| **实体组件生成** | [create-entities-components.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/admin/entity/create-entities-components.ts#L8-L27) | `createEntitiesComponents()` |
| 文件写入 | [generate-code.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/data-service-generator/src/generate-code.ts#L18-L46) | `writeModules()` |
| 状态管理 | [build-job-handler.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts#L101-L148) | `getBuildStatus()`, `setJobStatus()` |
| 失败通知 | [notify-build-manager.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/util/dsg-utils/src/build-manager-notifier/notify-build-manager.ts#L16-L71) | `BuildManagerNotifier` |

---

## 九、设计特点与潜在改进点

### 现有设计优势

1. **14 数据源统一汇总**：`getDSGResourceData()` 统一收集实体、角色、插件、模块等 14 个数据源
2. **AST 级模板**：比字符串模板更健壮，支持复杂的代码变换
3. **插件管道**：before/after 钩子提供了强大的扩展能力
4. **任务并行化**：Server/AdminUI 分离构建，独立容器运行
5. **增量合并**：每个任务完成后立即复制产物，不等待全部完成
6. **失败去重通知**：只有第一个失败任务触发通知，避免重复告警
7. **双日志系统**：兼顾内部调试和用户反馈

### 潜在改进点（新增纠正后的发现）

1. **完成通知只发送 buildId**：CodeGenerationSuccess 事件只包含 buildId，不包含 resourceId
   - Schema 定义：[code-generation-success/value.ts](file:///d:/fz/0601/solo-dogfeeding/code/23-amplication/libs/schema-registry/src/lib/code-generation-success/value.ts#L1-L6)
   - 说明：这是**设计如此**，不是 Bug。下游服务需要通过 buildId 查询数据库获取 resourceId
   - 潜在问题：增加了下游服务的数据库查询压力

2. **版本选择是外部依赖**：通过 Catalog Service HTTP 调用解析版本
   - 风险：Catalog Service 不可用时构建会失败
   - 建议：增加降级策略，使用用户指定版本作为 fallback

3. **dsg-assets 复制到每个 Job 目录**：存在冗余
   - 当前：Server 和 AdminUI Job 各复制一份相同的 assets
   - 建议：使用符号链接或共享挂载，减少磁盘占用

4. **latest-local 特殊处理**：硬编码判断 `codeGeneratorVersion !== "latest-local"`
   - 问题：本地开发时不会拆分任务，与生产环境行为不一致
   - 建议：通过配置或环境变量控制，而非硬编码版本号

5. **缺乏事务性回滚**：失败时已写入的文件不会被清理
   - 建议：先写入临时目录，全部成功后再原子性移动到目标目录
   - 或：记录已写入文件列表，失败时遍历删除

6. **部分产物问题**：Server 成功但 Admin UI 失败时，artifact 目录有不完整代码
   - 建议：引入"预备目录"概念，所有任务完成后才移动到最终目录

7. **错误上下文不足**：插件错误只显示事件名，缺少插件标识
   - 建议：在 `pluginWrapper` 中记录具体哪个插件失败

8. **文件覆盖策略**：`flag: "wx"` 在目录不干净时会导致构建失败
   - 建议：构建前先清理输出目录，或提供覆盖选项

9. **缺乏中间状态持久化**：大项目构建中断后需从头开始
   - 建议：支持增量构建，缓存已生成的模块

10. **内存占用**：所有 ModuleMap 常驻内存，大项目可能 OOM
    - 建议：支持流式写入，或分批生成分批写入

11. **Job 目录无自动清理**：大量构建会占用大量磁盘空间
    - 建议：增加 TTL 自动清理机制，或在构建成功后清理
