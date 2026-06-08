# Amplication SSO/OIDC 登录、令牌续期与会话失效协作分析

## 一、整体架构概述

Amplication 采用**双轨认证模式**：
1. **GitHub OAuth2** - 通过 Passport.js 的 `passport-github2` 策略实现
2. **OIDC (Auth0)** - 通过 `express-openid-connect` 中间件实现

系统基于 NestJS 框架构建，采用 JWT 作为会话令牌，结合 Cookie（短期临时传递）和 LocalStorage（长期存储）实现前后端分离的会话管理。

---

## 二、身份提供方（IdP）回调流程深度分析

### 2.1 支持的身份提供方

| 提供方 | 实现方式 | 核心模块 |
|--------|---------|---------|
| GitHub | Passport Strategy | [github.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/github.strategy.ts) |
| Auth0 (OIDC) | express-openid-connect 中间件 | [oidc.middleware.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/oidc.middleware.ts) |
| Local (邮箱密码) | 本地数据库校验 | [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L327-L364) |

### 2.2 Auth0 授权范围（Scope）与自定义 Claims

#### 环境变量配置

| 变量 | 用途 |
|------|------|
| `AUTH_ISSUER_BASE_URL` | Auth0 认证域名（如 `idp.provider.local`） |
| `AUTH_ISSUER_MANAGEMENT_BASE_URL` | Auth0 Management API 域名 |
| `AUTH_ISSUER_CLIENT_ID` | OIDC Client ID |
| `AUTH_ISSUER_CLIENT_SECRET` | OIDC Client Secret（同时用作 session 加密密钥） |
| `AUTH_ISSUER_CLIENT_DB_CONNECTION` | Auth0 数据库连接名称（用于用户名密码认证） |
| `AUTH_ISSUER_CLIENT_DB_CONNECTION_ID` | Auth0 数据库连接 ID |

完整配置见 [env.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/env.ts#L36-L44)。

#### OIDC Scope 与授权参数

在 [oidc.middleware.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/oidc.middleware.ts#L20-L38) 中：

```typescript
this.middleware = auth({
  authRequired: false,
  authorizationParams: {
    response_type: "code",              // 使用 Authorization Code Flow
    scope: "openid profile email",       // ⚠️ 未包含 offline_access
  },
  idpLogout: true,
  clientID,
  clientSecret,
  baseURL: `${baseURL}/auth`,
  routes: {
    login: false,                        // 禁用默认登录路由
    logout: false,                       // 禁用默认登出路由
    postLogoutRedirect: `${this.clientHost}/login`,
  },
  issuerBaseURL: `https://${issuerBaseUrl}`,
  secret: clientSecret,                  // session Cookie 加密密钥
});
```

**关键边界**：
- `scope: "openid profile email"` — **未请求 `offline_access`**，因此 Auth0 **不会返回 refresh_token**。这意味着 IdP 会话过期后，无法无感刷新，用户必须重新走 SSO 流程。
- `response_type: "code"` — 严格使用 OAuth 2.0 Authorization Code Flow，避免 Implicit Flow 的安全问题。
- `secret: clientSecret` — 复用 clientSecret 作为 session Cookie 加密密钥，减少配置项。

#### 自定义 OIDC Claims

Auth0 通过 Rules/Actions 将自定义字段注入到 id_token 中，在 [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/types.ts#L21-L30) 定义：

```typescript
interface AuthProfileCustomClaims {
  /**
   * [Custom claim] 用户登录来源（github/google/sso-integration/db 等）
   */
  identityOrigin?: string;
  /**
   * [Custom claim] 该用户在 IdP 侧累计登录次数
   */
  loginsCount?: number;
}
```

这两个 claims 的使用边界：

1. **`identityOrigin`** — 在 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L253-L274) 创建用户时，写入 Account 审计字段：
   ```typescript
   identityProvider: IdentityProvider.IdentityPlatform,
   identityOrigin: profile.identityOrigin,
   identityLoginsCount: profile.loginsCount,
   ```

2. **`loginsCount`** — 在 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L122-L153) 决定是否追踪首次注册事件：
   ```typescript
   trackCompleteEmailSignup(account, profile, existingUser) {
     const { identityOrigin, loginsCount } = profile;
     if (loginsCount != 1) {   // ⚠️ 只有首次登录（loginsCount===1）才追踪
       return;
     }
     // ...触发 CompleteEmailSignup 埋点
   }
   ```

#### 登录时的 UI 控制参数

在 [auth.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.controller.ts#L70-L95)：

```typescript
@Get(AUTH_LOGIN_PATH)
async auth0Login(@Req() request: Request, @Res() response: Response) {
  const screenHint = request.query.work_email
    ? { screen_hint: "signup", login_hint: request.query.work_email as string }
    : { screen_hint: "login-id" };
  await response.oidc.login({
    authorizationParams: { ...screenHint },
    returnTo: AUTH_AFTER_CALLBACK_PATH,
  });
}
```

边界含义：
- 当 URL 带 `work_email` 查询参数时，`screen_hint: "signup"` 强制 Auth0 展示注册页，`login_hint` 预填邮箱。
- 否则 `screen_hint: "login-id"` 展示登录页。

### 2.3 OIDC (Auth0) 回调完整流程

#### 第一步：OIDC 中间件初始化

在 [auth.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.module.ts#L84-L94) 中，`OpenIDConnectAuthMiddleware` 仅应用于以下 4 个路由：

```
/auth/login       → 发起 OIDC 登录
/auth/logout      → 发起 OIDC 登出
/auth/callback    → 接收 IdP 回调（GET + POST）
/auth/afterCallback → 业务层回调（签发本地 JWT）
```

**边界**：其他所有 API 路由不经过 `express-openid-connect` 中间件，完全依赖 JWT 鉴权。

#### 第二步：登录发起

（同 2.2 第四节描述）

#### 第三步：IdP 回调处理

Auth0 完成认证后，回调到 `/auth/callback`，同时支持 GET 和 POST 方法（用于兼容不同的 OIDC 响应模式）：

```typescript
// GET /auth/callback - 默认 Authorization Code Flow 回调
@Get(AUTH_CALLBACK_PATH)
async auth0Callback(@Req() request: Request, @Res() response: Response) {
  await response.oidc.callback({
    redirectUri: `${this.host}/${AUTH_CALLBACK_PATH}`,
  });
}

