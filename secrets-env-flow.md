# Secrets 与环境变量传递流程深度分析

## 1. 架构总览

整个系统在两个层面上处理 secrets 与环境变量：

| 层面 | 职责 | 关键代码位置 |
|------|------|-------------|
| **代码生成层 (DSG)** | 构建时根据资源配置动态生成 secrets 枚举、.env 文件、SecretsManager 模块 | `packages/data-service-generator/` |
| **运行时层 (Generated App)** | 应用启动时从环境/外部 provider 读取 secrets，通过 DI 注入到业务模块 | `packages/gpt-gateway/src/providers/secrets/` (参考实现) |

---

## 2. 密钥作用域 (Secrets Scope)

### 2.1 核心数据结构

密钥的声明由 `SecretsNameKey` 接口定义，位于 [plugin-events-params.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/libs/util/code-gen-types/src/plugin-events-params.types.ts#L371-L374)：

```typescript
export interface SecretsNameKey {
  name: string;   // 枚举成员名 (PascalCase，如 "JwtSecretKey")
  key: string;    // 实际环境变量名 (如 "JWT_SECRET_KEY")
}
```

运行时的枚举由代码生成器动态创建，输出示例（gpt-gateway 中）在 [secretsNameKey.enum.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/gpt-gateway/src/providers/secrets/secretsNameKey.enum.ts#L1-L3)：

```typescript
export enum EnumSecretsNameKey {
  JwtSecretKey = "JWT_SECRET_KEY",
}
```

### 2.2 作用域的三层边界

**第一层：枚举定义层 —— 编译期可见性**

`EnumSecretsNameKey` 是 secrets 的"白名单"。任何 secret 必须先在此枚举中注册，才能通过类型安全的方式被 `SecretsManagerService.getSecret<T>()` 读取。

- 枚举生成逻辑在 [create-secrets-manager.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/server/secrets-manager/create-secrets-manager.ts#L70-L83) 的 `createTSEnumSecretsNameKey()` 函数
- 转换规则：`name` 经 `pascalCase()` 转为枚举成员标识符，`key` 作为字符串字面量值

**第二层：Module 导出层 —— NestJS DI 容器可见性**

SecretsManager 本身通过 `SecretsManagerModule` 封装：

- 定义于 [secretsManager.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/gpt-gateway/src/providers/secrets/secretsManager.module.ts#L1-L8)
- `providers: [SecretsManagerService]` + `exports: [SecretsManagerService]`
- 只有显式 `imports: [SecretsManagerModule]` 的模块才能注入 `SecretsManagerService`

**第三层：运行时取值层 —— 环境变量可见性**

`SecretsManagerServiceBase.getSecret()` 最终调用 NestJS 的 `ConfigService.get(key.toString())`，只在已加载的环境变量中查找。见 [secretsManager.service.base.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/gpt-gateway/src/providers/secrets/base/secretsManager.service.base.ts#L8-L17)：

```typescript
async getSecret<T>(key: EnumSecretsNameKey): Promise<T | null> {
  const value = this.configService.get(key.toString());
  if (value) {
    return value;
  }
  return null;
}
```

> **关键设计**：返回 `null` 而非抛异常，把"secret 是否必须存在"的判断交给调用方（如 `jwtSecretFactory` 会抛错，而可选配置可能接受 `null`）。

---

## 3. 构建上下文传递机制 (Build Context Flow)

### 3.1 整体管线

```
generateCode()
  └─► generateCodeByResourceData()
       └─► createDataService()
            ├─► prepareContext()          // 初始化 DsgContext，注册插件
            ├─► createDTOs()
            ├─► createServer()            // Server 端代码生成
            │    └─► ...
            │    ├─► createSecretsManager({ secretsNameKey: [] })   // ★ 初始为空
            │    ├─► createDotEnvModule({ envVariables: ENV_VARIABLES })
            │    └─► ...
            └─► createAdminModules()      // Admin UI 代码生成
                 └─► createDotEnvModule()
```

入口：[generate-code.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/generate-code.ts#L48-L64)

### 3.2 DsgContext 单例上下文

[DsgContext](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/dsg-context.ts#L19-L81) 是贯穿构建过程的全局状态容器（单例模式）：

| 关键字段 | 作用 |
|---------|------|
| `appInfo.settings` | 资源级配置，占位符替换的数据源 |
| `plugins: PluginMap` | 按事件名组织的 before/after 插件钩子 |
| `modules: ModuleMap` | 已生成的文件集合 |
| `serverDirectories` / `clientDirectories` | 输出路径结构 |

### 3.3 插件包装器 (pluginWrapper) —— secrets 扩展的核心机制

每一个代码生成函数都被 `pluginWrapper()` 包裹，形成 **before 钩子 → 默认行为 → after 钩子** 的洋葱模型。

定义于 [plugin-wrapper.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/plugin-wrapper.ts#L59-L117)：

```typescript
const pluginWrapper = async (func, event, args) => {
  const context = DsgContext.getInstance;

  // 1. before 管道：插件可修改 eventParams (如追加 secretsNameKey)
  const updatedEventParams = beforePlugins
    ? await beforeEventsPipe(...beforePlugins)(context, args)
    : args;

  // 2. 默认行为（可被 context.utils.skipDefaultBehavior 跳过）
  const defaultBehaviorModules = await defaultBehavior(
    context, func, updatedEventParams
  );

  // 3. after 管道：插件可修改生成的 ModuleMap (如追加 .env 条目)
  const finalModules = afterPlugins
    ? await afterEventsPipe(...afterPlugins)(context, args, defaultBehaviorModules)
    : defaultBehaviorModules;

  return finalModules;
};
```

**插件注入 secrets 的两个时机**：

| 事件 | 时机 | 典型操作 |
|------|------|---------|
| `CreateServerSecretsManager` | before | 向 `eventParams.secretsNameKey[]` 追加新的密钥定义，使其被纳入 `EnumSecretsNameKey` |
| `CreateServerDotEnv` | after | 向生成的 `.env` 内容追加变量行，或修改已有变量 |

`EventNames` 枚举中明确定义了这两个事件：见 [plugins.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/libs/util/code-gen-types/src/plugins.types.ts#L77-L124)

### 3.4 .env 文件生成流程

Server 端 `.env` 生成位于 [create-dotenv.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/server/create-dotenv.ts#L28-L48)：

```
输入: envVariables: VariableDictionary = [{ KEY: "value" }, ...]
  │
  ├─► removeDuplicateKeys()       // 去重（后出现的覆盖先出现的）
  ├─► sortAlphabetically()        // 按 key 字母排序，保证输出稳定
  ├─► convertToKeyValueSting()    // 转为 "KEY=value\nKEY2=value2" 格式
  └─► replacePlaceholdersInCode() // 替换 ${resourceId} 等占位符，数据源为 appInfo.settings
       │
       └─► 输出: server/.env
```

- `VariableDictionary` 类型：`{ [variable: string]: string }[]`，定义于 [plugin-events-params.types.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/libs/util/code-gen-types/src/plugin-events-params.types.ts#L181-L183)
- 默认变量在 [constants.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/server/constants.ts#L7-L11)：`BCRYPT_SALT=10`、`COMPOSE_PROJECT_NAME=amp_${resourceId}`、`PORT=3000`
- 占位符替换引擎：[text-file-parser.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/utils/text-file-parser.ts#L1-L19)，匹配 `${key}` 并用 `appInfo.settings[key]` 替换

Admin 端 `.env` 生成在 [admin/create-dotenv.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/admin/create-dotenv.ts#L33-L60)，额外从模板文件提取预置变量：

```typescript
const code = await readCode(templatePath);               // 读取 create-dotenv.template.env
const extractedVariables = extractVariablesFromCode(code); // 从模板解析 KEY=VALUE 行
const allVariables = [...extractedVariables, ...envVariablesWithoutDuplicateKeys];
```

---

## 4. 外部 Provider 引用边界

### 4.1 边界接口设计

系统通过**两层抽象**隔离默认实现与外部 provider：

```
┌─────────────────────────────────────────────┐
│  业务代码 (AuthModule, JwtStrategy 等)      │
│  只依赖 EnumSecretsNameKey + 接口           │
└──────────────────┬──────────────────────────┘
                   │ 依赖注入
                   ▼
┌─────────────────────────────────────────────┐
│  SecretsManagerService (可继承)             │
│  extends SecretsManagerServiceBase          │
│  └─► getSecret(key: EnumSecretsNameKey)     │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│  ConfigService (NestJS)                     │
│  └─► 默认从 process.env 读取                │  ←── 可替换为 AWS Secrets Manager / Vault / ...
└─────────────────────────────────────────────┘
```

### 4.2 默认实现的边界

**基类**: [SecretsManagerServiceBase](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/server/secrets-manager/static/base/secretsManager.service.base.template.ts#L1-L17)

```typescript
export interface ISecretsManager {
  getSecret: (key: EnumSecretsNameKey) => Promise<any | null>;
}

export class SecretsManagerServiceBase implements ISecretsManager {
  constructor(protected readonly configService: ConfigService) {}
  async getSecret<T>(key: EnumSecretsNameKey): Promise<T | null> {
    const value = this.configService.get(key.toString());
    return value ?? null;
  }
}
```

**子类（生成时复制，无新增逻辑）**: [SecretsManagerService](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/data-service-generator/src/server/secrets-manager/static/secretsManager.service.template.ts#L1-L10)

```typescript
@Injectable()
export class SecretsManagerService extends SecretsManagerServiceBase {
  constructor(protected readonly configService: ConfigService) {
    super(configService);
  }
}
```

> **扩展点设计意图**：`SecretsManagerService` 作为空子类存在，是为了让用户/插件在不修改基类的情况下，通过**替换子类实现**来接入外部 secrets provider（例如改为调用 AWS SDK、HashiCorp Vault API 等）。基类保持最小功能，子类是定制化边界。

### 4.3 消费端边界：JWT Secret 工厂模式

以 JWT 鉴权为例，展示 secrets 如何从 SecretsManager 流向业务模块。完整链路在 [auth.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/gpt-gateway/src/auth/auth.module.ts#L1-L57)：

**步骤 1：独立 Factory Provider**

[jwtSecretFactory.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/gpt-gateway/src/auth/jwt/jwtSecretFactory.ts#L1-L19)：

```typescript
export const jwtSecretFactory = {
  provide: JWT_SECRET_KEY_PROVIDER_NAME,   // 字符串 token: "JWT_SECRET_KEY"
  useFactory: async (secretsService: SecretsManagerService): Promise<string> => {
    const secret = await secretsService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey);
    if (secret) return secret;
    throw new Error("jwtSecretFactory missing secret");  // 必须存在则抛错
  },
  inject: [SecretsManagerService],
};
```

**步骤 2：JwtModule 异步注册**（同一 auth.module.ts 中）

```typescript
JwtModule.registerAsync({
  imports: [SecretsManagerModule],
  inject: [SecretsManagerService, ConfigService],
  useFactory: async (secretsService, configService) => {
    const secret = await secretsService.getSecret<string>(EnumSecretsNameKey.JwtSecretKey);
    const expiresIn = configService.get(JWT_EXPIRATION);
    return { secret, signOptions: { expiresIn } };
  },
})
```

**步骤 3：JwtStrategy 通过 @Inject 消费**

[jwt.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/50-amplication/packages/gpt-gateway/src/auth/jwt/jwt.strategy.ts#L1-L14)：

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

### 4.4 边界总结

| 边界层 | 接口 | 可否替换 | 替换方式 |
|--------|------|---------|---------|
| Secrets 枚举定义 | `EnumSecretsNameKey` | 是 | 插件通过 `CreateServerSecretsManager` before 钩子追加 `SecretsNameKey[]` |
| SecretsManager 服务实现 | `ISecretsManager` / `SecretsManagerServiceBase` | 是 | 继承基类或重新实现 `SecretsManagerService`，改用外部 SDK |
| 配置源 (ConfigService 后端) | NestJS `ConfigModule` | 是 | 在生成的 `app.module.ts` 中替换 `ConfigModule.forRoot()` 的 `load` 配置，接入自定义配置 loader |
| 消费端注入方式 | `@Inject(TOKEN)` + `useFactory` | 是（不推荐） | 可改为直接读取，但破坏可测试性 |
| .env 变量列表 | `CreateServerDotEnvParams.envVariables` | 是 | 插件通过 `CreateServerDotEnv` after 钩子修改 ModuleMap |

---

## 5. 端到端流程示例：JWT_SECRET_KEY 从构建到运行

```
[构建阶段]
  │
  ├─ 插件在 CreateServerSecretsManager 的 before 钩子中追加：
  │    { name: "JwtSecretKey", key: "JWT_SECRET_KEY" }
  │
  ├─ createSecretsManagerInternal() 生成 EnumSecretsNameKey 枚举
  │    → JwtSecretKey = "JWT_SECRET_KEY"
  │
  ├─ 插件在 CreateServerDotEnv 的 after 钩子中追加 .env 行：
  │    JWT_SECRET_KEY=Change_ME!!!
  │
  └─ 代码写入磁盘
        server/src/providers/secrets/secretsNameKey.enum.ts
        server/src/providers/secrets/secretsManager.service.base.ts
        server/src/providers/secrets/secretsManager.service.ts
        server/src/providers/secrets/secretsManager.module.ts
        server/src/auth/jwt/jwtSecretFactory.ts
        server/.env

[运行阶段]
  │
  ├─ NestJS 启动，ConfigModule.forRoot({ isGlobal: true })
  │    → 自动加载 .env 文件到 process.env
  │
  ├─ AuthModule 导入 SecretsManagerModule
  │
  ├─ jwtSecretFactory.useFactory() 被调用
  │    └─► SecretsManagerService.getSecret(EnumSecretsNameKey.JwtSecretKey)
  │         └─► ConfigService.get("JWT_SECRET_KEY")
  │              └─► 返回 process.env.JWT_SECRET_KEY
  │
  ├─ JwtModule.registerAsync 拿到 secret，配置 JWT 签名
  │
  └─ JwtStrategy 通过 @Inject(JWT_SECRET_KEY_PROVIDER_NAME) 拿到 secret
       └─► Passport JWT 验证中间件使用该 secret 校验 token
```

---

## 6. 关键设计决策

### 6.1 为什么 SecretsManagerService 返回 Promise？

虽然默认实现是同步读 `ConfigService`，但接口设计为异步 `Promise<T | null>`，是为了给**外部 provider（如 AWS Secrets Manager、Vault）的异步 HTTP 调用**预留空间，无需改消费端代码。

### 6.2 为什么区分 EnumSecretsNameKey 与直接字符串？

- **类型安全**：传入不存在的 key 会在编译期报错
- **可发现性**：通过枚举可以看到项目中所有被声明的 secrets
- **可重构**：修改环境变量名只需改枚举值，影响范围可控

### 6.3 为什么用 Factory Provider 而非直接注入 SecretsManager？

以 `jwtSecretFactory` 为例：
- `JwtStrategy` 只需要 `string`（secret 本身），不需要知道 SecretsManager 的存在
- 符合 **依赖倒置原则**：高层策略（JWT 验证）不依赖低层实现（secrets 获取方式）
- 便于测试：测试时可直接 provide 一个字符串常量，无需 mock SecretsManager

### 6.4 .env 中为什么保留占位符 `${resourceId}` 而非直接替换？

构建时通过 `appInfo.settings` 做一次替换，但用户部署时可能需要再次调整。两层机制兼顾：
- 构建时：自动填充已知值（如资源 ID）
- 部署时：运维人员可直接修改 .env，无需重新生成代码
