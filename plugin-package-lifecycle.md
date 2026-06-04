# 插件包生命周期 (Plugin Package Lifecycle)

本文档详细梳理 Amplication 插件包从发现到生效的完整生命周期，涵盖五个核心阶段：**发现 → 安装 → 加载 → 配置注入 → 执行**。

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
            │  安装阶段   │  PluginInstallationService
            └─────────────┘
                   │
                   ▼  触发构建
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
| `GitPluginService` | [github-plugin.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts) | 从 GitHub 拉取插件目录 |
| `PluginService` | [plugin.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-plugin-api/src/plugin/plugin.service.ts) | 将插件元数据存入数据库 |
| `PluginCatalogService` | [pluginCatalog.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginCatalog/pluginCatalog.service.ts) | 为前端提供插件查询接口 |

### 执行流程

**1. 拉取插件目录清单**
- 调用 [getPlugins()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts#L153-L199) 方法
- 请求 `AMPLICATION_GITHUB_URL` 获取所有插件的 `.yml` 配置文件列表
- 使用 GitHub Token 避免 API 限流

**2. 逐个解析插件配置**
- 通过 [getPluginConfig()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts#L89-L137) 异步生成器遍历每个插件
- 下载并解析 YAML 配置，获取 `pluginId`、`npm`、`github`、`categories` 等元数据

**3. 获取 NPM 包信息**
- 调用 [fetchNpmData()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts#L43-L84) 
- 并行获取 NPM 版本信息 (`dist-tags`) 和下载量统计

**4. 同步到数据库**
- 在 [processCatalogPlugins()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-plugin-api/src/plugin/plugin.service.ts#L34-L97) 中：
  - `prisma.plugin.createMany()` 批量插入新插件（`skipDuplicates: true`）
  - `prisma.$transaction()` 批量更新现有插件信息
  - 同步插件分类信息到 `Category` 表

**5. 提供查询接口**
- [getPlugins()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginCatalog/pluginCatalog.service.ts#L96-L112) 按代码生成器类型过滤插件
- [getPluginWithLatestVersion()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginCatalog/pluginCatalog.service.ts#L36-L94) 获取插件详情及最新版本

### 数据模型
插件目录项结构参见 [PluginCatalogItem.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginCatalog/dto/PluginCatalogItem.ts)：
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
| `PluginInstallationService` | [pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) | 插件安装管理 |
| `PluginOrderService` | 同目录 | 插件执行顺序管理 |

### 执行流程

**1. 创建插件安装记录**
- 调用 [create()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L121-L171) 方法
- **前置校验**：
  - [validatePluginConfiguration()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L97-L119) 检查认证实体依赖
  - [findPluginInstallationByPluginId()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L82-L95) 防止重复安装
- 通过 `BlockService` 创建 `PluginInstallation` 类型的 Block 记录

**2. 设置执行顺序**
- 调用 [setOrder()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L200-L260)
- 新插件默认添加到末尾（`order: -1` 表示追加）
- [reOrderPlugins()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L28-L56) 处理顺序冲突

**3. 安装数据结构**
参见 [PluginInstallation.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/dto/PluginInstallation.ts)：
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

**4. 私有插件特殊处理**
- 构建前通过 [getInstalledPrivatePluginsForBuild()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L262-L279) 筛选
- 在 [downloadPrivatePlugins()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/build/build.service.ts#L623-L737) 中通过 Kafka 消息通知下载

---

## 阶段三：加载阶段 (Loading / Registration)

### 核心职责
在代码生成前，动态安装 NPM 包并加载插件模块，注册事件监听函数。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `dynamicPackagesInstallations` | [dynamic-package-installation.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts) | 动态安装 NPM 包 |
| `registerPlugins` | [register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/register-plugin.ts) | 加载并注册插件事件 |
| `prepareContext` | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/prepare-context.ts) | 上下文初始化入口 |

### 执行流程

**1. 触发时机**
在 [createDataService()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/create-data-service.ts#L15-L105) 中按顺序执行：
```typescript
// 步骤1: 准备默认插件
dSGResourceData.pluginInstallations = prepareDefaultPlugins(plugins);

// 步骤2: 动态安装 NPM 包
await dynamicPackagesInstallations(plugins, installPath, logger, buildLogger);

// 步骤3: 初始化上下文（含插件注册）
await prepareContext(dSGResourceData, logger, installPath);
```

**2. 动态安装 NPM 包**
- [DynamicPackageInstallationManager.install()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts#L25-L59) 逐个安装非私有插件
- 支持生命周期钩子：`onBeforeInstall` / `onAfterInstall` / `onError`
- 通过 `BuildManagerNotifier.notifyPluginVersion()` 上报实际安装版本

**3. 加载插件模块**
[getPluginFuncGenerator()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/register-plugin.ts#L44-L78) 异步生成器：
```typescript
// 支持三种加载路径
const localPackage = plugin.settings?.local 
  ? join("../../../../", plugin.settings?.destPath)    // 本地开发
  : plugin.isPrivate 
    ? getPrivatePluginPath(plugin.pluginId)            // 私有插件
    : undefined;
const packageName = localPackage || plugin.npm;        // 公共 NPM 包

// 动态导入
const func = await import(packageName);
```

**4. 注册插件事件**
[getAllPlugins()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/register-plugin.ts#L101-L124) 遍历插件列表：
```typescript
for await (const pluginFunc of getPluginFuncGenerator(pluginList)) {
  const initializeClass = new pluginFunc();  // 实例化插件类
  const pluginEvents = initializeClass.register();  // 调用 register() 获取事件映射
  pluginFuncsArr.push(pluginEvents);
}
```

**5. 构建插件映射表**
[registerPlugins()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/register-plugin.ts#L129-L160) 按事件分组：
```typescript
pluginFuncsArr.reduce((pluginMap, plugin) => {
  Object.keys(plugin).forEach(eventKey => {
    if (!pluginMap[eventKey]) {
      pluginMap[eventKey] = { before: [], after: [] };
    }
    // 收集 before/after 钩子
    pluginMap[eventKey].before.push(plugin[eventKey].before);
    pluginMap[eventKey].after.push(plugin[eventKey].after);
  });
  return pluginMap;
}, {});
```

**插件接口定义**参见 [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/libs/util/code-gen-types/src/plugins.types.ts#L126-L128)：
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
| `DsgContext` | [dsg-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/dsg-context.ts) | 单例上下文容器 |
| `prepareContext` | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/prepare-context.ts) | 上下文初始化 |

### 执行流程

**1. 上下文初始化**
在 [prepareContext()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/prepare-context.ts#L41-L124) 中：
```typescript
// 注册插件，获得事件映射表
const plugins = await registerPlugins(resourcePlugins, pluginInstallationPath);

// 注入到单例上下文
const context = DsgContext.getInstance;
context.plugins = plugins;                    // 事件映射表
context.pluginInstallations = resourcePlugins; // 安装配置原始数据
```

**2. 上下文数据结构**
参见 [DsgContext](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/dsg-context.ts#L19-L81)：
```typescript
class DsgContext implements types.DsgContext {
  // 插件事件映射表: { [EventName]: { before: Function[], after: Function[] } }
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

**3. 插件获取配置的方式**
插件在 `before`/`after` 钩子中通过 `dsgContext` 访问配置：
```typescript
// 插件代码示例
export function beforeCreateServer(dsgContext, eventParams) {
  // 获取当前插件的安装配置
  const myInstallation = dsgContext.pluginInstallations.find(
    p => p.pluginId === "my-plugin-id"
  );
  const userSettings = myInstallation?.settings;  // 用户配置
  const configurations = myInstallation?.configurations;  // 配置定义
}
```

---

## 阶段五：执行阶段 (Execution)

### 核心职责
在代码生成的各个事件点，按顺序执行插件的 `before` 和 `after` 钩子，实现对生成过程的干预。

### 关键组件
| 组件 | 文件路径 | 核心职责 |
|------|---------|---------|
| `pluginWrapper` | [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/plugin-wrapper.ts) | 插件执行包装器 |
| `EventNames` 枚举 | [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/libs/util/code-gen-types/src/plugins.types.ts#L77-L124) | 可用事件列表 |

### 可用事件类型
系统定义了 50+ 个代码生成事件，包括：
- `CreateServer` / `CreateAdminUI` - 服务/Admin UI 生成
- `CreateEntityController` / `CreateEntityService` / `CreateEntityResolver` - 实体各层代码
- `CreatePrismaSchema` / `CreateServerPackageJson` - 配置文件
- `CreateDTOs` / `CreateSeed` - 数据传输对象/种子数据
- `CreateMessageBroker*` - 消息总线相关

### 执行流程

**1. 事件触发**
代码生成器的每个函数都通过 `pluginWrapper` 包裹，示例参见 [create-controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/server/resource/controller/create-controller.ts#L162-L191)：
```typescript
await pluginWrapper(
  createControllerModule,                    // 原始生成函数
  EventNames.CreateEntityController,         // 事件名称
  { template, entityName, ... } as Params    // 事件参数
);
```

**2. pluginWrapper 执行管道**
[pluginWrapper()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L59-L117) 核心逻辑：
```typescript
const pluginWrapper = async (func, event, args) => {
  const context = DsgContext.getInstance;
  
  // 无插件注册时直接执行
  if (!context.plugins.hasOwnProperty(event)) {
    return await func(args);
  }
  
  const beforePlugins = context.plugins[event]?.before || [];
  const afterPlugins = context.plugins[event]?.after || [];
  
  // 阶段1: 执行 before 管道（从左到右）
  const updatedEventParams = await beforeEventsPipe(...beforePlugins)(context, args);
  
  // 阶段2: 执行默认生成逻辑
  const defaultBehaviorModules = await defaultBehavior(context, func, updatedEventParams);
  
  // 阶段3: 执行 after 管道（从左到右）
  const finalModules = await afterEventsPipe(...afterPlugins)(context, args, defaultBehaviorModules);
  
  // 将结果合并到上下文
  for (const module of finalModules.modules()) {
    context.modules.replace(module, module);
  }
  
  return finalModules;
};
```

**3. Before 事件管道**
[beforeEventsPipe()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L17-L23) 实现参数流式传递：
```typescript
const beforeEventsPipe = (...fns) => (context, eventParams) =>
  fns.reduce(
    async (res, fn) => fn(context, await res),  // 前一个的输出作为后一个的输入
    Promise.resolve(eventParams)
  );
```

**4. After 事件管道**
[afterEventsPipe()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L25-L31) 实现模块流式转换：
```typescript
const afterEventsPipe = (...fns) => (context, eventParams, modules) =>
  fns.reduce(
    async (res, fn) => fn(context, eventParams, await res),  // 前一个的输出作为后一个的输入
    Promise.resolve(modules)
  );
```

**5. 跳过默认行为**
插件可通过设置 `context.utils.skipDefaultBehavior = true` 跳过原始生成逻辑，参见 [defaultBehavior()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L40-L51)：
```typescript
const defaultBehavior = async (context, func, beforeFuncResults) => {
  if (context.utils.skipDefaultBehavior)
    return new ModuleMap(logger);  // 返回空模块集合
  return func(beforeFuncResults);
};
```

### 插件执行顺序
1. 同一事件的 `before` 钩子按插件安装顺序从先到后执行
2. 同一事件的 `after` 钩子按插件安装顺序从先到后执行
3. 插件顺序可通过 `PluginInstallationService.setOrder()` 调整
4. 顺序存储在 `PluginOrder` Block 中，由 [getOrderedPluginInstallations()](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L281-L307) 读取

### 错误处理
- 插件执行失败时抛出友好错误，包含事件名和原始错误信息
- 支持 `context.utils.abortGeneration(msg)` 主动中止构建
- 错误会记录到构建日志并通过 Kafka 上报

---

## 跨模块调用关系

```
amplication-server                          data-service-generator
┌───────────────────────────┐               ┌───────────────────────────┐
│ PluginCatalogService      │───查询 ───────▶│                           │
│   - getPlugins()          │               │  createDataService()      │
│   - getPluginWithLatest() │               │    │                      │
└───────────────────────────┘               │    ▼                      │
                                            │  dynamicPackagesInstall() │
┌───────────────────────────┐               │    │                      │
│ PluginInstallationService │───安装配置────▶│    ▼                      │
│   - create()              │               │  prepareContext()         │
│   - setOrder()            │   (通过 DSG   │    - registerPlugins()    │
│   - getOrderedPlugins()   │    Resource   │    - context.plugins = map│
└───────────────────────────┘    Data)      │    │                      │
                                            │    ▼                      │
┌───────────────────────────┐               │  代码生成函数             │
│ BuildService              │───触发构建 ──▶│    - pluginWrapper()      │
│   - create()              │               │      * before 管道        │
│   - generate()            │   (Kafka)     │      * 默认逻辑           │
│   - downloadPrivatePlugins()│             │      * after 管道         │
└───────────────────────────┘               └───────────────────────────┘
```

---

## 关键入口文件速查

| 阶段 | 入口文件 | 关键函数 |
|------|---------|---------|
| 发现 | [github-plugin.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts) | `getPlugins()`, `getPluginConfig()` |
| 安装 | [pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) | `create()`, `setOrder()` |
| 加载 | [register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/register-plugin.ts) | `registerPlugins()`, `getPluginFuncGenerator()` |
| 注入 | [prepare-context.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/prepare-context.ts) | `prepareContext()` |
| 执行 | [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/24-amplication/packages/data-service-generator/src/plugin-wrapper.ts) | `pluginWrapper()`, `beforeEventsPipe()`, `afterEventsPipe()` |