// POST /auth/callback - 兼容 form_post 响应模式
@Post(AUTH_CALLBACK_PATH)
async auth0CallbackPost(@Req() request: Request, @Res() response: Response) {
  await response.oidc.callback({
    redirectUri: `${this.host}/${AUTH_CALLBACK_PATH}`,
  });
}
```

`response.oidc.callback()` 由 `express-openid-connect` 提供，在内部执行：
1. 接收 Authorization Code（或 form_post 中的 id_token）
2. 向 IdP `/oauth/token` 端点换取 `access_token`、`id_token`（因无 `offline_access`，**无 refresh_token**）
3. 验证 id_token 的签名、`exp` 过期时间、`iss` issuer、`aud` audience
4. 将 tokenSet 和用户信息加密序列化，写入 **`appSession` Cookie**

> **关键边界**：`appSession` Cookie 由 `express-openid-connect` 管理。未显式配置 `session.rollingDuration` 和 `session.absoluteDuration`，使用库的默认值：
> - 默认 `rolling: true`（滑动续期，每次请求重置 cookie 过期时间）
> - 默认 `rollingDuration: 1 day`（86400 秒）
> - 默认 `absoluteDuration: 7 days`（604800 秒）
>
> 即：IdP 侧会话最长 7 天，连续 1 天无活动自动失效。

#### 第四步：回调后业务逻辑处理

`express-openid-connect` 处理完令牌交换后，会重定向到 `returnTo` 即 `/auth/afterCallback`：

```typescript
@Get(AUTH_AFTER_CALLBACK_PATH)
async authorizationCode(
  @Req() request: GitHubRequest,
  @Res() response: Response
): Promise<void> {
  requiresAuth();                          // ⚠️ 边界守卫：若 appSession 无效/过期，抛出 401
  const profile = <AuthProfile>request.oidc.user;  // 从 appSession 解密出用户信息
  await this.authService.loginOrSignUp(profile, response);
}
```

#### 第五步：登录或注册逻辑（含失效拦截）

在 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L584-L629) 的 `loginOrSignUp` 中：

```typescript
async loginOrSignUp(profile: AuthProfile, response: Response): Promise<void> {
  // 1. 通过 githubId 或 email 查找已有用户
  let user = await this.getAuthUser({
    account: {
      OR: [{ githubId: profile.sub }, { email: profile.email }],
    },
  });

  let isNew: boolean;
  const existingUser = !!user;

  // 2. ⚠️ 边界1：新用户 + SIGN_UP_DISABLED → 拒绝注册
  if (!user) {
    if (this.signupDisabled) {
      response.redirect(301, `${this.clientHost}?error=signup_disabled`);
      return;
    }
    user = await this.createUser(profile);
    isNew = true;
  }

  // 3. ⚠️ 边界2：已存在用户 + SIGN_UP_DISABLED + 免费套餐 → 拒绝登录
  if (
    this.signupDisabled &&
    existingUser &&
    (await this.isFreeTierWorkspace(user.workspace.id))
  ) {
    response.redirect(301, `${this.clientHost}?error=free_tier_disabled`);
    return;
  }

  // 4. 若首次用 SSO 登录，关联 githubId
  if (!user.account.githubId || user.account.githubId !== profile.sub) {
    user = await this.updateUser(user, { githubId: profile.sub });
    isNew = false;
  }

  // 5. 埋点（仅 loginsCount===1 时触发 CompleteEmailSignup）
  this.trackCompleteEmailSignup(user.account, profile, existingUser);

  // 6. 签发 JWT 并写入 Cookie
  await this.configureJtw(response, user, isNew);
}
```

**两个关键失效拦截**：
- **`signup_disabled`**：全局关闭注册时，新用户无法通过 SSO 创建账户
- **`free_tier_disabled`**：全局关闭注册时，已存在的免费套餐工作区用户也被拒绝登录（通过 `isFreeTierWorkspace` 判断）

#### 第六步：JWT 签发与 AJWT Cookie 传递

在 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L631-L670)：

```typescript
async configureJtw(response, user, isNew): Promise<void> {
  const token = await this.prepareToken(user);
  const url = stringifyUrl({
    url: this.clientHost,
    query: { "complete-signup": isNew ? "1" : "0" },
  });

  // 计算 Cookie domain：取 hostname 的最后两段（支持跨子域共享）
  const clientDomain = new URL(url).hostname;
  const cookieDomainParts = clientDomain.split(".");
  const cookieDomain = cookieDomainParts
    .slice(Math.max(cookieDomainParts.length - 2, 0))
    .join(".");

  // ⚠️ 写入临时 AJWT Cookie（精确属性）
  response.cookie("AJWT", token, {
    domain: cookieDomain,   // 例：app.amplication.com → .amplication.com
    secure: true,           // 仅通过 HTTPS 传输
    // 未设置 httpOnly → 前端 JS 可以读取（有意设计，让 setTokenFromCookie 能读到）
    // 未设置 maxAge/expires → 会话 Cookie，关闭浏览器即失效
    // 未设置 sameSite → 使用浏览器默认（Lax）
  });

  response.redirect(301, url);
}
```

### 2.4 GitHub OAuth2 回调流程

GitHub 回调流程与 OIDC 类似，但使用 Passport 中间件：

1. `GET /github` → `GitHubAuthGuard` 触发重定向到 GitHub 授权页
2. GitHub 认证后回调到 `GET /github/callback`
3. `GitHubAuthGuard` 调用 `GitHubStrategy.validate()` 完成用户查找/创建
4. 在 [github.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/github.strategy.ts#L17-L53) 中：

```typescript
async validate(
  accessToken: string,
  refreshToken: string,   // ⚠️ GitHub 返回 refresh_token，但系统完全未存储/使用
  profile: Profile,
  done
): Promise<void> {
  const email = await getEmail(accessToken);
  const user = await this.authService.getAuthUser({...});
  if (!user) {
    return done(null, await this.authService.createGitHubUser(profile, email), { isNew: true });
  }
  // ...同样受 SIGN_UP_DISABLED 等边界约束
  return done(null, user, { isNew: false });
}
```

5. 控制器中调用 `authService.configureJtw()`，流程与 OIDC 完全一致

---

## 三、令牌续期机制分析

### 3.1 令牌体系结构（完整）

| 令牌类型 | 生成位置 | 存储位置 | 用途 | 有效期/续期策略 |
|---------|---------|---------|-----|--------------|
| **User JWT** | [auth.service.ts#prepareToken](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L517-L527) | LocalStorage (`@@TOKEN`) + 内存变量 | 日常 GraphQL/REST API 鉴权 | **未设置 `expiresIn`**，理论永久有效。仅在工作区切换时重新签发 |
| **API Token** | [auth.service.ts#prepareApiToken](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L534-L546) | 数据库（bcrypt 哈希存储，明文只在创建时返回一次） | CI/CD、自动化脚本调用 | **滑动过期 30 天**。每次有效访问自动重置 `lastAccessAt`，连续 30 天无访问自动失效 |
| **IdP Token** (access_token/id_token) | Auth0 | `express-openid-connect` 加密 `appSession` Cookie | 维持 IdP 层会话、获取用户信息 | **最长 7 天，连续 1 天无活动失效**（库默认值）。无 `offline_access` 故无 refresh_token |
| **AJWT Cookie** | [auth.service.ts#configureJtw](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L631-L670) | 浏览器 Cookie（临时） | SSO 回调后向前端传递 JWT | **会话 Cookie**，未设置 `maxAge`，关闭浏览器立即失效 |

### 3.2 JWT 模块配置（关键边界）

在 [auth.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.module.ts#L34-L46)：

```typescript
JwtModule.registerAsync({
  useFactory: async (configService: ConfigService) => ({
    secret: configService.get("JWT_SECRET"),
    // ⚠️ 未配置 signOptions
    // ⚠️ 未配置 signOptions.expiresIn
    // ⚠️ 未配置 signOptions.issuer / audience / jwtid / subject / notBefore
  }),
  inject: [ConfigService],
}),
```

**核心结论**：`@nestjs/jwt` 的 `JwtService.sign()` 在没有 `signOptions.expiresIn` 时，生成的 JWT **不包含 `exp` claim**，签名永不过期。唯一的失效机制是服务端主动校验（用户被软删除）。

JWT 签发代码见 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L517-L546)：

```typescript
async prepareToken(user: AuthUser): Promise<string> {
  const tokenPayload: JwtDto = {
    accountId: user.account.id,
    userId: user.id,
    workspaceId: user.workspace.id,
    permissions: user.permissions,
    type: EnumTokenType.User,
  };
  return this.jwtService.sign(tokenPayload);  // ⚠️ 无任何 signOptions 覆盖
}

