# Secrets 与环境变量传递流程深度分析

> 路径引用说明：本文档所有代码引用均使用仓库根目录的相对路径，仓库根为 `50-amplication/`。

---

## 1. 架构总览

整个系统在两个层面上处理 secrets 与环境变量，形成"构建时生成 → 运行时注入"的完整闭环：

| 层面 | 职责 | 关键目录 |
|------|------|---------|
| **代码生成层 (DSG - Data Service Generator)** | 根据资源配置和插件声明，动态生成 secrets 枚举、.env 文件、SecretsManager 模块骨架、Auth 工厂代码 | `packages/data-service-generator/` |
| **运行时层 (Generated App)** | 应用启动时由 NestJS 加载环境变量，通过 DI 容器把 secrets 注入到各业务模块（JWT、DB、消息队列等） | `packages/gpt-gateway/`（作为生成代码的参考实例） |

运行时层的代码结构与 DSG 输出完全一致——`gpt-gateway` 本质上就是 DSG 生成出的一个完整应用实例，可作为阅读模板。

---

## 2. 密钥作用域与消费者划分 (Secrets Scope)

### 2.1 核心数据结构

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

### 2.2 动态枚举生成

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

### 2.3 空数组起步 —— 完全开放的扩展点

关键细节：在 `packages/data-service-generator/src/server/create-server.ts#L87-L90` 中，DSG 对 SecretsManager 的初始调用传入**空数组**：

```typescript
await context.logger.info("Creating SecretsManager...");
const secretsManagerModule = await createSecretsManager({
  secretsNameKey: [],   // ★ 初始为空，完全靠插件 before 钩子追加
});
```

这意味着：**默认情况下 `EnumSecretsNameKey` 会是一个空枚举**。所有具体的密钥声明（包括 JWT_SECRET_KEY、DB 密码等）都必须由插件通过 `CreateServerSecretsManager` 事件的 `before` 钩子注入。这是"核心框架无业务假设"的设计体现。

### 2.4 运行时消费者的三级划分

通过追踪 `packages/gpt-gateway` 中的所有配置读取调用，可以明确划分出三种消费者：

#### 第一级：通过 SecretsManagerService 读取（真正的密钥）

这类消费者只读取被纳入 `EnumSecretsNameKey` 的值，走完整的 `SecretsManagerService.getSecret<T>()` 链路。

| 消费者 | 读取的密钥 | 代码位置 |
|--------|----------|---------|
| `jwtSecretFactory` | `JWT_SECRET_KEY` | `packages/gpt-gateway/src/auth/jwt/jwtSecretFactory.ts#L1-L19` |
| `AuthModule` 中 `JwtModule.registerAsync` | `JWT_SECRET_KEY` | `packages/gpt-gateway/src/auth/auth.module.ts#L24-L44` |

典型模式（`jwtSecretFactory`）：

```typescript
useFactory: async (secretsService: SecretsManagerService): Promise<string> => {
  const secret = await secretsService.getSecret<string>(
    EnumSecretsNameKey.JwtSecretKey
  );
  if (secret) return secret;
  throw new Error("jwtSecretFactory missing secret");
},
inject: [SecretsManagerService],
```

**特征**：
- 依赖注入 `SecretsManagerService`
- 使用 `EnumSecretsNameKey.XXX` 作为类型安全的 key
- 异步调用（即使底层是同步的）
- 对"必须存在"的密钥做 null → 抛错处理

#### 第二级：直接通过 ConfigService.get() 读取（普通配置，非密钥）

这类消费者不经过 SecretsManager，直接注入 NestJS `ConfigService` 读取。它们读取的通常是：
- 非敏感配置（端口号、开关、路径）
- 数值/布尔类型的参数（需要 `=== "true"` 转换的）
- 非核心模块的配置（Kafka、静态文件服务等）

| 消费者 | 读取的变量 | 代码位置 |
|--------|----------|---------|
| `generateKafkaClientOptions` | `KAFKA_BROKERS`, `KAFKA_ENABLE_SSL`, `KAFKA_CLIENT_ID`, `KAFKA_GROUP_ID` | `packages/gpt-gateway/src/kafka/generateKafkaClientOptions.ts#L7-L22` |
| `PasswordService` 构造函数 | `BCRYPT_SALT` | `packages/gpt-gateway/src/auth/password.service.ts#L20-L23` |
| `AuthModule` 中 `JwtModule.registerAsync` | `JWT_EXPIRATION` | `packages/gpt-gateway/src/auth/auth.module.ts#L32` |
| `ServeStaticOptionsService` | `SERVE_STATIC_ROOT_PATH` | `packages/gpt-gateway/src/serveStaticOptions.service.ts#L25-L28` |
| `AppModule` 中 GraphQL 配置 | `GRAPHQL_SCHEMA_DEST`, `GRAPHQL_DEBUG`, `GRAPHQL_PLAYGROUND_ENABLED`, `GRAPHQL_INTROSPECTION_ENABLED` | `packages/gpt-gateway/src/app.module.ts#L50-L56` |

典型模式（Kafka）：

```typescript
const kafkaBrokersString = configService.get("KAFKA_BROKERS");
const kafkaEnableSSL = configService.get("KAFKA_ENABLE_SSL") === "true";
if (!kafkaBrokersString) {
  throw new Error("KAFKA_BROKERS environment variable must be defined");
}
```

**特征**：
- 依赖注入 `ConfigService`
- 使用裸字符串作为 key（如 `"KAFKA_BROKERS"`），无类型安全
- 同步调用
- **不经过 `EnumSecretsNameKey`**，因此这些变量也不会出现在 .env 的 secrets 枚举中

