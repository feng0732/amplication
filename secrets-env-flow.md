# Secrets 与环境变量传递流程深度分析

> 路径引用说明：本文档所有代码引用均使用仓库根目录的相对路径，仓库根为 `50-amplication/`。

---

## 1. 架构总览

整个系统在两个层面上处理 secrets 与环境变量，形成"构建时生成 → 运行时注入"的完整闭环：

| 层面 | 职责 | 关键目录 |
|------|------|---------|
| **代码生成层 (DSG - Data Service Generator)** | 根据资源配置和插件声明，动态生成 secrets 枚举、.env 文件、SecretsManager 模块骨架、Auth 工厂代码 | `packages/data-service-generator/` |
| **运行时层 (Generated App)** | 应用启动时由 NestJS 加载环境变量，通过 DI 容器（或直接方式）把 secrets 注入到各业务模块 | `packages/gpt-gateway/`（作为生成代码的参考实例） |

运行时层的代码结构与 DSG 输出完全一致——`gpt-gateway` 本质上就是 DSG 生成出的一个完整应用实例，可作为阅读模板。

---

## 2. 运行时消费者：按读取入口分类

基于 `packages/gpt-gateway` 代码库的完整扫描，环境变量消费者可划分为 **五类读取入口**。同一变量可能出现在多个入口中（如 `BCRYPT_SALT` 同时被 NestJS 服务和独立脚本读取）。

### 2.1 第一类：SecretsManagerService（密钥专属通道）

这是 Amplication 设计的**官方密钥读取通道**。经过类型安全的 `EnumSecretsNameKey` 枚举，支持未来替换为外部 provider（AWS Secrets Manager、Vault 等），消费端代码无需修改。

| 消费者 | 读取的变量/枚举值 | 必须/可选 | 代码位置 |
|--------|-----------------|----------|---------|
| `jwtSecretFactory` | `EnumSecretsNameKey.JwtSecretKey` → `JWT_SECRET_KEY` | 必须 | `packages/gpt-gateway/src/auth/jwt/jwtSecretFactory.ts#L1-L19` |
| `AuthModule` 中 `JwtModule.registerAsync` | `EnumSecretsNameKey.JwtSecretKey` → `JWT_SECRET_KEY` | 必须 | `packages/gpt-gateway/src/auth/auth.module.ts#L29-L31` |

读取链路：

```
SecretsManagerService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey)
  └─► SecretsManagerServiceBase.getSecret("JWT_SECRET_KEY")
       └─► ConfigService.get("JWT_SECRET_KEY")
            └─► process.env.JWT_SECRET_KEY
```

**特征**：
- 必须先通过插件向 `EnumSecretsNameKey` 注册，否则 TypeScript 编译不通过
- 异步调用（即使底层同步，接口强制 `Promise`）
- 对"必须存在"的密钥做 null → 抛错处理

### 2.2 第二类：ConfigService.get()（普通配置通道）

通过 NestJS 全局 `ConfigService` 直接读取，不走 SecretsManager。适用于非敏感的普通配置。

| 消费者 | 读取的变量 | 必须/可选 | 代码位置 |
|--------|----------|----------|---------|
| `PasswordService` 构造函数 | `BCRYPT_SALT` | 必须 | `packages/gpt-gateway/src/auth/password.service.ts#L20-L22` |
| `AuthModule` 中 `JwtModule.registerAsync` | `JWT_EXPIRATION` | 必须 | `packages/gpt-gateway/src/auth/auth.module.ts#L32` |
| `generateKafkaClientOptions` | `KAFKA_BROKERS` | 必须 | `packages/gpt-gateway/src/kafka/generateKafkaClientOptions.ts#L7` |
| 同上 | `KAFKA_ENABLE_SSL` | 可选（默认 `false`） | 同上 `#L8` |
| 同上 | `KAFKA_CLIENT_ID` | 必须 | 同上 `#L9` |
| 同上 | `KAFKA_GROUP_ID` | 必须 | 同上 `#L10` |
| `ServeStaticOptionsService` | `SERVE_STATIC_ROOT_PATH` | 可选（默认 undefined） | `packages/gpt-gateway/src/serveStaticOptions.service.ts#L26-L28` |
| `AppModule` GraphQL `forRootAsync` | `GRAPHQL_SCHEMA_DEST` | 可选（有默认值） | `packages/gpt-gateway/src/app.module.ts#L50` |
| 同上 | `GRAPHQL_DEBUG` | 可选（默认 `false`） | 同上 `#L53` |
| 同上 | `GRAPHQL_PLAYGROUND_ENABLED` | 可选（默认 `false`） | 同上 `#L54` |
| 同上 | `GRAPHQL_INTROSPECTION_ENABLED` | 可选（默认 `false`） | 同上 `#L56` |

**特征**：
- 使用裸字符串 key（如 `"KAFKA_BROKERS"`），无类型安全
- 同步调用
- 不经过 `EnumSecretsNameKey`，也不需要插件注册

**第一类 vs 第二类的边界划分**（在 `auth.module.ts#L24-L44` 中同时可见两种方式）：

```typescript
useFactory: async (secretsService, configService) => {
  // 真正的密钥 → 走 SecretsManagerService（第一类）
  const secret = await secretsService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey);
  // 普通配置 → 走 ConfigService 直读（第二类）
  const expiresIn = configService.get(JWT_EXPIRATION);
  ...
},
inject: [SecretsManagerService, ConfigService],
```

JWT 过期时间（`JWT_EXPIRATION`）虽然和 JWT 有关，但它只是一个时间长度字符串，不属于敏感密钥，所以走第二类。

### 2.3 第三类：process.env 直读（绕过 NestJS DI）

直接访问 Node.js 的 `process.env`，完全绕过 NestJS 的 ConfigService 和 DI 容器。

| 消费者 | 读取的变量 | 必须/可选 | 代码位置 |
|--------|----------|----------|---------|
| `main.ts`（NestJS 启动前） | `PORT` | 可选（默认 `3000`） | `packages/gpt-gateway/src/main.ts#L19` |
| `OpenaiService.createChatCompletion`（每次调用） | `OPENAI_API_KEY` | 必须（缺失则 OpenAI SDK 报错） | `packages/gpt-gateway/providers/openai/openai.service.ts#L28` |

典型代码：

```typescript
// main.ts —— NestJS 容器还没启动，无法依赖注入
const { PORT = 3000 } = process.env;
```

```typescript
// openai.service.ts —— 每次创建新的 OpenAI 实例时读取
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

**特征**：
- 读取时机在 NestJS 启动前（`main.ts`），或服务虽然在 DI 容器内注册但主动绕过注入机制直接读 `process.env`（`OpenaiService`）
- 无类型安全、无默认值管理、无 ConfigModule 的 `validate` 钩子
- 服务本身可被 NestJS 测试模块 mock（如 `{ provide: OpenaiService, useValue: {} }`），但无法通过 ConfigModule.forRoot 单独注入环境变量值

### 2.4 第四类：Prisma env()（Prisma DSL 内置函数）

在 Prisma schema 文件（`schema.prisma`）中使用 Prisma 自己的 `env()` 函数读取环境变量。这是 Prisma 生态的标准做法。

| 消费者 | 读取的变量 | 必须/可选 | 代码位置 |
|--------|----------|----------|---------|
| Prisma schema datasource | `DB_URL` | 必须（缺失则 `prisma generate` 和 `prisma migrate` 失败） | `packages/gpt-gateway/prisma/schema.prisma#L1-L4` |

