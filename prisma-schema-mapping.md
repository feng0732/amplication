# Prisma Schema 映射关系梳理

本文档按代码实现脉络梳理 Amplication 平台中**资源字段、关系约束和迁移定义**如何与 Prisma ORM 进行双向转换。

---

## 1. 总体架构与代码脉络

### 1.1 双向转换流水线

Amplication 中存在两条方向相反的转换流水线：

| 方向 | 功能 | 核心模块 |
|------|------|----------|
| **Entity → Prisma** | 从实体定义生成 Prisma schema 文件（代码生成输出） | `packages/data-service-generator/src/server/prisma/` |
| **Prisma → Entity** | 从外部 Prisma schema 导入并解析为 Amplication 实体 | `packages/amplication-server/src/core/prismaSchemaParser/` |

### 1.2 代码模块定位

**正向生成（代码生成）：**

- [create-prisma-schema-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-module.ts) — 对外暴露的模块入口，封装参数调用
- [create-prisma-schema.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema.ts) — schema 组装（datasource/generator/model/enum）
- [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) — 字段级别类型映射与关系字段处理
- [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/constants.ts) — 默认 datasource 与 generator 配置

**反向导入（Schema Import）：**

- [prismaSchemaParser.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/amplication-server/src/core/prismaSchemaParser/prismaSchemaParser.service.ts) — Prisma schema → Entity/Field 的解析与规范化管线
- [dbSchemaImport.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts) — 用户导入流程调度（Kafka 异步处理）

**类型定义：**

- [code-gen-types.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/code-gen-types/src/code-gen-types.ts) — Entity / EntityField / LookupResolvedProperties 等
- [models.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/code-gen-types/src/models.ts) — EnumDataType 枚举值定义（L991-L1011）
- [dto-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/dsg-utils/src/lib/dto-util.ts) — `createEnumName()` 命名规则

---

## 2. 资源字段映射（EnumDataType → Prisma ScalarType）

### 2.1 数据类型枚举