**重要的边界划分规则**（在 `auth.module.ts#L24-L44` 中同时可见两种方式）：

```typescript
useFactory: async (secretsService, configService) => {
  // 真正的密钥 → 走 SecretsManagerService
  const secret = await secretsService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey);
  // 普通配置 → 走 ConfigService 直读
  const expiresIn = configService.get(JWT_EXPIRATION);
  ...
},
inject: [SecretsManagerService, ConfigService],
```

JWT 过期时间（`JWT_EXPIRATION`）虽然和 JWT 有关，但它只是一个时间长度字符串，不属于敏感密钥，所以直接走 ConfigService。两者读取方式在消费端被明确区分，避免了"所有配置都当作 secrets"的过度设计。

#### 第三级：绕过 NestJS 容器，直接读 process.env

这类读取发生在 NestJS 容器初始化**之前**，无法依赖注入，只能直接访问 Node.js 的 `process.env`。

当前仅有一处：

```typescript
// packages/gpt-gateway/src/main.ts#L19
const { PORT = 3000 } = process.env;
```

`main.ts` 是应用入口，`NestFactory.create(AppModule)` 还没执行时就需要知道端口号，因此必须绕过 DI 容器。这是唯一的例外。

### 2.5 作用域的三层边界

**第一层：枚举定义层 —— 编译期白名单**

`EnumSecretsNameKey` 是类型安全的"门禁"。只有被纳入枚举的 key 才能被 `SecretsManagerService.getSecret<T>()` 合法读取。未声明的 key 在 TypeScript 编译阶段就会报错，避免拼写错误或越权访问。

注意：这个门禁只对第一级消费者（走 SecretsManagerService 的）有效。第二级、第三级消费者直接用裸字符串读 ConfigService/process.env，不受枚举约束。

**第二层：Module 导出层 —— NestJS DI 容器边界**

SecretsManager 通过独立的 Module 封装，运行时参考 `packages/gpt-gateway/src/providers/secrets/secretsManager.module.ts`：

```typescript
@Module({
  providers: [SecretsManagerService],
  exports: [SecretsManagerService],
})
export class SecretsManagerModule {}
```

只有显式 `imports: [SecretsManagerModule]` 的业务模块，其 provider/factory 才能注入 `SecretsManagerService`。典型如 `packages/gpt-gateway/src/auth/auth.module.ts#L21` 中 `AuthModule` 的导入声明。

而 `ConfigModule.forRoot({ isGlobal: true })` 是全局模块，任何模块无需显式 import 就能注入 `ConfigService`。这也是为什么第二级消费者（普通配置）随处可见，而第一级消费者（密钥）必须在特定模块内使用。

**第三层：运行时取值层 —— 环境变量边界**

读取的最底层实现位于 `packages/data-service-generator/src/server/secrets-manager/static/base/secretsManager.service.base.template.ts#L8-L17`（DSG 模板），运行时实例见 `packages/gpt-gateway/src/providers/secrets/base/secretsManager.service.base.ts`：

```typescript
export class SecretsManagerServiceBase implements ISecretsManager {
  constructor(protected readonly configService: ConfigService) {}
  async getSecret<T>(key: EnumSecretsNameKey): Promise<T | null> {
    const value = this.configService.get(key.toString());
    if (value) {
      return value;
    }
    return null;
  }
}
```

两个设计细节值得注意：
1. **返回 `Promise<T | null>`**：虽然默认的 `ConfigService.get` 是同步的，但接口强制异步。这是为外部 provider（AWS Secrets Manager、HashiCorp Vault 等需要网络 IO 的实现）预留扩展空间，消费端代码无需改动。
2. **返回 `null` 而非抛异常**：把"secret 是否必须存在"的判断权交给调用方。例如 JWT 签名密钥是必须的，`jwtSecretFactory` 会在拿到 `null` 时抛错；而某些可选特性的密钥则可以接受 `null` 并走降级逻辑。

---

## 3. 构建上下文传递机制 (Build Context Flow)

### 3.1 端到端构建管线

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

### 3.2 DsgContext —— 单例全局上下文

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
  // ... 更多字段

  private static instance: DsgContext;
  public static get getInstance(): DsgContext {
    return this.instance || (this.instance = new this());
  }
  private constructor() { ... }
}
```

与 secrets/env 相关的关键字段：
- `appInfo.settings`（类型：`ServiceSettings`）：资源级配置字典，是 `.env` 中 `${resourceId}` 等占位符替换的数据源
- `plugins`：所有插件钩子的注册表，结构见 `PluginMap` 类型
- `serverDirectories.baseDirectory` / `clientDirectories.baseDirectory`：决定 `.env` 文件的输出位置
- `modules`：插件跨事件共享已生成文件的渠道（详见 3.5 节）

### 3.3 插件注册 —— 事件钩子的装配

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

      // before 和 after 必须是函数才会被收集（用 Object.prototype.toString 判定）
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

最终形成的 `PluginMap` 结构（定义于 `libs/util/code-gen-types/src/plugins.types.ts#L70-L75`）：

```typescript
export type PluginMap = {
  [K in EventNames]?: {
    before?: PluginBeforeEvent<EventParams>[];
    after?: PluginAfterEvent<EventParams>[];
  };
};
```

### 3.4 pluginWrapper —— 洋葱模型的执行引擎（完整细节）

`packages/data-service-generator/src/plugin-wrapper.ts#L59-L117` 是每一个代码生成函数的通用包装器。逐段拆解如下：

#### 3.4.1 入口与控制标志重置

```typescript
const pluginWrapper: PluginWrapper = async (
  func,
  event,
  args
): Promise<ModuleMap> => {
  const context = DsgContext.getInstance;

  try {
    // 每次执行前重置控制标志
    context.utils.skipDefaultBehavior = false;
    context.utils.abort = false;
```

