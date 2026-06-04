# 插件包生命周期 (Plugin Package Lifecycle)

本文档详细梳理 Amplication 插件包从发现到生效的完整生命周期，涵盖五个核心阶段：**发现 → 安装 → 加载 → 配置注入 → 执行**。重点补全了**安装记录进入构建输入**的完整链路，包括启用筛选、顺序读取、私有插件异步下载触发生成器的完整关系。

---

## 整体流程概览

```
GitHub 插件目录 ──┐
                   ▼
            ┌─────────────┐
            │  发现阶段   │  PluginCatalogService / GitPluginService
            └─────────────┘
                   │
                   ▼  用户点击安装
            ┌─────────────┐
            │  安装阶段   │  PluginInstallationService + PluginOrderService
            └─────────────┘
                   │
                   ▼  触发构建
    ┌───────────────────────────────────┐
    │         构建入口分流              │
    │  BuildService.create()           │
    │  ├─ 有私有插件 → 下载流程        │
    │  └─ 无私有插件 → 直接生成        │
    └───────────────────────────────────┘
                   │
                   ▼
            ┌─────────────┐
            │  加载阶段   │  dynamicPackagesInstallations / registerPlugins
            └─────────────┘
                   │
                   ▼
            ┌─────────────┐
            │ 配置注入    │  prepareContext → DsgContext.plugins
            └─────────────┘
                   │
                   ▼  代码生成事件触发
            ┌─────────────┐
            │  执行阶段   │  pluginWrapper → before/after 事件管道
            └─────────────┘
```

---

## 阶段一：发现阶段 (Discovery)

### 核心职责
从 GitHub 插件目录仓库拉取所有可用插件的元数据，同步到本地数据库，供用户浏览和选择。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `GitPluginService` | [packages/amplication-plugin-api/src/plugin/github-plugin.service.ts](packages/amplication-plugin-api/src/plugin/github-plugin.service.ts) | 从 GitHub 拉取插件目录 |
| `PluginService` | [packages/amplication-plugin-api/src/plugin/plugin.service.ts](packages/amplication-plugin-api/src/plugin/plugin.service.ts) | 将插件元数据存入数据库 |
| `PluginCatalogService` | [packages/amplication-server/src/core/pluginCatalog/pluginCatalog.service.ts](packages/amplication-server/src/core/pluginCatalog/pluginCatalog.service.ts) | 为前端提供插件查询接口 |

### 执行流程