Amplication 内部使用 `EnumDataType` 表示 20 种字段数据类型（定义见 [models.ts#L991-L1011](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/code-gen-types/src/models.ts#L991-L1011)）：

```
Boolean, CreatedAt, DateTime, DecimalNumber, Email, File,
GeographicLocation, Id, Json, Lookup, MultiLineText,
MultiSelectOptionSet, OptionSet, Password, Roles,
SingleLineText, UpdatedAt, Username, WholeNumber
```

### 2.2 标量字段完整映射表

映射实现集中在 [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) 的 `createPrismaSchemaFieldsHandlers` 分发表（L77-L524）。

| Amplication EnumDataType | Prisma ScalarType | 附加属性/默认值 | 说明 |
|--------------------------|-------------------|-----------------|------|
| `SingleLineText` | `String` | — | 短文本 |
| `MultiLineText` | `String` | — | 长文本，底层与短文本一致 |
| `Email` | `String` | — | 邮箱，底层为 String |
| `WholeNumber` | `Int` 或 `BigInt` | 取决于 `properties.databaseFieldType`（默认 INT） | 见 `wholeNumberToPrismaScalarType`（L51-L56） |
| `DecimalNumber` | `Float` 或 `Decimal` | 取决于 `properties.databaseFieldType`（默认 FLOAT） | 见 `decimalNumberToPrismaScalarType`（L58-L63） |
| `DateTime` | `DateTime` | — | 普通时间戳 |
| `Boolean` | `Boolean` | — | — |
| `GeographicLocation` | `String` | — | 地理位置以 String 存储 |
| `Json` | `Json` | — | — |
| `File` | `Json` | — | 文件元数据以 Json 存储 |
| `Roles` | `Json` | `required: true` | 角色字段以 Json 数组存储 |
| `Username` | `String` | `unique: true` | 用户名强制唯一 |
| `Password` | `String` | — | 密码哈希存储 |

### 2.3 特殊系统字段（带自动行为）

| EnumDataType | Prisma 输出 | 关键属性 |
|--------------|------------|----------|
| `Id` | `String`（CUID/UUID）或 `Int`/`BigInt`（自增） | `@id`, `@default(cuid()/uuid()/autoincrement())` |
| `CreatedAt` | `DateTime` | `@default(now())` |
| `UpdatedAt` | `DateTime` | `@updatedAt` |

**Id 子类型映射**（[create-prisma-schema-fields.ts#L33-L49](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L33-L49)）：

| Id.idType | Prisma 标量类型 | 默认函数 |
|-----------|----------------|----------|
| `AUTO_INCREMENT` | `Int` | `autoincrement()` |
| `AUTO_INCREMENT_BIG_INT` | `BigInt` | `autoincrement()` |
| `CUID` | `String` | `cuid()` |
| `UUID` | `String` | `uuid()` |

### 2.4 通用字段属性传递

每个 EntityField 的以下属性直接透传至 Prisma：

- `field.required` → 字段是否可空（`?` 后缀）
- `field.unique` → `@unique` 注解
- `field.customAttributes` → 追加自定义 Prisma 属性字符串

---

## 3. 关系约束映射

### 3.1 关系字段（Lookup）核心处理

Lookup 类型的完整转换逻辑位于 [create-prisma-schema-fields.ts#L280-L366](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L280-L366)。

#### 3.1.1 Lookup 字段属性结构

Lookup 的 `properties` 结构见 [lookup.json](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/code-gen-types/src/schemas/lookup.json)：

```typescript
interface LookupResolvedProperties {
  relatedEntity: Entity;           // 关联目标实体
  relatedField: EntityField;       // 对端反向字段
  allowMultipleSelection: boolean; // 是否允许多选（决定一对多/多对多）
  isOneToOneWithoutForeignKey?: boolean; // 一对一且本方不存 FK
  fkFieldName: string;             // 外键列名（如 customerId）
}
```

#### 3.1.2 三种关系形态与 Prisma 输出

| 关系类型 | 判定条件 | 输出内容 |
|----------|---------|---------|
| **一对多（本方为多端）** | `allowMultipleSelection === false && !isOneToOneWithoutForeignKey` | ① ObjectField（关系导航）+ ② ScalarField（外键列 + `@unique` 当对端也是单值时） |
| **一对多（本方为一端）** | `allowMultipleSelection === true` | 仅 ObjectField，标记 `isList: true`，无外键列 |
| **一对一（本方无 FK）** | `isOneToOneWithoutForeignKey === true` | 仅 ObjectField，无外键列（对端负责存储 FK） |

**一对多本方为多端的典型输出：**

```prisma
customer   Customer @relation(fields: [customerId], references: [id])
customerId BigInt
```

代码在 L331-L365 同时生成**两个字段**：`createObjectField`（关系对象）+ `createScalarField`（外键标量），并通过 `idTypeToPrismaScalarType[idType]` 根据关联实体的 Id 类型选择外键列类型。

### 3.2 关系命名规则（relationName）

当同一对实体之间存在**多条关系路径**时，必须显式指定关系名避免歧义。

判断是否需要命名：`hasAnotherRelation`（[create-prisma-schema-fields.ts#L292-L297](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L292-L297)）—— 只要当前 entity 中存在另一个指向同一 relatedEntity 的 Lookup 字段，就需要命名。

命名算法：`createRelationName()`（[create-prisma-schema-fields.ts#L538-L583](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L538-L583)），优先级递减：

1. 若字段名与对方实体名（单/复数）完全对应 → `"{Remote}On{Local}"`（按字母序拼接）
2. 若某一端字段名全局唯一 → 直接使用该字段名
3. 兜底 → `PascalCase("{Entity} {field} {RelatedEntity} {relatedField}")`（排序后拼接）

### 3.3 枚举字段（OptionSet / MultiSelectOptionSet）

**命名规则**：`createEnumName(field, entity)` = `"Enum" + PascalCase(entity.name) + PascalCase(field.name)`，见 [dto-util.ts#L152-L154](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/dsg-utils/src/lib/dto-util.ts#L152-L154)。

| 类型 | Prisma 输出 |
|------|------------|
| `OptionSet` | `Enum{Entity}{Field}` 对象字段（单值） |
| `MultiSelectOptionSet` | `Enum{Entity}{Field}[]` 对象字段（数组 + 列表） |

枚举定义生成：在 `createPrismaSchemaInternal` 中通过 `getEnumFields(entity)` 汇总，调用 [createPrismaEnum()](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema.ts#L76-L85) 产出 Prisma `enum` 块。

---

## 4. Schema 组装与 ORM 输出

### 4.1 组装流程（Entity → Prisma 文件）

完整流程见 [createPrismaSchemaInternal()](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema.ts#L30-L74)：

```
1. 计算 fieldNamesCount（统计字段名出现频次，用于关系命名决策）
2. entities.map → createPrismaModel()
     └─ entity.fields.flatMap → createFieldsHandlers[field.dataType]()
           └─ 返回 ScalarField[] / ObjectField[]
3. entities.flatMap → getEnumFields() → createPrismaEnum()
4. 拼装 datasource（默认 postgresql + env("DB_URL")）
5. 拼装 generator（默认 prisma-client-js）
6. PrismaSchemaDSL.createSchema() + print() 输出 schema.prisma 文本
7. 写入 ModuleMap，路径 = "{serverDirectories.baseDirectory}/prisma/schema.prisma"
```

### 4.2 默认 Datasource 与 Generator

见 [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/constants.ts)：

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DB_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

### 4.3 生成产物示例

从测试快照 [create-data-service.spec.ts.snap#L3134-L3214](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/tests/__snapshots__/create-data-service.spec.ts.snap#L3134-L3214) 可看到完整输出形态：

```prisma
model User {
  id            String    @id @default(cuid())
  username      String    @unique
  roles         Json
  password      String
  manager       User?     @relation(name: "employees", fields: [managerId], references: [id])
  managerId     String?
  employees     User[]    @relation(name: "employees")
  interests     EnumUserInterests[]
  priority      EnumUserPriority
  profile       Profile?  @relation(fields: [profileId], references: [id])
  profileId     Int?      @unique
}

model Profile {
  id        Int      @id @default(autoincrement())
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  user      User?
}
```

---

## 5. 迁移定义生成与执行

### 5.1 生成项目中的迁移脚本

代码生成器在生成的服务项目 `package.json` 中注入以下 Prisma 迁移脚本（见 [package.json](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/package-json/package.json)）：

| Script | 命令 | 用途 |
|--------|------|------|
| `db:migrate-save` | `prisma migrate dev` | 开发环境：比较 schema 与数据库差异，生成并应用新迁移 |
| `db:migrate-up` | `prisma migrate deploy` | 生产环境：按序执行所有 pending 迁移 |
| `db:clean` | `prisma migrate reset` | 重置数据库（清空 + 重跑全部迁移） |
| `db:init` | `run-s "db:migrate-save -- --name 'initial version'" db:migrate-up seed` | 首次初始化链路 |
| `prisma:generate` | `prisma generate` | 根据 schema 重新生成 Prisma Client |

迁移**不直接生成 SQL 文件**，而是依赖 Prisma 官方 CLI 在部署/开发时基于 schema 与 `prisma/migrations/` 目录自动计算。

### 5.2 Prisma Schema 反向导入（Import 流程）

当用户上传已有 Prisma schema 时，平台通过 `PrismaSchemaParserService` 执行反向解析（[prismaSchemaParser.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/amplication-server/src/core/prismaSchemaParser/prismaSchemaParser.service.ts)）。

#### 5.2.1 七步准备管线（prepareOperations）

见 `prepareOperations` 数组（L108-L116），按顺序执行：

| 步骤 | 函数 | 职责 |
|------|------|------|
| 1 | `prepareModelNames` | 格式化 model 名（PascalCase）、处理冲突、必要时加 `@@map` |
| 2 | `prepareFieldNames` | 格式化 field 名（camelCase）、处理冲突、必要时加 `@map` |
| 3 | `prepareFieldTypes` | 将字段类型引用同步为重命名后的 model 名 |
| 4 | `prepareModelIdAttribute` | 处理 `@@id` 复合主键 → 降级为 `@@unique`（不支持复合主键） |
| 5 | `prepareIdField` | 规范化 id 字段：无 id 则自动补齐、非 id 字段叫 "id" 则改名、id 字段不叫 "id" 则改名 |
| 6 | `prepareModelCompositeTypeAttributes` | 同步 `@@unique` / `@@index` 中的字段名为重命名后的值 |
| 7 | `prepareRelationReferenceFields` | 同步 relation 中 `references: [...]` 的目标字段名 |

#### 5.2.2 字段判定顺序（convertPreparedSchemaForImportObjects）

L260-L417 中对每个 field 按以下优先级判断并调用对应 `convertPrismaXxxToEntityField`：

```
Id → Boolean → CreatedAt → UpdatedAt → DateTime
  → DecimalNumber → WholeNumber → SingleLineText
  → Json → OptionSet → MultiSelectOptionSet
  → Lookup (含多对多特殊处理)
```

以下字段被跳过（不生成独立 EntityField）：
- 外键标量列（`isFkFieldOfARelation`）—— 由 Lookup 字段统一表示
- 未注解的反向导航（`isNotAnnotatedRelationField` 且非多对多）

#### 5.2.3 多对多关系判定

`isManyToManyRelation()`（L1137-L1238）逻辑：当前字段是 `type[]` 数组 + 对端也存在指向本方 model 的数组类型字段 → 判定为多对多，双方仅各保留一个 Lookup 字段（`allowMultipleSelection: true`）。

---

## 6. 关键类型速查表

### 6.1 EntityField 结构

```typescript
// code-gen-types.ts
interface EntityField {
  id: string;
  permanentId: string;          // 跨版本稳定 ID
  name: string;                 // 字段代码名（camelCase）
  displayName: string;          // UI 显示名
  dataType: EnumDataType;       // 20 种数据类型之一
  properties: JsonValue;        // 各 dataType 专属配置（见 schemas/*.json）
  required: boolean;
  unique: boolean;
  searchable: boolean;
  customAttributes?: string;    // 透传至 Prisma 的自定义注解
  description: string;
}
```

### 6.2 PrismaSchemaDSL 依赖

代码生成使用第三方库 `prisma-schema-dsl` 构建 AST，核心工厂函数：

- `createScalarField(name, type, isList, isRequired, isUnique, isId, isUpdatedAt, defaultValue, ...)`
- `createObjectField(name, type, isList, isRequired, relationName, fields, references, ...)`
- `createModel(name, fields, ..., customAttributes)`
- `createEnum(name, values)`
- `createSchema(models, enums, dataSource, generators)`
- `print(schema)` → 最终 `.prisma` 文本

---

## 7. 文件索引

| 关注点 | 文件路径 |
|--------|---------|
| Prisma schema 生成入口 | [create-prisma-schema-module.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-module.ts) |
| Model/Enum 组装 | [create-prisma-schema.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema.ts) |
| 字段类型映射与关系 | [create-prisma-schema-fields.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) |
| Datasource/Generator 默认值 | [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/constants.ts) |
| 枚举命名规则 | [dto-util.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/dsg-utils/src/lib/dto-util.ts) |
| EnumDataType 定义 | [models.ts#L991-L1011](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/code-gen-types/src/models.ts#L991-L1011) |
| Lookup 属性 JSON Schema | [lookup.json](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/libs/util/code-gen-types/src/schemas/lookup.json) |
| 生成服务的迁移脚本 | [package.json](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/package-json/package.json) |
| Prisma Schema 反向解析管线 | [prismaSchemaParser.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/amplication-server/src/core/prismaSchemaParser/prismaSchemaParser.service.ts) |
| Schema Import 调度 | [dbSchemaImport.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts) |
| 主平台参考 schema | [schema.prisma](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/amplication-prisma-db/prisma/schema.prisma) |
| 单元测试（字段映射验证） | [create-prisma-schema.spec.ts](file:///d:/fz/0601/solo-dogfeeding/code/43-amplication/packages/data-service-generator/src/server/prisma/create-prisma-schema.spec.ts) |
