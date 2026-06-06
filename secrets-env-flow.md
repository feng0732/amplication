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

## 2. 密钥作用域 (Secrets Scope)

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

### 2.4 作用域的三层边界

**第一层：枚举定义层 —— 编译期白名单**

`EnumSecretsNameKey` 是类型安全的"门禁"。只有被纳入枚举的 key 才能被 `SecretsManagerService.getSecret<T>()` 合法读取。未声明的 key 在 TypeScript 编译阶段就会报错，避免拼写错误或越权访问。

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
            └─► createAdminModules()            // Admin UI 端
                 └─► createDotEnvModule()
```

### 3.2 DsgContext —— 单例全局上下文

`packages/data-service-generator/src/dsg-context.ts#L19-L81` 定义了贯穿整个构建过程的状态容器，采用经典单例模式：

```typescript
class DsgContext implements types.DsgContext {
  public appInfo!: types.AppInfo;
  public entities: types.Entity[] = [];
  public plugins: types.PluginMap = {};     // 事件名 → before/after 钩子数组
  public modules: types.ModuleMap;           // 已生成的文件集合
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
- `appInfo.settings`：资源级配置字典，是 `.env` 中 `${resourceId}` 等占位符替换的数据源
- `plugins`：所有插件钩子的注册表，结构见 `PluginMap` 类型
- `serverDirectories.baseDirectory` / `clientDirectories.baseDirectory`：决定 `.env` 文件的输出位置

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

      // before 和 after 必须是函数才会被收集
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

### 3.4 pluginWrapper —— 洋葱模型的执行引擎

`packages/data-service-generator/src/plugin-wrapper.ts#L59-L117` 是每一个代码生成函数的通用包装器，形成 **before 管道 → 默认行为 → after 管道** 的三层结构：

```typescript
const pluginWrapper = async (func, event, args) => {
  const context = DsgContext.getInstance;

  // 每次执行前重置控制标志
  context.utils.skipDefaultBehavior = false;
  context.utils.abort = false;

  // 无插件注册 → 直接执行默认行为
  if (!context.plugins.hasOwnProperty(event)) {
    return await func(args);
  }

  const beforePlugins = context.plugins[event]?.before || [];
  const afterPlugins = context.plugins[event]?.after || [];

  // 第一阶段：before 管道 —— 插件可修改 eventParams
  // reduce + await 确保按注册顺序串行执行，前一个的输出作为后一个的输入
  const updatedEventParams = beforePlugins
    ? await beforeEventsPipe(...beforePlugins)(context, args)
    : args;

  // 第二阶段：默认行为（可被 skipDefaultBehavior 跳过）
  const defaultBehaviorModules = await defaultBehavior(
    context, func, updatedEventParams
  );

  // 第三阶段：after 管道 —— 插件可修改生成的 ModuleMap
  const finalModules = afterPlugins
    ? await afterEventsPipe(...afterPlugins)(context, args, defaultBehaviorModules)
    : defaultBehaviorModules;

  // ★ 所有产物自动 upsert 到 context.modules，供后续步骤访问
  for (const module of finalModules.modules()) {
    context.modules.replace(module, module);
  }

  return finalModules;
};
```

`beforeEventsPipe` 的实现（同文件 `L17-L23`）展示了参数如何在插件间传递：

```typescript
const beforeEventsPipe = (...fns) => (context, eventParams) =>
  fns.reduce(
    async (res, fn) => fn(context, await res),  // 每个插件收到的是上一个插件返回的 eventParams
    Promise.resolve(eventParams)
  );
```

### 3.5 Secrets 与环境变量的插件注入时机