每次事件触发前都会把 `skipDefaultBehavior` 和 `abort` 重置为 `false`。这意味着一个插件在 `CreateServerSecretsManager` 的 before 钩子里设置的 `skipDefaultBehavior = true`，只会影响当前事件，不会"渗漏"到后续的 `CreateServerDotEnv` 等事件。

#### 3.4.2 无插件分支 —— 快路径

```typescript
    if (!context.plugins.hasOwnProperty(event)) {
      return await func(args);   // ★ 直接返回，跳过所有后续步骤
    }
```

当某个事件没有任何插件注册钩子时，会走这个快路径。**它跳过的内容包括：**

1. 不执行 before 管道（当然也没有插件需要执行）
2. 不经过 `defaultBehavior()` 包装 —— 意味着即使某插件（当然没有插件）想设置 `skipDefaultBehavior` 也不会起作用
3. 不执行 after 管道
4. **不执行 upsert 到 `context.modules`**（见 3.5 节）

但这不会造成模块丢失，因为：
- `createServerInternal()` 末尾（`create-server.ts#L144-L167`）会用 `moduleMap.mergeMany([...所有子函数返回值...])` 手动收集
- `create-data-service.ts#L75` 再 `await modules.merge(await createServer())` 把 Server 端全部产物合并到最终输出

无插件分支是性能优化 + 逻辑简化：绝大多数事件在无插件时不需要进入复杂管道。

#### 3.4.3 before 管道

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

**关键机制**：
- 使用 `reduce` + `await` 实现串行管道，前一个插件的返回值作为下一个插件的输入
- `eventParams`（如 `{ secretsNameKey: [] }`）是**可就地修改的对象引用**，插件也可以选择返回新对象
- 最终输出 `updatedEventParams` 只传给 `defaultBehavior`（见下一节）

#### 3.4.4 默认行为包装

```typescript
const defaultBehavior = async (
  context: DsgContext,
  func: (...args: any) => any,
  beforeFuncResults: any
): Promise<ModuleMap> => {
  if (context.utils.skipDefaultBehavior)
    return new ModuleMap(DsgContext.getInstance.logger);  // 跳过默认行为，返回空 Map

  return util.types.isAsyncFunction(func)
    ? await func(beforeFuncResults)
    : func(beforeFuncResults);
};
```

这里接收的是 `updatedEventParams`（经过 before 管道修改后的版本），而不是原始的 `args`。插件如果在 before 钩子里设置了 `skipDefaultBehavior = true`，默认行为就会被跳过，返回一个空的 `ModuleMap`。

#### 3.4.5 after 管道 —— 关键细节：收到的是原始 args

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

**这是一个极易被忽略的细节：after 钩子收到的 `eventParams` 是**原始的 `args`**，不是 before 管道输出的 `updatedEventParams`**。

实践中的影响（以 `CreateServerSecretsManager` 为例）：
- before 钩子向 `eventParams.secretsNameKey[]` push 了 `{ name: "JwtSecretKey", ... }`
- 因为 `eventParams` 是对象引用，`push` 操作会修改原始对象
- 所以 after 钩子拿到的 `eventParams.secretsNameKey` **仍然包含** before 插件追加的内容
- 但如果某个 before 插件选择返回了**新对象**（`return { ...eventParams, extra: "xxx" }`）而不是就地修改，则 after 钩子**看不到**那些新增字段

**推荐实践**：before 钩子应就地修改 `eventParams`，不要返回新对象，这样 after 钩子才能观察到完整的修改。

after 管道的模块流转：`defaultBehaviorModules` → 第一个 after 插件的 `res` → 该插件返回值 → 第二个 after 插件的 `res` → ... → `finalModules`。每个 after 插件都可以自由修改或完全替换 ModuleMap 内容。

#### 3.4.6 Upsert 到 context.modules —— 插件间共享文件

```typescript
    // Upsert all the final modules into the context.modules
    // ModuleMap.merge is not used because it would log a warning for each module
    for (const module of finalModules.modules()) {
      context.modules.replace(module, module);
    }

    return finalModules;
```

**这一步只有在"有插件注册"的分支才会执行**（无插件分支 L70-L72 已经 return 了）。

它的目的是**让后续事件的插件可以访问当前事件生成的文件**。例如：
1. `CreateServerSecretsManager` 事件生成了 `secretsNameKey.enum.ts`
2. Upsert 到 `context.modules`
3. 后续的 `CreateServerAuth` 事件的 after 钩子可以通过 `context.modules` 找到 `secretsNameKey.enum.ts`，读取枚举定义来生成正确的 import 语句

**双重保障的输出链路**：
- 路径 A（有插件时）：`pluginWrapper` upsert 到 `context.modules` + 函数返回值被 `mergeMany` 收集
- 路径 B（无插件时）：函数返回值被 `mergeMany` 收集
- 最终：两条路径都汇入 `create-data-service.ts#L75` 的 `modules.merge(createServer())`

### 3.5 Secrets 与环境变量的插件注入时机

| 事件 | 钩子阶段 | 插件可做什么 | 对应代码 |
|------|---------|------------|---------|
| `CreateServerSecretsManager` | **before** | 向 `eventParams.secretsNameKey[]` 追加 `{ name, key }`，使其被纳入 `EnumSecretsNameKey` 枚举 | `EventNames` 定义于 `libs/util/code-gen-types/src/plugins.types.ts#L120-L124` |
| `CreateServerDotEnv` | **before** | 向 `eventParams.envVariables` 追加或覆盖变量条目 | — |
| `CreateServerDotEnv` | **after** | 修改已生成的 `.env` 文件内容（如追加行、替换值）；也可通过 `context.modules` 找到其他已生成的文件协同修改 | — |
| `CreateAdminDotEnv` | before / after | 同上，针对 Admin UI 端 `.env` | — |
| `CreateServerAppModule` | after | 修改 `app.module.ts`，如替换 `ConfigModule.forRoot()` 的配置以接入自定义 loader | `packages/data-service-generator/src/server/app-module/create-app-module.ts#L92-L100` |

