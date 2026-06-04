# Blueprint 模板编排与 Golden Path 机制解析

## 一、核心概念

### 1.1 Blueprint 是什么？

Blueprint 是 Amplication 中的一个核心概念，用于定义和管理可复用的资源模板。它本质上是一个"蓝图"，定义了：
- 资源类型（Service、Component、MessageBroker 等）
- 代码生成器类型（NodeJs、DotNet、Blueprint）
- 自定义属性（Custom Properties）
- 资源关系（Blueprint Relations）

**核心数据结构** 定义在 [Blueprint.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/models/Blueprint.ts)：

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

**资源类型与代码生成器的对应关系** 在 [blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts#L29-L38) 定义：

| 资源类型 | 支持的代码生成器 |
|---------|----------------|
| Component | Blueprint |
| Service | NodeJs, DotNet |
| MessageBroker | Blueprint |

### 1.2 Golden Path 是什么？

Golden Path（黄金路径）是 Amplication Platform Console 中提出的一个产品概念，指通过以下机制来规范化和标准化开发流程：

1. **Blueprint 模板**：定义标准化的资源结构
2. **Private Plugins**：私有插件，用于集成最佳实践
3. **Live Templates**：实时模板，确保一致性

**相关描述** 在 [PlatformDashboard.tsx](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-client/src/Platform/PlatformDashboard.tsx#L64-L71)：

> "The Platform Console lets teams define, manage, and enforce development standards at scale. It streamlines service creation with Live Templates for consistency and Private Plugins to integrate best practices and Golden Paths."

**实现方式**：Golden Path 主要通过 **Blueprint + Plugin 机制** 实现，开发者通过编写 Blueprint 插件来嵌入团队的最佳实践和标准流程。

---

## 二、触发入口：完整调用链

### 2.1 触发流程图

```
用户 Commit 代码
    ↓
[amplication-server] BuildService.create()
    ↓
生成 Action 和 Steps
    ↓
有 Private Plugins? → 是 → 下载插件（Kafka: DOWNLOAD_PRIVATE_PLUGINS_REQUEST）
    ↓ 否
BuildService.generate()
    ↓
组装 DSGResourceData
    ↓
保存到共享存储 (/amplication-data/dsg-resource-data/{buildId})
    ↓
发送 Kafka 事件 (CODE_GENERATION_REQUEST_TOPIC)
    ↓
[amplication-build-manager] BuildRunnerService.runBuild()
    ↓
按业务域拆分 Job（可选）
    ↓
调用 DSG Runner (Argo Events HTTP)
    ↓
[generator-blueprints] 容器启动执行
    ↓
读取 BUILD_SPEC_PATH / BUILD_OUTPUT_PATH
    ↓
执行代码生成
    ↓
发送成功/失败事件 (CODE_GENERATION_SUCCESS/FAILURE)
```

### 2.2 关键入口点详解

#### 1. 服务端触发 - BuildService.create()

**文件**：[build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/core/build/build.service.ts#L268-L351)

核心流程：
```typescript
async create(args: CreateBuildArgs): Promise<Build> {
  // 1. 创建 Build 记录和 Action 步骤
  const build = await this.prisma.build.create({...});
  
  // 2. 检查资源类型（仅 Service 和 Component 生成代码）
  if (resource.resourceType !== Service && 
      resource.resourceType !== Component) {
    return;
  }

  // 3. 有私有插件先下载，否则直接生成
  if (resourcePrivatePlugins.length > 0) {
    await this.downloadPrivatePlugins(...);
  } else {
    await this.generate(logger, build, user);
  }
}
```

#### 2. 代码生成触发 - BuildService.generate()

**文件**：[build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/core/build/build.service.ts#L568-L618)

核心流程：
```typescript
private async generate(logger, build, user) {
  return this.actionService.run(
    build.actionId,
    GENERATE_STEP_NAME,      // "GENERATE_APPLICATION"
    GENERATE_STEP_MESSAGE,   // "Generating Application"
    async (step) => {
      // 1. 组装 DSGResourceData
      const dsgResourceData = await this.getDSGResourceData(...);
      
      // 2. 保存到共享存储
      await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);
      
      // 3. 发送 Kafka 事件
      const codeGenerationEvent = {
        key: null,
        value: { resourceId, buildId },
      };
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
        codeGenerationEvent
      );
    }
  );
}
```

**DSGResourceData 保存路径**：`/amplication-data/dsg-resource-data/{buildId}/resource-data.json`

#### 3. Build Manager 处理

**文件**：[build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L109-L159)

```typescript
async runBuild(resourceId: string, buildId: string) {
  // 1. 从共享存储读取 DSGResourceData
  const dsgResourceData = await this.readDsgResourceDataFromSharedStorage(buildId);
  
  // 2. 获取代码生成器版本
  const codeGeneratorVersion = await this.codeGeneratorService.getCodeGeneratorVersion(...);
  
  // 3. 按业务域拆分为多个 Job（如果启用）
  const jobs = await this.buildJobsHandlerService.splitBuildsIntoJobs(...);
  
  // 4. 逐个执行 Job
  for (const [jobBuildId, data] of jobs) {
    await this.runJob(resourceId, jobBuildId, data, ...);
  }
}
```

#### 4. DSG 容器入口

**文件**：[main.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/main.ts)

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

## 三、Blueprint 代码生成流程

### 3.1 生成流程总览

**核心入口文件**：[create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/create-data-service.ts)

```
createDataService()
    ↓
prepareContext()           # 准备上下文数据
    ├─ registerPlugins()   # 注册所有插件
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

### 3.2 上下文准备 - prepareContext

**文件**：[prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/prepare-context.ts)

**DsgContext 单例** 定义在 [dsg-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/dsg-context.ts)：

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

### 3.3 Plugin Wrapper 机制

**文件**：[plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/plugin-wrapper.ts)

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

### 3.4 Blueprint 事件列表

目前支持的 Blueprint 事件（用于插件扩展）：

| 事件名称 | 触发时机 | 用途 |
|---------|---------|------|
| `createBlueprint` | 整个 Blueprint 生成开始时 | 全局初始化或后处理 |
| `createModules` | 生成所有 Modules 时 | 批量处理 Modules |
| `createModule` | 生成单个 Module 时 | 处理单个 Module |

**注意**：目前 `create-module.ts` 的默认行为是空的，实际的代码生成完全由 **Blueprint 插件** 实现！

```typescript
// create-module.ts
async function createModuleInternal(eventParams) {
  // do nothing - the event is handled by the blueprint plugin
  return fileMap;
}
```

---

## 四、结果落点

### 4.1 生成结果存储

#### 1. DSG 容器内生成

**文件**：[generate-code.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/generate-code.ts#L20-L50)

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

**输出路径**：由环境变量 `BUILD_OUTPUT_PATH` 指定

#### 2. Job 结果复制到 Artifact

**文件**：[build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L372-L396)

```typescript
async copyFromJobToArtifact(resourceId: string, jobBuildId: string) {
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

  await copy(jobPath, artifactPath);
}
```

**Artifact 最终路径**：`/build-artifacts/{resourceId}/{buildId}/`

### 4.2 成功回调与后续流程

**文件**：[build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts#L93-L107)

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

**Server 端成功处理**：[build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/core/build/build.service.ts#L450-L495)

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
  
  // 3. 触发 Push to Git（如果配置了 Git 集成）
  // ...
}
```

---

## 五、Golden Path 实践：如何编写 Blueprint 插件

### 5.1 插件基本结构

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

### 5.2 插件注册流程

**文件**：[register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/register-plugin.ts)

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

## 六、关键文件索引

| 模块 | 文件路径 | 说明 |
|------|---------|------|
| **Blueprint 模型** | [Blueprint.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/models/Blueprint.ts) | Blueprint 数据模型 |
| **Blueprint 服务** | [blueprint.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/core/blueprint/blueprint.service.ts) | Blueprint CRUD 逻辑 |
| **Build 服务** | [build.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-server/src/core/build/build.service.ts) | 构建触发入口 |
| **Build Runner** | [build-runner.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/amplication-build-manager/src/build-runner/build-runner.service.ts) | DSG 任务调度 |
| **DSG 主入口** | [create-data-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/create-data-service.ts) | 代码生成主函数 |
| **上下文** | [dsg-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/dsg-context.ts) | DSG 单例上下文 |
| **插件包装器** | [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/plugin-wrapper.ts) | 插件事件执行管道 |
| **插件注册** | [register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/register-plugin.ts) | 插件加载与注册 |
| **Blueprint 生成** | [create-blueprint.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/blueprint/create-blueprint.ts) | Blueprint 核心生成函数 |
| **Module 生成** | [create-modules.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/blueprint/create-modules.ts) | Modules 批量生成 |
| **单个 Module** | [create-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/21-amplication/packages/generator-blueprints/src/blueprint/create-module.ts) | 单个 Module 生成（空实现，插件接管） |

---

## 七、总结

### Blueprint 编排核心机制

1. **定义层**：Blueprint 作为资源模板，定义类型、属性和关系
2. **触发层**：Commit → Build → Kafka → DSG 容器
3. **生成层**：Context 准备 → Plugin Wrapper → 事件管道
4. **扩展层**：通过 before/after 插件实现 Golden Path 标准嵌入
5. **结果层**：FileMap → 文件系统 → Artifact → Git

### Golden Path 的实现方式

Golden Path 不是一个具体的代码模块，而是一个**架构模式**：

- **标准化**：通过 Blueprint 定义统一的资源结构
- **可扩展**：通过 Plugin 机制在生成的各个阶段注入团队规范
- **强制执行**：Private Plugins 随 Build 自动下载和执行，无需开发者手动配置
- **灵活定制**：插件可以跳过默认行为、修改参数、添加/修改文件
