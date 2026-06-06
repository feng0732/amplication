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

### 2.3 前端表单校验体系（Formik + JSON Schema / Yup）

前端存在 **两套** 表单校验机制，分别基于 JSON Schema 和 Yup：

#### 2.3.1 JSON Schema 校验（主流）

[formikValidateJsonSchema.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/formikValidateJsonSchema.ts) 提供基于 Ajv 的通用 `validate()` 函数，与 Formik 的 `validate` prop 无缝对接。

核心实现：

```typescript
export function validate<T>(values: T, validationSchema): FormikErrors<T> {
  const errors: FormikErrors<T> = {};
  const ajv = new Ajv({ allErrors: true });

  // 自定义关键字：isNotEmpty（字符串非空/非纯空白）
  ajv.addKeyword("isNotEmpty", {
    type: "string",
    validate: (schema, data) =>
      typeof data === "string" && data.trim() !== "",
    errors: true,
  });

  ajvErrors(ajv);  // 启用 ajv-errors 自定义错误消息
  const isValid = ajv.validate(validationSchema, values);

  // 跨字段自定义校验：min < max
  if (minimumValue && minimumValue >= maximumValue) {
    set(errors, "minimumValue",
      "Minimum value can not be greater than, or equal to, the Maximum value");
  }

  // 将 ajv.errors 扁平化映射为 FormikErrors（嵌套路径用 lodash.set）
  if (!isValid && ajv.errors) {
    for (const error of ajv.errors) {
      const fieldName = error.dataPath.substring(1).replaceAll("/", ".");
      set(errors, fieldName, error.message);
    }
  }
  return errors;
}
```

内置常量错误文案：

```typescript
export const validationErrorMessages = {
  AT_LEAST_TWO_CHARACTERS: "Must be at least 2 characters long",
  NO_SYMBOLS_ERROR: "Unsupported character",
  AT_MOST_SIXTY_CHARACTERS: "Must be at most 60 characters long",
};
```

