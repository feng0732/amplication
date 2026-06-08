# Amplication 插件发布与版本治理全流程解析

本文档对照代码，详细解析 Amplication 平台中插件从**打包发布**、**版本记录**、**兼容校验**到**发布状态流转**的完整链路。

---

## 一、整体架构概览

插件系统涉及 5 个核心服务/模块，各司其职：

| 模块 | 职责 | 核心目录 |
|------|------|----------|
| `amplication-plugin-api` | 插件元数据与版本管理服务（PostgreSQL） | [amplication-plugin-api](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api) |
| `amplication-server` | 业务服务，管理插件安装、构建调度 | [amplication-server](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-server) |
| `data-service-generator` (DSG) | 代码生成器，运行时动态加载插件 | [data-service-generator](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator) |
| `data-service-generator-catalog` | DSG 版本目录服务，管理生成器自身版本 | [data-service-generator-catalog](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator-catalog) |
| `@amplication/dsg-utils` | 插件动态安装（tarball 下载）工具库 | [dsg-utils](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils) |

外部依赖：
- **GitHub `plugin-catalog` 仓库**：插件注册表（YAML 格式），即"官方插件市场"
- **npm registry**：插件实际代码包的存储与分发

---

## 二、插件打包（Packaging）

### 2.1 插件包结构

Amplication 插件本质是一个**标准 npm 包**，但必须满足以下约定：

1. **默认导出插件类**：实现 `AmplicationPlugin` 接口，必须有 `register()` 方法
2. **`.amplicationrc.json`**：插件根目录下的配置文件，包含 `settings` 和 `systemSettings`

