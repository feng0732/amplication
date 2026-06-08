# Amplication 代码生成中的 Formatter 与 Lint 集成分析

## 1. 概览

Amplication 的代码生成流程中存在**两个独立层级**的代码规整操作：

1. **AST 层面的 Lint/TypeScript 清理** — 发生在单文件生成过程中，在 AST 插值完成后、`print()` 输出字符串代码之前执行。清理函数来自 `@amplication/code-gen-utils`，作用是移除模板文件中的开发辅助内容（`eslint-disable`、`@ts-ignore`、`declare var/class/interface`），并非实际运行 ESLint 做静态分析。

2. **Prettier 格式化** — 发生在模块集合（`ModuleMap`）层面，一批文件生成完毕后通过 `replaceModulesCode(formatCode)` 统一调用 `prettier.format()` 进行格式化。

---

## 2. 核心工具清单

### 2.1 六个 AST 清理函数

均来自 `@amplication/code-gen-utils`，位于 [code-gen-utils/src/lib/ast/](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast)：

| 函数 | 作用 | 源码位置 |
|------|------|---------|
| `removeESLintComments(ast)` | 移除匹配 `eslint-disable` 的注释节点 | [remove-eslint-comments/main.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-eslint-comments/main.ts#L8-L18) |
| `removeTSIgnoreComments(ast)` | 移除 `@ts-ignore` 注释 | [remove-typescript-ignore-comments/](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-ignore-comments) |
| `removeImportsTSIgnoreComments(ast)` | 移除 import 语句上的 `@ts-ignore` | [remove-imports-typescript-ignore-comments/](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-imports-typescript-ignore-comments) |
| `removeTSVariableDeclares(ast)` | 移除 `declare var` 声明 | [remove-typescript-variable-declares/](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-variable-declares) |
| `removeTSClassDeclares(ast)` | 移除 `declare class` 声明 | [remove-typescript-class-declares/](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-class-declares) |
| `removeTSInterfaceDeclares(ast)` | 移除 `declare interface` 声明 | [remove-typescript-interface-declares/](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/ast/remove-typescript-interface-declares) |

**结论**：不存在实际的 ESLint 静态分析调用（代码中无 `import eslint`、`new ESLint()`、`eslint.lintFiles()` 等）。

### 2.2 formatCode — Prettier 格式化函数

**位置**：[files.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/files.ts#L7-L24)

根据文件后缀分派 parser：`.ts/.tsx` → `typescript`；`.json` → `json`；`.yml/.yaml` → `yaml`；`.md` → `markdown`；`.graphql` → `graphql`；其他后缀原样返回。

### 2.3 ModuleMap — 批量操作容器

**位置**：[file-map.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-types/src/files/file-map.ts)

- `replaceModulesCode(fn)` — 遍历 map 内所有文件，逐个转换代码（用于批量格式化）
- `replaceModulesPath(fn)` — 遍历转换路径
- `merge(map)` / `mergeMany([...maps])` — 合并多个模块集合

---

## 3. 通用处理模式

对于**每一个**使用 AST 模板（`.template.ts`）生成的 TypeScript 源文件，其标准处理链如下：

```
readFile(templatePath)          // 读取 AST 模板
  → interpolate(ast, mapping)   // 用变量替换模板中的占位符
  → ...各种 AST 操作...         // 添加 import、注入方法、设置权限、移除禁用方法等
  → AST 清理函数调用            // ← ESLint / TS 注释清理发生在这里
  → addImports(ast, [...])      // 补充依赖 import（部分生成在清理之后）
  → addAutoGenerationComment(ast)
  → print(ast).code             // AST → 字符串代码（recast 输出）
```

**关键观察**：
- AST 清理在 `print()` 之前、AST 插值之后执行
- 部分生成路径的 `addImports()` 放在清理**之后**，意味着这些新添加的 import 不会被清理逻辑影响
- 不同生成路径调用的清理函数子集不同，下一节逐路径列出

---

## 4. 各生成路径的 AST 清理覆盖与调用顺序

以下从 Server 端到 Admin 端，逐个生成路径列出具体的清理函数集合和调用顺序。

### 4.1 应用模块 AppModule

**位置**：[server/app-module/create-app-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/app-module/create-app-module.ts#L103-L161)

```typescript
// L136-L156
interpolate(template, templateMapping);
addImports(template, imports);
removeTSIgnoreComments(template);        // 1st
removeESLintComments(template);          // 2nd
removeTSVariableDeclares(template);      // 3rd
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ✅ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ❌ |
| removeTSInterfaceDeclares | ❌ |
| removeImportsTSIgnoreComments | ❌ |

---

### 4.2 资源模块（Entity Module，每个实体）

**位置**：[server/resource/module/create-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/module/create-module.ts)

**4.2.1 `entity.module.ts`**（L98-L189）

```typescript
// L116-L183
interpolate(template, templateMapping);
// ...条件移除 controller/grpc/resolver 引用...
addImports(template, [...imports]);
removeTSIgnoreComments(template);        // L177
removeESLintComments(template);          // L178
removeTSClassDeclares(template);         // L179
// → print(template).code
```

**4.2.2 `entity.module.base.ts`**（L191-L214）

```typescript
// L199-L208
interpolate(template, templateMapping);
removeTSIgnoreComments(template);        // L201
removeESLintComments(template);          // L202
removeTSClassDeclares(template);         // L203
addAutoGenerationComment(template);
// → print(template).code
```

| 清理函数 | entity.module.ts | entity.module.base.ts |
|---------|-----------------|----------------------|
| removeTSIgnoreComments | ✅ | ✅ |
| removeESLintComments | ✅ | ✅ |
| removeTSVariableDeclares | ❌ | ❌ |
| removeTSClassDeclares | ✅ | ✅ |
| removeTSInterfaceDeclares | ❌ | ❌ |
| removeImportsTSIgnoreComments | ❌ | ❌ |

---

### 4.3 资源控制器（REST Controller，每个实体）

**位置**：[server/resource/controller/create-controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/controller/create-controller.ts)

**4.3.1 `entity.controller.ts`**（L197-L237）

```typescript
// L209-L231
interpolate(template, templateMapping);
addImports(template, [serviceImport, baseImport]);
removeTSIgnoreComments(template);        // L223
removeESLintComments(template);          // L224
removeTSVariableDeclares(template);      // L225
removeTSInterfaceDeclares(template);     // L226
removeTSClassDeclares(template);         // L227
// → print(template).code
```

**4.3.2 `entity.controller.base.ts`**（L239-L375）

```typescript
// L258-L369
interpolate(template, templateMapping);
// ...注入 toMany relation 方法、custom action 方法、设置权限、移除禁用方法...
removeTSIgnoreComments(template);        // L349
removeESLintComments(template);          // L350
removeTSVariableDeclares(template);      // L351
removeTSInterfaceDeclares(template);     // L352
removeTSClassDeclares(template);         // L353
// addImports（在清理之后）
addAutoGenerationComment(template);
// → print(template).code
```

| 清理函数 | entity.controller.ts | entity.controller.base.ts |
|---------|---------------------|--------------------------|
| removeTSIgnoreComments | ✅ | ✅ |
| removeESLintComments | ✅ | ✅ |
| removeTSVariableDeclares | ✅ | ✅ |
| removeTSClassDeclares | ✅ | ✅ |
| removeTSInterfaceDeclares | ✅ | ✅ |
| removeImportsTSIgnoreComments | ❌ | ❌ |

---

### 4.4 资源服务（Service，每个实体）

**位置**：[server/resource/service/create-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/service/create-service.ts)

**4.4.1 `entity.service.ts`**（L110-L144）

```typescript
// L121-L139
interpolate(template, templateMapping);
removeTSClassDeclares(template);         // L122（注意：最先调用）
addImports(template, [baseImport]);
removeTSIgnoreComments(template);        // L132
removeESLintComments(template);          // L133
removeTSVariableDeclares(template);      // L134
removeTSInterfaceDeclares(template);     // L135
// → print(template).code
```

**4.4.2 `entity.service.base.ts`**（L146-L282）

```typescript
// L162-L276
interpolate(template, templateMapping);
// ...注入 toMany/toOne relation 方法、custom action 方法、移除禁用方法...
removeTSClassDeclares(template);         // L251（注意：最先调用）
removeTSIgnoreComments(template);        // L252
removeESLintComments(template);          // L253
removeTSVariableDeclares(template);      // L254
removeTSInterfaceDeclares(template);     // L255
// addImports（在清理之后）
addAutoGenerationComment(template);
// → print(template).code
```

| 清理函数 | entity.service.ts | entity.service.base.ts |
|---------|------------------|-----------------------|
| removeTSIgnoreComments | ✅ | ✅ |
| removeESLintComments | ✅ | ✅ |
| removeTSVariableDeclares | ✅ | ✅ |
| removeTSClassDeclares | ✅ | ✅ |
| removeTSInterfaceDeclares | ✅ | ✅ |
| removeImportsTSIgnoreComments | ❌ | ❌ |

**注意**：Service 路径将 `removeTSClassDeclares` 放在所有清理的最前面，而 Controller/Resolver 路径则将其放在最后。

---

### 4.5 资源解析器（GraphQL Resolver，每个实体）

**位置**：[server/resource/resolver/create-resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/resolver/create-resolver.ts)

**4.5.1 `entity.resolver.ts`**（L197-L251）

```typescript
// L211-L245
interpolate(template, templateMapping);
// addImports（在清理之前）
removeTSIgnoreComments(template);        // L236
removeImportsTSIgnoreComments(template); // L237
removeESLintComments(template);          // L238
removeTSVariableDeclares(template);      // L239
removeTSInterfaceDeclares(template);     // L240
removeTSClassDeclares(template);         // L241
// → print(template).code
```

**4.5.2 `entity.resolver.base.ts`**（L253-L423）

```typescript
// L275-L417
interpolate(template, templateMapping);
// ...注入 relation 方法、custom action 方法、设置权限、移除禁用方法...
removeTSIgnoreComments(template);        // L383
removeImportsTSIgnoreComments(template); // L384
removeESLintComments(template);          // L385
removeTSVariableDeclares(template);      // L386
removeTSInterfaceDeclares(template);     // L387
removeTSClassDeclares(template);         // L388
// addImports（在清理之后）
addAutoGenerationComment(template);
// → print(template).code
```

| 清理函数 | entity.resolver.ts | entity.resolver.base.ts |
|---------|-------------------|------------------------|
| removeTSIgnoreComments | ✅ | ✅ |
| removeESLintComments | ✅ | ✅ |
| removeTSVariableDeclares | ✅ | ✅ |
| removeTSClassDeclares | ✅ | ✅ |
| removeTSInterfaceDeclares | ✅ | ✅ |
| **removeImportsTSIgnoreComments** | ✅ | ✅ |

**注意**：Resolver 路径是资源模块生成路径中**唯一**调用 `removeImportsTSIgnoreComments` 的。

---

### 4.6 gRPC 控制器（每个实体）

**位置**：[server/resource/grpc-controller/create-grpc-controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/grpc-controller/create-grpc-controller.ts)

**4.6.1 `entity.grpc.controller.ts`**（L178-L222）

```typescript
// L194-L216
interpolate(template, templateMapping);
addImports(template, [serviceImport, baseImport]);
removeTSIgnoreComments(template);        // L208
removeESLintComments(template);          // L209
removeTSVariableDeclares(template);      // L210
removeTSInterfaceDeclares(template);     // L211
removeTSClassDeclares(template);         // L212
// → print(template).code
```

**4.6.2 `entity.grpc.controller.base.ts`**（L224-L340）

```typescript
// L246-L334
interpolate(template, templateMapping);
// ...注入 toMany relation 方法、设置权限...
// addImports（在清理之前）
removeTSIgnoreComments(template);        // L325
removeESLintComments(template);          // L326
removeTSVariableDeclares(template);      // L327
removeTSInterfaceDeclares(template);     // L328
removeTSClassDeclares(template);         // L329
addAutoGenerationComment(template);
// → print(template).code
```

| 清理函数 | entity.grpc.controller.ts | entity.grpc.controller.base.ts |
|---------|---------------------------|-------------------------------|
| removeTSIgnoreComments | ✅ | ✅ |
| removeESLintComments | ✅ | ✅ |
| removeTSVariableDeclares | ✅ | ✅ |
| removeTSClassDeclares | ✅ | ✅ |
| removeTSInterfaceDeclares | ✅ | ✅ |
| removeImportsTSIgnoreComments | ❌ | ❌ |

---

### 4.7 控制器测试用例（每个实体）

**位置**：[server/resource/test/create-controller-spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/test/create-controller-spec.ts#L192-L196)

```typescript
removeESLintComments(template);       // L192
removeTSIgnoreComments(template);     // L193
removeTSVariableDeclares(template);   // L194
removeTSClassDeclares(template);      // L195
removeTSInterfaceDeclares(template);  // L196
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ✅ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ✅ |
| removeTSInterfaceDeclares | ✅ |
| removeImportsTSIgnoreComments | ❌ |

---

### 4.8 Swagger 模块

**位置**：[server/swagger/create-swagger.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/swagger/create-swagger.ts#L54-L73)

```typescript
interpolate(template, templateMapping);
removeTSVariableDeclares(template);   // L62
removeTSIgnoreComments(template);     // L63
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ❌ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ❌ |
| removeTSInterfaceDeclares | ❌ |
| removeImportsTSIgnoreComments | ❌ |

---

### 4.9 Seed 脚本

**位置**：[server/seed/create-seed.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/seed/create-seed.ts#L126-L154)

```typescript
interpolate(template, templateMapping);
removeTSVariableDeclares(template);   // L137
// addImports（在清理之后）
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ❌ |
| removeESLintComments | ❌ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ❌ |
| removeTSInterfaceDeclares | ❌ |
| removeImportsTSIgnoreComments | ❌ |

---

### 4.10 GraphQL DTO Args（Create/Update/Delete/FindOne/FindMany/Count/RelationFilter）

**位置**：`server/resource/dto/graphql/*/create-*.ts`（共 7 个文件）

以 [create-create-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/dto/graphql/create/create-create-args.ts#L9-L30) 为例：

```typescript
interpolate(file, mapping);
removeTSClassDeclares(file);          // L27
// 不直接 print，返回 classDeclaration 嵌入上层文件
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ❌ |
| removeESLintComments | ❌ |
| removeTSVariableDeclares | ❌ |
| removeTSClassDeclares | ✅ |
| removeTSInterfaceDeclares | ❌ |
| removeImportsTSIgnoreComments | ❌ |

---

### 4.11 自定义模块（Custom Module）

**位置**：[server/custom-module/](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/custom-module)

调用顺序由 [create-custom-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/custom-module/create-custom-module.ts) 编排：

```
createServiceModules → createCustomModuleControllerModules → createCustomModuleResolverModules → createCustomModule
```

**4.11.1 自定义模块 Module（`custommodule.module.ts`）**

**位置**：[custom-module/create-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/custom-module/custom-module/create-module.ts#L75-L141)

```typescript
interpolate(template, templateMapping);
// ...条件移除 controller/resolver 引用...
addImports(template, [...imports]);
removeTSIgnoreComments(template);        // L129
removeESLintComments(template);          // L130
removeTSClassDeclares(template);         // L131
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ✅ |
| removeTSVariableDeclares | ❌ |
| removeTSClassDeclares | ✅ |
| removeTSInterfaceDeclares | ❌ |
| removeImportsTSIgnoreComments | ❌ |

**4.11.2 自定义模块 Controller（`custommodule.controller.ts`）**

**位置**：[custom-controller/create-controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/custom-module/custom-controller/create-controller.ts#L88-L140)

```typescript
interpolate(template, templateMapping);
// 注入 custom action 方法
addImports(template, [serviceImport, ...dtoImports, ...identifierImports]);
removeTSIgnoreComments(template);        // L126
removeESLintComments(template);          // L127
removeTSVariableDeclares(template);      // L128
removeTSInterfaceDeclares(template);     // L129
removeTSClassDeclares(template);         // L130
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ✅ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ✅ |
| removeTSInterfaceDeclares | ✅ |
| removeImportsTSIgnoreComments | ❌ |

**4.11.3 自定义模块 Service（`custommodule.service.ts`）**

**位置**：[custom-service/create-service.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/custom-module/custom-service/create-service.ts#L63-L103)

```typescript
interpolate(template, templateMapping);
// 注入 custom action 方法
removeTSClassDeclares(template);         // L84（最先调用）
removeTSIgnoreComments(template);        // L85
removeESLintComments(template);          // L86
removeTSVariableDeclares(template);      // L87
removeTSInterfaceDeclares(template);     // L88
// addImports（在清理之后）
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ✅ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ✅ |
| removeTSInterfaceDeclares | ✅ |
| removeImportsTSIgnoreComments | ❌ |

**4.11.4 自定义模块 Resolver（`custommodule.resolver.ts`）**

**位置**：[custom-resolver/create-resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/custom-module/custom-resolver/create-resolver.ts#L78-L132)

```typescript
interpolate(template, templateMapping);
// 注入 custom action 方法
addImports(template, [serviceImport, ...dtoImports, ...identifierImports]);
removeTSIgnoreComments(template);        // L117
removeImportsTSIgnoreComments(template); // L118
removeESLintComments(template);          // L119
removeTSVariableDeclares(template);      // L120
removeTSInterfaceDeclares(template);     // L121
removeTSClassDeclares(template);         // L122
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ✅ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ✅ |
| removeTSInterfaceDeclares | ✅ |
| **removeImportsTSIgnoreComments** | ✅ |

**注意**：自定义模块 Resolver 和资源 Resolver 一样调用了全部 6 个清理函数。

---

### 4.12 Admin UI — App.tsx

**位置**：[admin/app/create-app.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/app/create-app.ts#L40-L96)

```typescript
interpolate(template, mapping);
removeTSVariableDeclares(template);      // L68
removeTSIgnoreComments(template);        // L69
// addImports（在清理之后）
// → print(template).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ❌ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ❌ |
| removeTSInterfaceDeclares | ❌ |
| removeImportsTSIgnoreComments | ❌ |

---

### 4.13 Admin UI — Entity 组件

**位置**：[admin/entity/create-entity-component-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/entity/create-entity-component-module.ts#L9-L18)

```typescript
removeTSVariableDeclares(component.file);   // L12
removeTSInterfaceDeclares(component.file);  // L13
removeTSIgnoreComments(component.file);     // L14
// → print(component.file).code
```

| 清理函数 | 是否调用 |
|---------|---------|
| removeTSIgnoreComments | ✅ |
| removeESLintComments | ❌ |
| removeTSVariableDeclares | ✅ |
| removeTSClassDeclares | ❌ |
| removeTSInterfaceDeclares | ✅ |
| removeImportsTSIgnoreComments | ❌ |

---

### 4.14 AST 清理覆盖汇总表

| 生成路径 | removeTSIgnoreComments | removeESLintComments | removeTSVariableDeclares | removeTSClassDeclares | removeTSInterfaceDeclares | removeImportsTSIgnoreComments |
|---------|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| **AppModule** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **Entity Module** (.module.ts / .module.base.ts) | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **Entity Controller** (.controller.ts / .controller.base.ts) | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Entity Service** (.service.ts / .service.base.ts) | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Entity Resolver** (.resolver.ts / .resolver.base.ts) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Entity gRPC Controller** (.grpc.controller.ts / .grpc.controller.base.ts) | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Entity Controller Spec** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Swagger** | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Seed** | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **GraphQL DTO Args** (7种) | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Custom Module** (.module.ts) | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **Custom Controller** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Custom Service** | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Custom Resolver** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Admin App.tsx** | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Admin Entity Components** | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ |

**覆盖最完整的路径**：Entity Resolver、Custom Resolver（全部 6 个）
**覆盖最少的路径**：Seed（仅 1 个）、GraphQL DTO Args（仅 1 个）

---

## 5. 资源生成完整调用顺序（单实体）

**位置**：[create-resource.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/create-resource.ts#L32-L133)

```
createResourceModules(entity, dtoNameToPath)
  │
  ├─ 1. createServiceModules()
  │      ├─ createServiceModule()          → entity.service.ts         [AST清理：5个函数]
  │      └─ createServiceBaseModule()      → entity.service.base.ts    [AST清理：5个函数]
  │
  ├─ 2. createControllerModules()          [generateRestApi=true 时]
  │      ├─ createControllerModule()       → entity.controller.ts      [AST清理：5个函数]
  │      └─ createControllerBaseModule()   → entity.controller.base.ts [AST清理：5个函数]
  │
  ├─ 3. createGrpcControllerModules()      [generateGrpc=true 时]
  │      ├─ createGrpcControllerModule()       → entity.grpc.controller.ts      [AST清理：5个函数]
  │      └─ createGrpcControllerBaseModule()   → entity.grpc.controller.base.ts [AST清理：5个函数]
  │
  ├─ 4. createResolverModules()            [generateGraphQL=true 时]
  │      ├─ createResolverModule()         → entity.resolver.ts        [AST清理：全部6个函数]
  │      └─ createResolverBaseModule()     → entity.resolver.base.ts   [AST清理：全部6个函数]
  │
  ├─ 5. createModules()
  │      ├─ createModule()                 → entity.module.ts          [AST清理：3个函数]
  │      └─ createBaseModule()             → entity.module.base.ts     [AST清理：3个函数]
  │
  ├─ 6. createEntityControllerSpec()       [若有 controller 时]
  │      └─ 生成 entity.controller.spec.ts                              [AST清理：5个函数]
  │
  └─ 7. mergeMany([...所有模块...])
          ↓
     汇总返回给上游 createResourcesModules()
          ↓
     在 create-server.ts 中对整个 resourcesModules 统一
     replaceModulesCode(formatCode)        ← Prettier 格式化
```

---

## 6. 自定义模块生成完整调用顺序

**位置**：[create-custom-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/custom-module/create-custom-module.ts#L33-L119)

```
createSingleCustomModuleModules(customModule, dtoNameToPath)
  │
  ├─ 1. createServiceModules()             → custommodule.service.ts     [AST清理：5个函数]
  │
  ├─ 2. createCustomModuleControllerModules()   [generateRestApi=true]
  │      └─ createControllerModule()       → custommodule.controller.ts  [AST清理：5个函数]
  │
  ├─ 3. createCustomModuleResolverModules()     [generateGraphQL=true]
  │      └─ createResolverModule()         → custommodule.resolver.ts    [AST清理：全部6个函数]
  │
  ├─ 4. createCustomModule()
  │      └─ createModule()                 → custommodule.module.ts      [AST清理：3个函数]
  │
  └─ 5. mergeMany([...所有模块...])
          ↓
     汇总返回给上游 createCustomModulesModules()
          ↓
     ⚠️ 在 create-server.ts 中 **未被格式化**
     （customModulesModules 未调用 replaceModulesCode(formatCode)）
```

**重要**：自定义模块在 [create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts) 中被生成后**没有经过 Prettier 格式化**——`createCustomModulesModules()`（步骤7）的结果未出现在任何 `replaceModulesCode(formatCode)` 调用中。

---

## 7. Prettier 格式化覆盖范围

### 7.1 Server 端（按模块类别分批格式化）

**位置**：[create-server.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts)

| 执行顺序 | 模块 | 格式化位置 |
|---------|------|-----------|
| 13 | resourcesModules（Service + Controller + gRPC + Resolver + Module + Spec） | L104-L106 |
| 14 | dtoModules | L108 |
| 15 | swagger | L110 |
| 16 | appModule | L112 |
| 17 | seedModule | L114 |
| 18 | authModules | L116 |
| 19 | messageBrokerModules | L118-L120 |
| 20 | packageJsonModule | L122-L124 |
| 21 | typesRelatedFiles | 函数内部自格式化，[create-types-related-files.ts#L29](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-types-related-files/create-types-related-files.ts#L29) |
| 22 | mainFile | 函数内部自格式化，[create-main-file.ts#L69](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-main/create-main-file.ts#L69) |

**Server 端未格式化的模块**：`staticModules`、`customDtos`、`gitIgnore`、**`customModulesModules`**、`secretsManagerModule`、`prismaSchemaModule`、`dotEnvModule`、`connectMicroservicesModule`、`dockerComposeFile`、`dockerComposeDevFile`。

### 7.2 Admin 端（一次性批量格式化）

**位置**：[create-admin.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-admin.ts#L111-L121)

将 `enumRolesModule`、`rolesModule`、`appModuleMap`、`dtoModuleMap`、`entityTitleComponentsModules`、`entityComponentsModules` 合并到临时 `tsModules` 后一次性调用 `replaceModulesCode(formatCode)`。

`typesRelatedFiles` 在其函数内部自格式化（[create-types-related-files.ts#L25](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-types-related-files/create-types-related-files.ts#L25)）。

---

## 8. Plugin 介入点

**位置**：[plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/plugin-wrapper.ts)

```
pluginWrapper(defaultFunc, eventName, eventParams)
  ├─ beforeEventsPipe(...beforePlugins)   // 插件可修改 eventParams（含 template AST）
  ├─ defaultFunc(eventParams)             // 执行默认生成（含 AST 清理、print）
  └─ afterEventsPipe(...afterPlugins)     // 插件可修改返回的 ModuleMap
```

**插件可干预的两个层面**：
1. **Before 钩子**：在 AST 清理之前拿到 `template` AST，可自行修改节点、插入自定义清理逻辑
2. **After 钩子**：在默认生成完成后拿到 `ModuleMap`，可通过 `replaceModulesCode()` 追加自定义格式化或 lint 校验

插件是当前代码中扩展格式化 / Lint 能力的唯一标准路径。

---

## 9. 两条流水线对比

| 维度 | data-service-generator | generator-blueprints |
|------|----------------------|---------------------|
| 代码表示 | AST 模板（`recast`）→ `print()` 输出字符串 | `IAstNode` 树 |
| AST 清理 | 6 个 `remove*` 函数在 `print()` 前调用 | 无（新模板不包含需要清理的声明） |
| 格式化 | `formatCode()` → `prettier.format()`，ModuleMap 层面分批调用 | Writer 在 `IAstNode.toString()` 时直接输出格式化代码，无 Prettier 调用 |
| Lint 校验 | 无 | 无 |

---

## 10. 关键设计观察

1. **AST 清理覆盖度不一致**：Resolver 路径（实体+自定义）调用全部 6 个清理函数，而 Seed 仅调用 1 个、DTO Args 仅调用 1 个——缺少统一的清理入口。

2. **清理调用顺序不统一**：Service 路径将 `removeTSClassDeclares` 放在第一位，Controller/Resolver/gRPC 则放在最后一位；但各路径最终都在 `print()` 之前完成全部清理，顺序差异对最终输出无影响。

3. **自定义模块未被 Prettier 格式化**：Server 端的 `customModulesModules` 不在任何 `replaceModulesCode(formatCode)` 调用列表中，与实体资源模块（在 `resourcesModules` 内格式化）行为不一致。

4. **无真正的 Lint 校验**：代码中不存在任何 `import eslint` / `new ESLint()` / `eslint.lintFiles()` 调用，`removeESLintComments` 仅是移除注释文本。生成代码质量完全依赖模板正确性 + Prettier 格式化。

5. **`addImports` 位置的差异**：部分生成路径在 AST 清理**之前**调用 `addImports`（新增的 import 可能被清理），部分在**之后**（新增的 import 不受清理影响）。当前无一致性约束。