### 3.6 .env 文件生成详解

#### Server 端

`packages/data-service-generator/src/server/create-dotenv.ts#L28-L48`：

```
输入: envVariables: VariableDictionary = [{ BCRYPT_SALT: "10" }, ...]
  │
  ├─► removeDuplicateKeys()       // Map 去重，后出现的同 key 覆盖先出现的
  ├─► sortAlphabetically()        // 按 key 字母排序，保证多次构建输出稳定、diff 干净
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
- 去重逻辑 `create-dotenv.ts#L59-L67`：使用 `Map` 按 key 去重，后入的覆盖先入的。这意味着**插件的 before 钩子注入的变量会覆盖默认 `ENV_VARIABLES` 中的同名变量**，因为 before 管道修改的是传给 `createDotEnvModuleInternal` 的参数
- 占位符替换引擎（详见第 7 节完整分析）

#### Admin 端

`packages/data-service-generator/src/admin/create-dotenv.ts#L33-L60` 与 Server 端的关键区别：它有一个模板文件作为变量的基础来源。

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

变量提取由 `packages/data-service-generator/src/utils/dotenv.ts#L21-L33` 完成，按行用 `=` 分割：

```typescript
function extractVariablesFromCode(code: string): VariableDictionary {
  const arr: VariableDictionary = [];
  code.split("\n").forEach((line) => {
    const content = line.split("=");
    if (!content || content.length != 2) return;  // 非 KEY=VALUE 格式的行被静默丢弃
    const [key, value] = content;
    arr.push({ [key]: value });
  });
  return arr;
}
```

这个解析器很朴素：
- value 中的 `=` 会导致解析错误（如 `CONNECTION_STR=host=localhost;port=5432` 会被解析成 key=`CONNECTION_STR`，value=`host`）
- 不支持注释行（`# comment` 会被解析为 key=`# comment`，value 为空字符串，但因为 `length != 2` 被丢弃，结果是注释行被静默跳过——恰好符合预期）

---

## 4. 外部 Provider 引用边界

### 4.1 分层抽象架构

```
┌──────────────────────────────────────────────────────────┐
│  消费层：AuthModule / JwtStrategy / Prisma / Kafka ...   │
│  依赖方式：@Inject(TOKEN) 拿到字符串值（第一级消费者）    │
│           ConfigService.get(...)（第二级消费者）         │
│           process.env（第三级消费者）                     │
│  不感知 secrets 的获取方式                                │
└────────────────────────────┬─────────────────────────────┘
                             │ NestJS DI
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Factory 层：jwtSecretFactory / JwtModule.registerAsync  │
│  仅第一级消费者经过此层                                   │
│  依赖方式：注入 SecretsManagerService，调用 getSecret()   │
│  决定"必须/可选"语义，处理 null → 抛错或降级             │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Service 层：SecretsManagerService                       │
│  extends SecretsManagerServiceBase                       │
│  ★ 这是推荐的定制化扩展点（空子类）                       │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Base 层：SecretsManagerServiceBase implements ISecretsManager
│  依赖方式：注入 NestJS ConfigService                      │
│  实现最小化：ConfigService.get(key) → T | null            │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  配置源：ConfigModule.forRoot({ isGlobal: true })        │
│  默认：dotenv → process.env                               │
│  ★ 可替换：自定义 load 函数接入 AWS Secrets Manager、     │
│           HashiCorp Vault、K8s Secrets 等                 │
└────────────────────────────┬─────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│  进程环境：process.env (Node.js)                          │
│  第三级消费者直接访问（如 main.ts 的 PORT）                │
└──────────────────────────────────────────────────────────┘
```

### 4.2 接口契约

`packages/data-service-generator/src/server/secrets-manager/static/base/secretsManager.service.base.template.ts#L4-L6` 定义了 `ISecretsManager` 接口：

```typescript
export interface ISecretsManager {
  getSecret: (key: EnumSecretsNameKey) => Promise<any | null>;
}
```

契约非常精简：只承诺"给我枚举中的某个 key，我 Promise 你一个值或 null"。实现方可以自由选择同步读环境变量、异步调远程 API、带缓存、带密钥轮换逻辑等。

### 4.3 基类实现（不可变层）

`SecretsManagerServiceBase` 是 DSG 生成的模板代码，除非用户手动修改生成产物，否则插件不应改动它。它的唯一依赖是 NestJS 的 `ConfigService`：

```typescript
export class SecretsManagerServiceBase implements ISecretsManager {
  constructor(protected readonly configService: ConfigService) {}
  async getSecret<T>(key: EnumSecretsNameKey): Promise<T | null> {
    const value = this.configService.get(key.toString());
    return value ? value : null;
  }
}
```

`protected readonly configService` 使用 `protected` 而非 `private`，暗示子类可以访问它——但推荐的扩展方式是**完全重写 `getSecret` 的实现**而不是复用基类。

### 4.4 空子类 —— 预留的扩展点

`packages/data-service-generator/src/server/secrets-manager/static/secretsManager.service.template.ts` 是一个完全空的子类：

```typescript
@Injectable()
export class SecretsManagerService extends SecretsManagerServiceBase {
  constructor(protected readonly configService: ConfigService) {
    super(configService);
  }
}
```