实际代码：

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DB_URL")
}
```

**DSG 生成来源**：
- 配置定义于 `packages/data-service-generator/src/server/prisma/constants.ts#L12-L16`：
  ```typescript
  export const DATA_SOURCE: DataSource = {
    name: "postgres",
    provider: DataSourceProvider.PostgreSQL,
    url: { name: "DB_URL" },   // ★ 这里硬编码了 env 变量名
  };
  ```
- `create-prisma-schema.ts#L51-L67` 把 `DATA_SOURCE` 传给 `prisma-schema-dsl` 的 `print()` 函数，序列化为 `url = env("DB_URL")`

**特征**：
- 读取时机在 NestJS 启动前（`prisma generate` 在构建时执行，`prisma migrate` 在部署时执行）
- 完全不在 NestJS 的生命周期内，无法依赖注入
- Prisma 自身的 CLI 工具链需要这个值

### 2.5 第五类：脚本读取（独立 Node.js 脚本）

不在 NestJS 应用内运行的独立脚本（如 seed 脚本），需要自己手动加载 `.env` 文件。

| 消费者 | 读取的变量 | 必须/可选 | 代码位置 |
|--------|----------|----------|---------|
| `scripts/seed.ts`（数据库 seed 脚本） | `BCRYPT_SALT` | 必须 | `packages/gpt-gateway/scripts/seed.ts#L7-L18` |

实际代码：

```typescript
// 1. 手动加载 .env（NestJS 的 ConfigModule 不会自动加载）
dotenv.config();

// 2. 从 process.env 读取
const { BCRYPT_SALT } = process.env;

if (!BCRYPT_SALT) {
  throw new Error("BCRYPT_SALT environment variable must be defined");
}
```

**DSG 生成来源**：
- 模板文件 `packages/data-service-generator/src/server/seed/seed.template.ts#L11-L19` 预置了这段读取逻辑
- `create-seed.ts#L126-L154` 通过 AST interpolation 把模板生成到最终的 `scripts/seed.ts`

**特征**：
- 运行在 NestJS 外部，必须显式调用 `dotenv.config()` 才能读取 `.env`
- 直接读 `process.env`，与第三类类似，但有前置的 `dotenv.config()` 步骤

---

## 3. 三个重点变量的来源与归属分析

### 3.1 OPENAI_API_KEY

**来源**：`.env` 文件第 6 行
```env
OPENAI_API_KEY=[open-ai-key]
```

**归属**：第三类（process.env 直读）——虽然服务本身注册在 NestJS DI 容器内，但读取方式绕过了所有注入机制。

**读取位置**：`packages/gpt-gateway/providers/openai/openai.service.ts#L28`

