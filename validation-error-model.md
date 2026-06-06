# Validation 与错误模型处理路径

本文档系统性梳理 Amplication 项目中 **输入校验 → 业务错误 → 前端提示** 的完整映射链路。

---

## 一、整体架构概览

```
客户端 (React + Apollo Client)
       │  GraphQL 请求
       ▼
┌──────────────────────────────────────────────────────────┐
│  NestJS Global Pipeline                                  │
│  1. ValidationPipe (class-validator DTO 校验)            │
│  2. Resolver 方法 (业务逻辑)                              │
│     ├─ Service 层 throw AmplicationError / ...           │
│     └─ Prisma 层 throw PrismaClientKnownRequestError     │
│  3. GqlResolverExceptionsFilter (异常捕获与转换)         │
└──────────────────────────────────────────────────────────┘
       │  GraphQL errors[] 响应
       ▼
客户端 (formatError + 按 code 分发 / Snackbar / LimitationDialog)
```

---

## 二、输入校验（Input Validation）

### 2.1 全局校验管道

在 [main.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/main.ts#L69-L74) 中注册了全局 `ValidationPipe`：

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    forbidUnknownValues: false,
  })
);
```

该 Pipe 基于 `class-validator` + `class-transformer`，对所有传入 DTO 进行字段级校验。**校验失败会抛出 NestJS 内置的 `BadRequestException`**。

### 2.2 DTO 校验装饰器示例

| DTO 文件 | 装饰器 | 说明 |
|---------|--------|------|
| [signup.input.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/auth/dto/signup.input.ts) | `@IsEmail()` / `@IsNotEmpty()` / `@MinLength(8)` | 注册输入 |
| [login.input.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/auth/dto/login.input.ts) | `@IsEmail()` / `@IsNotEmpty()` / `@MinLength(8)` | 登录输入 |
| [change-password.input.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/auth/dto/change-password.input.ts) | `@IsNotEmpty()` / `@MinLength(8)` | 改密输入 |

典型用法：

```typescript
@InputType()
export class SignupInput {
  @Field({ nullable: false })
  @IsEmail()
  email: string;

  @Field({ nullable: false })
  @IsNotEmpty()
  @MinLength(8)
  password: string;
  // ...
}
```

> 注意：多数业务 DTO（如 [EntityCreateInput](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/dto/EntityCreateInput.ts)）并未使用 `class-validator` 装饰器，字段校验逻辑在 Service 层手动完成。

---

## 三、业务错误模型（Error Hierarchy）

### 3.1 错误类继承体系

所有需要向客户端透传的自定义错误都继承自 [AmplicationError](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/AmplicationError.ts)：

```
Error (内置)
 └── AmplicationError
      ├── ValidationError      (dto 或业务字段校验失败)
      ├── DataConflictError    (数据冲突，如版本不一致)
      └── BillingLimitationError (计费/配额限制)
