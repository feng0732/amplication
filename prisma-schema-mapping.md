# Prisma Schema 映射关系梳理

本文档按代码实现脉络梳理 Amplication 平台中**资源字段、关系约束和迁移定义**如何与 Prisma ORM 进行双向转换，并明确正向生成与反向导入的职责边界。

---

## 1. 总体架构与代码脉络

### 1.1 双向转换流水线

Amplication 中存在两条方向相反、彼此独立的转换流水线：

| 方向 | 功能 | 触发阶段 | 核心模块 |
|------|------|---------|----------|
| **Entity → Prisma（正向生成）** | 从实体定义生成 `schema.prisma` 文件，并配合静态模板驱动 Prisma Client | 代码构建时（`data-service-generator`） | `packages/data-service-generator/src/server/prisma/` |
| **Prisma → Entity（反向导入）** | 从外部 Prisma schema 解析并规范化为 Amplication 平台实体 | 用户导入时（平台服务端运行时） | `packages/amplication-server/src/core/prismaSchemaParser/` |

两条流水线**不共享任何转换代码**，使用各自的类型系统与解析逻辑。详见第 7 章「正向生成与反向导入的边界」。

### 1.2 代码模块定位

**正向生成（代码构建）：**

- [create-prisma-schema-module.ts](packages/data-service-generator/src/server/prisma/create-prisma-schema-module.ts) — 对外模块入口，封装参数并调用内层
- [create-prisma-schema.ts](packages/data-service-generator/src/server/prisma/create-prisma-schema.ts) — schema 组装（datasource / generator / model / enum）
- [create-prisma-schema-fields.ts](packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) — 字段级类型映射与关系字段处理
- [constants.ts](packages/data-service-generator/src/server/prisma/constants.ts) — 默认 datasource 与 generator 配置
- [create-server.ts](packages/data-service-generator/src/server/create-server.ts) — 服务器代码生成总调度（L128-L129 调用 Prisma schema 生成）
- [prisma.service.ts](packages/data-service-generator/src/server/static/src/prisma/prisma.service.ts) — 静态模板：NestJS `PrismaService`（继承 PrismaClient）
- [prisma.module.ts](packages/data-service-generator/src/server/static/src/prisma/prisma.module.ts) — 静态模板：全局 PrismaModule
- [Dockerfile](packages/data-service-generator/src/server/static/Dockerfile) — 容器构建中自动执行 `npm run prisma:generate`
- [README.md](packages/data-service-generator/src/server/static/README.md) — 入门指引中显式提示 `npm run prisma:generate`

**反向导入（平台运行时）：**

- [prismaSchemaParser.service.ts](packages/amplication-server/src/core/prismaSchemaParser/prismaSchemaParser.service.ts) — Prisma schema → Entity/Field 的七步规范化管线
- [schema-utils.ts](packages/amplication-server/src/core/prismaSchemaParser/schema-utils.ts) — 属性序列化、customAttributes 生成、枚举 @map 处理等工具
- [helpers.ts](packages/amplication-server/src/core/prismaSchemaParser/helpers.ts) — 已语义化属性过滤（filterOutAmplicationAttributesBasedOnFieldDataType）、命名格式化
- [dbSchemaImport.service.ts](packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts) — 用户导入流程调度（Kafka 异步处理）

**共享类型定义（仅正向生成使用）：**

- [code-gen-types.ts](libs/util/code-gen-types/src/code-gen-types.ts) — Entity / EntityField / LookupResolvedProperties 等
- [models.ts](libs/util/code-gen-types/src/models.ts) — EnumDataType 枚举值定义（L991-L1011）
- [dto-util.ts](libs/util/dsg-utils/src/lib/dto-util.ts) — `createEnumName()` 命名规则

---

## 2. 资源字段映射（EnumDataType → Prisma ScalarType）

### 2.1 数据类型枚举