**读取代码**：
```typescript
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

**模块注册路径的真实链路**：

```
AppModule (src/app.module.ts#L33)
  └─ imports: [TemplateModule, ...]
       │
       └─ TemplateModule (src/template/template.module.ts#L8-L12)
            ├─ imports: [TemplateModuleBase, forwardRef(() => AuthModule)]
            ├─ providers: [TemplateService, TemplateResolver, OpenaiService]   // ★ OpenaiService 直接列在这里
            └─ exports: [TemplateService]   // 只导出了 TemplateService，OpenaiService 是私有实现细节
                 │
                 └─ TemplateService (src/template/template.service.ts#L17)
                      └─ constructor(private openaiService: OpenaiService, ...)   // ★ 只有 TemplateService 能注入它
```

**三件关键事实**：

#### 事实一：OpenaiService 由 TemplateModule 直接注册

`OpenaiService` 不是通过 import 某个模块引入的，而是被直接列在 `TemplateModule` 的 `providers` 数组里（`packages/gpt-gateway/src/template/template.module.ts#L10`）：

```typescript
@Module({
  imports: [TemplateModuleBase, forwardRef(() => AuthModule)],
  providers: [TemplateService, TemplateResolver, OpenaiService],   // ★ 直接注册
  exports: [TemplateService],
})
export class TemplateModule {}
```

这意味着：
- `OpenaiService` 的实例生命周期由 `TemplateModule` 管理
- 其他模块即使 import 了 `TemplateModule`，也无法注入 `OpenaiService`（因为它不在 `exports` 里）
- 只有 `TemplateModule` 内部的 `TemplateService` 和 `TemplateResolver` 能使用它

`TemplateModule` 本身被 `AppModule`（`src/app.module.ts#L33`）和 `ConversationTypeModule`（`src/conversationType/conversationType.module.ts#L12`）import，所以它是正式的 DI 容器成员，不是"游离在外的独立 service"。

#### 事实二：OpenAIModule 存在但完全未被使用（僵尸模块）

`packages/gpt-gateway/providers/openai/openai.module.ts#L1-L8` 虽然定义了：

```typescript
@Module({
  providers: [OpenaiService],
  exports: [OpenaiService],
})
export class OpenAIModule {}
```

但代码库中**没有任何地方 import 了 `OpenAIModule`**：
- `AppModule` 的 imports 数组里没有 `OpenAIModule`
- `TemplateModule` 的 imports 数组里也没有 `OpenAIModule`
- 全项目 grep 不到 `import { OpenAIModule }` 的引用

它是一个被写出来但从未接入的僵尸模块。可能的历史原因：开发者最初打算让 OpenaiService 通过独立模块发布，但后来图省事直接把它塞进了 TemplateModule 的 providers，忘了删除 `openai.module.ts`。

#### 事实三：OpenaiService 没有构造函数，没有注入任何依赖（包括 SecretsManagerService 和 ConfigService）

`packages/gpt-gateway/providers/openai/openai.service.ts#L20-L54`：

```typescript
@Injectable()
export class OpenaiService {
  // ★ 没有 constructor()！
  async createChatCompletion(...) {
    const openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,   // ★ 硬编码 process.env
    });
    ...
  }
}
```

虽然它是 `@Injectable()` 装饰的 NestJS provider，但它：
1. **没有 constructor**——不接受任何依赖注入
2. 每次 `createChatCompletion()` 调用时就地 `new OpenAI({ apiKey: process.env.OPENAI_API_KEY })`
3. 完全绕过了 NestJS 提供的 ConfigService 和 SecretsManagerService

这是典型的"类被注册进了 DI 容器，但内部实现仍然是全局状态直读"的反模式。

**不走密钥链路（第一类 SecretsManagerService）的原因**：

1. **非 DSG 生成代码**：`gpt-gateway/providers/openai/` 和 `gpt-gateway/src/template/` 中涉及 OpenAI 的代码都是该项目的自定义扩展。DSG 生成的 providers 目录只有 `secrets/` 一个子目录，TemplateModule 的 DSG 模板（`template.module.base.ts`）也没有 OpenaiService。自定义代码未被纳入 Amplication 的插件体系，没有通过 `CreateServerSecretsManager` 的 before 钩子向 `EnumSecretsNameKey` 注册 `OpenAiApiKey`。

2. **TemplateModule 没有 import SecretsManagerModule**：TemplateModule 的 imports 数组只有 `[TemplateModuleBase, forwardRef(() => AuthModule)]`（`src/template/template.module.ts#L9`），没有 `SecretsManagerModule`。即使 OpenaiService 想注入 `SecretsManagerService`，NestJS 也会报 "Nest can't resolve dependencies"。

3. **OpenaiService 完全没有构造函数注入**：它没有 `constructor`，不接受任何依赖注入。即使 TemplateModule 正确 import 了 SecretsManagerModule，也无法把 SecretsManagerService 传入 OpenaiService——因为没人接收。

4. **每次调用就地 new SDK 的实现方式**：`OpenaiService` 在每次 `createChatCompletion()` 调用时才 `new OpenAI({...})`，而不是在构造函数里持有一个 SDK 单例。这种模式让开发者很自然地就地读 `process.env`，而不是把 async 的 `getSecret()` 调用塞进同步的构造逻辑。

5. **僵尸模块 OpenAIModule 的存在反而妨碍了正确接入**：如果开发者想通过 import OpenAIModule 来使用 OpenaiService，就会发现 OpenAIModule 也没有 import SecretsManagerModule——两个模块都"各自为政"，没有一个正确的接入点。

**实际上是设计疏漏**：从安全性角度看，`OPENAI_API_KEY` 是真正的敏感密钥，应该走第一类 SecretsManagerService。当前实现意味着：
- 如果未来替换为外部 Secrets Provider（如 AWS Secrets Manager），`OPENAI_API_KEY` 不会被自动纳入
- 缺失时不会抛出清晰的 Amplication 风格错误，而是由 OpenAI SDK 抛一个 `APIError`
- 没有出现在 `EnumSecretsNameKey` 枚举中，不利于"项目有哪些 secrets"的可发现性

**正确的接入方式（假设要修复）**：
1. 在插件中向 `CreateServerSecretsManager` 的 before 钩子追加 `{ name: "OpenAiApiKey", key: "OPENAI_API_KEY" }`
2. 在 TemplateModule 的 imports 中加入 `SecretsManagerModule`
3. 给 OpenaiService 加 constructor，注入 `SecretsManagerService`
4. 把 SDK 单例提到构造函数里（或使用 async Factory Provider 延迟创建），通过 `getSecret<string>(EnumSecretsNameKey.OpenAiApiKey)` 获取密钥

### 3.2 DB_URL

**来源**：`.env` 文件第 7 行
```env
DB_URL=postgres://admin:admin@localhost:5432/gpt-gateway
```

**归属**：第四类（Prisma env()）

**读取位置**：`packages/gpt-gateway/prisma/schema.prisma#L3`

**读取代码**：
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DB_URL")
}
```

**DSG 生成链路**：
```
packages/data-service-generator/src/server/prisma/constants.ts
  DATA_SOURCE = { url: { name: "DB_URL" } }
        │
        ▼
packages/data-service-generator/src/server/prisma/create-prisma-schema-module.ts
  createPrismaSchema({ dataSource: DATA_SOURCE, ... })
        │
        ▼
packages/data-service-generator/src/server/prisma/create-prisma-schema.ts#L51-L67
  prismaDataSource = { ...dataSource.url }
  PrismaSchemaDSL.createSchema(..., prismaDataSource, ...)
  PrismaSchemaDSL.print(schema)
        │
        ▼
输出 schema.prisma: url = env("DB_URL")
```

**不走密钥链路（第一类 SecretsManagerService）的原因**：

1. **技术边界不可跨越**：Prisma schema 是 Prisma 自己的 DSL（`.prisma` 文件），不是 TypeScript 代码，无法 import 或注入 NestJS 的 `SecretsManagerService`。Prisma 只识别它自己的 `env()` 函数。

2. **时机不可调和**：`prisma generate`（生成 Prisma Client）在**构建阶段**就需要 `DB_URL`，`prisma migrate deploy` 在**部署阶段**需要它——这两个动作都发生在 NestJS 应用启动之前。即使想通过 SecretsManagerService 读取，也没有运行的 DI 容器可用。

3. **Prisma 生态的强制约定**：Prisma 的所有官方文档和最佳实践都使用 `env("DATABASE_URL")` 或 `env("DB_URL")`。Amplication 遵循这个约定而不是发明自己的方案，降低了用户的认知成本。

4. **DSG 核心硬编码，不走插件管道**：`DB_URL` 直接写在 DSG 的 `constants.ts` 里，不需要通过 `CreateServerSecretsManager` 的 before 钩子由插件注入。这与 `JWT_SECRET_KEY`（由 auth 插件注入）形成鲜明对比——数据库是 Amplication 认为"每个项目一定需要"的少数几个硬假设之一。

> 注意：虽然 Prisma 层面用 `env()` 直接读，但部署时可以通过环境变量管理工具（如 Docker secrets、K8s Secrets）把 `DB_URL` 的值注入到 `process.env`，与 SecretsManager 的外部 provider 替换并不冲突。

### 3.3 BCRYPT_SALT

**来源**：`.env` 文件第 1 行（DSG 默认值）
```env
BCRYPT_SALT=10
```

DSG 默认值定义于 `packages/data-service-generator/src/server/constants.ts#L7-L11`：
```typescript
export const ENV_VARIABLES: VariableDictionary = [
  { BCRYPT_SALT: "10" },          // ★ DSG 核心内置，非插件注入
  { COMPOSE_PROJECT_NAME: "amp_${resourceId}" },
  { PORT: "3000" },
];
```

**归属**：双归属——第二类 + 第五类

| 场景 | 读取入口 | 代码位置 |
|------|---------|---------|
| NestJS 运行时密码哈希/比对 | 第二类：`ConfigService.get("BCRYPT_SALT")` | `packages/gpt-gateway/src/auth/password.service.ts#L20-L22` |
| 独立 seed 脚本创建初始用户 | 第五类：`dotenv.config()` + `process.env.BCRYPT_SALT` | `packages/gpt-gateway/scripts/seed.ts#L7-L18` |

**不走密钥链路（第一类 SecretsManagerService）的原因**：

1. **语义上不是密钥**：`BCRYPT_SALT` 实际上是 bcrypt 的**成本因子（rounds）**——一个正整数，决定了哈希计算需要多少次迭代。它不是需要保密的密钥：即使攻击者知道 `rounds=10`，也无法从哈希反推密码。Amplication 本身的注释也承认了这一点：`packages/gpt-gateway/src/auth/password.service.ts#L15-L17`：
   > "the salt to be used to hash the password. if specified as a **number** then a salt will be generated with the specified number of rounds and used"

2. **跨场景读取需求**：同一变量需要在两个完全独立的执行环境中读取：
   - NestJS 应用内部（`PasswordService`）—— 可走 DI 容器
   - 独立 seed 脚本（`scripts/seed.ts`）—— 无法走 DI 容器，需要 `dotenv.config()` + `process.env`
   
   如果把它放进 `EnumSecretsNameKey`，seed 脚本就无法复用 SecretsManagerService，需要自己实现一套读取逻辑——反而增加了复杂度。

3. **DSG 核心内置，非敏感默认值**：`BCRYPT_SALT` 的默认值 `10` 是公开的行业标准推荐值（2023 年 OwASP 建议成本因子至少 10），没有任何保密性。它和 `PORT=3000`、`COMPOSE_PROJECT_NAME=amp_xxx` 一样是"开箱即用的默认配置"，不是"需要每个项目独立保密的密钥"。

4. **需要数值解析**：`parseSalt()` 函数（`password.service.ts#L50-L64`）会尝试把字符串解析为整数 rounds。如果走 SecretsManagerService，这个解析逻辑仍然需要写在消费端——并没有简化代码。

---

## 4. 五类消费者全景对比表

| 类别 | 读取方式 | 类型安全 | 异步 | 是否受 EnumSecretsNameKey 约束 | NestJS DI 内 | 典型变量 |
|------|---------|---------|------|-------------------------------|------------|---------|
| **1. SecretsManagerService** | `SecretsManagerService.getSecret(EnumSecretsNameKey.X)` | ✅ | ✅ | ✅ | ✅ | `JWT_SECRET_KEY` |
| **2. ConfigService.get()** | `ConfigService.get("XXX")` | ❌ | ❌ | ❌ | ✅ | `BCRYPT_SALT`, `JWT_EXPIRATION`, `KAFKA_*`, `GRAPHQL_*` |
| **3. process.env 直读** | `process.env.XXX` | ❌ | ❌ | ❌ | ⚠️ 部分 | `PORT`（DI 外）, `OPENAI_API_KEY`（服务在 DI 内但绕过注入） |
| **4. Prisma env()** | `env("XXX")` in `.prisma` | ❌ | ❌ | ❌ | ❌（Prisma 自有 DSL） | `DB_URL` |
| **5. 脚本读取** | `dotenv.config()` + `process.env.XXX` | ❌ | ❌ | ❌ | ❌（独立脚本） | `BCRYPT_SALT`（seed 脚本中） |

---

## 5. 密钥作用域与消费者划分

### 5.1 核心数据结构

Secrets 的声明采用"语义名 + 环境变量名"双字段设计，定义于 `libs/util/code-gen-types/src/plugin-events-params.types.ts#L371-L374`：

```typescript
export interface SecretsNameKey {
  name: string;   // 语义名，将被转为 PascalCase 枚举成员（如 "JwtSecretKey"）
  key: string;    // 实际的环境变量键名（如 "JWT_SECRET_KEY"）
}
```

对应的构建期事件参数在同文件 `L363-L369`：

```typescript
export interface CreateServerSecretsManagerParams extends EventParams {
  /**
   * Array of secretsName secrectKey pairs that will generate the SecretsNameKey enum.
   * SecrectKey is used by the Secrets Manager Service to retrieve the secret value
   */
  secretsNameKey: SecretsNameKey[];
}
```

### 5.2 动态枚举生成

枚举 `EnumSecretsNameKey` 并非手写文件，而是在构建期由 AST 动态生成。生成逻辑位于 `packages/data-service-generator/src/server/secrets-manager/create-secrets-manager.ts#L70-L83`：

```typescript
function createTSEnumSecretsNameKey(
  secretsNameKey: SecretsNameKey[]
): namedTypes.TSEnumDeclaration {
  const ENUM_SECRETS_NAME_KEY = builders.identifier("EnumSecretsNameKey");
  return builders.tsEnumDeclaration(
    ENUM_SECRETS_NAME_KEY,
    secretsNameKey.map(({ name, key }) =>
      builders.tsEnumMember(
        builders.identifier(pascalCase(name)),  // name → 枚举标识符
        builders.stringLiteral(key)              // key → 枚举字符串字面量值
      )
    )
  );
}
```

**转换示例**：
- 输入 `{ name: "JwtSecretKey", key: "JWT_SECRET_KEY" }`
- 输出 `JwtSecretKey = "JWT_SECRET_KEY"`

生成后的枚举文件落点为 `{serverSrcDir}/providers/secrets/secretsNameKey.enum.ts`，运行时实例可参考 `packages/gpt-gateway/src/providers/secrets/secretsNameKey.enum.ts`：

```typescript
export enum EnumSecretsNameKey {
  JwtSecretKey = "JWT_SECRET_KEY",
}
```

**关键结论**：`EnumSecretsNameKey` 只包含第一类消费者（SecretsManagerService）使用的密钥。第二类到第五类的变量均不被纳入——这是有意的设计，枚举不是"项目所有环境变量"的全集，而是"需要经过密钥管理通道的敏感密钥"的白名单。

### 5.3 空数组起步 —— 完全开放的扩展点

在 `packages/data-service-generator/src/server/create-server.ts#L87-L90` 中，DSG 对 SecretsManager 的初始调用传入**空数组**：

```typescript
await context.logger.info("Creating SecretsManager...");
const secretsManagerModule = await createSecretsManager({
  secretsNameKey: [],   // ★ 初始为空，完全靠插件 before 钩子追加
});
```

这意味着：**默认情况下 `EnumSecretsNameKey` 会是一个空枚举**。所有具体的密钥声明（包括 `JWT_SECRET_KEY` 等）都必须由插件通过 `CreateServerSecretsManager` 事件的 `before` 钩子注入。

对比：
- `JWT_SECRET_KEY`：由 auth 插件通过 before 钩子注入 → 进入 `EnumSecretsNameKey` → 走第一类通道
- `DB_URL`：由 DSG 核心 `constants.ts` 硬编码 → 不进入枚举 → 走第四类 Prisma 通道
- `BCRYPT_SALT`：由 DSG 核心 `ENV_VARIABLES` 默认值内置 → 不进入枚举 → 走第二类/第五类通道
- `OPENAI_API_KEY`：由 `gpt-gateway` 项目自定义扩展使用；服务本身被 `TemplateModule.providers` 注册进 DI 容器，但未通过插件向枚举注册，且 OpenaiService 无 constructor 绕过了注入机制 → 走第三类通道

### 5.4 作用域的三层边界

**第一层：枚举定义层 —— 编译期白名单**

`EnumSecretsNameKey` 是类型安全的"门禁"。只有被纳入枚举的 key 才能被 `SecretsManagerService.getSecret<T>()` 合法读取。

注意：这个门禁只对第一类消费者（走 SecretsManagerService 的）有效。第二类到第五类消费者直接用裸字符串读 ConfigService/process.env/Prisma env()，不受枚举约束。

**第二层：Module 导出层 —— NestJS DI 容器边界**

SecretsManager 通过独立的 Module 封装，运行时参考 `packages/gpt-gateway/src/providers/secrets/secretsManager.module.ts`：

```typescript
@Module({
  providers: [SecretsManagerService],
  exports: [SecretsManagerService],
})
export class SecretsManagerModule {}
```

只有显式 `imports: [SecretsManagerModule]` 的业务模块，其 provider/factory 才能注入 `SecretsManagerService`。典型如 `packages/gpt-gateway/src/auth/auth.module.ts#L21`。

而 `ConfigModule.forRoot({ isGlobal: true })` 是全局模块，任何模块无需显式 import 就能注入 `ConfigService`。这也是为什么第二类消费者（普通配置）随处可见，而第一类消费者（密钥）必须在特定模块内使用。

**第三层：运行时取值层 —— 环境变量边界**

读取的最底层实现位于 `packages/data-service-generator/src/server/secrets-manager/static/base/secretsManager.service.base.template.ts#L8-L17`：

```typescript
export class SecretsManagerServiceBase implements ISecretsManager {
  constructor(protected readonly configService: ConfigService) {}
  async getSecret<T>(key: EnumSecretsNameKey): Promise<T | null> {
    const value = this.configService.get(key.toString());
    return value ? value : null;
  }
}
```

两个设计细节：
1. **返回 `Promise<T | null>`**：虽然默认的 `ConfigService.get` 是同步的，但接口强制异步。为外部 provider（AWS Secrets Manager、HashiCorp Vault 等需要网络 IO 的实现）预留扩展空间。
2. **返回 `null` 而非抛异常**：把"secret 是否必须存在"的判断权交给调用方。

---

## 6. 构建上下文传递机制

### 6.1 端到端构建管线

```
generateCode()                          [src/generate-code.ts]
  └─► generateCodeByResourceData()
       └─► createDataService()          [src/create-data-service.ts]
            ├─► prepareDefaultPlugins()        // 注入内置插件
            ├─► dynamicPackagesInstallations() // 动态安装插件 npm 包
            ├─► prepareContext()                // [src/prepare-context.ts]
            │    ├─► registerPlugins()          // 加载插件，按事件名分组
            │    ├─► 规范化 entities / roles / serviceTopics
            │    ├─► 计算 serverDirectories / clientDirectories
            │    └─► context.plugins = plugins  // ★ 挂载钩子表
            ├─► createDTOs()
            ├─► createServer()                  // [src/server/create-server.ts]
            │    ├─► ... 资源、DTO、Swagger ...
            │    ├─► createAuthModules()
            │    ├─► createMessageBroker()
            │    ├─► createSecretsManager({ secretsNameKey: [] })  // ★ 初始空
            │    ├─► createAppModule(...)       // 组装所有模块到 imports
            │    ├─► createPrismaSchemaModule()
            │    ├─► createDotEnvModule({ envVariables: ENV_VARIABLES })
            │    ├─► createSeed()
            │    └─► createDockerComposeFile() ...
            │
            │    └─► createServerInternal() 末尾手动 mergeMany 所有子模块返回值
            │
            └─► createAdminModules()            // Admin UI 端
                 └─► createDotEnvModule()

            └─► modules.merge(createServer() 返回值)    // [create-data-service.ts#L75]
            └─► modules.merge(createAdminModules() 返回值)
            └─► return modules;                        // 最终产物
```

### 6.2 DsgContext —— 单例全局上下文

`packages/data-service-generator/src/dsg-context.ts#L19-L81` 定义了贯穿整个构建过程的状态容器，采用经典单例模式：

```typescript
class DsgContext implements types.DsgContext {
  public appInfo!: types.AppInfo;
  public entities: types.Entity[] = [];
  public plugins: types.PluginMap = {};     // 事件名 → before/after 钩子数组
  public modules: types.ModuleMap;           // 插件间共享的已生成文件集合
  public serverDirectories!: serverDirectories;
  public clientDirectories!: clientDirectories;
  public utils: ContextUtil = {
    skipDefaultBehavior: false,              // 插件可设为 true 跳过默认生成
    abortGeneration: (msg) => { ... },
    abort: false,
    importStaticModules: readPluginStaticModules,
  };

  private static instance: DsgContext;
  public static get getInstance(): DsgContext {
    return this.instance || (this.instance = new this());
  }
  private constructor() { ... }
}
```

与 secrets/env 相关的关键字段：
- `appInfo.settings`（类型：`ServiceSettings`）：资源级配置字典，是 `.env` 中 `${resourceId}` 等占位符替换的数据源
- `plugins`：所有插件钩子的注册表
- `serverDirectories.baseDirectory` / `clientDirectories.baseDirectory`：决定 `.env` 文件的输出位置
- `modules`：插件跨事件共享已生成文件的渠道

### 6.3 插件注册 —— 事件钩子的装配

`packages/data-service-generator/src/register-plugin.ts#L129-L160` 完成插件加载与分类：

```typescript
const registerPlugins = async (pluginList, pluginInstallationPath?) => {
  const pluginMap: PluginMap = {};
  const pluginFuncsArr = await getAllPlugins(pluginList, pluginInstallationPath);

  pluginFuncsArr.reduce((pluginContext, plugin) => {
    Object.keys(plugin).forEach(eventKey => {
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

最终形成的 `PluginMap` 结构（`libs/util/code-gen-types/src/plugins.types.ts#L70-L75`）：

```typescript
export type PluginMap = {
  [K in EventNames]?: {
    before?: PluginBeforeEvent<EventParams>[];
    after?: PluginAfterEvent<EventParams>[];
  };
};
```

### 6.4 pluginWrapper —— 洋葱模型的执行引擎（完整细节）

`packages/data-service-generator/src/plugin-wrapper.ts#L59-L117` 是每一个代码生成函数的通用包装器。逐段拆解如下。

#### 入口与控制标志重置

```typescript
const pluginWrapper: PluginWrapper = async (
  func,
  event,
  args
): Promise<ModuleMap> => {
  const context = DsgContext.getInstance;

  try {
    context.utils.skipDefaultBehavior = false;
    context.utils.abort = false;
```

每次事件触发前都会把 `skipDefaultBehavior` 和 `abort` 重置为 `false`，避免渗漏到后续事件。

#### 无插件分支 —— 快路径

```typescript
    if (!context.plugins.hasOwnProperty(event)) {
      return await func(args);   // ★ 直接返回，跳过所有后续步骤
    }
```

当某个事件没有任何插件注册钩子时走这个快路径。**它跳过的内容包括：**

1. 不执行 before 管道
2. 不经过 `defaultBehavior()` 包装（即使想设置 `skipDefaultBehavior` 也无效）
3. 不执行 after 管道
4. **不执行 upsert 到 `context.modules`**

但模块不会丢失，因为：
- `createServerInternal()` 末尾（`create-server.ts#L144-L167`）会用 `moduleMap.mergeMany([...所有子函数返回值...])` 手动收集
- `create-data-service.ts#L75` 再 `await modules.merge(await createServer())` 把 Server 端全部产物合并到最终输出

无插件分支是性能优化 + 逻辑简化：绝大多数事件在无插件时不需要进入复杂管道。

#### before 管道

```typescript
    const beforePlugins = context.plugins[event]?.before || [];
    const afterPlugins = context.plugins[event]?.after || [];

    const updatedEventParams = beforePlugins
      ? await beforeEventsPipe(...beforePlugins)(context, args)
      : args;
```

`beforeEventsPipe` 的实现（同文件 `L17-L23`）：

```typescript
const beforeEventsPipe =
  (...fns: PluginBeforeEvent<EventParams>[]) =>
  (context: DsgContext, eventParams: EventParams) =>
    fns.reduce(
      async (res, fn) => fn(context, await res),
      Promise.resolve(eventParams)
    );
```

使用 `reduce` + `await` 实现串行管道，前一个插件的返回值作为下一个插件的输入。`eventParams` 是可就地修改的对象引用。

#### 默认行为包装

```typescript
const defaultBehavior = async (context, func, beforeFuncResults) => {
  if (context.utils.skipDefaultBehavior)
    return new ModuleMap(DsgContext.getInstance.logger);  // 跳过，返回空 Map
  return util.types.isAsyncFunction(func)
    ? await func(beforeFuncResults)
    : func(beforeFuncResults);
};
```

这里接收的是 `updatedEventParams`（经过 before 管道修改后的版本），不是原始 `args`。

#### after 管道 —— 关键细节：收到的是原始 args

```typescript
    const finalModules = afterPlugins
      ? await afterEventsPipe(...afterPlugins)(
          context,
          args,                   // ★ 注意：这里是原始 args，不是 updatedEventParams
          defaultBehaviorModules
        )
      : defaultBehaviorModules;
```

`afterEventsPipe` 的实现（同文件 `L25-L31`）：

```typescript
const afterEventsPipe =
  (...fns: PluginAfterEvent<EventParams>[]) =>
  (context: DsgContext, eventParams: EventParams, modules: ModuleMap) =>
    fns.reduce(
      async (res, fn) => fn(context, eventParams, await res),
      Promise.resolve(modules)
    );
```

**极易忽略的细节**：after 钩子收到的 `eventParams` 是**原始的 `args`**，不是 before 管道输出的 `updatedEventParams`。

实际影响：
- before 钩子如果就地修改（`eventParams.secretsNameKey.push(...)`），因为是对象引用，after 钩子仍能看到追加的内容
- 如果 before 钩子返回了全新对象（`return { ...eventParams, extra: "xxx" }`），after 钩子**看不到**新增的 `extra` 字段

插件约定：before 钩子应就地修改参数，不要返回新对象。

#### Upsert 到 context.modules —— 插件间共享文件

```typescript
    for (const module of finalModules.modules()) {
      context.modules.replace(module, module);
    }

    return finalModules;
```

**这一步只有在"有插件注册"的分支才会执行**（无插件分支已经 return）。目的是让后续事件的插件可以访问当前事件生成的文件。例如：
1. `CreateServerSecretsManager` 事件生成了 `secretsNameKey.enum.ts`
2. Upsert 到 `context.modules`
3. 后续的 `CreateServerAuth` 事件的 after 钩子可以通过 `context.modules` 找到该文件

**双重保障的输出链路**：
- 路径 A（有插件时）：pluginWrapper upsert + 函数返回值被 mergeMany 收集
- 路径 B（无插件时）：函数返回值被 mergeMany 收集
- 最终：都汇入 `create-data-service.ts#L75` 的 `modules.merge(createServer())`

### 6.5 Secrets 与环境变量的插件注入时机

| 事件 | 钩子阶段 | 插件可做什么 |
|------|---------|------------|
| `CreateServerSecretsManager` | **before** | 向 `eventParams.secretsNameKey[]` 追加 `{ name, key }`，使其被纳入 `EnumSecretsNameKey` 枚举（影响第一类消费者） |
| `CreateServerDotEnv` | **before** | 向 `eventParams.envVariables` 追加或覆盖变量条目（影响 `.env` 文件内容，所有五类消费者都从这里取值） |
| `CreateServerDotEnv` | **after** | 修改已生成的 `.env` 文件内容（如追加行、替换值）；也可通过 `context.modules` 找到其他已生成的文件协同修改 |
| `CreateAdminDotEnv` | before / after | 同上，针对 Admin UI 端 `.env` |
| `CreateServerAppModule` | after | 修改 `app.module.ts`，如替换 `ConfigModule.forRoot()` 的配置以接入自定义 loader |
| `CreatePrismaSchema` | before / after | 修改 Prisma schema，如替换 `env("DB_URL")` 为其他变量名（影响第四类消费者） |
| `CreateSeed` | before / after | 修改 seed 脚本的环境变量读取逻辑（影响第五类消费者） |

### 6.6 .env 文件生成详解

#### Server 端

`packages/data-service-generator/src/server/create-dotenv.ts#L28-L48`：

```
输入: envVariables: VariableDictionary = [{ BCRYPT_SALT: "10" }, ...]
  │
  ├─► removeDuplicateKeys()       // Map 去重，后出现的同 key 覆盖先出现的
  ├─► sortAlphabetically()        // 按 key 字母排序，保证多次构建输出稳定
  ├─► convertToKeyValueSting()    // 序列化为 "KEY=value\nKEY2=value2" 格式
  └─► replacePlaceholdersInCode() // ${resourceId} 等占位符替换
       │
       └─► 输出: {serverBaseDir}/.env
```

- `VariableDictionary` 类型：`{ [variable: string]: string }[]`，`libs/util/code-gen-types/src/plugin-events-params.types.ts#L181-L183`
- 默认变量在 `packages/data-service-generator/src/server/constants.ts#L7-L11`：
  ```typescript
  export const ENV_VARIABLES: VariableDictionary = [
    { BCRYPT_SALT: "10" },
    { COMPOSE_PROJECT_NAME: "amp_${resourceId}" },
    { PORT: "3000" },
  ];
  ```
- 去重逻辑使用 `Map` 按 key 去重，后入的覆盖先入的。**插件的 before 钩子注入的变量会覆盖默认 `ENV_VARIABLES` 中的同名变量**

#### Admin 端

`packages/data-service-generator/src/admin/create-dotenv.ts#L33-L60` 与 Server 端的关键区别：有模板文件作为变量基础来源。

```typescript
const code = await readCode(templatePath);                           // 读取 create-dotenv.template.env
const extractedVariables = extractVariablesFromCode(code);          // 解析模板中的预置变量
const allVariables = [
  ...extractedVariables,                                             // 模板中的先入
  ...envVariablesWithoutDuplicateKeys,                               // 插件注入的后入（可覆盖模板）
];
```

模板文件 `packages/data-service-generator/src/admin/create-dotenv.template.env` 预置了：
```
PORT=3001
VITE_REACT_APP_SERVER_URL=http://localhost:3000
```

变量提取由 `packages/data-service-generator/src/utils/dotenv.ts#L21-L33` 完成，非常朴素的解析：按行用 `=` 分割，非 `KEY=VALUE` 格式的行被静默丢弃。

---

## 7. 占位符替换机制深度分析

### 7.1 核心实现

`packages/data-service-generator/src/utils/text-file-parser.ts#L1-L19`：

```typescript
export function replacePlaceholdersInCode(
  code: string,
  mapping: { [key: string]: string | number | boolean | { [key: string]: any } }
): string {
  const regexStr = Object.keys(mapping)
    .map((key) => `\\$\{${key}}`)
    .join("|");

  const regex = new RegExp(regexStr, "gi");

  return code.replace(regex, (matched) => {
    const key = matched.slice(2, -1);
    if (mapping.hasOwnProperty(key)) {
      return mapping[key]?.toString() || "";
    } else {
      return matched;
    }
  });
}
```

### 7.2 动态正则构建 —— 只替换 mapping 中存在的 key

不是用通用的 `/\$\{[^}]+\}/g` 匹配所有 `${...}`，而是**根据 mapping 中的 key 动态构建正则**。

示例：如果 `mapping = { resourceId: "abc123", name: "myapp" }`，则：
```
regexStr = "\$\{resourceId}|\$\{name}"
regex    = /\$\{resourceId\}|\$\{name}/gi
```

**关键结果**：mapping 里没有的 key（如 `${DB_PASSWORD}`）根本不会被正则匹配到，原样保留。这比"先匹配所有，再判断有没有"更高效，也更语义明确。

### 7.3 替换逻辑的边界条件

| mapping[key] 的值 | 替换结果 | 说明 |
|-------------------|---------|------|
| `"abc"` | `"abc"` | 正常字符串 |
| `123` | `"123"` | 数字被 `toString()` 转为字符串 |
| `true` | `"true"` | 布尔值被 `toString()` 转为字符串 |
| `false` | `"false"` | `false?.toString()` 返回 `"false"`，且 `"false"` 是 truthy，不走 `|| ""` |
| `0` | `"0"` | `0.toString()` 返回 `"0"`，且 `"0"` 是 truthy，正常返回 |
| `""`（空字符串） | `""` | `""?.toString()` 返回 `""`，`"" || ""` 仍是 `""` |
| `null` | `""` | `null?.toString()` 返回 `undefined`，`undefined || ""` 返回 `""` |
| `undefined` | `""` | 同上 |
| `{ nested: "obj" }` | `"[object Object]"` | 对象被 `toString()` 序列化为 `"[object Object]"`，可能不是预期结果 |

真正会被意外替换为空字符串的只有 `null` 和 `undefined`——这恰好是合理的：配置项不存在就留空。

### 7.4 数据源：appInfo.settings

在 `.env` 生成中，mapping 来自：

```typescript
// packages/data-service-generator/src/server/create-dotenv.ts#L39
const serviceSettingsDic: { [key: string]: any } = appInfo.settings;
```

`appInfo.settings` 的类型是 `ServiceSettings`，定义于 `libs/util/code-gen-types/src/code-gen-types.ts#L33-L43`。

实际中典型 key：`resourceId`（用于 `COMPOSE_PROJECT_NAME=amp_${resourceId}`），以及 `name`、`version`、各插件注入的占位符。

### 7.5 仅在 .env 生成中使用

当前代码库中，`replacePlaceholdersInCode` 只被两处调用：
- `packages/data-service-generator/src/server/create-dotenv.ts#L43` — Server 端 `.env`
- `packages/data-service-generator/src/admin/create-dotenv.ts#L54` — Admin UI 端 `.env`

代码文件（`.ts`、`.tsx`）不经过占位符替换，动态内容通过 AST 操作直接拼接。职责划分：
- **配置文件**（`.env`）：用占位符替换，简单直接
- **代码文件**：用 AST 操作，保证语法正确、类型安全

---

## 8. 外部 Provider 引用边界

### 8.1 分层抽象架构

```
┌──────────────────────────────────────────────────────────────────┐
│  消费层（五类消费者，见第 2 章）                                  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  NestJS DI 容器内                                          │    │
│  │    1. SecretsManagerService.getSecret(...)                 │    │
│  │    2. ConfigService.get("XXX")                             │    │
│  │    3. process.env.OPENAI_API_KEY                           │    │
│  │       （OpenaiService 虽在 DI 内但绕过注入直读）           │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  NestJS DI 容器外（无法受益于下方的抽象层）                │    │
│  │    3. process.env.PORT        （main.ts，DI 启动前）       │    │
│  │    4. env("DB_URL")            （Prisma schema DSL）      │    │
│  │    5. dotenv.config() + process.env.BCRYPT_SALT           │    │
│  │       （独立 seed 脚本）                                    │    │
│  └──────────────────────────────────────────────────────────┘    │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Factory 层：jwtSecretFactory / JwtModule.registerAsync  │
│  仅第一类消费者经过此层                                   │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Service 层：SecretsManagerService（空子类，推荐扩展点） │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Base 层：SecretsManagerServiceBase implements ISecretsManager
│  依赖 NestJS ConfigService                                │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  配置源：ConfigModule.forRoot({ isGlobal: true })        │
│  默认：dotenv → process.env                               │
│  ★ 可替换：自定义 load 函数接入外部 provider              │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  进程环境：process.env (Node.js)                          │
│  第三、四、五类消费者直接访问                              │
└──────────────────────────────────────────────────────────┘
```

### 8.2 接口契约

`packages/data-service-generator/src/server/secrets-manager/static/base/secretsManager.service.base.template.ts#L4-L6`：

```typescript
export interface ISecretsManager {
  getSecret: (key: EnumSecretsNameKey) => Promise<any | null>;
}
```

契约非常精简：只承诺"给我枚举中的某个 key，我 Promise 你一个值或 null"。

### 8.3 基类实现

```typescript
export class SecretsManagerServiceBase implements ISecretsManager {
  constructor(protected readonly configService: ConfigService) {}
  async getSecret<T>(key: EnumSecretsNameKey): Promise<T | null> {
    const value = this.configService.get(key.toString());
    return value ? value : null;
  }
}
```

`protected readonly configService` 使用 `protected`，暗示子类可以访问——但推荐的扩展方式是完全重写 `getSecret` 的实现。

### 8.4 空子类 —— 预留的扩展点

`packages/data-service-generator/src/server/secrets-manager/static/secretsManager.service.template.ts` 是完全空的子类：

```typescript
@Injectable()
export class SecretsManagerService extends SecretsManagerServiceBase {
  constructor(protected readonly configService: ConfigService) {
    super(configService);
  }
}
```

当需要接入外部 secrets provider（如 AWS Secrets Manager）时，插件可以：
1. 通过 `CreateServerSecretsManager` 的 after 钩子替换整个 `secretsManager.service.ts`
2. 子类重写 `getSecret`，改为调用 AWS SDK，同时保留回退到 `ConfigService` 的逻辑
3. `SecretsManagerServiceBase` 和 `ISecretsManager` 保持不变

这种"基类稳定 + 子类可替换"是模板方法模式的变体。

### 8.5 ConfigModule —— 最深的替换边界

DSG 在 `packages/data-service-generator/src/server/app-module/create-app-module.ts#L72-L78` 中通过 AST 生成：

```typescript
const importModules = [
  ...nestModulesIds,
  callExpression`${CONFIG_MODULE_ID}.forRoot({ isGlobal: true })`,
  // ...
];
```

插件可以通过 `CreateServerAppModule` 的 after 钩子修改配置：

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  load: [customAwsSecretsLoader],  // 自定义加载器
  cache: true,
})
```

但**除非有集中式密钥管理需求，推荐优先扩展 `SecretsManagerService` 子类**：
- 改动范围小，只影响 secrets 读取
- 类型安全（受 `EnumSecretsNameKey` 约束）
- 不影响普通配置的读取路径

### 8.6 边界一览表

| 层级 | 接口 | 是否可替换 | 推荐替换方式 |
|------|------|-----------|------------|
| Secrets 枚举定义 | `EnumSecretsNameKey` | ✅ | `CreateServerSecretsManager` before 钩子追加 |
| Secrets Manager 接口 | `ISecretsManager` | ⚠️ 不推荐 | 改变契约会破坏所有消费端 |
| Secrets Manager 基类 | `SecretsManagerServiceBase` | ⚠️ 谨慎 | 一般无需改动 |
| Secrets Manager 实现 | `SecretsManagerService` | ✅ 推荐 | after 钩子替换整个文件，重写 `getSecret` |
| Factory Provider | `jwtSecretFactory` 等 | ✅ | 插件追加自己的 Factory |
| ConfigModule 配置 | `ConfigModule.forRoot(...)` | ✅ | `CreateServerAppModule` after 钩子修改 AST |
| .env 变量列表 | `CreateServerDotEnvParams.envVariables` | ✅ | before（追加变量）或 after（修改文件文本） |
| Prisma `env("DB_URL")` | Prisma DSL | ⚠️ 受限 | 可修改变量名，但 Prisma 必须用它自己的 `env()` |
| Seed 脚本读取逻辑 | `scripts/seed.ts` | ✅ | `CreateSeed` before/after 钩子修改模板 |

---

## 9. 端到端流程示例：JWT_SECRET_KEY 从构建到运行

```
┌─────────────────────────────────────────────────────────────┐
│                        构建阶段                              │
└─────────────────────────────────────────────────────────────┘
  │
  ├─ [DSG 启动] createServer() 被调用
  │    见 packages/data-service-generator/src/server/create-server.ts#L38
  │
  ├─ [createSecretsManager] 传入 { secretsNameKey: [] }
  │    见 #L88-L90
  │
  ├─ [pluginWrapper before 管道] Auth 插件追加：
  │    eventParams.secretsNameKey.push({
  │      name: "JwtSecretKey", key: "JWT_SECRET_KEY"
  │    })
  │
  ├─ [createSecretsManagerInternal] 生成 EnumSecretsNameKey 枚举
  │    同时复制静态模板（base/service/module）
  │
  ├─ [pluginWrapper after 管道]
  │    after 钩子收到原始 args（但对象引用已被 before 修改）
  │
  ├─ [pluginWrapper upsert]
  │    所有生成的文件 upsert 到 context.modules，供后续事件访问
  │
  ├─ [createDotEnvModule] 传入 ENV_VARIABLES
  │
  ├─ [pluginWrapper before 管道]
  │    同插件追加 { JWT_SECRET_KEY: "Change_ME!!!" }, { JWT_EXPIRATION: "2d" }
  │
  ├─ [createDotEnvModuleInternal]
  │    去重 → 排序 → 占位符替换
  │    输出 server/.env（含 JWT_SECRET_KEY、BCRYPT_SALT=10、DB_URL 等）
  │
  ├─ [createPrismaSchemaModule]
  │    从 constants.ts 硬编码输出 prisma/schema.prisma
  │    datasource db { url = env("DB_URL") }   ★ 不经过 EnumSecretsNameKey
  │
  ├─ [createSeed]
  │    从 seed.template.ts 生成 scripts/seed.ts
  │    dotenv.config() + process.env.BCRYPT_SALT   ★ 第五类消费者
  │
  ├─ [createServerInternal 末尾] mergeMany 收集所有子模块返回值
  │
  ├─ [create-data-service.ts] modules.merge(createServer()) 合并到最终输出
  │
  └─ 所有文件写入磁盘