```

各错误实现：
- [AmplicationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/AmplicationError.ts) — 基类，仅做标记
- [ValidationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/ValidationError.ts) — 校验错误
- [DataConflictError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/DataConflictError.ts) — 数据冲突
- [BillingLimitationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/BillingLimitationError.ts) — 带 `billingFeature` 与 `bypassAllowed` 元信息

### 3.2 GraphQL 错误码枚举

共享库 [graphql-error-codes.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/util/graphql-error-codes/src/lib/graphql-error-codes.ts) 定义了统一的错误码：

```typescript
export enum GraphQLErrorCode {
  INTERNAL_SERVER_ERROR    = "INTERNAL_SERVER_ERROR",
  BILLING_LIMITATION_ERROR = "BILLING_LIMITATION_ERROR",
  UNIQUE_KEY_VIOLATION     = "UNIQUE_KEY_VIOLATION",
}
```

这些错误码通过 ApolloError 的 `extensions.code` 字段传递给前端，前端用它做分支处理。

### 3.3 GraphQL 专用错误类

服务端在过滤器中将内部错误转换为可对外暴露的 ApolloError 子类：

| 类 | 文件 | code | extensions |
|----|------|------|-----------|
| `GraphQLUniqueKeyException` | [graphql-unique-key-error.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/graphql/graphql-unique-key-error.ts) | `UNIQUE_KEY_VIOLATION` | — |
| `GraphQLInternalServerError` | [graphql-internal-server-error.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/graphql/graphql-internal-server-error.ts) | `INTERNAL_SERVER_ERROR` | — |
| `GraphQLBillingError` | [graphql-billing-limitation-error.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/graphql/graphql-billing-limitation-error.ts) | `BILLING_LIMITATION_ERROR` | `{ bypassAllowed, billingFeature }` |

---

## 四、服务端异常过滤器（Exception Filters）

### 4.1 GraphQL 全局过滤器 GqlResolverExceptionsFilter

[GqlResolverExceptions.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts) 是整个错误处理的 **核心分发器**。它在各 Resolver 上通过 `@UseFilters(GqlResolverExceptionsFilter)` 启用，由 [exceptionFilters.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/exceptionFilters.module.ts) 统一导出。

转换逻辑（按优先级）：

| 捕获的异常类型 | 转换为 | 日志级别 |
|--------------|--------|---------|
| `PrismaClientKnownRequestError` with code `P2002` (唯一约束冲突) | `GraphQLUniqueKeyException` | info |
| `BillingLimitationError` | `GraphQLBillingError` (保留 billingFeature / bypassAllowed) | info |
| `AmplicationError` (及其子类) | `ApolloError(message)` | info |
| `HttpException` (含 ValidationPipe 抛出的 BadRequestException) | 原样返回 | info |
| 其他未知错误 | 生产：`GraphQLInternalServerError`<br/>开发：`ApolloError(原始message)` | **error** + 完整堆栈 |

关键代码片段：

```typescript
if (exception instanceof Prisma.PrismaClientKnownRequestError
    && exception.code === PRISMA_CODE_UNIQUE_KEY_VIOLATION) {
  const fields = (exception.meta as { target: string[] }).target;
  clientError = new GraphQLUniqueKeyException(fields);
} else if (exception instanceof BillingLimitationError) {
  clientError = new GraphQLBillingError(
    exception.message, exception.billingFeature, exception.bypassAllowed
  );
} else if (exception instanceof AmplicationError) {
  clientError = new ApolloError(exception.message);
} else if (exception instanceof HttpException) {
  clientError = exception;  // 含 ValidationPipe 的 400
} else {
  clientError = NODE_ENV === "production"
    ? new GraphQLInternalServerError()
    : new ApolloError(exception.message);
}
```

### 4.2 REST 接口的 HTTP 异常过滤器

对于 REST API（如 `amplication-plugin-api`、`data-service-generator`），使用 [HttpExceptions.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-plugin-api/src/filters/HttpExceptions.filter.ts) 处理 Prisma 错误，将 Prisma 错误码映射到 HTTP 状态码：

| Prisma Code | HTTP Status | 说明 |
|------------|-------------|------|
| `P2000` | 400 BAD_REQUEST | 值过长 |
| `P2002` | 409 CONFLICT | 唯一约束冲突（格式化输出冲突字段列表） |
| `P2025` | 404 NOT_FOUND | 记录未找到 |

### 4.3 OAuth 相关过滤器

- [AuthExceptionFilter](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/auth-exception.filter.ts)：认证流程出错时 `302 redirect` 到 `/login?error=...`
- [GithubAuthExceptionFilter](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/github-auth-exception.filter.ts)：GitHub OAuth 出错时类似重定向

---

## 五、服务端错误抛出位置示例

| 场景 | 位置 | 抛出的错误 |
|------|------|-----------|
| 不允许创建工作区（套餐限制） | [workspace.service.ts#L110](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L110-L113) | `BillingLimitationError` |
| Team ID 无效 | [team.service.ts#L84](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/team/team.service.ts#L84) | `AmplicationError("INVALID_TEAM_ID")` |
| 注册功能关闭 | [auth.service.ts#L159](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L159) | `AmplicationError(SIGN_UP_DISABLED)` |
| JWT 无效 / 用户不存在 | [jwt.strategy.ts#L32](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/auth/jwt.strategy.ts#L32-L40) | `UnauthorizedException` (HttpException) |
| 数据库唯一键冲突 | Prisma ORM 自动抛出 | `PrismaClientKnownRequestError (P2002)` |
| DTO 字段校验失败（email 格式 / 密码长度） | ValidationPipe 自动抛出 | `BadRequestException` (HttpException) |

---

## 六、前端错误接收与展示

### 6.1 Apollo Client 配置

[graphqlClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/graphqlClient.ts) 构建了 Apollo Client，支持 HTTP、WebSocket（订阅）、文件上传多条链路，但未配置全局错误 Link。错误处理由各组件各自处理。

### 6.2 通用错误格式化：formatError()

[util/error.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/error.ts) 提供统一的错误字符串提取：

```typescript
export function formatError(error) {
  if ((error as ApolloError).graphQLErrors) {
    const [gqlError] = (error as ApolloError).graphQLErrors;
    if (gqlError && gqlError.message) return gqlError.message;
  }
  if (error instanceof AxiosError) return (error as AxiosError).message;
  return String(error);
}
```

策略：**永远取 `graphQLErrors[0].message`**，这与后端过滤器只返回单个 clientError 相匹配。

### 6.3 错误展示组件

- [ErrorMessage.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/ErrorMessage.tsx) — 行内错误提示（带 alert_circle 图标），常用于表单顶部
- `Snackbar`（来自 `@amplication/ui/design-system`）— 右下角 Toast 通知
- `LimitationDialog`（来自 `@amplication/ui/design-system`）— 计费/配额限制弹窗，带"立即升级" / "稍后再说" / "跳过限制" 三种按钮

### 6.4 典型处理方式一：普通错误 → Snackbar / ErrorMessage

例如 [SignInForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/User/SignInForm.tsx#L29-L61)：

```tsx
const [login, { loading, data, error }] = useMutation(DO_LOGIN);
const errorMessage = formatError(error);

