# Amplication 代码生成中的 Formatter 与 Lint 集成分析

## 1. 概览

Amplication 的代码生成流程中，**格式化（Formatter）** 是通过 [Prettier](https://prettier.io/) 实现的，在生成过程的多个阶段对不同模块的代码分别进行格式化；**Lint** 并没有真正调用 ESLint 进行静态分析检查，而是通过 AST 清理工具从模板代码中移除 `eslint-disable` 等注释和 TypeScript 声明，避免这些开发辅助内容出现在最终产物中。

---

## 2. 核心工具位置与实现

### 2.1 formatCode — 代码格式化函数

**位置**：[files.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/files.ts#L7-L24)

```typescript
export const formatCode = (path: string, code: string): string => {
  if (path.endsWith(".ts") || path.endsWith(".tsx")) {
    return format(code, { parser: "typescript" });
  }
  if (path.endsWith(".json")) {
    return format(code, { parser: "json" });
  }
  if (path.endsWith(".yml") || path.endsWith(".yaml")) {
    return format(code, { parser: "yaml" });
  }
  if (path.endsWith(".md")) {
    return format(code, { parser: "markdown" });
  }
  if (path.endsWith(".graphql")) {
    return format(code, { parser: "graphql" });
  }
  return code;
};
```

**说明**：
- 底层依赖 `prettier` 的 `format()` 函数
- 根据文件后缀选择对应的 parser：`typescript` / `json` / `yaml` / `markdown` / `graphql`
- 不支持的后缀（如 `.prisma`, `.env`, `.js` 等）直接原样返回
- 从 [files.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/files.spec.ts) 可见其单测覆盖了所有支持的文件类型

### 2.2 removeESLintComments — 移除 ESLint 注释

**位置**：[main.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-eslint-comments/main.ts#L8-L18)

```typescript
export function removeESLintComments(ast: ASTNode): void {
  visit(ast, {
    visitComment(path) {
      const comment = path.value as namedTypes.Comment;
      if (comment.value.match(/^\s+eslint-disable/)) {
        path.prune();
      }
      this.traverse(path);
    },
  });
}
```

**说明**：
- 在 **AST 层面** 移除匹配 `eslint-disable` 的注释节点
- 并非实际执行 ESLint lint 检查，只是清理模板中用于本地开发的 lint 抑制注释
- 同目录下还有类似 AST 清理工具：
  - [remove-ts-ignore-comments](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-ignore-comments) — 移除 `@ts-ignore`
  - [remove-ts-variable-declares](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-variable-declares) — 移除 `declare var`
  - [remove-ts-class-declares](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-class-declares) — 移除 `declare class`
  - [remove-ts-interface-declares](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-interface-declares) — 移除 `declare interface`
  - [remove-imports-ts-ignore-comments](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-imports-typescript-ignore-comments) — 移除 import 上的 `@ts-ignore`

**重要结论：Amplication 代码生成流程中不存在实际的 ESLint Lint 校验步骤，只有 AST 层面的注释清理。**

### 2.3 ModuleMap / FileMap — 代码承载容器

**位置**：[file-map.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-types/src/files/file-map.ts)

关键方法：
- `replaceFilesCode(fn: (path: string, code: T) => T)` — 遍历所有文件，逐个转换代码内容（[L106-L111](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-types/src/files/file-map.ts#L106-L111)）
- `replaceFilesPath(fn: (path: string) => string)` — 遍历所有文件，逐个转换路径
- `merge(anotherMap)` / `mergeMany(maps)` — 合并多个文件映射

所有格式化操作都通过 `replaceModulesCode` 批量应用。

---

## 3. 主流程入口（两条独立流水线）

### 3.1 data-service-generator（传统字符串模板方式）

```
main.ts
  → generate-code.ts::generateCode()                          [读取 BUILD_SPEC_PATH → 调 generateCodeByResourceData]
    → create-data-service.ts::createDataService()              [核心生成入口]
      ├─ prepareContext()                                      [准备 DsgContext、注册插件]
      ├─ createDTOs()                                          [生成 DTO 定义]
      ├─ createServer()                                        [生成 Server 端代码，内部含格式化]
      ├─ createAdminModules()                                  [生成 Admin UI 代码，内部含格式化]
      └─ modules.replaceModulesPath(normalize)                 [统一转换为 Unix 路径分隔符]
    → writeModules()                                           [逐个写文件到磁盘]
```

### 3.2 generator-blueprints（新一代 AST 节点方式）

```
main.ts
  → generate-code.ts::generateCode()
    → create-data-service.ts::createDataService()
      ├─ prepareContext()
      ├─ createBlueprint()                                     [使用 AST 节点生成，不显式格式化]
      └─ context.files.replaceFilesPath(normalize)
    → writeModules()                                           [file.code.toString() 时由 AST Writer 隐式格式化]
```

**区别**：`generator-blueprints` 不调用 `formatCode`，它的每个 `IAstNode` 在 `toString()` 时已经由其 Writer 输出格式化好的代码。

---

## 4. Server 端格式化处理顺序（data-service-generator）

**位置**：[create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts)

`createServerInternal()` 函数中的执行顺序：

| 步骤 | 操作 | 是否格式化 |
|------|------|-----------|
| 1 | `readStaticModules()` — 读取静态模板文件 | 否（静态文件已预格式化） |
| 2 | `createCustomDtos()` | 否 |
| 3 | `createGitIgnore()` | 否 |
| 4 | `createServerPackageJson()` | 后续步骤 16 格式化 |
| 5 | `createDTOModules()` | 后续步骤 14 格式化 |
| 6 | `createResourcesModules()` | 后续步骤 13 格式化 |
| 7 | `createCustomModulesModules()` | 否 |
| 8 | `createAuthModules()` | 后续步骤 18 格式化 |
| 9 | `createSwagger()` | 后续步骤 15 格式化 |
| 10 | `createSeed()` | 后续步骤 17 格式化 |
| 11 | `createMessageBroker()` | 后续步骤 19 格式化 |
| 12 | `createSecretsManager()` | 否 |
| 13 | **格式化 resourcesModules** | ✅ `replaceModulesCode(formatCode)` — [L104-L106](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L104-L106) |
| 14 | **格式化 dtoModules** | ✅ `replaceModulesCode(formatCode)` — [L108](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L108) |
| 15 | **格式化 swagger** | ✅ `replaceModulesCode(formatCode)` — [L110](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L110) |
| 16 | **格式化 appModule** | ✅ `replaceModulesCode(formatCode)` — [L112](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L112) |
| 17 | **格式化 seedModule** | ✅ `replaceModulesCode(formatCode)` — [L114](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L114) |
| 18 | **格式化 authModules** | ✅ `replaceModulesCode(formatCode)` — [L116](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L116) |
| 19 | **格式化 messageBrokerModules** | ✅ `replaceModulesCode(formatCode)` — [L118-L120](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L118-L120) |
| 20 | **格式化 packageJsonModule** | ✅ `replaceModulesCode(formatCode)` — [L122-L124](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L122-L124) |
| 21 | `createTypesRelatedFiles()` — **内部自格式化** | ✅ 函数内部调 `replaceModulesCode(formatCode)` — [create-types-related-files.ts#L29](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-types-related-files/create-types-related-files.ts#L29) |
| 22 | `createMainFile()` — **内部自格式化** | ✅ 函数内部调 `replaceModulesCode(formatCode)` — [create-main-file.ts#L69](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-main/create-main-file.ts#L69) |
| 23 | `createPrismaSchemaModule()` | 否（Prisma schema 不使用 Prettier） |
| 24 | `createDotEnvModule()` | 否 |
| 25 | `connectMicroservices()` | 否 |
| 26 | `createDockerComposeFile()` | 否 |
| 27 | `createDockerComposeDevFile()` | 否 |
| 28 | `mergeMany([...所有模块...])` — 汇总到最终 moduleMap | — |

**未被格式化的模块**：`staticModules`、`customDtos`、`gitIgnore`、`customModulesModules`、`secretsManagerModule`、`prismaSchemaModule`、`dotEnvModule`、`connectMicroservicesModule`、`dockerComposeFile`、`dockerComposeDevFile`。

---

## 5. Admin UI 端格式化处理顺序

**位置**：[create-admin.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-admin.ts)

`createAdminModulesInternal()` 函数中的执行顺序：

| 步骤 | 操作 | 是否格式化 |
|------|------|-----------|
| 1 | `readStaticModules()` | 否 |
| 2 | `createGitIgnore()` | 否 |
| 3 | `createAdminUIPackageJson()` | 否 |
| 4 | `createPublicFiles()` | 否 |
| 5 | `createDTOModules()` | 后续统一格式化 |
| 6 | `createEnumRolesModule()` | 后续统一格式化 |
| 7 | `createRolesModule()` | 后续统一格式化 |
| 8 | `createEntityTitleComponents()` + modules | 后续统一格式化 |
| 9 | `createEntitiesComponents()` + modules | 后续统一格式化 |
| 10 | `createAppModule()` | 后续统一格式化 |
| 11 | `createDotEnvModule()` | 否 |
| 12 | **统一批量格式化所有 TS 模块** | ✅ 将 `enumRolesModule`、`rolesModule`、`appModuleMap`、`dtoModuleMap`、`entityTitleComponentsModules`、`entityComponentsModules` 合并到 `tsModules`，然后一次性 `replaceModulesCode(formatCode)` — [L111-L121](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-admin.ts#L111-L121) |
| 13 | `createTypesRelatedFiles()` — **内部自格式化** | ✅ 函数内部调 `replaceModulesCode(formatCode)` — [create-types-related-files.ts#L25](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-types-related-files/create-types-related-files.ts#L25) |
| 14 | `mergeMany([...所有模块...])` | — |

与 Server 端不同，Admin 端将所有 TypeScript 模块合并到一个临时的 `ModuleMap` 中一次性完成格式化，而不是逐个模块分开格式化。

---

## 6. AST 清理（注释移除）处理位置

AST 清理（包括 `removeESLintComments`）只在生成单文件测试用例的场景中显式调用：

**位置**：[create-controller-spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/test/create-controller-spec.ts#L192-L196)

```typescript
removeESLintComments(template);       // 移除 eslint-disable 注释
removeTSIgnoreComments(template);     // 移除 @ts-ignore 注释
removeTSVariableDeclares(template);   // 移除 declare var
removeTSClassDeclares(template);      // 移除 declare class
removeTSInterfaceDeclares(template);  // 移除 declare interface
```

这些清理发生在 **AST 插值完成后、`print(template)` 输出字符串代码之前**。其目的是让 `.template.ts` 模板文件在本地开发时能够通过 TypeScript 类型检查和 ESLint，但生成的最终代码不携带这些声明和抑制注释。

**注意**：常规的业务代码生成（Controller、Service、Resolver、DTO 等）并未调用这些 AST 清理函数，它们的模板文件本身不包含这些需要清理的内容。

---

## 7. Plugin 介入点（Before / After 钩子）

**位置**：[plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/plugin-wrapper.ts)

```
pluginWrapper(func, event, args)
  ├─ beforeEventsPipe(...beforePlugins)     // 插件可以在默认行为前修改 eventParams
  ├─ defaultBehavior()                      // 执行 DSG 默认生成（含内部的格式化）
  └─ afterEventsPipe(...afterPlugins)       // 插件可以在默认行为后修改 modules（ModuleMap）
```

插件通过 `after` 钩子拿到 `ModuleMap` 后，可以自行对其中的代码做进一步格式化或 lint 处理——这是当前代码中插件扩展格式化 / lint 能力的唯一路径。例如插件可以：

```typescript
// 插件 after 钩子示例
async (context, eventParams, modules) => {
  await modules.replaceModulesCode((path, code) => myCustomLintAndFormat(path, code));
  return modules;
}
```

---

## 8. 总结：完整调用链图

### 8.1 data-service-generator（字符串模板 + Prettier）

```
generateCode() [generate-code.ts]
  │
  ├─ createDataService() [create-data-service.ts]
  │    │
  │    ├─ prepareContext() ──→ registerPlugins()
  │    ├─ createDTOs()
  │    │
  │    ├─ createServer() [create-server.ts]
  │    │    │
  │    │    ├─ readStaticModules()
  │    │    ├─ createDTOModules() ──┐
  │    │    ├─ createResourcesModules() ─┤
  │    │    ├─ createSwagger() ───────┤
  │    │    ├─ createAppModule() ─────┤
  │    │    ├─ createSeed() ──────────┤
  │    │    ├─ createAuthModules() ───┤
  │    │    ├─ createMessageBroker() ─┤  分别调用
  │    │    ├─ createServerPackageJson() ─┤
  │    │    │                        │
  │    │    └─ 各自 replaceModulesCode(formatCode)
  │    │       formatCode → prettier.format()
  │    │
  │    │    ├─ createTypesRelatedFiles() ──→ 内部 replaceModulesCode(formatCode)
  │    │    ├─ createMainFile() ──────────→ 内部 replaceModulesCode(formatCode)
  │    │    ├─ createPrismaSchemaModule()  (不格式化)
  │    │    └─ ...其他模块 (不格式化)
  │    │
  │    └─ createAdminModules() [create-admin.ts]
  │         │
  │         ├─ 各类 TS 模块生成 ──→ 合并到 tsModules
  │         └─ tsModules.replaceModulesCode(formatCode)  ← 一次性批量格式化
  │
  └─ writeModules() ──→ 写文件到磁盘
```

### 8.2 generator-blueprints（AST 节点方式）

```
generateCode()
  │
  ├─ createDataService()
  │    │
  │    ├─ prepareContext()
  │    └─ createBlueprint()  (生成 IAstNode 树，无显式 formatCode)
  │
  └─ writeModules()
       └─ file.code.toString()  ← 由每个 AstNode 的 Writer 输出格式化代码
```

---

## 9. 关键设计观察

1. **格式化是分散的，不是统一的**：Server 端按模块类别分批格式化，Admin 端合并后一次性格式化，子模块（typesRelatedFiles、mainFile）各自内部格式化——没有一个"所有模块生成完毕后统一格式化"的步骤。

2. **没有真正的 Lint 校验步骤**：`removeESLintComments` 只是清理模板中的注释，并没有调用 ESLint API 对生成代码做静态分析。生成代码的质量依赖模板本身的正确性和 Prettier 格式化。

3. **部分文件类型不格式化**：`.prisma`、`.env`、`.gitignore`、`docker-compose.yml`（注：yml 支持但 docker 模块未调用 formatCode）等不经过 Prettier。

4. **插件是扩展格式化/Lint 的唯一入口**：通过 `after` 事件钩子，插件可以在 DSG 默认行为之后对 `ModuleMap` 中的任意文件执行自定义格式化或 lint 检查。

5. **两条生成流水线的差异**：旧的 `data-service-generator` 使用字符串拼接 + Prettier 后处理；新的 `generator-blueprints` 使用 AST 节点（`IAstNode`），由 Writer 在 `toString()` 时直接输出格式化代码，不再需要 Prettier 后处理。