┌─────────────────────────────────────────────────────────────┐
│                        运行阶段                              │
└─────────────────────────────────────────────────────────────┘
  │
  ├─ [Node 启动] main.ts 执行
  │    const { PORT = 3000 } = process.env;    // 第三类消费者
  │
  ├─ [NestJS 启动] AppModule 初始化
  │    ConfigModule.forRoot() 加载 .env → process.env
  │
  ├─ [AuthModule 初始化]
  │    导入 SecretsManagerModule → SecretsManagerService 可用
  │
  ├─ [jwtSecretFactory 调用]（第一类消费者）
  │    └─► SecretsManagerService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey)
  │         └─► ConfigService.get("JWT_SECRET_KEY")
  │              └─► process.env.JWT_SECRET_KEY
  │
  ├─ [PasswordService 构造]（第二类消费者）
  │    └─► ConfigService.get("BCRYPT_SALT")
  │         └─► process.env.BCRYPT_SALT = "10"
  │
  ├─ [JwtModule.registerAsync]
  │    第一类读 JWT_SECRET_KEY，第二类读 JWT_EXPIRATION
  │
  ├─ [Prisma Client 使用]（第四类消费者）
  │    Prisma 在内部自己读取 env("DB_URL") → process.env.DB_URL
  │
  ├─ [seed.ts 手动执行时]（第五类消费者）
  │    dotenv.config() → process.env.BCRYPT_SALT
  │
  ├─ [OpenaiService 使用时]（第三类消费者）
  │    new OpenAI({ apiKey: process.env.OPENAI_API_KEY })
  │
  └─ [请求到达] JwtStrategy 验证签名