类型定义见 [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/code-gen-types/src/plugins.types.ts#L126-L129)：

```typescript
export interface AmplicationPlugin {
  init?: (name: string, version: string) => void;
  register: () => Events;
}
```

`register()` 返回的 `Events` 对象将插件逻辑挂载到 DSG 各生命周期事件点（如 `CreateServerPackageJson`、`CreatePrismaSchema` 等），见 [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/code-gen-types/src/plugins.types.ts#L77-L124)。

### 2.2 打包发布脚本

平台内置统一的发布脚本 [scripts/publish.mjs](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/scripts/publish.mjs)，执行流程：

```
参数校验 (SemVer 格式)
  → 读取 Nx project graph 获取构建输出目录
  → 更新 build 输出中 package.json 的 version 字段
  → npm publish --access public --tag {tag}
```

关键代码片段 [publish.mjs:L24-L61](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/scripts/publish.mjs#L24-L61)：

```javascript
const [, , name, version, tag = "next"] = process.argv;
const validVersion = /^\d+\.\d+\.\d+(-\w+\.\d+)?/;
// ... SemVer 校验
const graph = readCachedProjectGraph();
const project = graph.nodes[name];
const outputPath = project.data?.targets?.build?.options?.outputPath;
process.chdir(outputPath);
// 更新 package.json version
json.version = version;
writeFileSync(`package.json`, JSON.stringify(json, null, 2));
// 发布到 npm
execSync(`npm publish --access public --tag ${tag}`);
```

**注意**：默认 tag 是 `"next"`，避免意外覆盖 `latest` 标签。

---

## 三、版本记录（Version Tracking）

插件版本记录通过**双源同步**机制完成：从 GitHub plugin-catalog 拉取插件元数据，从 npm registry 拉取版本列表。

### 3.1 数据模型

数据库模型定义在 [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/prisma/schema.prisma#L12-L40)：

```prisma
model Plugin {
  pluginId          String?  @unique        // 插件唯一 ID（如 "db-postgres"）
  name              String?
  npm               String?                  // npm 包名（如 "@amplication/plugin-db-postgres"）
  taggedVersions    Json?                    // npm dist-tags（{ "latest": "1.2.3" }）
  downloads         Int?                     // npm 下载量
  codeGeneratorName String   @default("NodeJs")  // 适配的代码生成器
  categories        Json?
  // ...
}

model PluginVersion {
  pluginIdVersion String   @unique           // 复合主键：{pluginId}_{version}
  pluginId        String?
  version         String?                    // SemVer 版本号
  isLatest        Boolean?                   // 是否为 latest 版本
  deprecated      String?                    // 弃用信息（npm deprecated 字段）
  settings        Json?                      // 从 .amplicationrc.json 提取的用户配置
  configurations  Json?                      // 从 .amplicationrc.json 提取的系统配置
  createdAt       DateTime @default(now())
}
```

### 3.2 同步流程：Plugin（插件元数据）

由 [GitPluginService](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts) 执行，核心流程在 [github-plugin.service.ts:L153-L199](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts#L153-L199)：

```
1. GET https://api.github.com/repos/amplication/plugin-catalog/contents/plugins
   → 获取插件目录下所有 .yml 文件列表
   
2. 对每个 .yml 文件：
   → 下载并解析 YAML（包含 pluginId、npm、github、categories、generator 等）
   → 并行调用 npm API 获取 packument（版本列表 + dist-tags）和 downloads
   
3. 组装 Plugin 对象并写入 DB：
   → prisma.plugin.createMany + prisma.plugin.update（事务）
   → prisma.category.createMany（分类去重）
```

### 3.3 同步流程：PluginVersion（插件版本列表）

由 [PluginVersionService.processPluginsVersions()](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/pluginVersion/pluginVersion.service.ts#L106-L197) 驱动，核心步骤：

```
1. NpmPluginVersionService.getAllPluginsVersions()
   → 遍历所有 Plugin
   → 对每个插件调用 npm packument API 获取所有版本
   
2. 版本过滤（SemVer 正则）见 [npm-plugin-version.service.ts:L17-L36](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/pluginVersion/npm-plugin-version.service.ts#L17-L36)：
   - IGNORE_PRERELEASE_PLUGIN_VERSIONS=true  → 仅保留 x.y.z 稳定版
   - IGNORE_PRERELEASE_PLUGIN_VERSIONS=false → 同时保留 x.y.z-beta.n 等预发布版
   
3. 对每个版本：
   a. 已存在 → 仅当 deprecated 或 isLatest 变化时更新
   b. 不存在 → 下载 tarball，提取 .amplicationrc.json 中的 settings/configurations
      → 写入 PluginVersion 记录
```

**版本设置提取**见 [pluginVersion.service.ts:L57-L92](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/pluginVersion/pluginVersion.service.ts#L57-L92)：

```typescript
// 从 npm tarball 中流式解压读取 package/.amplicationrc.json
const res = await fetch(tarBallUrl);
res.body.pipe(zlib.createGunzip()).pipe(extract);
extract.on("entry", function (header, stream, next) {
  if (header.name === fileName) {  // fileName = "package/.amplicationrc.json"
    stream.on("data", (chunk) => resolve(data.toString()));
  }
});
```

### 3.4 版本结构映射

[npm-plugin-version.service.ts:L44-L68](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/pluginVersion/npm-plugin-version.service.ts#L44-L68) 将 npm packument 转换为内部结构：

```typescript
pluginVersions.push({
  createdAt: new Date(npmManifest.time[value.version]),   // npm 发布时间
  deprecated: value.deprecated?.toString() || null,       // 弃用标记
  pluginIdVersion: `${pluginId}_${value.version}`,        // 复合唯一键
  version: value.version,                                 // SemVer
  tarballUrl: value.dist.tarball,                         // npm 包下载地址
  isLatest: npmManifest["dist-tags"].latest === value.version,  // 是否为 latest
});
```

---

## 四、兼容校验（Compatibility Validation）

兼容性体现在三个层面：**DSG 版本与插件版本匹配**、**插件版本语义化校验**、**插件配置校验**。

### 4.1 代码生成器（DSG）版本治理

DSG 自身版本独立管理，由 `data-service-generator-catalog` 服务维护，模型见 [catalog/schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator-catalog/prisma/schema.prisma#L23-L46)：

```prisma
model Generator {
  fullName  String?   @unique     // ECR 镜像名（如 "data-service-generator"）
  name      String?   @unique
  isActive  Boolean?  @default(false)
  version   Version[]
}

model Version {
  name           String                      // 版本标签（如 "v2.10.0"）
  isActive       Boolean                     // 是否可用
  isDeprecated   Boolean?
  generator      Generator?
  @@unique([name, generatorId])
}
```

DSG 版本通过 **AWS ECR 镜像标签**同步，见 [version.service.ts:L178-L252](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator-catalog/src/version/version.service.ts#L178-L252)：

```
syncVersions():
  → 遍历所有 isActive 的 Generator
  → 调用 AWS ECR API 列出镜像的所有 tags
  → 新 tag → 创建 Version 记录（isActive=false，需人工激活）
  → 已删除 tag → 标记 Version 为 deletedAt + isActive=false + isDeprecated=true
```

### 4.2 DSG 版本选择策略

用户构建时可指定版本策略，见 [version.service.ts:L96-L164](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator-catalog/src/version/version.service.ts#L96-L164)：

| 策略 (CodeGeneratorVersionStrategy) | 行为 |
|-------------------------------------|------|
| `Specific` | 使用精确指定的版本号 |
| `LatestMinor` | 同 major 版本下取最大 minor（如指定 v2.5.0 → 取 v2.x.y 最新） |
| `LatestMajor` | 取所有 isActive 版本中最大的（默认） |

版本排序逻辑 [version.service.ts:L40-L67](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator-catalog/src/version/version.service.ts#L40-L67) 是纯字符串数字比较（不依赖 semver 库）。

### 4.3 构建时版本比较

`amplication-build-manager` 中实现了版本比较函数 [code-generator-catalog.service.ts:L75-L112](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-build-manager/src/code-generator/code-generator-catalog.service.ts#L75-L112)，用于判断是否需要升级：

```typescript
compareVersions(currentVersion: string, version: string): number
// 返回 >0 表示 currentVersion 更新，<0 表示 version 更新
// 正则：/^(v?)(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)$/
// 预发布版本（如 v1.0.0-beta）直接抛错不支持
```

### 4.4 插件动态安装时的校验（精确对照代码）

插件在**构建运行时**被动态下载安装，由 `@amplication/dsg-utils` 包完成。整个处理流程分三层，每层对不同"异常版本"的处理方式不同。

#### 4.4.1 调用入口与数据来源

入口在 [dynamic-package-installation.ts:L30-L36](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts#L30-L36)：

```typescript
for (const plugins of packages.filter((plugin) => !plugin.isPrivate)) {
  const plugin: PackageInstallation = {
    name: plugins.npm,
    version: plugins.version,     // 直接从 PluginInstallation.version 取，可能是 "latest" / "1.2.3" / "invalid" 等任意字符串
    settings: plugins.settings,
    pluginId: plugins.pluginId,
  };
  await manager.install(plugin, { ...hooks });
}
```

`PluginInstallation.version` 是一个字符串字段（见 [models.ts:L2738](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/code-gen-types/src/models.ts#L2738)），默认值由 UI 或默认插件机制设置：
- 用户手动指定 → 如 `"1.5.0"`
- 默认插件（如 db-postgres）→ `"latest"`（见 [defaultPlugins.ts:L26](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/utils/dynamic-installation/defaultPlugins.ts#L26)）

---

#### 4.4.2 第一层：`DynamicPackageInstallationManager.install()` — SemVer 规范化

核心代码 [DynamicPackageInstallationManager.ts:L21-L30](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/DynamicPackageInstallationManager.ts#L21-L30)：

```typescript
const { name, version, settings, pluginId } = plugin;
onBeforeInstall && (await onBeforeInstall(plugin));
const validVersion = valid(version);   // semver.valid() 规范化

const tarball = new Tarball(
  { name, version: validVersion, settings, pluginId },  // ← 传给 Tarball 的是 validVersion，不是原始 version
  this.pluginInstallationPath,
  this.logger
);
```

`semver.valid()` 的返回值行为（npm `semver` 包）：

| 输入 `version` | `valid(version)` 返回值 | 传给 Tarball 的值 |
|----------------|------------------------|------------------|
| `"1.2.3"` | `"1.2.3"`（规范化字符串） | `"1.2.3"` |
| `"v1.2.3"` | `"1.2.3"`（自动去掉 v 前缀） | `"1.2.3"` |
| `"1.2.3-beta.1"` | `"1.2.3-beta.1"` | `"1.2.3-beta.1"` |
| `"latest"` | `null`（不是合法 SemVer） | `null` |
| `"invalid-version"` | `null`（不是合法 SemVer） | `null` |
| `"abc"` | `null` | `null` |
| `""`（空串） | `null` | `null` |

**关键事实**：`"latest"` 并不是被特殊识别的标签，而是因为它不是合法 SemVer，被 `valid()` 转为 `null`，间接触发 Tarball 中的 latest 分支。任何非 SemVer 字符串（如 `"abc"`、`"invalid"`）都会走到同一条路径。

---

#### 4.4.3 第二层：`Tarball.packageTarball()` — 分支决策与 npm 校验

核心代码 [Tarball.ts:L28-L80](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts#L28-L80)：

```typescript
private async packageTarball({ name, version }: PackageInstallation) {
  const fullPackageName = `${name}@${version}`;  // 注意：version 可能是 null，拼接成 "@xxx/plugin@null"
  const response = await packument(fullPackageName); // pacote 会忽略 @null，实际请求的是包名本身的 packument
  const latestTag = response["dist-tags"].latest;
  const latestVersion = response.versions[latestTag];
  const requestedVersion = response.versions[version || ""]; // version 为 null 时变成 response.versions[""] → undefined

  // ─── 分支 1：version 为 null 或字符串 "latest" ───
  if (!version || version === "latest") {
    return {
      tarball: latestVersion.dist.tarball,
      version: {
        requestedFullPackageName: fullPackageName,  // "@xxx/plugin@null"
        packageName: name,
        packageVersion: latestVersion.version,       // 实际解析到的版本号，如 "1.5.0"
      },
    };
  }

  // ─── 分支 2：version 是合法 SemVer，但 npm 上无此版本 ───
  if (!requestedVersion.version) {
    const suggestionMessage = `Please try to install another version, or the latest version: ${latestVersion}.`;
    await this.logger.error([`${name}@${version} is not available`, suggestionMessage].join(". "));
    throw new Error([`Could not find version ${version} for ${name}`, suggestionMessage].join(". "));
  }

  // ─── 分支 3：version 是合法 SemVer 且 npm 上存在 ───
  if (requestedVersion.deprecated) {
    await this.logger.warn(
      `${name}@${version} is deprecated, update it to avoid issues in the code generation.`
    );
  }

  return {
    tarball: requestedVersion.dist.tarball,
    version: {
      requestedFullPackageName: fullPackageName,
      packageName: name,
      packageVersion: requestedVersion.version,
    },
  };
}
```

**三种"异常"情况的精确处理对照**：

##### ▶️ 情况一：`version = "latest"`（或任何非 SemVer 字符串：`"abc"`、`""`、`null`）

```
DynamicPackageInstallationManager
  → valid("latest") = null
  → 传给 Tarball version: null

Tarball.packageTarball()
  → !version → true  (因为 null 是 falsy)
  → 走分支 1，下载 dist-tags.latest 指向的版本
  → 返回 packageVersion = 实际最新版本号（如 "1.5.0"）
```

**不抛错，不降级，静默用最新版本**。注意代码中 `version === "latest"` 这个判断实际上是"死代码"——因为 `valid("latest")` 已将字符串 `"latest"` 转为 `null`，不可能再等于字符串 `"latest"`。该条件仅作为防御性双保险存在。

##### ▶️ 情况二：`version = "9.9.9"`（合法 SemVer，但 npm 上不存在该版本）

这是整个链路中最复杂、最容易理解错的部分。必须区分**代码作者意图**、**代码即使执行了也存在的 Bug**、**实际执行路径**、**onError 传参**、**多层 catch 链路**和**用户最终看到的构建日志**六层。

---

**代码作者意图（Tarball 分支 2 想做什么）**

代码 [Tarball.ts:L52-L64](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts#L52-L64) 的设计初衷（理想中的行为）是：

```
输入：用户请求了不存在的 9.9.9，npm latest 实际是 1.5.0
期望输出（构建日志）：
  "@xxx/plugin@9.9.9 is not available. Please try to install another version, or the latest version: 1.5.0."
期望抛出：
  Error("Could not find version 9.9.9 for @xxx/plugin. Please try to install another version, or the latest version: 1.5.0.")
```

**但这段代码即使真的执行到了，也输出不了版本号——这是第 4 个 Bug。**

---

**代码即使真执行到也存在的 Bug（Bug 4/4）：latestVersion 是对象，不是字符串**

对照 [Tarball.ts:L37-L38](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts#L37-L38) 和 [Tarball.ts:L53](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts#L53)：

```typescript
const latestTag = response["dist-tags"].latest;          // 字符串 "1.5.0"
const latestVersion = response.versions[latestTag];      // Manifest 对象！不是字符串！
// response.versions 的类型是 { [version: string]: Manifest }
// Manifest 是 npm 包版本的完整元数据对象，包含 name、version、dist、dependencies、engines 等所有字段

const requestedVersion = response.versions[version || ""];

// ...

if (!requestedVersion.version) {
  const suggestionMessage =
    `Please try to install another version, or the latest version: ${latestVersion}.`;
  //                                                        ↑ 这里直接拼接对象
```

**JS 对象在模板字符串中的行为**：普通对象没有自定义 `toString()`，所以 `${latestVersion}` 会被转成字符串 `"[object Object]"`。

对比同文件 L47 的正确写法 [Tarball.ts:L47](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts#L47)（分支 1 里写对了）：

```typescript
// 分支 1（正确）：用 latestVersion.version 取字符串版本号
packageVersion: latestVersion.version,

// 分支 2（错误）：直接用 latestVersion 对象
`Please try to install another version, or the latest version: ${latestVersion}.`
```

**最终 suggestionMessage 实际会是**（如果分支 2 能执行到的话）：
```
"Please try to install another version, or the latest version: [object Object]."
```

用户完全看不到 `"1.5.0"` 这个有用的版本号建议，只能看到无意义的 `[object Object]`。错误提示质量为零。

---

**但这段代码实际上永远不会执行。**

---

**实际执行路径（对照代码逐行）**

```
plugin.version = "9.9.9"（合法 SemVer，但 npm 上无此版本）
  │
  ▼
[DynamicPackageInstallationManager.ts:L24]
  → valid("9.9.9") = "9.9.9"
  → 传给 Tarball version = "9.9.9"
  │
  ▼
[Tarball.ts:L39]
  → response = await packument("@xxx/plugin@9.9.9")
    （pacote 返回包完整 packument，包含所有已存在版本，但不含 9.9.9）
  → latestTag = response["dist-tags"].latest = "1.5.0"（字符串）
  → latestVersion = response.versions["1.5.0"]（Manifest 对象，含 version/name/dist/dependencies 等全部字段）
  → requestedVersion = response.versions["9.9.9"] → undefined
  │
  ▼
[Tarball.ts:L52] if (!requestedVersion.version)
  → requestedVersion 本身就是 undefined
  → JS 试图访问 undefined.version
  → 立即抛出原生 TypeError: Cannot read properties of undefined (reading 'version')
  → ⚠️  if 分支内部的 logger.error 和 throw new Error **完全不会执行**（死代码）
```

---

**onError 钩子传参细节（Bug 2/4）**

TypeError 首先被 [DynamicPackageInstallationManager.ts:L40-L43](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/DynamicPackageInstallationManager.ts#L40-L43) 捕获：

```typescript
} catch (error) {
  onError && (await onError(plugin));   // ← ⚠️ 只传了 plugin，没有传 error！
  throw error;
}
```

对比 [HookErrorFunction 类型定义](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/DynamicPackageInstallationManager.ts#L72-L75)：

```typescript
export type HookErrorFunction = (
  plugin: PackageInstallation,
  error?: Error    // error 被声明为可选参数，但调用方实际上从来不传
) => Promisable<void>;
```

调用方 [dynamic-package-installation.ts:L52-L57](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts#L52-L57)：

```typescript
onError: async (plugin, error) => {
  await buildLogger.error(
    `Failed to installed plugin: ${plugin.name}@${plugin.version}`,
    { ...error }   // ← error 是 undefined！{ ...undefined } = {}（空对象）
  );
},
```

**实际写入构建日志的内容**：只有纯文本 `"Failed to installed plugin: @xxx/plugin@9.9.9"`，第二参数是**空对象 `{}`**，没有任何 TypeError 的 message、stack 或其他错误详情。用户从这条日志里只能看到插件安装失败，完全不知道为什么失败。

---

**TypeError 完整冒泡与暴露路径（4 层 catch）**

`DynamicPackageInstallationManager.install()` 在 L42 `throw error` 后，TypeError 经过 **4 层 catch** 才最终终止 DSG 进程，每层都以不同方式记录日志：

```
  │
  ▼
[DynamicPackageInstallationManager.install()] L42: throw TypeError
  │
  ▼
dynamicPackagesInstallations() —— 无 try-catch，直接冒泡
  │
  ▼
[create-data-service.ts:L34-L39] createDataService()
  → await dynamicPackagesInstallations(...)
     ⚠️  这段调用写在 try 块（L41 开始）的外面！
     所以 [create-data-service.ts:L99-L104] 的 catch 捕获不到它
     TypeError 直接冒泡
  │
  ▼
第 1 层 catch：[generate-code.ts:L52-L63] generateCodeByResourceData()
  → L61: internalLogger.error(error.message, error)
       写入："Cannot read properties of undefined (reading 'version')" + 完整 error 对象
  → L62: throw error  （继续冒泡）
  │
  ▼
第 2 层 catch：[generate-code.ts:L84-L93] generateCode()
  → L90: context.logger.error(`Failed to generate code: ${error.message}`)
       写入构建日志："Failed to generate code: Cannot read properties of undefined (reading 'version')"
  → L91: buildManagerNotifier.failure()   通知 BuildManager 标记构建失败
  → L92: throw error  （继续冒泡）
  │
  ▼
第 3 层 catch：[main.ts:L5-L8]
  → L6: logger.error(err)    再次记录完整 err 对象（进程 stdout/stderr）
  → L7: process.exit(1)      DSG 容器进程以非零状态退出
  │
  ▼
DSG 进程终止，构建失败
```

---

**用户在构建日志里实际能看到的所有内容**

| 日志来源 | 实际输出内容 | 是否会出现 |
|----------|-------------|-----------|
| Tarball 分支 2 友好错误日志 | `"@xxx/plugin@9.9.9 is not available. Please try to install another version, or the latest version: 1.5.0."` | ❌ **永远不出现**（TypeError 先发生，死代码）。即使能执行到，也会是 `"... latest version: [object Object]."`（Bug 4/4），看不到实际版本号。 |
| Tarball 分支 2 抛错信息 | `Error: Could not find version 9.9.9 for @xxx/plugin. Please try...` | ❌ **永远不出现**（同上，死代码）。即使能执行到，message 中同样是 `[object Object]` 而非版本号。 |
| onError 钩子 `buildLogger.error` [dynamic-package-installation.ts:L53-L55](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/dynamic-package-installation.ts#L53-L55) | `"Failed to installed plugin: @xxx/plugin@9.9.9"` + `{}`（空对象，**无任何错误详情**） | ✅ 会出现，但对用户诊断问题几乎没有帮助 |
| `generateCodeByResourceData()` `internalLogger.error` [generate-code.ts:L61](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/generate-code.ts#L61) | `"Cannot read properties of undefined (reading 'version')"` + 完整 error 对象（含 stack） | ✅ 会出现（内部日志，通常不向用户展示） |
| `generateCode()` `context.logger.error` [generate-code.ts:L90](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/generate-code.ts#L90) | `"Failed to generate code: Cannot read properties of undefined (reading 'version')"` | ✅ 会出现在构建日志里（用户能看到） |
| `main.ts` `logger.error(err)` [main.ts:L6](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/main.ts#L6) | 完整 TypeError 对象（含 stack） | ✅ 会出现在容器进程 stderr（运维层面可见） |

---

> 🔍 **四处代码 Bug 汇总**：
>
> 1. **Tarball 分支 2 写法缺陷（分支不可达）** [Tarball.ts:L52](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts#L52)：`if (!requestedVersion.version)` 应改为 `if (!requestedVersion || !requestedVersion.version)`。当前 `requestedVersion = undefined` 时先抛 TypeError，友好错误分支成为死代码。
>
> 2. **Tarball 分支 2 建议文案类型错误（即使可达也无用）** [Tarball.ts:L38 + L53](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts#L38)：`latestVersion` 是 `Manifest` 对象（`response.versions[tag]` 返回 npm 完整版本元数据，含 name/version/dist/dependencies 等字段），不是版本号字符串。模板字符串直接拼接对象会输出 `"[object Object]"`，用户完全看不到建议版本号。正确写法是 `${latestVersion.version}`，同文件 L47 的分支 1 已正确使用该写法。
>
> 3. **onError 不传 error 参数** [DynamicPackageInstallationManager.ts:L41](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/DynamicPackageInstallationManager.ts#L41)：`onError(plugin)` 应改为 `onError(plugin, error)`。当前钩子第二参数永远是 `undefined`，展开后是空对象 `{}`，日志里没有任何错误详情。
>
> 4. **安装阶段在 try 外面** [create-data-service.ts:L34-L39](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/create-data-service.ts#L34-L39)：`dynamicPackagesInstallations()` 写在 L41 try 块外面，安装错误无法走 `createDataService` 内部统一的错误记录路径。
>
> 四重 Bug 叠加的用户体验：用户既看不到 `"请换版本或用最新版"` 的友好建议（分支不可达 + 即使可达也是 `[object Object]`），也看不到具体错误原因（onError 不传 error），只能先看到一条毫无信息量的 `"Failed to installed plugin"`，再看到一条晦涩的 `"Failed to generate code: Cannot read properties of undefined (reading 'version')"`。

##### ▶️ 情况三：`version = "1.0.0"`（合法 SemVer，npm 上存在，但被标记 deprecated）

```
DynamicPackageInstallationManager
  → valid("1.0.0") = "1.0.0"
  → 传给 Tarball version: "1.0.0"

Tarball.packageTarball()
  → 跳过分支 1、2
  → requestedVersion.deprecated → 真值
  → logger.warn("xxx is deprecated, update it...")
  → 继续执行分支 3，正常下载并返回 tarball
```

**仅警告，不中断，不降级**，继续使用该弃用版本完成构建。

---

#### 4.4.4 第三层：安装结果上报与版本号记录

安装成功后的版本上报 [DynamicPackageInstallationManager.ts:L32-L38](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/DynamicPackageInstallationManager.ts#L32-L38)：

```typescript
if (!settings?.local) {
  const installedVersion = await tarball.download();
  onAfterInstall &&
    (await onAfterInstall(plugin, {
      ...installedVersion,
      requestedFullPackageName: `${name}@${version}`, // ← 使用原始 version，不是 validVersion！
    }));
}
```

**关键细节**：上报给 BuildManager 的 `requestedFullPackageName` 使用**用户原始输入的 `version` 字符串**（如 `"latest"`），而非 Tarball 内部拼接的 `"@xxx/plugin@null"`，也不是规范化后的 SemVer。这样构建历史记录中看到的是：

| 用户设置 `plugin.version` | 实际安装 `packageVersion` | 上报 `requestedFullPackageName` |
|--------------------------|---------------------------|--------------------------------|
| `"latest"` | `"1.5.0"` | `"@amplication/plugin-db-postgres@latest"` |
| `"1.2.3"` | `"1.2.3"` | `"@amplication/plugin-db-postgres@1.2.3"` |
| `"v1.2.3"` | `"1.2.3"` | `"@amplication/plugin-db-postgres@v1.2.3"` |

---

#### 4.4.5 校验流程汇总图

```
PluginInstallation.version
  │  （字符串："latest" / "1.2.3" / "v1.2.3" / "9.9.9" / "invalid"）
  ▼
┌─────────────────────────────────────────────────┐
│ DynamicPackageInstallationManager.install()      │
│ semver.valid(version)                            │
└──────────────┬──────────────────────────────────┘
               │
     ┌─────────┴──────────┐
     │                    │
  null               "1.2.3" 等合法 SemVer
  （"latest"/"invalid"/""）
     │                    │
     ▼                    ▼
┌────────────────┐   ┌──────────────────────────────┐
│ Tarball 分支 1  │   │ Tarball 分支 2/3             │
│ !version=true   │   │ !version=false              │
│ 下载 latest 版  │   │ 查 response.versions[version]│
│ 不报错          │   └──────────┬───────────────────┘
└────────────────┘              │
                          ┌─────┴──────┐
                          │            │
                   requestedVersion   requestedVersion = undefined
                   存在               （npm 上无此版本）
                          │            │
                          ▼            ▼
                   ┌────────────┐  ┌──────────────────────────────────────┐
                   │ deprecated?│  │ TypeError: 访问 undefined.version    │
                   │ 是→warn    │  │ ↓                                    │
                   │ 否→无操作  │  │ DynamicPackageInstallationManager   │
                   └─────┬──────┘  │   catch(plugin) → onError(plugin)    │
                         │          │   ⚠️ 只传 plugin，不传 error！       │
                         ▼          │   ↓                                    │
                   正常下载 tarball  │   onError 钩子：                      │
                   解压到 modules/   │     "Failed to installed plugin" + {} │
                         │          │   ↓                                    │
                         ▼          │   throw TypeError（重新抛出）          │
               onAfterInstall       │   ↓                                    │
               上报 BuildManager    │ ╔════════════════════════════════════╗  │
               （用原始 version）   │ ║ createDataService 中在 try 外    ║  │
                                    │ ║ generateCodeByResourceData: catch ║  │
                                    │ ║   internalLogger.error(err)       ║  │
                                    │ ║ generateCode: catch               ║  │
                                    │ ║   "Failed to generate code: ..."  ║  │
                                    │ ║   buildManagerNotifier.failure()  ║  │
                                    │ ║ main.ts: catch                    ║  │
                                    │ ║   logger.error(err) + exit(1)     ║  │
                                    │ ╚════════════════════════════════════╝  │
                                    │   ↓                                    │
                                    │ DSG 进程终止，构建失败                │
                                    └──────────────────────────────────────┘

  ⚠️  4 处 Bug 汇总：
      ① Tarball: if(!requestedVersion.version) 死代码
         （应写为 if(!requestedVersion || !requestedVersion.version)）
      ② Tarball: 建议文案 ${latestVersion} 直接拼对象
         （latestVersion 是 Manifest 对象，输出 [object Object]，
          正确写法是 ${latestVersion.version}，分支 1 已写对）
      ③ DynamicPackageInstallationManager: onError 不传 error 参数
      ④ createDataService: 动态安装调用在 try 块外面
```

### 4.5 插件安装配置校验

在 `amplication-server` 的 `PluginInstallationService` 中，见 [pluginInstallation.service.ts:L97-L119](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L97-L119)：

```typescript
async validatePluginConfiguration(resourceId, configurations, user) {
  // 例如 AUTH 插件要求 requireAuthenticationEntity=true
  // 此时必须校验资源已配置 Authentication Entity
  if (configurations[REQUIRES_AUTHENTICATION_ENTITY] !== "true") return;
  const authEntity = await this.resourceService.getAuthEntityName(resourceId, user);
  if (isEmpty(authEntity)) {
    throw new AmplicationError(
      "The plugin requires an authentication entity. Please select the authentication entity in the service settings."
    );
  }
}
```

此外还会校验**同一插件不重复安装** [pluginInstallation.service.ts:L133-L146](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts#L133-L146)。

### 4.6 插件与 DSG 的隐式兼容：事件系统

插件通过 `register()` 返回事件钩子（before/after）与 DSG 交互，见 [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/plugin-wrapper.ts)。这种设计天然解耦：

- **向前兼容**：DSG 新增事件 → 旧插件忽略（无该事件钩子，不影响运行）
- **向后兼容**：插件依赖新事件 → 旧 DSG 无该事件 → 插件逻辑不触发（但插件本身加载正常）

---

## 五、发布状态与完整流程串联

### 5.1 状态字段总览

| 字段 | 所在模型 | 含义 | 设置时机 |
|------|----------|------|----------|
| `Plugin.taggedVersions` | Plugin | npm dist-tags（如 `{ "latest": "1.2.3", "next": "2.0.0-beta.1" }`） | 从 GitHub+npm 同步时 |
| `Plugin.downloads` | Plugin | npm 最近一周下载量 | 同步时计算 |
| `PluginVersion.isLatest` | PluginVersion | 是否为 npm `latest` 标签指向的版本 | 同步时与 `dist-tags.latest` 比较 |
| `PluginVersion.deprecated` | PluginVersion | npm deprecated 字段（弃用提示信息） | 同步时读取 |
| `Version.isActive` | Version (DSG) | 该 DSG 版本是否允许用户选择使用 | 管理员手动激活 |
| `Version.isDeprecated` | Version (DSG) | 该 DSG 版本已弃用 | ECR tag 被删除时自动标记 |

### 5.2 插件发布 CI 工作流

#### 平台整体发布 [release.production.yml](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/.github/workflows/release.production.yml)

```
workflow_dispatch (手动触发，输入 version)
  → SemVer 格式校验 (^v[0-9]+\.[0-9]+\.[0-9]+$)
  → 检查 git tag 是否已存在
  → nx affected 检测是否有需要发布的应用
  → 更新各 app 的 version.ts
  → 自动 commit + 创建 GitHub Release（含 changelog）
```

#### DSG 独立发布 [release.dsg.production.yml](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/.github/workflows/release.dsg.production.yml)

DSG 作为核心组件有独立发布流程：

```
workflow_dispatch (输入 version)
  → 校验 package.json 中当前版本不等于输入版本
  → 校验 git tag dsg/{version} 不存在
  → 更新 data-service-generator/package.json 的 version 字段
  → nx package:container（构建 Docker 镜像并推送到 AWS ECR，打上版本 tag）
  → 推送 git tag dsg/{version}
  → 创建 GitHub Release（仅包含 app:data-service-generator 标签的 PR）
```

镜像推送到 ECR 后，`data-service-generator-catalog` 的 `syncVersions()` 定时任务会发现新 tag 并创建 Version 记录（isActive=false，管理员激活后用户可选择）。

### 5.3 完整端到端流程：从插件发布到代码生成

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     阶段 1: 插件开发者发布插件                                │
│                                                                              │
│  ① 编写插件代码，实现 AmplicationPlugin 接口                                 │
│  ② 根目录放置 .amplicationrc.json                                            │
│  ③ npm publish（或使用 scripts/publish.mjs）                                 │
│        ↓                                                                     │
│  ④ 向 amplication/plugin-catalog 仓库提交 PR，新增 plugins/{pluginId}.yml    │
└──────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                阶段 2: amplication-plugin-api 同步元数据                     │
│                                                                              │
│  ⑤ PluginService.processCatalogPlugins()                                     │
│     → 从 GitHub plugin-catalog 拉取所有 .yml                                 │
│     → 并行调用 npm packument + downloads API                                 │
│     → upsert 到 Plugin 表（含 taggedVersions、downloads）                    │
│                                                                              │
│  ⑥ PluginVersionService.processPluginsVersions(plugins)                      │
│     → NpmPluginVersionService.getAllPluginsVersions()                        │
│       → 对每个插件调用 npm packument                                         │
│       → SemVer 正则过滤版本（可配置是否忽略预发布）                           │
│     → 对每个新版本：                                                         │
│       → 下载 tarball，解压读取 .amplicationrc.json                           │
│       → 创建 PluginVersion 记录（settings、configurations、isLatest、deprecated）│
│     → 对已有版本：仅更新 deprecated 或 isLatest 变化                         │
└──────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                  阶段 3: 用户安装与版本选择                                   │
│                                                                              │
│  ⑦ amplication-client 从 PluginCatalogService 查询可用插件                  │
│     → GraphQL 请求 amplication-plugin-api                                    │
│     → 过滤 conditions: codeGeneratorName、deprecated==null                   │
│     → 按 isLatest 标识获取最新可用版本                                       │
│                                                                              │
│  ⑧ 用户在 UI 中启用插件 → PluginInstallationService.create()                 │
│     → validatePluginConfiguration()（如 AUTH 插件需认证实体）                │
│     → 检查同资源下不重复安装                                                 │
│     → 创建 PluginInstallation Block（保存 pluginId、npm、version、enabled）  │
│     → setOrder() 将插件加入执行顺序链                                        │
└──────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                    阶段 4: 构建时动态加载与校验                               │
│                                                                              │
│  ⑨ 用户触发构建 → amplication-server BuildService                            │
│     → getOrderedPluginInstallations(resourceId)（按 order 排序）             │
│     → 如有私有插件：通过 git-sync-manager 下载到 dsg-assets/private-plugins/ │
│     → 选择 DSG 版本（LatestMajor / LatestMinor / Specific）                  │
│       → CodeGeneratorCatalogService.getCodeGeneratorVersion()               │
│                                                                              │
│  ⑩ DSG (data-service-generator) 容器启动                                    │
│     → createDataService() [create-data-service.ts:L15-L105](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/create-data-service.ts#L15-L105)│
│       → prepareDefaultPlugins()（如未装数据库插件，自动补 db-postgres，version="latest"）│
│       → dynamicPackagesInstallations() [dsg-utils]                           │
│            → DynamicPackageInstallationManager.install()                     │
│                 → semver.valid(version) 规范化（"latest"→null, "v1.2.3"→"1.2.3"）│
│                 → Tarball.download()                                         │
│                      → Tarball.packageTarball() 三分支决策：                  │
│                         • version=null（含 "latest"/非法字符串）→ 下 latest 版 │
│                         • 合法 SemVer 但 npm 不存在 → TypeError（访问 undefined.version）│
│                                              → onError: "Failed to installed plugin"│
│                                              → rethrow TypeError → 中断 DSG  │
│                                              （注意：友好错误分支是死代码）     │
│                         • 合法 SemVer 存在 → deprecated? warn+继续 : 直接继续 │
│                      → 下载 tarball 解压到 amplication_modules/             │
│                 → onAfterInstall：用原始 version 字符串上报 BuildManager     │
│                                                                              │
│  ⑪ prepareContext() → registerPlugins()                                      │
│     → 动态 import 每个插件包的 default export                                │
│     → new PluginClass().register() 获取 Events 钩子映射                      │
│     → 组装 PluginMap: { eventName: { before: [...], after: [...] } }         │
│     → 存入 DsgContext.plugins                                                │
│                                                                              │
│  ⑫ 代码生成各阶段通过 pluginWrapper() 执行插件钩子                           │
│     → beforeEventsPipe（顺序链式修改 eventParams）                           │
│     → DSG 默认行为（可被 skipDefaultBehavior 跳过）                          │
│     → afterEventsPipe（顺序链式修改生成的 ModuleMap）                        │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 默认插件机制

当用户未安装某类关键插件时，系统会自动补充默认插件，见 [defaultPlugins.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/utils/dynamic-installation/defaultPlugins.ts)：

```typescript
const defaultPlugins: DefaultPlugin[] = [
  {
    categoryPluginIds: [POSTGRESQL_PLUGIN_ID, MYSQL_PLUGIN_ID, MONGO_PLUGIN_ID, MSSQL_PLUGIN_ID],
    defaultCategoryPlugin: {
      pluginId: POSTGRESQL_PLUGIN_ID,
      npm: "@amplication/plugin-db-postgres",
      version: "latest",  // 注意：字符串 "latest" 不是合法 SemVer，经 semver.valid() 转为 null
      enabled: true,      //       间接触发 Tarball 中 !version 分支 → 下载 npm dist-tags.latest
    },
  },
];
// 用户已安装任一数据库插件 → 不补充；否则自动补 PostgreSQL
```

处理链路对照（详见 4.4.3 情况一）：
```
defaultCategoryPlugin.version = "latest"
  → semver.valid("latest") = null
  → Tarball: !null = true → 走分支 1 → 下载 npm latest 标签版本
```

### 5.5 私有插件的特殊路径

企业用户可将私有插件托管在 Git 仓库中，流程见 [private-plugin.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/ee/packages/git-sync-manager/src/private-plugin/private-plugin.service.ts)：

```
BuildService.getPrivatePluginsWithVersion()
  → 对 version="latest" 的私有插件：
     → 读取 PrivatePlugin Block 的 versions[] 列表
     → 过滤 enabled=true 且不含 "dev"
     → semver.compareBuild() 降序排序取最大
  → 按 pluginRepositoryResourceId 分组（同一 Git 仓库批量下载）
  → Kafka 发送 DOWNLOAD_PRIVATE_PLUGINS 消息
  → git-sync-manager 消费消息：
     → GitClientService.downloadPrivatePlugins()（clone 指定分支）
     → copyPluginFilesToDsgAssetsDir() 复制到 dsg-assets/{resourceId}-{buildId}/private-plugins/
  → DSG registerPlugins() 时通过 isPrivate 标志从本地路径 import（非 npm）
```

---

## 六、关键文件速查表

| 关注点 | 文件路径 |
|--------|----------|
| 插件数据模型 | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/prisma/schema.prisma) |
| DSG 版本模型 | [catalog/schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator-catalog/prisma/schema.prisma) |
| GitHub 插件同步 | [github-plugin.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/plugin/github-plugin.service.ts) |
| npm 版本拉取与过滤 | [npm-plugin-version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/pluginVersion/npm-plugin-version.service.ts) |
| 插件版本 upsert 与设置提取 | [pluginVersion.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-plugin-api/src/pluginVersion/pluginVersion.service.ts) |
| 插件安装与配置校验 | [pluginInstallation.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-server/src/core/pluginInstallation/pluginInstallation.service.ts) |
| DSG 动态安装插件 | [DynamicPackageInstallationManager.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/DynamicPackageInstallationManager.ts) |
| Tarball 下载与版本存在性校验 | [Tarball.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/libs/util/dsg-utils/src/dynamic-installation/Tarball.ts) |
| DSG 插件注册与事件映射 | [register-plugin.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/register-plugin.ts) |
| DSG 插件事件执行包装器 | [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator/src/plugin-wrapper.ts) |
| DSG 版本同步与策略 | [version.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/data-service-generator-catalog/src/version/version.service.ts) |
| DSG 版本比较 | [code-generator-catalog.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/packages/amplication-build-manager/src/code-generator/code-generator-catalog.service.ts) |
| npm 发布脚本 | [publish.mjs](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/scripts/publish.mjs) |
| DSG 生产发布 CI | [release.dsg.production.yml](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/.github/workflows/release.dsg.production.yml) |
| 私有插件下载 | [private-plugin.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/104-amplication/ee/packages/git-sync-manager/src/private-plugin/private-plugin.service.ts) |