**典型用法（EntityForm）**：[EntityForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Entity/EntityForm.tsx#L45-L99)

```tsx
const FORM_SCHEMA = {
  required: ["name", "displayName", "pluralDisplayName"],
  properties: {
    displayName: { type: "string", minLength: 2 },
    name:        { type: "string", minLength: 2 },
    pluralDisplayName: { type: "string", minLength: 2 },
  },
  errorMessage: {
    properties: {
      displayName: AT_LEAST_TWO_CHARACTERS,
      name:        AT_LEAST_TWO_CHARACTERS,
      pluralDisplayName: AT_LEAST_TWO_CHARACTERS,
    },
  },
};

<Formik
  validate={(values) => {
    // 先做跨字段自定义校验
    if (isEqual(values.name, values.pluralDisplayName)) {
      return { pluralDisplayName: "Name and plural display names cannot be equal..." };
    }
    return validate(values, FORM_SCHEMA);  // 再跑 JSON Schema
  }}
  ...
```

#### 2.3.2 Yup Schema 校验（仅少量场景）

部分表单（如 Git 创建仓库）使用 [Yup](https://github.com/jquense/yup)，例如 [CreateGitFormSchema.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Resource/git/dialogs/GitCreateRepo/CreateGitFormSchema/CreateGitFormSchema.ts)：

```typescript
export const CreateGitFormSchema = object().shape({
  name: string()
    .min(2, "Git repository name require minimum of 2 characters")
    .required("Repository name is missing"),
  isPublic: bool().required("Must select if repo is private"),
});
```

#### 2.3.3 字段级内联校验（NameField 与 TopicNameField 对比）

项目中存在两个功能类似的字段级校验组件：`NameField`（通用实体/字段名）和 `TopicNameField`（Topic 专用名），二者均通过 Formik `useField` 的 `validate` 回调做即时正则校验。

##### NameField（通用）

[NameField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx#L7-L49) 支持普通和首字母大写两种模式：

```tsx
const NAME_REGEX = /^(?![0-9])[a-zA-Z0-9$_]+$/;          // 普通模式（数字不允许开头）
const CAPITALIZED_NAME_REGEX = /^[A-Z][a-zA-Z0-9$_]+$/;     // 大写模式（Entity 等场景）

const [field] = useField<string>({
  ...rest,
  validate: (value) => (value.match(regexp) ? undefined : helpText),
});

return (
  <div>
    <TextInput {...field} label="Name" pattern={pattern} />
    <ErrorMessage name="name" component="div" className="amplication-label__error" />
  </div>
);
```

##### TopicNameField（Topic 专用）

[TopicNameField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/TopicNameField.tsx#L6-L38) 正则更宽松（允许 `.`、`-`、`_`，不强制首字母大写）：

```tsx
const TOPIC_REGEX = /^[a-zA-Z0-9._-]+$/;
const HELP_TEXT = "Name must only contain letters, numbers, dash, underscore or dot.";

const TopicNameField = ({ name, ...rest }: Props) => {
  const [field] = useField<string>({
    name,                                         // ← useField 绑定的是 props.name
    validate: (value) => (TOPIC_REGEX.test(value) ? undefined : HELP_TEXT),
  });
  return (
    <div>
      <TextInput {...field} {...rest} pattern={TOPIC_PATTERN} />
      <ErrorMessage
        name="name"                                // ← ⚠️ Bug：硬编码 "name" 而非 props.name
        component="div"
        className="amplication-label__error"
      />
    </div>
  );
};
```

##### 错误绑定 Bug：`ErrorMessage name` 硬编码

两个组件都存在同一个 Bug——[NameField.tsx#L43-L46](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx#L43-L46) 和 [TopicNameField.tsx#L32-L35](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/TopicNameField.tsx#L32-L35) 中 `<ErrorMessage name="name" />` 都被写死为字符串 `"name"`，而没有从 props 读取动态的字段名。

| 组件 | useField 绑定 | ErrorMessage 绑定 | 当表单字段名不是 "name" 时 |
|------|--------------|------------------|--------------------------|
| NameField | 通过 `...rest` 透传 `name` | **硬编码 `"name"`** | 表单能正确阻止提交（validate 仍生效），但错误提示永远绑定到 `formik.errors["name"]`，不会在该字段下方显示 |
| TopicNameField | 通过 props `name` 绑定 | **硬编码 `"name"`** | 同上 |

TopicForm 中的使用（[TopicForm.tsx#L68](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Topics/TopicForm.tsx#L68)）恰好字段名就叫 `"name"`，因此 Bug 被掩盖：

```tsx
<TopicNameField label="Name" name="name" />  // name === "name"，碰巧正常
```

> 该 `TOPIC_REGEX = /^[a-zA-Z0-9._-]+$/` 与服务端 DTO 上的 `@Matches(/^[a-zA-Z0-9._-]+$/)`（[TopicCreateInput.ts#L12](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/topic/dto/TopicCreateInput.ts#L12)）完全一致，实现"前后端校验规则同源"（详见第九节映射表）。

#### 2.3.4 自动保存与校验触发：FormikAutoSave

[formikAutoSave.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/formikAutoSave.tsx) 通过 `useFormikContext` 监听 `formik.values` + `formik.dirty`，防抖 1s 后自动 `submitForm()`，并通过 `.catch(onError)` 将服务端错误向外传递：

```tsx
const debouncedSubmit = useCallback(
  debounce(() => {
    return formik.submitForm().then(() => {}).catch(onError);
  }, debounceMS),
  [formik.submitForm, debounceMS]
);

useEffect(() => {
  if (formik.dirty) debouncedSubmit();
}, [debouncedSubmit, formik.values, formik.dirty]);
```

---

## 三、业务错误模型（Error Hierarchy）

### 3.1 错误类继承体系

所有需要向客户端透传的自定义错误都继承自 [AmplicationError](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/AmplicationError.ts)：

```
Error (内置)
 └── AmplicationError
      ├── ValidationError          (dto 或业务字段校验失败)
      ├── DataConflictError        (数据冲突，如版本不一致)
      ├── BillingLimitationError   (计费/配额限制，带 billingFeature)
      └── ReservedNameError        (使用了保留关键字)
```

各错误实现：
- [AmplicationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/AmplicationError.ts) — 基类，仅做标记
- [ValidationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/ValidationError.ts) — 校验错误
- [DataConflictError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/DataConflictError.ts) — 数据冲突
- [BillingLimitationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/BillingLimitationError.ts) — 带 `billingFeature` 与 `bypassAllowed` 元信息
- [ReservedNameError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/resource/ReservedNameError.ts) — `name` 字段使用了 JavaScript/Prisma 保留字

### 3.1.1 Service 层手动校验（弥补 DTO 校验缺失）

由于多数业务 DTO 没有 class-validator 装饰器，实际的字段合法性校验在 Service 层通过正则和函数手动完成，以 `entity.service.ts` 为例：

| 校验项 | 代码位置 | 校验逻辑 | 抛出的错误 |
|-------|---------|---------|-----------|
| 字段名称格式 | [entity.service.ts#L3285-L3289](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L3285-L3289) | `NAME_REGEX = /^(?![0-9])[a-zA-Z0-9$_]+$/` | `ConflictException(NAME_VALIDATION_ERROR_MESSAGE)` |
| 保留字检查 | [entity.service.ts#L1201-L1203](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L1201-L1203) | `isReservedName(name)` | `ReservedNameError(name)` |
| name 与 pluralDisplayName 不能相同 | [entity.service.ts#L1195-L1198](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L1195-L1198) | `newName === newPluralDisplayName` | `AmplicationError("The entity name and plural display name cannot be the same.")` |
| Topic name 正则 | [TopicCreateInput.ts#L12](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/topic/dto/TopicCreateInput.ts#L12) | `@Matches(/^[a-zA-Z0-9._-]+$/)` | ValidationPipe 抛出 `BadRequestException` |

> 保留字列表定义在 [reservedNames.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/reservedNames.ts)，包含 60+ JavaScript/TypeScript 关键字以及 `field`、`app`、`auth` 等 Prisma/业务保留字。

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

#### 6.5.1 useCommits Hook：错误识别与状态暴露

[useCommits.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/hooks/useCommits.ts) 承担了 Billing 错误的识别、结构化与状态维护：

```
useCommits 返回的相关状态：
 ├─ commitChanges                    : (data) => void           触发 commit 的函数
 ├─ commitChangesError               : ApolloError | undefined  完整的 mutation 错误对象
 ├─ commitChangesLoading             : boolean
 ├─ commitChangesLimitationError     : BillingError | undefined 仅提取 billing 信息
 ├─ isOpenLimitationDialog           : boolean                  弹窗打开状态（hook 内部维护）
 └─ bypassLimitations                : boolean                  当前订阅是否允许绕过限制
```

识别与提取过程：

```typescript
// useCommits 内部状态
const [isOpenLimitationDialog, setOpenLimitationDialog] = useState(false);

// 1. useMutation onError：按 error code 判断是否为计费限制
const [commit, { error: commitChangesError }] = useMutation(COMMIT_CHANGES, {
  onError: (error: ApolloError) => {
    setCommitRunning(false);
    setPendingChangesError(true);
    setOpenLimitationDialog(
      error?.graphQLErrors?.some(
        (gqlError) =>
          gqlError.extensions.code === GraphQLErrorCode.BILLING_LIMITATION_ERROR
      ) ?? false
    );
  },
});

// 2. useMemo 中从 graphQLErrors 里"挑出" billing 错误并结构化
const commitChangesLimitationError = useMemo((): BillingError => {
  const limitation = commitChangesError?.graphQLErrors?.find(
    (gqlError) =>
      gqlError.extensions.code === GraphQLErrorCode.BILLING_LIMITATION_ERROR
  );
  if (!limitation) return;
  return {
    message: formatLimitationError(limitation.message),  // 剥掉 "LimitationError: " 前缀
    billingFeature: limitation.extensions.billingFeature as string,
  };
}, [commitChangesError]);

// 3. 根据当前订阅判断是否允许绕过限制（非 Pro 才显示 bypass 按钮）
const bypassLimitations = useMemo(() => {
  return currentWorkspace?.subscription?.subscriptionPlan !== EnumSubscriptionPlan.Pro;
}, [currentWorkspace]);
```

#### 6.5.2 CommitButton：弹窗状态机与双状态问题

[CommitButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx) 从 `useCommits` 取到 `commitChangesLimitationError`、`bypassLimitations` 等数据，但 **弹窗开关状态并未复用 hook 中的 `isOpenLimitationDialog`**，而是在组件内又声明了一份：

```tsx
// CommitButton 内部——与 useCommits 中的同名状态相互独立
const [isOpenLimitationDialog, setOpenLimitationDialog] = useState<boolean>(false);

const {
  commitChanges,
  commitChangesError,
  commitChangesLimitationError,  // ← 使用 hook 提供的错误信息
  bypassLimitations,
  // isOpenLimitationDialog    ← 但 hook 返回的这个状态被完全忽略！
} = useCommits(currentProject?.id);
```

> ⚠️ **设计缺陷（双重不一致）**：
> 1. `useCommits` 在 `onError` 中 `setOpenLimitationDialog(true)`，但 CommitButton 解构 hook 返回值时**完全没有使用** `isOpenLimitationDialog`，hook 中的那份 state 从未被消费。
> 2. CommitButton 自身维护的 `useState<boolean>(false)` 初始值为 `false`，且组件中**不存在任何 `useEffect` 将 `commitChangesLimitationError` 的出现同步为 `setOpenLimitationDialog(true)`**。

#### 6.5.2.1 Dialog 组件 isOpen 的双重守卫

深入 [Dialog.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/ui/design-system/src/lib/components/Dialog/Dialog.tsx#L27-L61)（`LimitationDialog` 的基础组件）可以看到 `isOpen` 实际上承担了 **两层守卫**：

```tsx
export const Dialog: React.FC<Props> = ({
  isOpen = false,
  onDismiss,
  children,
  ...
}) => {
  return isOpen ? (                           // ← 第一层守卫：决定是否渲染整个 DOM
    <MuiDialog
      fullScreen={false}
      open={isOpen}                           // ← 第二层守卫：MUI 内部动画/模态显示
      onClick={(e) => e.stopPropagation()}
      onClose={onDismiss}
      ...
    >
      ...
    </MuiDialog>
  ) : null;
};
```

而 [LimitationDialog.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/ui/design-system/src/lib/components/LimitationDialog/LimitationDialog.tsx#L20-L73) 仅透传 `isOpen`：

```tsx
export const LimitationDialog = ({
  isOpen, message, allowBypassLimitation, onConfirm, onBypass, onDismiss,
}: Props) => {
  return (
    <Dialog title="" className={CLASS_NAME} isOpen={isOpen} onDismiss={onDismiss}>
      ...
    </Dialog>
  );
};
```

**对 Billing 路径的实际影响**：当 CommitButton 本地 `isOpenLimitationDialog = false`（初始值）时，不仅 MUI Dialog 处于关闭状态，而且整个 Dialog DOM 节点都不会被渲染（第一层守卫直接 `return null`）。因此，仅靠外层 JSX 的 `commitChangesError && isLimitationError` 条件把 `<LimitationDialog>` 组件挂载到 React 树上还不够——**必须同时让 `isOpen=true`**，弹窗才能被用户看见。

#### 6.5.2.2 isOpen 的实际"打开"路径分析

CommitButton 中把本地 state 置为 `true` 的调用点仅存在于三个关闭回调里——而且设的都是 `false`：

| 位置 | 调用 |
|------|------|
| [CommitButton.tsx#L178](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L178) onConfirm | `setOpenLimitationDialog(false)` |
| [CommitButton.tsx#L187](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L187) onDismiss | `setOpenLimitationDialog(false)` |
| [CommitButton.tsx#L197](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L197) onBypass | `setOpenLimitationDialog(false)` |

**不存在 `setOpenLimitationDialog(true)` 的调用。** 这意味着弹窗理论上永远打不开——只有一种例外：当开发者直接修改 MUI Dialog 的行为，或 Dialog 的默认值被改为 `true`。目前的实现依赖 **错误出现时恰好 Dialog 被 mount，同时 MUI Dialog `open={false}` 不会阻止初始渲染动画**（MUI 规范上 `open={false}` 就是隐藏的）。

#### 6.5.2.3 实际可见性逻辑与修复建议

当前 JSX（[CommitButton.tsx#L165-L202](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L165-L202)）：

```tsx
{commitChangesError && isLimitationError ? (
  <LimitationDialog
    isOpen={isOpenLimitationDialog}   // ← 永远是 false
    message={commitChangesLimitationError.message}
    allowBypassLimitation={bypassLimitations}
    onConfirm={() => { ... setOpenLimitationDialog(false) }}
    onDismiss={() => { ... setOpenLimitationDialog(false) }}
    onBypass={() => { ... setOpenLimitationDialog(false) }}
  />
) : (
  <Snackbar open={Boolean(commitChangesError)} message={errorMessage} />
)}
```

修复建议（两者取其一）：
1. **直接使用 hook 返回的 `isOpenLimitationDialog`**：`useCommits` 的 onError 已经把它设为 `true`，解构时用它即可
2. **在 CommitButton 中加 useEffect 同步错误到本地 state**：
   ```tsx
   useEffect(() => {
     if (commitChangesLimitationError) {
       setOpenLimitationDialog(true);
     }
   }, [commitChangesLimitationError]);
   ```

**完整交互流程（Billing 路径，含 isOpen 标注）**：

```
用户点击"Generate the code"
        │
        ▼
commitChanges({ ..., bypassLimitations: bypassLimitationsRef.current })
        │
        ▼
useMutation COMMIT_CHANGES ──► 服务端 throw BillingLimitationError
        │
        ▼
onError 触发:
  setCommitRunning(false)
  setPendingChangesError(true)
  setOpenLimitationDialog(true)    // hook 内部 state（未被消费）
        │
        ▼
commitChangesError 更新 → commitChangesLimitationError 被计算出
        │
        ▼
JSX: commitChangesError && isLimitationError = true
        │
        ▼
<LimitationDialog isOpen={本地 state = false}> 被挂载
        │
        ├─► Dialog 第一层守卫 isOpen ? ... : null
        │       isOpen = false → return null（Dialog DOM 不渲染）
        │
        ├─ 用户不可见，也无法点击 onConfirm / onDismiss / onBypass
        │
        ▼
（实际上只会走 Snackbar 分支，因为 Dialog 不可见）
```

#### 6.5.3 Billing 错误在其它消费点的降级处理

`PublishTemplatesChangesButton.tsx` 等其它使用 `useCommits` 的组件并未处理 billing 分支，它们直接：

```tsx
const { commitChanges, commitChangesError, commitChangesLoading } = useCommits(projectId);
const errorMessage = formatError(commitChangesError);
// ...
<Snackbar open={Boolean(errorMessage)} message={errorMessage} />
```

即即使返回的是 `BILLING_LIMITATION_ERROR`，也会退化为普通 Snackbar 文本提示 `"LimitationError: ..."`，没有升级引导弹窗。

---

## 七、完整链路对照表（含前端本地校验）

| 触发场景 | 校验层 | 抛出 / 产生 | 过滤器转换后 | GraphQL 响应 / 前端错误对象 | 前端展示 |
|---------|--------|------------|-------------|---------------------------|---------|
| 表单输入长度 < 2（displayName、name 等） | 前端本地 JSON Schema | `FormikErrors.displayName = "Must be at least 2 characters long"` | — | `formik.errors` 内联 | `Formik <ErrorMessage />` 在字段下红色提示 |
| Entity 名等于 pluralDisplayName | 前端本地跨字段校验 | `FormikErrors.pluralDisplayName = "Name and plural display names cannot be equal..."` | — | `formik.errors` 内联 | `<ErrorMessage name="pluralDisplayName" />` |
| Name 字段格式非法（数字开头/含非法字符） | 前端本地 NameField 组件内联 | `FormikErrors.name = HELP_TEXT` | — | `formik.errors` 内联 | `<ErrorMessage name="name" />` 在字段下方 |
| DTO 字段无效（密码 < 8 位、email 格式错） | 服务端 ValidationPipe | `BadRequestException` (class-validator) | 原样返回 | `message` 包含 NestJS 校验数组 | `ErrorMessage` / `Snackbar` 显示 message |
| Topic name 正则不匹配 `/^[a-zA-Z0-9._-]+$/` | 服务端 ValidationPipe（@Matches） | `BadRequestException` | 原样返回 | `message` = class-validator 错误文本 | `ErrorMessage` / `Snackbar` |
| Name 字段格式非法（数字开头）——前端绕过 | 服务端 entity.service.ts | `ConflictException(NAME_VALIDATION_ERROR_MESSAGE)` | `ApolloError(message)` | `message` = "Name must only contain letters..." | `Snackbar` / 全局错误展示 |
| 使用保留字（`class`、`auth`、`field` 等） | 服务端 entity.service.ts | `ReservedNameError(name)` ——AmplicationError 子类 | `ApolloError(message)` | `message` = `"xxx" is a reserved name and cannot be used.` | `Snackbar` |
| name 与 pluralDisplayName 相同——前端绕过 | 服务端 entity.service.ts | `AmplicationError("The entity name and plural display name cannot be the same.")` | `ApolloError(message)` | `message` = 原字符串 | `Snackbar` |
| 业务规则不满足（Team ID 非法） | 服务端 Service | `AmplicationError(message)` | `ApolloError(message)` | `message` = 传入字符串 | `Snackbar` / `ErrorMessage` |
| 数据冲突（手动校验） | 服务端 Service | `DataConflictError(message)` | `ApolloError(message)` | `message` = 传入字符串 | `Snackbar` |
| 唯一键冲突（email 已注册） | 服务端 Prisma | `PrismaClientKnownRequestError (P2002)` | `GraphQLUniqueKeyException(fields)` | `extensions.code = UNIQUE_KEY_VIOLATION` <br/> `message = "Another record with the same key already exist (...)"` | `Snackbar` 显示 message |
| Topic name 格式非法（含禁止字符） | 前端本地 TopicNameField 组件内联 | `FormikErrors[props.name] = HELP_TEXT` | — | `formik.errors` 内联 | `<ErrorMessage name="name" />`（硬编码 "name"，详见 Bug 分析） |
| Topic name 格式非法——前端绕过 | 服务端 ValidationPipe（@Matches） | `BadRequestException` | 原样返回 | `message` = class-validator 错误文本 | `ErrorMessage` / `Snackbar` |
| 套餐限制（工作区数已满 / Commit 数超限） | 服务端 workspace/... | `BillingLimitationError` | `GraphQLBillingError` | `extensions.code = BILLING_LIMITATION_ERROR` <br/> `extensions.billingFeature` <br/> `extensions.bypassAllowed` | CommitButton：**`LimitationDialog` 不可见**（isOpen 永远 false，Dialog DOM 不渲染） <br/> 实际退化为外层 Snackbar / 与其它组件同级别降级显示 |
| 未登录 / JWT 失效 | 服务端 Passport/JwtStrategy | `UnauthorizedException` (HttpException) | 原样返回 | HTTP 401 | Apollo 透传，路由守卫跳转登录 |
| 未预期异常 / 500 | 服务端任意位置 | 任意 Error | 生产：`GraphQLInternalServerError`<br/>开发：`ApolloError(原始message)` | `extensions.code = INTERNAL_SERVER_ERROR` | `Snackbar` 显示 "Internal server error" |

---

## 八、前后端校验规则映射表

以下规则在前端本地（即时反馈）与服务端（最终兜底）两侧以不同方式实现：

| 校验项 | 前端实现 | 服务端实现 | 是否完全一致 |
|-------|---------|-----------|------------|
| **字段名称格式** <br/> 字母/数字/$/_ 且不以数字开头 | [NameField.tsx#L7](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx#L7) <br/> `NAME_REGEX = /^(?![0-9])[a-zA-Z0-9$_]+$/` | [entity.service.ts#L147](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L147) <br/> `NAME_REGEX = /^(?![0-9])[a-zA-Z0-9$_]+$/` <br/> [entity.service.ts#L3285-L3289](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L3285-L3289) `validateFieldName()` | ✅ 正则完全相同（但前端有大写版本 `CAPITALIZED_NAME_REGEX` 用于 Entity，服务端未区分） |
| **错误文案** <br/> "Name must only contain letters, numbers..." | [NameField.tsx#L9-L15](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx#L9-L15) <br/> `HELP_TEXT` / `CAPITALIZED_HELP_TEXT` | [entity.service.ts#L148-L149](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L148-L149) <br/> `NAME_VALIDATION_ERROR_MESSAGE` | ✅ 普通版本完全一致 |
| **最小长度 2 字符** <br/> displayName / name / pluralDisplayName | JSON Schema `minLength: 2` + `AT_LEAST_TWO_CHARACTERS` 常量 | 仅在少数 DTO 上通过 `@MinLength(8)`（如密码），Entity 等场景服务端无 class-validator 校验，仅由 Prisma schema 兜底 | ⚠️ 前端更严格 |
| **name ≠ pluralDisplayName** | [EntityForm.tsx#L93-L97](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Entity/EntityForm.tsx#L93-L97) <br/> 跨字段自定义校验 `isEqual()` | [entity.service.ts#L1195-L1198](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L1195-L1198) <br/> `newName === newPluralDisplayName` | ✅ 逻辑相同（错误文案略有差异） |
| **保留字检查** | ❌ 前端无对应校验 | [reservedNames.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/reservedNames.ts) 60+ 关键字 <br/> [entity.service.ts#L1201-L1203](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L1201-L1203) 抛 `ReservedNameError` | ❌ 前端缺失（创建 entity 时服务端会自动 append `Model`/`Field`，更新时抛错） |
| **Topic name 格式** <br/> `^[a-zA-Z0-9._-]+$` | [TopicNameField.tsx#L6](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/TopicNameField.tsx#L6) <br/> `TOPIC_REGEX = /^[a-zA-Z0-9._-]+$/` | [TopicCreateInput.ts#L12](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/topic/dto/TopicCreateInput.ts#L12) <br/> `@Matches(/^[a-zA-Z0-9._-]+$/)` | ✅ 正则完全一致 |
| **minimumValue < maximumValue** | [formikValidateJsonSchema.ts#L52-L62](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/formikValidateJsonSchema.ts#L52-L62) 跨字段自定义 | [entity.service.ts#L151-L152](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts#L151-L152) `NUMBER_WITH_INVALID_MINIMUM_VALUE` | ✅ 逻辑相同（错误文案略有差异：`greater than, or equal to,` vs `greater than or equal to,`） |
| **错误提示字段绑定** <br/> Formik `<ErrorMessage name>` | [NameField.tsx#L43-L46](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx#L43-L46) <br/> [TopicNameField.tsx#L32-L35](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/TopicNameField.tsx#L32-L35) <br/> 均硬编码 `name="name"` | — | ⚠️ 前端 Bug：当表单字段名不是 `"name"` 时错误提示不显示，但校验仍阻止提交 |
| **Billing 弹窗 isOpen 状态同步** <br/> 错误出现时自动打开弹窗 | `useCommits` hook 中 `onError` → `setOpenLimitationDialog(true)` | — | ❌ 状态未消费：`CommitButton` 解构 hook 时忽略该状态，本地 state 永远 `false`，Dialog 外层守卫 `return isOpen ? ... : null` 直接返回 `null`（见 §6.5.2.1 双重守卫分析） |

> **设计提示**：前端注释 `/** @todo share code with server */`（[NameField.tsx#L6](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx#L6)）明确指出 NAME_REGEX 等规则未来应抽成共享库以避免前后端漂移。

---

## 九、关键文件索引

| 类别 | 文件路径 |
|------|---------|
| 全局校验管道 | [main.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/main.ts#L69-L74) |
| 错误基类 | [AmplicationError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/errors/AmplicationError.ts) |
| 保留字错误 | [ReservedNameError.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/resource/ReservedNameError.ts) |
| 保留字列表 | [reservedNames.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/reservedNames.ts) |
| Service 层字段校验 | [entity.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/entity.service.ts)（NAME_REGEX、isReservedName、跨字段比较） |
| GraphQL 错误码 | [graphql-error-codes.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/util/graphql-error-codes/src/lib/graphql-error-codes.ts) |
| GraphQL 异常过滤器（核心） | [GqlResolverExceptions.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/GqlResolverExceptions.filter.ts) |
| 过滤器模块 | [exceptionFilters.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/filters/exceptionFilters.module.ts) |
| REST Prisma 错误映射 | [HttpExceptions.filter.ts (plugin-api)](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-plugin-api/src/filters/HttpExceptions.filter.ts) |
| 前端 JSON Schema 校验工具 | [formikValidateJsonSchema.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/formikValidateJsonSchema.ts) |
| 前端 Yup 校验示例 | [CreateGitFormSchema.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Resource/git/dialogs/GitCreateRepo/CreateGitFormSchema/CreateGitFormSchema.ts) |
| 前端字段级内联校验（通用） | [NameField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx)（⚠️ ErrorMessage name 硬编码 bug） |
| 前端字段级内联校验（Topic） | [TopicNameField.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/TopicNameField.tsx)（⚠️ ErrorMessage name 硬编码 bug） |
| Topic 表单综合示例 | [TopicForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Topics/TopicForm.tsx) |
| 前端自动保存触发校验 | [formikAutoSave.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/formikAutoSave.tsx) |
| 前端表单校验综合示例 | [EntityForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Entity/EntityForm.tsx)、[WorkspaceForm.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Workspaces/WorkspaceForm.tsx) |
| 前端错误格式化 | [util/error.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/util/error.ts) |
| 前端 Billing 错误识别 hook | [useCommits.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/hooks/useCommits.ts) |
| 前端 Billing 弹窗按钮 | [CommitButton.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx)（⚠️ isOpen 永远 false，状态未同步） |
| 设计系统 - Dialog 基础组件 | [Dialog.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/ui/design-system/src/lib/components/Dialog/Dialog.tsx)（isOpen 双重守卫） |
| 设计系统 - LimitationDialog 组件 | [LimitationDialog.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/ui/design-system/src/lib/components/LimitationDialog/LimitationDialog.tsx) |
| 前端通用错误展示组件 | [ErrorMessage.tsx](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/ErrorMessage.tsx) |

---

## 十、发现的问题汇总

| # | 问题 | 位置 | 影响 |
|---|------|------|------|
| P1 | **Billing 弹窗不可见**：`CommitButton` 本地 `isOpenLimitationDialog` 永远为 `false`，且未从 hook 同步错误状态 | [CommitButton.tsx#L57-L58](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L57-L58) + [Dialog.tsx#L36](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/libs/ui/design-system/src/lib/components/Dialog/Dialog.tsx#L36) 双重守卫 | 套餐限制触发时用户看不到升级引导弹窗，降级为普通 Snackbar |
| P2 | **hook 状态冗余未消费**：`useCommits` 维护 `isOpenLimitationDialog` 并在 `onError` 中写入，但 `CommitButton` 完全忽略该返回值 | [useCommits.ts#L49](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/hooks/useCommits.ts#L49) + [CommitButton.tsx#L65-L71](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/VersionControl/CommitButton.tsx#L65-L71) | 代码冗余 + P1 的根源 |
| P3 | **NameField 错误绑定硬编码**：`<ErrorMessage name="name" />` 写死为字符串，不随 props 动态变化 | [NameField.tsx#L43-L46](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/NameField.tsx#L43-L46) | 表单字段名不是 `"name"` 时，校验仍阻止提交但错误提示不显示在字段下方 |
| P4 | **TopicNameField 错误绑定硬编码**：同上 | [TopicNameField.tsx#L32-L35](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-client/src/Components/TopicNameField.tsx#L32-L35) | 同上 |
| P5 | **前端缺失保留字校验**：60+ 关键字（`class`、`auth`、`field` 等）在前端无即时校验，仅服务端拦截 | [reservedNames.ts](file:///d:/fz/0601/solo-dogfeeding/code/49-amplication/packages/amplication-server/src/core/entity/reservedNames.ts) | 用户输入保留字后必须等到服务端返回才知道失败，且更新 entity 时才会抛出 `ReservedNameError`（创建时服务端自动加 `Model`/`Field` 后缀静默修复） |