这是有意为之的设计。当需要接入外部 secrets provider（如 AWS Secrets Manager）时，插件可以：

1. 通过 `CreateServerSecretsManager` 的 after 钩子替换整个 `secretsManager.service.ts` 的内容
2. 子类重写 `getSecret`，改为调用 AWS SDK，同时保留回退到 `ConfigService` 的逻辑
3. `SecretsManagerServiceBase` 和 `ISecretsManager` 保持不变，保证消费端代码稳定

这种"基类稳定 + 子类可替换"的模式是经典的**模板方法模式 (Template Method)** 的变体。

### 4.5 消费端边界 —— JWT 场景完整链路

以 JWT 鉴权为例，从 secrets 读取到业务使用经过了三层解耦。

**步骤 1：Factory Provider 封装"必须存在"语义**

`packages/gpt-gateway/src/auth/jwt/jwtSecretFactory.ts`：

```typescript
export const jwtSecretFactory = {
  provide: JWT_SECRET_KEY_PROVIDER_NAME,   // 字符串 token: "JWT_SECRET_KEY"
  useFactory: async (
    secretsService: SecretsManagerService
  ): Promise<string> => {
    const secret = await secretsService.getSecret<string>(
      EnumSecretsNameKey.JwtSecretKey
    );
    if (secret) return secret;
    throw new Error("jwtSecretFactory missing secret");  // 必须密钥 → null 即失败
  },
  inject: [SecretsManagerService],
};
```

Provider token `JWT_SECRET_KEY_PROVIDER_NAME` 定义于 `packages/gpt-gateway/src/constants.ts#L1`：
```typescript
export const JWT_SECRET_KEY_PROVIDER_NAME = "JWT_SECRET_KEY";
```

**步骤 2：JwtModule 异步注册 —— 第三方模块的适配器**

`packages/gpt-gateway/src/auth/auth.module.ts#L22-L44`：

```typescript
JwtModule.registerAsync({
  imports: [SecretsManagerModule],
  inject: [SecretsManagerService, ConfigService],
  useFactory: async (secretsService, configService) => {
    const secret = await secretsService.getSecret<string>(
      EnumSecretsNameKey.JwtSecretKey
    );
    const expiresIn = configService.get(JWT_EXPIRATION);   // 非 secrets 走 ConfigService 直读
    if (!secret) throw new Error("Didn't get a valid jwt secret");
    if (!expiresIn) throw new Error("Jwt expire in value is not valid");
    return { secret, signOptions: { expiresIn } };
  },
})
```

这里展示了明确的边界划分：
- 真正的 secrets（JWT 签名密钥）通过 `SecretsManagerService` 读取
- 普通配置（JWT 过期时间）直接走 `ConfigService.get()`

**步骤 3：业务 Strategy 只拿值 —— 完全不感知来源**

`packages/gpt-gateway/src/auth/jwt/jwt.strategy.ts#L7-L14`：

```typescript
@Injectable()
export class JwtStrategy extends JwtStrategyBase {
  constructor(
    @Inject(JWT_SECRET_KEY_PROVIDER_NAME) secretOrKey: string,
    protected readonly userService: UserService
  ) {
    super(secretOrKey, userService);
  }
}
```

`JwtStrategy` 的构造参数只是一个 `string`。测试时只需：

```typescript
{ provide: JWT_SECRET_KEY_PROVIDER_NAME, useValue: "test-secret" }
```

即可完成注入，无需 mock 任何外部服务。

基类 `packages/gpt-gateway/src/auth/jwt/base/jwt.strategy.base.ts#L8-L21` 最终把字符串传给 Passport：

```typescript
export class JwtStrategyBase extends PassportStrategy(Strategy) {
  constructor(
    protected readonly secretOrKey: string,
    protected readonly userService: UserService
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey,    // ★ 最终在这里被 Passport 使用
    });
  }
}
```

### 4.6 ConfigModule —— 最深的替换边界

所有配置（无论是否是 secrets）最终都流经 NestJS 的 `ConfigModule`。DSG 在 `packages/data-service-generator/src/server/app-module/create-app-module.ts#L72-L78` 中通过 AST 生成：

```typescript
const importModules = [
  ...nestModulesIds,
  callExpression`${CONFIG_MODULE_ID}.forRoot({ isGlobal: true })`,
  // ...
];
```

生成的模板 `packages/data-service-generator/src/server/app-module/app.module.template.ts#L1-L10` 本身极简：

```typescript
@Module({
  controllers: [],
  imports: MODULES,       // MODULES 是被 AST 动态替换的占位符
  providers: [],
})
export class AppModule {}
```