async prepareApiToken(...): Promise<string> {
  const tokenPayload: JwtDto = {
    ...,
    type: EnumTokenType.ApiToken,
    tokenId: apiToken.id,
  };
  return this.jwtService.sign(tokenPayload);  // ⚠️ 同样无 expiresIn
}
```

### 3.3 User JWT 的"续签"机制

严格来说，User JWT **没有续签机制**，只有**重新签发**。唯一触发重新签发的场景是**切换当前工作区**：

```
用户选择新工作区
   → 前端 GraphQL 请求 updateCurrentUser（切换 workspaceId）
   → 后端 prepareToken() 重新签发包含新 workspaceId 的 JWT
   → 前端 setToken() 覆盖 LocalStorage
   → ⚠️ 旧 JWT 仍然有效（签名正确、无 exp），可继续使用
```

### 3.4 API Token 滑动续期（精确边界）

在 [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L425-L452)：

```typescript
const TOKEN_EXPIRY_DAYS = 30;

async validateApiToken(args): Promise<boolean> {
  const lastAccessThreshold = subDays(new Date(), TOKEN_EXPIRY_DAYS);

  // 原子操作：查找符合条件的 token 并同时更新 lastAccessAt
  const apiToken = await this.prismaService.apiToken.updateMany({
    where: {
      userId: args.userId,
      id: args.tokenId,
      lastAccessAt: { gt: lastAccessThreshold },  // ⚠️ 边界：> 30天前，非 >=
      user: { deletedAt: null },                   // 用户未软删除
      // ⚠️ token hash 正确性由 JWT 签名保障，此处不再额外比对
    },
    data: {
      lastAccessAt: new Date(),  // 滑动续期：只有条件满足才会重置
    },
  });

  return apiToken.count === 1;
}
```

**精确边界**：
- `lastAccessAt > T-30天` 而非 `>=`，意味着精确 30×24h 前那一刻已经失效
- `updateMany` 是原子操作：续期和校验在同一 SQL 中完成，避免并发问题
- 若 `lastAccessAt` 超过阈值，`updateMany` 返回 `count === 0`，校验失败

### 3.5 前端令牌获取与持久化

#### Cookie → LocalStorage 传递

应用入口 [index.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/index.tsx#L18)：

```typescript
setTokenFromCookie();
```

实现（[authentication.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/authentication.ts#L27-L39)）：

```typescript
export function setTokenFromCookie(): void {
  const tokenFromCookie = getCookie("AJWT");
  if (tokenFromCookie && !token) {   // ⚠️ 边界：只在内存中无 token 时才写入，防止覆盖
    setToken(tokenFromCookie);
    // 删除临时 Cookie（精确的 domain 计算）
    const cookieDomainParts = window.location.hostname.split(".");
    const temporaryCookieDomain = cookieDomainParts
      .slice(Math.max(cookieDomainParts.length - 2, 0))
      .join(".");
    expireCookie("AJWT", temporaryCookieDomain);
  }
}
```

#### 令牌存储与读取

```typescript
export function setToken(newToken: string) {
  token = newToken;                              // 内存变量（快速读取）
  localStorage.setItem("@@TOKEN", newToken);     // LocalStorage 持久化
  eventEmitter.emit("change", token);            // 通知订阅者（路由守卫等）
}