| 事件 | 钩子阶段 | 插件可做什么 | 对应代码 |
|------|---------|------------|---------|
| `CreateServerSecretsManager` | **before** | 向 `eventParams.secretsNameKey[]` 追加 `{ name, key }`，使其被纳入 `EnumSecretsNameKey` 枚举 | `EventNames` 定义于 `libs/util/code-gen-types/src/plugins.types.ts#L120-L124` |
| `CreateServerDotEnv` | **before** | 向 `eventParams.envVariables` 追加或覆盖变量条目 | — |
| `CreateServerDotEnv` | **after** | 修改已生成的 `.env` 文件内容（如追加行、替换值） | — |
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
- 占位符替换引擎 `packages/data-service-generator/src/utils/text-file-parser.ts#L1-L19`：用正则匹配所有 `${key}`，在 `appInfo.settings` 字典里查找替换值，找不到则保留原样（留给部署时人工填写）
- 去重逻辑 `create-dotenv.ts#L59-L67`：使用 `Map` 按 key 去重，后入的覆盖先入的。这意味着**插件的 before 钩子注入的变量会覆盖默认 `ENV_VARIABLES` 中的同名变量**，因为 before 管道修改的是传给 `createDotEnvModuleInternal` 的参数

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

变量提取由 `packages/data-service-generator/src/utils/dotenv.ts#L21-L33` 完成，按行用 `=` 分割，非常朴素的解析：

```typescript
function extractVariablesFromCode(code: string): VariableDictionary {
  const arr: VariableDictionary = [];
  code.split("\n").forEach((line) => {
    const content = line.split("=");
    if (!content || content.length != 2) return;
    const [key, value] = content;
    arr.push({ [key]: value });
  });
  return arr;
}
```

---

## 4. 外部 Provider 引用边界

### 4.1 分层抽象架构

```
┌──────────────────────────────────────────────────────────┐
│  消费层：AuthModule / JwtStrategy / Prisma / Kafka ...   │
│  依赖方式：@Inject(TOKEN) 拿到字符串值                    │
│  不感知 secrets 的获取方式                                │
└────────────────────────────┬─────────────────────────────┘
                             │ NestJS DI
                             ▼
┌──────────────────────────────────────────────────────────┐
│  Factory 层：jwtSecretFactory / JwtModule.registerAsync  │
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
└──────────────────────────────────────────────────────────┘
```

### 4.2 接口契约

`packages/data-service-generator/src/server/secrets-manager/static/base/secretsManager.service.base.template.ts#L4-L6` 定义了 `ISecretsManager` 接口：

```typescript
export interface ISecretsManager {
  getSecret: (key: EnumSecretsNameKey) => Promise<any | null>;
}
```

契约非常精简：只承诺"给我枚举中的某个 key，我 Promise 你一个值或 null"。实现方可以自由选择同步读环境变量、异步调远程 API、带缓存、带轮换等。

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

这里展示了一个重要的**边界划分规则**：
- 真正的 secrets（如 JWT 签名密钥）通过 `SecretsManagerService` 读取，便于未来替换为外部 provider
- 普通配置（如 JWT 过期时间）直接走 `ConfigService.get()`，不经过 SecretsManager

两者的读取方式在消费端被明确区分，避免了"所有配置都当作 secrets"的过度设计。

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

`JwtStrategy` 的构造参数只是一个 `string`。它既不知道 `SecretsManagerService` 的存在，也不知道 `.env` 文件的存在。测试时只需：

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

这意味着插件可以通过 `CreateServerAppModule` 的 after 钩子，把生成出的：

```typescript
ConfigModule.forRoot({ isGlobal: true })
```

修改为例如：

```typescript
ConfigModule.forRoot({
  isGlobal: true,
  load: [customAwsSecretsLoader],  // 自定义加载器，从 AWS 拉取 secrets
  cache: true,
})
```

这是系统最深层的扩展点——在 `SecretsManagerService` 之下替换配置源。但**除非有特殊需求（如集中式密钥管理），推荐优先扩展 `SecretsManagerService` 子类**，因为：
- 改动范围小，只影响 secrets 读取
- 类型安全（受 `EnumSecretsNameKey` 约束）
- 不影响普通配置的读取路径

### 4.7 边界一览表