插件可以通过 `CreateServerAppModule` 的 after 钩子修改配置，例如：

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  load: [customAwsSecretsLoader],  // 自定义加载器，从 AWS 拉取 secrets
  cache: true,
})
```

但**除非有集中式密钥管理需求，推荐优先扩展 `SecretsManagerService` 子类**，因为：
- 改动范围小，只影响 secrets 读取
- 类型安全（受 `EnumSecretsNameKey` 约束）
- 不影响普通配置的读取路径

### 4.7 边界一览表

| 层级 | 接口 | 是否可被插件/用户替换 | 推荐替换方式 | 对应代码 |
|------|------|---------------------|------------|---------|
| Secrets 枚举定义 | `EnumSecretsNameKey` | ✅ | `CreateServerSecretsManager` before 钩子追加 `SecretsNameKey[]` | `packages/data-service-generator/src/server/secrets-manager/create-secrets-manager.ts#L70-L83` |
| Secrets Manager 接口 | `ISecretsManager` | ⚠️ 不推荐 | 改变契约会破坏所有消费端 | `packages/data-service-generator/src/server/secrets-manager/static/base/secretsManager.service.base.template.ts#L4-L6` |
| Secrets Manager 基类 | `SecretsManagerServiceBase` | ⚠️ 谨慎 | 一般无需改动，除非需要增加缓存、轮换等通用能力 | 同上 `#L8-L17` |
| Secrets Manager 实现 | `SecretsManagerService` | ✅ 推荐 | `CreateServerSecretsManager` after 钩子替换整个文件，重写 `getSecret` 接入外部 SDK | `packages/data-service-generator/src/server/secrets-manager/static/secretsManager.service.template.ts#L1-L10` |
| Factory Provider | `jwtSecretFactory` 等 | ✅ | 插件可追加自己的 Factory，或通过 `CreateServerAuth` 钩子修改 AuthModule | `packages/gpt-gateway/src/auth/jwt/jwtSecretFactory.ts` |
| ConfigModule 配置 | `ConfigModule.forRoot(...)` | ✅ | `CreateServerAppModule` after 钩子修改生成的 app.module.ts AST | `packages/data-service-generator/src/server/app-module/create-app-module.ts#L72-L78` |
| .env 变量列表 | `CreateServerDotEnvParams.envVariables` | ✅ | `CreateServerDotEnv` before（追加/覆盖变量）或 after（修改文件文本） | `packages/data-service-generator/src/server/create-dotenv.ts#L28-L48` |

---

## 5. 端到端流程示例：JWT_SECRET_KEY 从构建到运行

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
  ├─ [pluginWrapper 执行 before 管道]
  │    某 Auth 插件的 CreateServerSecretsManager before 钩子：
  │
  │    async (context, eventParams) => {
  │      eventParams.secretsNameKey.push({
  │        name: "JwtSecretKey",
  │        key: "JWT_SECRET_KEY"
  │      });
  │      return eventParams;
  │    }
  │
  ├─ [createSecretsManagerInternal] 动态生成枚举
  │    createTSEnumSecretsNameKey([...]) 输出：
  │      enum EnumSecretsNameKey {
  │        JwtSecretKey = "JWT_SECRET_KEY",
  │      }
  │    同时复制静态模板：
  │      secretsManager.service.base.ts
  │      secretsManager.service.ts（空子类）
  │      secretsManager.module.ts
  │
  ├─ [pluginWrapper after 管道]
  │    after 钩子收到原始 args（但对象引用已被 before 修改，
  │    所以 eventParams.secretsNameKey 仍然包含 JwtSecretKey）
  │
  ├─ [pluginWrapper upsert]
  │    所有生成的文件被 upsert 到 context.modules，
  │    供后续事件（如 CreateServerAuth）的插件访问
  │
  ├─ [createDotEnvModule] 传入 ENV_VARIABLES
  │
  ├─ [pluginWrapper 执行 before 管道]
  │    同插件的 CreateServerDotEnv before 钩子追加：
  │      { JWT_SECRET_KEY: "Change_ME!!!" }
  │      { JWT_EXPIRATION: "2d" }
  │
  ├─ [createDotEnvModuleInternal]
  │    去重 → 排序 → 占位符替换（见第 7 节）
  │    输出 server/.env：
  │      BCRYPT_SALT=10
  │      COMPOSE_PROJECT_NAME=amp_cll5bbdjs093...
  │      JWT_EXPIRATION=2d
  │      JWT_SECRET_KEY=Change_ME!!!
  │      PORT=3000
  │
  ├─ [createServerInternal 末尾] mergeMany 收集所有子模块返回值
  │
  ├─ [create-data-service.ts] modules.merge(createServer()) 合并到最终输出
  │
  ├─ [createAppModule] 扫描所有 *.module.ts 自动加入 imports
  │    SecretsManagerModule 被自动注册到 AppModule.imports
  │    ConfigModule.forRoot({ isGlobal: true }) 被注入
  │
  └─ 所有文件写入磁盘

┌─────────────────────────────────────────────────────────────┐
│                        运行阶段                              │
└─────────────────────────────────────────────────────────────┘
  │
  ├─ [Node 启动] main.ts 执行
  │    const { PORT = 3000 } = process.env;    // 第三级消费者，绕过 DI
  │
  ├─ [NestJS 启动] AppModule 初始化
  │    ConfigModule.forRoot() 自动加载 server/.env → process.env
  │
  ├─ [AuthModule 初始化]
  │    导入 SecretsManagerModule → SecretsManagerService 可用
  │
  ├─ [jwtSecretFactory.useFactory 被调用] （第一级消费者）
  │    └─► SecretsManagerService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey)
  │         └─► SecretsManagerServiceBase.getSecret("JWT_SECRET_KEY")
  │              └─► ConfigService.get("JWT_SECRET_KEY")           （ConfigService）
  │                   └─► 返回 process.env.JWT_SECRET_KEY = "Change_ME!!!"
  │
  ├─ [JwtModule.registerAsync]
  │    第一级消费者读 JWT_SECRET_KEY，第二级消费者读 JWT_EXPIRATION
  │    → JwtModule 内部配置：{ secret: "Change_ME!!!", signOptions: { expiresIn: "2d" } }
  │
  ├─ [JwtStrategy 实例化]
  │    @Inject(JWT_SECRET_KEY_PROVIDER_NAME) → 拿到字符串 "Change_ME!!!"
  │    → Passport Strategy 初始化时将其设为 secretOrKey
  │
  └─ [请求到达]
       Authorization: Bearer <token>
         └─► JwtStrategy 自动使用 secretOrKey 验证签名