**1. 拉取插件目录清单**
- 调用 [`getPlugins()`](packages/amplication-plugin-api/src/plugin/github-plugin.service.ts#L153-L199) 方法
- 请求 `AMPLICATION_GITHUB_URL` 获取所有插件的 `.yml` 配置文件列表
- 使用 GitHub Token 避免 API 限流

**2. 逐个解析插件配置**
- 通过 [`getPluginConfig()`](packages/amplication-plugin-api/src/plugin/github-plugin.service.ts#L89-L137) 异步生成器遍历每个插件
- 下载并解析 YAML 配置，获取 `pluginId`、`npm`、`github`、`categories` 等元数据

**3. 获取 NPM 包信息**
- 调用 [`fetchNpmData()`](packages/amplication-plugin-api/src/plugin/github-plugin.service.ts#L43-L84)
- 并行获取 NPM 版本信息 (`dist-tags`) 和下载量统计

**4. 同步到数据库**
- 在 [`processCatalogPlugins()`](packages/amplication-plugin-api/src/plugin/plugin.service.ts#L34-L97) 中：
  - `prisma.plugin.createMany()` 批量插入新插件（`skipDuplicates: true`）
  - `prisma.$transaction()` 批量更新现有插件信息
  - 同步插件分类信息到 `Category` 表

**5. 提供查询接口**
- [`getPlugins()`](packages/amplication-server/src/core/pluginCatalog/pluginCatalog.service.ts#L96-L112) 按代码生成器类型过滤插件
- [`getPluginWithLatestVersion()`](packages/amplication-server/src/core/pluginCatalog/pluginCatalog.service.ts#L36-L94) 获取插件详情及最新版本

### 数据模型
插件目录项结构参见 [PluginCatalogItem.ts](packages/amplication-server/src/core/pluginCatalog/dto/PluginCatalogItem.ts)：
```typescript
class PluginCatalogItem {
  pluginId: string;           // 插件唯一标识
  name: string;              // 显示名称
  npm: string;               // NPM 包名
  version: string;           // 版本号
  settings: JsonValue;       // 配置 Schema
  configurations: JsonValue; // 配置项
  categories: string[];      // 分类
  codeGeneratorName: string; // 适用的代码生成器
}
```

---

## 阶段二：安装阶段 (Installation)

### 核心职责
用户将插件安装到指定资源（Service/Blueprint），存储安装配置和执行顺序。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `PluginInstallationService` | [packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) | 插件安装管理 |
| `PluginOrderService` | [packages/amplication-server/src/core/pluginInstallation/pluginOrder.service.ts](packages/amplication-server/src/core/pluginInstallation/pluginOrder.service.ts) | 插件执行顺序管理 |
| `PluginOrder` DTO | [packages/amplication-server/src/core/pluginInstallation/dto/PluginOrder.ts](packages/amplication-server/src/core/pluginInstallation/dto/PluginOrder.ts) | 顺序数据结构 |

### 执行流程

**1. 创建插件安装记录**
- 调用 [`create()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L121-L171) 方法
- **前置校验**：
  - [`validatePluginConfiguration()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L97-L119) 检查认证实体依赖
  - [`findPluginInstallationByPluginId()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L82-L95) 防止重复安装
- 通过 `BlockService` 创建 `PluginInstallation` 类型的 Block 记录

**2. 设置执行顺序**
- 调用 [`setOrder()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L200-L260)
- 新插件默认添加到末尾（`order: -1` 表示追加）
- [`reOrderPlugins()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L28-L56) 处理顺序冲突

**3. 安装数据结构**
参见 [PluginInstallation.ts](packages/amplication-server/src/core/pluginInstallation/dto/PluginInstallation.ts)：
```typescript
class PluginInstallation extends IBlock {
  pluginId: string;           // 关联的目录插件ID
  enabled: boolean;           // 是否启用
  npm: string;               // NPM 包名
  version: string;           // 安装版本
  settings?: JsonValue;      // 用户配置值
  configurations?: JsonValue;// 插件配置项定义
  isPrivate?: boolean;       // 是否为私有插件
}
```

**4. 顺序数据结构**
参见 [PluginOrder.ts](packages/amplication-server/src/core/pluginInstallation/dto/PluginOrder.ts)：
```typescript
class PluginOrder extends IBlock {
  order!: PluginOrderItem[] & JsonValue;  // [{ pluginId: "xxx", order: 1 }, ...]
}
```

---

## 阶段二点五：安装记录 → 构建输入 (Installation → Build Input)

### 核心职责
构建触发时，从数据库读取插件安装记录，经过**启用筛选**、**顺序读取**，组装到 `DSGResourceData` 中作为代码生成器的输入。对于私有插件，需先经过异步下载流程再触发生成器。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `BuildService.create()` | [packages/amplication-server/src/core/build/build.service.ts#L268-L352](packages/amplication-server/src/core/build/build.service.ts#L268-L352) | 构建入口，分流私有/公共插件 |
| `BuildService.getDSGResourceData()` | [packages/amplication-server/src/core/build/build.service.ts#L1384-L1521](packages/amplication-server/src/core/build/build.service.ts#L1384-L1521) | 组装 DSG 输入数据 |
| `BuildController` | [packages/amplication-server/src/core/build/build.controller.ts](packages/amplication-server/src/core/build/build.controller.ts) | Kafka 消息消费者 |
| `PrivatePluginController` | [ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts](ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts) | 私有插件下载处理器 |
| `KAFKA_TOPICS` 枚举 | [libs/schema-registry/src/index.ts#L29-L69](libs/schema-registry/src/index.ts#L29-L69) | Kafka 主题定义 |

---

### 完整数据流 (Installation → Build Input)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     BuildService.create(buildArgs)                      │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
                                      ▼
                ┌──────────────────────────────────────────┐
                │  1. 筛选私有启用插件                     │
                │  getInstalledPrivatePluginsForBuild()   │
                │  → filter(plugin.enabled)               │
                └───────────────────┬──────────────────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │  有私有插件?              │
                        └─────┬───────────────────┬───┘
                              │ Yes               │ No
                              ▼                   ▼
                ┌──────────────────────┐    ┌─────────────────┐
                │ 2. 下载私有插件      │    │ 2. 直接生成代码  │
                │ downloadPrivatePlugins() │   │ generate()      │
                └───────────┬──────────┘    └─────────┬───────┘
                              │                         │
                              ▼                         │
                ┌──────────────────────────────┐        │
                │ 发送 Kafka 请求              │        │
                │ DOWNLOAD_PRIVATE_PLUGINS_    │        │
                │ REQUEST_TOPIC                │        │
                └──────────────────────────────┘        │
                              │                         │
  ┌───────────────────────────┼─────────────────────────┼─────────────┐
  │  git-sync-manager 服务    │                         │             │
  └───────────────────────────┼─────────────────────────┼─────────────┘
                              │                         │
                              ▼                         │
                ┌──────────────────────────────┐        │
                │ PrivatePluginController      │        │
                │ .downloadPrivatePlugins()    │        │
                │ → 实际下载插件到共享存储    │        │
                └───────────┬──────────────────┘        │
                              │                         │
                              ▼                         │
                ┌──────────────────────────────┐        │
                │ 发送 Kafka 成功/失败         │        │
                │ DOWNLOAD_PRIVATE_PLUGINS_    │        │
                │ SUCCESS_TOPIC / FAILURE_TOPIC│        │
                └──────────────────────────────┘        │
                              │                         │
  ┌───────────────────────────┼─────────────────────────┼─────────────┐
  │  amplication-server 服务  │                         │             │
  └───────────────────────────┼─────────────────────────┼─────────────┘
                              │                         │
                              ▼                         │
                ┌──────────────────────────────┐        │
                │ BuildController              │        │
                │ .onDownloadPrivatePluginsSuccess() │   │
                └───────────┬──────────────────┘        │
                              │                         │
                              ▼                         │
                ┌──────────────────────────────┐        │
                │ BuildService                 │◄───────┘
                │ .onDownloadPrivatePluginSuccess() │
                │ → 调用 generate()             │
                └───────────┬──────────────────┘
                              │
                              ▼
                ┌──────────────────────────────┐
                │ 3. 组装 DSG 输入数据         │
                │ generate()                   │
                │ → getDSGResourceData()       │
                └───────────┬──────────────────┘
                              │
                              ▼
                ┌──────────────────────────────┐
                │ 3a. 读取有序插件             │
                │ getOrderedPluginInstallations() │
                │ → 从 PluginOrderService 读顺序 │
                │ → 按 pluginId 匹配安装记录   │
                └───────────┬──────────────────┘
                              │
                              ▼
                ┌──────────────────────────────┐
                │ 3b. 启用筛选                 │
                │ .filter(plugin => plugin.enabled) │
                └───────────┬──────────────────┘
                              │
                              ▼
                ┌──────────────────────────────┐
                │ 3c. 存入 DSGResourceData     │
                │ dsgResourceData.pluginInstallations │
                │   = orderedEnabledPlugins    │
                └───────────┬──────────────────┘
                              │
                              ▼
                ┌──────────────────────────────┐
                │ 4. 保存到共享存储            │
                │ saveDsgResourceDataToSharedStorage() │
                │ → /amplication-data/dsg-resource-data/{buildId}/resource-data.json │
                └───────────┬──────────────────┘
                              │
                              ▼
                ┌──────────────────────────────┐
                │ 5. 发送代码生成请求          │
                │ CODE_GENERATION_REQUEST_TOPIC │
                │ (仅带 buildId, resourceId)   │
                └──────────────────────────────┘
```

---

### 关键代码分析

#### 1. 启用筛选逻辑

**筛选位置 1 - 私有插件构建前筛选**：
[`getInstalledPrivatePluginsForBuild()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L262-L279)
```typescript
async getInstalledPrivatePluginsForBuild(resourceId: string): Promise<PluginInstallation[]> {
  const plugins = await this.findManyBySettings(
    { where: { resource: { id: resourceId } } },
    { path: ["isPrivate"], equals: true }  // 先按 isPrivate 筛选
  );
  return plugins.filter((plugin) => plugin.enabled);  // 再按 enabled 筛选
}
```

**筛选位置 2 - 构建输入时筛选**：
[`getDSGResourceData()`](packages/amplication-server/src/core/build/build.service.ts#L1395-L1399)
```typescript
const orderedPlugins = (
  await this.pluginInstallationService.getOrderedPluginInstallations(resourceId)
).filter((plugin) => plugin.enabled);  // 关键：只将启用的插件传给 DSG
```

> **设计意图**：两处筛选各司其职
> - 位置1 用于判断是否需要走私有插件下载流程
> - 位置2 用于确保最终进入代码生成器的只有启用的插件

#### 2. 顺序读取逻辑

[`getOrderedPluginInstallations()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L281-L307)
```typescript
async getOrderedPluginInstallations(resourceId: string): Promise<PluginInstallation[]> {
  // 步骤1: 读取 PluginOrder Block（存储用户定义的顺序）
  const pluginOrder = await this.pluginOrderService.findByResourceId({
    where: { id: resourceId },
  });

  // 步骤2: 读取该资源所有插件安装记录
  const resourcePluginInstallations = await super.findMany({
    where: { resource: { id: resourceId } },
  });

  // 步骤3: 无顺序配置时直接返回原始顺序
  if (!pluginOrder) return resourcePluginInstallations;

  // 步骤4: 按 PluginOrder 中定义的顺序匹配安装记录
  const orderedPluginInstallations: PluginInstallation[] = [];
  pluginOrder.order.forEach((pluginOrder) => {
    const current = resourcePluginInstallations.find(
      (plugin) => plugin.pluginId === pluginOrder.pluginId  // 按 pluginId 关联
    );
    current && orderedPluginInstallations.push(current);
  });

  return orderedPluginInstallations;
}
```

[`PluginOrderService.findByResourceId()`](packages/amplication-server/src/core/pluginInstallation/pluginOrder.service.ts#L30-L40)
```typescript
async findByResourceId(args: FindOneArgs): Promise<PluginOrder | null> {
  const [pluginOrder] = await super.findMany({
    where: { resource: { id: args.where.id } },  // 一个资源只有一个 PluginOrder Block
  });
  return pluginOrder;
}
```

#### 3. 私有插件下载 → 触发生成器的完整异步链路

**步骤1: 构建入口判断分流**：
[`BuildService.create()`](packages/amplication-server/src/core/build/build.service.ts#L333-L349)
```typescript
const resourcePrivatePlugins =
  await this.pluginInstallationService.getInstalledPrivatePluginsForBuild(resourceId);

if (resourcePrivatePlugins.length > 0) {
  // 有私有插件：先走下载流程，下载完成后异步触发生成
  await this.downloadPrivatePlugins(logger, build, user, resourcePrivatePlugins);
} else {
  // 无私有插件：直接触发代码生成
  await this.generate(logger, build, user);
}
```

**步骤2: 发送下载请求到 Kafka**：
[`BuildService.downloadPrivatePlugins()`](packages/amplication-server/src/core/build/build.service.ts#L623-L737)
```typescript
const downloadPrivatePluginsRequest: DownloadPrivatePluginsRequest.KafkaEvent = {
  key: { resourceId },
  value: {
    buildId: build.id,
    resourceId,
    repositoryPlugins: repositoryPlugins,  // 按仓库分组的插件列表
  },
};
await this.kafkaProducerService.emitMessage(
  KAFKA_TOPICS.DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC,
  downloadPrivatePluginsRequest
);
```

**步骤3: git-sync-manager 消费并下载**：
[`PrivatePluginController.downloadPrivatePlugins()`](ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts#L32-L84)
```typescript
@EventPattern(KAFKA_TOPICS.DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC)
async downloadPrivatePlugins(
  @Payload() message: DownloadPrivatePluginsRequest.Value,
  @Ctx() context: KafkaContext
) {
  try {
    // 实际下载插件到共享存储
    const { pluginPaths } = await KafkaPacemaker.wrapLongRunningMethod(
      context,
      () => this.privatePluginService.downloadPrivatePlugins(validArgs)
    );

    // 下载成功，发送成功消息
    const successEvent: DownloadPrivatePluginsSuccess.KafkaEvent = {
      key: { resourceId: eventKey.resourceId },
      value: { buildId: validArgs.buildId, pluginPaths },
    };
    await this.producerService.emitMessage(
      KAFKA_TOPICS.DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC,
      successEvent
    );
  } catch (error) {
    // 下载失败，发送失败消息
    const failureEvent: DownloadPrivatePluginsFailure.KafkaEvent = { ... };
    await this.producerService.emitMessage(
      KAFKA_TOPICS.DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC,
      failureEvent
    );
  }
}
```

**步骤4: amplication-server 消费成功消息并触发生成**：
[`BuildController.onDownloadPrivatePluginsSuccess()`](packages/amplication-server/src/core/build/build.controller.ts#L160-L173)
```typescript
@EventPattern(KAFKA_TOPICS.DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC)
async onDownloadPrivatePluginsSuccess(
  @Payload() message: DownloadPrivatePluginsSuccess.Value
): Promise<void> {
  const args = plainToInstance(DownloadPrivatePluginsSuccess.Value, message);
  await this.buildService.onDownloadPrivatePluginSuccess(args);  // 转发到 service
}
```

[`BuildService.onDownloadPrivatePluginSuccess()`](packages/amplication-server/src/core/build/build.service.ts#L1012-L1042)
```typescript
public async onDownloadPrivatePluginSuccess(
  response: DownloadPrivatePluginsSuccess.Value
): Promise<void> {
  const { buildId } = response;
  // ... 获取 build 和 user 信息 ...

  // 关键：私有插件下载完成后，触发代码生成
  await this.generate(logger, build, user);  // ← 与无私有插件时调用的是同一个 generate()

  await this.actionService.complete(step, EnumActionStepStatus.Success);
}
```

> **设计要点**：
> - 两条分支最终都汇入同一个 `generate()` 方法，确保后续流程一致
> - 异步链路通过 Kafka 解耦，避免长请求阻塞
> - 支持成功/失败/日志三种 Kafka 消息，完整反馈下载状态

#### 4. `generate()` 组装 DSG 输入

[`BuildService.generate()`](packages/amplication-server/src/core/build/build.service.ts#L568-L618)
```typescript
private async generate(logger: ILogger, build: Build, user: User): Promise<string> {
  return this.actionService.run(build.actionId, GENERATE_STEP_NAME, GENERATE_STEP_MESSAGE,
    async (step) => {
      const { resourceId, id: buildId, version: buildVersion } = build;

      // 步骤1: 组装完整的 DSGResourceData（包含插件列表）
      const dsgResourceData = await this.getDSGResourceData(
        resource, buildId, buildVersion, user
      );

      // 步骤2: 保存到共享存储（避免 Kafka 消息过大）
      await this.saveDsgResourceDataToSharedStorage(buildId, dsgResourceData);

      // 步骤3: 发送轻量代码生成请求（仅带 ID，DSG 自行读取文件）
      const codeGenerationEvent: CodeGenerationRequest.KafkaEvent = {
        key: null,
        value: { resourceId, buildId },  // 轻量消息
      };
      await this.kafkaProducerService.emitMessage(
        KAFKA_TOPICS.CODE_GENERATION_REQUEST_TOPIC,
        codeGenerationEvent
      );
    }
  );
}
```

[`BuildService.getDSGResourceData()`](packages/amplication-server/src/core/build/build.service.ts#L1384-L1421)
```typescript
async getDSGResourceData(...): Promise<CodeGenTypes.DSGResourceData> {
  const resourceId = resource.id;

  // 关键：获取有序且启用的插件列表
  const orderedPlugins = (
    await this.pluginInstallationService.getOrderedPluginInstallations(resourceId)
  ).filter((plugin) => plugin.enabled);

  // ... 组装其他数据（entities, roles, modules 等）...

  const dsgResourceData: CodeGenTypes.DSGResourceData = {
    entities: ...,
    roles: ...,
    pluginInstallations: orderedPlugins,  // ← 插件列表注入点
    resourceSettings: ...,
    // ... 其他字段
  };

  return omitDeep(dsgResourceData, DSG_RESOURCE_DATA_PROPERTIES_TO_REMOVE);
}
```

---

## 阶段三：加载阶段 (Loading / Registration)

### 核心职责
代码生成器（DSG）消费 Kafka 请求后，从共享存储读取 `DSGResourceData`，动态安装 NPM 包并加载插件模块，注册事件监听函数。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `dynamicPackagesInstallations` | [libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts](libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts) | 动态安装 NPM 包 |
| `registerPlugins` | [packages/data-service-generator/src/register-plugin.ts](packages/data-service-generator/src/register-plugin.ts) | 加载并注册插件事件 |
| `prepareContext` | [packages/data-service-generator/src/prepare-context.ts](packages/data-service-generator/src/prepare-context.ts) | 上下文初始化入口 |
| `generateCode` | [packages/data-service-generator/src/generate-code.ts](packages/data-service-generator/src/generate-code.ts) | DSG 入口函数 |

### 执行流程

**0. DSG 入口**：
[`generateCode()`](packages/data-service-generator/src/generate-code.ts#L66-L94)
```typescript
export const generateCode = async (): Promise<void> => {
  const buildSpecPath = process.env.BUILD_SPEC_PATH;  // 共享存储路径
  const resourceData = await readInputJson(buildSpecPath);  // 读取 JSON 文件
  await generateCodeByResourceData(resourceData, buildOutputPath);
};
```

**1. 触发时机**：
在 [`createDataService()`](packages/data-service-generator/src/create-data-service.ts#L15-L105) 中按顺序执行：
```typescript
export async function createDataService(
  dSGResourceData: DSGResourceData,
  internalLogger: ILogger,
  pluginInstallationPath?: string
): Promise<ModuleMap> {
  // 步骤1: 准备默认插件（注入内置必需插件）
  dSGResourceData.pluginInstallations = prepareDefaultPlugins(
    dSGResourceData.pluginInstallations
  );

  // 步骤2: 动态安装 NPM 包（仅公共插件，私有插件已下载）
  await dynamicPackagesInstallations(
    dSGResourceData.pluginInstallations,
    pluginInstallationPath,
    internalLogger,
    context.logger
  );

  // 步骤3: 初始化上下文（含插件注册）
  await prepareContext(dSGResourceData, internalLogger, pluginInstallationPath);

  // ... 后续代码生成 ...
}
```

**2. 动态安装 NPM 包**：
[`dynamicPackagesInstallations()`](libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts#L11-L62)
```typescript
export async function dynamicPackagesInstallations(
  packages: PluginInstallation[],
  pluginInstallationPath: string,
  logger: ILogger,
  buildLogger: IBuildLogger
): Promise<void> {
  const manager = new DynamicPackageInstallationManager(
    pluginInstallationPath,
    buildLogger
  );

  // 只安装非私有插件（私有插件已由 git-sync-manager 下载到指定位置）
  for (const plugins of packages.filter((plugin) => !plugin.isPrivate)) {
    const plugin: PackageInstallation = {
      name: plugins.npm,
      version: plugins.version,
      settings: plugins.settings,
      pluginId: plugins.pluginId,
    };
    await manager.install(plugin, {
      onBeforeInstall: async (plugin) => { ... },
      onAfterInstall: async (plugin, installedPluginVersion) => {
        buildManagerNotifier.notifyPluginVersion(installedPluginVersion);
      },
      onError: async (plugin, error) => { ... },
    });
  }
}
```

**3. 加载插件模块**：
[`getPluginFuncGenerator()`](packages/data-service-generator/src/register-plugin.ts#L44-L78) 异步生成器：
```typescript
async function* getPluginFuncGenerator(
  pluginList: PluginInstallation[],
  pluginInstallationPath?: string
): AsyncGenerator<new () => AmplicationPlugin> {
  do {
    // 支持三种加载路径
    const localPackage = pluginList[index].settings?.local
      ? join("../../../../", pluginList[index].settings?.destPath)    // 本地开发
      : pluginList[index].isPrivate
        ? getPrivatePluginPath(pluginList[index].pluginId)            // 私有插件（已下载路径）
        : undefined;
    const packageName = localPackage || pluginList[index].npm;        // 公共 NPM 包

    // 动态导入
    const func = await getPlugin(packageName, localPackage ? undefined : pluginInstallationPath);

    func.default.prototype.pluginName = packageName;
    yield func.default;
  } while (pluginListLength > index);
}
```

**4. 注册插件事件**：
[`getAllPlugins()`](packages/data-service-generator/src/register-plugin.ts#L101-L124) 遍历插件列表：
```typescript
const getAllPlugins = async (pluginList, pluginInstallationPath) => {
  for await (const pluginFunc of getPluginFuncGenerator(pluginList, ...)) {
    const initializeClass = new pluginFunc();  // 实例化插件类
    const pluginEvents = initializeClass.register();  // 调用 register() 获取事件映射
    pluginFuncsArr.push(pluginEvents);
  }
  return pluginFuncsArr;
};
```

**5. 构建插件映射表**：
[`registerPlugins()`](packages/data-service-generator/src/register-plugin.ts#L129-L160) 按事件分组：
```typescript
const registerPlugins = async (pluginList, pluginInstallationPath) => {
  const pluginMap: PluginMap = {};

  const pluginFuncsArr = await getAllPlugins(pluginList, pluginInstallationPath);

  pluginFuncsArr.reduce((pluginContext, plugin) => {
    Object.keys(plugin).forEach((eventKey) => {
      if (!pluginMap.hasOwnProperty(eventKey))
        pluginContext[eventKey as EventNames] = { before: [], after: [] };

      const { before, after } = plugin[eventKey as keyof Events] || {};

      functionsObject.includes(Object.prototype.toString.call(before)) &&
        pluginContext[eventKey].before.push(before);
      functionsObject.includes(Object.prototype.toString.call(after)) &&
        pluginContext[eventKey].after.push(after);
    });
    return pluginContext;
  }, pluginMap);

  return pluginMap;
};
```

**插件接口定义**参见 [plugins.types.ts](libs/util/code-gen-types/src/plugins.types.ts#L126-L128)：
```typescript
interface AmplicationPlugin {
  init?: (name: string, version: string) => void;
  register: () => Events;  // 返回 { [EventName]: { before?, after? } }
}
```

---

## 阶段四：配置注入阶段 (Configuration Injection)

### 核心职责
将插件映射表、安装配置及用户设置注入到 DSG 上下文，供执行阶段使用。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `DsgContext` | [packages/data-service-generator/src/dsg-context.ts](packages/data-service-generator/src/dsg-context.ts) | 单例上下文容器 |
| `prepareContext` | [packages/data-service-generator/src/prepare-context.ts](packages/data-service-generator/src/prepare-context.ts) | 上下文初始化 |

### 执行流程

**1. 上下文初始化**：
在 [`prepareContext()`](packages/data-service-generator/src/prepare-context.ts#L41-L124) 中：
```typescript
export async function prepareContext(
  dSGResourceData: DSGResourceData,
  internalLogger: ILogger,
  pluginInstallationPath?: string
): Promise<void> {
  const { pluginInstallations: resourcePlugins, ... } = dSGResourceData;

  // 注册插件，获得事件映射表
  const plugins = await registerPlugins(resourcePlugins, pluginInstallationPath);

  // 注入到单例上下文
  const context = DsgContext.getInstance;
  context.plugins = plugins;                    // 事件映射表: { [EventName]: { before: [], after: [] } }
  context.pluginInstallations = resourcePlugins; // 安装配置原始数据（含用户 settings）
}
```

**2. 上下文数据结构**：
参见 [DsgContext](packages/data-service-generator/src/dsg-context.ts#L19-L81)：
```typescript
class DsgContext implements types.DsgContext {
  // 插件事件映射表
  plugins: types.PluginMap = {};

  // 插件安装配置（含用户 settings）
  pluginInstallations: types.PluginInstallation[] = [];

  // 工具方法，供插件使用
  utils: ContextUtil = {
    skipDefaultBehavior: boolean,    // 跳过默认代码生成
    abortGeneration: (msg) => void,  // 中止构建
    importStaticModules: (source, basePath) => Promise<ModuleMap>,
  };

  // ... 其他上下文数据（entities, roles, modules 等）
}
```

**3. 插件获取配置的方式**：
插件在 `before`/`after` 钩子中通过 `dsgContext` 访问配置：
```typescript
// 插件代码示例
export function beforeCreateServer(dsgContext, eventParams) {
  // 获取当前插件的安装配置
  const myInstallation = dsgContext.pluginInstallations.find(
    p => p.pluginId === "my-plugin-id"
  );
  const userSettings = myInstallation?.settings;      // 用户配置值
  const configurations = myInstallation?.configurations;  // 配置项定义
}
```

---

## 阶段五：执行阶段 (Execution)

### 核心职责
在代码生成的各个事件点，按顺序执行插件的 `before` 和 `after` 钩子，实现对生成过程的干预。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `pluginWrapper` | [packages/data-service-generator/src/plugin-wrapper.ts](packages/data-service-generator/src/plugin-wrapper.ts) | 插件执行包装器 |
| `EventNames` 枚举 | [libs/util/code-gen-types/src/plugins.types.ts](libs/util/code-gen-types/src/plugins.types.ts#L77-L124) | 可用事件列表 |

### 可用事件类型
系统定义了 50+ 个代码生成事件，包括：
- `CreateServer` / `CreateAdminUI` - 服务/Admin UI 生成
- `CreateEntityController` / `CreateEntityService` / `CreateEntityResolver` - 实体各层代码
- `CreatePrismaSchema` / `CreateServerPackageJson` - 配置文件
- `CreateDTOs` / `CreateSeed` - 数据传输对象/种子数据
- `CreateMessageBroker*` - 消息总线相关

### 执行流程

**1. 事件触发**：
代码生成器的每个函数都通过 `pluginWrapper` 包裹，示例参见 [create-controller.ts](packages/data-service-generator/src/server/resource/controller/create-controller.ts#L162-L191)：
```typescript
await moduleMap.mergeMany([
  await pluginWrapper(
    createControllerModule,                    // 原始生成函数
    EventNames.CreateEntityController,         // 事件名称
    { template, entityName, ... } as Params    // 事件参数
  ),
  await pluginWrapper(
    createControllerBaseModule,
    EventNames.CreateEntityControllerBase,
    { ... } as Params
  ),
]);
```

**2. pluginWrapper 执行管道**：
[`pluginWrapper()`](packages/data-service-generator/src/plugin-wrapper.ts#L59-L117) 核心逻辑：
```typescript
const pluginWrapper = async (func, event, args) => {
  const context = DsgContext.getInstance;

  // 无插件注册时直接执行
  if (!context.plugins.hasOwnProperty(event)) {
    return await func(args);
  }

  const beforePlugins = context.plugins[event]?.before || [];
  const afterPlugins = context.plugins[event]?.after || [];

  // 阶段1: 执行 before 管道（从左到右，流式传递参数）
  const updatedEventParams = beforePlugins
    ? await beforeEventsPipe(...beforePlugins)(context, args)
    : args;

  // 阶段2: 执行默认生成逻辑
  const defaultBehaviorModules = await defaultBehavior(
    context,
    func,
    updatedEventParams
  );

  // 阶段3: 执行 after 管道（从左到右，流式传递模块）
  const finalModules = afterPlugins
    ? await afterEventsPipe(...afterPlugins)(context, args, defaultBehaviorModules)
    : defaultBehaviorModules;

  // 将结果合并到上下文
  for (const module of finalModules.modules()) {
    context.modules.replace(module, module);
  }

  return finalModules;
};
```

**3. Before 事件管道**：
[`beforeEventsPipe()`](packages/data-service-generator/src/plugin-wrapper.ts#L17-L23) 实现参数流式传递：
```typescript
const beforeEventsPipe = (...fns) => (context, eventParams) =>
  fns.reduce(
    async (res, fn) => fn(context, await res),  // 前一个的输出作为后一个的输入
    Promise.resolve(eventParams)
  );
```

**4. After 事件管道**：
[`afterEventsPipe()`](packages/data-service-generator/src/plugin-wrapper.ts#L25-L31) 实现模块流式转换：
```typescript
const afterEventsPipe = (...fns) => (context, eventParams, modules) =>
  fns.reduce(
    async (res, fn) => fn(context, eventParams, await res),  // 前一个的输出作为后一个的输入
    Promise.resolve(modules)
  );
```

**5. 跳过默认行为**：
插件可通过设置 `context.utils.skipDefaultBehavior = true` 跳过原始生成逻辑，参见 [`defaultBehavior()`](packages/data-service-generator/src/plugin-wrapper.ts#L40-L51)：
```typescript
const defaultBehavior = async (context, func, beforeFuncResults) => {
  if (context.utils.skipDefaultBehavior)
    return new ModuleMap(DsgContext.getInstance.logger);  // 返回空模块集合
  return func(beforeFuncResults);
};
```

### 插件执行顺序
1. 同一事件的 `before` 钩子按插件安装顺序从先到后执行
2. 同一事件的 `after` 钩子按插件安装顺序从先到后执行
3. 插件顺序可通过 `PluginInstallationService.setOrder()` 调整
4. 顺序存储在 `PluginOrder` Block 中，由 [`getOrderedPluginInstallations()`](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L281-L307) 读取

### 错误处理
- 插件执行失败时抛出友好错误，包含事件名和原始错误信息
- 支持 `context.utils.abortGeneration(msg)` 主动中止构建
- 错误会记录到构建日志并通过 Kafka 上报

---

## 跨模块调用关系

```
amplication-server                          data-service-generator
┌───────────────────────────────────┐       ┌───────────────────────────────────┐
│ PluginCatalogService              │───查询──▶│                                   │
│   - getPlugins()                  │       │  generateCode()                   │
│   - getPluginWithLatest()         │       │    - readInputJson() 读共享存储   │
└───────────────────────────────────┘       │    │                              │
                                            │    ▼                              │
┌───────────────────────────────────┐       │  createDataService()              │
│ PluginInstallationService         │───配置──▶│    - prepareDefaultPlugins()   │
│   - create()                      │ (通过   │    - dynamicPackagesInstall()  │
│   - setOrder()                    │ DSG    │    - prepareContext()           │
│   - getOrderedPluginInstallations()│ Resource│      - registerPlugins()      │
└───────────────────────────────────┘  Data) │      - context.plugins = map    │
                                            │      - context.pluginInstallations│
┌───────────────────────────────────┐       │    │                              │
│ BuildService                      │───构建──▶│    ▼                              │
│   - create()  ←───────────────┐   │ (Kafka)│  代码生成函数                    │
│   - generate()               │   │        │    - pluginWrapper()              │
│   - downloadPrivatePlugins() │   │        │      * before 管道                │
│   - onDownloadPrivatePluginSuccess() │    │      * 默认逻辑                  │
│   ▲                            │   │        │      * after 管道                 │
└───┼────────────────────────────┼───┘        └───────────────────────────────────┘
    │                            │
    │ Kafka 成功消息             │ Kafka 请求消息
    ▼                            ▼
┌───────────────────────────────────┐
│ PrivatePluginController           │  (git-sync-manager)
│   - downloadPrivatePlugins()      │
│   → 实际下载插件到共享存储        │
└───────────────────────────────────┘
```

---

## 关键入口文件速查

| 阶段 | 入口文件 | 关键函数 |
|------|---------|---------|
| 发现 | [packages/amplication-plugin-api/src/plugin/github-plugin.service.ts](packages/amplication-plugin-api/src/plugin/github-plugin.service.ts) | `getPlugins()`, `getPluginConfig()` |
| 安装 | [packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts](packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) | `create()`, `setOrder()` |
| **构建输入** | [packages/amplication-server/src/core/build/build.service.ts](packages/amplication-server/src/core/build/build.service.ts) | `create()`, `generate()`, `getDSGResourceData()`, `onDownloadPrivatePluginSuccess()` |
| 构建输入（Kafka消费） | [packages/amplication-server/src/core/build/build.controller.ts](packages/amplication-server/src/core/build/build.controller.ts) | `onDownloadPrivatePluginsSuccess()` |
| 私有插件下载 | [ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts](ee/packages/git-sync-manager/src/private-plugin/private-plugin.controller.ts) | `downloadPrivatePlugins()` |
| 加载 | [packages/data-service-generator/src/register-plugin.ts](packages/data-service-generator/src/register-plugin.ts) | `registerPlugins()`, `getPluginFuncGenerator()` |
| 注入 | [packages/data-service-generator/src/prepare-context.ts](packages/data-service-generator/src/prepare-context.ts) | `prepareContext()` |
| 执行 | [packages/data-service-generator/src/plugin-wrapper.ts](packages/data-service-generator/src/plugin-wrapper.ts) | `pluginWrapper()`, `beforeEventsPipe()`, `afterEventsPipe()` |

---

## 附录：Kafka 主题清单

与插件相关的 Kafka 主题定义在 [libs/schema-registry/src/index.ts](libs/schema-registry/src/index.ts#L29-L69)：

| 主题 | 用途 | 方向 |
|------|------|------|
| `DOWNLOAD_PRIVATE_PLUGINS_REQUEST_TOPIC` | 请求下载私有插件 | amplication-server → git-sync-manager |
| `DOWNLOAD_PRIVATE_PLUGINS_SUCCESS_TOPIC` | 私有插件下载成功 | git-sync-manager → amplication-server |
| `DOWNLOAD_PRIVATE_PLUGINS_FAILURE_TOPIC` | 私有插件下载失败 | git-sync-manager → amplication-server |
| `DOWNLOAD_PRIVATE_PLUGINS_LOG_TOPIC` | 私有插件下载日志 | git-sync-manager → amplication-server |
| `CODE_GENERATION_REQUEST_TOPIC` | 请求代码生成 | amplication-server → data-service-generator |
| `BUILD_PLUGIN_NOTIFY_VERSION_TOPIC` | 上报插件实际安装版本 | DSG → amplication-server |