return (
  <Formik ...>
    <Form>
      {errorMessage && <ErrorMessage errorMessage={errorMessage} />}
      {/* ...表单字段... */}
    </Form>
  </Formik>
);
```

[Signup.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/User/Signup.tsx#L137) 则使用 Snackbar：

```tsx
<Snackbar open={Boolean(error)} message={errorMessage} />
```

### 6.5 典型处理方式二：Billing 错误 → 弹窗（按 error code 分支）

[useCommits.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/hooks/useCommits.ts#L164-L233) 展示了如何基于 `extensions.code` 做精细化处理：

```typescript
// 1. onError 中识别 billing 错误并打开弹窗
onError: (error: ApolloError) => {
  setOpenLimitationDialog(
    error?.graphQLErrors?.some(
      (gqlError) =>
        gqlError.extensions.code === GraphQLErrorCode.BILLING_LIMITATION_ERROR
    ) ?? false
  );
},

// 2. useMemo 中提取结构化 billing 错误信息
const commitChangesLimitationError = useMemo((): BillingError => {
  const limitation = commitChangesError?.graphQLErrors?.find(
    (gqlError) =>
      gqlError.extensions.code === GraphQLErrorCode.BILLING_LIMITATION_ERROR
  );
  if (!limitation) return;
  return {
    message: formatLimitationError(limitation.message),  // 去掉 "LimitationError: " 前缀
    billingFeature: limitation.extensions.billingFeature as string,
  };
}, [commitChangesError]);
```

然后在 [CommitButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L165-L202) 中根据错误类型选择展示方式：

```tsx
{commitChangesError && isLimitationError ? (
  <LimitationDialog
    isOpen={isOpenLimitationDialog}
    message={commitChangesLimitationError.message}
    allowBypassLimitation={bypassLimitations}
    onConfirm={() => redirectToPurchase()}     // 跳转付费页面
    onDismiss={() => { bypassLimitationsRef.current = false; }}
    onBypass={() => { bypassLimitationsRef.current = true; }}  // 免费用户可临时绕过
  />
) : (
  <Snackbar open={Boolean(commitChangesError)} message={errorMessage} />
)}
```

---

## 七、完整链路对照表

| 触发场景 | 服务端抛出 | 过滤器转换后 | GraphQL 响应 `errors[0]` | 前端展示 |
|---------|-----------|-------------|------------------------|---------|
| DTO 字段无效（密码 < 8 位、email 格式错） | `BadRequestException` (ValidationPipe) | 原样返回 | `message` 包含 NestJS 校验数组 | `ErrorMessage` / `Snackbar` 显示 message |
| 业务规则不满足（Team ID 非法） | `AmplicationError(message)` | `ApolloError(message)` | `message` = 传入字符串 | `Snackbar` / `ErrorMessage` |
| 数据冲突（手动校验） | `DataConflictError(message)` | `ApolloError(message)` | `message` = 传入字符串 | `Snackbar` |
| 唯一键冲突（email 已注册） | `Prisma.P2002` | `GraphQLUniqueKeyException(fields)` | `extensions.code = UNIQUE_KEY_VIOLATION` <br/> `message = "Another record with the same key already exist (...)"` | `Snackbar` 显示 message |
| 套餐限制（工作区数已满） | `BillingLimitationError` | `GraphQLBillingError` | `extensions.code = BILLING_LIMITATION_ERROR` <br/> `extensions.billingFeature` <br/> `extensions.bypassAllowed` | `LimitationDialog` + 升级/绕过按钮 |
| 未登录 / JWT 失效 | `UnauthorizedException` | 原样返回 | HTTP 401 | Apollo 透传，路由守卫跳转登录 |
| 未预期异常 / 500 | 任意 Error | 生产：`GraphQLInternalServerError`<br/>开发：`ApolloError(原始message)` | `extensions.code = INTERNAL_SERVER_ERROR` | `Snackbar` 显示 "Internal server error" |

---

## 八、关键文件索引

| 类别 | 文件路径 |
|------|---------|
| 全局校验管道 | [main.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/main.ts#L69-L74) |
| 错误基类 | [AmplicationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/AmplicationError.ts) |
| GraphQL 错误码 | [graphql-error-codes.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/util/graphql-error-codes/src/lib/graphql-error-codes.ts) |
| GraphQL 异常过滤器（核心） | [GqlResolverExceptions.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts) |
| 过滤器模块 | [exceptionFilters.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/exceptionFilters.module.ts) |
| REST Prisma 错误映射 | [HttpExceptions.filter.ts (plugin-api)](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-plugin-api/src/filters/HttpExceptions.filter.ts) |
| 前端错误格式化 | [util/error.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/error.ts) |
| 前端 Billing 错误识别 | [useCommits.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/hooks/useCommits.ts) |
| 前端弹窗展示 | [CommitButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx) |