Amplication 内部使用 `EnumDataType` 表示 **19 种**字段数据类型（定义见 [models.ts#L991-L1011](libs/util/code-gen-types/src/models.ts#L991-L1011)）：

```
Boolean, CreatedAt, DateTime, DecimalNumber, Email, File,
GeographicLocation, Id, Json, Lookup, MultiLineText,
MultiSelectOptionSet, OptionSet, Password, Roles,
SingleLineText, UpdatedAt, Username, WholeNumber
```

> 说明：19 个枚举值在 [create-prisma-schema-fields.ts#L77-L524](packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L77-L524) 的 `createPrismaSchemaFieldsHandlers` 分发表中逐一对应处理，无遗漏。

### 2.2 标量字段完整映射表

映射实现集中在 `createPrismaSchemaFieldsHandlers` 分发表。

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

**Id 子类型映射**（[create-prisma-schema-fields.ts#L33-L49](packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L33-L49)）：

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

Lookup 类型的完整转换逻辑位于 [create-prisma-schema-fields.ts#L280-L366](packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L280-L366)。

#### 3.1.1 Lookup 字段属性结构

Lookup 的 `properties` 结构见 [lookup.json](libs/util/code-gen-types/src/schemas/lookup.json)，解析后的形态为：

```typescript
// code-gen-types.ts#L98-L106
interface LookupResolvedProperties {
  relatedEntity: Entity;                    // 关联目标实体
  relatedField: EntityField;                // 对端反向字段
  allowMultipleSelection: boolean;          // 是否允许多选
  isOneToOneWithoutForeignKey?: boolean;    // 一对一且本方不存 FK
  fkFieldName: string;                      // 外键列名（如 customerId）
}
```

#### 3.1.2 三种关系形态与 Prisma 输出

| 关系类型 | 判定条件 | 输出内容 |
|----------|---------|---------|
| **一对多（本方为多端）** | `allowMultipleSelection === false && !isOneToOneWithoutForeignKey` | ① ObjectField（关系导航）+ ② ScalarField（外键列；当对端也是单值时外键加 `@unique` 即退化为一对一） |
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

判断是否需要命名：`hasAnotherRelation`（[create-prisma-schema-fields.ts#L292-L297](packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L292-L297)）—— 只要当前 entity 中存在另一个指向同一 relatedEntity 的 Lookup 字段，就需要命名。

命名算法：`createRelationName()`（[create-prisma-schema-fields.ts#L538-L583](packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts#L538-L583)），优先级递减：

1. 若字段名与对方实体名（单/复数）完全对应 → `"{Remote}On{Local}"`（按字母序拼接）
2. 若某一端字段名全局唯一 → 直接使用该字段名
3. 兜底 → `PascalCase("{Entity} {field} {RelatedEntity} {relatedField}")`（排序后拼接）

### 3.3 枚举字段（OptionSet / MultiSelectOptionSet）

**命名规则**：`createEnumName(field, entity)` = `"Enum" + PascalCase(entity.name) + PascalCase(field.name)`，见 [dto-util.ts#L152-L154](libs/util/dsg-utils/src/lib/dto-util.ts#L152-L154)。

| 类型 | Prisma 输出 |
|------|------------|
| `OptionSet` | `Enum{Entity}{Field}` 对象字段（单值） |
| `MultiSelectOptionSet` | `Enum{Entity}{Field}[]` 对象字段（数组 + 列表） |

枚举定义生成：在 `createPrismaSchemaInternal` 中通过 `getEnumFields(entity)` 汇总，调用 [createPrismaEnum()](packages/data-service-generator/src/server/prisma/create-prisma-schema.ts#L76-L85) 产出 Prisma `enum` 块。

---

## 4. Schema 组装与 ORM 输出

### 4.1 组装流程（Entity → Prisma 文件）

完整流程见 [createPrismaSchemaInternal()](packages/data-service-generator/src/server/prisma/create-prisma-schema.ts#L30-L74)：

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

调用入口见 [create-server.ts#L128-L129](packages/data-service-generator/src/server/create-server.ts#L128-L129)。

### 4.2 默认 Datasource 与 Generator

见 [constants.ts](packages/data-service-generator/src/server/prisma/constants.ts)：

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

从测试快照 `create-data-service.spec.ts.snap` 可看到完整输出形态：

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

## 5. Schema 生成后驱动 Prisma Client 的完整链路

Amplication 的代码生成器**不直接调用 `prisma generate`**，而是通过生成的项目资产组合形成一条完整的"构建 → 生成 Client → 注入 → 调用"链路。链路的四个环节如下：

### 5.1 环节一：生成 `schema.prisma` 文件

由 `createPrismaSchemaModule(entities)` 产出，写入目标项目 `prisma/schema.prisma`，其中 `generator client { provider = "prisma-client-js" }` 声明了 Prisma Client 的生成方式。

### 5.2 环节二：静态模板 —— PrismaService 与 PrismaModule

两个静态文件在 [create-server.ts#L48-L52](packages/data-service-generator/src/server/create-server.ts#L48-L52) 随 `readStaticModules()` 整体拷贝到目标项目：

**[prisma.service.ts](packages/data-service-generator/src/server/static/src/prisma/prisma.service.ts)**

```typescript
import { Injectable, OnModuleInit, INestApplication } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

`PrismaService` 通过**继承** `@prisma/client` 暴露的 `PrismaClient` 类，获得所有 model 的 CRUD 访问器（如 `this.prisma.customer.findMany()`）。`@prisma/client` 是在用户执行 `prisma generate` 时才生成到 `node_modules` 中的包。

**[prisma.module.ts](packages/data-service-generator/src/server/static/src/prisma/prisma.module.ts)**

```typescript
import { Global, Module } from "@nestjs/common";
import { PrismaService } from "./prisma.service";

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

`PrismaModule` 被标记为 `@Global()`，在 AppModule 导入后，项目内所有 Service 都可以直接注入 `PrismaService`。

### 5.3 环节三：生成的业务 Service 注入并调用 PrismaService

每个实体的 Base Service 模板（`{Entity}ServiceBase`）直接依赖注入 `PrismaService`，并用 `@prisma/client` 暴露的强类型（`Prisma.{Entity}Args` / `Prisma.{Entity}`）作为方法签名。例如 Customer：

```typescript
import { PrismaService } from "../../prisma/prisma.service";
import { Prisma, Customer as PrismaCustomer, Order as PrismaOrder } from "@prisma/client";

export class CustomerServiceBase {
  constructor(protected readonly prisma: PrismaService) {}

  async count(args: Omit<Prisma.CustomerCountArgs, "select">): Promise<number> {
    return this.prisma.customer.count(args);
  }
  async findMany<T extends Prisma.CustomerFindManyArgs>(...): Promise<...> {
    return this.prisma.customer.findMany(args);
  }
  // ... findOne / create / update / delete / 关系查询等
}
```

最终的 `CustomerService` 通过继承 `CustomerServiceBase` 获得完整能力：

```typescript
@Injectable()
export class CustomerService extends CustomerServiceBase {
  constructor(protected readonly prisma: PrismaService) {
    super(prisma);
  }
}
```

### 5.4 环节四：`prisma generate` 的触发时机

`prisma generate` 命令通过 [package.json](packages/data-service-generator/src/server/package-json/package.json) 暴露给开发者：

```json
{
  "scripts": {
    "prisma:generate": "prisma generate"
  },
  "dependencies": {
    "@prisma/client": "^6.2.1"
  },
  "devDependencies": {
    "prisma": "^6.2.1"
  }
}
```

**注意**：生成的 `package.json` 中**没有 `postinstall` 钩子自动触发**，开发者需要在以下场景手动执行：

- 首次克隆项目 `npm install` 之后
- 每次 `schema.prisma` 发生变更之后
- CI/CD 流水线中，在 `nest build` 之前

除了 `package.json` 脚本入口外，项目在以下静态模板中也内置了 `prisma:generate` 的自动触发或显式提示：

**Dockerfile（容器构建自动执行）**

生成的 [Dockerfile](packages/data-service-generator/src/server/static/Dockerfile) 在多阶段构建的 base 阶段明确执行 `prisma:generate`，确保容器镜像中包含强类型 Client：

```dockerfile
COPY prisma/schema.prisma ./prisma/
RUN npm run prisma:generate    # 在 COPY . . 与 npm run build 之间执行
RUN npm run build
```

**README.md（用户操作提示）**

生成的 [README.md](packages/data-service-generator/src/server/static/README.md) 在 "Step 2.1: Scripts - pre-requisites" 中显式提示用户在 `npm install` 之后执行 `prisma:generate`：

```sh
# installation of the dependencies
$ npm install
# generate the prisma client
$ npm run prisma:generate
```

**迁移脚本与 Client 生成的关系：** `db:migrate-save`（即 `prisma migrate dev`）在生成迁移文件的同时会自动触发 `prisma generate`；但 `db:migrate-up`（`prisma migrate deploy`）只执行迁移 SQL，不会重新生成 Client。

### 5.5 链路总览

```
Entity[] (code-gen-types)
   │
   ▼  createPrismaSchemaModule()
prisma/schema.prisma  (含 generator client 声明)
   │
   │  开发者执行 npm run prisma:generate (或 prisma migrate dev 隐式触发)
   ▼
node_modules/@prisma/client/  (Prisma 官方生成的强类型 Client)
   │
   ▼  静态模板 prisma.service.ts 继承 PrismaClient
src/prisma/PrismaService (NestJS Injectable)
   │
   ▼  被每个实体 Service 构造注入
src/{entity}/base/{Entity}ServiceBase → 调用 this.prisma.{entity}.{operation}()
   │
   ▼
业务 Controller / Resolver → 对外暴露 REST / GraphQL 接口
```

---

## 6. 迁移定义生成与执行

### 6.1 迁移脚本注入

代码生成器在生成的服务项目 `package.json` 中注入以下 Prisma 迁移脚本（见 [package.json](packages/data-service-generator/src/server/package-json/package.json)）：

| Script | 命令 | 用途 |
|--------|------|------|
| `db:migrate-save` | `prisma migrate dev` | 开发环境：比较 schema 与数据库差异，生成并应用新迁移（同时隐式触发 `prisma generate`） |
| `db:migrate-up` | `prisma migrate deploy` | 生产环境：按序执行所有 pending 迁移（不重新生成 Client） |
| `db:clean` | `prisma migrate reset` | 重置数据库（清空 + 重跑全部迁移） |
| `db:init` | `run-s "db:migrate-save -- --name 'initial version'" db:migrate-up seed` | 首次初始化链路 |
| `prisma:generate` | `prisma generate` | 根据 schema 重新生成 Prisma Client |

迁移**不直接生成 SQL 文件**，而是依赖 Prisma 官方 CLI 在部署/开发时基于 `schema.prisma` 与 `prisma/migrations/` 目录自动计算差异。

### 6.2 Prisma Schema 反向导入（Import 流程）

当用户上传已有 Prisma schema 时，平台通过 `PrismaSchemaParserService` 执行反向解析（[prismaSchemaParser.service.ts](packages/amplication-server/src/core/prismaSchemaParser/prismaSchemaParser.service.ts)）。

#### 6.2.1 七步准备管线（prepareOperations）

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

#### 6.2.2 字段判定顺序

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

#### 6.2.3 多对多关系判定

`isManyToManyRelation()`（L1137-L1238）逻辑：当前字段是 `type[]` 数组 + 对端也存在指向本方 model 的数组类型字段 → 判定为多对多，双方仅各保留一个 Lookup 字段（`allowMultipleSelection: true`）。

#### 6.2.4 customAttributes 保留与过滤机制

反向导入会将 Prisma 属性序列化后按字段数据类型过滤，最终写入实体和字段的 `customAttributes` 字段：

**实体级 customAttributes**（[prismaSchemaParser.service.ts](packages/amplication-server/src/core/prismaSchemaParser/prismaSchemaParser.service.ts) `convertModelToEntity` L1280-L1285）：
- 收集 model 上所有 block attribute（`@@unique`、`@@index`、`@@map` 等）
- 经 [schema-utils.ts](packages/amplication-server/src/core/prismaSchemaParser/schema-utils.ts) 的 `prepareModelAttributes()` 序列化为 `@@xxx(...)` 字符串数组
- 以空格拼接后写入 `entity.customAttributes`

**字段级 customAttributes**（[schema-utils.ts](packages/amplication-server/src/core/prismaSchemaParser/schema-utils.ts) `createOneEntityFieldCommonProperties` L78-L123）：
1. `prepareFieldAttributes(field.attributes)` 将属性 AST 序列化为 `@xxx(...)` 字符串数组
2. `filterOutAmplicationAttributesBasedOnFieldDataType(fieldDataType, ...)`（定义于 [helpers.ts](packages/amplication-server/src/core/prismaSchemaParser/helpers.ts) L79-L96）按字段类型过滤已被 Amplication 语义化的属性：

| 字段数据类型 | 过滤掉的属性 |
|-------------|-------------|
| `Id` | `@id`、`@id()`、`@default(now())`、`@default(cuid())`、`@default(uuid())`、`@default(autoincrement())`；MongoDB 额外过滤 `@map("_id")`、`@db.ObjectId` |
| `CreatedAt` | `@default(now())` |
| `UpdatedAt` | 以 `@updatedAt` 开头的所有属性 |
| `Lookup` | 以 `@relation` 开头的所有属性 |
| 其余标量类型 | 以 `@unique` 开头的所有属性 |

3. 额外过滤空值 `@default()`
4. 剩余属性以空格拼接写入 `field.customAttributes`

#### 6.2.5 枚举 @map / @@map 的跳过与告警

处理 `OptionSet` / `MultiSelectOptionSet` 时，[schema-utils.ts](packages/amplication-server/src/core/prismaSchemaParser/schema-utils.ts) 的 `handleEnumMapAttribute()`（L432-L500）对枚举映射做如下处理：

| 位置 | 处理方式 | 用户日志级别 | 日志内容 |
|------|---------|-------------|---------|
| 枚举块级 `@@map` | 跳过，不写入选项集 | Warning | "The enum '{name}' has been created, but it has not been mapped. Mapping an enum name is not supported." |
| 枚举项级 `@map` | 跳过该属性，选项 label/value 均使用枚举项原始名称 | Warning | "The option '{name}' has been created in the enum '{name}', but its value has not been mapped" |
| 普通枚举项 | 正常加入选项集 | Info | "The option '{name}' has been created in the enum '{name}'" |

---

## 7. 正向生成与反向导入的边界

### 7.1 职责对比

| 维度 | 正向生成（Entity → Prisma） | 反向导入（Prisma → Entity） |
|------|----------------------------|----------------------------|
| **触发阶段** | 代码构建时（`dsg` pipeline） | 用户在平台 UI 上传 `.prisma` 文件时 |
| **所在包** | `packages/data-service-generator` | `packages/amplication-server` |
| **输入** | `Entity[]`（精简版 code-gen-types） | 原始 `.prisma` 文件文本 |
| **输出** | `ModuleMap` 中的 `prisma/schema.prisma` 文件 | 存数据库的 `Entity` + `EntityField` 记录（平台模型） |
| **处理方向** | 结构生成（确定性） | 语义识别 + 规范化（启发式，可能信息损失） |
| **是否使用 DTO 工具** | 是（`@amplication/dsg-utils`） | 否（独立实现） |

### 7.2 不可逆转换与信息损失

反向导入并非正向生成的严格逆运算，以下信息在反向导入时会**损失或做启发式推断**：

| 正向生成输出 | 反向导入能否精确还原 | 原因 |
|-------------|---------------------|------|
| `OptionSet` vs `MultiSelectOptionSet` | 可精确区分（通过是否为 `type[]` 数组） | ✅ |
| `SingleLineText` vs `MultiLineText` vs `Email` vs `Username` vs `Password` vs `GeographicLocation` | ❌ 全部还原为 `SingleLineText` | Prisma 中均映射为 `String`，丢失语义区分 |
| `File` vs `Json` vs `Roles` | ❌ 全部还原为 `Json` | Prisma 中均映射为 `Json` |
| `DecimalNumber` vs `WholeNumber` 的子类型（Float/Decimal/Int/BigInt） | ✅ 可精确区分 | Prisma 保留原始标量类型 |
| Id 的具体子类型（CUID / UUID / AUTO_INCREMENT） | ⚠️ 可部分还原 | 根据 `@default(cuid())` / `@default(uuid())` / `@default(autoincrement())` 识别 |
| `displayName` / `description` / `searchable` | ❌ 完全丢失 | Prisma schema 不承载这些元数据 |
| 实体权限配置（`EntityPermission`） | ❌ 完全丢失 | Prisma schema 不承载权限信息 |
| `permanentId`（跨版本稳定 ID） | ❌ 生成新的 UUID | Prisma schema 中没有该概念 |
| `customAttributes`（字段级） | ✅ 保留过滤后的属性 | 先通过 `prepareFieldAttributes()` 序列化为 `@xxx(...)` 字符串数组，再由 `filterOutAmplicationAttributesBasedOnFieldDataType()` 按字段数据类型过滤掉已被 Amplication 语义化的属性（如 Id 类型过滤 `@id`/`@default(cuid/uuid/autoincrement())`、CreatedAt 过滤 `@default(now())`、UpdatedAt 过滤 `@updatedAt`、Lookup 过滤 `@relation`、其余标量过滤 `@unique`），并去除空值 `@default()`，剩余属性以空格拼接存入 `field.customAttributes` |
| `customAttributes`（实体级） | ✅ 完整保留 model 块属性 | 通过 `prepareModelAttributes()` 将 model 上的 block attribute（如 `@@unique`、`@@index`、`@@map`）序列化为 `@@xxx(...)` 字符串，以空格拼接存入 `entity.customAttributes` |
| 枚举（OptionSet/MultiSelectOptionSet）的 `@map`/`@@map` | ❌ 跳过并告警 | `handleEnumMapAttribute()` 中枚举块级 `@@map` 和枚举项级 `@map` 均被跳过，同时向用户 emit Warning 级日志（"Mapping an enum name is not supported" / "its value has not been mapped"） |

### 7.3 不共享代码的设计原因

两条流水线使用完全独立的实现，主要基于以下原因：

1. **类型系统不同**：正向生成使用 `@amplication/code-gen-types` 中的精简 `Entity` / `EntityField`；反向导入操作的是平台数据库模型 `models.Entity` / `models.EntityField`（含 `__typename`、`resourceId`、`createdAt`、`updatedAt` 等字段）
2. **处理复杂度不同**：正向生成是确定性的一对多展开；反向导入需要处理命名冲突、语法降级（如复合主键 → `@@unique`）、启发式识别（如推断 Id 类型）
3. **演化路径不同**：正向生成跟随 DSG 插件系统演进（通过 `EventNames.CreatePrismaSchema` 事件允许插件扩展）；反向导入属于平台运行时功能，跟随数据库模型演化

---

## 8. 关键类型速查表

### 8.1 EntityField 结构

```typescript
// code-gen-types.ts#L91-L96
export type EntityField = Omit<
  models.EntityField,
  "__typename" | "createdAt" | "updatedAt" | "position" | "dataType"
> & {
  dataType: models.EnumDataType;  // 19 种数据类型之一
};
```

补充重要字段：
- `id: string` — 字段 ID
- `permanentId: string` — 跨版本稳定 ID
- `name: string` — 字段代码名（camelCase）
- `displayName: string` — UI 显示名
- `properties: JsonValue` — 各 dataType 专属配置（见 `schemas/*.json`）
- `required: boolean`
- `unique: boolean`
- `searchable: boolean`
- `customAttributes?: string` — 透传至 Prisma 的自定义注解
- `description: string`

### 8.2 PrismaSchemaDSL 依赖

代码生成使用第三方库 `prisma-schema-dsl` 构建 AST，核心工厂函数：

- `createScalarField(name, type, isList, isRequired, isUnique, isId, isUpdatedAt, defaultValue, ...)`
- `createObjectField(name, type, isList, isRequired, relationName, fields, references, ...)`
- `createModel(name, fields, ..., customAttributes)`
- `createEnum(name, values)`
- `createSchema(models, enums, dataSource, generators)`
- `print(schema)` → 最终 `.prisma` 文本

---

## 9. 文件索引

| 关注点 | 文件路径 |
|--------|---------|
| Prisma schema 生成入口 | [create-prisma-schema-module.ts](packages/data-service-generator/src/server/prisma/create-prisma-schema-module.ts) |
| Model/Enum 组装 | [create-prisma-schema.ts](packages/data-service-generator/src/server/prisma/create-prisma-schema.ts) |
| 字段类型映射与关系 | [create-prisma-schema-fields.ts](packages/data-service-generator/src/server/prisma/create-prisma-schema-fields.ts) |
| Datasource/Generator 默认值 | [constants.ts](packages/data-service-generator/src/server/prisma/constants.ts) |
| 服务器代码生成总调度（调用入口） | [create-server.ts](packages/data-service-generator/src/server/create-server.ts) |
| PrismaService 静态模板 | [prisma.service.ts](packages/data-service-generator/src/server/static/src/prisma/prisma.service.ts) |
| PrismaModule 静态模板 | [prisma.module.ts](packages/data-service-generator/src/server/static/src/prisma/prisma.module.ts) |
| Dockerfile（容器构建自动执行 prisma:generate） | [Dockerfile](packages/data-service-generator/src/server/static/Dockerfile) |
| README（显式提示 prisma:generate） | [README.md](packages/data-service-generator/src/server/static/README.md) |
| 枚举命名规则 | [dto-util.ts](libs/util/dsg-utils/src/lib/dto-util.ts) |
| EnumDataType 定义（19 个值） | [models.ts#L991-L1011](libs/util/code-gen-types/src/models.ts#L991-L1011) |
| Entity/EntityField/Relation 类型 | [code-gen-types.ts](libs/util/code-gen-types/src/code-gen-types.ts) |
| Lookup 属性 JSON Schema | [lookup.json](libs/util/code-gen-types/src/schemas/lookup.json) |
| 生成服务的迁移脚本与 Prisma 依赖 | [package.json](packages/data-service-generator/src/server/package-json/package.json) |
| Prisma Schema 反向解析管线 | [prismaSchemaParser.service.ts](packages/amplication-server/src/core/prismaSchemaParser/prismaSchemaParser.service.ts) |
| 反向导入工具（属性过滤、枚举映射处理等） | [schema-utils.ts](packages/amplication-server/src/core/prismaSchemaParser/schema-utils.ts) |
| 反向导入辅助（属性过滤、命名格式化） | [helpers.ts](packages/amplication-server/src/core/prismaSchemaParser/helpers.ts) |
| Schema Import 调度 | [dbSchemaImport.service.ts](packages/amplication-server/src/core/dbSchemaImport/dbSchemaImport.service.ts) |
| 主平台参考 schema | [schema.prisma](packages/amplication-prisma-db/prisma/schema.prisma) |
| 单元测试（字段映射验证） | [create-prisma-schema.spec.ts](packages/data-service-generator/src/server/prisma/create-prisma-schema.spec.ts) |
