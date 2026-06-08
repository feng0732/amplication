# Amplication 代码生成中的 Formatter 与 Lint 集成分析

## 1. 概览

Amplication 的代码生成流程中存在**两个独立层级**的代码规整操作：

1. **AST 层面的 Lint/TypeScript 清理** — 发生在单文件生成过程中，在 AST 插值完成后、`print()` 输出字符串代码之前执行。清理函数来自 `@amplication/code-gen-utils`，作用是移除模板文件中的开发辅助内容（`eslint-disable`、`@ts-ignore`、`declare var/class/interface`），**并非实际运行 ESLint 做静态分析**。

2. **Prettier 格式化** — 发生在模块集合（`ModuleMap`）层面，一批文件生成完毕后通过 `replaceModulesCode(formatCode)` 统一调用 `prettier.format()` 进行格式化。

本文件重点分析：**哪些生成路径不调用 ESLint / TypeScript AST 清理函数**，以及用户特别关注的边界模块——微服务连接、主文件、数据传输对象（DTO）、枚举、消息代理、密钥管理——的完整 `print` 输出路径。

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

### 2.2 formatCode — Prettier 格式化函数

**位置**：[files.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/libs/util/code-gen-utils/src/lib/files.ts#L7-L24)

根据文件后缀分派 parser：`.ts/.tsx` → `typescript`；`.json` → `json`；`.yml/.yaml` → `yaml`；`.md` → `markdown`；`.graphql` → `graphql`；其他后缀原样返回。

---

## 3. 用户关注的边界模块详细分析

以下逐个分析用户指定的六大边界模块，列出其 `print` 输出路径、AST 清理覆盖情况、以及 Prettier 格式化情况。

---

### 3.1 微服务连接（connectMicroservices）

**生成入口**：[connect-microservices.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/connect-microservices/connect-microservices.ts)

#### 处理链与 print 输出路径

```typescript
// connectMicroservices() [L21-L29]
//   → readFile(connect-microservices.template.ts)  [L22]
//   → pluginWrapper(connectMicroservicesInternal, ...)
//
// connectMicroservicesInternal() [L31-L48]
//   → addImports(template, importContainedIdentifiers(template, IMPORTABLE_IDS))  [L36-L38]
//   → print(template).code  ← 代码输出点 [L42]
//   → 写入 moduleMap，返回
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（仅调用 `addImports`） |
| **Prettier 格式化** | ❌ 未格式化。`connectMicroservicesModule` 在 [create-server.ts#L165](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L165) 被加入最终 moduleMap，但不存在于任何 `replaceModulesCode(formatCode)` 调用中 |
| **输出文件** | `src/connectMicroservices.ts` |

**代码生成器**：使用 `recast` 的 `print()` 将 AST 模板直接输出，无 AST 清理介入。

---

### 3.2 主文件（main.ts）

**生成入口**：[create-main-file.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-main/create-main-file.ts)

#### 处理链与 print 输出路径

```typescript
// createMainFile() [L20-L27]
//   → readFile(main.template.ts)  [L22]
//   → pluginWrapper(createMainFileInternal, ...)
//
// createMainFileInternal() [L29-L72]
//   → (可选) hasBigIntFields: 解析 MAIN_TS_WITH_BIGINT_FILE_NAME，注入到 main 函数体 [L35-L56]
//   → print(template).code  ← 代码输出点 [L62]
//   → moduleMap.replaceModulesPath(...)  [L68]
//   → moduleMap.replaceModulesCode((path, code) => formatCode(path, code))  ← 格式化 [L69]
//   → 返回
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（仅可能插入 BigInt AST 节点） |
| **Prettier 格式化** | ✅ 在函数内部自格式化（L69） |
| **输出文件** | `src/main.ts` |

**说明**：主文件模板本身不含需要清理的 `declare` 声明和 ESLint 注释，因此未调用清理函数。

---

### 3.3 数据传输对象（DTOs）

DTO 生成分布在 **Server 端资源 DTO**、**Server 端自定义 DTO**、**Server 端 GraphQL Args** 和 **Admin 端 DTO** 四条路径。

#### 3.3.1 Server 端资源 DTO（Entity DTOs）

**生成入口**：[create-dtos.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/create-dtos.ts)

**print 输出点**：[create-dto-module.ts#L128](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/dto/create-dto-module.ts#L128)（普通 DTO）和 [create-enum-dto-module.ts#L36](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/dto/create-enum-dto-module.ts#L36)（枚举 DTO）

```typescript
// createDTOModules() → createDTOModulesInternal() [L50-L70]
//   → 遍历所有 DTO
//      ├─ Class 类型: createDTOModule(dto, dtoNameToPath)
//      │    → createDTOFile(dto, path, dtoNameToPath)
//      │         → builders.file(builders.program(statements))
//      │         → addImports(file, ...)  ← 注入依赖 import
//      │    → addAutoGenerationComment(file)
//      │    → print(file).code  ← 代码输出点
//      │
//      └─ Enum 类型: createEnumDTOModule(dto, dtoNameToPath)
//           → createDTOFile(dto, path, dtoNameToPath)  (同上)
//           → 注入 registerEnumType() 调用 [create-enum-dto-module.ts#L25-L28]
//           → addImports(file, [from "@nestjs/graphql"])
//           → addAutoGenerationComment(file)
//           → print(file).code  ← 代码输出点
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（Class 和 Enum DTO 均从头构建 AST，不从 .template.ts 文件读取） |
| **Prettier 格式化** | ✅ 在 [create-server.ts#L108](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L108) 通过 `dtoModules.replaceModulesCode(formatCode)` 格式化 |
| **输出文件** | `src/<entity>/base/<DtoName>.ts`、`src/<entity>/base/<EnumName>.ts` 等 |

#### 3.3.2 Server 端 GraphQL Args

**生成入口**：7 个文件，如 [create-create-args.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/dto/graphql/create/create-create-args.ts#L9-L30)

```typescript
// createCreateArgs(entity, createInput)
//   → readFile(templatePath)
//   → interpolate(file, mapping)
//   → removeTSClassDeclares(file)  ← 唯一调用的清理函数
//   → (不直接 print) 返回 classDeclaration，嵌入上层 DTO
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ✅ 仅调用 `removeTSClassDeclares`（1 个） |
| **Prettier 格式化** | ✅ 属于 `dtoModules`，在 [create-server.ts#L108](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L108) 格式化 |
| **输出文件** | `src/<entity>/dto/graphql/<operation>/<ArgsName>.ts`（作为上层 DTO 的一部分输出） |

#### 3.3.3 Server 端自定义 DTO（Custom DTOs）

**生成入口**：[create-custom-dtos.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/dto/custom-types/create-custom-dtos.ts)

```typescript
// createCustomDtos() [L40-L95]
//   → 遍历 moduleActionsAndDtoMap
//      → 对每个 Custom 或 CustomEnum DTO
//         ├─ Custom: createDto(dto) → classDeclaration（从头构建）
//         └─ CustomEnum: createEnumDTO(dto) → tsEnumDeclaration（从头构建）
//      → createDTOModule() / createEnumDTOModule()
//           → createDTOFile() → addImports() → print(file).code  ← 代码输出点
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（从头构建 AST，不使用模板） |
| **Prettier 格式化** | ❌ 未格式化。`customDtos` 在 [create-server.ts#L73](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L73) 生成，但未出现在任何 `replaceModulesCode(formatCode)` 调用中 |
| **输出文件** | `src/<customModuleName>/<DtoName>.ts` |

#### 3.3.4 Admin 端 DTOs

**生成入口**：[admin/create-dto-modules.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-dto-modules.ts)

```typescript
// createDTOModules(dtos, dtoNameToPath) [L12-L33]
//   → 遍历所有 Server DTO
//      → transformServerDTOToClientDTO()  ← Class 转 TSTypeAlias
//      → createDTOFile(dto, modulePath, dtoNameToPath)
//      → print(file).code  ← 代码输出点 [L28]
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（从已构建的 Server DTO AST 转换而来，不重新读取模板） |
| **Prettier 格式化** | ✅ 在 [create-admin.ts#L121](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-admin.ts#L121) 通过 `tsModules.replaceModulesCode(formatCode)` 格式化 |
| **输出文件** | Admin UI 项目 `src/**/*DTO*.ts` |

---

### 3.4 枚举（Enums）

枚举生成分布在 5 条路径。

#### 3.4.1 资源实体字段枚举（嵌入 DTO）

**生成入口**：[create-enum-dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/dto/create-enum-dto.ts#L10-L20)

```typescript
createEnumDTO(field, entity)
  → builders.tsEnumDeclaration(...)  ← 从头构建
  → (不直接 print) 返回 TSEnumDeclaration，嵌入上层 createEnumDTOModule → print
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ 一个都没有调用 |
| **Prettier 格式化** | ✅ 属于 `dtoModules`（见 3.3.1） |

#### 3.4.2 枚举 DTO 模块（独立枚举文件）

**生成入口**：[create-enum-dto-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/resource/dto/create-enum-dto-module.ts)

print 输出点：L36（见 3.3.1）。

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ 一个都没有调用 |
| **Prettier 格式化** | ✅ 属于 `dtoModules` |

#### 3.4.3 消息代理 Topics 枚举

**生成入口**：[topics-enum/createTopicsEnum.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/message-broker/topics-enum/createTopicsEnum.ts#L27-L63)

```typescript
createTopicsEnumInternal(eventParams)
  → builders.file(builders.program([]))  ← 全新空文件
  → EnumBuilder(pascalCase(MBName) + "Topics")
  → serviceTopic.patterns.forEach → astEnum.createMember(...)
  → builders.exportDeclaration(false, astEnum.ast)
  → astFile.program.body.push(...topics)
  → print(astFile).code  ← 代码输出点 [L58]
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（从头构建，完全不使用模板文件） |
| **Prettier 格式化** | ✅ 属于 `messageBrokerModules`，在 [create-server.ts#L118-L120](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L118-L120) 格式化 |
| **输出文件** | `src/message-broker/topics.ts` |

#### 3.4.4 Admin 端角色枚举（EnumRoles）

**生成入口**：[admin/create-enum-roles.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-enum-roles.ts#L10-L19)

```typescript
createEnumRolesModule(roles)
  → createRolesEnumDeclaration(roles)
       → builders.tsEnumDeclaration(ENUM_ROLES_ID, [...tsEnumMember])  ← 从头构建
  → createDTOFile(enumDeclaration, MODULE_PATH, {})
  → print(file).code  ← 代码输出点 [L17]
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（从头构建） |
| **Prettier 格式化** | ✅ 在 [create-admin.ts#L112-L121](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-admin.ts#L112-L121) 通过 `tsModules.replaceModulesCode(formatCode)` 格式化 |
| **输出文件** | `src/user/EnumRoles.ts` |

#### 3.4.5 密钥管理 SecretsNameKey 枚举

**生成入口**：[secrets-manager/create-secrets-manager.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/secrets-manager/create-secrets-manager.ts#L28-L68)

```typescript
createSecretsManagerInternal({ secretsNameKey })
  // 枚举部分：
  → createTSEnumSecretsNameKey(secretsNameKey)
       → builders.tsEnumDeclaration(
            builders.identifier("EnumSecretsNameKey"),
            secretsNameKey.map(...)
         )  ← 从头构建
  → createDTOFile(enumDeclaration, ENUM_MODULE_PATH, {})
  → print(enumFile).code  ← 代码输出点 [L51]

  // 静态文件部分：
  → staticFilesPath.forEach: fsPromises.readFile(modulePath, encoding)
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（从头构建枚举 AST + 直接读取静态文件） |
| **Prettier 格式化** | ❌ 未格式化。`secretsManagerModule` 在 [create-server.ts#L88](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L88) 生成，但不在任何 `replaceModulesCode(formatCode)` 调用中 |
| **输出文件** | `src/providers/secrets/secretsNameKey.enum.ts`（枚举） + `src/providers/secrets/**/*`（静态文件） |

---

### 3.5 消息代理（Message Broker）

**生成入口**：[message-broker/create-service-message-broker-modules.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/message-broker/create-service-message-broker-modules.ts#L23-L40)

消息代理包含 4 个子模块：

| 子模块 | 生成函数 | 处理方式 | AST 清理 | print 输出 |
|--------|---------|---------|---------|-----------|
| Client Options Factory | [createMessageBrokerClientOptions()](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/message-broker/generate-message-broker-client-options/generate-message-broker-client-options.ts#L9-L17) | 返回空 `ModuleMap` | — | — |
| NestJS Module | [createMessageBrokerModule()](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/message-broker/message-broker-module/create-message-broker-module.ts#L9-L17) | 返回空 `ModuleMap` | — | — |
| Service + Service Base | [createMessageBrokerServiceModules()](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/message-broker/message-broker-service/create-message-broker-service.ts#L5-L20) | 返回两个空 `ModuleMap` 合并结果 | — | — |
| Topics Enum | [createTopicsEnum()](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/message-broker/topics-enum/createTopicsEnum.ts) | 从头构建 AST → `print(astFile).code`（L58） | ❌ 一个都没有调用 | ✅ `src/message-broker/topics.ts` |

#### 汇总

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（3 个空壳模块 + 1 个从头构建枚举） |
| **Prettier 格式化** | ✅ 整个 `messageBrokerModules` 在 [create-server.ts#L118-L120](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L118-L120) 被格式化（但 3 个空模块格式化无意义） |
| **实际输出文件** | `src/message-broker/topics.ts`（其余三个子模块由插件填充） |

---

### 3.6 密钥管理（Secrets Manager）

**生成入口**：[secrets-manager/create-secrets-manager.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/secrets-manager/create-secrets-manager.ts)

#### 处理链与 print 输出路径

```typescript
createSecretsManagerInternal({ secretsNameKey }) [L28-L68]
  // ① 枚举文件：
  → createTSEnumSecretsNameKey(secretsNameKey)
       → builders.tsEnumDeclaration("EnumSecretsNameKey", [...members])
  → createDTOFile(enumDeclaration, "secretsNameKey.enum.ts", {})
  → print(enumFile).code  ← 代码输出点 ① [L51]
  → moduleMap.set({ path: ENUM_MODULE_PATH, code })

  // ② 静态文件：
  → fg(secretManagerStaticFilesDirectory + "/**/*")  ← fast-glob 扫描
  → staticFilesPath.forEach:
       fsPromises.readFile(modulePath, encoding)  ← 直接读取模板文件内容
       moduleMap.set({ path, code })  ← 代码输出点 ②（无 print 调用）
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | ❌ **一个都没有调用**（枚举从头构建 AST；静态文件直接读取磁盘内容） |
| **Prettier 格式化** | ❌ 未格式化。`secretsManagerModule` 不在任何 `replaceModulesCode(formatCode)` 调用中 |
| **输出文件** | `src/providers/secrets/secretsNameKey.enum.ts` + `src/providers/secrets/**/*`（从 static/ 目录复制的所有模板文件） |

---

## 4. 其他不调用 AST 清理函数的生成路径

除用户关注的六大边界模块外，以下路径也不调用任何 AST 清理函数：

### 4.1 静态模块（Static Modules）

**生成入口**：[read-static-modules.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/utils/read-static-modules.ts)

```typescript
readStaticModulesInner({ source, basePath })
  → fg(source + "/**/*")  ← 扫描目录
  → 过滤 .DS_Store 等
  → 每个文件: fs.promises.readFile(module, encoding)  ← 直接读磁盘
  → moduleMap.set({ path, code })
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — 不使用 AST，纯文件拷贝 |
| **Prettier 格式化** | ❌ 未格式化。Server 端和 Admin 端的 `staticModules` 均不在任何 `replaceModulesCode(formatCode)` 调用中 |
| **输出文件** | Server `src/**/*`（static/ 目录模板）、Admin `**/*`（admin/static/ 目录模板） |

### 4.2 Prisma Schema

**生成入口**：[prisma/create-prisma-schema.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema.ts#L30-L74)

```typescript
createPrismaSchemaInternal({ entities, ... })
  → entities.map(createPrismaModel)
  → entities.flatMap(createPrismaEnum)
  → PrismaSchemaDSL.createSchema(models, enums, dataSource, [generator])
  → PrismaSchemaDSL.print(schema)  ← 代码输出点（非 recast print）
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — 不使用 recast AST，使用专用 `prisma-schema-dsl` 库 |
| **Prettier 格式化** | ❌ 未格式化。`.prisma` 后缀不在 `formatCode` 的支持列表中，且 `prismaSchemaModule` 也不在任何 `replaceModulesCode(formatCode)` 调用中 |
| **输出文件** | `prisma/schema.prisma` |

### 4.3 .env 文件

**Server 端入口**：[create-dotenv.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-dotenv.ts#L28-L48)

```typescript
createDotEnvModuleInternal({ envVariables })
  → sortAlphabetically(removeDuplicateKeys(envVariables))
  → convertToKeyValueSting(...)  ← 字符串拼接
  → replacePlaceholdersInCode(codeWithEnvVariables, appInfo.settings)  ← ${name} 占位符替换
```

**Admin 端入口**：[admin/create-dotenv.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-dotenv.ts#L33-L60)

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — 纯文本处理，不使用 AST |
| **Prettier 格式化** | ❌ 未格式化。`.env` 后缀不在 `formatCode` 支持列表中 |
| **输出文件** | `/.env`（Server 和 Admin 各一份） |

### 4.4 Docker Compose 文件

**入口**：[create-docker-compose.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/docker-compose/create-docker-compose.ts#L31-L51) 和 [create-docker-compose-dev.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/docker-compose/create-docker-compose-dev.ts#L42-L61)

```typescript
createDockerComposeFileInternal(eventParams)
  → fs.readFile(templatePath, "utf-8")
  → prepareYamlFile(fileContent, updateProperties)  ← YAML 操作
  → moduleMap.set({ path, code: preparedFile })
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — YAML 处理，不使用 TypeScript AST |
| **Prettier 格式化** | ❌ 未格式化。虽然 `formatCode` 支持 YAML parser，但 `dockerComposeFile` 和 `dockerComposeDevFile` 不在任何 `replaceModulesCode(formatCode)` 调用中 |
| **输出文件** | `/docker-compose.yml`、`/docker-compose.dev.yml` |

### 4.5 .gitignore

**Server 端入口**：[gitignore/create-gitignore.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/gitignore/create-gitignore.ts#L27-L42)

```typescript
createGitIgnoreInternal({ gitignorePaths })
  → formatGitignorePaths(gitignorePaths)  ← 字符串格式化
  → moduleMap.set({ path: ".gitignore", code })
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — 纯文本处理 |
| **Prettier 格式化** | ❌ 未格式化（`.gitignore` 不是 Prettier 支持的类型） |
| **输出文件** | `/.gitignore` |

### 4.6 Types Related Files（BigInt / Decimal Filter 等）

**Server 端入口**：[create-types-related-files.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-types-related-files/create-types-related-files.ts#L14-L32)

**Admin 端入口**：[admin/create-types-related-files.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-types-related-files/create-types-related-files.ts#L13-L28)

```typescript
createTypesRelatedFiles()
  → createGraphQLBigInt() / createBigIntFilters() / createDecimalFilters()
       → fs.readFile(filePath, "utf-8")  ← 直接读取模板文件
       → moduleMap.set({ path, code: fileContent })
  → moduleMap.replaceModulesCode(formatCode)  ← 函数内部自格式化
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — 直接读文件，无 AST 插值操作 |
| **Prettier 格式化** | ✅ 在函数内部自格式化 |
| **输出文件** | Server：`src/util/GraphQLBigInt.ts`、`src/util/*Filter.ts`；Admin：`src/util/*Filter.ts` |

### 4.7 Auth Modules

**入口**：[auth/create-auth.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/auth/create-auth.ts#L14-L22)

```typescript
createAuthModules()
  → pluginWrapper(() => new ModuleMap(DsgContext.getInstance.logger), ...)
  ← 默认返回空 ModuleMap，插件可填充
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — 默认不生成文件 |
| **Prettier 格式化** | ✅ 空模块在 [create-server.ts#L116](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L116) 格式化（无实际效果） |

### 4.8 Server package.json

**入口**：[package-json/create-package-json.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/package-json/create-package-json.ts#L38-L60)

```typescript
createServerPackageJsonInternal({ updateProperties })
  → fs.readFile(package.json.template)
  → updatePackageJSONs(moduleMap, ...)  ← JSON 合并
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — JSON 处理，不使用 AST |
| **Prettier 格式化** | ✅ 在 [create-server.ts#L122-L124](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L122-L124) 格式化 |
| **输出文件** | `/package.json` |

### 4.9 Admin 端 Roles 模块

**入口**：[admin/create-roles-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-roles-module.ts#L3-L12)

```typescript
createRolesModule(roles, srcDirectory)
  → {
      path: `${srcDirectory}/user/roles.ts`,
      code: `export const ROLES = ${JSON.stringify(roles, null, 2)}`
    }
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — 纯字符串拼接 |
| **Prettier 格式化** | ✅ 属于 `tsModules`，在 [create-admin.ts#L121](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/create-admin.ts#L121) 格式化 |
| **输出文件** | `src/user/roles.ts` |

### 4.10 Admin 端 Public Files

**入口**：[admin/public-files/create-public-files.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/admin/public-files/create-public-files.ts#L9-L44)

```typescript
createPublicFiles()
  → createIndexHTMLModule(): readCode() + .replace()  ← 字符串替换
  → createManifestModule(): JSON.stringify()  ← 字符串生成
```

| 项目 | 状态 |
|------|------|
| **AST 清理函数** | — HTML/JSON 处理，不使用 TypeScript AST |
| **Prettier 格式化** | ❌ 未格式化。`publicFilesModules` 不在 `tsModules` 中 |
| **输出文件** | `/index.html`、`/public/manifest.json` |

### 4.11 Server 端自定义模块（Custom Module）整体

虽然自定义模块的 Controller、Service、Resolver 内部调用了 AST 清理函数（5~6 个），但整个 `customModulesModules` 集合在 [create-server.ts#L73](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-server.ts#L73) 生成后**未经过 Prettier 格式化**——它不在任何 `replaceModulesCode(formatCode)` 调用中。

---

## 5. 全局汇总：不调用 AST 清理函数的生成路径

### 5.1 完整分类表

| 分类 | 生成路径 | 代码输出方式 | 清理函数 | Prettier 格式化 |
|------|---------|------------|---------|----------------|
| **用户指定的边界模块** | | | | |
| 微服务连接 | connectMicroservices | `print(template).code`（模板） | ❌ 0 个 | ❌ 未格式化 |
| 主文件 | createMainFile | `print(template).code`（模板，可选 BigInt 注入） | ❌ 0 个 | ✅ 自格式化 |
| 数据传输对象-资源 DTO | createDTOModules | `print(file).code`（从头构建 AST） | ❌ 0 个 | ✅ dtoModules |
| 数据传输对象-自定义 DTO | createCustomDtos | `print(file).code`（从头构建 AST） | ❌ 0 个 | ❌ 未格式化 |
| 数据传输对象-Admin DTO | admin/createDTOModules | `print(file).code`（Server DTO 转换） | ❌ 0 个 | ✅ tsModules |
| 枚举-资源字段枚举 | createEnumDTO（嵌入 DTO） | 嵌入 DTO 的 print 输出 | ❌ 0 个 | ✅ dtoModules |
| 枚举-MB Topics | createTopicsEnum | `print(astFile).code`（从头构建） | ❌ 0 个 | ✅ messageBrokerModules |
| 枚举-Admin EnumRoles | createEnumRolesModule | `print(file).code`（从头构建） | ❌ 0 个 | ✅ tsModules |
| 枚举-SecretsNameKey | createSecretsManager 枚举部分 | `print(enumFile).code`（从头构建） | ❌ 0 个 | ❌ 未格式化 |
| 消息代理-ClientOptions/Module/Service | createMessageBroker 3 子模块 | 返回空 ModuleMap | — | ✅ messageBrokerModules |
| 消息代理-Topics Enum | createTopicsEnum（见上） | 从头构建 print | ❌ 0 个 | ✅ messageBrokerModules |
| 密钥管理-枚举 | createSecretsManager 枚举（见上） | 从头构建 print | ❌ 0 个 | ❌ 未格式化 |
| 密钥管理-静态文件 | createSecretsManager 静态文件 | `fsPromises.readFile()` | — | ❌ 未格式化 |
| **其他不清理的路径** | | | | |
| 静态模块 | readStaticModules | `fs.promises.readFile()` | — | ❌ 未格式化 |
| Prisma Schema | createPrismaSchemaModule | `PrismaSchemaDSL.print()` | — | ❌ 未格式化 |
| .env | Server/Admin createDotEnvModule | 字符串拼接 + 占位符替换 | — | ❌ 未格式化 |
| Docker Compose | createDockerComposeFile/Dev | `prepareYamlFile()` | — | ❌ 未格式化 |
| .gitignore | Server/Admin createGitIgnore | `formatGitignorePaths()` | — | ❌ 未格式化 |
| Types Related Files | Server/Admin createTypesRelatedFiles | `fs.readFile()` | — | ✅ 自格式化 |
| Auth Modules | createAuthModules | 空 ModuleMap | — | ✅ 空格式化 |
| package.json | Server/Admin | `fs.readFile()` + JSON 合并 | — | ✅ packageJsonModule |
| Admin Roles | createRolesModule | `JSON.stringify()` | — | ✅ tsModules |
| Admin Public Files | createPublicFiles | `readCode()` + 字符串替换 / `JSON.stringify()` | — | ❌ 未格式化 |
| **部分清理的路径** | | | | |
| GraphQL Args | create*Args（7 种） | 嵌入 DTO print | ✅ 仅 `removeTSClassDeclares` | ✅ dtoModules |
| Admin App.tsx | createAppModule | `print(template).code` | ✅ 2 个 (`removeTSVariableDeclares`, `removeTSIgnoreComments`) | ✅ tsModules |
| Admin Entity Components | createEntityComponentModules | `print(file).code` | ✅ 3 个（无 `removeESLintComments`） | ✅ tsModules |
| Seed | createSeed | `print(template).code` | ✅ 1 个 (`removeTSVariableDeclares`) | ✅ seedModule |
| Swagger | createSwagger | `print(template).code` | ✅ 2 个 (`removeTSVariableDeclares`, `removeTSIgnoreComments`) | ✅ swagger |

### 5.2 代码输出方式分类

从 print 输出方式可以看出"不调用清理函数"的根本原因分为三类：

1. **从头构建 AST（`builders.*` / `EnumBuilder`）** — 不读取 `.template.ts` 模板文件，因此 AST 中本就不存在需要清理的 `declare` 声明和 ESLint 注释。
   - 路径：资源 DTO、自定义 DTO、Admin DTO、所有枚举（Topics/EnumRoles/SecretsNameKey）、消息代理 Topics

2. **直接读取磁盘文件（`fs.readFile` / `readCode` / `readStaticModules`）** — 跳过 recast AST 层，直接将字符串内容写入 ModuleMap。
   - 路径：静态模块、Types Related Files、Docker Compose、密钥管理静态文件、.gitignore、package.json、Admin Public Files

3. **不使用 TypeScript AST（专用库/字符串处理）** — 例如 Prisma Schema DSL、JSON.stringify、YAML prepare、`.env` 键值拼接等。
   - 路径：Prisma Schema、.env、Admin Roles、Admin manifest.json

---

## 6. 关键设计观察

1. **"从头构建 AST"的路径天然不需要清理**：资源 DTO、所有枚举、Admin DTO 等路径不使用 `.template.ts` 模板文件，而是用 `builders.tsEnumDeclaration`、`builders.classDeclaration`、`builders.tsTypeAliasDeclaration` 等从头构建 AST——这些节点中本就不会有 `declare var/class/interface` 或 `eslint-disable` 注释，因此无需调用清理函数。

2. **主文件和微服务连接是例外**：它们**使用了 `.template.ts` 模板**（[main.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/create-main/main.template.ts) 和 [connect-microservices.template.ts](file:///d:/fz/0601/solo-dogfeeding/code/105-amplication/packages/data-service-generator/src/server/connect-microservices/connect-microservices.template.ts)），但模板文件本身不包含需要清理的声明和注释，因此开发者未调用清理函数。这是一个**潜在的不一致点**——如果未来模板中加入了需要清理的内容（如 `declare var`），生成代码会直接暴露这些声明。

3. **Prettier 格式化覆盖的盲区**：`connectMicroservicesModule`、`customDtos`、`customModulesModules`、`secretsManagerModule`、`staticModules`（Server+Admin）、`prismaSchemaModule`、`dotEnvModule`（Server+Admin）、`connectMicroservicesModule`、`dockerComposeFile`、`dockerComposeDevFile`、`gitIgnore`、Admin `publicFilesModules` 均未被 Prettier 格式化。其中 `connectMicroservicesModule`（TS 文件）和 `customDtos`/`customModulesModules`（TS 文件）属于 Prettier 支持的类型但遗漏了格式化调用。

4. **GraphQL Args 是 DTO 家族中唯一调用清理的路径**：7 种 GraphQL Args 文件读取了 `.template.ts`（含 `declare class`），因此调用了 `removeTSClassDeclares`。而普通 Class DTO、Enum DTO、Admin DTO 均从头构建 AST，无需清理。