```

---

## 10. 关键设计决策解析

### 10.1 为什么 `getSecret` 返回 `Promise<T | null>` 而不是 `T`？

虽然默认实现只是同步调用 `ConfigService.get()`，但接口强制异步签名：
1. **为外部 provider 预留空间**：AWS Secrets Manager、HashiCorp Vault 等都需要异步网络调用
2. **`null` 而非异常**：把"必须/可选"的语义交给调用方决定

### 10.2 为什么区分 `EnumSecretsNameKey` 与直接字符串常量？

- **类型安全**：`getSecret("JWT_SECRET_KEEY")` 这种拼写错误在编译期就被拦截
- **可发现性**：IDE 自动补全会列出项目中所有合法 secrets
- **可重构**：修改环境变量名只需改枚举值，消费端代码引用的枚举成员名可以保持稳定

### 10.3 为什么用 Factory Provider 间接注入，而不是让 JwtStrategy 直接依赖 SecretsManagerService？

1. **依赖倒置**：`JwtStrategy`（高层策略）不依赖 `SecretsManagerService`（低层实现细节），只依赖一个字符串值
2. **可测试性**：单元测试只需 `{ provide: JWT_SECRET_KEY_PROVIDER_NAME, useValue: "test" }`
3. **关注点分离**：Factory 层集中处理"从哪里取、取不到怎么办"的容错逻辑

### 10.4 为什么 `createSecretsManager` 初始传入空数组？

Amplication 插件化架构的核心原则：**DSG 核心不做任何业务假设**。JWT、数据库、第三方集成等都作为独立插件存在。如果 DSG 硬编码了 JWT_SECRET_KEY，不使用 JWT 的项目就会有多余的枚举成员。

核心是机制，插件是内容。

### 10.5 为什么 .env 变量排序 + 去重？

- **排序**：保证多次构建输出稳定，避免无意义的 Git diff
- **去重**：多个插件可能声明同一个变量，用 Map 去重保证后注册的插件覆盖先注册的

### 10.6 为什么 after 钩子收到原始 args 而非 updatedEventParams？

设计意图：
- `eventParams` 通常是对象引用，before 钩子的就地修改（`.push()`）会反映到原始对象上，after 钩子实际上能看到绝大多数修改
- 避免了"before 插件返回的新对象里包含大量中间态字段，after 钩子意外依赖"的耦合

潜在陷阱：如果某个 before 插件返回了全新的对象，after 钩子看不到新增字段。插件约定：before 钩子应就地修改参数。

### 10.7 为什么 DB_URL 走 Prisma env() 而不是 SecretsManagerService？

1. **技术边界不可跨越**：Prisma schema 是 DSL，不是 TypeScript，无法注入 NestJS 服务
2. **时机不可调和**：`prisma generate` 和 `prisma migrate` 在 NestJS 启动前就需要 DB_URL
3. **Prisma 生态约定**：Prisma 官方文档全部使用 `env("DATABASE_URL")`

### 10.8 为什么 BCRYPT_SALT 不走密钥链路？

1. **语义上不是密钥**：它是 bcrypt 的成本因子（rounds），公开的行业标准推荐值，不需要保密
2. **跨场景读取需求**：NestJS 服务和独立 seed 脚本都需要读取，seed 脚本无法走 DI 容器
3. **DSG 核心内置**：默认值 `10` 是公开默认配置，和 `PORT=3000` 同级别

### 10.9 为什么 OPENAI_API_KEY 走 process.env 直读？

1. **非 DSG 生成代码**：`OpenaiService` 是 `gpt-gateway` 项目的自定义扩展（不在 DSG 生成的 `providers/secrets/` 范围内，也不在 TemplateModule 的 DSG 模板里），没有通过插件注册进 `EnumSecretsNameKey` 枚举。

2. **注册路径特殊**：不是通过 `OpenAIModule` 接入（`OpenAIModule` 存在但完全未被 import，是僵尸模块），而是被直接塞进了 `TemplateModule.providers` 数组。而 `TemplateModule` 本身没有 import `SecretsManagerModule`，导致即使 OpenaiService 想注入也拿不到。

3. **服务实现层面绕过注入**：`OpenaiService` 甚至没有 constructor——没有任何依赖注入入口。虽然它被 `@Injectable()` 装饰且注册在 DI 容器内，但内部完全靠 `process.env` 硬编码读取。

4. **设计疏漏**：从安全性角度它应该走 SecretsManagerService。需要修复的话必须同时做四件事：插件注册枚举 → TemplateModule 导入 SecretsManagerModule → OpenaiService 增加 constructor 注入 → 把 SDK 创建逻辑从方法体移到构造函数或 async Factory。