```

---

## 6. 关键设计决策解析

### 6.1 为什么 `getSecret` 返回 `Promise<T | null>` 而不是 `T`？

虽然默认实现只是同步调用 `ConfigService.get()`，但接口强制异步签名有两个目的：
1. **为外部 provider 预留空间**：AWS Secrets Manager、HashiCorp Vault、Azure Key Vault 等都需要异步网络调用
2. **`null` 而非异常**：把"必须/可选"的语义交给调用方决定。`jwtSecretFactory` 对 JWT 密钥采取"缺失即失败"策略，而某些可选特性的插件可以选择 `null` 时禁用该特性

### 6.2 为什么区分 `EnumSecretsNameKey` 与直接字符串常量？

- **类型安全**：`getSecret("JWT_SECRET_KEEY")` 这种拼写错误在编译期就被拦截
- **可发现性**：IDE 自动补全会列出项目中所有合法 secrets，不需要翻文档
- **可重构**：修改环境变量名只需改枚举值（或插件注入的 `key` 字段），消费端代码引用的枚举成员名可以保持稳定

### 6.3 为什么用 Factory Provider 间接注入，而不是让 JwtStrategy 直接依赖 SecretsManagerService？

以 `jwtSecretFactory` 为例的三层解耦设计：
1. **依赖倒置**：`JwtStrategy`（高层策略）不依赖 `SecretsManagerService`（低层实现细节），只依赖一个字符串值
2. **可测试性**：单元测试只需 `{ provide: JWT_SECRET_KEY_PROVIDER_NAME, useValue: "test" }`，无需 mock `ConfigService` 或外部 SDK
3. **关注点分离**：Factory 层集中处理"从哪里取、取不到怎么办"的容错逻辑，业务层专注于"拿到值之后怎么用"

### 6.4 为什么 `createSecretsManager` 初始传入空数组，而不是内置 JWT_SECRET_KEY 等常见值？

这是 Amplication 插件化架构的核心原则：**DSG 核心不做任何业务假设**。JWT 鉴权、数据库连接、第三方集成等都作为独立插件存在。如果 DSG 硬编码了 JWT_SECRET_KEY，那么不使用 JWT 的项目（如纯 API 网关）就会有多余的枚举成员，破坏了"只生成你需要的"原则。

插件在 `CreateServerSecretsManager` 的 before 钩子中注入自己需要的 secrets，由 DSG 核心统一合并去重后生成枚举——核心是机制，插件是内容。

### 6.5 为什么 .env 变量排序 + 去重？

- **排序（`sortAlphabetically`）**：保证多次构建输出稳定。如果每次生成的 `.env` 变量顺序不同，Git 会产生无意义的 diff，影响代码评审和变更追踪
- **去重（`removeDuplicateKeys`）**：多个插件可能声明同一个变量（如都需要 `DB_URL`），用 `Map` 去重保证后注册的插件覆盖先注册的，形成明确的优先级

### 6.6 为什么 after 钩子收到原始 args 而非 updatedEventParams？

这是一个需要理解的设计选择，有正反两面：

**正面（设计意图）**：
- `eventParams` 通常是**对象引用**，before 钩子的就地修改（如 `.push()`）会反映到原始对象上，after 钩子实际上能看到绝大多数修改
- 避免了"before 插件返回的新对象里包含大量中间态字段，after 钩子意外依赖"的耦合
- after 钩子的主要职责是**修改 ModuleMap**，不是查看参数

**潜在陷阱**：
- 如果某个 before 插件返回了全新的对象（`return { ...eventParams, extra }`），after 钩子看不到新增的 `extra` 字段
- 因此插件约定：before 钩子应就地修改参数，不要返回新对象

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

第一步不是用通用的 `/\$\{[^}]+\}/g` 匹配所有 `${...}`，而是**根据 mapping 中的 key 动态构建正则**：

```typescript
const regexStr = Object.keys(mapping).map(key => `\\$\{${key}}`).join("|");
```

示例：如果 `mapping = { resourceId: "abc123", name: "myapp" }`，则：
```
regexStr = "\$\{resourceId}|\$\{name}"
regex    = /\$\{resourceId\}|\$\{name}/gi
```

**关键结果**：mapping 里没有的 key（如 `${DB_PASSWORD}`）**根本不会被正则匹配到**，替换函数不会被调用，这些占位符就原样保留在输出中。

这比"先匹配所有，再判断有没有"的实现更高效，也更语义明确：不需要替换的内容连碰都不碰。

### 7.3 替换逻辑的边界条件

替换回调函数：

```typescript
(matched) => {
  const key = matched.slice(2, -1);              // "${resourceId}" → "resourceId"
  if (mapping.hasOwnProperty(key)) {             // 防御式检查（理论上正则已保证）
    return mapping[key]?.toString() || "";       // 值 → 字符串
  } else {
    return matched;                              // 理论上不会走到这分支
  }
}
```

**边界条件清单**：

| mapping[key] 的值 | 替换结果 | 说明 |
|-------------------|---------|------|
| `"abc"` | `"abc"` | 正常字符串 |
| `123` | `"123"` | 数字被 `toString()` 转为字符串 |
| `true` | `"true"` | 布尔值被 `toString()` 转为字符串 |
| `false` | `""`（空字符串） | ⚠️ **陷阱**：`false.toString()` 是 `"true"`？不 —— `false?.toString()` 返回 `"false"`，但 `|| ""` 因为 `"false"` 是 truthy 所以没问题，实际返回 `"false"` |
| `0` | `"0"` | `0.toString()` 返回 `"0"`，且 `"0"` 是 truthy，所以没问题 |
| `""`（空字符串） | `""` | `""?.toString()` 是 `""`，`"" || ""` 还是 `""` |
| `null` | `""`（空字符串） | ⚠️ `null?.toString()` 返回 `undefined`，`undefined || ""` 返回 `""` |
| `undefined` | `""`（空字符串） | ⚠️ 同上 |
| `{ nested: "obj" }` | `"[object Object]"` | ⚠️ 对象被 `toString()` 序列化为 `"[object Object]"`，可能不是预期结果 |

关于 `false` 的修正：代码 `mapping[key]?.toString() || ""` 中，`?.` 只在值为 `null/undefined` 时短路返回 `undefined`。对于 `false`、`0`、`""` 这些 falsy 但非空的值，`?.toString()` 会正常执行：
- `false?.toString()` → `"false"`（truthy）→ 返回 `"false"` ✅
- `(0)?.toString()` → `"0"`（truthy）→ 返回 `"0"` ✅
- `("")?.toString()` → `""`（falsy）→ `"" || ""` → `""` ✅（和原值一致）

所以真正会被意外替换为空字符串的只有 `null` 和 `undefined`——这恰好是合理的：配置项不存在就留空。

### 7.4 数据源：appInfo.settings

在 `.env` 生成中，mapping 来自：

```typescript
// packages/data-service-generator/src/server/create-dotenv.ts#L39
const serviceSettingsDic: { [key: string]: any } = appInfo.settings;
```

`appInfo.settings` 的类型是 `ServiceSettings`，定义于 `libs/util/code-gen-types/src/code-gen-types.ts#L33-L43`：