export function getToken(): string {
  return token || localStorage.getItem("@@TOKEN");  // 内存优先，fallback 到 LocalStorage
}
```

#### GraphQL 请求附加

在 [graphqlClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/graphqlClient.ts#L45-L70)：

```typescript
const authLink = setContext((_, { headers }) => {
  const token = getToken();
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : "",  // ⚠️ 无 token 时也会发空字符串
      "analytics-session-id": getSessionId(),
    },
  };
});

// WebSocket 订阅同样附加
new GraphQLWsLink(
  createClient({
    connectionParams: () => ({
      authorization: `Bearer ${getToken()}`,
    }),
  })
);
```

### 3.6 后端 JWT 验证

在 [jwt.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/jwt.strategy.ts#L15-L48)：

```typescript
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(configService: ConfigService, private authService: AuthService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: true,   // ⚠️ 即使存在 exp claim 也忽略（因为根本没设置）
      secretOrKey: configService.get("JWT_SECRET"),
    });
  }

  async validate(req, payload: JwtDto): Promise<AuthUser> {
    // API Token 额外校验滑动过期
    if (payload.type === EnumTokenType.ApiToken) {
      const jwt = ExtractJwt.fromAuthHeaderAsBearerToken()(req);
      const isValid = await this.authService.validateApiToken({
        userId: payload.userId,
        tokenId: payload.tokenId,
        token: jwt,
      });
      if (!isValid === true) {
        throw new UnauthorizedException();
      }
    }

    // ⚠️ 所有 Token 都要检查用户是否存在（含软删除检查）
    const user = await this.authService.getAuthUser({ id: payload.userId });
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

**验证链路完整边界**：
1. Passport-JWT 自动校验 JWT 签名（密钥 `JWT_SECRET`）
2. `ignoreExpiration: true` 跳过 exp 检查
3. API Token 额外走 `validateApiToken` 检查滑动过期 + 用户未删除
4. User Token 只检查用户存在性（`getAuthUser` 过滤 `deletedAt != null`）

---

## 四、会话失效机制分析

### 4.1 五层会话失效边界（完整）

```
┌──────────────────────────────────────────────────────────────────────┐
│ Layer 5: 前端 LocalStorage / Apollo Cache                           │
│   - 用户主动登出：unsetToken() + apolloClient.clearStore()            │
│   - 浏览器清除数据、隐私模式退出、JS 异常等                            │
│   - 无 401 自动处理链路，失效后用户看到的是 GraphQL 错误弹窗           │
├──────────────────────────────────────────────────────────────────────┤
│ Layer 4: 用户活跃性评估（影响 License 许可）                           │
│   - USER_LAST_ACTIVE_DAYS=30：30 天内无 lastActive 更新               │
│     → bulkUpdateWorkspaceProjectsAndResourcesLicensed 将 workspace   │
│       的 projects/services 标记为 unlicensed                          │
│   - 注意：这不是直接的会话失效，是 License 功能层面的降级               │
├──────────────────────────────────────────────────────────────────────┤
│ Layer 3: User JWT 有效性（无状态 + 数据库校验）                        │
│   - JWT 无 expiresIn，签名永久有效                                     │
│   - 通过 getAuthUser 查询 user.deletedAt 判软删除                     │
│   - SIGN_UP_DISABLED + 免费套餐 → 下次 SSO 登录时拦截（非请求级）       │
│   - 工作区切换 → 重新签发新 Token，旧 Token 理论仍有效                 │
├──────────────────────────────────────────────────────────────────────┤
│ Layer 2: API Token 滑动过期                                           │
│   - lastAccessAt 超过 30 天自动失效（> 阈值）                          │
│   - 用户主动 deleteApiToken                                           │
│   - 关联用户被软删除                                                   │
├──────────────────────────────────────────────────────────────────────┤
│ Layer 1: IdP 会话 (express-openid-connect)                            │
│   - appSession Cookie：加密存储 IdP tokens                            │
│   - rolling=true：每次请求重置 1 天滚动窗口                            │
│   - absoluteDuration=7天：最长 7 天后强制失效                         │
│   - idpLogout=true：/auth/logout 同步销毁 Auth0 侧会话                │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 主动登出完整链路

#### 前端 → 后端 → IdP 完整时序

在 [WorkspaceHeader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceHeader.tsx#L59-L64)：

```typescript
const handleSignOut = useCallback(() => {
  unsetToken();                    // Step 1: 清内存 + LocalStorage
  apolloClient.clearStore();       // Step 2: 清 Apollo 缓存（敏感数据不残留）
  window.location.replace(REACT_APP_AUTH_LOGOUT_URI);  // Step 3: 跳后端 /auth/logout
}, [apolloClient]);
```

前端 env 配置（[env.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/env.ts#L17-L18)）：
```typescript
NX_REACT_APP_AUTH_LOGIN_URI: REACT_APP_AUTH_LOGIN_URI,
NX_REACT_APP_AUTH_LOGOUT_URI: REACT_APP_AUTH_LOGOUT_URI,
```

后端 [auth.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.controller.ts#L131-L138)：

```typescript
@Get(AUTH_LOGOUT_PATH)
async auth0Logout(@Req() request: Request, @Res() response: Response) {
  await response.oidc.logout({
    returnTo: `${this.clientHost}/login`,  // Step 4: Auth0 登出完回跳到这里
  });
}
```

`response.oidc.logout()` 执行：
1. 清除服务端 `appSession` Cookie（包含 access_token、id_token）
2. 构造 Auth0 `/v2/logout` URL，携带 `client_id` 和 `returnTo`
3. 302 重定向到 Auth0 登出端点 → Auth0 销毁 SSO 会话
4. Auth0 302 回跳到 `returnTo` → 前端 `/login`

#### `unsetToken` 实现

在 [authentication.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/authentication.ts#L47-L51)：

```typescript
export function unsetToken(): void {
  token = null;                            // 清内存
  localStorage.removeItem("@@TOKEN");      // 清持久化
  eventEmitter.emit("change", token);      // 通知订阅者（PrivateRoute 等）
}
```

### 4.3 用户活跃性与会话/许可边界

#### `setLastActivity` 更新时机

在 [workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L79-L92)：

```typescript
@Query(() => Workspace, { nullable: true })
async currentWorkspace(@UserEntity() currentUser: User): Promise<Workspace | null> {
  await this.analytics.trackWithContext({...});
  await this.userService.setLastActivity(currentUser.id);  // ⚠️ 每次查询都更新
  const externalId = await this.userService.setNotificationRegistry(currentUser);
  return { ...currentUser.workspace, externalId };
}
```

实现（[user.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/user/user.service.ts#L161-L173)）：

```typescript
async setLastActivity(userId: string, lastActive: Date = new Date()): Promise<User> {
  return this.prisma.user.update({
    where: { id: userId },
    data: { lastActive },
  });
}
```

#### `USER_LAST_ACTIVE_DAYS` 的真实影响

这个变量**不直接使会话失效**，而是用于 License 许可的批量评估。在 [workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L609-L658)：

```typescript
private userLastActiveDays: number;
constructor(...) {
  this.userLastActiveDays =
    Number(this.configService.get<string>(Env.USER_LAST_ACTIVE_DAYS)) ?? 30;
}

async bulkUpdateWorkspaceProjectsAndResourcesLicensed(useUserLastActive: boolean): Promise<boolean> {
  const userLastActiveQuery = useUserLastActive
    ? { some: { lastActive: { gte: new Date(date.setDate(date.getDate() - this.userLastActiveDays)) } } }
    : {};

  const workspaces = await this.prisma.workspace.findMany({
    where: {
      users: userLastActiveQuery,  // ⚠️ 只处理 30 天内有活跃用户的 workspace
      projects: { some: { deletedAt: null, resources: { ... } } },
    },
    select: { id: true },
  });

  for (const workspace of workspaces) {
    await this.subscriptionService.updateProjectLicensed(workspace.id);
    await this.subscriptionService.updateServiceLicensed(workspace.id);
  }
  return true;
}
```

**边界含义**：
- 工作区 30 天内无任何用户活跃 → 该工作区不参与 License 批量更新
- 工作区的 projects/services 可能因此被标记为 `licensed: false`，影响功能可用性
- 这是 **License 功能边界**，不是严格的会话失效，但实际效果接近（功能不可用）

### 4.4 被动失效场景（完整列表）

#### 场景 1：用户被软删除
- 触发：`UserService.delete()` 设置 `deletedAt = new Date()`
- 失效时机：下一次请求经过 `JwtStrategy.validate()` → `getAuthUser` 返回 null → `UnauthorizedException`
- 影响范围：User JWT + API Token 同时失效

#### 场景 2：JWT 签名校验失败
- 触发：`JWT_SECRET` 变更、Token 被篡改、Token 格式错误
- 失效时机：Passport-JWT 策略内部自动抛出 401
- 影响范围：所有 JWT 同时失效

#### 场景 3：API Token 30 天滑动过期
- 触发：连续 30 天无任何 API 调用
- 失效时机：`validateApiToken` 中 `lastAccessAt` 条件不满足 → 返回 false → `UnauthorizedException`
- 边界：`lastAccessAt > threshold`（严格大于），精确到毫秒

#### 场景 4：IdP 会话过期
- 触发：`appSession` Cookie 超过 rollingDuration（1天无活动）或 absoluteDuration（7天）
- 失效时机：下次走 `/auth/login` 或 `/auth/afterCallback` 时 `requiresAuth()` 抛 401
- 后续：被 `AuthExceptionFilter` 捕获 → 302 重定向到 `/login?error=...`

#### 场景 5：SIGN_UP_DISABLED + 免费套餐用户
- 触发：管理员设置 `SIGNUP_DISABLED=true`
- 失效时机：仅在用户**重新 SSO 登录**时拦截（`loginOrSignUp` 中检查），**不影响已有会话中的活跃请求**
- 重定向到 `?error=free_tier_disabled`

#### 场景 6：SIGN_UP_DISABLED + 新用户
- 触发：`SIGN_UP_DISABLED=true`，用户首次通过 SSO 登录
- 失效时机：`loginOrSignUp` 立即拦截
- 重定向到 `?error=signup_disabled`

#### 场景 7：前端路由守卫（无 Token）
- 触发：LocalStorage 无 Token（清缓存、首次访问、登出后）
- 失效时机：访问 PrivateRoute 包裹的任意路由
- 行为：`useAuthenticated()` 返回 false → `<Redirect to="/login">`

#### 场景 8：AJWT 临时 Cookie 自然失效
- 触发：关闭浏览器（会话 Cookie），或前端已调用 `expireCookie` 删除
- 影响：只影响 SSO 回调那一次传递，LocalStorage 已有 Token 不受影响

---

## 五、三者协作关系全景

### 5.1 完整登录时序（OIDC 流程，含边界）

```
用户浏览器                     后端 (amplication-server)            Auth0 IdP
    │                                 │                                 │
    │ 1. 点击 Continue with SSO       │                                 │
    │    (可选带 ?work_email=)         │                                 │
    │────────────────────────────────>│                                 │
    │                                 │                                 │
    │ 2. 302 → Auth0 /authorize       │                                 │
    │    response_type=code            │                                 │
    │    scope=openid profile email    │                                 │
    │    screen_hint=signup/login-id   │                                 │
    │<────────────────────────────────│                                 │
    │                                 │                                 │
    │ 3. 用户认证/授权                 │                                 │
    │──────────────────────────────────────────────────────────────────>│
    │                                 │                                 │
    │ 4. 302 → /auth/callback         │                                 │
    │    带 Authorization Code         │                                 │
    │────────────────────────────────>│                                 │
    │                                 │ 5. POST /oauth/token            │
    │                                 │    (Code → Token 交换)          │
    │                                 │────────────────────────────────>│
    │                                 │                                 │
    │                                 │ 6. 返回 access_token + id_token  │
    │                                 │    (无 refresh_token)            │
    │                                 │<────────────────────────────────│
    │                                 │                                 │
    │                                 │ 7. 校验 id_token (签名/exp/iss) │
    │                                 │ 8. 写入加密 appSession Cookie    │
    │                                 │    (rolling 1天, absolute 7天)  │
    │                                 │                                 │
    │ 9. 302 → /auth/afterCallback    │                                 │
    │<────────────────────────────────│                                 │
    │                                 │                                 │
    │────────────────────────────────>│                                 │
    │                                 │ 10. requiresAuth() 校验 session │
    │                                 │ 11. 从 profile 取 identityOrigin │
    │                                 │     loginsCount, sub, email     │
    │                                 │ 12. SIGN_UP_DISABLED 检查       │
    │                                 │ 13. free_tier 检查              │
    │                                 │ 14. 查库/创建 User+Account       │
    │                                 │ 15. prepareToken() 签 JWT       │
    │                                 │     (无 expiresIn)               │
    │                                 │ 16. 写 AJWT Cookie (会话级)      │
    │ 17. 302 → 前端应用               │                                 │
    │<────────────────────────────────│                                 │
    │                                 │                                 │
    │ 18. setTokenFromCookie()         │                                 │
    │     → 读 AJWT → LocalStorage     │                                 │
    │     → expireCookie 删 AJWT       │                                 │
    │ 19. 后续请求附加 Bearer JWT       │                                 │
    │────────────────────────────────>│                                 │
```

### 5.2 日常请求校验链路（含失效边界）

```
前端 (Apollo Client)                     后端 (NestJS)
     │                                         │
     │ 1. GraphQL Query                       │
     │    authLink: Authorization: Bearer XXX  │
     │────────────────────────────────────────>│
     │                                         │
     │                                         │ 2. GqlAuthGuard.canActivate()
     │                                         │    └─> AuthGuard('jwt').canActivate()
     │                                         │        └─> JwtStrategy.validate()
     │                                         │            ├─> Passport 校验签名
     │                                         │            ├─> ignoreExpiration: true
     │                                         │            ├─> [API Token] validateApiToken()
     │                                         │            │    └─> lastAccessAt > NOW-30day ?
     │                                         │            │        ├─ Yes → update lastAccessAt
     │                                         │            │        └─ No  → 401 Unauthorized
     │                                         │            └─> getAuthUser(id)
     │                                         │                └─> deletedAt != null ? 401 : user
     │                                         │
     │                                         │ 3. authorizeContext() 权限检查
     │                                         │    (RBAC per resource)
     │                                         │
     │                                         │ 4. Resolver 执行业务
     │                                         │    [workspace.resolver] currentWorkspace
     │                                         │    └─> setLastActivity(userId) → 更新 lastActive
     │                                         │
     │ 5. data / UnauthorizedError             │
     │<────────────────────────────────────────│
     │                                         │
     │ ⚠️ 前端无全局 401 处理：                  │
     │    不会自动 unsetToken + 跳登录页         │
     │    依赖具体组件错误边界处理                │
```

### 5.3 令牌续期 × 会话失效交互矩阵（完整）

| 事件 | User JWT | API Token | IdP Session (appSession) | User.lastActive | 触发条件/边界代码 |
|-----|---------|-----------|--------------------------|-----------------|-------------------|
| 日常 GraphQL 请求 | 无变化（永不过期） | `lastAccessAt` 滑动续期 | 不经过中间件（无变化） | `currentWorkspace` 查询时更新 | [jwt.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/jwt.strategy.ts) [workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts#L86) |
| 切换工作区 | 重新签发（新 workspaceId） | 不变 | 不变 | 不变 | `updateCurrentUser` → `prepareToken` |
| 30 天无 API 调用 | 不变（无过期） | 失效（滑动窗口） | 不变 | 不变（但 lastActive 30天+影响 License） | [auth.service.ts#validateApiToken](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L425-L452) |
| 7 天无 SSO 登录 | 不变（本地 JWT 独立） | 不变 | 失效（absoluteDuration） | 不变 | `express-openid-connect` 默认配置 |
| 1 天无 SSO 侧活动 | 不变 | 不变 | 失效（rollingDuration） | 不变 | 同上 |
| 主动登出（前端按钮） | LocalStorage + 内存 清除 | 不变（需手动删） | 重定向 → /auth/logout → Auth0 登出 | 不变 | [WorkspaceHeader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceHeader.tsx#L59-L64) |
| 用户被软删除 | 下次请求 401 | 下次请求 401 | 不变（SSO 下次登录时拦截） | 不变 | [user.service.ts#delete](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/user/user.service.ts#L140-L159) |
| `SIGNUP_DISABLED=true` | 现有会话继续有效 | 不变 | 下次 SSO 登录时拦截 新用户 | 不变 | [auth.service.ts#loginOrSignUp](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L595-L603) |
| `SIGNUP_DISABLED=true` + 免费用户 | 现有会话继续有效 | 不变 | 下次 SSO 登录时拦截 | 不变 | [auth.service.ts#loginOrSignUp](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts#L608-L619) |
| `JWT_SECRET` 变更 | 全部失效 | 全部失效 | 不变 | 不变 | `JwtModule.registerAsync` |
| 浏览器关闭 | LocalStorage 保留，内存清除 | 不变 | `appSession` 会话 Cookie 清除 | 不变 | 浏览器行为 |
| 30+ 天无 `currentWorkspace` 查询 | 不变 | 不变 | 不变 | License 批量更新时 workspace 被排除，可能 unlicensed | [workspace.service.ts#bulkUpdateWorkspaceProjectsAndResourcesLicensed](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts#L609-L658) |

### 5.4 设计要点与潜在风险（完整）

#### 设计优势

1. **最小权限 Cookie 传递**：JWT 通过临时 AJWT Cookie 从后端传递到前端，立即转移到 LocalStorage 并删除 Cookie。`secure: true` + 会话级 + 精确 domain，最小化暴露窗口。

2. **OIDC 标准合规**：`response_type: "code"` 使用 Authorization Code Flow，`express-openid-connect` 处理 state、nonce、token 验证等协议细节。

3. **API Token 滑动续期**：原子 `updateMany` 同时完成校验和续期，30 天窗口兼顾安全与自动化脚本可用性。

4. **多层防御纵深**：前端 LocalStorage → 路由守卫 → JWT 签名校验 → 用户存在性检查 → API Token 滑动窗口 → SIGN_UP_DISABLED 登录拦截。

5. **License 与会话解耦**：`USER_LAST_ACTIVE_DAYS` 只影响 License 评估，不直接中断会话，避免活跃用户因批处理延迟被误踢。

#### 潜在风险与改进建议

1. **User JWT 永久有效（高风险）**：`JwtModule` 未设置 `signOptions.expiresIn`，`JwtStrategy` 显式 `ignoreExpiration: true`。Token 一旦泄露可被永久使用。**建议**：设置 `expiresIn: "1d"`，新增 refresh token 或依赖 IdP 会话实现无感续签。

2. **无 Token 撤销机制（高风险）**：除用户软删除外，无法主动撤销 User JWT。改密码、权限变更后，旧 Token 仍然有效。**建议**：引入短期 JWT + Redis 黑名单，或在 JWT 中加入 token 版本号，用户变更时递增版本。

3. **未请求 `offline_access`（中风险）**：`scope` 不含 `offline_access`，无法从 Auth0 获取 refresh_token，IdP 会话过期（7天 absolute）后用户必须重新登录。**建议**：评估是否需要在 scope 中加入 `offline_access` 并存储 refresh_token 实现无感续签。

4. **前端无全局 401 自动处理（中风险）**：Apollo Client 未配置 `onError` 链路统一处理 401。JWT 失效或用户被删时，前端只会在 UI 上显示 GraphQL 错误，不会自动清理 Token 并跳转登录页。**建议**：

```typescript
// graphqlClient.ts 新增
const errorLink = onError(({ graphQLErrors, networkError }) => {
  if (graphQLErrors) {
    for (const err of graphQLErrors) {
      if (err.extensions?.code === 'UNAUTHENTICATED') {
        unsetToken();
        window.location.replace(REACT_APP_AUTH_LOGIN_URI);
      }
    }
  }
});
// ApolloLink.from([errorLink, authLink, ...])
```

5. **AJWT Cookie 未设置 `httpOnly`（低风险）**：因前端需要 JS 读取 Cookie，有意不设 httpOnly。但如果存在 XSS，攻击者可窃取 JWT。已通过立即 expireCookie 最小化窗口，风险可控。

6. **工作区切换后旧 JWT 仍有效（低风险）**：切换工作区只在前端覆盖 LocalStorage，旧 JWT 签名正确仍可使用。但 workspaceId 与实际资源的 RBAC 检查会在 resolver 层拦截越权访问，实际风险有限。

---

## 六、核心文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| OIDC 中间件配置 | [oidc.middleware.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/oidc.middleware.ts) |
| 认证控制器（登录/回调/登出） | [auth.controller.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.controller.ts) |
| 认证服务核心（JWT 签发/校验/登录拦截） | [auth.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.service.ts) |
| JWT 策略（签名校验/API Token 滑动校验） | [jwt.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/jwt.strategy.ts) |
| JWT 载荷定义 | [jwt.dto.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/dto/jwt.dto.ts) |
| GitHub OAuth 策略 | [github.strategy.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/github.strategy.ts) |
| Auth0 Management API 客户端 | [auth0.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/idp/auth0.service.ts) |
| OIDC 自定义 Claims + AuthProfile 定义 | [types.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/types.ts) |
| 认证模块（JWT 配置/中间件注册） | [auth.module.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/auth/auth.module.ts) |
| GraphQL 认证守卫 | [gql-auth.guard.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/guards/gql-auth.guard.ts) |
| OIDC 认证异常过滤器（重定向登录） | [auth-exception.filter.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/filters/auth-exception.filter.ts) |
| 用户活跃性更新（setLastActivity） | [user.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/user/user.service.ts) |
| 工作区 License 批量更新（USER_LAST_ACTIVE_DAYS） | [workspace.service.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/workspace/workspace.service.ts) |
| 工作区 Resolver（currentWorkspace 触发活跃更新） | [workspace.resolver.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/core/workspace/workspace.resolver.ts) |
| 环境变量定义（JWT_SECRET/AUTH_*/USER_LAST_ACTIVE_DAYS） | [env.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-server/src/env.ts) |
| 前端令牌管理（setToken/unsetToken/setTokenFromCookie） | [authentication.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/authentication.ts) |
| 前端 Cookie 工具（expireCookie/getCookie） | [cookie.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/util/cookie.ts) |
| 前端 Apollo Client（authLink/GraphQL 令牌附加） | [graphqlClient.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/graphqlClient.ts) |
| 前端路由守卫（PrivateRoute/useAuthenticated） | [PrivateRoute.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/authentication/PrivateRoute.tsx) |
| 前端登出按钮（WorkspaceHeader/handleSignOut） | [WorkspaceHeader.tsx](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/Workspaces/WorkspaceHeader/WorkspaceHeader.tsx) |
| 前端环境变量（AUTH_LOGOUT_URI/LOGIN_URI） | [env.ts](file:///d:/fz/0601/solo-dogfeeding/code/101-amplication/packages/amplication-client/src/env.ts) |
