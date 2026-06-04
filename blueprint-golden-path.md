# Blueprint 模板编排与 Golden Path 机制解析

## 一、核心概念

### 1.1 Blueprint 是什么？

Blueprint 是 Amplication 中的一个核心概念，用于定义和管理可复用的资源模板。它本质上是一个"蓝图"，定义了：
- 资源类型（Service、Component、MessageBroker 等）
- 代码生成器类型（NodeJs、DotNet、Blueprint）
- 自定义属性（Custom Properties）
- 资源关系（Blueprint Relations）

**核心数据结构** 定义在 [packages/amplication-server/src/models/Blueprint.ts](packages/amplication-server/src/models/Blueprint.ts)：

```typescript
class Blueprint {
  id: string;                    // 唯一标识
  name: string;                  // 名称
  key: string;                   // 唯一键（大写蛇形命名）
  color?: string;                // 颜色标识
  enabled: boolean;              // 是否启用
  description?: string;          // 描述
  resourceType: EnumResourceType; // 资源类型
  codeGeneratorName?: string;    // 代码生成器名称
  useBusinessDomain: boolean;    // 是否使用业务域
  relations?: BlueprintRelation[]; // 蓝图关系
  properties?: CustomProperty[];  // 自定义属性
}
```

**资源类型与代码生成器的对应关系** 在 [packages/amplication-server/src/core/blueprint/blueprint.service.ts](packages/amplication-server/src/core/blueprint/blueprint.service.ts#L29-L38) 定义：

| 资源类型 | 支持的代码生成器 |
|---------|----------------|
| Component | Blueprint |
| Service | NodeJs, DotNet |
| MessageBroker | Blueprint |

### 1.2 Golden Path 是什么？

Golden Path（黄金路径）是 Amplication Platform Console 中提出的一个产品概念，指通过以下机制来规范化和标准化开发流程：

1. **Blueprint 模板**：定义标准化的资源结构
2. **Service Template**：服务模板，封装完整的资源配置和插件集合
3. **Private Plugins**：私有插件，用于集成最佳实践
4. **Live Templates**：实时模板，确保一致性

**相关描述** 在 [packages/amplication-client/src/Platform/PlatformDashboard.tsx](packages/amplication-client/src/Platform/PlatformDashboard.tsx#L64-L71)：

> "The Platform Console lets teams define, manage, and enforce development standards at scale. It streamlines service creation with Live Templates for consistency and Private Plugins to integrate best practices and Golden Paths."

**实现方式**：Golden Path 主要通过 **Blueprint + Service Template + Plugin 机制** 实现：
- Blueprint 定义资源的元模型和约束
- Service Template 基于 Blueprint 创建具体的可复用模板
- Plugin 机制在代码生成的各个阶段嵌入团队的最佳实践和标准流程

---

## 二、模板管理体系（Blueprint + Service Template）

### 2.1 模板分层架构

```
┌─────────────────────────────────────────┐
│              Blueprint                  │
│  (定义资源类型、属性、关系约束)         │
└─────────────────┬───────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│          Service Template               │
│  (基于 Blueprint，封装配置、插件、版本) │
└─────────────────┬───────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│              Resource                   │
│  (从 Template 创建的实际资源实例)       │
└─────────────────────────────────────────┘
```

### 2.2 Service Template 核心服务

**核心文件**：[packages/amplication-server/src/core/resource/serviceTemplate.service.ts](packages/amplication-server/src/core/resource/serviceTemplate.service.ts)

#### 2.2.1 创建 Service Template

```typescript
async createServiceTemplate(args: CreateServiceTemplateArgs, user: User): Promise<Resource> {
  // 1. 创建 ServiceTemplate 类型的资源
  const resource = await this.resourceService.createResource({
    data: {
      ...rest,
      resourceType: EnumResourceType.ServiceTemplate,
    },
  }, user);

  // 2. 创建 Service 默认对象（settings、roles 等）
  await this.resourceService.createServiceDefaultObjects(
    resource, user, false, serviceSettings
  );

  // 3. 安装指定的插件
  if (args.data.plugins?.plugins) {
    await this.resourceService.installPlugins(...);
  }

  return resource;
}
```

#### 2.2.2 从现有资源创建模板

```typescript
async createTemplateFromExistingResource(args, user): Promise<Resource> {
  // 1. 验证资源必须关联 Blueprint
  if (!resource.blueprintId) {
    throw new AmplicationError(
      `This method only support resources with a blueprint`
    );
  }

  // 2. 创建模板资源，关联相同的 Blueprint
  const template = await this.resourceService.createResource({
    data: {
      name: `${resource.name}-template`,
      resourceType: EnumResourceType.ServiceTemplate,
      blueprint: resource.blueprintId
        ? { connect: { id: resource.blueprintId } }
        : undefined,
      // ...
    },
  }, user);

  // 3. 复制服务设置，替换路径中的服务名为占位符
  serviceSettings.serverSettings.serverPath =
    `${serverBasePath}/{{SERVICE_NAME}}`;

  // 4. 复制插件安装配置
  await this.copyPluginInstallations(resourceId, template.id, user);

  return template;
}
```

#### 2.2.3 从模板创建资源（Golden Path 核心入口）

```typescript
async createResourceFromTemplate(args, user): Promise<Resource> {
  // 1. 获取可用模板（当前项目 + 公开项目）
  const serviceTemplates = await this.availableServiceTemplatesForProject(...);

  // 2. 获取模板最新版本
  const templateVersion = await this.resourceVersionService.getLatest(template.id);

  // 3. 验证模板关联的 Blueprint 已启用
  const blueprint = await this.prisma.blueprint.findUnique(...);
  if (!blueprint.enabled) {
    throw new AmplicationError(`The selected template is based on a disabled blueprint.`);
  }

  // 4. 根据 Blueprint 的 resourceType 创建 Service 或 Component
  let newResource: Resource;
  if (resourceType === EnumResourceType.Component) {
    newResource = await this.internalCreateComponentFromTemplate(...);
  } else {
    newResource = await this.internalCreateServiceFromTemplate(...);
  }

  // 5. 记录模板版本关联（用于后续版本升级）
  await this.resourceTemplateVersionService.updateResourceTemplateVersion({
    where: { id: newResource.id },
    data: {
      serviceTemplateId: template.id,
      version: templateVersion.version,
    },
  }, user);

  // 6. 复制模板的插件配置到新资源
  await this.copyPluginInstallations(args.data.serviceTemplate.id, newResource.id, user);

  // 7. 可选：创建后立即构建（buildAfterCreation）
  if (args.data.buildAfterCreation) {
    await this.projectService.commit({
      data: {
        message: "Create resource from template",
        commitStrategy: EnumCommitStrategy.Specific,
        resourceIds: [newResource.id],
        // ...
      },
    }, user);
  }

  return newResource;
}
```

#### 2.2.4 模板版本升级

```typescript
async upgradeServiceToLatestTemplateVersion(args, user): Promise<Resource> {
  // 1. 获取资源当前使用的模板版本
  const serviceTemplateVersion =
    await this.resourceService.getServiceTemplateSettings(resourceId, user);

  // 2. 获取模板最新版本
  const latestVersion = await this.resourceVersionService.getLatest(template.id);

  // 3. 比较版本差异（新增、修改、删除的 Blocks）
  const changes = await this.resourceVersionService.compareResourceVersions({
    where: {
      resource: { id: template.id },
      sourceVersion: serviceTemplateVersion.version,
      targetVersion: latestVersion.version,
    },
  });

  // 4. 合并变更到资源
  // - 新增的 Blocks（插件、代码引擎版本等）
  // - 修改的 Blocks
  // - 删除的 Blocks
  await Promise.all([createdPromises, deletedPromises, updatedPromises]);

  // 5. 更新资源关联的模板版本
  await this.resourceTemplateVersionService.updateResourceTemplateVersion(...);

  // 6. 解决版本过期告警
  await this.outdatedVersionAlertService.resolvesServiceTemplateUpdated({
    resourceId: resourceId,
  });

  return resource;
}
```

---

## 三、触发入口：完整调用链

### 3.1 触发流程图（含私有插件下载）

```
用户 Commit 代码
    ↓
[amplication-server] BuildService.create()
    │
    │  创建 Build(status=Running, gitStatus=Waiting)
    │  创建 Action + 仅 1 个初始 Step:
    │    └─ ADD_TO_QUEUE (创建即 Success)
    │
    │  其余 Step 由 actionService.run() 在各流程入口动态创建:
    │    └─ DOWNLOAD_PRIVATE_PLUGINS  ← downloadPrivatePlugins() 时创建
    │    └─ GENERATE_APPLICATION      ← generate() 时创建
    │    └─ PUSH_TO_GIT_PROVIDER      ← saveToGitProvider() 时创建
    │
    ├─ 有 Private Plugins?
    │   ├─ 是 → downloadPrivatePlugins()
    │   │        │
    │   │        │  actionService.run() → 创建 ActionStep DOWNLOAD_PRIVATE_PLUGINS (Running)
    │   │        │
    │   │        ▼
    │   │      Kafka: DOWNLOAD_PRIVATE_PLUGINS_REQUEST
    │   │        │
    │   │        ▼
    │   │      [git-sync-manager] 下载私有插件到共享存储
    │   │        │
    │   │        ├─ 成功 → Kafka: DOWNLOAD_PRIVATE_PLUGINS_SUCCESS
    │   │        │     │
    │   │        │     ▼
    │   │        │   BuildController.onDownloadPrivatePluginsSuccess()
    │   │        │     │
    │   │        │     ▼
    │   │        │   BuildService.onDownloadPrivatePluginSuccess()
    │   │        │     │
    │   │        │     │ ① generate() → 创建 ActionStep GENERATE_APPLICATION (Running)
    │   │        │     │ ② ActionStep DOWNLOAD_PRIVATE_PLUGINS: Success
    │   │        │     │
    │   │        │     └──→ 进入 generate() 流程 ──┐
    │   │        │                                  │
    │   │        ├─ 失败 → Kafka: DOWNLOAD_PRIVATE_PLUGINS_FAILURE
    │   │        │     │
    │   │        │     ▼
    │   │        │   BuildController.onDownloadPrivatePluginsFailure()
    │   │        │     │
    │   │        │     ▼
    │   │        │   BuildService.onDownloadPrivatePluginFailure()
    │   │        │     │
    │   │        │     │ ① onDownloadPrivatePluginLog() 写入错误日志
    │   │        │     │ ② ActionStep DOWNLOAD_PRIVATE_PLUGINS: Failed
    │   │        │     │ ③ Build.status = Failed
    │   │        │     │ ④ Build.gitStatus = Canceled
    │   │        │     │ （此时 GENERATE_APPLICATION Step 尚未创建，Redis 无 JobStatus 记录）
    │   │        │     │
    │   │        │     └──→ 构建终止，不再进入后续流程
    │   │        │
    │   └─ 否 ──────────────────────────────────────┘
    │                                                │
    └────────────────────────────────────────────────┘
                       │
                       ▼
           generate() 开始代码生成
                       │
                       │  actionService.run() → 创建 ActionStep GENERATE_APPLICATION (Running)
                       │
                       ▼
组装 DSGResourceData → 保存到共享存储
                       │
                       ▼
发送 Kafka: CODE_GENERATION_REQUEST_TOPIC
                       │
                       ▼
[amplication-build-manager] BuildRunnerService.runBuild()
                       │
                       ▼
按业务域拆分 Job（Server + AdminUI，可选）
  → Redis: {buildId}-server = InProgress
  → Redis: {buildId}-admin-ui = InProgress
                       │
                       ▼
调用 DSG Runner (Argo Events HTTP)
                       │
                       ▼
[generator-blueprints] 容器启动执行
                       │
                       ▼
执行代码生成（Context → Plugin Wrapper → Blueprint 插件）
                       │
                       ▼
发送成功/失败回调 → BuildRunnerService.handleDsgJobCompleted()
                       │
                       ▼
更新 Job 状态 (Redis)，聚合所有 Job 状态
  ├─ 全部成功 → Kafka: CODE_GENERATION_SUCCESS
  │                ↓
  │              BuildController.onCodeGenerationSuccess()
  │                ↓
  │              BuildService.saveToGitProvider()
  │                ↓
  │              actionService.run() → 创建 ActionStep PUSH_TO_GIT_PROVIDER (Running)
  │                ↓
  │              ... 推送到 Git ...
  ├─ 任一失败 → Kafka: CODE_GENERATION_FAILURE
  └─ 进行中   → 等待其他 Job
```

### 3.2 关键入口点详解

#### 3.2.1 服务端触发 - BuildService.create()

**文件**：[packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L268-L351)

核心流程：
```typescript
async create(args: CreateBuildArgs): Promise<Build> {
  // 1. 创建 Build 记录，初始状态为 Running / Waiting
  //    同时创建 Action，但仅创建 1 个初始 Step:
  //    - ADD_TO_QUEUE (创建即标记 Success)
  //    其余 Step 由后续 actionService.run() 动态创建
  const build = await this.prisma.build.create({
    data: {
      status: EnumBuildStatus.Running,
      gitStatus: EnumBuildGitStatus.Waiting,
      action: {
        create: {
          steps: {
            create: createInitialStepData(version, args.data.message),
            // ↑ 仅创建 ADD_TO_QUEUE step，状态直接为 Success
          },
        },
      },
      // ...
    },
  });

  // 2. 检查资源类型（仅 Service 和 Component 生成代码）
  if (resource.resourceType !== EnumResourceType.Service &&
      resource.resourceType !== EnumResourceType.Component) {
    return;
  }

  // 3. 有私有插件先下载，否则直接生成
  const resourcePrivatePlugins =
    await this.pluginInstallationService.getInstalledPrivatePluginsForBuild(resourceId);

  if (resourcePrivatePlugins.length > 0) {
    // → actionService.run() 在此动态创建 DOWNLOAD_PRIVATE_PLUGINS step
    await this.downloadPrivatePlugins(logger, build, user, resourcePrivatePlugins);
  } else {
    // → actionService.run() 在此动态创建 GENERATE_APPLICATION step
    await this.generate(logger, build, user);
  }
}
```

#### 3.2.2 私有插件下载请求

**文件**：[packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L623-L737)

```typescript
private async downloadPrivatePlugins(logger, build, user, privatePlugins) {
  return this.actionService.run(
    build.actionId,
    DOWNLOAD_PRIVATE_PLUGINS_STEP_NAME,    // "DOWNLOAD_PRIVATE_PLUGINS"
    DOWNLOAD_PRIVATE_PLUGINS_STEP_MESSAGE, // "Downloading private plugins"
    // ↑ actionService.run() 内部：
    //   1. createStep() → 动态创建 ActionStep DOWNLOAD_PRIVATE_PLUGINS (Running)
    //   2. 执行 stepFunction
    //   3. leaveStepOpenAfterSuccessfulExecution=true → 不自动 complete
    //      Step 保持 Running，等待 Kafka 回调关闭
    async (step) => {
      // 1. 获取每个插件的具体版本号
      const pluginVersions = await this.getPrivatePluginsWithVersion(...);

      // 2. 按插件仓库分组
      const repositoryPlugins = [...]; // 按仓库分组的插件列表

      // 3. 发送 Kafka 事件给 git-sync-manager
      const downloadPrivatePluginsRequest = {
        key: { resourceId },
        value: {
          buildId: build.id,
          resourceId,
          repositoryPlugins: repositoryPlugins,
        },
      };

      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC,
        downloadPrivatePluginsRequest
      );
    },
    true  // ← leaveStepOpenAfterSuccessfulExecution=true，不自动 complete
  );
}
```

#### 3.2.3 私有插件下载成功回调

**Kafka 监听**：[packages/amplication-server/src/core/build/build.controller.ts](packages/amplication-server/src/core/build/build.controller.ts#L160-L173)

```typescript
@EventPattern(KAFKA_TOPICS.DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC)
async onDownloadPrivatePluginsSuccess(@Payload() message) {
  const args = plainToInstance(DownloadPrivatePluginsSuccess.Value, message);
  await this.buildService.onDownloadPrivatePluginSuccess(args);
}
```

**业务处理**：[packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L1012-L1042)

```typescript
public async onDownloadPrivatePluginSuccess(response): Promise<void> {
  const { buildId } = response;

  const step = await this.getBuildStep(buildId, DOWNLOAD_PRIVATE_PLUGINS_STEP_NAME);
  const build = await this.findOne({ where: { id: buildId } });
  const user = await this.userService.findUser({ where: { id: build.userId } });

  const logger = this.logger.child({ buildId, resourceId: build.resourceId, ... });

  // ① generate() 内部调用 actionService.run()
  //    → 动态创建 ActionStep GENERATE_APPLICATION (Running)
  //    → 组装 DSGResourceData、保存到共享存储、发 Kafka 消息
  //    → 因为 leaveStepOpenAfterSuccessfulExecution=true，不会立即 complete
  await this.generate(logger, build, user);

  // ② 后完成 DOWNLOAD_PRIVATE_PLUGINS 步骤
  await this.actionService.complete(step, EnumActionStepStatus.Success);
}
```

> **关键细节**：
>
> 1. **三个异步 Step 都使用 `leaveStepOpen=true`**：
>    - DOWNLOAD_PRIVATE_PLUGINS（第 735 行）也传入了 `true`，发送 Kafka 后保持 Running 等待回调关闭
>    - GENERATE_APPLICATION（第 616 行）也传入了 `true`，发送 Kafka 后保持 Running 等待回调关闭
>    - PUSH_TO_GIT_PROVIDER（第 1340 行）也传入了 `true`，发送 Kafka 后保持 Running 等待回调关闭
>
> 2. **DOWNLOAD_PRIVATE_PLUGINS 的独有特性**：
>    - 成功回调中**先启动下一步（generate），后 complete 自己**
>    - 顺序：① `generate()` → ② `actionService.complete(DOWNLOAD_PRIVATE_PLUGINS, Success)`
>    - 这意味着 GENERATE_APPLICATION Step 创建时，DOWNLOAD_PRIVATE_PLUGINS Step 仍处于 Running 状态
>
> 3. **GENERATE_APPLICATION 的回调顺序不同**：
>    - 先 complete 自己（GENERATE_APPLICATION → Success）
>    - 再触发下一步（saveToGitProvider）
>    - 因此 PUSH_TO_GIT_PROVIDER Step 创建时，GENERATE_APPLICATION Step 已经是 Success 状态

**失败回调**：[packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L1044-L1081)

```typescript
public async onDownloadPrivatePluginFailure(response): Promise<void> {
  const { buildId } = response;

  // ① 记录错误日志到 ActionStep
  await this.onDownloadPrivatePluginLog({
    buildId, level: "error", message: response.errorMessage, ...
  });

  // ② ActionStep DOWNLOAD_PRIVATE_PLUGINS → Failed
  const step = await this.getBuildStep(buildId, DOWNLOAD_PRIVATE_PLUGINS_STEP_NAME);
  await this.actionService.complete(step, EnumActionStepStatus.Failed);

  // ③ Build.status → Failed, Build.gitStatus → Canceled
  await this.updateBuildStatuses(
    buildId,
    EnumBuildStatus.Failed,
    EnumBuildGitStatus.Canceled
  );
}
```

**下载失败时各层状态变化的详细说明**：

| 层级 | 状态变化 | 说明 |
|------|---------|------|
| **Build.status** | `Running` → `Failed` | 数据库 `prisma.build` 记录整体构建失败 |
| **Build.gitStatus** | `Waiting` → `Canceled` | 代码未生成，Git 推送被取消 |
| **ActionStep DOWNLOAD_PRIVATE_PLUGINS** | `Running` → `Failed` | 插件下载步骤失败，错误信息写入步骤日志 |
| **ActionStep GENERATE_APPLICATION** | 未创建 / 仍为初始状态 | 因为 `generate()` 从未被调用，此步骤不会启动 |
| **ActionStep PUSH_TO_GIT_PROVIDER** | 未创建 / 仍为初始状态 | 前置步骤失败，此步骤不会启动 |
| **Redis JobStatus** | **无记录** | 插件下载失败发生在 `generate()` 之前，Build Manager 从未收到 `CODE_GENERATION_REQUEST`，因此不会拆分 Job，Redis 中没有任何 JobStatus 记录 |
| **DSG 容器** | 未启动 | 代码生成容器从未被调度 |

> **关键**：下载失败时，构建流程在 `amplication-server` 层即终止，不会产生任何 Kafka 事件发往 `amplication-build-manager`，因此 Job 拆分、Redis 状态、DSG 容器执行等后续环节全部不会发生。

#### 3.2.4 代码生成触发 - BuildService.generate()

**文件**：[packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L568-L618)

```typescript
private async generate(logger, build, user) {
  return this.actionService.run(
    build.actionId,
    GENERATE_STEP_NAME,      // "GENERATE_APPLICATION"
    GENERATE_STEP_MESSAGE,   // "Generating Application"
    // ↑ actionService.run() 内部：
    //   1. createStep() → 动态创建 ActionStep GENERATE_APPLICATION (Running)
    //   2. 执行 stepFunction（组装数据、保存、发 Kafka）
    //   3. 因为 leaveStepOpenAfterSuccessfulExecution=true，
    //      不自动 complete，Step 保持 Running 等待 DSG 回调关闭
    async (step) => {
      // 1. 组装完整的 DSGResourceData（包含 entities、roles、modules、plugins 等）
      const dsgResourceData = await this.getDSGResourceData(
        resource, buildId, buildVersion, user
      );

      // 2. 保存到共享存储
      await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);

      // 3. 发送 Kafka 事件（轻量级，只传 resourceId 和 buildId）
      const codeGenerationEvent = {
        key: null,
        value: { resourceId, buildId },
      };
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
        codeGenerationEvent
      );
    },
    true  // ← leaveStepOpenAfterSuccessfulExecution=true，不自动 complete
  );
}
```

**DSGResourceData 保存路径**：`/amplication-data/dsg-resource-data/{buildId}/resource-data.json`

#### 3.2.5 Build Manager 处理

**文件**：[packages/amplication-build-manager/src/build-runner/build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159)

```typescript
async runBuild(resourceId: string, buildId: string) {
  // 1. 从共享存储读取 DSGResourceData
  const dsgResourceData = await this.readDsgResourceDataFromSharedStorage(buildId);

  // 2. 获取代码生成器版本
  const codeGeneratorVersion =
    await this.codeGeneratorService.getCodeGeneratorVersion(...);

  // 3. 按业务域拆分为多个 Job（核心机制！）
  const jobs = await this.buildJobsHandlerService.splitBuildsIntoJobs(
    dsgResourceData, buildId, codeGeneratorVersion
  );

  // 4. 逐个执行 Job
  for (const [jobBuildId, data] of jobs) {
    await this.runJob(resourceId, jobBuildId, data, ...);
  }
}
```

#### 3.2.6 DSG 容器入口

**文件**：[packages/generator-blueprints/src/main.ts](packages/generator-blueprints/src/main.ts)

```typescript
// 通过环境变量控制：
// - BUILD_SPEC_PATH: 输入 JSON 路径
// - BUILD_OUTPUT_PATH: 输出路径
// - BUILD_MANAGER_URL: Build Manager 回调地址
// - RESOURCE_ID, BUILD_ID

generateCode().catch(async (err) => {
  logger.error(err);
  process.exit(1);
});
```

---

## 四、构建作业状态管理与关联

### 4.1 作业拆分机制（Build Jobs Handler）

**核心文件**：[packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts](packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts)

#### 4.1.1 拆分条件

```typescript
async splitBuildsIntoJobs(dsgResourceData, buildId, codeGeneratorVersion) {
  // 满足以下条件才拆分：
  const shouldSplitBuild =
    dsgResourceData.resourceType === EnumResourceType.Service &&
    codeGeneratorVersion !== "latest-local" &&
    this.codeGeneratorService.compareVersions(
      codeGeneratorVersion,
      this.minDsgVersionToSplitBuild // 最小支持拆分的 DSG 版本
    ) >= 0;

  const jobs: ResourceTuple[] = [];

  if (shouldSplitBuild) {
    const { generateServer, generateAdminUI } =
      dsgResourceData.resourceInfo.settings;

    // 拆分 Server Job
    if (generateServer) {
      const serverDSGResourceData = cloneDeep(dsgResourceData);
      serverDSGResourceData.resourceInfo.settings.adminUISettings.generateAdminUI = false;
      const jobBuildId = this.generateJobBuildId(buildId, EnumDomainName.Server);
      await this.setJobStatus(jobBuildId, EnumJobStatus.InProgress);
      jobs.push([jobBuildId, serverDSGResourceData]);
    }

    // 拆分 AdminUI Job
    if (generateAdminUI) {
      const adminUiDSGResourceData = cloneDeep(dsgResourceData);
      adminUiDSGResourceData.resourceInfo.settings.serverSettings.generateServer = false;
      const jobBuildId = this.generateJobBuildId(buildId, EnumDomainName.AdminUI);
      await this.setJobStatus(jobBuildId, EnumJobStatus.InProgress);
      jobs.push([jobBuildId, adminUiDSGResourceData]);
    }
  } else {
    // 不拆分，单 Job
    await this.setJobStatus(buildId, EnumJobStatus.InProgress);
    jobs.push([buildId, dsgResourceData]);
  }

  return jobs;
}
```

**Job ID 命名规则**：`{buildId}-{domain}`，例如：`clx123-server`、`clx123-admin-ui`

**状态枚举**：[packages/amplication-build-manager/src/types.ts](packages/amplication-build-manager/src/types.ts#L6-L10)

```typescript
export enum EnumJobStatus {
  InProgress = "in-progress",
  Success = "success",
  Failure = "failure",
}
```

#### 4.1.2 Redis 状态存储

每个 Build 的状态存储在 Redis 中，结构如下：

```typescript
// Key: buildId (无后缀)
// Value: { [jobBuildId]: EnumJobStatus }
type RedisValue = Record<JobBuildId<BuildId>, EnumJobStatus>;

// 示例（拆分后的 Build）:
// Key: "clx123"
// Value:
{
  "clx123-server": "in-progress",
  "clx123-admin-ui": "success"
}
```

**设置 Job 状态**：

```typescript
async setJobStatus(jobBuildId: string, status: EnumJobStatus): Promise<void> {
  const key = this.extractBuildId(jobBuildId);  // 提取原始 buildId
  const currentVal = await this.redisService.get<RedisValue>(key);
  const newVal = {
    ...currentVal,
    [jobBuildId]: status,
  };
  await this.redisService.set<RedisValue>(key, newVal);
}
```

#### 4.1.3 聚合状态计算

```typescript
async getBuildStatus(key: BuildId): Promise<EnumJobStatus> {
  const buildValue = await this.redisService.get<RedisValue>(key);
  const jobsStatus = Object.values(buildValue);

  // 全部成功 → Success
  const allSucceeded = jobsStatus.every(
    (status) => status === EnumJobStatus.Success
  );
  if (allSucceeded) return EnumJobStatus.Success;

  // 任一失败 → Failure
  const atLeaseOneFailed = jobsStatus.some(
    (status) => status === EnumJobStatus.Failure
  );
  if (atLeaseOneFailed) return EnumJobStatus.Failure;

  // 任一进行中 → InProgress
  const atLeastOneInProgress = jobsStatus.some(
    (status) => status === EnumJobStatus.InProgress
  );
  if (atLeastOneInProgress) return EnumJobStatus.InProgress;
}
```

#### 4.1.4 Job 完成后的状态流转

**文件**：[packages/amplication-build-manager/src/build-runner/build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L208-L255)

```typescript
async handleDsgJobCompleted(resourceId: string, jobBuildId: string) {
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);
  let otherJobsHaveNotFailed = true;

  try {
    // 1. 检查当前整体状态（避免重复处理失败）
    const currentBuildStatus =
      await this.buildJobsHandlerService.getBuildStatus(buildId);
    otherJobsHaveNotFailed = currentBuildStatus !== EnumJobStatus.Failure;

    // 2. 从 Job 目录复制代码到 Artifact 目录
    await this.copyFromJobToArtifact(resourceId, jobBuildId);

    // 3. 更新当前 Job 状态为 Success
    await this.buildJobsHandlerService.setJobStatus(
      jobBuildId, EnumJobStatus.Success
    );

    // 4. 再次计算整体状态
    const buildStatus =
      await this.buildJobsHandlerService.getBuildStatus(buildId);

    if (buildStatus === EnumJobStatus.InProgress) {
      // 还有 Job 在运行，等待
      return;
    }

    if (buildStatus === EnumJobStatus.Success) {
      // 所有 Job 都成功了
      const dsgResourceData =
        await this.buildJobsHandlerService.extractDsgResourceData(jobBuildId);

      // 如果有 Packages 需要生成，发送给 Package Manager
      if (dsgResourceData.packages?.length > 0 && this.enablePackageManager) {
        await this.generatePackages(buildId, resourceId, dsgResourceData);
      } else {
        // 没有 Packages，直接完成
        await this.codeGenerationAndPackagesCompleted(jobBuildId);
      }
    }
  } catch (error) {
    if (otherJobsHaveNotFailed) {
      await this.emitCodeGenerationFailure(buildId, error.message);
    }
  }
}
```

### 4.2 状态关联全景图

```
┌─────────────────────────────────────────────────────────────┐
│                     Build 生命周期                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Build.status (DB: prisma.build)                            │
│  ├─ Running      ← 初始创建                                  │
│  ├─ Failed       ← 任一步骤失败                              │
│  └─ Completed    ← PUSH_TO_GIT 成功                          │
│                                                             │
│  Build.gitStatus (DB: prisma.build)                         │
│  ├─ Waiting      ← 初始创建                                  │
│  ├─ Canceled     ← 代码生成失败                              │
│  ├─ Failed       ← Push to Git 失败                          │
│  └─ Completed    ← Push to Git 成功                          │
│                                                             │
│  ActionStep (DB: prisma.actionStep) — 动态创建               │
│  ├─ ADD_TO_QUEUE             ← Build.create() 时创建(Success)│
│  ├─ DOWNLOAD_PRIVATE_PLUGINS ← downloadPrivatePlugins()时创建│
│  │   (仅在有私有插件时才创建)                                 │
│  ├─ GENERATE_APPLICATION     ← generate() 时创建             │
│  │   (创建后保持 Running，等待 DSG 回调关闭)                  │
│  └─ PUSH_TO_GIT_PROVIDER     ← saveToGitProvider() 时创建   │
│      (仅在有 Git 配置时才创建)                                │
│                                                             │
│  EnumJobStatus (Redis) — 由 Build Manager 管理              │
│  ├─ {buildId}-server     ← InProgress / Success / Failure   │
│  └─ {buildId}-admin-ui   ← InProgress / Success / Failure   │
│  (仅在 DSG 容器启动后才存在)                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### ActionStep 动态创建机制

每个 ActionStep 并非在 Build 创建时一次性全部创建，而是由 `actionService.run()` 在各流程入口动态创建：

**`actionService.run()` 的内部逻辑**（[action.service.ts#L275-L295](packages/amplication-server/src/core/action/action.service.ts#L275-L295)）：

```typescript
async run(actionId, stepName, message, stepFunction, leaveStepOpenAfterSuccessfulExecution = false) {
  // 1. 动态创建 Step（status=Running）
  const step = await this.createStep(actionId, stepName, message);

  try {
    // 2. 执行业务函数
    const result = await stepFunction(step);

    // 3. 如果不需要保持打开，自动标记 Success
    if (!leaveStepOpenAfterSuccessfulExecution) {
      await this.complete(step, EnumActionStepStatus.Success);
    }
    return result;
  } catch (error) {
    // 4. 出错自动标记 Failed
    await this.log(step, EnumActionLogLevel.Error, error.message);
    await this.complete(step, EnumActionStepStatus.Failed);
    throw error;
  }
}
```

| ActionStep | 创建时机 | leaveStepOpen | 由谁 complete | 特殊行为 |
|-----------|---------|---------------|-------------|---------|
| ADD_TO_QUEUE | `Build.create()` | false | 自身（创建即 Success） | 同步完成 |
| DOWNLOAD_PRIVATE_PLUGINS | `downloadPrivatePlugins()` | true | `onDownloadPrivatePluginSuccess()`/`Failure()` | **先启动下一步，再 complete 自己**<br>回调中顺序：① `generate()` → ② `complete(Success)` |
| GENERATE_APPLICATION | `generate()` | true | `onCodeGenerationSuccess()`/`Failure()` | 先 complete 自己，再触发下一步（saveToGitProvider） |
| PUSH_TO_GIT_PROVIDER | `saveToGitProvider()` | true | Git Sync Manager Kafka 回调 | 最后一步，无后续 |

---

## 五、Blueprint 代码生成流程

### 5.1 生成流程总览

**核心入口文件**：[packages/generator-blueprints/src/create-data-service.ts](packages/generator-blueprints/src/create-data-service.ts)

```
createDataService()
    ↓
prepareContext()           # 准备上下文数据
    ├─ registerPlugins()   # 注册所有插件（包括 Private Plugins）
    ├─ resolveLookupFields() # 解析关联字段
    ├─ prepareModuleActionsAndDtos() # 组装 Module 数据
    └─ prepareEntityActions()  # 组装 Entity Action
    ↓
createBlueprint()          # 核心生成函数
    └─ createModulesFiles()
        └─ createModuleFiles()  # 每个 Module 的生成
            ↓ （通过 Plugin Wrapper 调用插件）
Plugin before 事件 → 默认行为 → Plugin after 事件
    ↓
context.files 收集所有文件
    ↓
normalize path (Unix 格式)
    ↓
返回 FileMap
```

### 5.2 上下文准备 - prepareContext

**文件**：[packages/generator-blueprints/src/prepare-context.ts](packages/generator-blueprints/src/prepare-context.ts)

**DsgContext 单例** 定义在 [packages/generator-blueprints/src/dsg-context.ts](packages/generator-blueprints/src/dsg-context.ts)：

```typescript
class DsgContext {
  public appInfo!: types.AppInfo;           // 应用信息
  public entities: types.Entity[] = [];     // 实体列表
  public roles: types.Role[] = [];          // 角色列表
  public files: FileMap<IAstNode>;           // 生成的文件集合
  public plugins: types.blueprintTypes.PluginMap; // 插件映射
  public moduleActionsAndDtoMap: ModuleActionsAndDtosMap;
  public entityActionsMap: types.EntityActionsMap;
  public serviceTopics: types.ServiceTopics[];

  // 工具函数
  public utils: {
    skipDefaultBehavior: boolean;   // 插件可设置跳过默认行为
    abortGeneration: (msg) => void; // 中止生成
    importStaticFiles: ...;         // 导入静态文件
    replacePlaceholders: ...;       // 替换占位符
  };
}
```

**关键数据处理**：

1. **Module Actions & DTOs 组装**：
   - 将 ModuleContainer、ModuleAction、ModuleDto 关联起来
   - 解析 DTO 属性之间的引用关系
   - 为 GraphQL 生成添加装饰器（ArgsType、InputType、ObjectType）

2. **Entity Actions 组装**：
   - 为每个 Entity 生成默认 Actions（Create/Read/Update/Delete/Search）
   - 为关联字段生成默认 Actions（ChildrenFind/ChildrenConnect 等）
   - 支持自定义 Action（Custom）

### 5.3 Plugin Wrapper 机制（Golden Path 核心）

**文件**：[packages/generator-blueprints/src/plugin-wrapper.ts](packages/generator-blueprints/src/plugin-wrapper.ts)

这是实现 Golden Path 的核心机制！插件可以在**每个生成事件的 before 和 after 阶段**介入：

```typescript
const pluginWrapper: PluginWrapper = async (func, event, args) => {
  const context = DsgContext.getInstance;

  // 1. 执行所有 before 插件（管道式）
  const updatedEventParams = beforePlugins
    ? await beforeEventsPipe(...beforePlugins)(context, args)
    : args;

  // 2. 执行默认行为（插件可设置 skipDefaultBehavior 跳过）
  const defaultBehaviorModules = await defaultBehavior(
    context, func, updatedEventParams
  );

  // 3. 执行所有 after 插件（管道式）
  const finalFiles = afterPlugins
    ? await afterEventsPipe(...afterPlugins)(context, args, defaultBehaviorModules)
    : defaultBehaviorModules;

  // 4. 将文件合并到上下文
  for (const file of finalFiles.getAll()) {
    context.files.replace(file, file);
  }

  return finalFiles;
};
```

### 5.4 Blueprint 事件列表

目前支持的 Blueprint 事件（用于插件扩展）：

| 事件名称 | 触发时机 | 用途 |
|---------|---------|------|
| `createBlueprint` | 整个 Blueprint 生成开始时 | 全局初始化或后处理 |
| `createModules` | 生成所有 Modules 时 | 批量处理 Modules |
| `createModule` | 生成单个 Module 时 | 处理单个 Module |

**注意**：目前 `create-module.ts` 的默认行为是空的，实际的代码生成完全由 **Blueprint 插件** 实现！

```typescript
// packages/generator-blueprints/src/blueprint/create-module.ts
async function createModuleInternal(eventParams) {
  // do nothing - the event is handled by the blueprint plugin
  return fileMap;
}
```

### 5.5 私有插件加载路径

**文件**：[packages/generator-blueprints/src/register-plugin.ts](packages/generator-blueprints/src/register-plugin.ts#L22-L35)

```typescript
const getPrivatePluginPath = (pluginId: string) => {
  const buildSpecPath = process.env.BUILD_SPEC_PATH;
  const buildJobFolder = buildSpecPath?.replace("/input.json", "");

  // 私有插件被下载到的路径：
  // {buildJobFolder}/dsg-assets/private-plugins/{pluginId}
  return join(
    buildJobFolder,
    DSG_ASSETS_FOLDER,        // "dsg-assets"
    PRIVATE_PLUGINS_FOLDER,   // "private-plugins"
    pluginId
  );
};
```

---

## 六、结果落点

### 6.1 生成结果存储

#### 6.1.1 DSG 容器内生成

**文件**：[packages/generator-blueprints/src/generate-code.ts](packages/generator-blueprints/src/generate-code.ts#L20-L50)

```typescript
async function writeModules(files: FileMap<IAstNode>, destination: string) {
  // 创建基础目录
  await mkdir(destination, { recursive: true });

  // 遍历所有文件写入
  for await (const file of files.getAll()) {
    const filePath = join(destination, file.path);
    await mkdir(dirname(filePath), { recursive: true });

    // 调用 code.toString() 触发每个 AstNode 的正确 writer
    await writeFile(filePath, file.code.toString(), {
      encoding: getFileEncoding(filePath),
      flag: "wx",  // 只写新模式，文件已存在则失败
    });
  }
}
```

**输出路径**：由环境变量 `BUILD_OUTPUT_PATH` 指定（在 Job 目录内）

#### 6.1.2 Job 结果复制到 Artifact

**文件**：[packages/amplication-build-manager/src/build-runner/build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L372-L396)

```typescript
async copyFromJobToArtifact(resourceId: string, jobBuildId: string) {
  const buildId = this.buildJobsHandlerService.extractBuildId(jobBuildId);

  const jobPath = join(
    this.configService.get(Env.DSG_JOBS_BASE_FOLDER),
    jobBuildId,
    this.configService.get(Env.DSG_JOBS_CODE_FOLDER)  // "generated"
  );

  const artifactPath = join(
    this.configService.get(Env.BUILD_ARTIFACTS_BASE_FOLDER),
    resourceId,
    buildId
  );

  // 如果有多个 Job（Server + AdminUI），它们的代码会被合并到同一个 artifact 目录
  await copy(jobPath, artifactPath);
}
```

**各阶段路径汇总**：

| 阶段 | 路径 |
|------|------|
| DSGResourceData 输入 | `/amplication-data/dsg-resource-data/{buildId}/resource-data.json` |
| DSG Job 工作目录 | `/dsg-jobs/{jobBuildId}/` |
| DSG Job 代码输出 | `/dsg-jobs/{jobBuildId}/generated/` |
| 私有插件下载路径 | `/dsg-jobs/{jobBuildId}/dsg-assets/private-plugins/{pluginId}/` |
| 最终 Artifact | `/build-artifacts/{resourceId}/{buildId}/` |

### 6.2 成功回调与后续流程

**文件**：[packages/amplication-build-manager/src/build-runner/build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L93-L107)

```typescript
async codeGenerationAndPackagesCompleted(buildIdOrJobBuildId: string) {
  const successEvent: CodeGenerationSuccess.KafkaEvent = {
    key: null,
    value: { buildId },
  };

  await this.producerService.emitMessage(
    KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC,
    successEvent
  );
}
```

**Server 端成功处理**：[packages/amplication-server/src/core/build/build.controller.ts](packages/amplication-server/src/core/build/build.controller.ts#L87-L101)

```typescript
@EventPattern(KAFKA_TOPICS.CODE_GENERATION_SUCCESS_TOPIC)
async onCodeGenerationSuccess(@Payload() message) {
  const args = plainToInstance(CodeGenerationSuccess.Value, message);

  // 1. 推送到 Git Provider（如果配置了）
  await this.buildService.saveToGitProvider(args.buildId);

  // 2. 完成 GENERATE_APPLICATION 步骤，发送用户通知
  await this.buildService.onCodeGenerationSuccess(args.buildId);
}
```

**BuildService 成功处理**：[packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts#L450-L495)

```typescript
async onCodeGenerationSuccess(buildId: string) {
  // 1. 完成 GENERATE_APPLICATION 步骤
  await this.actionService.complete(step, EnumActionStepStatus.Success);

  // 2. 发送 USER_BUILD_TOPIC 事件（用户通知、统计等）
  this.kafkaProducerService.emitMessage(
    KAFKA_TOPICS.USER_BUILD_TOPIC,
    {
      key: {},
      value: {
        commitId, resourceId, buildId, projectId, ...
      },
    }
  );

  // 3. 如果配置了 Git，PUSH_TO_GIT 步骤将继续执行
  // ...
}
```

---

## 七、Golden Path 实践：如何编写 Blueprint 插件

### 7.1 插件基本结构

```typescript
import {
  blueprintTypes,
  blueprintPluginEventsTypes
} from "@amplication/code-gen-types";

class MyGoldenPathPlugin implements blueprintTypes.AmplicationPlugin {
  register(): blueprintPluginEventsTypes.BlueprintEvents {
    return {
      [blueprintTypes.BlueprintEventNames.createModule]: {
        before: this.beforeCreateModule.bind(this),
        after: this.afterCreateModule.bind(this),
      },
      [blueprintTypes.BlueprintEventNames.createBlueprint]: {
        after: this.afterCreateBlueprint.bind(this),
      },
    };
  }

  async beforeCreateModule(context, eventParams) {
    // 在生成 Module 前修改参数
    context.logger.info("Applying Golden Path standards...");
    return eventParams;
  }

  async afterCreateModule(context, eventParams, files) {
    // 在生成 Module 后添加/修改文件
    const myFile = createMyStandardFile();
    files.set(myFile);
    return files;
  }

  async afterCreateBlueprint(context, eventParams, files) {
    // 在整个 Blueprint 生成后添加全局文件
    const readme = createGoldenPathReadme();
    files.set(readme);
    return files;
  }
}
```

### 7.2 插件注册流程

**文件**：[packages/generator-blueprints/src/register-plugin.ts](packages/generator-blueprints/src/register-plugin.ts)

```typescript
// 1. 从 npm 或本地路径导入插件
const func = await import(packageName);

// 2. 实例化并调用 register() 获取事件映射
const initializeClass = new pluginFunc();
const pluginEvents = initializeClass.register();

// 3. 将 before/after 函数按事件归类到 PluginMap
pluginMap[eventKey] = {
  before: [...],  // 该事件的所有 before 插件
  after: [...],   // 该事件的所有 after 插件
};
```

---

## 八、关键文件索引

| 模块 | 文件路径 | 说明 |
|------|---------|------|
| **Blueprint 模型** | [packages/amplication-server/src/models/Blueprint.ts](packages/amplication-server/src/models/Blueprint.ts) | Blueprint 数据模型 |
| **Blueprint 服务** | [packages/amplication-server/src/core/blueprint/blueprint.service.ts](packages/amplication-server/src/core/blueprint/blueprint.service.ts) | Blueprint CRUD 逻辑 |
| **Service Template** | [packages/amplication-server/src/core/resource/serviceTemplate.service.ts](packages/amplication-server/src/core/resource/serviceTemplate.service.ts) | 服务模板管理（核心） |
| **Build 服务** | [packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts) | 构建触发入口 |
| **Action 服务** | [packages/amplication-server/src/core/action/action.service.ts](packages/amplication-server/src/core/action/action.service.ts) | ActionStep 动态创建与状态管理 |
| **Build Controller** | [packages/amplication-server/src/core/build/build.controller.ts](packages/amplication-server/src/core/build/build.controller.ts) | Kafka 事件监听与回调 |
| **Build Runner** | [packages/amplication-build-manager/src/build-runner/build-runner.service.ts](packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | DSG 任务调度 |
| **Build Jobs Handler** | [packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts](packages/amplication-build-manager/src/build-job-handler/build-job-handler.service.ts) | 作业拆分与状态管理 |
| **Build Types** | [packages/amplication-build-manager/src/types.ts](packages/amplication-build-manager/src/types.ts) | 类型定义（JobStatus 等） |
| **DSG 主入口** | [packages/generator-blueprints/src/create-data-service.ts](packages/generator-blueprints/src/create-data-service.ts) | 代码生成主函数 |
| **DSG Context** | [packages/generator-blueprints/src/dsg-context.ts](packages/generator-blueprints/src/dsg-context.ts) | DSG 单例上下文 |
| **Plugin Wrapper** | [packages/generator-blueprints/src/plugin-wrapper.ts](packages/generator-blueprints/src/plugin-wrapper.ts) | 插件事件执行管道 |
| **Plugin 注册** | [packages/generator-blueprints/src/register-plugin.ts](packages/generator-blueprints/src/register-plugin.ts) | 插件加载与注册 |
| **Blueprint 生成** | [packages/generator-blueprints/src/blueprint/create-blueprint.ts](packages/generator-blueprints/src/blueprint/create-blueprint.ts) | Blueprint 核心生成函数 |
| **Modules 生成** | [packages/generator-blueprints/src/blueprint/create-modules.ts](packages/generator-blueprints/src/blueprint/create-modules.ts) | Modules 批量生成 |
| **单个 Module** | [packages/generator-blueprints/src/blueprint/create-module.ts](packages/generator-blueprints/src/blueprint/create-module.ts) | 单个 Module 生成（空实现，插件接管） |

---

## 九、总结

### 9.1 Blueprint 编排核心机制

1. **定义层**：Blueprint 作为资源元模型，定义类型、属性和关系约束
2. **模板层**：Service Template 基于 Blueprint，封装完整的配置、插件和版本管理
3. **触发层**：Commit → Build → Private Plugins Download → Kafka → DSG 容器
4. **作业层**：按业务域拆分 Job，Redis 管理 Job 状态，聚合后决定整体成败
5. **生成层**：Context 准备 → Plugin Wrapper → 事件管道 → Blueprint 插件
6. **扩展层**：通过 before/after 插件实现 Golden Path 标准嵌入
7. **结果层**：FileMap → Job 目录 → Artifact 目录 → Git

### 9.2 Golden Path 的实现方式

Golden Path 不是一个具体的代码模块，而是一个**完整的架构模式**，通过以下组件协同实现：

| 组件 | 作用 |
|------|------|
| **Blueprint** | 定义资源的元模型和约束，确保所有资源遵循相同的结构 |
| **Service Template** | 封装最佳实践配置、推荐插件集，实现一键创建标准化资源 |
| **Private Plugins** | 随 Build 自动下载执行，在代码生成的各个阶段注入团队规范 |
| **Plugin Wrapper** | 提供 before/after 扩展点，支持管道式处理和跳过默认行为 |
| **模板版本管理** | 支持模板升级和变更合并，确保资源持续符合最新标准 |

### 9.3 私有插件下载与代码生成的时序关联

#### 成功路径

```
用户 Commit
    │
    ▼
BuildService.create()
    │
    │  创建 Build + Action + ADD_TO_QUEUE Step(创建即 Success)
    │
    ├─ 有私有插件?
    │   ├─ 是 → downloadPrivatePlugins()
    │   │        │
    │   │        │  actionService.run() → 动态创建 DOWNLOAD_PRIVATE_PLUGINS Step(Running)
    │   │        │
    │   │        ▼
    │   │      Kafka: DOWNLOAD_PRIVATE_PLUGINS_REQUEST
    │   │        │
    │   │        ▼
    │   │      git-sync-manager 下载插件到共享存储
    │   │        │
    │   │        ▼
    │   │      Kafka: DOWNLOAD_PRIVATE_PLUGINS_SUCCESS
    │   │        │
    │   │        ▼
    │   │      BuildService.onDownloadPrivatePluginSuccess()
    │   │        │
    │   │        │ ① generate() → 动态创建 GENERATE_APPLICATION Step(Running)
    │   │        │ ② DOWNLOAD_PRIVATE_PLUGINS Step: Success
    │   │        │
    │   │        └──→ 进入 generate() ──┐
    │   │                              │
    │   └─ 否 ────────────────────────┘
    │                                 │
    └─────────────────────────────────┘
                      │
                      ▼
              generate() 开始代码生成
                      │
                      ▼
              ... DSG 完成 → CODE_GENERATION_SUCCESS ...
                      │
                      ▼
              saveToGitProvider()
                      │
                      │  actionService.run() → 动态创建 PUSH_TO_GIT_PROVIDER Step(Running)
                      │
                      ▼
              ... 推送到 Git ...
```

#### 失败路径

```
git-sync-manager 下载插件失败
    │
    ▼
Kafka: DOWNLOAD_PRIVATE_PLUGINS_FAILURE
    │
    ▼
BuildService.onDownloadPrivatePluginFailure()
    │
    │ ① onDownloadPrivatePluginLog() → 写入错误日志
    │ ② ActionStep DOWNLOAD_PRIVATE_PLUGINS: Running → Failed
    │ ③ Build.status: Running → Failed
    │ ④ Build.gitStatus: Waiting → Canceled
    │
    └──→ 构建终止
         （Job 未创建，Redis 无记录，DSG 容器未启动）
```

### 9.4 各阶段失败时的状态对比

| 失败阶段 | Build.status | Build.gitStatus | DOWNLOAD_PRIVATE_PLUGINS Step | GENERATE_APPLICATION Step | Redis JobStatus | 后续是否继续 |
|---------|-------------|-----------------|------------------------------|--------------------------|-----------------|------------|
| **私有插件下载失败** | Failed | Canceled | Failed | 未启动 | 无记录 | 否 |
| **代码生成失败** | Failed | Canceled | Success | Failed | 可能有残留记录 | 否 |
| **Push to Git 失败** | Failed | Failed | Success | Success | 全部 Success | 否 |
| **全部成功** | Completed | Completed | Success | Success | 全部 Success | 是 |