```typescript
export type AppInfo = {
  name: string;
  description: string;
  version: string;
  id: string;
  url: string;
  settings: ServiceSettings;   // ← 这是 mapping 的来源
  codeGeneratorVersionOptions: models.CodeGeneratorVersionOptionsInput;
  // ...
};
```

**实际中会包含的典型 key**（基于 `${resourceId}` 在默认 `ENV_VARIABLES` 中的使用推断）：
- `resourceId`：资源的唯一标识符，用于 `COMPOSE_PROJECT_NAME=amp_${resourceId}`
- 可能还有 `name`、`version`、`dbConnectionString` 等插件注入的占位符

### 7.5 仅在 .env 生成中使用

当前代码库中，`replacePlaceholdersInCode` 只被两处调用：

| 调用位置 | 被替换的内容 |
|---------|------------|
| `packages/data-service-generator/src/server/create-dotenv.ts#L43` | Server 端 `.env` 文件文本 |
| `packages/data-service-generator/src/admin/create-dotenv.ts#L54` | Admin UI 端 `.env` 文件文本 |

代码文件（`.ts`、`.tsx`、模板文件等）**不经过占位符替换**。代码中的动态内容通过 AST 操作（`recast` + `ast-types` 的 `builders`）直接拼接，而不是字符串替换。

这是一个明确的职责划分：
- **配置文件**（`.env`）：用占位符替换，简单直接
- **代码文件**：用 AST 操作，保证语法正确、类型安全

---

## 8. 消费者一览表（完整清单）

基于 `packages/gpt-gateway` 代码库的实际消费者统计：

### 第一级：SecretsManagerService 消费者

| 消费者 | 读取的枚举值 | 必须/可选 | 代码位置 |
|--------|------------|----------|---------|
| `jwtSecretFactory` | `EnumSecretsNameKey.JwtSecretKey` | 必须 | `packages/gpt-gateway/src/auth/jwt/jwtSecretFactory.ts#L1-L19` |
| `AuthModule` `JwtModule.registerAsync` | `EnumSecretsNameKey.JwtSecretKey` | 必须 | `packages/gpt-gateway/src/auth/auth.module.ts#L29-L31` |

### 第二级：ConfigService.get() 消费者

| 消费者 | 读取的变量 | 必须/可选 | 代码位置 |
|--------|----------|----------|---------|
| `generateKafkaClientOptions` | `KAFKA_BROKERS` | 必须 | `packages/gpt-gateway/src/kafka/generateKafkaClientOptions.ts#L7` |
| 同上 | `KAFKA_ENABLE_SSL` | 可选（默认 `false`） | 同上 `#L8` |
| 同上 | `KAFKA_CLIENT_ID` | 必须 | 同上 `#L9` |
| 同上 | `KAFKA_GROUP_ID` | 必须 | 同上 `#L10` |
| `PasswordService` 构造函数 | `BCRYPT_SALT` | 必须 | `packages/gpt-gateway/src/auth/password.service.ts#L20-L22` |
| `AuthModule` `JwtModule.registerAsync` | `JWT_EXPIRATION` | 必须 | `packages/gpt-gateway/src/auth/auth.module.ts#L32` |
| `ServeStaticOptionsService` | `SERVE_STATIC_ROOT_PATH` | 可选（默认 undefined 则走内置 swagger 路径） | `packages/gpt-gateway/src/serveStaticOptions.service.ts#L26-L28` |
| `AppModule` GraphQL 配置 | `GRAPHQL_SCHEMA_DEST` | 可选（有默认值） | `packages/gpt-gateway/src/app.module.ts#L50` |
| 同上 | `GRAPHQL_DEBUG` | 可选（默认 `false`） | 同上 `#L53` |
| 同上 | `GRAPHQL_PLAYGROUND_ENABLED` | 可选（默认 `false`） | 同上 `#L54` |
| 同上 | `GRAPHQL_INTROSPECTION_ENABLED` | 可选（默认 `false`） | 同上 `#L56` |

### 第三级：process.env 直读

| 消费者 | 读取的变量 | 必须/可选 | 代码位置 |
|--------|----------|----------|---------|
| `main.ts`（NestJS 启动前） | `PORT` | 可选（默认 `3000`） | `packages/gpt-gateway/src/main.ts#L19` |