| 层级 | 接口 | 是否可被插件/用户替换 | 推荐替换方式 | 对应代码 |
|------|------|---------------------|------------|---------|
| Secrets 枚举定义 | `EnumSecretsNameKey` | ✅ | `CreateServerSecretsManager` before 钩子追加 `SecretsNameKey[]` | `create-secrets-manager.ts#L70-L83` |
| Secrets Manager 接口 | `ISecretsManager` | ⚠️ 不推荐 | 改变契约会破坏所有消费端 | `secretsManager.service.base.template.ts#L4-L6` |
| Secrets Manager 基类 | `SecretsManagerServiceBase` | ⚠️ 谨慎 | 一般无需改动，除非需要增加缓存、轮换等通用能力 | 同上 `#L8-L17` |
| Secrets Manager 实现 | `SecretsManagerService` | ✅ 推荐 | `CreateServerSecretsManager` after 钩子替换整个文件，重写 `getSecret` 接入外部 SDK | `secretsManager.service.template.ts#L1-L10` |
| Factory Provider | `jwtSecretFactory` 等 | ✅ | 插件可追加自己的 Factory，或通过 `CreateServerAuth` 钩子修改 AuthModule | `jwtSecretFactory.ts` |
| ConfigModule 配置 | `ConfigModule.forRoot(...)` | ✅ | `CreateServerAppModule` after 钩子修改生成的 app.module.ts AST | `create-app-module.ts#L72-L78` |
| .env 变量列表 | `CreateServerDotEnvParams.envVariables` | ✅ | `CreateServerDotEnv` before（追加/覆盖变量）或 after（修改文件文本） | `create-dotenv.ts#L28-L48` |

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
  ├─ [createDotEnvModule] 传入 ENV_VARIABLES
  │
  ├─ [pluginWrapper 执行 before 管道]
  │    同插件的 CreateServerDotEnv before 钩子追加：
  │      { JWT_SECRET_KEY: "Change_ME!!!" }
  │      { JWT_EXPIRATION: "2d" }
  │
  ├─ [createDotEnvModuleInternal] 去重 → 排序 → 占位符替换
  │    输出 server/.env：
  │      BCRYPT_SALT=10
  │      COMPOSE_PROJECT_NAME=amp_cll5bbdjs093...
  │      JWT_EXPIRATION=2d
  │      JWT_SECRET_KEY=Change_ME!!!
  │      PORT=3000
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
  ├─ [NestJS 启动] AppModule 初始化
  │    ConfigModule.forRoot() 自动加载 server/.env → process.env
  │
  ├─ [AuthModule 初始化]
  │    导入 SecretsManagerModule → SecretsManagerService 可用
  │
  ├─ [jwtSecretFactory.useFactory 被调用]
  │    └─► SecretsManagerService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey)
  │         └─► SecretsManagerServiceBase.getSecret("JWT_SECRET_KEY")
  │              └─► ConfigService.get("JWT_SECRET_KEY")
  │                   └─► 返回 process.env.JWT_SECRET_KEY = "Change_ME!!!"
  │
  ├─ [JwtModule.registerAsync] 拿到 secret 和 expiresIn
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

### 6.6 为什么保留 `${resourceId}` 等占位符，而不是在构建时完全替换为字面量？

`replacePlaceholdersInCode` 的设计是"替换能替换的，保留替换不了的"：

```typescript
return code.replace(regex, (matched) => {
  const key = matched.slice(2, -1);
  if (mapping.hasOwnProperty(key)) {
    return mapping[key]?.toString() || "";
  } else {
    return matched;   // ★ 找不到就保留原样
  }
});
```

这形成了两级配置机制：
1. **构建时替换**：已知的资源级配置（如 `resourceId`、项目名）直接写入 `.env`，减少部署时的配置量
2. **部署时覆盖**：运维人员拿到生成的 `.env` 后，可以修改那些故意保留的占位符（或直接替换已填入的值），无需重新触发代码生成

生成的 `.env` 文件的注释（虽然 DSG 当前未生成注释）本质上是"开发模板 + 生产配置"的中间产物。
